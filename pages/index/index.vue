<template>
  <view class="page">

    <!-- 背景光斑：毛玻璃的前提是背后有东西可虚化 -->
    <view class="bg">
      <view class="blob b1"></view>
      <view class="blob b2"></view>
      <view class="blob b3"></view>
    </view>

    <!-- 顶部标题区 -->
    <view class="hero glass">
      <view class="hero-title-row">
        <text class="hero-title">🔐 你妈本</text>
        <text class="hero-count">{{ totalCount }} 条</text>
      </view>
      <view class="search-bar">
        <text class="search-icon">🔍</text>
        <input
          class="search-input"
          v-model="keyword"
          placeholder="搜索类型 / 标题 / 账号"
          placeholder-class="ph"
          confirm-type="search"
        />
        <text v-if="keyword" class="search-clear" @click="keyword = ''">✕</text>
      </view>
    </view>

    <!-- 每个类型一个区块 -->
    <view v-for="(group, gi) in filteredGroups" :key="group.type" class="group glass">

      <!-- 类型标题 -->
      <view class="group-header">
        <view class="group-badge">{{ group.type.slice(0, 1) }}</view>
        <text class="group-title">{{ group.type }}</text>
        <text class="group-count">{{ group.items.length }} 个</text>
      </view>

      <!-- 账号列表：默认只显示第一个 -->
      <view
        v-for="(item, idx) in visibleItems(group)"
        :key="item.id"
        class="item-card"
        :style="{ animationDelay: (idx * 0.06) + 's' }"
      >
        <view class="item-line">
          <text class="item-title">{{ item.title }}</text>
          <text class="item-del" @click="removeItem(group.type, item.id)">删除</text>
        </view>

        <view class="item-line" @click="copyUsername(item.username)">
          <text class="item-username">{{ item.username }}</text>
        </view>

        <view class="item-line pwd-line" @click="toggleShow(item.id)">
          <text class="item-pwd" :class="{ revealed: showMap[item.id] }">
            {{ showMap[item.id] ? decode(item.password) : '• • • • • • • •' }}
          </text>
          <text class="pwd-eye">{{ showMap[item.id] ? '🙈' : '👁️' }}</text>
        </view>

        <view class="item-actions">
          <view class="btn" hover-class="btn-press" @click="copyUsername(item.username)">复制账号</view>
          <view class="btn" hover-class="btn-press" @click="copyPassword(item)">复制密码</view>
          <view class="btn" hover-class="btn-press" @click="editItem(group.type, item)">编辑</view>
        </view>
      </view>

      <!-- 展开/收起 -->
      <view
        v-if="group.items.length > 1"
        class="toggle"
        hover-class="btn-press"
        @click="toggleExpand(group.type)"
      >
        <text v-if="!expanded[group.type]">
          还有 {{ group.items.length - 1 }} 个账号　展开 ▾
        </text>
        <text v-else>收起 ▴</text>
      </view>

    </view>

    <view v-if="filteredGroups.length === 0" class="empty">
      <text class="empty-icon">🗄️</text>
      <text class="empty-text">{{ keyword ? '没有匹配的账号' : '还没有账号，点右下角 + 添加' }}</text>
    </view>

    <!-- 悬浮 + -->
    <view class="fab" hover-class="fab-press" @click="openAdd">＋</view>

    <!-- 新增/编辑弹窗：底部抽屉，配合 adjustResize 不会被键盘挡住 -->
    <view v-if="showEditor" class="mask" @click="closeEditor">
      <view class="editor glass-strong" @click.stop>
        <view class="editor-grip"></view>
        <view class="editor-title">{{ editingId ? '编辑账号' : '新增账号' }}</view>

        <view class="field">
          <text class="field-label">类型</text>
          <input class="input" v-model="form.type" placeholder="如 QQ、微信" placeholder-class="ph" :cursor-spacing="20" />
        </view>
        <view class="field">
          <text class="field-label">标题</text>
          <input class="input" v-model="form.title" placeholder="如 大号" placeholder-class="ph" :cursor-spacing="20" />
        </view>
        <view class="field">
          <text class="field-label">账号</text>
          <input class="input" v-model="form.username" placeholder="用户名 / 手机 / 邮箱" placeholder-class="ph" :cursor-spacing="20" />
        </view>
        <view class="field">
          <text class="field-label">密码</text>
          <view class="pwd-input-row">
            <input class="input pwd-input" v-model="form.password" password placeholder="输入或点 🎲 生成" placeholder-class="ph" :cursor-spacing="20" />
            <view class="dice" hover-class="btn-press" @click="genPassword">🎲</view>
          </view>
        </view>

        <view class="editor-actions">
          <view class="btn ghost" hover-class="btn-press" @click="closeEditor">取消</view>
          <view class="btn primary" hover-class="btn-press" @click="saveItem">保存</view>
        </view>
        <view class="safe-bottom"></view>
      </view>
    </view>

  </view>
