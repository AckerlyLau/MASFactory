<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue';
import { useRuntimeStore } from '../stores/runtime';
import SessionDetail from './SessionDetail.vue';

const runtime = useRuntimeStore();

const sessions = computed(() => runtime.runSessions);
const selectedId = computed(() => runtime.selectedRunSessionId);

const layoutRef = ref<HTMLDivElement | null>(null);
const sidebarWidth = ref<number>(280);
const SIDEBAR_WIDTH_KEY = 'masfactory-visualizer.run.sidebarWidth';
const SIDEBAR_WIDTH_MIN = 88;
const DETAIL_WIDTH_MIN = 320;
const SIDEBAR_WIDTH_MAX = 620;

let resizeMove: ((e: MouseEvent) => void) | null = null;
let resizeUp: (() => void) | null = null;

function clamp(n: number, min: number, max: number): number {
  return Math.max(min, Math.min(max, n));
}

function loadSidebarWidth(): void {
  try {
    const raw = window.localStorage.getItem(SIDEBAR_WIDTH_KEY);
    const next = Number(raw);
    if (Number.isFinite(next) && next > 0) {
      sidebarWidth.value = clamp(Math.round(next), SIDEBAR_WIDTH_MIN, SIDEBAR_WIDTH_MAX);
    }
  } catch {
    // ignore
  }
}

function persistSidebarWidth(): void {
  try {
    window.localStorage.setItem(SIDEBAR_WIDTH_KEY, String(Math.round(sidebarWidth.value)));
  } catch {
    // ignore
  }
}

function resetSidebarWidth(): void {
  sidebarWidth.value = 280;
  persistSidebarWidth();
}

function startResizeSidebar(e: MouseEvent): void {
  if (e.button !== 0) return;
  if (!layoutRef.value) return;
  e.preventDefault();
  e.stopPropagation();

  const root = layoutRef.value;
  const applyAt = (clientX: number) => {
    const rect = root.getBoundingClientRect();
    const maxFromContainer = Math.max(
      SIDEBAR_WIDTH_MIN,
      Math.floor(rect.width - 10 - DETAIL_WIDTH_MIN)
    );
    const max = clamp(maxFromContainer, SIDEBAR_WIDTH_MIN, SIDEBAR_WIDTH_MAX);
    sidebarWidth.value = clamp(Math.round(clientX - rect.left), SIDEBAR_WIDTH_MIN, max);
  };

  const prevCursor = document.body.style.cursor;
  const prevUserSelect = document.body.style.userSelect;
  document.body.style.cursor = 'col-resize';
  document.body.style.userSelect = 'none';

  applyAt(e.clientX);

  resizeMove = (evt: MouseEvent) => applyAt(evt.clientX);
  resizeUp = () => {
    if (resizeMove) window.removeEventListener('mousemove', resizeMove, true);
    if (resizeUp) window.removeEventListener('mouseup', resizeUp, true);
    resizeMove = null;
    resizeUp = null;
    document.body.style.cursor = prevCursor;
    document.body.style.userSelect = prevUserSelect;
    persistSidebarWidth();
  };

  window.addEventListener('mousemove', resizeMove, true);
  window.addEventListener('mouseup', resizeUp, true);
}

function isAlive(sessionId: string): boolean {
  return runtime.isSessionAlive(sessionId);
}

function openSession(sessionId: string) {
  runtime.selectRunSession(sessionId);
}

function openSessionInTab(sessionId: string) {
  runtime.openSessionInTab(sessionId);
}

function deleteSession(sessionId: string) {
  runtime.deleteSession(sessionId);
}

function backToList() {
  runtime.clearRunSelection();
}

onMounted(() => {
  loadSidebarWidth();
});

onBeforeUnmount(() => {
  if (resizeMove) window.removeEventListener('mousemove', resizeMove, true);
  if (resizeUp) window.removeEventListener('mouseup', resizeUp, true);
  resizeMove = null;
  resizeUp = null;
});
</script>

