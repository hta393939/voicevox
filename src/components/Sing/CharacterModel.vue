<template>
  <div ref="canvasContainer" class="canvas-container"></div>
</template>

<script setup lang="ts">
import { ref, watch, toRaw, computed, onUnmounted, onMounted } from "vue";
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { VRMHumanBoneName, VRMLoaderPlugin } from "@pixiv/three-vrm";
import type { VRMCore } from "@pixiv/three-vrm";
import { useStore } from "@/store";
import { frequencyToNoteNumber, secondToTick } from "@/sing/domain";
import { FramePhoneme } from "@/openapi";

type Section = {
  readonly startFrame: number;
  readonly frameLength: number;
  readonly endFrame: number; // この値は含まない
  startTick: number;
  endTick: number;
  expression: string;
  weight: number;
  readonly phonemeString: string;
  noteNumber: number;
};

type OneFrame = {
  // 開始フレーム
  readonly startFrame: number;
  readonly frameLength: number;
  readonly endFrame: number; // この値は含まない
  startTick: number;
  endTick: number;
  readonly sections: Section[];
};

const props =
  defineProps<{
    isActivated: boolean;
    playheadTicks: number;
    offsetX: number;
  }>();

const store = useStore();
const queries = computed(() => {
  const phrases = [...store.state.phrases.values()];
  return phrases.map((value) => value.query);
});

/**
 * 送信先
 */
let ws: WebSocket | null = null; // 送信先

const config = {
  model: "Zundamon(Human)_VRM_10.vrm",
  //"model": "ずんだもん.vrm",
  //"model": "Noほうれい線.vrm",
  isZero: false,
  dir: {
    y: -20, // Zundamon 用
    //"y": -10 // ずんだ
  },
  isElbow: false,
  camera: {
    position: [0, 1.15, 0.5], // Zundamon 用
    //"position": [0, 1.1, 0.9] // ずんだ
  },
  motion: {
    long: {
      enable: true,
      ratio: 1.5,
      //"expression": "blink"
      expression: "Hauu", // Zunda
    },
    high: {
      enable: false,
      tone: 70, // 68とか70
      ratio: 0.9,
    },
    last: {
      expression: "Wink_L", // Zunda
      //"expression": "happy" // ずんだ
    },
  },
};
/*
try {
  const configText = await fetch("./res/model/config.json");
  config = JSON.parse(configText);
} catch(e) {
  // 何もしない
} */

const canvasContainer = ref<HTMLElement | null>(null);
let resizeObserver: ResizeObserver | undefined;
let canvasWidth: number | undefined;
let canvasHeight: number | undefined;

let renderer: THREE.WebGLRenderer | undefined;
let scene: THREE.Scene | undefined;
let camera: THREE.OrthographicCamera | THREE.PerspectiveCamera | undefined;
let clock: THREE.Clock | undefined;
let currentVrm: VRMCore | undefined;
let requestId: number | undefined;
let renderInNextFrame = false;
let lastTpqn = 480;
// 口を閉じた状態
const EXP_CLOSE = "__silent";
let lastEndTick = 0;
let isEnd = false;
let lastRot: [number, number, number] | null = null;
// 母音と判定する phoneme をキーとする
const vowelMap = new Map<string, string>([
  ["a", "aa"],
  ["i", "ih"],
  ["u", "ou"],
  ["e", "ee"],
  ["o", "oh"],
]);

// vrm0.0 のときはオイラー回転を変換する
const _conv = (x: number, y: number, z: number) => {
  if (!config.isZero) {
    return [x, y, z];
  }
  return [-x, y, -z];
};

const frameMap = new Map<string, OneFrame>();

/**
 * フレーム単位でセクション配列を返す
 */
const searchSections = (phonemes: FramePhoneme[]) => {
  const sections: Section[] = [];
  let currentFrame = 0;
  for (let i = 0; i < phonemes.length; i++) {
    const phoneme = phonemes[i];
    sections.push({
      startFrame: currentFrame,
      frameLength: phoneme.frameLength,
      endFrame: currentFrame + phoneme.frameLength,
      startTick: 0,
      endTick: 0,
      expression: "",
      weight: 1,
      phonemeString: phoneme.phoneme,
      noteNumber: 60,
    });
    currentFrame += phoneme.frameLength;
  }
  return sections;
};

