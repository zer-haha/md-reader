<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed, watch, nextTick } from "vue";
import { open } from "@tauri-apps/plugin-dialog";
import { readTextFile, stat, writeTextFile } from "@tauri-apps/plugin-fs";
import { getCurrentWebview } from "@tauri-apps/api/webview";
import { getCurrentWindow } from "@tauri-apps/api/window";
import {
  Copy,
  Code2,
  Download,
  Eye,
  FileDown,
  FileText,
  FolderOpen,
  List,
  LoaderCircle,
  Moon,
  PanelLeft,
  Printer,
  RefreshCw,
  Search,
  Settings,
  Save,
  Sun,
  X,
  Plus,
} from "@lucide/vue";
import { useI18n } from "vue-i18n";
import { persistLocale, type AppLocale } from "./i18n";
import MarkdownView from "./components/MarkdownView.vue";
import SourceEditor from "./components/SourceEditor.vue";
import FileTree from "./components/FileTree.vue";
import TocPanel from "./components/TocPanel.vue";
import FindBar from "./components/FindBar.vue";
import SearchPanel from "./components/SearchPanel.vue";
import SettingsDialog from "./components/SettingsDialog.vue";
import { useFileTree } from "./composables/useFileTree";
import { useFileWatcher } from "./composables/useFileWatcher";
import { extractHeadings, type Heading } from "./composables/useMarkdown";
import { useResizable } from "./composables/useResizable";
import { useScrollSpy } from "./composables/useScrollSpy";
import { useFindInPage } from "./composables/useFindInPage";
import { useHistory } from "./composables/useHistory";
import { useReadingSettings } from "./composables/useReadingSettings";
import {
  restoreWindowState,
  watchWindowState,
} from "./composables/useWindowState";
import {
  exportToHtml,
  exportToTxt,
  exportToPdf,
  checkPdfEngine,
  printDocument,
} from "./composables/useExport";

interface FileTab {
  path: string;
  name: string;
  content: string;
  headings: Heading[];
  errorMsg: string;
  loaded: boolean;
  dirty: boolean;
}

interface ContextMenuState {
  visible: boolean;
  x: number;
  y: number;
}

const TABS_STORAGE = "md-reader-tabs";
const ACTIVE_TAB_STORAGE = "md-reader-active-tab";
const TAB_MODE_STORAGE = "md-reader-tab-mode";

function basename(path: string): string {
  const parts = path.split(/[\\/]/);
  return parts[parts.length - 1] || path;
}

function loadSavedTabs(): FileTab[] {
  try {
    const raw = localStorage.getItem(TABS_STORAGE);
    const paths = raw ? (JSON.parse(raw) as unknown) : [];
    if (!Array.isArray(paths)) return [];
    return Array.from(
      new Set(paths.filter((p): p is string => typeof p === "string" && !!p))
    ).map((path) => ({
        path,
        name: basename(path),
        content: "",
        headings: [],
        errorMsg: "",
        loaded: false,
        dirty: false,
      }));
  } catch {
    return [];
  }
}

const { t, locale } = useI18n();

function toggleLocale() {
  const next = locale.value === "zh-CN" ? "en-US" : "zh-CN";
  locale.value = next;
  persistLocale(next as AppLocale);
}

const {
  rootDir,
  tree,
  loading: treeLoading,
  error: treeError,
  refresh: refreshTree,
  openFolder,
  restoreRoot,
  clearRoot,
} = useFileTree();

const watcher = useFileWatcher();
const { pushRecent, saveScroll, getScroll } = useHistory();
const { apply: applyReadingSettings } = useReadingSettings();

const content = ref<string>("");
const currentFile = ref<string>("");
const errorMsg = ref<string>("");
const headings = ref<Heading[]>([]);
const tabs = ref<FileTab[]>(loadSavedTabs());
const activeTabPath = ref(localStorage.getItem(ACTIVE_TAB_STORAGE) || "");
const tabMode = ref(localStorage.getItem(TAB_MODE_STORAGE) !== "0");
type ViewMode = "preview" | "source";

const viewMode = ref<ViewMode>("preview");
const standaloneDirty = ref(false);

const theme = ref<"light" | "dark">(
  (localStorage.getItem("md-reader-theme") as "light" | "dark") || "light"
);
const showFileTree = ref<boolean>(
  localStorage.getItem("md-reader-show-tree") !== "0"
);
const showToc = ref<boolean>(
  localStorage.getItem("md-reader-show-toc") !== "0"
);
const showSettings = ref(false);
const leftMode = ref<"files" | "search">("files");
const showExportMenu = ref(false);
const exportBusy = ref(false);
const exportToast = ref("");
const pdfEnginePath = ref<string | null>(null);
const renderTick = ref(0);
const fileSize = ref<number | null>(null);
const selectedText = ref("");
const contextMenu = ref<ContextMenuState>({
  visible: false,
  x: 0,
  y: 0,
});

const { width: leftWidth, startResize: resizeLeft } = useResizable(
  "md-reader-left-w",
  260
);
const { width: rightWidth, startResize: resizeRight } = useResizable(
  "md-reader-right-w",
  240,
  { inverse: true }
);

const viewerEl = ref<HTMLElement | null>(null);
const markdownRef = ref<{ root: HTMLElement | null } | null>(null);
const sourceRef = ref<{ root: HTMLElement | null } | null>(null);
const markdownBodyRef = computed(() => markdownRef.value?.root ?? null);
const bodyRef = computed(() =>
  viewMode.value === "source"
    ? sourceRef.value?.root ?? null
    : markdownBodyRef.value
);

const { activeId, onScroll, jumpTo } = useScrollSpy(viewerEl, bodyRef);
const find = useFindInPage(bodyRef);

interface ViewScrollState {
  preview: number;
  source: number;
  previewReady: boolean;
  sourceReady: boolean;
}

const viewScrollByFile = new Map<string, ViewScrollState>();

function getViewScrollState(path = currentFile.value): ViewScrollState {
  const key = path || "__empty__";
  let state = viewScrollByFile.get(key);
  if (!state) {
    state = {
      preview: 0,
      source: 0,
      previewReady: false,
      sourceReady: false,
    };
    viewScrollByFile.set(key, state);
  }
  return state;
}

function rememberModeScroll(mode: ViewMode = viewMode.value) {
  if (!currentFile.value || !viewerEl.value) return;
  const state = getViewScrollState();
  if (mode === "preview") {
    state.preview = viewerEl.value.scrollTop;
    state.previewReady = true;
  } else {
    state.source = viewerEl.value.scrollTop;
    state.sourceReady = true;
  }
}

function rememberInitialPreviewScroll(path: string, top: number) {
  const state = getViewScrollState(path);
  state.preview = top;
  state.previewReady = top > 0;
  state.source = 0;
  state.sourceReady = false;
}

