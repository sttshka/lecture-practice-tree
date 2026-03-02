<script setup>
import { computed, ref } from "vue";
import ChevronRightIcon from "./icons/ChevronRightIcon.vue";
import FileIcon from "./icons/FileIcon.vue";
import FolderIcon from "./icons/FolderIcon.vue";

const props = defineProps({
  treeData: { type: Object, default: null },
  routeMode: { type: Boolean, default: false },
  currentPath: { type: Array, default: () => [] },
});

const emit = defineEmits(["navigate"]);

const rootChildren = computed(() => {
  if (!props.treeData || typeof props.treeData !== "object") {
    return {};
  }
  return props.treeData.root ?? {};
});

function sortFolderContents(entries) {
  const pairs = Object.entries(entries);
  return pairs.sort(([nameA, nodeA], [nameB, nodeB]) => {
    const aIsFolder = nodeA?.type === "folder";
    const bIsFolder = nodeB?.type === "folder";
    if (aIsFolder && !bIsFolder) return -1;
    if (!aIsFolder && bIsFolder) return 1;
    return nameA.localeCompare(nameB);
  });
}

function getChildrenAtPath(path) {
  let currentLevel = rootChildren.value;
  for (const segment of path) {
    const node = currentLevel?.[segment];
    if (!node || node.type !== "folder") {
      return null;
    }
    currentLevel = node.children ?? {};
  }
  return currentLevel;
}

const currentFolderContents = computed(() => {
  const children = getChildrenAtPath(props.currentPath);
  if (!children) {
    return [];
  }
  const sorted = sortFolderContents(children);
  return sorted.map(([name, node]) => ({ name, node }));
});

const expandedFolders = ref(new Set());

// обход дерева в глубину и сбор плоского списка всех нод
const allNodesFlattened = computed(() => {
  const result = [];

  function walk(entries, depth, pathPrefix, ancestorKeys) {
    const sorted = sortFolderContents(entries);
    for (const [name, node] of sorted) {
      const path = [...pathPrefix, name];
      const pathKey = path.join("/");
      result.push({
        name,
        node,
        depth,
        path,
        pathKey,
        ancestorFolderKeys: ancestorKeys,
      });
      const isFolderWithChildren = node?.type === "folder" && node.children;
      if (isFolderWithChildren) {
        walk(node.children, depth + 1, path, [...ancestorKeys, pathKey]);
      }
    }
  }

  walk(rootChildren.value, 0, [], [])
  return result;
});

const nodesVisibleInTree = computed(() => {
  return allNodesFlattened.value.filter((item) => {
    const allAncestorsExpanded = item.ancestorFolderKeys.every((key) =>
      expandedFolders.value.has(key)
    );
    console.log({allAncestorsExpanded});
    return allAncestorsExpanded;
  });
});

const breadcrumbItems = computed(() => {
  const items = [{ label: "root", path: [] }];
  props.currentPath.forEach((segment, index) => {
    const pathToSegment = props.currentPath.slice(0, index + 1);
    items.push({ label: segment, path: pathToSegment });
  });
  return items;
});

function emitNavigate(path) {
  emit("navigate", path);
}

function navigateIntoFolder(folderName) {
  const newPath = [...props.currentPath, folderName];
  emitNavigate(newPath);
}

function toggleFolderExpanded(pathKey) {
  const next = new Set(expandedFolders.value);
  if (next.has(pathKey)) {
    next.delete(pathKey);
  } else {
    next.add(pathKey);
  }
  expandedFolders.value = next;
}

function isFolderExpanded(pathKey) {
  return expandedFolders.value.has(pathKey);
}
</script>

<template>
  <div class="w-full">
    <div v-if="!routeMode">
      <div v-if="allNodesFlattened.length === 0" class="text-sm text-slate-400">Пусто</div>
      <ul v-else class="space-y-1">
        <li
          v-for="item in nodesVisibleInTree"
          :key="item.pathKey"
          class="rounded-xl border border-transparent bg-transparent transition"
          :class="item.node.type === 'folder' ? 'hover:border-gray-500 hover:bg-slate-950/50' : ''"
        >
          <div
            class="flex min-h-10 items-center gap-2 px-3 text-slate-100"
            :class="item.node.type === 'folder' ? 'cursor-pointer' : ''"
            :style="{ paddingLeft: `${item.depth * 18 + 12}px` }"
            @click="item.node.type === 'folder' ? toggleFolderExpanded(item.pathKey) : null"
          >
            <button
              v-if="item.node.type === 'folder'"
              type="button"
              class="inline-flex h-5 w-5 items-center justify-center rounded text-slate-400 hover:bg-slate-800 hover:text-slate-200"
              @click.stop="toggleFolderExpanded(item.pathKey)"
            >
              <ChevronRightIcon :expanded="isFolderExpanded(item.pathKey)" />
            </button>
            <span v-else class="inline-block h-5 w-5"></span>

            <FolderIcon v-if="item.node.type === 'folder'" class="text-slate-300" />
            <FileIcon v-else class="text-slate-400" />
            <span class="truncate text-sm">{{ item.name }}</span>
          </div>
        </li>
      </ul>
    </div>

    <div v-else>
      <div class="mb-3 flex flex-wrap items-center gap-2">
        <template v-for="(crumb, index) in breadcrumbItems" :key="crumb.path.join('/') || 'root'">
          <button
            type="button"
            class="rounded-full border border-slate-700/90 bg-slate-900 px-3 py-1 text-xs text-slate-200 transition hover:border-gray-500"
            @click="emitNavigate(crumb.path)"
          >
            {{ crumb.label }}
          </button>
          <span
            v-if="index < breadcrumbItems.length - 1"
            class="text-slate-500"
            aria-hidden="true"
          >/</span>
        </template>
      </div>

      <div v-if="currentFolderContents.length === 0" class="text-sm text-slate-400">Папка пуста</div>
      <ul v-else class="space-y-2">
        <li v-for="item in currentFolderContents" :key="item.name">
          <button
            v-if="item.node.type === 'folder'"
            type="button"
            class="flex min-h-10 w-full items-center gap-2 rounded-xl border border-slate-700/90 bg-slate-800/60 px-3 text-left text-sm text-slate-100 transition hover:border-gray-500"
            @click="navigateIntoFolder(item.name)"
          >
            <FolderIcon class="text-slate-200" />
            <span class="truncate">{{ item.name }}</span>
          </button>
          <div
            v-else
            class="flex min-h-10 items-center gap-2 rounded-xl border border-slate-800/90 bg-slate-950/70 px-3 text-sm text-slate-200"
          >
            <FileIcon class="text-slate-400" />
            <span class="truncate">{{ item.name }}</span>
          </div>
        </li>
      </ul>
    </div>
  </div>
</template>