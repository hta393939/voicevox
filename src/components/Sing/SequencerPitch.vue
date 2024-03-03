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
import { LineStrip } from "@/sing/graphics/lineStrip";
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
  defineProps<{ isActivated: boolean; offsetX: number; offsetY: number }>();

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

  const phrases = toRaw(store.state.phrases);
  const zoomX = store.state.sequencerZoomX;
  const zoomY = store.state.sequencerZoomY;
  const offsetX = props.offsetX;
  const offsetY = props.offsetY;

  if (clock) {
    const deltaTime = clock.getDelta();

    const ang =
      ((((Date.now() + offsetX + offsetY + zoomX + zoomY) % 2000) as number) /
        2000.0) *
      Math.PI *
      2;
    currentVrm?.expressionManager?.setValue("aa", Math.cos(ang));
    currentVrm?.expressionManager?.setValue("blink", 1);

    {
      const bone = currentVrm?.humanoid?.getNormalizedBone("leftUpperArm");
      if (bone) {
        //console.log("bone", bone);
        bone.node?.rotation?.set(
          -Math.PI * 0.25,
          -Math.PI * 0.25,
          -Math.PI * 0.25
        );
      }
    }
    {
      const bone = currentVrm?.humanoid?.getNormalizedBone("rightUpperArm");
      if (bone) {
        bone.node?.rotation?.set(0, Math.PI * 0.25, Math.PI * 0.25);
      }
    }

    if (currentVrm) {
      currentVrm.update(deltaTime);
    }
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
    props.offsetX,
    props.offsetY,
  ],
  () => {
    renderInNextFrame = true;
  }
);

let isInstantiated = false;

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
  renderer.setClearColor(0x000000, 0.0);
  scene = new THREE.Scene();

  clock = new THREE.Clock();
  clock.start();

  function setCamera(width: number, height: number) {
    camera = new THREE.PerspectiveCamera(45, width / height, 0.02, 100);
    camera.position.set(0, 0.6, 2);
    camera.lookAt(new THREE.Vector3(0, 0.6, 0));
    renderer?.setViewport(width / 2, 0, width / 2, height / 2); // height は下から
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