function getRememberedModeTop(mode: ViewMode) {
  const state = getViewScrollState();
  if (mode === "preview" && state.previewReady) return state.preview;
  if (mode === "source" && state.sourceReady) return state.source;
  return null;
}

const fileName = computed(() => {
  if (!currentFile.value) return t("app.noFile");
  const parts = currentFile.value.split(/[\\/]/);
  return parts[parts.length - 1];
});

const fileSizeText = computed(() =>
  fileSize.value == null ? "大小 -" : `大小 ${formatBytes(fileSize.value)}`
);

const documentCharText = computed(() =>
  content.value ? `字符 ${content.value.length.toLocaleString()}` : "字符 -"
);

const currentDirty = computed(() => {
  if (!currentFile.value) return false;
  if (tabMode.value) {
    return !!tabs.value.find((tab) => tab.path === currentFile.value)?.dirty;
  }
  return standaloneDirty.value;
});

function formatBytes(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  const kb = bytes / 1024;
  if (kb < 1024) return `${kb.toFixed(kb < 10 ? 1 : 0)} KB`;
  const mb = kb / 1024;
  return `${mb.toFixed(mb < 10 ? 1 : 0)} MB`;
}

async function updateFileInfo(path: string, fallbackText = "") {
  if (!path) {
    fileSize.value = null;
    return;
  }
  try {
    const info = await stat(path);
    fileSize.value = info.size;
  } catch {
    fileSize.value = fallbackText ? new Blob([fallbackText]).size : null;
  }
}

function closeContextMenu() {
  contextMenu.value.visible = false;
}

function openContextMenu(event: MouseEvent) {
  event.preventDefault();
  showExportMenu.value = false;
  selectedText.value = window.getSelection()?.toString().trim() || "";
  const menuWidth = 250;
  const menuHeight = 330;
  const margin = 8;
  contextMenu.value = {
    visible: true,
    x: Math.max(
      margin,
      Math.min(event.clientX, window.innerWidth - menuWidth - margin)
    ),
    y: Math.max(
      margin,
      Math.min(event.clientY, window.innerHeight - menuHeight - margin)
    ),
  };
}

async function runContextAction(action: () => void | Promise<void>) {
  closeContextMenu();
  await action();
}

async function copyText(text: string) {
  if (!text) return;
  try {
    await navigator.clipboard.writeText(text);
  } catch {
    const textarea = document.createElement("textarea");
    textarea.value = text;
    textarea.style.position = "fixed";
    textarea.style.opacity = "0";
    document.body.appendChild(textarea);
    textarea.select();
    document.execCommand("copy");
    document.body.removeChild(textarea);
  }
  exportToast.value = "已复制";
}

async function copySelectedText() {
  await copyText(selectedText.value);
}

async function copyCurrentFilePath() {
  await copyText(currentFile.value);
}

async function reloadCurrentFile() {
  if (currentFile.value) await loadFile(currentFile.value, "", true);
}

function setCurrentDirty(dirty: boolean) {
  standaloneDirty.value = dirty;
  if (tabMode.value && currentFile.value) {
    const tab = tabs.value.find((item) => item.path === currentFile.value);
    if (tab) tab.dirty = dirty;
  }
}

function syncActiveTab(nextHeadings = headings.value) {
  if (!tabMode.value || !currentFile.value) return;
  const tab = tabs.value.find((item) => item.path === currentFile.value);
  if (!tab) return;
  tab.content = content.value;
  tab.headings = nextHeadings;
  tab.errorMsg = errorMsg.value;
  tab.loaded = true;
}

let headingUpdateTimer: number | null = null;
function updateHeadingsSoon(text: string) {
  if (headingUpdateTimer) clearTimeout(headingUpdateTimer);
  headingUpdateTimer = window.setTimeout(() => {
    const nextHeadings = extractHeadings(text);
    headings.value = nextHeadings;
    syncActiveTab(nextHeadings);
  }, 180);
}

function onSourceInput(text: string) {
  content.value = text;
  errorMsg.value = "";
  setCurrentDirty(true);
  syncActiveTab();
  updateHeadingsSoon(text);
}

async function saveCurrentFile() {
  if (!currentFile.value) return;
  try {
    await writeTextFile(currentFile.value, content.value);
    setCurrentDirty(false);
    syncActiveTab();
    await updateFileInfo(currentFile.value, content.value);
    exportToast.value = "已保存";
  } catch (e: any) {
    errorMsg.value = `保存失败: ${e?.message || e}`;
  }
}

function restoreViewerTop(top: number) {
  nextTick(() => {
    requestAnimationFrame(() => {
      const el = viewerEl.value;
      if (!el) return;
      const maxTop = Math.max(0, el.scrollHeight - el.clientHeight);
      el.scrollTop = Math.min(top, maxTop);
      onScroll();
    });
  });
}

function getViewerRatio() {
  const el = viewerEl.value;
  if (!el) return 0;
  const maxTop = Math.max(0, el.scrollHeight - el.clientHeight);
  return maxTop > 0 ? el.scrollTop / maxTop : 0;
}

function restoreViewerRatio(ratio: number) {
  nextTick(() => {
    requestAnimationFrame(() => {
      const el = viewerEl.value;
      if (!el) return;
      const maxTop = Math.max(0, el.scrollHeight - el.clientHeight);
      el.scrollTop = Math.max(0, Math.min(maxTop, maxTop * ratio));
      onScroll();
    });
  });
}

function getPreviewHeadingNearTop() {
  const viewer = viewerEl.value;
  const body = markdownBodyRef.value;
  if (!viewer || !body) return activeId.value;
  const viewerTop = viewer.getBoundingClientRect().top;
  const nodes = Array.from(
    body.querySelectorAll<HTMLElement>("h1, h2, h3, h4, h5, h6")
  );
  let current = "";
  for (const node of nodes) {
    if (node.getBoundingClientRect().top - viewerTop < 130) {
      current = node.id;
    } else {
      break;
    }
  }
  return current || activeId.value;
}

