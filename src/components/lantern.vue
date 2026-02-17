<template>
  <div class="pointer-events-none fixed inset-x-0 top-2.5 z-9999 opacity-80">
    <div
      v-for="(text, index) in texts"
      :key="index"
      :class="[
        'fixed transition-all duration-500',
        positionClasses[index] || positionClasses[0],

        'max-md:-top-5 max-md:scale-[0.4]',

        index === 0 && 'max-md:-left-8',
        index === 1 && 'max-md:left-2',
        index === 2 && 'max-md:right-2',
        index === 3 && 'max-md:-right-8',
      ]">
      <div
        class="relative h-23.75 w-30 animate-[swing_3.5s_ease-in-out_infinite] rounded-full bg-linear-to-b from-red-600 via-red-500 to-red-700 shadow-[0_0_25px_rgba(255,80,0,0.6),0_0_60px_rgba(255,120,0,0.4)]">
        <!-- 悬挂线 -->
        <div
          class="absolute -top-6 left-1/2 h-6 w-0.5 -translate-x-1/2 bg-amber-500" />

        <!-- 主体 -->
        <div
          class="absolute inset-0 mx-auto mt-1.25 flex h-22.5 w-25 justify-center rounded-full border-2 border-amber-400/80 bg-red-500/20">
          <div
            class="flex h-20.5 w-16.25 items-center justify-center rounded-[60%] border border-amber-400/60 bg-red-500/10">
            <span
              class="bg-linear-to-b from-amber-200 to-amber-500 bg-clip-text text-[2.6rem] font-bold text-transparent drop-shadow-md md:text-[3.5rem]"
              style="
                font-family: 'KaiTi', 'STKaiti', 'Microsoft YaHei', sans-serif;
              ">
              {{ text }}
            </span>
          </div>
        </div>

        <!-- 穗子 -->
        <div
          class="absolute top-22.5 left-1/2 h-6 w-1 origin-top -translate-x-1/2 animate-[swing_4.5s_ease-in-out_infinite] rounded-b-md bg-orange-500">
          <div
            class="absolute top-5 left-1/2 h-9 w-3 -translate-x-1/2 rounded-b-md bg-orange-500" />
          <div
            class="absolute top-4 left-1/2 h-3 w-3 -translate-x-1/2 rounded-full bg-amber-400" />
        </div>

        <!-- 顶部 -->
        <div
          class="absolute -top-2 left-1/2 h-3 w-15 -translate-x-1/2 rounded-md border border-amber-400 bg-linear-to-r from-amber-500 via-orange-400 to-amber-500" />

        <!-- 底部 -->
        <div
          class="absolute -bottom-2 left-1/2 h-3 w-15 -translate-x-1/2 rounded-md border border-amber-400 bg-linear-to-r from-amber-500 via-orange-400 to-amber-500" />
      </div>
    </div>
  </div>
  <KongmingLantern
    ref="lanternRef"
    v-model:release="isLanternReleased"
    :initialCount="initialCount"
    :maxCount="maxCount"
    :stop-spawning="stopSpawningLanterns"
    :visible="showLanterns"
    :speed-multiplier="5" />
</template>

<script setup>
  import { computed, ref, onMounted, onBeforeUnmount } from "vue";
  import KongmingLantern from "./kongmingLantern.vue";
  import { useMediaQuery } from "@vueuse/core";

  const props = defineProps({
    text: {
      type: String,
      default: "新年快乐",
    },
  });

  const texts = computed(() => props.text.split(""));

  const positionClasses = [
    "top-3 left-2.5",
    "top-3 left-[110px] z-10",
    "top-3 right-[110px] z-10",
    "top-3 right-2.5",
  ];

  const lanternRef = ref(null);
  const isLanternReleased = ref(false);
  const stopSpawningLanterns = ref(false);
  const showLanterns = ref(true);

  const isMobile = useMediaQuery("(max-width: 640px)");
  const initialCount = computed(() => (isMobile.value ? 40 : 80));
  const maxCount = computed(() => (isMobile.value ? 60 : 200));

  const AUTO_RELEASE_KEY = "lantern_auto_released_session_2026";

  let timer = null;

  onMounted(() => {
    const hasReleased = sessionStorage.getItem(AUTO_RELEASE_KEY);
    if (hasReleased) return;

    timer = setTimeout(() => {
      if (!isLanternReleased.value) {
        isLanternReleased.value = true;
        stopSpawningLanterns.value = true;

        sessionStorage.setItem(AUTO_RELEASE_KEY, "1");
      }
    }, 1800);
  });

  onBeforeUnmount(() => {
    if (timer) {
      clearTimeout(timer);
      timer = null;
    }
  });
</script>
