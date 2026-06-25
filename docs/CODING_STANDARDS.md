# EXIF照片编辑器 — 项目编码规范

> 适用对象: 本项目所有前端开发人员  
> 最后更新: 2026-06-25

---

## 一、总则

### 1.1 核心理念
- **可读性优先于简洁性** — 代码是写给人看的
- **一致性大于个人偏好** — 统一风格降低认知成本
- **原生优先** — 不引入不必要的第三方依赖，控制包体积

### 1.2 适用范围
本规范覆盖以下文件类型：
- `.js` — 页面逻辑、工具函数、服务层
- `.wxml` — 页面模板
- `.wxss` — 页面样式
- `.json` — 页面/组件配置

---

## 二、JavaScript 编码规范

### 2.1 命名约定

| 类型 | 规则 | 示例 |
|------|------|------|
| 文件名 | kebab-case | `exif-service.js`, `template-card.js` |
| 变量/函数 | camelCase | `userName`, `getExifData()` |
| 常量 | UPPER_SNAKE_CASE | `MAX_IMAGE_SIZE`, `API_BASE_URL` |
| 类/组件 | PascalCase | `ExifForm`, `TemplateCard` |
| 私有方法 | `_` 前缀 | `_parseRawExif()`, `_formatDate()` |
| 事件处理 | `handle` + 动作 | `handleTapSave`, `handleInputChange` |
| 布尔变量 | `is/has/can` 前缀 | `isLoading`, `hasExif`, `canEdit` |

### 2.2 文件结构

```javascript
// 1. 依赖导入
const { request } = require('../../utils/request');
const { parseExif } = require('../../utils/exif');

// 2. 常量定义
const MAX_RETRY = 3;

// 3. 页面定义
Page({
  data: {
    // 视图状态
  },

  // 4. 生命周期（按执行顺序排列）
  onLoad() {},
  onShow() {},
  onReady() {},

  // 5. 事件处理（公有）
  handleTapSave() {},

  // 6. 业务方法（私有）
  _loadImage() {},
});

// 7. 模块导出（工具类文件）
module.exports = { parseExif };
```

### 2.3 异步处理

```javascript
// ✅ 推荐: async/await
async onLoad() {
  try {
    const data = await this._fetchExif();
    this.setData({ exif: data });
  } catch (err) {
    this._handleError(err);
  }
}

// ❌ 避免: 回调地狱
wx.request({
  success(res) {
    wx.request({
      success(res2) {
        // 层层嵌套
      }
    })
  }
})
```

### 2.4 setData 规范

```javascript
// ✅ 推荐: 最小化 setData 频率与数据量
this.setData({
  'exif.make': 'Canon',
  'exif.model': 'EOS R5',
  hasChanges: true,
});

// ❌ 避免: 全量更新或多余字段
this.setData({
  exif: {
    ...this.data.exif,
    make: 'Canon',
  },
  // 不要传入视图不需要的字段
});

// ❌ 避免: 高频调用
for (let i = 0; i < 100; i++) {
  this.setData({ [`list[${i}]`]: value }); // 100 次 setData！
}
// 改为数组拼接后一次更新
```

### 2.5 错误处理

```javascript
// 统一错误处理模式
const handleError = (err, context = '') => {
  console.error(`[${context}]`, err);
  wx.showToast({
    title: err.message || '操作失败，请重试',
    icon: 'none',
    duration: 2000,
  });
};
```

---

## 三、WXML 编码规范

### 3.1 属性顺序

```xml
<view
  class="container"
  id="main-view"
  wx:if="{{condition}}"
  wx:for="{{list}}"
  wx:key="id"
  bindtap="handleTap"
  data-id="{{item.id}}"
>
```

标准顺序: `class` → `id` → 条件渲染 → 列表渲染 → 事件绑定 → data-*

### 3.2 条件渲染选择