// 母音すべての 0～1 のウェイトを返す
const makeWeight = (target: string, weight: number) => {
  const weights = new Map<string, number>([
    ["aa", 0],
    ["ih", 0],
    ["ou", 0],
    ["ee", 0],
    ["oh", 0],
  ]);
  if (weights.has(target)) {
    weights.set(target, weight);
  }
  return weights;
};

const searchSection = (headTick: number) => {
  const result: Section = {
    startFrame: 0,
    frameLength: 0,
    endFrame: 0,
    startTick: 0,
    endTick: 0,
    expression: EXP_CLOSE,
    weight: 1,
    phonemeString: "pau",
    noteNumber: 60,
  };
  for (const oneframe of frameMap.values()) {
    if (headTick < oneframe.startTick || oneframe.endTick < headTick) {
      continue;
    }
    for (const section of oneframe.sections) {
      if (headTick < section.startTick || section.endTick < headTick) {
        continue;
      }
      if (section.phonemeString === "pau") {
        continue; // フレームの最後が後続の先頭にかぶる
      }
      return section;
    }
  }
  return result;
};

const degToRad = (deg: number) => {
  return (deg * Math.PI) / 180;
};

const render = () => {
  if (!canvasWidth) {
    throw new Error("canvasWidth is undefined.");
  }
  if (!canvasHeight) {
    throw new Error("canvasHeight is undefined.");
  }
  if (!renderer) {
    throw new Error("renderer is undefined.");
  }
  if (!scene) {
    throw new Error("scene is undefined.");
  }
  if (!camera) {
    throw new Error("camera is undefined.");
  }
  if (!clock) {
    throw new Error("clock is undefined.");
  }
  // フレーズ配列
  const phrases = toRaw(store.state.phrases);
  // 再生位置 チック単位
  const playheadTicks = props.playheadTicks;

  // 無くなったフレーズを調べて、そのフレーズに対応する値を削除する。phraseKey は uuidv4 みたいなもの
  let isDelete = false;
  for (const phraseKey of frameMap.keys()) {
    if (!phrases.has(phraseKey)) {
      frameMap.delete(phraseKey);
      isDelete = true;
    }
  }
  if (isDelete) {
    lastEndTick = 0;
    for (const oneframe of frameMap.values()) {
      lastEndTick = Math.max(lastEndTick, oneframe.endTick);
    }
  }

  // phoneme, expression の生成・更新を行う export type Phrase は src/store/type.ts にある
  for (const [phraseKey, phrase] of phrases) {
    if (!phrase.singer || !phrase.query || !phrase.startTime) {
      continue;
    }

    const tempos = [toRaw(phrase.tempos[0])];
    const tpqn = phrase.tpqn;
    lastTpqn = tpqn;
    const startTime = phrase.startTime; // 秒
    const f0 = phrase.query.f0; // 配列であって
    const phonemes = phrase.query.phonemes;
    const engineId = phrase.singer.engineId;
    // だいたい 24000 / 256
    const frameRate = store.state.engineManifests[engineId].frameRate;
    let oneframe = frameMap.get(phraseKey);
    document.body.dataset["xPhraseKey"] = phraseKey;

    // フレーズに対応するピッチラインが無かったら生成する
    if (oneframe) {
      continue;
    }
    // 区間を調べる
    const sections = searchSections(phonemes);
    oneframe = {
      startFrame: sections[0].startFrame,
      endFrame: sections[sections.length - 1].endFrame,
      frameLength: 0,
      startTick: 0,
      endTick: 0,
      sections,
    };
    oneframe.startTick = secondToTick(
      startTime, // 秒単位
      tempos, // position は tick
      tpqn
    );
    oneframe.endTick = secondToTick(
      startTime + (oneframe.endFrame - oneframe.startFrame) / frameRate,
      tempos,
      tpqn
    );
    let s = "" + frameRate;
    const num = sections.length;
    for (let i = 0; i < num; ++i) {
      const section = sections[i];
      const startFrame = section.startFrame;
      const freq = f0[startFrame];
      const noteNumber = frequencyToNoteNumber(freq);
      //console.log("noteNumber", noteNumber);
      section.noteNumber = noteNumber;
      section.startTick = secondToTick(
        startTime + section.startFrame / frameRate,
        tempos,
        tpqn
      );
      section.endTick = secondToTick(
        startTime + section.endFrame / frameRate,
        tempos,
        tpqn
      );
      if (section.phonemeString !== "pau") {
        lastEndTick = Math.max(lastEndTick, section.endTick);
      }
      let exp = EXP_CLOSE;
      let weight = 1;
      const consonantWeight = 0.7;
      switch (section.phonemeString) {
        case "m":
        case "b":
        case "p":
        case "my":
        case "by":
        case "py":
        case "N":
        case "v":
        case "pau":
          exp = EXP_CLOSE;
          break;
        case "w":
          weight = consonantWeight;
          exp = "ou";
          break;
        case "a":
          exp = "aa";
          break;
        case "i":
          exp = "ih";
          break;
        case "u":
          exp = "ou";
          break;
        case "e":
          exp = "ee";
          break;
        case "o":
          exp = "oh";
          break;
        default:
          {
            weight = consonantWeight;
            const next = sections[i + 1];
            const nextExp = vowelMap.get(next?.phonemeString);
            if (nextExp) {
              exp = nextExp;
            } else {
              exp = "ou";
            }
          }
          break;
      }
      section.expression = exp;
      section.weight = weight;
      s += `, {${section.phonemeString}, ${startFrame}}`;
    }
    frameMap.set(phraseKey, oneframe);
    {
      document.body.dataset["xPhoneme"] = s;
    }
  }

  const deltaTime = clock.getDelta();
  // モーション
  let mod = 0;
  if (Number.isFinite(playheadTicks)) {
    // 実時間ではなく小節数スケールで進む
    mod = (playheadTicks % (lastTpqn * 4)) / (lastTpqn * 4);
  }
  const pose = new Map<string, [number, number, number]>([
    ["leftUpperArm", [0, 0, -degToRad(50)]],
    ["leftHand", [0, -degToRad(10 + 3 * 0), degToRad(12 - 2)]],
    ["leftIndexProximal", [0, 0, degToRad(8)]],
    ["leftMiddleProximal", [0, 0, degToRad(8)]],
    ["leftRingProximal", [0, 0, degToRad(4)]],
    ["rightUpperArm", [0, 0, degToRad(50)]],
    ["rightHand", [0, degToRad(10), -degToRad(12 - 2)]],
    ["rightIndexProximal", [0, 0, -degToRad(8)]],
    ["rightMiddleProximal", [0, 0, -degToRad(8)]],
    ["rightRingProximal", [0, 0, -degToRad(4)]],
  ]);
  if (config.isElbow) {
    pose.set("rightLowerArm", [degToRad(30), degToRad(140), 0]);
    pose.set("leftLowerArm", [degToRad(30), -degToRad(140), 0]);
  }
  pose.set("hips", [0, degToRad(config.dir.y + (config.isZero ? 180 : 0)), 0]);
  let zrot = degToRad(1.8 * ease(mod));
  let xrot = degToRad(3 * (1 - Math.sin(degToRad(180 * ((mod * 4) % 1)))));

  const pastTick = playheadTicks - lastEndTick;
  if (pastTick > 0) {
    if (!isEnd) {
      if (!lastRot) {
        lastRot = [xrot, 0, zrot];
      }

      const t = Math.min(1, pastTick / lastTpqn);
      const cosEase = (tt: number) => {
        return (1 - Math.cos(tt * Math.PI)) * 0.5;
      };
      zrot = lastRot[2] * (1 - cosEase(t));
      xrot = lastRot[0] * (1 - cosEase(t));
      if (t === 1) {
        isEnd = true;
        lastRot = null;
      }
    }
  } else {
    isEnd = false;
    lastRot = null;
    const t = playheadTicks / lastTpqn;
    if (t < 1) {
      const rot = twoPointEase(
        {
          x0: 0,
          y0: 0,
          x1: 1,
          y1: -1,
        },
        t
      );
      zrot = degToRad(1.8 * rot);
    }
  }
  if (isEnd) {
    zrot = 0;
    xrot = 0;
  }
  pose.set("chest", [0, 0, zrot]);
  pose.set("head", [xrot, 0, zrot * 4]);

  // 表情
  const section = searchSection(playheadTicks);
  const lip = section.expression;
  {
    document.body.dataset["xLip"] = `, ${lip}`;
  }

  const weights = makeWeight(isEnd ? EXP_CLOSE : lip, section.weight);
  const len = section.endTick - section.startTick;
  let isHauu = false;
  // 高い音
  if (config.motion.high.enable) {
    if (
      len >= lastTpqn * config.motion.high.ratio &&
      section.noteNumber >= config.motion.high.tone &&
      lip !== EXP_CLOSE
    ) {
      // 高いとき Hauu. 先頭が切れるので1にすると1拍の子音は非対象
      isHauu = true;
    }
  }
  // 長い音
  if (config.motion.long.enable) {
    if (len > lastTpqn * config.motion.long.ratio && lip !== EXP_CLOSE) {
      isHauu = true;
    }
  }
  const hauuWeight = isHauu ? 1 : 0;
  weights.set(config.motion.long.expression, hauuWeight);

  // 最後にウインクなどする場合
  const endAppend = Math.min(1, (pastTick - lastTpqn * 3) / 24);
  weights.set(config.motion.last.expression, endAppend);
  // 首かしげ
  if (endAppend > 0) {
    pose.set("head", [0, 0, degToRad(-5 * endAppend)]);
  }

  // 表情の反映
  let isNeutral = true;
  for (const [key, value] of weights) {
    if (value > 0) {
      isNeutral = false;
    }
  }
  if (isNeutral) {
    weights.set("neutral", 1);
  }
  for (const [key, value] of weights) {
    currentVrm?.expressionManager?.setValue(key, value);
  }
  for (const [key, value] of pose) {
    const bone = currentVrm?.humanoid?.getNormalizedBone(
      key as VRMHumanBoneName
    );
    if (!bone) {
      continue;
    }
    const converted = _conv(...value);
    bone.node?.rotation?.set(converted[0], converted[1], converted[2]);
  }
  // 揺れものの計算
  currentVrm?.update(deltaTime);

  renderer.render(scene, camera);

  if (ws?.readyState === WebSocket.OPEN) {
    const obj = {
      type: "motion",
      //ts: playheadTicks,
      ts100: Math.round(playheadTicks * 100),
      e: {} as any,
      b: {} as any,
    };
    {
      for (const [key, value] of weights) {
        obj.e[key] = value;
      }
      for (const [key, value] of pose) {
        const converted = _conv(...value);
        obj.b[key] = [...converted];
      }
    }
    ws.send(JSON.stringify(obj));
  }
};

