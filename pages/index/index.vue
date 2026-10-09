<template>
  <view class="page">

    <!-- 每个类型一个区块 -->
    <view v-for="group in groups" :key="group.type" class="group">

      <!-- 类型标题 -->
      <view class="group-header">
        <text class="group-title">{{ group.type }}</text>
        <text class="group-count">{{ group.items.length }} 个</text>
      </view>

      <!-- 账号列表：默认只显示第一个 -->
      <view
        v-for="(item, idx) in visibleItems(group)"
        :key="item.id"
        class="item-card"
      >
        <view class="item-line">
          <text class="item-title">{{ item.title }}</text>
          <text class="item-del" @click="removeItem(group.type, item.id)">删除</text>
        </view>

        <view class="item-line">
          <text class="item-username">{{ item.username }}</text>
        </view>

        <view class="item-line">
          <text class="item-pwd">
            {{ showMap[item.id] ? decode(item.password) : '••••••••' }}
          </text>
        </view>

        <view class="item-actions">
          <view class="btn" @click="copyUsername(item.username)">复制账号</view>
          <view class="btn" @click="toggleShow(item.id)">
            {{ showMap[item.id] ? '隐藏' : '显示' }}
          </view>
          <view class="btn" @click="editItem(group.type, item)">编辑</view>
        </view>
      </view>

      <!-- 展开/收起 -->
      <view
        v-if="group.items.length > 1"
        class="toggle"
        @click="toggleExpand(group.type)"
      >
        <text v-if="!expanded[group.type]">
          还有 {{ group.items.length - 1 }} 个账号　展开 ▾
        </text>
        <text v-else>收起 ▴</text>
      </view>

    </view>

    <view v-if="groups.length === 0" class="empty">
      还没有账号，点右下角 + 添加
    </view>

    <!-- 悬浮 + -->
    <view class="fab" @click="openAdd">＋</view>

    <!-- 新增/编辑弹窗 -->
    <view v-if="showEditor" class="mask" @click="closeEditor">
      <view class="editor" @click.stop>
        <view class="editor-title">{{ editingId ? '编辑账号' : '新增账号' }}</view>

        <input class="input" v-model="form.type" placeholder="类型（如 QQ、微信）" />
        <input class="input" v-model="form.title" placeholder="标题（如 大号）" />
        <input class="input" v-model="form.username" placeholder="账号" />
        <input class="input" v-model="form.password" placeholder="密码" />

        <view class="editor-actions">
          <view class="btn" @click="closeEditor">取消</view>
          <view class="btn primary" @click="saveItem">保存</view>
        </view>
      </view>
    </view>

  </view>
</template>

<script setup>
import { ref, reactive } from 'vue'

const groups = ref([])
const showMap = ref({})          // 每个条目密码是否显示
const expanded = reactive({})    // 每个类型是否展开
const showEditor = ref(false)
const editingId = ref(null)
const editingType = ref('')      // 编辑时记录原类型，防止改类型出错

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

/* ---------- 你的骚套路 ---------- */
function encode(str) {
  if (!str) return ''
  return btoa(str).split('').reverse().join('')
}

function decode(str) {
  if (!str) return ''
  try {
    return atob(str.split('').reverse().join(''))
  } catch (e) {
    return '解密失败'
  }
}

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
  showMap.value[id] = !showMap.value[id]
}

function copyUsername(username) {
  uni.setClipboardData({
    data: username,
    success: () => uni.showToast({ title: '已复制账号', icon: 'none' })
  })
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
  padding-bottom: 180rpx;
  background: #f5f6f8;
  min-height: 100vh;
}

/* ---------- 类型区块 ---------- */
.group {
  background: #fff;
  border-radius: 20rpx;
  margin-bottom: 28rpx;
  overflow: hidden;
  box-shadow: 0 4rpx 12rpx rgba(0, 0, 0, 0.05);
}

.group-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24rpx 28rpx;
  border-bottom: 1rpx solid #f0f0f0;
}

.group-title {
  font-size: 34rpx;
  font-weight: 600;
  color: #222;
}

.group-count {
  font-size: 26rpx;
  color: #999;
}

/* ---------- 账号卡片 ---------- */
.item-card {
  padding: 24rpx 28rpx;
  border-bottom: 1rpx solid #f5f5f5;
}

.item-card:last-child {
  border-bottom: none;
}

.item-line {
  margin-bottom: 10rpx;
}

.item-title {
  font-size: 30rpx;
  font-weight: 600;
  color: #222;
  margin-right: 16rpx;
}

.item-del {
  font-size: 24rpx;
  color: #ef4444;
  float: right;
}

.item-username {
  font-size: 30rpx;
  color: #333;
}

.item-pwd {
  font-size: 30rpx;
  color: #666;
  letter-spacing: 2rpx;
}

.item-actions {
  display: flex;
  gap: 16rpx;
  margin-top: 18rpx;
}

.btn {
  flex: 1;
  text-align: center;
  padding: 14rpx 0;
  font-size: 26rpx;
  border-radius: 12rpx;
  background: #f1f3f5;
  color: #333;
}

.btn.primary {
  background: #3b82f6;
  color: #fff;
}

/* ---------- 展开/收起 ---------- */
.toggle {
  text-align: center;
  padding: 20rpx 0;
  font-size: 26rpx;
  color: #3b82f6;
  background: #fafbfc;
}

/* ---------- 悬浮按钮 ---------- */
.fab {
  position: fixed;
  right: 40rpx;
  bottom: 60rpx;
  width: 110rpx;
  height: 110rpx;
  border-radius: 50%;
  background: #3b82f6;
  color: #fff;
  font-size: 60rpx;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 8rpx 20rpx rgba(59, 130, 246, 0.4);
}

/* ---------- 弹窗 ---------- */
.mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 99;
}

.editor {
  width: 600rpx;
  background: #fff;
  border-radius: 24rpx;
  padding: 40rpx;
}

.editor-title {
  font-size: 34rpx;
  font-weight: 600;
  text-align: center;
  margin-bottom: 30rpx;
}

.input {
  background: #f5f6f8;
  border-radius: 12rpx;
  padding: 20rpx 24rpx;
  font-size: 30rpx;
  margin-bottom: 20rpx;
}

.editor-actions {
  display: flex;
  gap: 20rpx;
  margin-top: 10rpx;
}

.empty {
  text-align: center;
  color: #999;
  font-size: 28rpx;
  margin-top: 120rpx;
}
</style>
