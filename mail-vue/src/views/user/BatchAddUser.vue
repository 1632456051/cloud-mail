<template>
  <Icon class="icon" icon="mdi:account-multiple-plus-outline" width="23" height="23" @click="open"/>
  <el-dialog v-model="show" title="批量添加用户" @closed="reset">
    <div class="batch-container">

      <div class="block">
        <div class="label">用户名生成方式</div>
        <el-radio-group v-model="mode">
          <el-radio value="random">随机批量生成</el-radio>
          <el-radio value="manual">手动指定</el-radio>
        </el-radio-group>
      </div>

      <template v-if="mode === 'random'">
        <div class="block">
          <div class="label">生成数量（1 - 200）</div>
          <el-input-number v-model="count" :min="1" :max="200" />
        </div>
        <div class="block">
          <div class="label">用户名前缀（可选，例如 user）</div>
          <el-input v-model="prefix" placeholder="留空则纯随机" />
        </div>
      </template>

      <template v-if="mode === 'manual'">
        <div class="block">
          <div class="label">用户名列表（每行一个，不含 @域名）</div>
          <el-input v-model="manualText" type="textarea" :rows="6" placeholder="alice&#10;bob&#10;charlie" />
        </div>
      </template>

      <div class="block">
        <div class="label">域名</div>
        <el-select v-model="domain" placeholder="选择域名">
          <el-option v-for="d in domainList" :key="d" :label="d" :value="d" />
        </el-select>
      </div>

      <div class="block">
        <div class="label">角色</div>
        <el-select v-model="roleType" placeholder="选择角色">
          <el-option v-for="r in roleList" :key="r.roleId" :label="r.name" :value="r.roleId" />
        </el-select>
      </div>

      <div class="block">
        <div class="label">密码</div>
        <el-radio-group v-model="pwdMode">
          <el-radio value="random">随机密码</el-radio>
          <el-radio value="custom">统一密码</el-radio>
        </el-radio-group>
      </div>

      <template v-if="pwdMode === 'random'">
        <div class="block">
          <div class="label">随机密码长度（≥ 6）</div>
          <el-input-number v-model="pwdLen" :min="6" :max="64" />
        </div>
        <div class="block checkbox-row">
          <el-checkbox v-model="useLetter">包含字母</el-checkbox>
          <el-checkbox v-model="useDigit">包含数字</el-checkbox>
          <el-checkbox v-model="useSymbol">包含符号</el-checkbox>
        </div>
      </template>

      <template v-else>
        <div class="block">
          <div class="label">统一密码（≥ 6 位）</div>
          <el-input v-model="customPwd" type="password" show-password placeholder="设置统一密码" />
        </div>
      </template>

      <div class="block checkbox-row">
        <el-checkbox v-model="saveToDesktop">生成后下载账号密码文件（CSV，可存到桌面）</el-checkbox>
      </div>

      <el-progress v-if="running" :percentage="progress" :stroke-width="14" />

      <div v-if="result" class="result" :class="result.fail ? 'result-warn' : 'result-ok'">
        成功 {{ result.success }} 个，失败 {{ result.fail }} 个
        <div v-if="result.emails && result.emails.length" class="errors">
          <div class="err-row ok-row">已创建（最多显示 10 个）：{{ result.emails.slice(0, 10).join('、') }}</div>
        </div>
        <div v-if="result.errors && result.errors.length" class="errors">
          <div v-for="(e, i) in result.errors" :key="i" class="err-row">{{ e.email }}：{{ e.msg }}</div>
        </div>
      </div>
    </div>

    <template #footer>
      <el-button @click="show = false">关闭</el-button>
      <el-button type="primary" :loading="running" @click="start">
        {{ running ? '生成中…' : '开始批量添加' }}
      </el-button>
    </template>
  </el-dialog>
</template>

<script setup>
import { computed, reactive, ref, watch } from 'vue'
import { Icon } from '@iconify/vue'
import { userAdd } from '@/request/user.js'
import { roleSelectUse } from '@/request/role.js'
import { useSettingStore } from '@/store/setting.js'
import { useUserStore } from '@/store/user.js'

const emit = defineEmits(['refresh'])

const settingStore = useSettingStore()
const userStore = useUserStore()
// computed：域名列表是异步加载的，这样保证加载完成后下拉框一定能拿到值
const domainList = computed(() => settingStore.domainList || [])
const roleList = reactive([])
roleSelectUse().then(list => {
  roleList.length = 0
  roleList.push(...list)
})

const show = ref(false)
const mode = ref('random')
const count = ref(10)
const prefix = ref('user')
const manualText = ref('')
const domain = ref('')
const roleType = ref(null)
const pwdMode = ref('random')
const pwdLen = ref(12)
const useLetter = ref(true)
const useDigit = ref(true)
const useSymbol = ref(false)
const customPwd = ref('')
const saveToDesktop = ref(false)

const running = ref(false)
const progress = ref(0)
const result = ref(null)

// 域名列表到位后自动选中第一个，避免用户忘记选域名
watch(domainList, (list) => {
  if (!domain.value && list && list.length) {
    domain.value = list[0]
  }
}, { immediate: true, deep: true })

function open() {
  show.value = true
}

const LETTERS = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
const DIGITS = '0123456789'
const SYMBOLS = '!@#$%^&*()-_=+[]{};:,.?'

