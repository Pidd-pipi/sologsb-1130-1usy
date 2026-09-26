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
import { listAllFrames, listAllProps } from '../db/api';
import { framesToDuration } from '../utils/frameMath';
import { formatDateTime } from '../utils/format';
import ShotProgress from '../components/common/ShotProgress.vue';
import StatusTag from '../components/common/StatusTag.vue';
import EmptyState from '../components/common/EmptyState.vue';
import type { FrameEntry } from '../types/frame';
import type { PropState } from '../types/prop';

const router = useRouter();
const shotStore = useShotStore();
const frameStore = useFrameStore();
const { shots } = storeToRefs(shotStore);
const { summaries, overall, loadTakes, loading } = useProgress();

const allFrames = ref<FrameEntry[]>([]);
const allProps = ref<PropState[]>([]);

onMounted(async () => {
  await shotStore.load();
  await loadTakes();
  allFrames.value = await listAllFrames();
  allProps.value = await listAllProps();
});

/* ---------------- 同机位复制镜头 ---------------- */

const copyOpen = ref(false);
const copySourceId = ref<number | null>(null);
const copyCode = ref('');
const copySubmitting = ref(false);
const copyError = ref('');
const duplicateCode = ref(false);

function openCopy(sourceId?: number) {
  const firstId = sourceId ?? shots.value[0]?.id;
  copySourceId.value = typeof firstId === 'number' ? firstId : null;
  copyCode.value = '';
  copyError.value = '';
  duplicateCode.value = false;
  copySubmitting.value = false;
  copyOpen.value = true;
}

function closeCopy() {
  if (copySubmitting.value) return;
  copyOpen.value = false;
}

const copySource = computed(() => shots.value.find((s) => s.id === copySourceId.value));

/** 复制时随原镜头沿用的内容，用于弹窗内预览 */
const copyInherit = computed(() => {
  const shot = copySource.value;
  if (!shot) return { frameCount: 0, propCount: 0, duration: 0 };
  const frameCount = allFrames.value.filter((f) => f.shotId === shot.id).length;
  const propCount = allProps.value.filter((p) => p.shotId === shot.id).length;
  return {
    frameCount,
    propCount,
    duration: framesToDuration(shot.endFrame - shot.startFrame + 1, shot.fps),
  };
});

function checkDuplicate() {
  duplicateCode.value = shotStore.shots.some((s) => s.code.trim().toUpperCase() === copyCode.value.trim().toUpperCase());
  return duplicateCode.value;
}

function validateCopy(): string {
  if (copySourceId.value === null) return '请选择原镜头';
  if (!copyCode.value.trim()) return '请填写新镜号';
  if (!/^[A-Za-z]{1,3}\d{1,3}$/.test(copyCode.value.trim())) return '镜号格式形如 S01';
  if (checkDuplicate()) return '该镜号已存在，请换一个';
  return '';
}

