<script setup lang="ts">
import { ref, watch, nextTick, onMounted } from "vue";
import { useI18n } from "vue-i18n";
import { ChevronDown, ChevronUp, X } from "@lucide/vue";

const { t } = useI18n();

const props = defineProps<{
  visible: boolean;
  query: string;
  caseSensitive: boolean;
  total: number;
  activeIndex: number;
}>();

const emit = defineEmits<{
  (e: "update:query", v: string): void;
  (e: "update:caseSensitive", v: boolean): void;
  (e: "search"): void;
  (e: "next"): void;
  (e: "prev"): void;
  (e: "close"): void;
}>();

const inputRef = ref<HTMLInputElement | null>(null);

function onInput(evt: Event) {
  emit("update:query", (evt.target as HTMLInputElement).value);
  emit("search");
}

function onKey(evt: KeyboardEvent) {
  if (evt.key === "Enter") {
    evt.preventDefault();
    if (evt.shiftKey) emit("prev");
    else emit("next");
  } else if (evt.key === "Escape") {
    evt.preventDefault();
    emit("close");
  }
}

function toggleCase() {
  emit("update:caseSensitive", !props.caseSensitive);
  emit("search");
}

watch(
  () => props.visible,
  async (v) => {
    if (v) {
      await nextTick();
      inputRef.value?.focus();
      inputRef.value?.select();
    }
  }
);

onMounted(() => {
  if (props.visible) inputRef.value?.focus();
});
</script>

<template>
  <div v-if="visible" class="find-bar">
    <input
      ref="inputRef"
      type="text"
      class="find-input"
      :placeholder="t('find.placeholder')"
      :value="query"
      @input="onInput"
      @keydown="onKey"
    />
    <span class="counter">
      {{ total === 0 ? "0/0" : `${activeIndex + 1}/${total}` }}
    </span>
    <button
      class="ic"
      :class="{ active: caseSensitive }"
      @click="toggleCase"
      :title="t('find.caseSensitive')"
    >
      Aa
    </button>
    <button class="ic" @click="emit('prev')" :title="t('find.previous') + ' (Shift+Enter)'">
      <ChevronUp :size="15" :stroke-width="2" />
    </button>
    <button class="ic" @click="emit('next')" :title="t('find.next') + ' (Enter)'">
      <ChevronDown :size="15" :stroke-width="2" />
    </button>
    <button class="ic" @click="emit('close')" :title="t('find.close') + ' (Esc)'">
      <X :size="15" :stroke-width="2" />
    </button>
  </div>
</template>

<style scoped>
.find-bar {
  position: sticky;
  top: 10px;
  width: fit-content;
  margin: 10px 16px -48px auto;
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 8px;
  background: color-mix(in srgb, var(--bg-toolbar) 94%, #ffffff);
  border: 1px solid var(--border);
  border-radius: 8px;
  box-shadow: 0 8px 22px rgba(15, 23, 42, 0.18);
  z-index: 35;
}
.find-input {
  width: 230px;
  height: 28px;
  padding: 0 9px;
  font-size: 13px;
  background: var(--bg);
  color: var(--fg);
  border: 1px solid var(--border);
  border-radius: 6px;
  outline: none;
}
.find-input:focus {
  border-color: var(--link);
}
.counter {
  font-size: 12px;
  color: var(--fg-muted);
  min-width: 44px;
  text-align: center;
}
.ic {
  width: 28px;
  height: 28px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  padding: 0;
  background: transparent;
  color: var(--fg);
  border: 1px solid transparent;
  border-radius: 6px;
  cursor: pointer;
}
.ic:hover {
  background: var(--bg-btn-hover);
}
.ic.active {
  background: var(--bg-active);
  color: var(--link);
}
</style>
