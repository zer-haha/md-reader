<script setup lang="ts">
import { Check, RotateCcw, X } from "@lucide/vue";
import { useI18n } from "vue-i18n";
import { useReadingSettings } from "../composables/useReadingSettings";

const { t } = useI18n();

defineProps<{
  visible: boolean;
  tabMode: boolean;
}>();

const emit = defineEmits<{
  (e: "close"): void;
  (e: "update:tabMode", value: boolean): void;
}>();

const {
  settings,
  fontOptions,
  setFontSize,
  setLineHeight,
  setMaxWidth,
  setFontFamily,
  reset,
} = useReadingSettings();
</script>

<template>
  <div v-if="visible" class="overlay" @click.self="emit('close')">
    <section class="dialog" role="dialog" aria-modal="true">
      <header class="dialog-head">
        <div>
          <div class="eyebrow">阅读器设置</div>
          <h2>{{ t("settings.title") }}</h2>
        </div>
        <button class="icon-btn" @click="emit('close')" :title="t('find.close')">
          <X :size="18" :stroke-width="2" />
        </button>
      </header>

      <div class="content">
        <section class="section">
          <div class="section-title">打开方式</div>
          <button
            class="switch-row"
            :class="{ active: tabMode }"
            role="switch"
            :aria-checked="tabMode"
            @click="emit('update:tabMode', !tabMode)"
          >
            <span class="setting-copy">
              <strong>{{ t("settings.tabMode") }}</strong>
              <small>打开多个 Markdown 文件时，用标签页集中管理。</small>
            </span>
            <span class="switch">
              <span></span>
            </span>
          </button>
        </section>

        <section class="section">
          <div class="section-title">正文显示</div>

          <label class="control-row">
            <span class="setting-copy">
              <strong>{{ t("settings.fontSize") }}</strong>
              <small>控制正文阅读大小。</small>
            </span>
            <input
              class="range"
              type="range"
              :value="settings.fontSize"
              min="12"
              max="24"
              step="1"
              @input="(e) => setFontSize(Number((e.target as HTMLInputElement).value))"
            />
            <output>{{ settings.fontSize }}px</output>
          </label>

          <label class="control-row">
            <span class="setting-copy">
              <strong>{{ t("settings.lineHeight") }}</strong>
              <small>行距越大，长文阅读越轻松。</small>
            </span>
            <input
              class="range"
              type="range"
              :value="settings.lineHeight"
              min="1.3"
              max="2.2"
              step="0.05"
              @input="(e) => setLineHeight(Number((e.target as HTMLInputElement).value))"
            />
            <output>{{ settings.lineHeight.toFixed(2) }}</output>
          </label>

          <label class="control-row">
            <span class="setting-copy">
              <strong>{{ t("settings.maxWidth") }}</strong>
              <small>控制正文区域宽度，表格多时可以调宽。</small>
            </span>
            <input
              class="range"
              type="range"
              :value="settings.maxWidth"
              min="640"
              max="1400"
              step="20"
              @input="(e) => setMaxWidth(Number((e.target as HTMLInputElement).value))"
            />
            <output>{{ settings.maxWidth }}px</output>
          </label>

          <label class="control-row select-row">
            <span class="setting-copy">
              <strong>{{ t("settings.fontFamily") }}</strong>
              <small>切换正文使用的字体风格。</small>
            </span>
            <select
              class="select-control"
              :value="settings.fontFamily"
              @change="(e) => setFontFamily((e.target as HTMLSelectElement).value)"
            >
              <option
                v-for="opt in fontOptions"
                :key="opt.value"
                :value="opt.value"
              >
                {{ opt.label }}
              </option>
            </select>
          </label>
        </section>
      </div>

      <footer class="footer">
        <button class="dialog-btn" @click="reset">
          <RotateCcw :size="15" :stroke-width="2" />
          <span>{{ t("settings.reset") }}</span>
        </button>
        <button class="dialog-btn primary" @click="emit('close')">
          <Check :size="15" :stroke-width="2" />
          <span>{{ t("settings.done") }}</span>
        </button>
      </footer>
    </section>
  </div>