function getSourceHeadingLines() {
  const lines = content.value.split(/\r?\n/);
  const result: Array<{ id: string; line: number }> = [];
  let headingIndex = 0;
  lines.forEach((line, index) => {
    const match = line.match(/^(#{1,6})\s+(.+?)\s*#*\s*$/);
    if (!match) return;
    const level = match[1].length;
    while (
      headingIndex < headings.value.length &&
      headings.value[headingIndex].level !== level
    ) {
      headingIndex++;
    }
    const heading = headings.value[headingIndex];
    if (heading) {
      result.push({ id: heading.id, line: index });
      headingIndex++;
    }
  });
  return result;
}

function getSourceLineHeight(root: HTMLElement) {
  const style = getComputedStyle(root);
  const parsed = Number.parseFloat(style.lineHeight);
  if (Number.isFinite(parsed)) return parsed;
  const fontSize = Number.parseFloat(style.fontSize) || 16;
  return fontSize * 1.75;
}

function getElementTopInViewer(el: HTMLElement, viewer: HTMLElement) {
  return (
    el.getBoundingClientRect().top -
    viewer.getBoundingClientRect().top +
    viewer.scrollTop
  );
}

function getSourceHeadingNearTop() {
  const viewer = viewerEl.value;
  const root = sourceRef.value?.root;
  if (!viewer || !root) return "";
  const lineHeight = getSourceLineHeight(root);
  const rootTop = getElementTopInViewer(root, viewer);
  const line = Math.max(
    0,
    Math.floor((viewer.scrollTop - rootTop + 16) / lineHeight)
  );
  let current = "";
  for (const item of getSourceHeadingLines()) {
    if (item.line <= line) current = item.id;
    else break;
  }
  return current;
}

function scrollPreviewToHeading(id: string, fallbackRatio: number) {
  nextTick(() => {
    requestAnimationFrame(() => {
      const viewer = viewerEl.value;
      const body = markdownBodyRef.value;
      const target = id ? body?.querySelector<HTMLElement>(`#${CSS.escape(id)}`) : null;
      if (!viewer || !body || !target) {
        restoreViewerRatio(fallbackRatio);
        return;
      }
      const offset =
        target.getBoundingClientRect().top -
        viewer.getBoundingClientRect().top +
        viewer.scrollTop -
        12;
      viewer.scrollTop = Math.max(0, offset);
      onScroll();
    });
  });
}

function scrollSourceToHeading(id: string, fallbackRatio: number) {
  nextTick(() => {
    requestAnimationFrame(() => {
      const viewer = viewerEl.value;
      const root = sourceRef.value?.root;
      const target = getSourceHeadingLines().find((item) => item.id === id);
      if (!viewer || !root || !target) {
        restoreViewerRatio(fallbackRatio);
        return;
      }
      viewer.scrollTop = Math.max(
        0,
        getElementTopInViewer(root, viewer) +
          target.line * getSourceLineHeight(root) -
          18
      );
      onScroll();
    });
  });
}

function setViewMode(mode: ViewMode) {
  if (viewMode.value === mode) return;
  rememberModeScroll();
  const ratio = getViewerRatio();
  const headingId =
    viewMode.value === "source"
      ? getSourceHeadingNearTop()
      : getPreviewHeadingNearTop();
  find.clearHighlights();
  viewMode.value = mode;
  const rememberedTop = getRememberedModeTop(mode);
  if (rememberedTop != null) {
    restoreViewerTop(rememberedTop);
  } else if (mode === "source") {
    scrollSourceToHeading(headingId, ratio);
  } else {
    scrollPreviewToHeading(headingId, ratio);
  }
}

async function getMarkdownBodyForAction(): Promise<HTMLElement | null> {
  if (viewMode.value === "source") {
    setViewMode("preview");
  }
  for (let i = 0; i < 8; i++) {
    await nextTick();
    await new Promise<void>((resolve) => requestAnimationFrame(() => resolve()));
    if (markdownBodyRef.value) return markdownBodyRef.value;
  }
  return null;
}

let pendingScrollTop = 0;
let pendingHash = "";
let shouldRestoreFileScroll = false;

function persistTabs() {
  localStorage.setItem(
    TABS_STORAGE,
    JSON.stringify(tabs.value.map((tab) => tab.path))
  );
  if (activeTabPath.value) {
    localStorage.setItem(ACTIVE_TAB_STORAGE, activeTabPath.value);
  } else {
    localStorage.removeItem(ACTIVE_TAB_STORAGE);
  }
}

function ensureTab(path: string): FileTab {
  let tab = tabs.value.find((item) => item.path === path);
  if (!tab) {
    tab = {
      path,
      name: basename(path),
      content: "",
      headings: [],
      errorMsg: "",
      loaded: false,
      dirty: false,
    };
    tabs.value.push(tab);
    persistTabs();
  }
  return tab;
}

function activateTab(tab: FileTab, hash = "") {
  activeTabPath.value = tab.path;
  currentFile.value = tab.path;
  content.value = tab.content;
  errorMsg.value = tab.errorMsg;
  headings.value = tab.headings;
  standaloneDirty.value = tab.dirty;
  pendingHash = hash;
  pendingScrollTop = hash ? 0 : getScroll(tab.path);
  shouldRestoreFileScroll = true;
  rememberInitialPreviewScroll(tab.path, pendingScrollTop);
  find.clearHighlights();
  persistTabs();
  void updateFileInfo(tab.path, tab.content);
  if (viewMode.value === "source") void nextTick(() => onRendered("source"));
}

async function switchTab(path: string) {
  const tab = tabs.value.find((item) => item.path === path);
  if (!tab || tab.path === activeTabPath.value) return;
  saveCurrentScroll();
  if (tab.loaded) {
    activateTab(tab);
    return;
  }
  await loadFile(path, "", true);
}

async function closeTab(path: string) {
  const index = tabs.value.findIndex((tab) => tab.path === path);
  if (index < 0) return;
  const wasActive = activeTabPath.value === path;
  if (wasActive) saveCurrentScroll();
  tabs.value.splice(index, 1);
  if (!wasActive) {
    persistTabs();
    return;
  }
  const next = tabs.value[index] || tabs.value[index - 1];
  if (next) {
    activeTabPath.value = "";
    await switchTab(next.path);
    } else {
      activeTabPath.value = "";
      currentFile.value = "";
      content.value = "";
      errorMsg.value = "";
      headings.value = [];
      fileSize.value = null;
      standaloneDirty.value = false;
      find.clearHighlights();
      persistTabs();
    }
}

function setTabMode(enabled: boolean) {
  tabMode.value = enabled;
  localStorage.setItem(TAB_MODE_STORAGE, enabled ? "1" : "0");
  if (enabled && currentFile.value) {
    const tab = ensureTab(currentFile.value);
    tab.content = content.value;
    tab.headings = headings.value;
    tab.errorMsg = errorMsg.value;
    tab.loaded = !!content.value;
    tab.dirty = standaloneDirty.value;
    activateTab(tab);
  }
}

async function loadFile(path: string, hash = "", forceReload = false) {
  if (currentFile.value === path && hash && !forceReload) {
    pendingHash = hash;
    jumpTo(hash);
    return;
  }
  if (tabMode.value && !forceReload) {
    const existing = tabs.value.find((tab) => tab.path === path);
    if (existing?.loaded) {
      saveCurrentScroll();
      activateTab(existing, hash);
      return;
    }
  }
  try {
    saveCurrentScroll();
    const text = await readTextFile(path);
    const nextHeadings = extractHeadings(text);
    if (tabMode.value) {
      const tab = ensureTab(path);
      tab.content = text;
      tab.headings = nextHeadings;
      tab.errorMsg = "";
      tab.loaded = true;
      tab.dirty = false;
      activateTab(tab, hash);
    } else {
      content.value = text;
      currentFile.value = path;
      errorMsg.value = "";
      headings.value = nextHeadings;
      standaloneDirty.value = false;
      pendingHash = hash;
      pendingScrollTop = hash ? 0 : getScroll(path);
      shouldRestoreFileScroll = true;
      rememberInitialPreviewScroll(path, pendingScrollTop);
      find.clearHighlights();
      if (viewMode.value === "source") void nextTick(onRendered);
    }
    pushRecent(path);
    void updateFileInfo(path, text);
  } catch (e: any) {
    const msg = `${t("errors.readFailed")}: ${e?.message || e}`;
    if (tabMode.value) {
      const tab = ensureTab(path);
      tab.content = "";
      tab.headings = [];
      tab.errorMsg = msg;
      tab.loaded = false;
      tab.dirty = false;
      activateTab(tab, hash);
    } else {
      currentFile.value = path;
      content.value = "";
      headings.value = [];
      errorMsg.value = msg;
      standaloneDirty.value = false;
    }
    fileSize.value = null;
  }
}

function onRendered(mode: ViewMode = viewMode.value) {
  nextTick(() => {
    if (!viewerEl.value) return;
    if (mode !== viewMode.value) return;
    if (!shouldRestoreFileScroll) {
      onScroll();
      return;
    }
    if (pendingHash) {
      jumpTo(pendingHash);
      pendingHash = "";
    } else if (pendingScrollTop > 0) {
      viewerEl.value.scrollTop = pendingScrollTop;
    } else {
      viewerEl.value.scrollTop = 0;
    }
    shouldRestoreFileScroll = false;
    rememberModeScroll();
    onScroll();
  });
}

function saveCurrentScroll() {
  rememberModeScroll();
  if (currentFile.value && viewerEl.value) {
    if (viewMode.value === "preview") {
      saveScroll(currentFile.value, viewerEl.value.scrollTop);
    }
  }
}

async function pickFile() {
  const selected = await open({
    multiple: false,
    filters: [
      { name: "Markdown", extensions: ["md", "markdown", "mdx", "txt"] },
    ],
  });
  if (typeof selected === "string") await loadFile(selected);
}

async function pickFolder() {
  const dir = await openFolder();
  if (dir) await startWatching(dir);
}

async function startWatching(dir: string) {
  await watcher.start(dir, async (paths) => {
    await refreshTree();
    if (currentFile.value && paths.includes(currentFile.value)) {
      if (currentDirty.value) {
        exportToast.value = "文件已变化，当前有未保存修改";
      } else {
        await loadFile(currentFile.value, "", true);
      }
    }
  });
}

function closeFolder() {
  void watcher.stop();
  clearRoot();
}

function toggleTheme() {
  const top = viewerEl.value?.scrollTop ?? 0;
  theme.value = theme.value === "light" ? "dark" : "light";
  localStorage.setItem("md-reader-theme", theme.value);
  applyTheme();
  // Force Mermaid/KaTeX re-render so charts follow the new theme.
  renderTick.value++;
  restoreViewerTop(top);
}

function applyTheme() {
  document.documentElement.dataset.theme = theme.value;
  try {
    getCurrentWindow().setTheme(theme.value === "dark" ? "dark" : "light");
  } catch {
    /* ignore in non-Tauri environments */
  }
}

async function exportHtml() {
  const body = await getMarkdownBodyForAction();
  if (!body || !content.value) return;
  try {
    await exportToHtml(body, fileName.value || "document.html");
  } catch (e: any) {
    errorMsg.value = `${t("export.exportFailed")}: ${e?.message ?? e}`;
  }
}

async function exportTxt() {
  showExportMenu.value = false;
  if (!content.value) return;
  exportBusy.value = true;
  exportToast.value = t("export.generatingTxt");
  try {
    const out = await exportToTxt(content.value, fileName.value || "document");
    if (out) exportToast.value = `${t("export.exportedTxt")}: ${out}`;
    else exportToast.value = "";
  } catch (e: any) {
    errorMsg.value = `${t("export.txtFailed")}: ${e?.message ?? e}`;
    exportToast.value = "";
  } finally {
    exportBusy.value = false;
  }
}

async function exportPdf() {
  const body = await getMarkdownBodyForAction();
  if (!body || !content.value) return;
  showExportMenu.value = false;
  exportBusy.value = true;
  exportToast.value = t("export.generatingPdf");
  try {
    const result = await exportToPdf(
      body,
      fileName.value || "document",
      fileName.value,
      async () => {
        const picked = await open({
          title: t("export.chooseEdgePath"),
          multiple: false,
          filters: [
            { name: "Edge / Chrome", extensions: ["exe"] },
            { name: t("app.allFiles"), extensions: ["*"] },
          ],
        });
        return typeof picked === "string" ? picked : null;
      }
    );
    if (result) {
      pdfEnginePath.value = result.edge_path;
      const sec = (result.elapsed_ms / 1000).toFixed(1);
      exportToast.value = `${t("export.exportedPdf")} (${sec}s): ${result.out_path}`;
    } else {
      exportToast.value = "";
    }
  } catch (e: any) {
    const msg = e?.message || (typeof e === "string" ? e : JSON.stringify(e));
    errorMsg.value = `${t("export.pdfFailed")}: ${msg}`;
    exportToast.value = "";
  } finally {
    exportBusy.value = false;
  }
}

async function doPrint() {
  const body = await getMarkdownBodyForAction();
  if (body) printDocument(body, fileName.value);
}

function onSearchOpen(path: string, _line: number) {
  void loadFile(path);
}

function onInternalLink(path: string, hash: string) {
  void loadFile(path, hash);
}

function onKeydown(e: KeyboardEvent) {
  const mod = e.ctrlKey || e.metaKey;
  if (mod && e.key.toLowerCase() === "f" && !e.shiftKey) {
    e.preventDefault();
    find.open();
  } else if (mod && e.shiftKey && e.key.toLowerCase() === "f") {
    e.preventDefault();
    leftMode.value = "search";
    showFileTree.value = true;
  } else if (mod && e.key === ",") {
    e.preventDefault();
    showSettings.value = true;
  } else if (mod && e.key.toLowerCase() === "p") {
    e.preventDefault();
    void doPrint();
  } else if (mod && e.key.toLowerCase() === "r") {
    e.preventDefault();
    void reloadCurrentFile();
  } else if (mod && !e.shiftKey && e.key.toLowerCase() === "s") {
    e.preventDefault();
    if (currentFile.value && currentDirty.value) {
      void saveCurrentFile();
    } else if (currentFile.value) {
      exportToast.value = t("app.noUnsavedChanges");
    }
  } else if (mod && !e.shiftKey && e.key.toLowerCase() === "e") {
    e.preventDefault();
    if (content.value && !exportBusy.value) {
      showExportMenu.value = !showExportMenu.value;
    }
  } else if (e.key === "Escape") {
    if (contextMenu.value.visible) closeContextMenu();
    else if (find.visible.value) find.close();
    else if (showSettings.value) showSettings.value = false;
  }
}

let scrollSaveTimer: number | null = null;
function onViewerScroll() {
  onScroll();
  closeContextMenu();
  if (scrollSaveTimer) clearTimeout(scrollSaveTimer);
  scrollSaveTimer = window.setTimeout(saveCurrentScroll, 400);
}

watch(showFileTree, (v) =>
  localStorage.setItem("md-reader-show-tree", v ? "1" : "0")
);
watch(showToc, (v) =>
  localStorage.setItem("md-reader-show-toc", v ? "1" : "0")
);

let unlistenDrop: (() => void) | null = null;
let unlistenOpen: (() => void) | null = null;
let unlistenWindow: (() => void) | null = null;

async function restoreInitialFile() {
  const last = localStorage.getItem("md-reader-last-file");
  const savedActive = activeTabPath.value;
  const savedFirst = tabs.value[0]?.path || "";
  const target = tabMode.value ? savedActive || savedFirst || last : last;
  if (target && !currentFile.value) {
    try {
      await loadFile(target);
    } catch {
      /* ignore */
    }
  }
}

let windowRevealed = false;
async function revealWindow() {
  if (windowRevealed) return;
  windowRevealed = true;
  await nextTick();
  try {
    const window = getCurrentWindow();
    await window.show();
    await window.setFocus();
  } catch {
    /* ignore */
  }
}

onMounted(async () => {
  applyTheme();
  applyReadingSettings();
  await restoreWindowState();
  void watchWindowState().then((unlisten) => (unlistenWindow = unlisten));
  const revealTimer = window.setTimeout(() => {
    void revealWindow();
  }, 800);

  // 监听系统双击关联文件或已有窗口收到的新文件。
  try {
    const { listen } = await import("@tauri-apps/api/event");
    unlistenOpen = await listen<string>("md-reader://open-file", async (e) => {
      const path = e.payload;
      if (typeof path === "string" && path) {
        await loadFile(path);
      }
    });
  } catch (e) {
    console.warn("listen open-file unavailable", e);
  }

  await restoreInitialFile();
  window.clearTimeout(revealTimer);
  await revealWindow();

  void (async () => {
    try {
      await restoreRoot();
      if (rootDir.value) await startWatching(rootDir.value);
    } catch {
      /* ignore */
    }
  })();

  window.setTimeout(() => {
    void checkPdfEngine().then((p) => (pdfEnginePath.value = p));
  }, 500);

  try {
    const webview = getCurrentWebview();
    unlistenDrop = await webview.onDragDropEvent(async (event) => {
      if (event.payload.type === "drop") {
        const paths = event.payload.paths;
        if (paths && paths.length > 0) {
          const target = paths.find((p) =>
            /\.(md|markdown|mdx|txt)$/i.test(p)
          );
          if (target) await loadFile(target);
        }
      }
    });
  } catch (e) {
    console.warn("drag-drop unavailable", e);
  }
  window.addEventListener("keydown", onKeydown);
});

onUnmounted(() => {
  unlistenDrop?.();
  unlistenOpen?.();
  unlistenWindow?.();
  void watcher.stop();
  if (headingUpdateTimer) clearTimeout(headingUpdateTimer);
  window.removeEventListener("keydown", onKeydown);
});

watch(currentFile, (v) => {
  if (v) localStorage.setItem("md-reader-last-file", v);
});

watch(exportToast, (v) => {
  if (v) {
    window.setTimeout(() => {
      exportToast.value = "";
    }, 3500);
  }
});
</script>

<template>
  <div class="app" @contextmenu="openContextMenu" @click="closeContextMenu">
    <header class="toolbar">
      <div class="toolbar-group toolbar-left" role="group" aria-label="文件操作">
        <button class="btn action" @click="pickFile" :title="t('app.file') + ' .md'">
          <FileText :size="15" :stroke-width="1.9" />
          <span>{{ t("app.file") }}</span>
        </button>
        <button class="btn action" @click="pickFolder" :title="t('app.folder')">
          <FolderOpen :size="15" :stroke-width="1.9" />
          <span>{{ t("app.folder") }}</span>
        </button>
        <button
          v-if="rootDir"
          class="btn tool"
          @click="refreshTree"
          :disabled="treeLoading"
          :title="t('app.refresh')"
        >
          <RefreshCw :size="15" :stroke-width="1.9" />
          <span>{{ t("app.refresh") }}</span>
        </button>
        <button
          v-if="rootDir"
          class="btn tool"
          @click="closeFolder"
          :title="t('app.closeFolder')"
        >
          <X :size="15" :stroke-width="1.9" />
          <span>{{ t("app.closeFolder") }}</span>
        </button>
      </div>
      <div class="toolbar-group toolbar-right" role="group" aria-label="阅读工具">
        <div v-if="content" class="view-switch" role="group" aria-label="显示模式">
          <button
            class="view-btn"
            :class="{ active: viewMode === 'preview' }"
            :title="t('toolbar.previewMode')"
            @click="setViewMode('preview')"
          >
            <Eye :size="15" :stroke-width="1.9" />
            <span>{{ t("toolbar.preview") }}</span>
          </button>
          <button
            class="view-btn"
            :class="{ active: viewMode === 'source' }"
            :title="t('toolbar.sourceMode')"
            @click="setViewMode('source')"
          >
            <Code2 :size="15" :stroke-width="1.9" />
            <span>{{ t("toolbar.source") }}</span>
          </button>
        </div>
        <button
          v-if="content"
          class="btn tool save-btn"
          :class="{ invisible: viewMode !== 'source' }"
          @click="saveCurrentFile"
          :disabled="viewMode !== 'source' || !content || !currentFile || !currentDirty"
          :aria-hidden="viewMode !== 'source'"
          :tabindex="viewMode === 'source' ? 0 : -1"
          :title="t('toolbar.save') + ' (Ctrl+S)'"
        >
          <Save :size="15" :stroke-width="1.9" />
          <span>{{ t("toolbar.save") }}</span>
        </button>
        <button
          class="btn tool"
          @click="find.open()"
          :title="t('toolbar.find') + ' (Ctrl+F)'"
          :disabled="!content"
        >
          <Search :size="15" :stroke-width="1.9" />
          <span>{{ t("toolbar.find") }}</span>
        </button>
      <div class="export-wrap">
        <button
          class="btn tool"
          @click="showExportMenu = !showExportMenu"
          :disabled="!content || exportBusy"
          :title="exportBusy ? t('export.exportBusy') : t('export.exportShortcut')"
        >
          <LoaderCircle
            v-if="exportBusy"
            class="spin"
            :size="15"
            :stroke-width="1.9"
          />
          <Download v-else :size="15" :stroke-width="1.9" />
          <span>{{ exportBusy ? t("export.exportBusy") : t("toolbar.export") }}</span>
        </button>
        <div v-if="showExportMenu" class="export-menu" @click.stop>
          <button class="menu-item" @click="exportHtml(); showExportMenu = false">
            <span class="mi-label">{{ t("export.html") }}</span>
            <span class="mi-hint">{{ t("export.htmlHint") }}</span>
          </button>
          <button
            class="menu-item"
            @click="exportTxt"
          >
            <span class="mi-label">{{ t("export.txt") }}</span>
            <span class="mi-hint">{{ t("export.txtHint") }}</span>
          </button>
          <button
            class="menu-item"
            @click="exportPdf"
            :title="pdfEnginePath ? t('app.usePath', { path: pdfEnginePath }) : t('app.specifyEdgePath')"
          >
            <span class="mi-label">{{ t("export.pdf") }}</span>
            <span class="mi-hint">
              {{ pdfEnginePath ? t("export.pdfHint") : t("export.pdfNoEdge") }}
            </span>
          </button>
          <div class="menu-divider"></div>
          <button class="menu-item" @click="doPrint(); showExportMenu = false">
            <span class="mi-label">{{ t("export.print") }}</span>
            <span class="mi-hint">{{ t("export.printHint") }}</span>
          </button>
        </div>
      </div>
      <button
        class="btn tool"
        @click="showSettings = true"
        :title="t('toolbar.settings') + ' (Ctrl+,)'"
      >
        <Settings :size="15" :stroke-width="1.9" />
        <span>{{ t("toolbar.settings") }}</span>
      </button>
      <button
        class="btn tool"
        @click="showFileTree = !showFileTree"
        :title="t('app.toggleSidebar')"
      >
        <PanelLeft :size="15" :stroke-width="1.9" />
        <span>侧栏</span>
      </button>
      <button
        class="btn tool"
        @click="showToc = !showToc"
        :title="t('app.toggleToc')"
      >
        <List :size="15" :stroke-width="1.9" />
        <span>大纲</span>
      </button>
      <button class="btn tool" @click="toggleTheme" :title="t('app.toggleTheme')">
        <Moon v-if="theme === 'light'" :size="15" :stroke-width="1.9" />
        <Sun v-else :size="15" :stroke-width="1.9" />
        <span>{{ theme === "light" ? "夜间" : "日间" }}</span>
      </button>
      <button class="btn lang" @click="toggleLocale" :title="t('app.switchLanguage')">
        {{ locale === "zh-CN" ? "EN" : "中" }}
      </button>
      </div>
    </header>

    <div v-if="tabMode && tabs.length" class="tabbar">
      <div
        v-for="tab in tabs"
        :key="tab.path"
        class="tab-pill"
        :class="{ active: activeTabPath === tab.path }"
        :title="tab.path"
        role="button"
        tabindex="0"
        @click="switchTab(tab.path)"
        @keydown.enter.prevent="switchTab(tab.path)"
        @keydown.space.prevent="switchTab(tab.path)"
      >
        <span class="tab-name">{{ tab.name }}</span>
        <span v-if="tab.dirty" class="dirty-dot" title="未保存"></span>
        <button
          class="tab-close"
          :title="t('tabs.close')"
          @click.stop="closeTab(tab.path)"
        >
          <X :size="13" :stroke-width="2" />
        </button>
      </div>
      <button class="tab-add" :title="t('tabs.open')" @click="pickFile">
        <Plus :size="16" :stroke-width="2" />
      </button>
    </div>

    <main class="layout">
      <aside
        v-if="showFileTree"
        class="left"
        :style="{ width: leftWidth + 'px' }"
      >
        <div class="panel-tabs">
          <button
            class="tab"
            :class="{ active: leftMode === 'files' }"
            @click="leftMode = 'files'"
          >
            {{ t("app.files") }}
          </button>
          <button
            class="tab"
            :class="{ active: leftMode === 'search' }"
            @click="leftMode = 'search'"
            :title="t('app.search') + ' (Ctrl+Shift+F)'"
          >
            {{ t("app.search") }}
          </button>
        </div>
        <div v-if="leftMode === 'files'" class="panel-body">
          <div class="panel-header">
            <span>{{ rootDir ? t("app.files") : t("app.noFolder") }}</span>
            <span v-if="treeLoading" class="muted">…</span>
          </div>
          <div v-if="treeError" class="panel-error">{{ treeError }}</div>
          <div class="tree-scroll">
            <FileTree
              v-if="rootDir"
              :nodes="tree"
              :current-path="currentFile"
              @open="loadFile"
            />
            <div v-else class="empty-tip">
              {{ t("app.openFolderHint").split("\n")[0] }}<br />{{ t("app.openFolderHint").split("\n")[1] }}
            </div>
          </div>
        </div>
        <div v-else class="panel-body">
          <SearchPanel
            :visible="true"
            :root-dir="rootDir"
            @close="leftMode = 'files'"
            @open="onSearchOpen"
          />
        </div>
      </aside>

      <div
        v-if="showFileTree"
        class="resizer"
        @pointerdown="resizeLeft"
      ></div>

      <section
        ref="viewerEl"
        class="viewer"
        @scroll.passive="onViewerScroll"
      >
        <FindBar
          :visible="find.visible.value"
          :query="find.query.value"
          :case-sensitive="find.caseSensitive.value"
          :total="find.total.value"
          :active-index="find.activeIndex.value"
          @update:query="(v) => (find.query.value = v)"
          @update:case-sensitive="(v) => (find.caseSensitive.value = v)"
          @search="find.search"
          @next="find.next"
          @prev="find.prev"
          @close="find.close"
        />
        <div v-if="errorMsg" class="error">{{ errorMsg }}</div>
        <div v-else-if="!content" class="empty">
          <div class="empty-title">{{ t("app.emptyTitle") }}</div>
          <div class="empty-hint">
            {{ t("app.emptyHint").split("\n")[0] }}<br />{{ t("app.emptyHint").split("\n")[1] }}
          </div>
          <div class="shortcut-hint">
            {{ t("app.shortcutHint") }}
          </div>
        </div>
        <template v-else>
          <MarkdownView
            v-show="viewMode === 'preview'"
            ref="markdownRef"
            :source="content"
            :current-file="currentFile"
            :root-dir="rootDir"
            :render-tick="renderTick"
            @rendered="() => onRendered('preview')"
            @internal-link="onInternalLink"
          />
          <SourceEditor
            v-show="viewMode === 'source'"
            ref="sourceRef"
            :model-value="content"
            :current-file="currentFile"
            @update:model-value="onSourceInput"
            @rendered="() => onRendered('source')"
          />
        </template>
      </section>

      <div v-if="showToc" class="resizer" @pointerdown="resizeRight"></div>

      <aside
        v-if="showToc"
        class="right"
        :style="{ width: rightWidth + 'px' }"
      >
        <TocPanel
          :headings="headings"
          :active-id="activeId"
          @jump="jumpTo"
        />
      </aside>
    </main>

    <footer class="statusbar">
      <span class="status-path" :title="currentFile">
        {{ currentFile || "未打开文件" }}
      </span>
      <span>{{ fileSizeText }}</span>
      <span>{{ documentCharText }}</span>
      <span>标题 {{ headings.length }}</span>
      <span v-if="currentDirty" class="dirty-status">未保存</span>
    </footer>

    <div
      v-if="contextMenu.visible"
      class="context-menu"
      :style="{ left: contextMenu.x + 'px', top: contextMenu.y + 'px' }"
      @click.stop
      @contextmenu.prevent.stop
    >
      <button
        class="context-item"
        :disabled="!content"
        @click="runContextAction(() => find.open())"
      >
        <Search :size="15" :stroke-width="1.9" />
        <span>查找当前文档</span>
        <kbd>Ctrl+F</kbd>
      </button>
      <button
        class="context-item"
        :disabled="!currentFile"
        @click="runContextAction(reloadCurrentFile)"
      >
        <RefreshCw :size="15" :stroke-width="1.9" />
        <span>重新读取文件</span>
        <kbd>Ctrl+R</kbd>
      </button>
      <button
        class="context-item"
        :disabled="!selectedText"
        @click="runContextAction(copySelectedText)"
      >
        <Copy :size="15" :stroke-width="1.9" />
        <span>复制选中文字</span>
      </button>
      <button
        class="context-item"
        :disabled="!currentFile"
        @click="runContextAction(copyCurrentFilePath)"
      >
        <FileText :size="15" :stroke-width="1.9" />
        <span>复制文件路径</span>
      </button>
      <div class="context-divider"></div>
      <button
        class="context-item"
        :disabled="!content || exportBusy"
        @click="runContextAction(exportPdf)"
      >
        <FileDown :size="15" :stroke-width="1.9" />
        <span>导出为 PDF</span>
      </button>
      <button
        class="context-item"
        :disabled="!content || exportBusy"
        @click="runContextAction(exportTxt)"
      >
        <FileText :size="15" :stroke-width="1.9" />
        <span>导出为 TXT</span>
      </button>
      <button
        class="context-item"
        :disabled="!content || exportBusy"
        @click="runContextAction(exportHtml)"
      >
        <FileText :size="15" :stroke-width="1.9" />
        <span>导出为 HTML</span>
      </button>
      <button
        class="context-item"
        :disabled="!content"
        @click="runContextAction(doPrint)"
      >
        <Printer :size="15" :stroke-width="1.9" />
        <span>打印</span>
        <kbd>Ctrl+P</kbd>
      </button>
    </div>

    <SettingsDialog
      :visible="showSettings"
      :tab-mode="tabMode"
      @update:tab-mode="setTabMode"
      @close="showSettings = false"
    />

    <div v-if="exportToast" class="toast" @click="exportToast = ''">
      {{ exportToast }}
    </div>
    <div
      v-if="showExportMenu"
      class="menu-overlay"
      @click="showExportMenu = false"
    ></div>
  </div>
</template>

<style scoped>
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
}
.toolbar {
  flex: 0 0 auto;
  position: relative;
  z-index: 60;
  display: flex;
  align-items: center;
  gap: 8px;
  min-height: 42px;
  padding: 6px 10px;
  border-bottom: 1px solid var(--shell-toolbar-border);
  background: var(--shell-toolbar-bg);
  user-select: none;
  overflow: visible;
}
.toolbar-group {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}
.toolbar-left {
  flex: 0 0 auto;
}
.toolbar-right {
  flex: 0 0 auto;
  margin-left: auto;
}
.tabbar {
  flex: 0 0 auto;
  display: flex;
  align-items: flex-end;
  gap: 2px;
  padding: 5px 10px 0;
  height: 36px;
  border-bottom: 1px solid var(--shell-sidebar-border);
  background: var(--shell-sidebar-bg);
  overflow-x: auto;
  overflow-y: hidden;
  scrollbar-gutter: stable;
}
.tab-pill {
  flex: 0 0 190px;
  min-width: 92px;
  max-width: 220px;
  height: 31px;
  display: flex;
  align-items: center;
  position: relative;
  padding: 0 30px 0 10px;
  border: 1px solid var(--border);
  border-bottom: none;
  border-radius: 7px 7px 0 0;
  background: var(--bg-btn);
  color: var(--fg-muted);
  cursor: pointer;
  user-select: none;
}
.tab-pill:hover {
  background: var(--bg-btn-hover);
  color: var(--fg);
}
.tab-pill.active {
  background: var(--bg);
  color: var(--fg);
  box-shadow: 0 -1px 0 var(--link) inset;
}
.tab-name {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 12px;
}
.dirty-dot {
  position: absolute;
  right: 28px;
  width: 6px;
  height: 6px;
  border-radius: 999px;
  background: var(--mdr-danger);
  box-shadow: 0 0 0 2px var(--bg-btn);
}
.tab-close,
.tab-add {
  border: none;
  color: var(--fg-muted);
  background: transparent;
  cursor: pointer;
}
.tab-close {
  flex: 0 0 auto;
  position: absolute;
  top: 50%;
  right: 6px;
  width: 20px;
  height: 20px;
  transform: translateY(-50%);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 5px;
  padding: 0;
  opacity: 0.55;
}
.tab-close:hover,
.tab-add:hover {
  color: var(--fg);
  background: var(--bg-btn-hover);
  opacity: 1;
}
.tab-add {
  flex: 0 0 auto;
  width: 28px;
  height: 28px;
  margin-bottom: 1px;
  border-radius: 6px;
  font-size: 18px;
  line-height: 24px;
}
.filename {
  flex: 1 1 auto;
  font-size: 13px;
  color: var(--shell-filename-color);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  margin: 0 6px;
  min-width: 120px;
}
.btn {
  height: 30px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  font-size: 13px;
  line-height: 1;
  padding: 0 10px;
  border-radius: 6px;
  border: 1px solid var(--border);
  background: var(--bg-btn);
  color: var(--fg);
  cursor: pointer;
  box-shadow: 0 1px 1px rgba(15, 23, 42, 0.03);
  transition:
    background-color 0.12s,
    border-color 0.12s,
    color 0.12s,
    box-shadow 0.12s;
}
.btn:hover {
  background: var(--bg-btn-hover);
  border-color: color-mix(in srgb, var(--link) 26%, var(--border));
  box-shadow: 0 1px 2px rgba(15, 23, 42, 0.08);
}
.btn:disabled {
  opacity: 0.45;
  cursor: default;
  box-shadow: none;
}
.btn.icon {
  width: 32px;
  padding: 0;
}
.btn svg {
  display: block;
  flex: 0 0 auto;
}
.btn.action {
  min-width: 48px;
  font-weight: 500;
}
.btn.tool {
  white-space: nowrap;
}
.btn.lang {
  min-width: 40px;
  padding: 0 10px;
  font-weight: 500;
}
.save-btn {
  width: 70px;
}
.save-btn.invisible {
  visibility: hidden;
  pointer-events: none;
}
.view-switch {
  display: inline-flex;
  height: 30px;
  padding: 2px;
  border: 1px solid var(--border);
  border-radius: 7px;
  background: color-mix(in srgb, var(--bg-btn) 82%, var(--bg-toolbar));
}
.view-btn {
  height: 24px;
  display: inline-flex;
  align-items: center;
  gap: 5px;
  padding: 0 9px;
  border: 0;
  border-radius: 5px;
  background: transparent;
  color: var(--fg-muted);
  font-size: 12px;
  cursor: pointer;
  white-space: nowrap;
}
.view-btn:hover {
  color: var(--fg);
  background: var(--bg-btn-hover);
}
.view-btn.active {
  color: var(--link);
  background: var(--bg);
  box-shadow: 0 1px 2px rgba(15, 23, 42, 0.08);
}
.spin {
  animation: spin 0.9s linear infinite;
}
:global(:root[data-theme="dark"]) .btn {
  box-shadow: none;
}
:global(:root[data-theme="dark"]) .btn:hover {
  border-color: var(--mdr-panel-muted);
}
:global(:root[data-theme="dark"]) .view-btn.active {
  background: var(--mdr-panel-raised);
  color: #eaf2ff;
  box-shadow: none;
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
.layout {
  flex: 1 1 auto;
  display: flex;
  min-height: 0;
}
.left,
.right {
  flex: 0 0 auto;
  background: var(--shell-sidebar-bg);
  border-right: 1px solid var(--shell-sidebar-border);
  display: flex;
  flex-direction: column;
  min-width: 160px;
  overflow: hidden;
}
.right {
  background: var(--shell-right-bg);
  border-right: none;
  border-left: 1px solid var(--shell-sidebar-border);
}
.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0.6px;
  color: var(--shell-panel-header-color);
  border-bottom: 1px solid var(--shell-panel-header-border);
}
.panel-error {
  padding: 8px 12px;
  font-size: 12px;
  color: #c00;
  background: rgba(255, 0, 0, 0.06);
}
.tree-scroll {
  flex: 1 1 auto;
  overflow: auto;
  scrollbar-gutter: stable;
}
.empty-tip {
  padding: 16px 12px;
  font-size: 12px;
  color: var(--fg-muted);
  line-height: 1.7;
}
.muted {
  color: var(--fg-muted);
}
.resizer {
  flex: 0 0 4px;
  cursor: col-resize;
  background: transparent;
  position: relative;
}
.resizer:hover {
  background: var(--link);
  opacity: 0.4;
}
.viewer {
  flex: 1 1 auto;
  overflow-x: hidden;
  overflow-y: scroll;
  scrollbar-gutter: stable;
  overflow-anchor: none;
  background: var(--bg);
  min-width: 0;
  position: relative;
}
.empty {
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 12px;
  color: var(--fg-muted);
}
.empty-title {
  font-size: 28px;
  font-weight: 600;
}
.empty-hint {
  font-size: 14px;
  text-align: center;
}
.shortcut-hint {
  font-size: 12px;
  color: var(--fg-muted);
  margin-top: 8px;
}
.panel-tabs {
  display: flex;
  border-bottom: 1px solid var(--shell-sidebar-border);
  background: var(--shell-sidebar-bg);
}
.tab {
  flex: 1;
  padding: 6px 0;
  font-size: 12px;
  background: transparent;
  color: var(--shell-tab-color);
  border: none;
  border-bottom: 2px solid transparent;
  cursor: pointer;
}
.tab:hover {
  color: var(--shell-tab-hover-color);
}
.tab.active {
  color: var(--shell-tab-active-color);
  border-bottom-color: var(--shell-tab-active-border);
}
.panel-body {
  flex: 1 1 auto;
  display: flex;
  flex-direction: column;
  min-height: 0;
  overflow: hidden;
}
.export-wrap {
  position: relative;
}
.export-menu {
  position: absolute;
  top: 100%;
  right: 0;
  margin-top: 4px;
  background: var(--shell-export-bg);
  border: 1px solid var(--border);
  border-radius: 8px;
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.18);
  min-width: 260px;
  padding: 4px;
  z-index: 90;
}
.menu-item {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 2px;
  width: 100%;
  padding: 8px 12px;
  background: transparent;
  border: none;
  border-radius: 6px;
  color: var(--fg);
  cursor: pointer;
  text-align: left;
}
.menu-item:hover:not(:disabled) {
  background: var(--shell-export-hover-bg);
}
.menu-item:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
.mi-label {
  font-size: 13px;
  font-weight: 500;
}
.mi-hint {
  font-size: 11px;
  color: var(--fg-muted);
}
.menu-divider {
  height: 1px;
  background: var(--border);
  margin: 4px 0;
}
.menu-overlay {
  position: fixed;
  inset: 0;
  z-index: 20;
}
.statusbar {
  flex: 0 0 auto;
  display: flex;
  align-items: center;
  gap: 14px;
  min-height: 28px;
  padding: 0 12px;
  border-top: 1px solid var(--shell-sidebar-border);
  background: var(--shell-sidebar-bg);
  color: var(--fg-muted);
  font-size: 12px;
  user-select: none;
}
.status-path {
  flex: 1 1 auto;
  min-width: 0;
  color: var(--fg);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.dirty-status {
  color: var(--mdr-danger);
  font-weight: 650;
}
.context-menu {
  position: fixed;
  width: 250px;
  padding: 6px;
  border: 1px solid var(--border);
  border-radius: 9px;
  background: var(--shell-export-bg);
  color: var(--fg);
  box-shadow: 0 14px 34px rgba(15, 23, 42, 0.22);
  z-index: 70;
}
.context-item {
  width: 100%;
  height: 34px;
  display: grid;
  grid-template-columns: 20px 1fr auto;
  align-items: center;
  gap: 8px;
  padding: 0 9px;
  border: 0;
  border-radius: 7px;
  background: transparent;
  color: var(--fg);
  font-size: 13px;
  text-align: left;
  cursor: pointer;
}
.context-item:hover:not(:disabled) {
  background: var(--shell-export-hover-bg);
}
.context-item:disabled {
  opacity: 0.42;
  cursor: default;
}
.context-item kbd {
  color: var(--fg-muted);
  font-family: inherit;
  font-size: 11px;
}
.context-divider {
  height: 1px;
  margin: 6px 4px;
  background: var(--border);
}
.toast {
  position: fixed;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  padding: 10px 20px;
  background: rgba(35, 134, 54, 0.95);
  color: #fff;
  border-radius: 8px;
  font-size: 13px;
  z-index: 40;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.25);
  cursor: pointer;
  max-width: 70%;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.error {
  margin: 24px;
  padding: 12px 16px;
  border-radius: 6px;
  background: #fee;
  color: #c00;
  border: 1px solid #fcc;
}
</style>