async function submitCopy() {
  copyError.value = validateCopy();
  if (copyError.value) return;
  copySubmitting.value = true;
  try {
    const created = await shotStore.duplicate(copySourceId.value as number, copyCode.value.trim().toUpperCase());
    copyOpen.value = false;
    await frameStore.loadForShot(created.id as number);
    await loadTakes();
    allFrames.value = await listAllFrames();
    allProps.value = await listAllProps();
    // 保存成功后直接进入新镜头，继续同机位拍摄
    await router.push(`/shots/${created.id}`);
  } catch (e) {
    copyError.value = e instanceof Error ? e.message : '复制失败，请重试';
  } finally {
    copySubmitting.value = false;
  }
}

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
        <button type="button" class="btn" data-testid="copy-shot-entry" :disabled="!shots.length" @click="openCopy()">复制镜头</button>
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
            <td class="row-actions">
              <button type="button" class="btn small" @click="goDetail(row.shot.id)">查看详情</button>
              <button type="button" class="btn small" data-testid="copy-shot-row" @click="openCopy(row.shot.id)">复制镜头</button>
            </td>
          </tr>
        </tbody>
      </table>

      <p v-if="rows.length" class="muted footer-note">
        最近更新：{{ formatDateTime(Math.max(...shots.map((s) => s.updatedAt || 0))) }}
      </p>
    </div>

    <div v-if="copyOpen" class="modal-mask" data-testid="copy-shot-modal" @click.self="closeCopy">
      <div class="modal" role="dialog" aria-modal="true" aria-labelledby="copy-shot-title">
        <header class="modal-head">
          <h2 id="copy-shot-title">复制镜头（同机位连镜）</h2>
          <button type="button" class="modal-close" :disabled="copySubmitting" @click="closeCopy">×</button>
        </header>

        <div class="modal-body">
          <p class="modal-tip">
            新镜头沿用原镜头的帧率、时长、起始帧、每一帧的曝光设置与道具轨迹；实拍记录不继承，状态从「未开机」重新开始。
          </p>

          <label class="field">
            <span>原镜头</span>
            <select
              v-model.number="copySourceId"
              data-testid="copy-shot-source"
              :disabled="copySubmitting"
              @change="checkDuplicate"
            >
              <option v-for="s in shots" :key="s.id" :value="s.id">
                {{ s.code }} · {{ s.sceneName }}（{{ s.fps }}fps · 帧 {{ s.startFrame }}–{{ s.endFrame }}）
              </option>
            </select>
          </label>

          <label class="field">
            <span>新镜号</span>
            <input
              v-model="copyCode"
              type="text"
              maxlength="8"
              placeholder="例如 S02"
              data-testid="copy-shot-code"
              :disabled="copySubmitting"
              @input="checkDuplicate"
              @keyup.enter="submitCopy"
            />
            <small v-if="duplicateCode" class="err">该镜号已存在</small>
          </label>

          <dl v-if="copySource" class="inherit-list" data-testid="copy-shot-inherit">
            <div><dt>场景</dt><dd>{{ copySource.sceneName }}</dd></div>
            <div><dt>帧率 / 时长</dt><dd>{{ copySource.fps }} fps · {{ copyInherit.duration }} s</dd></div>
            <div><dt>起始帧 / 帧区间</dt><dd class="mono">{{ copySource.startFrame }} – {{ copySource.endFrame }}</dd></div>
            <div><dt>沿用帧条目</dt><dd>{{ copyInherit.frameCount }} 条（逐帧曝光设置）</dd></div>
            <div><dt>沿用道具轨迹</dt><dd>{{ copyInherit.propCount }} 条</dd></div>
            <div><dt>实拍记录</dt><dd class="muted">不继承</dd></div>
          </dl>

          <p v-if="copyError" class="err big" data-testid="copy-shot-error">{{ copyError }}</p>
        </div>

        <footer class="modal-foot">
          <button type="button" class="btn" :disabled="copySubmitting" @click="closeCopy">取消</button>
          <button
            type="button"
            class="btn primary"
            data-testid="copy-shot-submit"
            :disabled="copySubmitting || duplicateCode"
            @click="submitCopy"
          >
            {{ copySubmitting ? '复制中…' : '复制并进入拍摄' }}
          </button>
        </footer>
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
.row-actions {
  display: flex;
  gap: 6px;
  white-space: nowrap;
}
.modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(31, 45, 61, 0.42);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 16px;
  z-index: 100;
}
.modal {
  background: #fff;
  border-radius: 12px;
  width: min(520px, 100%);
  max-height: 90vh;
  overflow-y: auto;
  box-shadow: 0 18px 48px rgba(31, 45, 61, 0.22);
}
.modal-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 14px 18px;
  border-bottom: 1px solid #eef1f6;
}
.modal-head h2 {
  margin: 0;
  font-size: 16px;
}
.modal-close {
  border: none;
  background: none;
  font-size: 20px;
  line-height: 1;
  color: #8a94a6;
  cursor: pointer;
  padding: 2px 6px;
}
.modal-close:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}
.modal-body {
  padding: 16px 18px;
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.modal-tip {
  margin: 0;
  font-size: 12px;
  color: #6b7686;
  background: #f5f8ff;
  border: 1px solid #dbe6ff;
  border-radius: 8px;
  padding: 8px 10px;
}
.modal-body .field {
  display: flex;
  flex-direction: column;
  gap: 4px;
  font-size: 12px;
  color: #5a6472;
}
.modal-body .field input,
.modal-body .field select {
  height: 32px;
  border: 1px solid #cfd6e0;
  border-radius: 6px;
  padding: 0 8px;
  font-size: 13px;
  background: #fff;
  color: #1f2d3d;
}
.modal-body .field input:disabled,
.modal-body .field select:disabled {
  background: #f2f4f8;
  cursor: not-allowed;
}
.inherit-list {
  margin: 0;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px 16px;
  background: #fafbfd;
  border: 1px solid #eef1f6;
  border-radius: 8px;
  padding: 10px 12px;
  font-size: 12px;
}
.inherit-list div {
  display: flex;
  justify-content: space-between;
  gap: 8px;
}
.inherit-list dt {
  color: #8a94a6;
}
.inherit-list dd {
  margin: 0;
  color: #1f2d3d;
  text-align: right;
}
.err {
  color: #c45656;
  font-size: 12px;
}
.err.big {
  font-size: 13px;
  margin: 0;
}
.modal-foot {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  padding: 12px 18px;
  border-top: 1px solid #eef1f6;
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
</style>
