<script setup>
import { computed, onBeforeUnmount, onMounted, ref, watch } from "vue";
import FileViewer from "./components/FileViewer.vue";

const data = ref(null);
const isLoading = ref(true);

const routeMode = ref(false)
const currentPath = ref([]);

const hasData = computed(() => !!data.value?.root);

function getPathFromUrl() {
  const url = new URL(window.location.href);
  const pathString = url.searchParams.get("path");
  if (!pathString) {
    return [];
  }
  const segments = pathString.split("/").map((part) => part.trim()).filter(Boolean);
  return segments;
}

function isRoutingEnabledInUrl() {
  const url = new URL(window.location.href);
  return url.searchParams.get("route") === "1";
}

function isPathValid(treeData, path) {
  const root = treeData?.root;
  if (!root || !Array.isArray(path)) {
    return false;
  }

  let currentLevel = root;
  for (const segment of path) {
    const node = currentLevel?.[segment];
    const isFolder = node?.type === "folder";
    if (!node || !isFolder) {
      return false;
    }
    currentLevel = node.children ?? {}
  }
  return true
}

function syncUrlWithState(path, replace = false) {
  const url = new URL(window.location.href);
  url.searchParams.set("route", routeMode.value ? "1" : "0");

  const hasPath = path.length > 0 && routeMode.value;
  if (hasPath) {
    url.searchParams.set("path", path.join("/"));
  } else {
    url.searchParams.delete("path");
  }

  const newUrl = `${url.pathname}${url.search}${url.hash}`;
  if (replace) {
    window.history.replaceState({}, "", newUrl);
  } else {
    window.history.pushState({}, "", newUrl);
  }
}

function handleFolderNavigate(path, pushToHistory = true) {
  if (!data.value || !isPathValid(data.value, path)) {
    return
  }
  currentPath.value = path;

  if (routeMode.value) {
    const useReplace = !pushToHistory;
    syncUrlWithState(path, useReplace);
  }
}

function handlePopState() {
  if (!routeMode.value || !data.value) {
    return;
  }
  const pathFromUrl = getPathFromUrl();
  currentPath.value = isPathValid(data.value, pathFromUrl) ? pathFromUrl : [];
}

async function loadTreeData() {
  try {
    isLoading.value = true;
    const response = await fetch("/data.json");
    const payload = await response.json();
    data.value = payload;

    routeMode.value = isRoutingEnabledInUrl();
    const pathFromUrl = getPathFromUrl();
    currentPath.value = isPathValid(payload, pathFromUrl) ? pathFromUrl : [];

    if (routeMode.value) {
      syncUrlWithState(currentPath.value, true);
    }
  } catch (error) {
    console.error(error);
  } finally {
    isLoading.value = false;
  }
}

watch(routeMode, (routingEnabled) => {
  if (!data.value) {
    return;
  }
  if (routingEnabled) {
    const pathFromUrl = getPathFromUrl();
    currentPath.value = isPathValid(data.value, pathFromUrl) ? pathFromUrl : [];
  }
  syncUrlWithState(currentPath.value, true);
});

onMounted(async () => {
  await loadTreeData();
  window.addEventListener("popstate", handlePopState);
});

onBeforeUnmount(() => {
  window.removeEventListener("popstate", handlePopState);
});
</script>

<template>
  <main class="mx-auto min-h-screen w-full max-w-5xl bg-slate-900 px-4 py-10 text-slate-100 sm:px-6">
    <div class="rounded-2xl border border-slate-700/80 bg-slate-950/90 p-5 shadow-2xl shadow-slate-950/40 sm:p-7">
      <header class="mb-2 flex flex-col gap-3 sm:flex-row sm:items-center sm:justify-between">
        <h1 class="text-2xl font-semibold">Файл-вьювер</h1>
        <label class="inline-flex cursor-pointer items-center gap-3">
          Маршрутизация
          <input v-model="routeMode" type="checkbox" class="peer sr-only" />
          <span
            class="relative h-7 w-12 rounded-full border border-slate-700 bg-slate-800 transition
            after:absolute after:left-[3px] after:top-[3px] after:h-5 after:w-5 after:rounded-full after:bg-white after:transition-transform
            peer-checked:border-indigo-500 peer-checked:bg-indigo-600 peer-checked:after:translate-x-5"
          ></span>
        </label>
      </header>

      <div v-if="isLoading" class="text-sm text-slate-300">Загрузка...</div>
      <FileViewer
        v-else-if="hasData"
        :tree-data="data"
        :route-mode="routeMode"
        :current-path="currentPath"
        @navigate="handleFolderNavigate"
      />
    </div>
  </main>
</template>
