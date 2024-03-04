<template>
  <div ref="canvasContainer" class="canvas-container"></div>
</template>

<script setup lang="ts">
import { ref, watch, toRaw, computed, onUnmounted, onMounted } from "vue";
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { VRMLoaderPlugin } from "@pixiv/three-vrm";
import { useStore } from "@/store";
import { frequencyToNoteNumber, secondToTick } from "@/sing/domain";
import { noteNumberToBaseY, tickToBaseX } from "@/sing/viewHelper";
import { FramePhoneme } from "@/openapi";

type VoicedSection = {
  readonly startFrame: number;
  readonly frameLength: number;
};

type PitchLine = {
  readonly startFrame: number;
  readonly frameLength: number;
  readonly frameTicksArray: number[];
};

const props =
  defineProps<{
    isActivated: boolean;
    playheadTicks: number;
    offsetX: number;
    offsetY: number;
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
let currentVrm: any | undefined;
let requestId: number | undefined;
let renderInNextFrame = false;
let lastBpm = 120;
let lastTpqn = 480;

/**
 * このコンポーネントの変数
 */
const pitchLinesMap = new Map<string, PitchLine[]>();

const searchVoicedSections = (phonemes: FramePhoneme[]) => {
  const voicedSections: VoicedSection[] = [];
  let currentFrame = 0;
  let voicedSectionStartFrame = 0;
  let voicedSectionFrameLength = 0;
  // NOTE: 一旦、pauではない区間を有声区間とする
  for (let i = 0; i < phonemes.length; i++) {
    const phoneme = phonemes[i];
    if (phoneme.phoneme !== "pau") {
      if (i === 0 || phonemes[i - 1].phoneme === "pau") {
        voicedSectionStartFrame = currentFrame;
      }
      voicedSectionFrameLength += phoneme.frameLength;
      if (i + 1 === phonemes.length || phonemes[i + 1].phoneme === "pau") {
        voicedSections.push({
          startFrame: voicedSectionStartFrame,
          frameLength: voicedSectionFrameLength,
        });
      }
    }
    currentFrame += phoneme.frameLength;
  }
  return voicedSections;
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
/**
 * store.state.phrases からアクセスできるようになるもの
   */
  const phrases = toRaw(store.state.phrases);
  const zoomX = store.state.sequencerZoomX;
  const zoomY = store.state.sequencerZoomY;
  const playheadTicks = props.playheadTicks;
  const offsetX = props.offsetX;
  const offsetY = props.offsetY;

  // 無くなったフレーズを調べて、そのフレーズに対応するピッチラインを削除する
  for (const [phraseKey, pitchLines] of pitchLinesMap) {
    if (!phrases.has(phraseKey)) {
      pitchLinesMap.delete(phraseKey);
    }
  }
  // ピッチラインの生成・更新を行う export type Phrase は type.ts にある
  for (const [phraseKey, phrase] of phrases) {
    if (!phrase.singer || !phrase.query || !phrase.startTime) {
      continue;
    }
    const tempos = [toRaw(phrase.tempos[0])];
    lastBpm = tempos[0].bpm;
    const tpqn = phrase.tpqn;
    lastTpqn = tpqn;
    const startTime = phrase.startTime;
    const f0 = phrase.query.f0;
    const phonemes = phrase.query.phonemes;
    const engineId = phrase.singer.engineId;
    const frameRate = store.state.engineManifests[engineId].frameRate;
    let pitchLines = pitchLinesMap.get(phraseKey);

    // フレーズに対応するピッチラインが無かったら生成する
    if (!pitchLines) {
      // 有声区間を調べる
      const voicedSections = searchVoicedSections(phonemes);
      // 有声区間のピッチラインを生成
      pitchLines = [];
      for (const voicedSection of voicedSections) {
        const startFrame = voicedSection.startFrame;
        const frameLength = voicedSection.frameLength;
        // 各フレームのticksは前もって計算しておく [s0 1 2 3]
        const frameTicksArray: number[] = [];
        for (let j = 0; j < frameLength; j++) {
          const ticks = secondToTick(
            startTime + (startFrame + j) / frameRate,
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
      }
      // lineStripをステージに追加
      pitchLinesMap.set(phraseKey, pitchLines);
    }

    /* // ピッチラインを更新
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
      if (startX >= canvasWidth || endX <= 0) {
        pitchLine.lineStrip.renderable = false;
        continue;
      }
      pitchLine.lineStrip.renderable = true;

      // ポイントを計算してlineStripに設定＆更新
      for (let j = 0; j < pitchLine.frameLength; j++) {
        const ticks = pitchLine.frameTicksArray[j];
        const baseX = tickToBaseX(ticks, tpqn);
        const x = baseX * zoomX - offsetX;
        const freq = f0[pitchLine.startFrame + j];
        const noteNumber = frequencyToNoteNumber(freq);
        const baseY = noteNumberToBaseY(noteNumber);
        const y = baseY * zoomY - offsetY;
        pitchLine.lineStrip.setPoint(j, x, y);
      }
      pitchLine.lineStrip.update();
    }*/
  }

  if (clock) {
    const deltaTime = clock.getDelta();

    const ang =
      ((((Date.now() + offsetX + offsetY + zoomX + zoomY) % 2000) as number) /
        2000.0) *
      Math.PI *
      2;
    const topology = ((Date.now() / 1000.0) * Math.PI * 2.0 * lastBpm / 4.0) / 60.0;

    currentVrm?.expressionManager?.setValue("blink", (Math.cos(ang) + 1) * 0.5);
    const weights = makeWeight("ou");
    for (const [key, value] of weights) {
      currentVrm?.expressionManager?.setValue(key, value);
    }

    const pose = new Map<string, [number, number, number]>();
    //pose.set("leftLowerArm", [0, Math.PI * 1.25, 0]);
    pose.set("leftUpperArm", [0, 0, -Math.PI * 0.25]);
    let mod = 0;
    if (Number.isFinite(playheadTicks)) {
      // 実時間ではなく小節数スケールで進む
      pose.set("rightLowerArm", [0, 0, (playheadTicks / lastTpqn) * 2 * Math.PI * 0.25]);
      mod = (playheadTicks % (lastTpqn * 4)) / (lastTpqn * 4);
    }
    pose.set("chest", [0, 0, Math.PI * 0.005 * ease(mod)]);
    pose.set("head", [0, 0, Math.PI * 0.02 * ease(mod)]);
    for (const [key, value] of pose) {
      const bone = currentVrm?.humanoid?.getNormalizedBone(key);
      if (!bone) {
        continue;
      }
      bone.node?.rotation?.set(...value);
    }
    currentVrm?.update(deltaTime);
  }

  renderer.render(scene, camera);
};

watch(queries, () => {
  renderInNextFrame = true;
});

watch(
  () => [
    store.state.sequencerZoomX,
    store.state.sequencerZoomY,
    props.playheadTicks,
    props.offsetX,
    props.offsetY,
  ],
  () => {
    renderInNextFrame = true;
  }
);

let isInstantiated = false;

const linear = (_this: any, t: number) => {
  return (_this.y1 - _this.y0) * (t - _this.x0) / (_this.x1 - _this.x0) + _this.y0;
};

const first = 0.125;
const second = 0.125;

const sections = [
  { x0: 0, x1: first, y0: -1, y1: -1, f: (t: number) => -1 },
  { x0: first, x1: 0.25, y0: -1, y1: 0, f: linear },
  { x0: 0.25, x1: 0.25 + second, y0: 0, y1: 0, f: (t: number) => 0 },
  { x0: 0.25 + second, x1: 0.5, y0: 0, y1: 1, f: linear },
  { x0: 0.5, x1: 0.5 + first, y0: 1, y1: 1, f: (t: number) => 1 },
  { x0: 0.5 + first, x1: 0.75, y0: 1, y1: 0, f: linear },
  { x0: 0.75, x1: 0.75 + second, y0: 0, y1: 0, f: (t: number) => 0 },
  { x0: 0.75 + second, x1: 1, y0: 0, y1: -1, f: linear },
];

function ease(t: number) {
  for (const section of sections) {
    if (section.x0 <= t && t < section.x1) {
      return section.f(section, t);
    }
  }
  return 1;
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
  renderer.setClearColor(0x000000, 0.5);
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
  pitchLinesMap.clear();
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
  overflow: hidden;
  z-index: 0;
  pointer-events: none;
}
</style>