<template>
  <div class="tab-root">
    <div class="header">
      <div class="title">Run</div>
      <div class="meta">MASFACTORY_VISUALIZER_PORT: {{ runtime.port ?? '—' }}</div>
    </div>

    <div v-if="sessions.length === 0 && !selectedId" class="empty">No active runtime processes.</div>

    <!-- No selection: keep the legacy "full list" layout -->
    <div v-else-if="!selectedId" class="list">
      <div
        v-for="s in sessions"
        :key="s.id"
        class="row"
        :class="{ attention: runtime.humanPendingCount(s.id) > 0, ended: !isAlive(s.id) }"
        @contextmenu.prevent
      >
        <div class="main">
          <div class="name">
            <span class="badge">{{ s.mode }}</span>
            <span class="graph">{{ s.graphName ?? '(unknown graph)' }}</span>
            <span v-if="!isAlive(s.id)" class="ended-badge">Exited</span>
            <span v-if="runtime.humanPendingCount(s.id) > 0" class="attention-badge">Human</span>
          </div>
          <div class="sub">
            <span>PID: {{ s.pid ?? '—' }}</span>
            <span class="dot">·</span>
            <span>Session: {{ s.id }}</span>
          </div>
        </div>
        <div class="actions">
          <button class="btn" @click="openSession(s.id)">View</button>
          <button class="btn secondary" :disabled="!isAlive(s.id)" @click="openSessionInTab(s.id)">
            Open Tab
          </button>
          <button v-if="!isAlive(s.id)" class="btn danger" @click="deleteSession(s.id)">Delete</button>
        </div>
      </div>
    </div>

    <!-- Selected: sidebar list + right detail -->
    <div v-else ref="layoutRef" class="split">
      <aside class="sidebar" :style="{ width: `${sidebarWidth}px`, minWidth: `${sidebarWidth}px` }">
        <div
          class="sidebar-resize-handle"
          title="Drag to resize session list width (double-click to reset)"
          @mousedown="startResizeSidebar"
          @dblclick="resetSidebarWidth"
        ></div>
        <button
          v-for="s in sessions"
          :key="s.id"
          class="session"
          :class="{
            active: s.id === selectedId,
            attention: runtime.humanPendingCount(s.id) > 0,
            ended: !isAlive(s.id)
          }"
          @click="openSession(s.id)"
          @contextmenu.prevent
        >
          <div class="session-name">
            <span class="session-title">{{ s.graphName ?? '(unknown graph)' }}</span>
            <span v-if="!isAlive(s.id)" class="session-meta">
              <span class="ended-pill">exited</span>
              <span class="delete-pill" title="Delete" @click.stop="deleteSession(s.id)">×</span>
            </span>
          </div>
          <div class="session-sub">PID: {{ s.pid ?? '—' }} · {{ s.id }}</div>
        </button>
      </aside>

      <section class="detail">
        <SessionDetail :session-id="selectedId" :show-back="true" @back="backToList" />
      </section>
    </div>
  </div>
</template>

<style scoped>
.tab-root {
  height: 100%;
  padding: 12px;
  box-sizing: border-box;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 10px;
}

.title {
  font-size: 14px;
  font-weight: 600;
}

.meta {
  font-size: 12px;
  opacity: 0.8;
}

.empty {
  padding: 12px;
  opacity: 0.8;
}

.list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.row {
  display: flex;
  gap: 10px;
  align-items: center;
  padding: 10px;
  border: 1px solid var(--vscode-panel-border, #2d2d2d);
  border-radius: 8px;
  background: var(--vscode-editor-background);
}

.row.attention {
  border-color: rgba(215, 186, 125, 0.75);
  animation: attentionPulse 1.2s ease-in-out infinite;
}

.main {
  flex: 1;
  min-width: 0;
}

.actions {
  display: flex;
  gap: 8px;
  flex-shrink: 0;
}

.name {
  display: flex;
  gap: 8px;
  align-items: center;
}

.badge {
  font-size: 11px;
  padding: 2px 6px;
  border-radius: 999px;
  background: rgba(14, 99, 156, 0.25);
  border: 1px solid rgba(14, 99, 156, 0.35);
}

.attention-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 999px;
  border: 1px solid rgba(215, 186, 125, 0.55);
  background: rgba(215, 186, 125, 0.16);
}

