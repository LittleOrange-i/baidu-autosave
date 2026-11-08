<template>
  <div class="subscriptions-view">
    <div class="page-header">
      <h1 class="page-title">订阅同步</h1>
      <p class="page-desc">
        自动读取百度网盘「订阅分享管理」里的历史订阅，一键批量创建转存任务；已经转存过的会自动跳过，只补新文件。
      </p>
    </div>

    <!-- 主功能卡片 -->
    <el-card class="hero-card">
      <div class="hero-top">
        <div class="hero-switch">
          <span class="hero-label">启用订阅同步</span>
          <el-switch v-model="form.enabled" size="large" />
        </div>
        <div class="hero-hint" :class="{ on: form.enabled }">
          {{ form.enabled ? '已开启，可以按下面的频率自动帮你同步' : '开启后即可自动同步你的历史订阅' }}
        </div>
      </div>

      <el-form label-width="120px" class="hero-form">
        <el-form-item label="自动同步">
          <el-switch v-model="form.auto_sync" />
          <span class="inline-tip">打开后会按下面的频率自动检查，无需手动点</span>
        </el-form-item>

        <el-form-item label="同步频率" v-if="form.auto_sync">
          <el-select v-model="form.schedule" style="width: 260px" placeholder="请选择同步频率">
            <el-option
              v-for="opt in FREQUENCY_OPTIONS"
              :key="opt.value"
              :label="opt.label"
              :value="opt.value"
            />
          </el-select>
        </el-form-item>

        <el-form-item label="保存到">
          <el-input v-model="form.save_dir_prefix" style="width: 260px" />
          <div class="form-tip">每个订阅会保存到「保存位置 / 订阅名」文件夹</div>
        </el-form-item>

        <el-form-item label="失效提醒">
          <el-switch v-model="form.alert_enabled" />
          <div class="form-tip">接口失效或登录过期时，通过你已配置的通知方式（含钉钉）提醒你</div>
        </el-form-item>

        <el-form-item label="高级设置">
          <el-button link type="primary" @click="showAdvanced = !showAdvanced">
            {{ showAdvanced ? '收起' : '展开' }}
          </el-button>
          <div v-show="showAdvanced" class="advanced-box">
            <el-form-item label="最大翻页数">
              <el-input-number v-model="form.max_pages" :min="1" :max="100" />
            </el-form-item>
            <el-form-item label="会话 ID">
              <el-input v-model="form.session_id" style="width: 320px" placeholder="留空自动识别，一般无需填写" />
            </el-form-item>
          </div>
        </el-form-item>
      </el-form>

      <div class="hero-actions">
        <el-button type="primary" size="large" :loading="saving" @click="saveSettings">保存设置</el-button>
        <el-button size="large" :loading="previewing" @click="handlePreview">预览订阅</el-button>
        <el-button type="success" size="large" :loading="syncing" @click="handleSync">立即同步</el-button>
      </div>
    </el-card>

    <!-- 识别结果 -->
    <el-card v-if="previewItems.length" class="list-card">
      <template #header>
        <div class="card-header">
          <span>识别到的订阅（共 {{ previewItems.length }} 个）</span>
          <el-tag type="success">待新增 {{ newCount }}</el-tag>
          <el-tag type="info">已存在 {{ skipCount }}</el-tag>
        </div>
      </template>
      <el-table :data="previewItems" size="small" style="width: 100%">
        <el-table-column prop="name" label="订阅名称" min-width="150" show-overflow-tooltip />
        <el-table-column prop="url" label="分享链接" min-width="260" show-overflow-tooltip />
        <el-table-column label="状态" width="100">
          <template #default="{ row }">
            <el-tag v-if="row.status === 'added'" type="success" size="small">已新增</el-tag>
            <el-tag v-else-if="row.status === 'new'" size="small">待新增</el-tag>
            <el-tag v-else-if="row.status === 'skipped'" type="info" size="small">已存在</el-tag>
            <el-tag v-else type="danger" size="small">失败</el-tag>
          </template>
        </el-table-column>
      </el-table>
    </el-card>

    <!-- 同步日志 -->
    <el-card class="log-card">
      <template #header>
        <div class="card-header">
          <span>同步日志</span>
          <el-button link type="primary" :loading="logsLoading" @click="loadLogs">刷新</el-button>
        </div>
      </template>
      <div class="subscription-logs" v-loading="logsLoading">
        <div v-if="logs.length === 0" class="log-empty">暂无日志</div>
        <div
          v-for="(log, index) in logs"
          :key="index"
          class="log-item"
          :class="{ 'log-error': log.level === 'ERROR' }"
        >
          <span class="log-time">{{ log.timestamp }}</span>
          <span class="log-msg">{{ log.message }}</span>
        </div>
      </div>
    </el-card>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted } from 'vue'
import { ElMessage } from 'element-plus'
import { storeToRefs } from 'pinia'
import { useConfigStore } from '@/stores'
import { apiService } from '@/services'

const configStore = useConfigStore()
const { config, saving } = storeToRefs(configStore)

// 同步频率选项：对小白友好，直接用中文描述，内部自动转换成定时表达式
const FREQUENCY_OPTIONS = [
  { label: '每 10 分钟', value: '*/10 * * * *' },
  { label: '每 30 分钟', value: '*/30 * * * *' },
  { label: '每小时', value: '0 * * * *' },
  { label: '每 3 小时', value: '0 */3 * * *' },
  { label: '每 6 小时', value: '0 */6 * * *' },
  { label: '每 12 小时', value: '0 */12 * * *' },
  { label: '每天一次（凌晨 3 点）', value: '0 3 * * *' }
]

