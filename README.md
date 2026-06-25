# EXIF相片信息修复 小程序

> 一款修复照片 EXIF 参数（设备、镜头、时间等）的微信小程序，通过模板系统实现快捷批量处理。

## 快速开始

```bash
git clone https://github.com/moshang149/exif-editor-miniapp.git
```

1. 用**微信开发者工具**打开项目目录
2. 工具 → 构建 npm（构建 TDesign 组件库）
3. 编译运行，模拟器中即可预览

## 项目结构

```
pages/          小程序页面（当前为 TDesign 组件 demo）
components/     复用组件
utils/          工具函数
services/       业务逻辑层
docs/           项目文档（需求/设计/进度/规范）
subpackages/    分包（关于页等）
```

## 技术栈

- 微信原生框架
- TDesign Miniprogram（腾讯 UI 组件库）
- Skyline 渲染引擎
- EXIF 客户端 Canvas 处理

## 文档

| 文档 | 说明 |
|------|------|
| [项目主索引](docs/INDEX.md) | 全景概览，新成员入口 |
| [编码规范](docs/CODING_STANDARDS.md) | JS/WXML/WXSS 代码风格 |
| [需求文档](docs/REQUIREMENTS.md) | 功能需求与边界 |
| [概要设计](docs/TECH_OVERVIEW.md) | 架构与技术选型 |
| [详细设计](docs/TECH_DETAIL.md) | 各模块实现方案 |
| [项目进度](docs/SCHEDULE.md) | 里程碑与任务清单 |

## 开发状态

当前处于 **M1 项目启动** 阶段，核心开发即将开始。