watch(queries, () => {
  renderInNextFrame = true;
});

watch(
  () => [props.playheadTicks, props.offsetX],
  () => {
    renderInNextFrame = true;
  }
);

let isInstantiated = false;

type TwoPoint = {
  readonly x0: number;
  readonly y0: number;
  readonly x1: number;
  readonly y1: number;
};

const twoPointEase = (_this: TwoPoint, t: number) => {
  return (
    _this.y0 +
    (1 - Math.cos((Math.PI * (t - _this.x0)) / (_this.x1 - _this.x0))) *
      0.5 *
      (_this.y1 - _this.y0)
  );
};

const sections = [
  { x0: 0, x1: 0.25, y0: -1, y1: -1, f: () => -1 },
  { x0: 0.25, x1: 0.5, y0: -1, y1: 1, f: twoPointEase },
  { x0: 0.5, x1: 0.75, y0: 1, y1: 1, f: () => 1 },
  { x0: 0.75, x1: 1, y0: 1, y1: -1, f: twoPointEase },
];

function ease(t: number) {
  for (const section of sections) {
    if (section.x0 <= t && t < section.x1) {
      return section.f(section, t);
    }
  }
  return 1;
}

const initWS = () => {
  ws = new WebSocket("ws://127.0.0.1:40080/out1234");
  ws.onopen = () => {
    console.log("onopen fire");
  };
  ws.onerror = () => {
    console.log("onerror fire");
  };
  ws.onclose = () => {
    console.log("onclose fire");
    ws = null;
  };
  ws.onmessage = (ev) => {
    //console.log("onmessage", ev.data);
  };
};