</template>

<script setup>
import { ref, reactive, computed } from 'vue'

const groups = ref([])
const showMap = ref({})          // 每个条目密码是否显示
const expanded = reactive({})    // 每个类型是否展开
const showEditor = ref(false)
const editingId = ref(null)
const editingType = ref('')      // 编辑时记录原类型，防止改类型出错
const keyword = ref('')

// 自动隐藏密码的定时器（App 端 setTimeout 比 rAF 可靠）
const hideTimers = {}

const form = ref({
  type: '',
  title: '',
  username: '',
  password: ''
})

/* ---------- 读写 ---------- */
function load() {
  const data = uni.getStorageSync('pwd_groups')
  groups.value = data ? JSON.parse(data) : []
}

function save() {
  uni.setStorageSync('pwd_groups', JSON.stringify(groups.value))
}

/* ---------- 编解码（与离线 H5 版互通） ---------- */
function encode(str) {
  if (!str) return ''
  return btoa(unescape(encodeURIComponent(str))).split('').reverse().join('')
}

function decode(str) {
  if (!str) return ''
  try {
    return decodeURIComponent(escape(atob(str.split('').reverse().join(''))))
  } catch (e) {
    return '解密失败'
  }
}

/* ---------- 搜索过滤 ---------- */
const filteredGroups = computed(() => {
  const kw = keyword.value.trim().toLowerCase()
  if (!kw) return groups.value
  return groups.value
    .map(g => ({
      ...g,
      items: g.items.filter(i =>
        g.type.toLowerCase().includes(kw) ||
        i.title.toLowerCase().includes(kw) ||
        i.username.toLowerCase().includes(kw)
      )
    }))
    .filter(g => g.items.length > 0)
})

const totalCount = computed(() =>
  groups.value.reduce((sum, g) => sum + g.items.length, 0)
)

/* ---------- 每个类型显示哪些条目 ---------- */
function visibleItems(group) {
  if (expanded[group.type]) return group.items
  return group.items.slice(0, 1)   // 默认只显示第一个
}

function toggleExpand(type) {
  expanded[type] = !expanded[type]
}

/* ---------- 交互 ---------- */
function toggleShow(id) {
  const next = !showMap.value[id]
  showMap.value[id] = next
  // 显示 10 秒后自动隐藏，防止密码一直暴露在屏幕上
  if (hideTimers[id]) clearTimeout(hideTimers[id])
  if (next) {
    hideTimers[id] = setTimeout(() => {
      showMap.value[id] = false
    }, 10000)
  }
}

function copyText(text, tip) {
  uni.setClipboardData({
    data: text,
    success: () => uni.showToast({ title: tip, icon: 'none' })
  })
}

function copyUsername(username) {
  copyText(username, '已复制账号')
}

function copyPassword(item) {
  copyText(decode(item.password), '已复制密码')
}