```xml
<!-- 频繁切换用 hidden -->
<view hidden="{{!showDetail}}">详情内容</view>

<!-- 不常变化用 wx:if -->
<view wx:if="{{isVip}}">VIP 专属功能</view>

<!-- 多分支用 block + wx:if -->
<block wx:if="{{status === 'loading'}}">
  <view>加载中...</view>
</block>
<block wx:elif="{{status === 'error'}}">
  <view>加载失败</view>
</block>
<block wx:else>
  <view>正常内容</view>
</block>
```

### 3.3 模板复用

```xml
<!-- 组件化优先 -->
<exif-form
  fields="{{exifFields}}"
  bind:change="handleExifChange"
/>

<!-- 简单片段可内联 -->
<template is="exifRow" data="{{...item}}" />
```

---

## 四、WXSS 编码规范

### 4.1 命名约定

采用 **BEM 变体**（Block__Element--Modifier）：

```css
/* Block */
.editor { }

/* Element */
.editor__toolbar { }
.editor__preview { }

/* Modifier */
.editor__toolbar--hidden { }
.editor__button--primary { }
```

### 4.2 属性排序

1. 定位 → 2. 盒模型 → 3. 排版 → 4. 视觉 → 5. 其他

```css
.selector {
  /* 1. 定位 */
  position: relative;
  top: 0;
  z-index: 1;

  /* 2. 盒模型 */
  display: flex;
  width: 100%;
  height: 48px;
  padding: 0 16px;
  margin: 8px 0;

  /* 3. 排版 */
  font-size: 14px;
  line-height: 1.5;
  text-align: center;

  /* 4. 视觉 */
  color: #333;
  background: #fff;
  border-radius: 8px;

  /* 5. 其他 */
  transition: all 0.3s;
}
```

### 4.3 单位使用

```css
/* rpx 用于需要适配不同屏幕的尺寸 */
.container { width: 750rpx; padding: 32rpx; }  /* 推荐 */

/* px 用于固定尺寸（图标、边框等） */
.icon { width: 24px; height: 24px; }
.divider { border-bottom: 1px solid #eee; }
```

---

## 五、JSON 配置规范

### 5.1 页面配置

```json
{
  "navigationBarTitleText": "EXIF 编辑",
  "navigationBarBackgroundColor": "#ffffff",
  "usingComponents": {
    "exif-form": "/components/exif-form/exif-form",
    "t-button": "tdesign-miniprogram/button/button"
  }
}
```

### 5.2 组件配置

```json
{
  "component": true,
  "styleIsolation": "isolated"
}
```

---

## 六、Git 提交规范

### 6.1 分支策略

```
main          — 生产分支，仅通过 MR 合入
├── dev       — 开发分支
│   ├── feat/exif-parser     — 功能分支
│   ├── feat/template-mgr    — 功能分支
│   ├── fix/image-upload     — 修复分支
│   └── perf/setdata-optimize — 优化分支
```

**远程仓库**: https://github.com/moshang149/exif-editor-miniapp

### 6.2 Commit Message 格式

```
<type>(<scope>): <subject>

[body]
```

**type 类型:**

| 类型 | 说明 |
|------|------|
| feat | 新功能 |
| fix | Bug 修复 |
| docs | 文档更新 |
| style | 代码格式（不影响逻辑） |
| refactor | 重构 |
| perf | 性能优化 |
| test | 测试 |
| chore | 构建/工具变更 |

**示例:**

```
feat(editor): 添加 EXIF 设备信息编辑表单组件

实现设备品牌、型号、序列号三个字段的输入校验与双向绑定。
支持模板数据预填充。
```

---

## 七、审查清单

提交代码前自查:

- [ ] setData 调用次数是否已最小化
- [ ] 是否处理了异步操作的错误状态
- [ ] 新增组件是否注册到 `usingComponents`
- [ ] 是否新增了未经授权的 API 调用
- [ ] 包大小是否在限制范围内（主包 < 2MB）
- [ ] 是否有未使用的导入或变量
- [ ] WXSS 是否全部使用 rpx 而非 px
