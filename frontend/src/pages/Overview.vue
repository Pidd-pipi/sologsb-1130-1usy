<script setup lang="ts">
/**
 * 进度总览：列出各镜头状态、帧数、预计时长与完成百分比，
 * 累计全片张数与待拍张数。消费 Shot、TakeLog、FrameEntry。
 */
import { computed, onMounted, ref } from 'vue';
import { useRouter } from 'vue-router';
import { storeToRefs } from 'pinia';
import { useShotStore } from '../stores/shotStore';
import { useFrameStore } from '../stores/frameStore';
import { useProgress } from '../hooks/useProgress';
import { listAllFrames } from '../db/api';
import { framesToDuration } from '../utils/frameMath';
import { formatDateTime } from '../utils/format';
import ShotProgress from '../components/common/ShotProgress.vue';
import StatusTag from '../components/common/StatusTag.vue';
import EmptyState from '../components/common/EmptyState.vue';
import type { FrameEntry } from '../types/frame';
import type { Shot } from '../types/shot';

const router = useRouter();
const shotStore = useShotStore();
const frameStore = useFrameStore();
const { shots } = storeToRefs(shotStore);
const { summaries, overall, loadTakes, loading } = useProgress();

const allFrames = ref<FrameEntry[]>([]);

onMounted(async () => {
  await shotStore.load();
  await loadTakes();
  allFrames.value = await listAllFrames();
});

const summaryOf = (shotId: number | undefined) => summaries.value.find((s) => s.shotId === shotId);

const rows = computed(() =>
  shots.value.map((shot) => {
    const frames = allFrames.value.filter((f) => f.shotId === shot.id);
    const summary = summaryOf(shot.id);
    return {
      shot,
      frameCount: frames.length,
      duration: framesToDuration(shot.endFrame - shot.startFrame + 1, shot.fps),
      summary,
    };
  }),
);

const waitingFrames = computed(() => overall.value.remaining);
const statusCount = computed(() => ({
  idle: shots.value.filter((s) => s.status === '未开机').length,
  shooting: shots.value.filter((s) => s.status === '拍摄中').length,
  done: shots.value.filter((s) => s.status === '已完成').length,
}));

function goDetail(id: number | undefined) {
  if (typeof id !== 'number') return;
  void frameStore.loadForShot(id);
  void router.push(`/shots/${id}`);
}

/* ---------------- 复制镜头 ---------------- */

const copySource = ref<Shot | null>(null);
const copyCode = ref('');
const copyError = ref('');
const copying = ref(false);

/** 建议新镜号：取现有镜号最大数字 +1 */
const suggestedCopyCode = computed(() => {
  const nums = shots.value
    .map((s) => Number((s.code.match(/\d+/) ?? [])[0]))
    .filter((n) => Number.isFinite(n));
  const next = nums.length ? Math.max(...nums) + 1 : 1;
  return `S${String(next).padStart(2, '0')}`;
});

function openCopy(shot: Shot) {
  copySource.value = shot;
  copyCode.value = suggestedCopyCode.value;
  copyError.value = '';
}

function closeCopy() {
  if (copying.value) return;
  copySource.value = null;
}

function validateCopy(): string {
  const code = copyCode.value.trim();
  if (!code) return '请填写新镜号';
  if (!/^[A-Za-z]{1,3}\d{1,3}$/.test(code)) return '镜号格式形如 S01';
  if (shots.value.some((s) => s.code.trim().toUpperCase() === code.toUpperCase())) {
    return '该镜号已存在，请换一个';
  }
  return '';
}

async function submitCopy() {
  const source = copySource.value;
  if (!source || typeof source.id !== 'number') return;
  copyError.value = validateCopy();
  if (copyError.value) return;
  copying.value = true;
  try {
    const saved = await shotStore.duplicate(source.id, copyCode.value);
    copySource.value = null;
    await frameStore.loadForShot(saved.id as number);
    await router.push(`/shots/${saved.id}`);
  } catch (e) {
    copyError.value = e instanceof Error ? e.message : '复制失败，请重试';
  } finally {
    copying.value = false;
  }
}
</script>

