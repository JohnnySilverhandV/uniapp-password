# WORKBUDDY 实战 06：密码本毛玻璃改造与安卓键盘修复

> 更新点（v3，2026-11-04）：新增导入/导出（换机迁移）；v2：托管撤回，本地 APP 不托管；v1：整体改造。

## 〇、导入/导出（换机迁移，v3 新增）

方案：**剪贴板直通**，不碰文件系统，App/H5 两端逻辑一致。

- 导出：全量 JSON `{app:'pwd-vault', ver:1, time, count, groups}` 复制到剪贴板，用户经微信/备忘录任意渠道带到新手机
- 导入：粘贴 JSON → 校验（`JSON.parse` + groups 数组结构 + 逐组过滤）→ 三选一弹窗：合并导入（按 id 去重追加）/ 覆盖导入 / 取消
- node 实测通过：round-trip、同 id 去重、坏 JSON 拦截、非导出格式拦截
- 交互坑：H5 别用 `confirm()` 区分「合并/覆盖」——用户点取消想放弃，语义却变成覆盖，会误删数据。三按钮弹窗才安全

## 一、任务与交付

- 项目：`uniapp-password`（Vue 3 单页 uni-app 密码管理器，Gitee：johnny-silver-hand123/uniapp-password）
- 需求：毛玻璃效果 + 更好的交互与视觉 + 解决安卓键盘挡住编辑弹窗
- 交付形态两份（延续笔记 02 的双版本策略）：
  - uni-app 源码 `pages/index/index.vue` → HBuilderX 打 APK
  - 单文件网页版 `offline-app/index.html` → 零依赖，浏览器直接打开（仅本地实测用）
- ~~托管链接（H5 版）~~ **已撤销（2026-11-04）**：本项目是本地 APP，按笔记 05 v2 规则不托管；H5 版只作本地验证工具
- 数据互通：localStorage / Storage 键 `pwd_groups`，编解码方案两端一致

## 二、安卓键盘遮挡：根治方案（本次核心）

原实现的问题：编辑弹窗是 `position: fixed; inset: 0; align-items: center` 屏幕居中。安卓键盘弹出时页面视口不重排，键盘直接盖住下半屏——输入框在弹窗下半部分，全被挡住。

两级修复，缺一不可：

1. **pages.json 配 adjustResize**（根治视口不重排）：
```json
"style": {
  "app-plus": { "softinputMode": "adjustResize", "bounce": "none" }
}
```
2. **弹窗从居中改底部抽屉**：`align-items: flex-end` + `border-radius: 32rpx 32rpx 0 0` + `translateY(100%)→0` 入场动画 + 底部留 `env(safe-area-inset-bottom)`。键盘弹出时页面被压缩，抽屉贴底自然浮在键盘上方，不再依赖任何 JS 计算。

H5 端同构实现：移动端浏览器键盘同样会压缩视口，底部抽屉方案在 H5 上天然正确，两端一套逻辑。

保险措施：所有 input 加 `:cursor-spacing="20"`。

## 三、毛玻璃：直接复用笔记 02 的配方，零试错

笔记 02 的结论「毛玻璃的前提是背后有可虚化的东西」直接照做：

- 深色渐变底（`#0b1020 → #141b33`）+ 三层彩色光斑（蓝 #2563eb / 粉 #db2777 / 紫 #7c3aed），`filter: blur(80rpx)`，各 18/22/26s 错周期漂移
- 卡片：`rgba(255,255,255,0.08)` + `backdrop-filter: blur(24rpx) saturate(160%)` + 1rpx 白色半透明描边
- `position: fixed` 光斑层 + 内容层 `position: relative`，保证 blur 采样到光斑

深色主题下玻璃卡片的对比度经验：白玻璃透明度 0.08 够隐约、描边 0.15 出轮廓、正文文字用 `rgba(255,255,255,0.65~0.9)` 分层级。

## 四、交互升级清单

| 功能 | 实现 |
|---|---|
| 搜索过滤 | computed 过滤 type/title/username，H5 用同一逻辑 |
| 复制密码 | 原来只有复制账号；密码管理器没有复制密码说不过去 |
| 密码自动隐藏 | 显示后 10 秒 `setTimeout` 自动收回（防偷窥），`clearTimeout` 处理重复点击 |
| 🎲 强密码生成 | 16 位，剔除易混淆字符（0/O/1/l/I），`crypto.getRandomValues` 优先、`Math.random` 兜底 |
| 按压反馈 | uni-app 用 `hover-class="btn-press"`（scale 0.95 + 降透明），H5 用 `:active` |
| 入场动画 | 卡片 `fadeUp` + 按 idx 交错延迟 0.06s |
| FAB | 渐变玻璃 + 按压缩放旋转 |

## 五、一个顺手修掉的老 bug：中文密码

原编解码 `btoa(str)` 对非 ASCII 字符直接抛异常——存中文密码必崩。修复：`btoa(unescape(encodeURIComponent(str)))`，解码对应 `decodeURIComponent(escape(atob(...)))`。**旧 ASCII 数据在新解码下结果不变，天然兼容**（node 实测三种 case 通过：中文 round-trip / 旧数据解码 / 非法数据兜底）。

## 六、复用与验证

- 笔记 02 结论全部成立，本次没有踩新坑：svg/rAF 一开始就没用；毛玻璃先铺光斑一次到位
- H5 版编解码用 node 单测验证（纯函数可直接 eval）
- 浏览器工具截图超 10MiB 限制、DOM 快照返回空——这个环境的浏览器验证通道不可靠，纯逻辑验证走 node 更快

## 七、待办（需用户确认）

- git 已本地提交两个 commit，**未推送**（远端目前只挂了 Gitee，双推 GitHub 待配）
- App 版需 HBuilderX 重新打包 APK 才能真机验证键盘修复效果