const initialize = () => {
  const canvasContainerElement = canvasContainer.value;
  if (!canvasContainerElement) {
    throw new Error("canvasContainerElement is null.");
  }

  canvasWidth = canvasContainerElement.clientWidth;
  canvasHeight = canvasContainerElement.clientHeight;

  const canvasElement = document.createElement("canvas");
  canvasElement.width = canvasWidth;
  canvasElement.height = canvasHeight;
  canvasContainerElement.appendChild(canvasElement);
  renderer = new THREE.WebGLRenderer({
    canvas: canvasElement,
    antialias: true,
  });
  renderer.setClearColor(0x000000, 0);
  scene = new THREE.Scene();

  clock = new THREE.Clock();
  clock.start();

  function setCamera(width: number, height: number) {
    camera = new THREE.PerspectiveCamera(45, width / height, 0.02, 100);
    const pos = config.camera.position;
    camera.position.set(pos[0], pos[1], pos[2]);
    camera.lookAt(new THREE.Vector3(0, pos[1], 0));
    const portWidth = width * 0.6;
    const portHeight = height * 0.6;
    renderer?.setViewport(width - portWidth, 0, portWidth, portHeight); // height は下から
  }

  setCamera(4, 3);

  {
    const light = new THREE.DirectionalLight(0xffffff, Math.PI);
    light.position.set(1, 1, 1);
    scene?.add(light);
  }

  const callback = () => {
    if (renderInNextFrame) {
      render();
      //renderInNextFrame = false;
    }
    requestId = window.requestAnimationFrame(callback);
  };
  renderInNextFrame = true;
  requestId = window.requestAnimationFrame(callback);

  resizeObserver = new ResizeObserver(() => {
    if (renderer == undefined) {
      throw new Error("renderer is undefined.");
    }
    const canvasContainerWidth = canvasContainerElement.clientWidth;
    const canvasContainerHeight = canvasContainerElement.clientHeight;

    if (canvasContainerWidth > 0 && canvasContainerHeight > 0) {
      canvasWidth = canvasContainerWidth;
      canvasHeight = canvasContainerHeight;
      renderer.setSize(canvasWidth, canvasHeight);
      renderInNextFrame = true;
      setCamera(canvasWidth, canvasHeight);
    }
  });
  resizeObserver.observe(canvasContainerElement);

  {
    const loader = new GLTFLoader();
    loader.register((parser) => {
      return new VRMLoaderPlugin(parser);
    });
    loader.load(
      `./res/model/${config.model}`,
      (gltf) => {
        const vrm = gltf.userData.vrm;
        scene?.add(vrm.scene);
        console.log("vrm success", vrm);
        currentVrm = vrm;

        console.log(
          "vrm expression",
          vrm?.expressionManager?.mouthExpressionNames
        );
      },
      (progress: ProgressEvent) => {
        console.log("vrm progress", (100 * progress.loaded) / progress.total);
      },
      (error) => {
        console.error("vrm loader", error);
      }
    );
  }

  isInstantiated = true;

  if (!ws) {
    initWS();
  }
};

const cleanUp = () => {
  if (requestId != undefined) {
    window.cancelAnimationFrame(requestId);
  }
  scene?.clear();
  frameMap.clear();
  renderer?.dispose();
  resizeObserver?.disconnect();

  isInstantiated = false;
};

let isMounted = false;

onMounted(() => {
  isMounted = true;
  if (props.isActivated) {
    initialize();
  }
});

watch(
  () => props.isActivated,
  (isActivated) => {
    if (!isMounted) {
      return;
    }
    if (isActivated && !isInstantiated) {
      initialize();
    }
    if (!isActivated && isInstantiated) {
      cleanUp();
    }
  }
);

onUnmounted(() => {
  if (isInstantiated) {
    cleanUp();
  }
});
</script>

<style scoped lang="scss">
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

.canvas-container {
  opacity: 0.5;
  overflow: hidden;
  z-index: 0;
  pointer-events: none;
}
</style>
