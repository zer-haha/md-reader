<script setup lang="ts">
import { computed, nextTick, onMounted, ref, watch } from "vue";

const props = defineProps<{
  modelValue: string;
  currentFile: string;
}>();

const emit = defineEmits<{
  (e: "update:modelValue", value: string): void;
  (e: "rendered", el: HTMLElement): void;
}>();

const root = ref<HTMLElement | null>(null);

function escapeHtml(value: string): string {
  return value
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;");
}

function highlightInline(value: string): string {
  return escapeHtml(value)
    .replace(/(`[^`]*`)/g, '<span class="src-code">$1</span>')
    .replace(/(\*\*[^*]+\*\*)/g, '<span class="src-strong">$1</span>')
    .replace(/(__[^_]+__)/g, '<span class="src-strong">$1</span>')
    .replace(/(\[[^\]]+\])(\([^)]+\))/g, '<span class="src-link">$1</span><span class="src-url">$2</span>');
}

function highlightSource(source: string): string {
  const lines = source.split("\n");
  let inFence = false;

  return lines
    .map((line) => {
      if (/^\s*```/.test(line)) {
        inFence = !inFence;
        return `<span class="src-fence">${escapeHtml(line)}</span>`;
      }

      if (inFence) return `<span class="src-codeblock">${escapeHtml(line)}</span>`;

      const heading = line.match(/^(#{1,6})(\s+.*)?$/);
      if (heading) {
        return `<span class="src-marker">${escapeHtml(heading[1])}</span><span class="src-heading">${highlightInline(heading[2] || "")}</span>`;
      }

      const quote = line.match(/^(\s*>+\s?)(.*)$/);
      if (quote) {
        return `<span class="src-quote-marker">${escapeHtml(quote[1])}</span><span class="src-quote">${highlightInline(quote[2])}</span>`;
      }

      const list = line.match(/^(\s*(?:[-*+]|\d+\.)\s+)(.*)$/);
      if (list) {
        return `<span class="src-list-marker">${escapeHtml(list[1])}</span>${highlightInline(list[2])}`;
      }

      if (/^\s*\|.*\|\s*$/.test(line)) {
        return `<span class="src-table">${highlightInline(line)}</span>`;
      }

      if (/^\s*(-{3,}|\*{3,}|_{3,})\s*$/.test(line)) {
        return `<span class="src-hr">${escapeHtml(line)}</span>`;
      }

      return highlightInline(line);
    })
    .join("\n");
}

const highlighted = computed(() => highlightSource(props.modelValue));

function onInput(event: Event) {
  emit("update:modelValue", (event.target as HTMLTextAreaElement).value);
}

function emitRendered() {
  nextTick(() => {
    if (root.value) emit("rendered", root.value);
  });
}

onMounted(emitRendered);
watch(() => props.currentFile, emitRendered, { flush: "post" });

defineExpose({ root });
</script>

<template>
  <div class="source-wrap">
    <div class="source-shell">
      <pre ref="root" class="source-highlight" aria-hidden="true" v-html="highlighted"></pre>
      <textarea
        class="source-input"
        :value="modelValue"
        spellcheck="false"
        @input="onInput"
      ></textarea>
    </div>
  </div>
</template>

<style scoped>
.source-wrap {
  padding: 32px 48px 80px;
  max-width: var(--reader-max-width, 1040px);
  margin: 0 auto;
}
.source-shell {
  display: grid;
  min-height: calc(100vh - 150px);
  border: 1px solid var(--border);
  border-radius: 10px;
  background: var(--source-bg, #fbfdff);
  overflow: hidden;
  box-shadow: 0 1px 2px rgba(15, 23, 42, 0.04);
}
.source-highlight,
.source-input {
  grid-area: 1 / 1;
  width: 100%;
  min-height: 100%;
  margin: 0;
  padding: 18px 20px;
  border: 0;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
  tab-size: 2;
  font-family: ui-monospace, SFMono-Regular, "Cascadia Code", Consolas,
    "Microsoft YaHei UI", monospace;
  font-size: var(--reader-font-size, 16px);
  line-height: var(--reader-line-height, 1.75);
}
.source-highlight {
  pointer-events: none;
  color: var(--source-text, #263244);
}
.source-input {
  resize: none;
  outline: none;
  overflow: hidden;
  background: transparent;
  color: transparent;
  caret-color: var(--source-caret, #0969da);
  -webkit-text-fill-color: transparent;
}
.source-input::selection {
  background: rgba(9, 105, 218, 0.18);
}
:deep(.src-marker),
:deep(.src-list-marker),
:deep(.src-quote-marker) {
  color: var(--source-marker, #64748b);
}
:deep(.src-heading) {
  color: var(--source-heading, #0b63b6);
  font-weight: 700;
}
:deep(.src-code),
:deep(.src-codeblock),
:deep(.src-fence) {
  color: var(--source-code, #7c3aed);
}
:deep(.src-strong) {
  color: var(--source-strong, #b42318);
  font-weight: 700;
}
:deep(.src-link) {
  color: var(--source-link, #0f766e);
}
:deep(.src-url) {
  color: var(--source-url, #64748b);
}
:deep(.src-quote) {
  color: var(--source-quote, #218044);
}
:deep(.src-table) {
  color: var(--source-table, #0b4f99);
}
:deep(.src-hr) {
  color: var(--source-marker, #64748b);
}
</style>