function genPassword() {
  const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZabcdefghjkmnpqrstuvwxyz23456789!@#$%&*'
  let pwd = ''
  const arr = new Uint32Array(16)
  if (typeof crypto !== 'undefined' && crypto.getRandomValues) {
    crypto.getRandomValues(arr)
    for (let i = 0; i < 16; i++) pwd += chars[arr[i] % chars.length]
  } else {
    for (let i = 0; i < 16; i++) pwd += chars[Math.floor(Math.random() * chars.length)]
  }
  form.value.password = pwd
}

/* ---------- 新增 / 编辑 ---------- */
function openAdd() {
  editingId.value = null
  editingType.value = ''
  form.value = { type: '', title: '', username: '', password: '' }
  showEditor.value = true
}

function editItem(type, item) {
  editingId.value = item.id
  editingType.value = type
  form.value = {
    type: type,
    title: item.title,
    username: item.username,
    password: decode(item.password)
  }
  showEditor.value = true
}

function closeEditor() {
  showEditor.value = false
}

function saveItem() {
  const f = form.value
  if (!f.type || !f.title || !f.username || !f.password) {
    uni.showToast({ title: '请填写完整', icon: 'none' })
    return
  }

  if (editingId.value) {
    // 编辑：先找到原类型，删掉旧数据
    const oldGroup = groups.value.find(g => g.type === editingType.value)
    if (oldGroup) {
      oldGroup.items = oldGroup.items.filter(i => i.id !== editingId.value)
      if (oldGroup.items.length === 0) {
        groups.value = groups.value.filter(g => g.type !== editingType.value)
      }
    }
    // 再按新类型插入
    pushItem(f)
  } else {
    pushItem(f)
  }

  save()
  showEditor.value = false
  uni.showToast({ title: '已保存', icon: 'none' })
}

function pushItem(f) {
  let g = groups.value.find(g => g.type === f.type)
  if (!g) {
    g = { type: f.type, items: [] }
    groups.value.push(g)
  }
  g.items.push({
    id: Date.now().toString() + Math.random().toString(36).slice(2, 6),
    title: f.title,
    username: f.username,
    password: encode(f.password)
  })
}

function removeItem(type, id) {
  uni.showModal({
    title: '确认删除',
    content: '删除后不可恢复',
    success: (res) => {
      if (!res.confirm) return
      const g = groups.value.find(g => g.type === type)
      if (!g) return
      g.items = g.items.filter(i => i.id !== id)
      if (g.items.length === 0) {
        groups.value = groups.value.filter(x => x.type !== type)
      }
      save()
    }
  })
}

load()
</script>

<style scoped>
.page {
  padding: 24rpx;
  padding-bottom: 200rpx;
  background: linear-gradient(160deg, #0b1020 0%, #141b33 55%, #0b1020 100%);
  min-height: 100vh;
  position: relative;
}

/* ---------- 背景光斑 ---------- */
.bg {
  position: fixed;
  inset: 0;
  z-index: 0;
  overflow: hidden;
  pointer-events: none;
}

.blob {
  position: absolute;
  border-radius: 50%;
  filter: blur(80rpx);
  opacity: 0.5;
}

.b1 {
  width: 520rpx;
  height: 520rpx;
  background: #2563eb;
  top: -120rpx;
  left: -140rpx;
  animation: float1 18s ease-in-out infinite alternate;
}

.b2 {
  width: 440rpx;
  height: 440rpx;
  background: #db2777;
  top: 30%;
  right: -160rpx;
  animation: float2 22s ease-in-out infinite alternate;
}

.b3 {
  width: 480rpx;
  height: 480rpx;
  background: #7c3aed;
  bottom: -140rpx;
  left: 15%;
  animation: float3 26s ease-in-out infinite alternate;
}

@keyframes float1 {
  from { transform: translate(0, 0) scale(1); }
  to   { transform: translate(120rpx, 160rpx) scale(1.15); }
}

@keyframes float2 {
  from { transform: translate(0, 0) scale(1.1); }
  to   { transform: translate(-140rpx, 120rpx) scale(0.9); }
}

@keyframes float3 {
  from { transform: translate(0, 0) scale(1); }
  to   { transform: translate(100rpx, -140rpx) scale(1.2); }
}

/* ---------- 玻璃质感 ---------- */
.glass {
  background: rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(24rpx) saturate(160%);
  -webkit-backdrop-filter: blur(24rpx) saturate(160%);
  border: 1rpx solid rgba(255, 255, 255, 0.15);
}

.glass-strong {
  background: rgba(30, 38, 66, 0.92);
  backdrop-filter: blur(32rpx) saturate(160%);
  -webkit-backdrop-filter: blur(32rpx) saturate(160%);
  border: 1rpx solid rgba(255, 255, 255, 0.18);
}

/* ---------- 顶部标题区 ---------- */
.hero {
  position: relative;
  z-index: 1;
  border-radius: 24rpx;
  padding: 28rpx;
  margin-bottom: 28rpx;
}

.hero-title-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 22rpx;
}