function randomString(len, pool) {
  let s = ''
  for (let i = 0; i < len; i++) {
    s += pool[Math.floor(Math.random() * pool.length)]
  }
  return s
}

function genPassword() {
  if (pwdMode.value === 'custom') {
    return customPwd.value
  }
  let pool = ''
  if (useLetter.value) pool += LETTERS
  if (useDigit.value) pool += DIGITS
  if (useSymbol.value) pool += SYMBOLS
  if (!pool) pool = LETTERS + DIGITS
  return randomString(pwdLen.value, pool)
}

function buildList() {
  const list = []
  if (mode.value === 'random') {
    const n = Math.min(Math.max(parseInt(count.value) || 1, 1), 200)
    for (let i = 0; i < n; i++) {
      const local = (prefix.value || '') + randomString(8, LETTERS + DIGITS)
      list.push({ email: local + domain.value, password: genPassword() })
    }
  } else {
    const lines = manualText.value
      .split('\n')
      .map(s => s.trim())
      .filter(Boolean)
      .map(s => s.split('@')[0].trim())
      .filter(Boolean)
    for (const local of lines) {
      list.push({ email: local + domain.value, password: genPassword() })
    }
  }
  return list
}

function extractMsg(e) {
  try {
    if (typeof e === 'string') return e
    if (e?.response?.data?.msg) return e.response.data.msg
    if (e?.response?.data?.message) return e.response.data.message
    if (e?.msg) return e.msg
    if (e?.message) return e.message
  } catch (err) {
    // ignore
  }
  return '添加失败'
}

async function start() {
  if (running.value) return

  if (!domain.value) {
    ElMessage({ message: '请选择域名', type: 'error', plain: true })
    return
  }
  if (!roleType.value) {
    ElMessage({ message: '请选择角色', type: 'error', plain: true })
    return
  }
  if (pwdMode.value === 'custom') {
    if (!customPwd.value) {
      ElMessage({ message: '请填写统一密码', type: 'error', plain: true })
      return
    }
    if (customPwd.value.length < 6) {
      ElMessage({ message: '密码至少 6 位', type: 'error', plain: true })
      return
    }
  } else {
    if (pwdLen.value < 6) {
      ElMessage({ message: '密码长度至少 6 位', type: 'error', plain: true })
      return
    }
    if (!useLetter.value && !useDigit.value && !useSymbol.value) {
      ElMessage({ message: '请至少选择一种字符类型', type: 'error', plain: true })
      return
    }
  }

  const list = buildList()
  if (list.length === 0) {
    ElMessage({ message: '没有可添加的用户', type: 'error', plain: true })
    return
  }

  running.value = true
  progress.value = 0
  result.value = { success: 0, fail: 0, errors: [], emails: [] }
  const created = []

  for (let i = 0; i < list.length; i++) {
    const item = list[i]
    try {
      await userAdd({ email: item.email, password: item.password, type: roleType.value })
      result.value.success++
      result.value.emails.push(item.email)
      created.push(item)
    } catch (e) {
      result.value.fail++
      result.value.errors.push({ email: item.email, msg: extractMsg(e) })
    }
    progress.value = Math.floor(((i + 1) / list.length) * 100)
  }

  running.value = false

  if (saveToDesktop.value && created.length) {
    downloadCsv(created)
  }

  // 双保险刷新列表：既触发父组件的 @refresh，也驱动 userStore.refreshList（父组件里有 watch）
  emit('refresh')
  userStore.refreshList++

  ElMessage({
    message: `批量添加完成：成功 ${result.value.success}，失败 ${result.value.fail}`,
    type: result.value.fail ? 'warning' : 'success',
    plain: true
  })
}

function downloadCsv(list) {
  const header = '邮箱,密码\n'
  const body = list.map(u => `${u.email},${u.password}`).join('\n')
  const blob = new Blob(['\ufeff' + header + body], { type: 'text/csv;charset=utf-8;' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  const ts = new Date().toISOString().slice(0, 19).replace(/[:T]/g, '-')
  a.download = `mail-users-${ts}.csv`
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

function reset() {
  result.value = null
  progress.value = 0
}
</script>

<style scoped>
.icon {
  cursor: pointer;
}

.batch-container {
  display: grid;
  grid-template-columns: 1fr;
  gap: 14px;
}

.label {
  font-size: 13px;
  color: var(--el-text-color-regular);
  margin-bottom: 6px;
}

.block :deep(.el-select),
.block :deep(.el-input),
.block :deep(.el-input-number) {
  width: 100%;
}

.checkbox-row {
  display: flex;
  flex-wrap: wrap;
  gap: 14px;
}

.result {
  margin-top: 4px;
  padding: 10px 12px;
  border-radius: 6px;
  font-size: 13px;
  line-height: 1.6;
}

.result-ok {
  background: var(--el-color-success-light-9);
  color: var(--el-color-success);
}

.result-warn {
  background: var(--el-color-warning-light-9);
  color: var(--el-color-warning-dark-2);
}

.errors {
  margin-top: 6px;
  max-height: 160px;
  overflow: auto;
}

.err-row {
  font-size: 12px;
  color: var(--el-color-danger);
  word-break: break-all;
}

.ok-row {
  color: var(--el-color-success);
}
</style>