// 订阅同步配置
const form = reactive({
  enabled: false,
  auto_sync: false,
  schedule: '0 * * * *',
  session_id: '',
  save_dir_prefix: '/订阅',
  max_pages: 10,
  alert_enabled: true
})

const showAdvanced = ref(false)
const previewing = ref(false)
const syncing = ref(false)
const logsLoading = ref(false)
const previewItems = ref<Array<{ name: string; url: string; status: string }>>([])
const logs = ref<Array<{ timestamp: string; level: string; message: string }>>([])

const newCount = computed(() => previewItems.value.filter(i => i.status === 'new' || i.status === 'added').length)
const skipCount = computed(() => previewItems.value.filter(i => i.status === 'skipped').length)

// 从配置初始化表单
const initForm = () => {
  const sub = config.value.subscription
  if (!sub) return
  form.enabled = sub.enabled ?? false
  form.auto_sync = sub.auto_sync ?? false
  form.schedule = sub.schedule ?? '0 * * * *'
  form.session_id = sub.session_id ?? ''
  form.save_dir_prefix = sub.save_dir_prefix ?? '/订阅'
  form.max_pages = sub.max_pages ?? 10
  form.alert_enabled = sub.alert_enabled ?? true
}

// 保存设置
const saveSettings = async () => {
  try {
    await configStore.updateSubscriptionConfig({ ...form })
    ElMessage.success('订阅同步设置已保存')
  } catch (error) {
    ElMessage.error(`保存失败：${error}`)
  }
}

// 预览订阅（不创建任务）
const handlePreview = async () => {
  previewing.value = true
  try {
    const res = await apiService.previewSubscriptions()
    if (res.success) {
      previewItems.value = res.items || []
      ElMessage.success(`识别到 ${previewItems.value.length} 个订阅：待新增 ${newCount.value}，已存在 ${skipCount.value}`)
    } else {
      ElMessage.error(res.message || '预览失败')
    }
    await loadLogs()
  } catch (error) {
    ElMessage.error(`预览失败：${error}`)
  } finally {
    previewing.value = false
  }
}

// 立即同步
const handleSync = async () => {
  syncing.value = true
  try {
    const res = await apiService.syncSubscriptions()
    if (res.success) {
      previewItems.value = res.items || []
      ElMessage.success(`同步完成：新增 ${res.added ?? 0}，跳过 ${res.skipped ?? 0}，失败 ${res.failed ?? 0}`)
    } else {
      ElMessage.error(res.message || '同步失败')
    }
    await loadLogs()
  } catch (error) {
    ElMessage.error(`同步失败：${error}`)
  } finally {
    syncing.value = false
  }
}

// 加载同步日志
const loadLogs = async () => {
  logsLoading.value = true
  try {
    const res = await apiService.getSubscriptionLogs()
    if (res.success) {
      logs.value = res.logs || res.data?.logs || []
    }
  } catch (error) {
    console.error('获取订阅日志失败:', error)
  } finally {
    logsLoading.value = false
  }
}

onMounted(async () => {
  await configStore.fetchConfig()
  initForm()
  loadLogs()
})
</script>

<style scoped>
.subscriptions-view {
  padding: 24px;
  min-height: 100vh;
  background-color: #f5f5f5;
}

.page-header {
  margin-bottom: 20px;
}

.page-title {
  font-size: 24px;
  font-weight: 600;
  color: #333;
  margin: 0 0 8px;
}

.page-desc {
  color: #909399;
  font-size: 13px;
  margin: 0;
}

.hero-card {
  border-radius: 8px;
  border: 1px solid #e4e7ed;
  margin-bottom: 20px;
}

.hero-top {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 12px;
  padding-bottom: 16px;
  margin-bottom: 16px;
  border-bottom: 1px dashed #ebeef5;
}

.hero-switch {
  display: flex;
  align-items: center;
  gap: 12px;
}

.hero-label {
  font-size: 18px;
  font-weight: 600;
  color: #303133;
}

.hero-hint {
  color: #909399;
  font-size: 13px;
}

.hero-hint.on {
  color: #67c23a;
}

.hero-form {
  max-width: 720px;
}

.hero-actions {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  margin-top: 12px;
}

.inline-tip,
.form-tip {
  color: #909399;
  font-size: 12px;
  margin-left: 10px;
}

.form-tip {
  display: block;
  margin-left: 0;
  margin-top: 4px;
}

.advanced-box {
  margin-top: 8px;
}

.card-header {
  display: flex;
  align-items: center;
  gap: 10px;
}

.list-card,
.log-card {
  border-radius: 8px;
  margin-bottom: 20px;
}

.subscription-logs {
  max-height: 320px;
  overflow-y: auto;
  background: #fafafa;
  border: 1px solid #ebeef5;
  border-radius: 4px;
  padding: 12px;
}

.log-item {
  display: flex;
  gap: 12px;
  font-size: 13px;
  line-height: 1.8;
  border-bottom: 1px dashed #eee;
}

.log-item:last-child {
  border-bottom: none;
}

.log-item.log-error .log-msg {
  color: #f56c6c;
}

.log-time {
  color: #909399;
  flex-shrink: 0;
}

.log-empty {
  color: #909399;
  font-size: 13px;
}
</style>
