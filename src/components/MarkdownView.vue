<script setup lang="ts">
import { ref, watch, onMounted, nextTick } from "vue";
import {
  renderMarkdown,
  renderMath,
  renderMermaid,
} from "../composables/useMarkdown";
import { rewriteImagesAndLinks } from "../composables/useLinkRewriter";

const props = defineProps<{
  source: string;
  currentFile: string;
  rootDir: string;
  renderTick?: number;
}>();
const emit = defineEmits<{
  (e: "rendered", el: HTMLElement): void;
  (e: "internal-link", path: string, hash: string): void;
}>();

const html = ref<string>("");
const root = ref<HTMLElement | null>(null);

let updateRun = 0;

async function update(forceMermaid = false) {
  const run = ++updateRun;
  html.value = renderMarkdown(props.source);
  await nextTick();
  if (run !== updateRun) return;
  if (root.value) {
    rewriteImagesAndLinks(
      root.value,
      { currentFile: props.currentFile, rootDir: props.rootDir },
      (path, hash) => emit("internal-link", path, hash)
    );
    await renderMath(root.value);
    await renderMermaid(root.value, forceMermaid);
    emit("rendered", root.value);
  }
}

async function refreshMermaidTheme() {
  const run = ++updateRun;
  await nextTick();
  if (run !== updateRun || !root.value) return;
  await renderMermaid(root.value, true);
}

onMounted(() => update());
watch(
  () => [props.source, props.currentFile, props.rootDir],
  () => update(),
  { flush: "post" }
);
watch(
  () => props.renderTick,
  () => refreshMermaidTheme(),
  { flush: "post" }
);

defineExpose({ root });
</script>

<template>
  <article ref="root" class="markdown-body" v-html="html"></article>
</template>

<style scoped>
.markdown-body {
  padding: 32px 48px 80px;
  max-width: var(--reader-max-width, 900px);
  margin: 0 auto;
  line-height: var(--reader-line-height, 1.75);
  font-size: var(--reader-font-size, 16px);
  font-family: var(--reader-font-family, inherit);
  color: var(--fg);
}

:root[data-theme="dark"] .markdown-body {
  background: transparent;
  border: none;
  border-radius: 0;
  box-shadow: none;
}
</style>
