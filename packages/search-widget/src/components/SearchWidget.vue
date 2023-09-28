<script lang="ts" setup>
import { VModal } from "@halo-dev/components";
import { ref, watch } from "vue";
import debounce from "lodash.debounce";
import axios from "axios";
import type { Result, Hit } from "@/types/model";

const props = withDefaults(
  defineProps<{
    visible: boolean;
    baseUrl?: string;
  }>(),
  {
    visible: false,
    baseUrl: "",
  }
);

const emit = defineEmits<{
  (e: "update:visible", visible: boolean): void;
}>();

const searchInput = ref<HTMLInputElement | null>(null);
const keyword = ref("");

const selectedIndex = ref(0);
const searchResults = ref<Result>({
  hits: [],
  keyword: "",
  total: 0,
  limit: 0,
  processingTimeMills: 0,
});

const handleSearch = debounce(() => {
  axios
    .get<Result>(
      `${
        props.baseUrl
      }/apis/api.halo.run/v1alpha1/indices/post?keyword=${keyword.value.trim()}&highlightPreTag=<mark>&highlightPostTag=</mark>`
    )
    .then((response) => {
      searchResults.value = response.data;
    });
}, 300);

const handleKeydown = (e: KeyboardEvent) => {
  if (!props.visible) {
    return;
  }

  const { key, ctrlKey } = e;

  if (key === "ArrowUp" || (key === "k" && ctrlKey)) {
    selectedIndex.value = Math.max(0, selectedIndex.value - 1);
    e.preventDefault();
  }

  if (key === "ArrowDown" || (key === "j" && ctrlKey)) {
    selectedIndex.value = Math.min(
      searchResults.value.hits.length - 1,
      selectedIndex.value + 1
    );
    e.preventDefault();
  }

  if (key === "Enter") {
    const searchResult = searchResults.value.hits[selectedIndex.value];
    if (searchResult) {
      handleOpenLink(searchResult);
    }
  }

  if (key === "Escape") {
    onVisibleChange(false);
    e.preventDefault();
  }
};

const handleOpenLink = (hit: Hit) => {
  window.location.href = hit.permalink;
};

watch(
  () => selectedIndex.value,
  (index) => {
    if (index > 0) {
      document.getElementById(`search-item-${index}`)?.scrollIntoView();
      return;
    }
    document.getElementById("search-input")?.scrollIntoView();
  }
);

watch(
  () => keyword.value,
  () => {
    selectedIndex.value = 0;
  }
);

watch(
  () => props.visible,
  (visible) => {
    if (visible) {
      setTimeout(() => {
        searchInput.value?.focus();
      }, 100);

      document.addEventListener("keydown", handleKeydown);
    } else {
      document.removeEventListener("keydown", handleKeydown);
      keyword.value = "";
      selectedIndex.value = 0;
    }
  }
);

const onVisibleChange = (visible: boolean) => {
  emit("update:visible", visible);
};
</script>

<template>
  <VModal
    :visible="visible"
    :body-class="['!p-0']"
    :width="650"
    :centered="false"
    layer-closable
    @update:visible="onVisibleChange"
  >
    <div id="search-input" class="m-2 p-2">
      <div class=" text-blue-600 dark:text-amber-400 mb-3 font-bold text-base">搜索</div>
      <input
        ref="searchInput"
        v-model="keyword"
        placeholder="输入关键词以搜索"
        class="w-full py-1 outline-none rounded-lg transition-all bg-gray-50 border px-2 border-gray-300 hover:border-blue-600 text-gray-600 dark:bg-zinc-700 dark:border-zinc-600 dark:text-zinc-400 dark:hover:border-amber-400"
        autocomplete="off"
        autocorrect="off"
        spellcheck="false"
        @input="handleSearch"
      />
    </div>
    <div class="px-4 mb-4">
      <div
        v-if="!searchResults.hits.length"
        class="flex items-center justify-center text-sm text-gray-500"
      >
        <span>没有搜索结果</span>
      </div>
      <ul
        v-else
        class="box-border flex h-full w-full flex-col gap-1"
        role="list"
      >
        <li
          v-for="(item, itemIndex) in searchResults.hits"
          :id="`search-item-${itemIndex}`"
          :key="itemIndex"
          class="cursor-pointer"
          @click="handleOpenLink(item)"
        >
          <div
            class="flex flex-col gap-1 px-2 py-2 rounded-md hover:bg-gray-50 dark:hover:bg-zinc-700 transition-all"
            :class="{
              'bg-gray-100 dark:bg-zinc-900': selectedIndex === itemIndex,
            }"
          >
            <div
              v-if="item.title"
              class="text-sm font-semibold text-gray-600 dark:text-zinc-400"
              v-html="item.title"
            ></div>
            <div
              v-if="item.content"
              class="text-xs font-light text-gray-500 dark:text-zinc-400"
              v-html="item.content"
            ></div>
          </div>
        </li>
      </ul>
    </div>
    <div class="border-t border-gray-100 dark:border-zinc-600 px-4 py-2.5">
      <div class="flex items-center justify-end">
        <span class="mr-1 text-xs text-gray-600 dark:text-zinc-400">选择</span>
        <kbd
          class="mr-1 w-5 rounded border p-0.5 text-center text-[10px] text-gray-600 shadow-sm dark:text-zinc-400 dark:border-zinc-600"
        >
          ↑
        </kbd>
        <kbd
          class="mr-5 w-5 rounded border p-0.5 text-center text-[10px] text-gray-600 shadow-sm dark:text-zinc-400 dark:border-zinc-600"
        >
          ↓
        </kbd>
        <span class="mr-1 text-xs text-gray-600 dark:text-zinc-400">确认</span>
        <kbd
          class="mr-5 rounded border p-0.5 text-[10px] text-gray-600 shadow-sm dark:text-zinc-400 dark:border-zinc-600"
        >
          Enter
        </kbd>
        <span class="mr-1 text-xs text-gray-600 dark:text-zinc-400">关闭</span>
        <kbd class="rounded border p-0.5 text-[10px] text-gray-600 shadow-sm dark:text-zinc-400 dark:border-zinc-600">
          Esc
        </kbd>
      </div>
    </div>
  </VModal>
</template>
