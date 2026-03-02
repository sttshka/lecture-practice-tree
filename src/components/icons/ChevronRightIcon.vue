<template>
  <svg
    xmlns="http://www.w3.org/2000/svg"
    viewBox="0 0 20 20"
    fill="currentColor"
    class="h-4 w-4 shrink-0 transition-transform"
    :class="expanded ? 'rotate-90' : ''"
    aria-hidden="true"
  >
    <path
      fill-rule="evenodd"
      d="M7.21 4.47a.75.75 0 011.06 0l4.25 4.25a.75.75 0 010 1.06L8.27 14.03a.75.75 0 11-1.06-1.06L10.94 9.25 7.21 5.53a.75.75 0 010-1.06z"
      clip-rule="evenodd"
    />
  </svg>
</template>

<script setup>
defineProps({
  expanded: {
    type: Boolean,
    default: false,
  },
});
</script>
