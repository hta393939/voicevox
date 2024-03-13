<template>
  <div class="frame-page-wrap" :class="{ hide: !isShowSinger }">
    <iframe class="frame-page" :src="framePath"></iframe>
  </div>
</template>

<script setup lang="ts">
import { computed } from "vue";
import { useStore } from "@/store";

const store = useStore();
const isShowSinger = computed(() => store.state.isShowSinger);

const framePath = computed(() => {
  return "./res/model/index.html?_dc=12345678912";
});
</script>

<style scoped lang="scss">
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

// 画面左下に固定表示
// 幅固定、高さ可変、画像のアスペクト比を保持、wrapのwidthに合わせてheightを調整
// bottom位置はスクロールバーの上に表示
.frame-page-wrap {
  overflow: hidden;
  contain: layout;
  pointer-events: none;
  position: fixed;
  bottom: 0;
  width: 100%;
}

.frame-page {
  border: none;
  width: 100%;
  height: 6rem;
}
</style>