<template>
  <section class="page">
    <header class="page-head">
      <div>
        <h1>进度总览</h1>
        <p class="sub">定格动画拍摄全片的镜头状态、帧序规模与实拍完成度</p>
      </div>
      <div class="head-actions">
        <button type="button" class="btn primary" @click="router.push('/shots/new')">新建镜头</button>
        <button type="button" class="btn" @click="router.push('/frames')">帧序编排台</button>
      </div>
    </header>

    <div class="stat-row">
      <div class="stat">
        <span class="label">镜头总数</span>
        <span class="value">{{ shots.length }}</span>
        <span class="hint">未开机 {{ statusCount.idle }} · 拍摄中 {{ statusCount.shooting }} · 已完成 {{ statusCount.done }}</span>
      </div>
      <div class="stat">
        <span class="label">全片计划张数</span>
        <span class="value">{{ overall.planned }}</span>
        <span class="hint">已登记帧条目 {{ allFrames.length }} 条</span>
      </div>
      <div class="stat">
        <span class="label">累计实拍张数</span>
        <span class="value">{{ overall.taken }}</span>
        <span class="hint">废帧 {{ overall.wasted }} 张</span>
      </div>
      <div class="stat">
        <span class="label">待拍张数</span>
        <span class="value">{{ waitingFrames }}</span>
        <span class="hint">整体完成 {{ overall.percent }}%</span>
      </div>
    </div>

    <div class="panel">
      <div class="panel-head">
        <h2>镜头清单</h2>
        <span class="muted">{{ loading ? '读取实拍记录中…' : '数据来源：IndexedDB（gbstopmotion-db）' }}</span>
      </div>

      <EmptyState
        v-if="!rows.length"
        title="还没有镜头"
        description="创建第一个镜头后，这里会汇总各镜头的帧数与完成百分比。"
        action-text="新建镜头"
        @action="router.push('/shots/new')"
      />

      <table v-else class="table" data-testid="shot-table">
        <thead>
          <tr>
            <th>镜号</th>
            <th>场景</th>
            <th>状态</th>
            <th>帧率</th>
            <th>帧区间</th>
            <th>帧条目</th>
            <th>预计时长</th>
            <th>完成度</th>
            <th>负责人</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in rows" :key="row.shot.id">
            <td class="mono">{{ row.shot.code }}</td>
            <td>{{ row.shot.sceneName }}</td>
            <td><StatusTag :status="row.shot.status" size="small" /></td>
            <td>{{ row.shot.fps }} fps</td>
            <td class="mono">{{ row.shot.startFrame }} – {{ row.shot.endFrame }}</td>
            <td>{{ row.frameCount }}</td>
            <td>{{ row.duration }} s</td>
            <td class="progress-cell">
              <ShotProgress
                compact
                :planned="row.summary?.planned ?? 0"
                :taken="row.summary?.taken ?? 0"
                :wasted="row.summary?.wasted ?? 0"
                :remaining="row.summary?.remaining ?? 0"
                :percent="row.summary?.percent ?? 0"
              />
            </td>
            <td>{{ row.shot.owner || '未指派' }}</td>
            <td>
              <div class="row-actions">
                <button type="button" class="btn small" @click="goDetail(row.shot.id)">查看详情</button>
                <button
                  type="button"
                  class="btn small"
                  data-testid="copy-shot"
                  @click="openCopy(row.shot)"
                >复制</button>
              </div>
            </td>
          </tr>
        </tbody>
      </table>

      <p v-if="rows.length" class="muted footer-note">
        最近更新：{{ formatDateTime(Math.max(...shots.map((s) => s.updatedAt || 0))) }}
      </p>
    </div>

    <div v-if="copySource" class="modal-mask" data-testid="copy-dialog" @click.self="closeCopy">
      <div class="modal" role="dialog" aria-modal="true" aria-label="复制镜头">
        <h3>复制镜头 {{ copySource.code }}</h3>
        <p class="muted">
          新镜头将沿用 {{ copySource.sceneName }} 的帧率（{{ copySource.fps }} fps）、时长、
          起始帧（{{ copySource.startFrame }}）、逐帧曝光设置与道具轨迹；实拍记录不会复制。
        </p>
        <label class="field">
          <span>新镜号</span>
          <input
            v-model="copyCode"
            type="text"
            maxlength="8"
            data-testid="copy-code"
            :disabled="copying"
            @keyup.enter="submitCopy"
          />
        </label>
        <button type="button" class="link-btn" @click="copyCode = suggestedCopyCode">
          使用建议镜号 {{ suggestedCopyCode }}
        </button>
        <p v-if="copyError" class="err" data-testid="copy-error">{{ copyError }}</p>
        <div class="modal-actions">
          <button
            type="button"
            class="btn primary"
            :disabled="copying"
            data-testid="copy-submit"
            @click="submitCopy"
          >{{ copying ? '复制中…' : '复制并进入新镜头' }}</button>
          <button type="button" class="btn" :disabled="copying" @click="closeCopy">取消</button>
        </div>
      </div>
    </div>
  </section>
