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
import { secondToTick } from "@/sing/domain";
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
let lastBpm = 120;
let lastTpqn = 480;
// 口を閉じた状態
const EXP_CLOSE = "__silent";
let lastEndTick = 0;
let endingDir = 0;
let isEnd = false;

const vowelMap = new Map<string, string>([
  ["a", "aa"],
  ["i", "ih"],
  ["u", "ou"],
  ["e", "ee"],
  ["o", "oh"],
]);

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
    });
    currentFrame += phoneme.frameLength;
  }
  return sections;
};

const makeWeight = (target: string) => {
  const weights = new Map<string, number>();
  weights.set("aa", 0);
  weights.set("ih", 0);
  weights.set("ou", 0);
  weights.set("ee", 0);
  weights.set("oh", 0);
  if (weights.has(target)) {
    weights.set(target, 1);
  }
  return weights;
};

const searchSection = (headTick: number) => {
  const section: Section = {
    startFrame: 0,
    frameLength: 0,
    endFrame: 0,
    startTick: 0,
    endTick: 0,
    expression: EXP_CLOSE,
    weight: 1,
    phonemeString: "pau",
  };
  for (const oneframe of frameMap.values()) {
    if (headTick < oneframe.startTick || oneframe.endTick < headTick) {
      continue;
    }
    for (const section of oneframe.sections) {
      if (headTick < section.startTick || section.endTick < headTick) {
        continue;
      }
      return section;
    }
  }
  return section;
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
  const playheadTicks = props.playheadTicks;
  const offsetX = props.offsetX;

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
  // ピッチラインの生成・更新を行う export type Phrase は type.ts にある
  /*
  for (const [phraseKey, phrase] of phrases) {
    if (!phrase.singer || !phrase.query || !phrase.startTime) {
      continue;
    }

    const tempos = [toRaw(phrase.tempos[0])];
    lastBpm = tempos[0].bpm;
    const tpqn = phrase.tpqn;
    lastTpqn = tpqn;
    const startTime = phrase.startTime; // 秒
    //const f0 = phrase.query.f0; // 配列であって
    const phonemes = phrase.query.phonemes;
    const engineId = phrase.singer.engineId;
    // だいたい 24000 / 256
    const frameRate = store.state.engineManifests[engineId].frameRate;
    lastFrameRate = frameRate;
    let pitchLines = pitchLinesMap.get(phraseKey);
    lastPhraseKey = phraseKey;
    document.body.dataset["xPhraseKey"] = phraseKey;

    // フレーズに対応するピッチラインが無かったら生成する
    if (!pitchLines) {
      // 区間を調べる
      const sections = searchSections(phonemes);
      // 有声区間のピッチラインを生成
      pitchLines = [];
      let s = "" + frameRate;
      for (const section of sections) {
        const startFrame = section.startFrame;
        const frameLength = section.frameLength;
        // 各フレームのticksは前もって計算しておく [s0 1 2 3]
        const frameTicksArray: number[] = [];
        for (let j = 0; j < frameLength; j++) {
          const ticks = secondToTick(
            startTime + (startFrame + j) / frameRate, // 秒単位
            tempos,
            tpqn
          );
          frameTicksArray.push(ticks);
        }
        pitchLines.push({
          startFrame,
          frameLength,
          frameTicksArray,
        });
        lastLip = phonemeToExpression.get(section.phonemeString) || lastLip;
        //lastPhonemeString = section.phonemeString || lastPhonemeString;
        s += `, {${section.phonemeString}, ${startFrame}, ${frameLength}}`;
      }
      // lineStripをステージに追加
      pitchLinesMap.set(phraseKey, pitchLines);

      {
        document.body.dataset["xPhoneme"] = s;
        //if (s.length > 0) console.log(s);
      }
    }

    // ピッチラインを更新
    for (let i = 0; i < pitchLines.length; i++) {
      const pitchLine = pitchLines[i];

      // カリングを行う
      const startTicks = pitchLine.frameTicksArray[0];
      const startBaseX = tickToBaseX(startTicks, tpqn);
      const startX = startBaseX * zoomX - offsetX;
      const lastIndex = pitchLine.frameLength - 1;
      const endTicks = pitchLine.frameTicksArray[lastIndex];
      const endBaseX = tickToBaseX(endTicks, tpqn);
      const endX = endBaseX * zoomX - offsetX;

      // ポイントを計算してlineStripに設定＆更新
      for (let j = 0; j < pitchLine.frameLength; j++) {
        const ticks = pitchLine.frameTicksArray[j];
        const baseX = tickToBaseX(ticks, tpqn);
        const x = baseX * zoomX - offsetX;
        const freq = f0[pitchLine.startFrame + j];
        const noteNumber = frequencyToNoteNumber(freq);
        const baseY = noteNumberToBaseY(noteNumber);
        const y = baseY * zoomY - offsetY;
      }
    }
  } */

  // phoneme, expression の生成・更新を行う export type Phrase は type.ts にある
  for (const [phraseKey, phrase] of phrases) {
    if (!phrase.singer || !phrase.query || !phrase.startTime) {
      continue;
    }

    const tempos = [toRaw(phrase.tempos[0])];
    lastBpm = tempos[0].bpm;
    const tpqn = phrase.tpqn;
    lastTpqn = tpqn;
    const startTime = phrase.startTime; // 秒
    //const f0 = phrase.query.f0; // 配列であって
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
      lastEndTick = Math.max(lastEndTick, section.endTick);
      let exp = EXP_CLOSE;
      let weight = 1;
      const consonantWeight = 0.7;
      switch (section.phonemeString) {
        case "m":
        case "b":
        case "p":
        case "N":
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

  const ang =
    ((((Date.now() + offsetX) % 2000) as number) / 2000.0) * Math.PI * 2;

  let mod = 0;
  if (Number.isFinite(playheadTicks)) {
    // 実時間ではなく小節数スケールで進む
    //pose.set("rightLowerArm", [0, 0, (playheadTicks / lastTpqn) * 2 * Math.PI * 0.25]);
    mod = (playheadTicks % (lastTpqn * 4)) / (lastTpqn * 4);
  }
  const pose = new Map<string, [number, number, number]>([
    ["leftUpperArm", [0, 0, -degToRad(50)]],
    ["leftLowerArm", [degToRad(30), -degToRad(140), 0]],
    ["leftHand", [0, -degToRad(10 + 3), degToRad(12)]],
    ["rightUpperArm", [0, 0, degToRad(50)]],
    ["rightLowerArm", [degToRad(30), degToRad(140), 0]],
    ["rightHand", [0, degToRad(10), -degToRad(12)]],
  ]);
  let zrot = degToRad(1.8 * ease(mod));
  let xrot = degToRad(3 * (1 - Math.sin(degToRad(180 * ((mod * 4) % 1)))));

  const past = playheadTicks - lastEndTick;
  if (past > 0) {
    if (!isEnd) {
      const t = Math.min(1, past / lastTpqn);
      zrot = zrot * (1 - t);
      xrot = xrot * (1 - t);
      if (t === 1) {
        isEnd = true;
      }
    }
  } else {
    isEnd = false;
    const t = playheadTicks / lastTpqn;
    if (t < 1) {
      const rot = cosEase({
          x0: 0,
          y0: 0,
          x1: 1,
          y1: -1,
      }, t);
      zrot = degToRad(1.8 * rot);
    }
  }
  if (isEnd) {
    zrot = 0;
    xrot = 0;
  }
  pose.set("chest", [0, 0, zrot]);
  pose.set("head", [xrot, 0, zrot * 4]);

  for (const [key, value] of pose) {
    const bone = currentVrm?.humanoid?.getNormalizedBone(
      key as VRMHumanBoneName
    );
    if (!bone) {
      continue;
    }
    bone.node?.rotation?.set(...value);
  }

  const section = searchSection(playheadTicks);
  const lip = section.expression;
  {
    document.body.dataset["xLip"] = `, ${lip}, ${lastBpm}`;
  }

  const weights = makeWeight(isEnd ? EXP_CLOSE : lip);
  weights.set("Hauu", Math.round((Math.cos(ang) + 1) * 0.5));
  for (const [key, value] of weights) {
    currentVrm?.expressionManager?.setValue(key, value);
  }

  currentVrm?.update(deltaTime);

  renderer.render(scene, camera);
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

const cosEase = (_this: TwoPoint, t: number) => {
  return (
    _this.y0 +
    (1 - Math.cos((Math.PI * (t - _this.x0)) / (_this.x1 - _this.x0))) *
      0.5 *
      (_this.y1 - _this.y0)
  );
};

const sections = [
  { x0: 0, x1: 0.25, y0: -1, y1: -1, f: () => -1 },
  { x0: 0.25, x1: 0.5, y0: -1, y1: 1, f: cosEase },
  { x0: 0.5, x1: 0.75, y0: 1, y1: 1, f: () => 1 },
  { x0: 0.75, x1: 1, y0: 1, y1: -1, f: cosEase },
];

function ease(t: number) {
  for (const section of sections) {
    if (section.x0 <= t && t < section.x1) {
      return section.f(section, t);
    }
  }
  return 1;
}

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
    const lookHeight = 1.15;
    camera.position.set(0, lookHeight, 0.5);
    camera.lookAt(new THREE.Vector3(0, lookHeight, 0));
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
      "./res/model/Zundamon(Human)_VRM_10.vrm",
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