.hero-title {
  font-size: 40rpx;
  font-weight: 700;
  color: #f5f7ff;
  letter-spacing: 2rpx;
}

.hero-count {
  font-size: 26rpx;
  color: rgba(255, 255, 255, 0.55);
}

.search-bar {
  display: flex;
  align-items: center;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 16rpx;
  padding: 14rpx 20rpx;
}

.search-icon {
  font-size: 28rpx;
  margin-right: 12rpx;
}

.search-input {
  flex: 1;
  font-size: 28rpx;
  color: #f5f7ff;
}

.ph {
  color: rgba(255, 255, 255, 0.35);
}

.search-clear {
  font-size: 26rpx;
  color: rgba(255, 255, 255, 0.5);
  padding: 4rpx 8rpx;
}

/* ---------- 类型区块 ---------- */
.group {
  position: relative;
  z-index: 1;
  border-radius: 24rpx;
  margin-bottom: 28rpx;
  overflow: hidden;
}

.group-header {
  display: flex;
  align-items: center;
  padding: 24rpx 28rpx;
  border-bottom: 1rpx solid rgba(255, 255, 255, 0.1);
}

.group-badge {
  width: 52rpx;
  height: 52rpx;
  border-radius: 14rpx;
  background: linear-gradient(135deg, #3b82f6, #8b5cf6);
  color: #fff;
  font-size: 28rpx;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-right: 16rpx;
}

.group-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #f5f7ff;
  flex: 1;
}

.group-count {
  font-size: 26rpx;
  color: rgba(255, 255, 255, 0.45);
}

/* ---------- 账号卡片 ---------- */
.item-card {
  padding: 24rpx 28rpx;
  border-bottom: 1rpx solid rgba(255, 255, 255, 0.07);
  animation: fadeUp 0.4s cubic-bezier(0.22, 0.8, 0.36, 1) both;
}

.item-card:last-child {
  border-bottom: none;
}