.ended-badge {
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 999px;
  border: 1px solid rgba(120, 120, 120, 0.5);
  background: rgba(120, 120, 120, 0.16);
  opacity: 0.9;
}

.graph {
  font-weight: 500;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sub {
  font-size: 12px;
  opacity: 0.8;
  margin-top: 4px;
  display: flex;
  gap: 6px;
  align-items: center;
}

.dot {
  opacity: 0.6;
}

.btn {
  font-size: 12px;
  padding: 6px 10px;
  border-radius: 6px;
  border: 1px solid var(--vscode-button-border, transparent);
  background: var(--vscode-button-background, #0e639c);
  color: var(--vscode-button-foreground, #fff);
  cursor: pointer;
}

.btn:hover {
  background: var(--vscode-button-hoverBackground, #1177bb);
}

.btn.secondary {
  background: transparent;
  color: var(--vscode-editor-foreground);
  border-color: var(--vscode-panel-border, #2d2d2d);
}

.btn.secondary:hover {
  background: var(--vscode-list-hoverBackground, rgba(255, 255, 255, 0.06));
}

.btn.danger {
  background: rgba(244, 135, 113, 0.16);
  border-color: rgba(244, 135, 113, 0.35);
  color: var(--vscode-editor-foreground);
}

.btn.danger:hover {
  background: rgba(244, 135, 113, 0.24);
}

.split {
  display: flex;
  gap: 10px;
  height: calc(100% - 34px);
  min-height: 0;
}

.sidebar {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 8px;
  overflow: auto;
  border-right: 1px solid var(--vscode-panel-border, #2d2d2d);
  padding-right: 10px;
  box-sizing: border-box;
}

.sidebar-resize-handle {
  position: absolute;
  top: 0;
  right: -5px;
  width: 10px;
  height: 100%;
  cursor: col-resize;
  z-index: 5;
}

.sidebar-resize-handle::after {
  content: '';
  position: absolute;
  top: 0;
  left: 4px;
  width: 2px;
  height: 100%;
  border-radius: 999px;
  background: rgba(128, 128, 128, 0.22);
  transition: background 120ms ease;
}

.sidebar-resize-handle:hover::after {
  background: rgba(128, 128, 128, 0.45);
}

.session {
  text-align: left;
  padding: 10px;
  border: 1px solid var(--vscode-panel-border, #2d2d2d);
  border-radius: 8px;
  background: transparent;
  color: var(--vscode-editor-foreground);
  cursor: pointer;
}

.session:hover {
  background: var(--vscode-list-hoverBackground, rgba(255, 255, 255, 0.06));
}

.session.active {
  border-color: rgba(14, 99, 156, 0.6);
  background: rgba(14, 99, 156, 0.12);
}

.session.attention:not(.active) {
  border-color: rgba(215, 186, 125, 0.75);
  background: rgba(215, 186, 125, 0.08);
}

.session.attention:not(.active):hover {
  background: rgba(215, 186, 125, 0.12);
}

.session-name {
  font-weight: 600;
  display: flex;
  align-items: center;
  min-width: 0;
}

.session-title {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.session-meta {
  margin-left: auto;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.ended-pill {
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 999px;
  border: 1px solid rgba(120, 120, 120, 0.5);
  background: rgba(120, 120, 120, 0.16);
  opacity: 0.9;
}

.delete-pill {
  width: 18px;
  height: 18px;
  border-radius: 999px;
  border: 1px solid rgba(244, 135, 113, 0.45);
  background: rgba(244, 135, 113, 0.18);
  color: var(--vscode-editor-foreground);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
  cursor: pointer;
  user-select: none;
}

.delete-pill:hover {
  background: rgba(244, 135, 113, 0.26);
}

.session-sub {
  margin-top: 4px;
  font-size: 12px;
  opacity: 0.8;
}

.detail {
  flex: 1;
  min-width: 0;
  min-height: 0;
}

@keyframes attentionPulse {
  0% {
    box-shadow: 0 0 0 rgba(215, 186, 125, 0);
  }
  50% {
    box-shadow: 0 0 18px rgba(215, 186, 125, 0.24);
  }
  100% {
    box-shadow: 0 0 0 rgba(215, 186, 125, 0);
  }
}
</style>