</template>

<style scoped>
.page {
  display: flex;
  flex-direction: column;
  gap: 18px;
}
.page-head {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  gap: 16px;
}
h1 {
  margin: 0;
  font-size: 22px;
}
.sub {
  margin: 4px 0 0;
  color: #6b7686;
  font-size: 13px;
}
.head-actions {
  display: flex;
  gap: 10px;
}
.stat-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 12px;
}
.stat {
  background: #fff;
  border: 1px solid #e2e7ef;
  border-radius: 10px;
  padding: 14px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.stat .label {
  font-size: 12px;
  color: #6b7686;
}
.stat .value {
  font-size: 24px;
  font-weight: 700;
  color: #1f2d3d;
}
.stat .hint {
  font-size: 12px;
  color: #8a94a6;
}
.panel {
  background: #fff;
  border: 1px solid #e2e7ef;
  border-radius: 10px;
  padding: 16px;
}
.panel-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
}
.panel-head h2 {
  margin: 0;
  font-size: 16px;
}
.muted {
  color: #8a94a6;
  font-size: 12px;
}
.table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}
.table th,
.table td {
  text-align: left;
  padding: 10px 8px;
  border-bottom: 1px solid #eef1f6;
  vertical-align: middle;
}
.table th {
  color: #6b7686;
  font-weight: 600;
  font-size: 12px;
}
.mono {
  font-family: ui-monospace, SFMono-Regular, Menlo, monospace;
}
.progress-cell {
  min-width: 210px;
}
.btn {
  height: 32px;
  padding: 0 14px;
  border-radius: 6px;
  border: 1px solid #cfd6e0;
  background: #fff;
  color: #1f2d3d;
  cursor: pointer;
  font-size: 13px;
}
.btn.primary {
  background: #2f6fed;
  border-color: #2f6fed;
  color: #fff;
}
.btn.small {
  height: 28px;
  padding: 0 10px;
  font-size: 12px;
}
.footer-note {
  margin: 10px 0 0;
}
.row-actions {
  display: flex;
  gap: 8px;
}
.modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(31, 45, 61, 0.45);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
}
.modal {
  width: 380px;
  max-width: calc(100vw - 32px);
  background: #fff;
  border-radius: 10px;
  padding: 18px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.modal h3 {
  margin: 0;
  font-size: 16px;
}
.modal .muted {
  margin: 0;
  line-height: 1.6;
}
.field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 12px;
  color: #5a6472;
}
.field input {
  height: 32px;
  border: 1px solid #cfd6e0;
  border-radius: 6px;
  padding: 0 8px;
  font-size: 13px;
  background: #fff;
  color: #1f2d3d;
}
.link-btn {
  align-self: flex-start;
  border: none;
  background: none;
  color: #2f6fed;
  cursor: pointer;
  font-size: 12px;
  padding: 0;
}
.err {
  margin: 0;
  color: #c45656;
  font-size: 12px;
}
.modal-actions {
  display: flex;
  gap: 10px;
}
.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

</style>