</template>

<style scoped>
.overlay {
  position: fixed;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
  background: rgba(15, 23, 42, 0.38);
  z-index: 50;
}
.dialog {
  width: min(640px, calc(100vw - 32px));
  max-height: min(760px, calc(100vh - 48px));
  display: flex;
  flex-direction: column;
  background: var(--bg);
  border: 1px solid var(--border);
  border-radius: 12px;
  color: var(--fg);
  box-shadow: 0 24px 60px rgba(15, 23, 42, 0.26);
  overflow: hidden;
}
.dialog-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 20px 22px 16px;
  border-bottom: 1px solid var(--border);
  background: var(--bg-toolbar);
}
.eyebrow {
  margin-bottom: 4px;
  color: var(--fg-muted);
  font-size: 12px;
}
h2 {
  margin: 0;
  font-size: 18px;
  font-weight: 650;
}
.icon-btn {
  width: 32px;
  height: 32px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--bg-btn);
  color: var(--fg);
  cursor: pointer;
}
.icon-btn:hover {
  background: var(--bg-btn-hover);
}
.content {
  padding: 18px 22px;
  overflow: auto;
}
.section + .section {
  margin-top: 18px;
}
.section-title {
  margin-bottom: 10px;
  color: var(--fg-muted);
  font-size: 12px;
  font-weight: 650;
}
.switch-row,
.control-row {
  width: 100%;
  display: grid;
  grid-template-columns: minmax(190px, 1fr) minmax(180px, 240px) 74px;
  align-items: center;
  gap: 16px;
  padding: 13px 14px;
  border: 1px solid var(--border);
  border-radius: 10px;
  background: var(--bg-btn);
}
.control-row + .control-row {
  margin-top: 10px;
}
.switch-row {
  grid-template-columns: 1fr auto;
  text-align: left;
  cursor: pointer;
}
.switch-row:hover,
.control-row:hover {
  border-color: color-mix(in srgb, var(--link) 32%, var(--border));
}
.setting-copy {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.setting-copy strong {
  font-size: 13px;
  font-weight: 650;
}
.setting-copy small {
  color: var(--fg-muted);
  font-size: 12px;
  line-height: 1.45;
}
.switch {
  width: 42px;
  height: 24px;
  display: inline-flex;
  align-items: center;
  padding: 2px;
  border-radius: 999px;
  background: var(--border);
  transition: background-color 0.16s;
}
.switch span {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #fff;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.25);
  transition: transform 0.16s;
}
.switch-row.active .switch {
  background: var(--link);
}
.switch-row.active .switch span {
  transform: translateX(18px);
}
.range {
  width: 100%;
  accent-color: var(--link);
}
output {
  color: var(--fg-muted);
  font-size: 12px;
  font-variant-numeric: tabular-nums;
  text-align: right;
}
.select-row {
  grid-template-columns: minmax(190px, 1fr) minmax(180px, 314px);
}
.select-control {
  width: 100%;
  height: 34px;
  padding: 0 10px;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--bg);
  color: var(--fg);
}
.footer {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  padding: 14px 22px 18px;
  border-top: 1px solid var(--border);
  background: var(--bg-toolbar);
}
.dialog-btn {
  height: 32px;
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 0 13px;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: var(--bg-btn);
  color: var(--fg);
  cursor: pointer;
}
.dialog-btn:hover {
  background: var(--bg-btn-hover);
}
.dialog-btn.primary {
  border-color: var(--link);
  background: var(--link);
  color: #fff;
}

@media (max-width: 720px) {
  .control-row,
  .select-row {
    grid-template-columns: 1fr;
    gap: 10px;
  }
  output {
    text-align: left;
  }
  .footer {
    justify-content: stretch;
  }
  .dialog-btn {
    flex: 1;
    justify-content: center;
  }
}
</style>
