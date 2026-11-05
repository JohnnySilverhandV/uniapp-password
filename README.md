# PasswordVault（你妈本👀）

一个赛博玻璃风格的个人密码管理器，基于 uni-app（Vue 3）单页应用，一套代码双端交付：

- **App 端**：`pages/index/index.vue`，HBuilderX 云打包出 APK，日常使用
- **H5 端**：`offline-app/index.html`，零依赖单文件，浏览器直接打开，打包前快速验证交互

两端数据互通（存储键 `pwd_groups`，编码方案一致），换机迁移无缝衔接。

> 📱 **Android 安装包 v1.1.0**：[Gitee 下载（国内直连，推荐）](https://gitee.com/johnny-silver-hand123/uniapp-password/releases/download/v1.1.0/PasswordVault-v1.1.0.apk) ｜ [GitHub 下载](https://github.com/JohnnySilverhandV/uniapp-password/releases/download/v1.1.0/PasswordVault-v1.1.0.apk)
>
> 🌐 **网页版（H5）**：[offline-app/index.html](offline-app/index.html) — 单文件、零依赖，浏览器直接打开

## 源码仓库

| 平台 | 地址 |
|---|---|
| GitHub | https://github.com/JohnnySilverhandV/uniapp-password |
| Gitee（国内直连） | https://gitee.com/johnny-silver-hand123/uniapp-password |

## 功能

- **毛玻璃视觉**：深色渐变底 + 三层彩色光斑，卡片 / 搜索栏 / 弹窗全玻璃质感，卡片交错入场动画
- **安卓键盘修复**：`softinputMode: adjustResize` + 底部抽屉弹窗，键盘不再遮挡输入框
- **导入 / 导出**：剪贴板直通换机迁移，合并（按 id 去重）/ 覆盖两种导入方式
- **搜索过滤**：按类型 / 标题 / 账号实时过滤
- **密码安全细节**：显示 10 秒自动隐藏、16 位强密码生成（剔除易混字符）、复制账号 / 密码
- **中文密码**：编解码带 UTF-8 包装，旧 ASCII 数据无损兼容

## 构建

1. HBuilderX 打开工程，云打包（Android）
2. 产物在 `unpackage/release/apk/`

## 换机迁移

```
旧手机：📤 导出（JSON 复制到剪贴板）
   ↓ 通过微信 / 备忘录等任意渠道带到新手机
新手机：📥 导入 → 粘贴 → 合并导入
```

## 技术备忘

- App 端模板不支持 `<svg>`（编译期被丢弃，静默失效），视觉元素用 CSS / view 实现
- App 端避免 `requestAnimationFrame` 做关键动画，`setInterval` 更可靠
- 编解码用 `btoa(unescape(encodeURIComponent(str)))` 包装，纯 ASCII 旧数据天然兼容

## 更新日志

### v1.1.0

- 毛玻璃视觉重构（光斑 + 玻璃卡片 + 入场动画）
- 修复安卓键盘遮挡编辑弹窗
- 新增导入 / 导出（换机迁移）
- 修复中文密码存储崩溃
- 新增搜索、复制密码、密码自动隐藏、强密码生成

### v1.0.0

- 首个版本：分组管理账号，支持复制 / 显隐 / 编辑 / 删除
