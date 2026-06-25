# EXIF相片信息修复 小程序 — 项目主索引

> 最后更新: 2026-06-25  
> 项目代号: `exif-editor-miniapp`  
> 目标: 快速上线一款支持修复照片 EXIF 参数（设备、镜头、时间等）的微信小程序

---

## 零、工作区约定

本项目采用 **双区开发模式**：

| 区域 | 路径 | 用途 | 工具 |
|------|------|------|------|
| **文档区** | `exif-editor-miniapp/docs/`（WorkBuddy 工作区） | 需求、设计、进度等文档编写 | CodeBuddy |
| **代码区** | `C:\Users\admin\WeChatProjects\miniprogram-1` | 小程序代码开发与调试 | 微信开发者工具 |

CodeBuddy 处理文档类任务时写入文档区，处理编码任务时写入代码区。

---

## 一、项目概览

本项目是一款面向摄影爱好者与社交媒体用户的微信小程序，核心功能为：

1. **相册选图** — 从手机相册选择照片
2. **参数修复** — 修复设备信息、镜头信息、拍摄时间等 EXIF 元数据
3. **模板系统** — 预设常用模板，支持一键套用、批量处理与自定义保存
4. **相册保存** — 处理后的照片保存至系统相册，体验对标 iPhone 相册

---

## 二、文档导航

| 文档 | 路径 | 说明 |
|------|------|------|
| 📐 **编码规范** | [CODING_STANDARDS.md](./CODING_STANDARDS.md) | JS/WXML/WXSS 代码风格、命名约定、Git 规范 |
| 📋 **需求文档** | [REQUIREMENTS.md](./REQUIREMENTS.md) | 功能需求清单、用户故事、边界定义 |
| 🏗 **概要设计** | [TECH_OVERVIEW.md](./TECH_OVERVIEW.md) | 整体架构、模块划分、数据流、技术选型 |
| 🔧 **详细设计** | [TECH_DETAIL.md](./TECH_DETAIL.md) | 各模块的具体实现方案、接口定义、数据结构 |
| 📅 **项目进度** | [SCHEDULE.md](./SCHEDULE.md) | 里程碑 → 阶段任务 → 具体条目的分层计划 |

---

## 三、项目结构（规划）

```
exif-editor-miniapp/
├── app.js                        # 应用入口，全局生命周期
├── app.json                      # 全局配置（路由、窗口、tabBar）
├── app.wxss                      # 全局样式
├── project.config.json           # IDE 配置
├── sitemap.json                  # 搜索索引配置
├── pages/
│   ├── index/                    # 首页（相册选图入口 + 模板推荐）
│   ├── editor/                   # EXIF 编辑页（核心编辑界面）
│   ├── template/                 # 模板管理页
│   └── preview/                  # 预览与保存页
├── components/
│   ├── exif-form/                # EXIF 字段编辑表单组件
│   ├── template-card/            # 模板卡片组件
│   ├── image-picker/             # 图片选择器组件
│   └── save-panel/               # 保存面板组件
├── utils/
│   ├── request.js                # 统一网络请求封装
│   ├── exif.js                   # EXIF 数据解析与序列化
│   ├── image.js                  # 图片处理工具（压缩、Canvas 合成）
│   ├── storage.js               # 本地存储管理（模板数据）
│   └── auth.js                   # 微信登录与授权
├── services/
│   ├── exif-service.js           # EXIF 读写业务逻辑
│   └── template-service.js       # 模板 CRUD 业务逻辑
├── subpackages/
│   └── about/                    # 关于页 + 帮助（分包）
└── docs/                         # 项目文档（当前目录）
```

---

## 四、技术栈

| 层面 | 选型 | 说明 |
|------|------|------|
| 开发框架 | 微信原生框架 | 体积最小，启动最快 |
| UI 组件库 | TDesign Miniprogram | 腾讯官方出品，60+ 组件 |
| 样式方案 | WXSS + CSS Variables | 原生方案，无额外依赖 |
| 状态管理 | 全局 globalData + 页面级 | 轻量场景无需引入第三方 |
| 图片处理 | Canvas API + wx.getImageInfo | 客户端处理，减少服务端依赖 |
| 网络请求 | wx.request + Promise 封装 | 统一鉴权与错误处理 |

---

## 五、关键约束

- 主包大小 ≤ 2MB，分包总计 ≤ 20MB
- 所有网络请求必须 HTTPS，域名须在后台白名单注册
- EXIF 编辑涉及用户照片隐私，需明确授权后方可读取
- 保存到相册需 `scope.writePhotosAlbum` 授权
- 需要通过微信审核，不得包含违规内容

---

## 六、环境配置

| 环境 | AppID | 后端 API |
|------|-------|----------|
| 开发 | `wxc843a55dc151a088` | 暂无（客户端处理优先） |
| 体验 | `wxc843a55dc151a088` | 暂无 |
| 生产 | `wxc843a55dc151a088` | 暂无 |

## 七、Git 仓库

| 项目 | 地址 |
|------|------|
| GitHub | https://github.com/moshang149/exif-editor-miniapp |
| 本地代码区 | `C:\Users\admin\WeChatProjects\miniprogram-1` |
| 当前分支 | `main` |

---

## 八、当前状态（新会话必读）

| 项目 | 状态 |
|------|------|
| M1 阶段 | 脚手架 3/7 完成（AppID ✅ / TDesign ✅ / 项目结构 ✅），env.js/请求封装/域名注册待做 |
| UI 设计 | 未启动 |
| 核心开发 | 未启动，下一个待做项 = 首页页面开发 |
| 文档 | 全部 7 份就绪，随开发同步更新 |
| Git | 已推送 GitHub，每次修改后 commit+push |

**新会话开局建议顺序：** 先读本 INDEX.md → 读 MEMORY.md → 看 SCHEDULE.md 确认进度 → 开始编码任务

---

## 九、团队与分工（规划）

| 角色 | 职责 |
|------|------|
| 产品经理 | 需求定义、用户验证 |
| 前端开发 | 小程序页面与组件开发 |
| 后端开发 | API 接口、EXIF 云端处理（如需） |
| UI 设计 | 界面设计与交互规范 |

---

## 十、参考资源

- [微信小程序官方文档](https://developers.weixin.qq.com/miniprogram/dev/framework/)
- [TDesign 小程序组件库](https://tdesign.tencent.com/miniprogram/)
- [EXIF 标准规范](https://www.exif.org/Exif2-2.PDF)
- [微信小程序审核规范](https://developers.weixin.qq.com/miniprogram/product/review/)