@keyframes fadeUp {
  from {
    opacity: 0;
    transform: translateY(24rpx);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.item-line {
  margin-bottom: 10rpx;
}

.item-title {
  font-size: 30rpx;
  font-weight: 600;
  color: #f5f7ff;
  margin-right: 16rpx;
}

.item-del {
  font-size: 24rpx;
  color: #f87171;
  float: right;
  padding: 2rpx 8rpx;
}

.item-username {
  font-size: 28rpx;
  color: rgba(255, 255, 255, 0.65);
}

.pwd-line {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.item-pwd {
  font-size: 28rpx;
  color: rgba(255, 255, 255, 0.55);
  letter-spacing: 2rpx;
}

.item-pwd.revealed {
  color: #6ee7b7;
  animation: reveal 0.25s ease;
}

@keyframes reveal {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.pwd-eye {
  font-size: 28rpx;
  padding: 4rpx 8rpx;
}

.item-actions {
  display: flex;
  gap: 16rpx;
  margin-top: 18rpx;
}

/* ---------- 按钮 ---------- */
.btn {
  flex: 1;
  text-align: center;
  padding: 14rpx 0;
  font-size: 26rpx;
  border-radius: 14rpx;
  background: rgba(255, 255, 255, 0.1);
  border: 1rpx solid rgba(255, 255, 255, 0.12);
  color: rgba(255, 255, 255, 0.85);
  transition: transform 0.12s ease, opacity 0.12s ease;
}

.btn-press {
  transform: scale(0.95);
  opacity: 0.7;
}

.btn.primary {
  background: linear-gradient(135deg, #3b82f6, #8b5cf6);
  border: none;
  color: #fff;
  font-weight: 600;
}

.btn.ghost {
  background: rgba(255, 255, 255, 0.06);
}

/* ---------- 展开/收起 ---------- */
.toggle {
  text-align: center;
  padding: 20rpx 0;
  font-size: 26rpx;
  color: #93c5fd;
  background: rgba(255, 255, 255, 0.04);
}

/* ---------- 悬浮按钮 ---------- */
.fab {
  position: fixed;
  right: 40rpx;
  bottom: 60rpx;
  width: 112rpx;
  height: 112rpx;
  border-radius: 50%;
  background: linear-gradient(135deg, #3b82f6, #8b5cf6);
  color: #fff;
  font-size: 60rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 8rpx 32rpx rgba(99, 102, 241, 0.5);
  border: 1rpx solid rgba(255, 255, 255, 0.25);
  transition: transform 0.15s ease;
  z-index: 10;
}

.fab-press {
  transform: scale(0.88) rotate(90deg);
}

/* ---------- 弹窗：底部抽屉 ---------- */
.mask {
  position: fixed;
  inset: 0;
  background: rgba(5, 8, 18, 0.6);
  display: flex;
  align-items: flex-end;
  justify-content: center;
  z-index: 99;
  animation: maskIn 0.25s ease both;
}

@keyframes maskIn {
  from { opacity: 0; }
  to   { opacity: 1; }
}

.editor {
  width: 100%;
  border-radius: 32rpx 32rpx 0 0;
  padding: 16rpx 40rpx 40rpx;
  animation: sheetUp 0.3s cubic-bezier(0.22, 0.8, 0.36, 1) both;
}

@keyframes sheetUp {
  from { transform: translateY(100%); }
  to   { transform: translateY(0); }
}

.editor-grip {
  width: 72rpx;
  height: 8rpx;
  border-radius: 8rpx;
  background: rgba(255, 255, 255, 0.25);
  margin: 8rpx auto 24rpx;
}

.editor-title {
  font-size: 34rpx;
  font-weight: 600;
  text-align: center;
  color: #f5f7ff;
  margin-bottom: 30rpx;
}

.field {
  margin-bottom: 20rpx;
}

.field-label {
  display: block;
  font-size: 24rpx;
  color: rgba(255, 255, 255, 0.5);
  margin-bottom: 10rpx;
}

.input {
  background: rgba(255, 255, 255, 0.08);
  border: 1rpx solid rgba(255, 255, 255, 0.12);
  border-radius: 14rpx;
  padding: 20rpx 24rpx;
  font-size: 30rpx;
  color: #f5f7ff;
}

.pwd-input-row {
  display: flex;
  align-items: center;
  gap: 16rpx;
}

.pwd-input {
  flex: 1;
}

.dice {
  width: 80rpx;
  height: 80rpx;
  border-radius: 14rpx;
  background: rgba(255, 255, 255, 0.1);
  border: 1rpx solid rgba(255, 255, 255, 0.12);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 36rpx;
}

.editor-actions {
  display: flex;
  gap: 20rpx;
  margin-top: 28rpx;
}

.safe-bottom {
  height: env(safe-area-inset-bottom);
}

/* ---------- 空状态 ---------- */
.empty {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-top: 140rpx;
}

.empty-icon {
  font-size: 80rpx;
  margin-bottom: 20rpx;
  animation: bob 2.4s ease-in-out infinite;
}

@keyframes bob {
  0%, 100% { transform: translateY(0); }
  50%      { transform: translateY(-14rpx); }
}

.empty-text {
  color: rgba(255, 255, 255, 0.45);
  font-size: 28rpx;
}
</style>
