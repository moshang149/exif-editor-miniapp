# EXIF照片编辑器 — 详细设计文档

> 版本: v1.0  
> 最后更新: 2026-06-25  
> 关联: [需求文档](./REQUIREMENTS.md) | [概要设计](./TECH_OVERVIEW.md)

---

## 1. 编码约定说明

以下所有代码示例遵循 [编码规范](./CODING_STANDARDS.md)。

---

## 2. 全局状态设计

### 2.1 App 全局数据

```javascript
// app.js
App({
  globalData: {
    userInfo: null,           // 微信用户信息
    currentImage: null,       // 当前编辑的图片信息
    currentExifData: null,    // 当前编辑的 EXIF 数据
    editHistory: [],          // 撤销栈
    templateCache: null,      // 模板列表缓存
  },

  onLaunch() {
    // 预加载模板数据
    this._initTemplates();
  },

  _initTemplates() {
    const { loadBuiltinTemplates, loadUserTemplates } = require('./services/template-service');
    this.globalData.templateCache = {
      builtin: loadBuiltinTemplates(),
      user: loadUserTemplates(),
    };
  },
});
```

### 2.2 页面间数据传递

```javascript
// 使用 EventChannel（页面跳转时传递大型数据）
// 跳转方
wx.navigateTo({
  url: '/pages/editor/editor',
  success: (res) => {
    res.eventChannel.emit('initImage', { path, exifData });
  },
});

// 接收方
Page({
  onLoad() {
    const eventChannel = this.getOpenerEventChannel();
    eventChannel.on('initImage', ({ path, exifData }) => {
      this.setData({ imagePath: path, exifData });
    });
  },
});
```

---

## 3. 模块详细设计

### 3.1 图片获取模块（F-001）

**对应需求**: F-001-1 ~ F-001-3

#### 3.1.1 文件结构

```
utils/image.js   → 图片处理工具函数
components/image-picker/ → 图片选择器组件
```

#### 3.1.2 核心实现

```javascript
// utils/image.js — 图片选择与处理

/**
 * 选择图片（相册或相机）
 * @param {'album'|'camera'} sourceType
 * @returns {Promise<{path: string, size: number, type: string, exif: object}>}
 */
async function chooseImage(sourceType = 'album') {
  const MAX_SIZE = 20 * 1024 * 1024; // 20MB

  try {
    // 微信新版 API
    const res = await wx.chooseMedia({
      count: 1,
      mediaType: ['image'],
      sourceType: [sourceType],
      sizeType: ['original'],
    });

    const tempFile = res.tempFiles[0];

    // 检查文件大小
    if (tempFile.size > MAX_SIZE) {
      wx.showToast({ title: '图片不能超过 20MB', icon: 'none' });
      return null;
    }

    // 获取图片信息和 EXIF
    const imageInfo = await _getImageInfo(tempFile.tempFilePath);

    return {
      path: tempFile.tempFilePath,
      size: tempFile.size,
      type: imageInfo.type,
      width: imageInfo.width,
      height: imageInfo.height,
      exif: imageInfo.orientation ? { orientation: imageInfo.orientation } : {},
    };
  } catch (err) {
    if (err.errMsg && err.errMsg.includes('cancel')) {
      return null; // 用户取消
    }
    throw err;
  }
}

/**
 * 获取图片基本信息（promisify wx.getImageInfo）
 */
function _getImageInfo(src) {
  return new Promise((resolve, reject) => {
    wx.getImageInfo({
      src,
      success: resolve,
      fail: reject,
    });
  });
}

module.exports = { chooseImage };
```

#### 3.1.3 图像选择器组件

```javascript
// components/image-picker/image-picker.js
Component({
  properties: {
    sourceType: {
      type: String,
      value: 'album', // 'album' | 'camera'
    },
  },

  methods: {
    async handlePick() {
      this.triggerEvent('beforepick');
      const result = await chooseImage(this.data.sourceType);
      if (result) {
        this.triggerEvent('picked', { image: result });
      }
    },
  },
});
```

```xml
<!-- components/image-picker/image-picker.wxml -->
<view class="picker" bindtap="handlePick">
  <slot>
    <view class="picker__placeholder">
      <image class="picker__icon" src="/assets/icons/add-photo.svg" />
      <text class="picker__text">选择照片</text>
    </view>
  </slot>
</view>
```

#### 3.1.4 数据流

```
用户点击 → chooseImage() → wx.chooseMedia → 返回 tempFilePath
  → wx.getImageInfo → 读取宽高/类型/orientation
  → 返回 {path, size, type, width, height, exif: {orientation}}
  → 触发 picked 事件 → 父页面接收
```

---

### 3.2 EXIF 查看模块（F-002）

**对应需求**: F-002-1, F-002-2

#### 3.2.1 文件结构

```
utils/exif.js          → EXIF 解析工具
services/exif-service.js → EXIF 业务逻辑
```

#### 3.2.2 核心实现

```javascript
// utils/exif.js — EXIF 数据解析（客户端 JS 实现）

/**
 * 从图片 ArrayBuffer 中解析 EXIF 数据
 * 基于 exif-js 核心逻辑，适配小程序环境
 */
function parseExifFromArrayBuffer(buffer) {
  // EXIF 数据以 0xFFE1 标记开始
  const dataView = new DataView(buffer);
  const exifData = {};

  if (dataView.getUint8(0) !== 0xFF || dataView.getUint8(1) !== 0xD8) {
    return null; // 不是 JPEG
  }

  let offset = 2;
  while (offset < dataView.byteLength) {
    if (dataView.getUint8(offset) !== 0xFF) break;

    const marker = dataView.getUint8(offset + 1);
    if (marker === 0xE1) {
      // 找到 EXIF 段
      const exifOffset = _parseExifSegment(dataView, offset + 4);
      Object.assign(exifData, exifOffset);
      break;
    }
    offset += 2 + dataView.getUint16(offset + 2);
  }

  return _normalizeExifData(exifData);
}

/**
 * 解析 EXIF IFD（Image File Directory）
 */
function _parseExifSegment(dataView, start) {
  const result = {};
  const tiffOffset = start + 6;
  const littleEndian = dataView.getUint16(start) === 0x4949;

  // ... IFD 解析逻辑（标签映射表省略，包含常用标签 0x010F~0xA420）

  return result;
}

/**
 * 将原始 EXIF 标签数据标准化为 ExifData 模型
 */
function _normalizeExifData(raw) {
  return {
    make: raw.Make || '',
    model: raw.Model || '',
    software: raw.Software || '',
    lensMake: raw.LensMake || '',
    lensModel: raw.LensModel || '',
    focalLength: raw.FocalLength ? `${raw.FocalLength}mm` : '',
    aperture: raw.FNumber ? `f/${raw.FNumber}` : '',
    shutterSpeed: raw.ExposureTime ? formatShutter(raw.ExposureTime) : '',
    iso: raw.ISOSpeedRatings || 0,
    exposureBias: raw.ExposureBiasValue ? `${raw.ExposureBiasValue} EV` : '',
    dateTimeOriginal: raw.DateTimeOriginal || '',
    latitude: raw.GPSLatitude ? _convertGPSToDecimal(raw.GPSLatitude, raw.GPSLatitudeRef) : null,
    longitude: raw.GPSLongitude ? _convertGPSToDecimal(raw.GPSLongitude, raw.GPSLongitudeRef) : null,
  };
}
```

#### 3.2.3 服务层封装

```javascript
// services/exif-service.js

const { parseExifFromArrayBuffer } = require('../utils/exif');
const { chooseImage } = require('../utils/image');

/**
 * 完整流程：选取图片 → 读取 EXIF → 返回标准化数据
 * @returns {Promise<{imagePath, exifData, hasExif}>}
 */
async function loadImageWithExif(sourceType = 'album') {
  const image = await chooseImage(sourceType);
  if (!image) return null;

  // 读取文件二进制数据以解析 EXIF
  const fs = wx.getFileSystemManager();
  const buffer = fs.readFileSync(image.path);

  const exifData = parseExifFromArrayBuffer(buffer);
  const hasExif = exifData !== null && Object.keys(exifData).some(k => exifData[k]);

  return {
    imagePath: image.path,
    exifData: exifData || _createEmptyExif(),
    hasExif,
    imageInfo: {
      width: image.width,
      height: image.height,
      type: image.type,
    },
  };
}

/**
 * 创建空的 ExifData 对象
 */
function _createEmptyExif() {
  return {
    make: '', model: '', software: '',
    lensMake: '', lensModel: '',
    focalLength: '', aperture: '', shutterSpeed: '',
    iso: 0, exposureBias: '',
    dateTimeOriginal: '',
    latitude: null, longitude: null,
  };
}

module.exports = { loadImageWithExif };
```

#### 3.2.4 数据流

```
用户选图 → chooseImage() 返回 buffer
  → parseExifFromArrayBuffer(buffer) 解析二进制
  → _normalizeExifData(raw) 标准化为 ExifData 模型
  → 返回 {imagePath, exifData, hasExif}
  → 编辑器页面 setData 渲染
```

---

### 3.3 EXIF 编辑模块（F-003）

**对应需求**: F-003-1 ~ F-003-7

#### 3.3.1 文件结构

```
pages/editor/editor.js     → 编辑页逻辑（含撤销/重做）
components/exif-form/       → EXIF 字段编辑表单组件
utils/exif.js               → EXIF 数据校验
```

#### 3.3.2 撤销/重做（命令模式）

```javascript
// pages/editor/editor.js — 编辑页核心逻辑

const MAX_HISTORY = 20;

Page({
  data: {
    exifData: null,         // 当前 EXIF 数据
    imagePath: '',
    hasChanges: false,
    canUndo: false,
    canRedo: false,
    templateApplied: null,  // 当前使用的模板信息
  },

  // 撤销栈（不参与视图渲染，放在 this 上）
  _undoStack: [],
  _redoStack: [],

  /**
   * 记录一次字段修改（供 exif-form 组件调用）
   */
  recordChange(field, oldValue, newValue) {
    // 入栈
    this._undoStack.push({ field, oldValue, newValue, timestamp: Date.now() });
    // 限制栈深度
    if (this._undoStack.length > MAX_HISTORY) {
      this._undoStack.shift();
    }
    // 清空重做栈（新操作使旧重做记录无效）
    this._redoStack = [];

    this._updateHistoryState();
  },

  /**
   * 撤销
   */
  handleUndo() {
    if (this._undoStack.length === 0) return;

    const action = this._undoStack.pop();
    this._redoStack.push(action);

    this.setData({
      [`exifData.${action.field}`]: action.oldValue,
    });
    this._updateHistoryState();
  },

  /**
   * 重做
   */
  handleRedo() {
    if (this._redoStack.length === 0) return;

    const action = this._redoStack.pop();
    this._undoStack.push(action);

    this.setData({
      [`exifData.${action.field}`]: action.newValue,
    });
    this._updateHistoryState();
  },

  _updateHistoryState() {
    this.setData({
      canUndo: this._undoStack.length > 0,
      canRedo: this._redoStack.length > 0,
      hasChanges: this._undoStack.length > 0,
    });
  },

  /**
   * 一键清除 EXIF
   */
  handleClearExif() {
    wx.showModal({
      title: '确认清除',
      content: '将清空所有可编辑的 EXIF 信息，此操作不可恢复',
      success: (res) => {
        if (res.confirm) {
          const oldData = { ...this.data.exifData };
          const emptyData = _createEmptyExif();

          // 记录每个字段的变更
          Object.keys(emptyData).forEach(key => {
            if (oldData[key] !== emptyData[key]) {
              this._undoStack.push({
                field: key,
                oldValue: oldData[key],
                newValue: emptyData[key],
                timestamp: Date.now(),
              });
            }
          });

          this.setData({ exifData: emptyData });
          this._redoStack = [];
          this._updateHistoryState();
        }
      },
    });
  },
});
```

#### 3.3.3 EXIF 表单组件

```javascript
// components/exif-form/exif-form.js

Component({
  properties: {
    exifData: {
      type: Object,
      value: {},
    },
    readonly: {
      type: Boolean,
      value: false,
    },
  },

  /**
   * 字段配置（驱动表单渲染）
   */
  data: {
    fields: [
      // 设备信息组
      { key: 'make',          label: '设备品牌',   type: 'text',   group: '设备信息', placeholder: '如 Canon' },
      { key: 'model',         label: '设备型号',   type: 'text',   group: '设备信息', placeholder: '如 EOS R5' },
      { key: 'software',      label: '处理软件',   type: 'text',   group: '设备信息', placeholder: '如 Adobe Lightroom' },
      // 镜头信息组
      { key: 'lensMake',      label: '镜头品牌',   type: 'text',   group: '镜头信息', placeholder: '如 Canon' },
      { key: 'lensModel',     label: '镜头型号',   type: 'text',   group: '镜头信息', placeholder: '如 RF 70-200mm F2.8' },
      { key: 'focalLength',   label: '焦距',       type: 'text',   group: '镜头信息', placeholder: '如 70mm' },
      { key: 'aperture',      label: '光圈',       type: 'text',   group: '拍摄参数', placeholder: '如 f/2.8' },
      // 拍摄参数组
      { key: 'shutterSpeed',  label: '快门速度',   type: 'text',   group: '拍摄参数', placeholder: '如 1/1000' },
      { key: 'iso',           label: 'ISO',        type: 'number', group: '拍摄参数', placeholder: '如 100' },
      { key: 'exposureBias',  label: '曝光补偿',   type: 'text',   group: '拍摄参数', placeholder: '如 +0.3 EV' },
      { key: 'dateTimeOriginal', label: '拍摄时间', type: 'datetime', group: '时间地点', placeholder: '' },
      // GPS 组
      { key: 'latitude',      label: '纬度',       type: 'number', group: '时间地点', placeholder: '如 39.9042' },
      { key: 'longitude',     label: '经度',       type: 'number', group: '时间地点', placeholder: '如 116.4074' },
    ],
  },

  methods: {
    /**
     * 字段值变更（带防抖的批量更新）
     */
    handleFieldChange(e) {
      const { field, value, oldValue } = e.detail;

      // 触发父页面记录变更（撤销/重做用）
      this.triggerEvent('fieldchange', { field, value, oldValue });

      // 通知父页面更新数据
      this.triggerEvent('update', {
        field,
        value: _validateField(field, value),
      });
    },

    /**
     * 批量填充（模板套用时调用）
     */
    applyTemplate(templateData) {
      const updates = {};
      Object.keys(templateData).forEach(key => {
        if (templateData[key] !== undefined && templateData[key] !== null) {
          updates[`exifData.${key}`] = templateData[key];
        }
      });

      // 一次性 setData
      this.setData(updates);
    },
  },
});
```

#### 3.3.4 字段校验

```javascript
// utils/exif.js — 字段校验函数

const validators = {
  iso: (v) => {
    const n = Number(v);
    return Number.isInteger(n) && n >= 1 && n <= 102400;
  },
  latitude: (v) => {
    const n = Number(v);
    return !isNaN(n) && n >= -90 && n <= 90;
  },
  longitude: (v) => {
    const n = Number(v);
    return !isNaN(n) && n >= -180 && n <= 180;
  },
  focalLength: (v) => /^\d+(\.\d+)?mm$/.test(v),
  aperture: (v) => /^f\/\d+(\.\d+)?$/.test(v),
};

function validateField(field, value) {
  const validator = validators[field];
  if (!validator) return { valid: true };
  return { valid: validator(value), message: validator(value) ? '' : `无效的${field}值` };
}

module.exports = { validateField, validators };
```

#### 3.3.5 数据流

```
用户修改字段 → handleFieldChange() → validateField()
  → triggerEvent('fieldchange', {field, value, oldValue})  ← 父页面记录到撤销栈
  → triggerEvent('update', {field, value})                  ← 父页面 setData 更新
  → 关联字段联动（如 Latitude + Longitude 同步校验）
```

---

### 3.4 模板系统模块（F-004）

**对应需求**: F-004-1 ~ F-004-5

#### 3.4.1 文件结构

```
services/template-service.js  → 模板 CRUD 逻辑
pages/template/template.js    → 模板管理页
components/template-card/      → 模板卡片组件
```

#### 3.4.2 核心实现

```javascript
// services/template-service.js

const STORAGE_KEY = 'user_templates';
const MAX_USER_TEMPLATES = 50;

/**
 * 内置模板数据
 */
const BUILTIN_TEMPLATES = [
  {
    id: 'builtin_iphone15pro',
    name: 'iPhone 15 Pro',
    category: 'builtin',
    icon: '/assets/templates/iphone15pro.png',
    exifData: {
      make: 'Apple',
      model: 'iPhone 15 Pro',
      software: 'iOS 17',
      lensMake: 'Apple',
      lensModel: 'iPhone 15 Pro back triple camera 6.86mm f/1.78',
      focalLength: '6.86mm',
      aperture: 'f/1.78',
      shutterSpeed: '',
      iso: 0,
      exposureBias: '',
      dateTimeOriginal: '',
      latitude: null,
      longitude: null,
    },
  },
  {
    id: 'builtin_canon_r5',
    name: 'Canon EOS R5',
    category: 'builtin',
    icon: '/assets/templates/canon_r5.png',
    exifData: {
      make: 'Canon',
      model: 'Canon EOS R5',
      software: 'Adobe Lightroom',
      lensMake: 'Canon',
      lensModel: 'RF 70-200mm F2.8 L IS USM',
      focalLength: '70mm',
      aperture: 'f/2.8',
      shutterSpeed: '1/1000',
      iso: 100,
      exposureBias: '',
      dateTimeOriginal: '',
      latitude: null,
      longitude: null,
    },
  },
  {
    id: 'builtin_leica_m11',
    name: 'Leica M11',
    category: 'builtin',
    icon: '/assets/templates/leica_m11.png',
    exifData: {
      make: 'Leica Camera AG',
      model: 'LEICA M11',
      lensMake: 'Leica',
      lensModel: 'Summilux-M 35mm f/1.4 ASPH',
      focalLength: '35mm',
      aperture: 'f/1.4',
      iso: 200,
      exposureBias: '',
      dateTimeOriginal: '',
      latitude: null,
      longitude: null,
    },
  },
  {
    id: 'builtin_sony_a7m4',
    name: 'Sony A7M4',
    category: 'builtin',
    icon: '/assets/templates/sony_a7m4.png',
    exifData: {
      make: 'SONY',
      model: 'ILCE-7M4',
      software: 'Capture One',
      lensMake: 'SONY',
      lensModel: 'FE 24-70mm F2.8 GM II',
      focalLength: '50mm',
      aperture: 'f/2.8',
      shutterSpeed: '1/500',
      iso: 400,
      exposureBias: '+0.3 EV',
      dateTimeOriginal: '',
      latitude: null,
      longitude: null,
    },
  },
  {
    id: 'builtin_fujifilm_xt5',
    name: 'Fujifilm X-T5',
    category: 'builtin',
    icon: '/assets/templates/fujifilm_xt5.png',
    exifData: {
      make: 'FUJIFILM',
      model: 'X-T5',
      software: '',
      lensMake: 'FUJIFILM',
      lensModel: 'XF 33mm F1.4 R LM WR',
      focalLength: '33mm',
      aperture: 'f/1.4',
      shutterSpeed: '1/250',
      iso: 160,
      exposureBias: '',
      dateTimeOriginal: '',
      latitude: null,
      longitude: null,
    },
  },
  {
    id: 'builtin_nikon_z8',
    name: 'Nikon Z8',
    category: 'builtin',
    icon: '/assets/templates/nikon_z8.png',
    exifData: {
      make: 'NIKON CORPORATION',
      model: 'NIKON Z 8',
      lensMake: 'NIKON',
      lensModel: 'NIKKOR Z 24-120mm f/4 S',
      focalLength: '85mm',
      aperture: 'f/4',
      shutterSpeed: '1/640',
      iso: 64,
      exposureBias: '',
      dateTimeOriginal: '',
      latitude: null,
      longitude: null,
    },
  },
];

/**
 * 获取所有模板（内置 + 用户自定义）
 */
function getAllTemplates() {
  const builtin = BUILTIN_TEMPLATES;
  const user = loadUserTemplates();
  return [...builtin, ...user];
}

/**
 * 加载用户自定义模板
 */
function loadUserTemplates() {
  try {
    const data = wx.getStorageSync(STORAGE_KEY);
    return data ? JSON.parse(data) : [];
  } catch {
    return [];
  }
}

/**
 * 保存用户模板
 */
function saveUserTemplate(name, exifData) {
  const templates = loadUserTemplates();

  if (templates.length >= MAX_USER_TEMPLATES) {
    wx.showToast({ title: `最多保存 ${MAX_USER_TEMPLATES} 个模板`, icon: 'none' });
    return null;
  }

  const template = {
    id: `user_${Date.now()}_${Math.random().toString(36).slice(2, 8)}`,
    name,
    category: 'user',
    exifData: { ...exifData },
    createdAt: Date.now(),
  };

  templates.push(template);
  wx.setStorageSync(STORAGE_KEY, JSON.stringify(templates));

  return template;
}

/**
 * 删除用户模板
 */
function deleteUserTemplate(id) {
  let templates = loadUserTemplates();
  templates = templates.filter(t => t.id !== id);
  wx.setStorageSync(STORAGE_KEY, JSON.stringify(templates));
}

/**
 * 重命名用户模板
 */
function renameUserTemplate(id, newName) {
  const templates = loadUserTemplates();
  const index = templates.findIndex(t => t.id === id);
  if (index === -1) return false;

  templates[index].name = newName;
  wx.setStorageSync(STORAGE_KEY, JSON.stringify(templates));
  return true;
}

/**
 * 搜索模板
 */
function searchTemplates(keyword) {
  const all = getAllTemplates();
  if (!keyword) return all;

  const kw = keyword.toLowerCase();
  return all.filter(t =>
    t.name.toLowerCase().includes(kw) ||
    t.exifData.make.toLowerCase().includes(kw) ||
    t.exifData.model.toLowerCase().includes(kw)
  );
}

module.exports = {
  getAllTemplates,
  loadUserTemplates,
  saveUserTemplate,
  deleteUserTemplate,
  renameUserTemplate,
  searchTemplates,
  BUILTIN_TEMPLATES,
};
```

#### 3.4.3 数据流

```
模板选择流程:
  编辑器页 → wx.navigateTo('/pages/template/template')
  → 模板页调用 getAllTemplates() 加载列表
  → 用户点击模板 → getOpenerEventChannel().emit('templateSelected', templateData)
  → 编辑器接收 → setData 填充 exifData → 重新渲染表单

模板创建流程:
  编辑器页 → 弹出命名对话框
  → saveUserTemplate(name, currentExifData)
  → 写入 wx.Storage → Toast "已保存"
  → 模板列表自动刷新
```

---

### 3.5 图片保存模块（F-005）

**对应需求**: F-005-1 ~ F-005-4

#### 3.5.1 文件结构

```
utils/image.js           → 图片导出与保存
components/save-panel/   → 保存面板组件
pages/preview/preview.js → 预览与保存页
```

#### 3.5.2 核心实现

```javascript
// utils/image.js — 图片导出与保存

/**
 * 将 EXIF 数据写入图片并保存到相册
 *
 * 微信小程序限制：Canvas 导出的图片无法保留原始 EXIF 二进制数据。
 * 因此采用两步策略：
 *   1. Canvas 绘制原图 → 导出干净的 JPEG
 *   2. 如果服务端可用 → 上传并注入 EXIF 后下载
 *   3. 如果纯客户端 → 保存不含新 EXIF 的图片（降级）
 *
 * @param {string} imagePath  - 原图临时路径
 * @param {object} exifData   - 要写入的 EXIF 数据
 * @param {object} callbacks  - 进度回调
 */
async function saveImageWithExif(imagePath, exifData, callbacks = {}) {
  const { onProgress, onComplete, onError } = callbacks;

  try {
    // 步骤 1: 压缩/处理原图（如果过大）
    onProgress && onProgress({ step: 'compress', progress: 10 });
    const processedPath = await _processImage(imagePath);

    // 步骤 2: Canvas 重绘导出
    onProgress && onProgress({ step: 'render', progress: 40 });
    const renderedPath = await _renderImageWithOverlay(processedPath, exifData);

    // 步骤 3: 保存到系统相册
    onProgress && onProgress({ step: 'save', progress: 80 });
    await _saveToAlbum(renderedPath);

    onProgress && onProgress({ step: 'done', progress: 100 });
    onComplete && onComplete({ path: renderedPath });
  } catch (err) {
    onError && onError(err);
    throw err;
  }
}

/**
 * 压缩大图
 */
async function _processImage(imagePath) {
  const info = await _getImageInfo(imagePath);
  const MAX_DIM = 4096;

  if (info.width <= MAX_DIM && info.height <= MAX_DIM) {
    return imagePath;
  }

  const quality = _calculateQuality(info.width, info.height, MAX_DIM);
  const res = await wx.compressImage({
    src: imagePath,
    quality: quality,
  });
  return res.tempFilePath;
}

/**
 * 使用 Canvas 重绘图片（可选：叠加水印/边框）
 */
function _renderImageWithOverlay(imagePath, exifData) {
  return new Promise((resolve, reject) => {
    const query = wx.createSelectorQuery();
    query.select('#offscreen-canvas')
      .fields({ node: true, size: true })
      .exec((res) => {
        const canvas = res[0].node;
        const ctx = canvas.getContext('2d');

        const img = canvas.createImage();
        img.onload = () => {
          canvas.width = img.width;
          canvas.height = img.height;
          ctx.drawImage(img, 0, 0);

          wx.canvasToTempFilePath({
            canvas,
            fileType: 'jpg',
            quality: 0.95,
            success: (res) => resolve(res.tempFilePath),
            fail: reject,
          });
        };
        img.onerror = reject;
        img.src = imagePath;
      });
  });
}

/**
 * 保存到系统相册
 */
function _saveToAlbum(filePath) {
  return new Promise((resolve, reject) => {
    // 先检查权限
    wx.getSetting({
      success: (setting) => {
        if (setting.authSetting['scope.writePhotosAlbum'] === false) {
          // 之前拒绝过，引导用户去设置
          wx.showModal({
            title: '需要相册权限',
            content: '请在设置中允许小程序访问您的相册',
            confirmText: '去设置',
            success: (modalRes) => {
              if (modalRes.confirm) {
                wx.openSetting();
              }
              reject(new Error('PERMISSION_DENIED'));
            },
          });
          return;
        }

        wx.saveImageToPhotosAlbum({
          filePath,
          success: resolve,
          fail: (err) => {
            if (err.errMsg.includes('auth deny')) {
              wx.showModal({
                title: '需要相册权限',
                content: '请允许小程序保存图片到相册',
                confirmText: '去授权',
                success: (modalRes) => {
                  if (modalRes.confirm) {
                    wx.openSetting();
                  }
                },
              });
            }
            reject(err);
          },
        });
      },
    });
  });
}

module.exports = { saveImageWithExif };
```

#### 3.5.3 保存面板组件

```javascript
// components/save-panel/save-panel.js

Component({
  properties: {
    imagePath: String,
    exifData: Object,
    visible: Boolean,
  },

  data: {
    saving: false,
    progress: 0,
    statusText: '',
  },

  methods: {
    async handleSave() {
      this.setData({ saving: true, progress: 0, statusText: '正在处理图片...' });

      try {
        await saveImageWithExif(
          this.data.imagePath,
          this.data.exifData,
          {
            onProgress: ({ progress, step }) => {
              const texts = {
                compress: '正在压缩图片...',
                render: '正在写入数据...',
                save: '正在保存到相册...',
                done: '已完成',
              };
              this.setData({ progress, statusText: texts[step] || '' });
            },
          }
        );

        this.setData({ saving: false, statusText: '已保存到相册' });

        // 延迟隐藏面板，让用户看到成功状态
        setTimeout(() => {
          this.triggerEvent('saved');
        }, 1500);

        // iPhone 相册风格的成功反馈
        wx.showToast({
          title: '已保存到"照片"',
          icon: 'success',
          duration: 2000,
        });
      } catch (err) {
        this.setData({
          saving: false,
          statusText: err.message || '保存失败',
        });
        this.triggerEvent('error', { error: err });
      }
    },

    handleContinue() {
      this.triggerEvent('continue');
    },

    handleCancel() {
      this.triggerEvent('cancel');
    },
  },
});
```

#### 3.5.4 数据流

```
用户点击保存 → save-panel.handleSave()
  → saveImageWithExif(imagePath, exifData, callbacks)
  → _processImage() 压缩
  → _renderImageWithOverlay() Canvas 重绘
  → _saveToAlbum() 写入系统相册
  → onProgress 更新进度条
  → onComplete / onError 回调
  → 显示保存结果
```

---

### 3.6 历史记录模块（P1）

**对应需求**: F-006-1 ~ F-006-3

```javascript
// services/history-service.js (P1 实现)

const HISTORY_KEY = 'edit_history';
const MAX_HISTORY = 20;

/**
 * 保存编辑记录
 */
function saveHistory({ imagePath, thumbnailPath, exifData, templateName, timestamp }) {
  const records = loadHistory();

  // 去重：同一图片的最新编辑覆盖旧记录
  const filtered = records.filter(r => r.imagePath !== imagePath);

  records.unshift({
    id: `hist_${Date.now()}`,
    imagePath,
    thumbnailPath,
    exifData: { ...exifData },
    templateName,
    timestamp,
  });

  if (records.length > MAX_HISTORY) {
    records.pop();
  }

  wx.setStorageSync(HISTORY_KEY, JSON.stringify(records));
}

/**
 * 加载历史记录
 */
function loadHistory() {
  try {
    return JSON.parse(wx.getStorageSync(HISTORY_KEY) || '[]');
  } catch {
    return [];
  }
}

module.exports = { saveHistory, loadHistory };
```

---

## 4. 页面交互细节

### 4.1 编辑器页（pages/editor）

```
┌──────────────────────────────────┐
│  ← 返回    EXIF 编辑器    撤销/重做 │  ← 顶部导航
├──────────────────────────────────┤
│                                  │
│     ┌────────────────────┐       │
│     │                    │       │
│     │    图片预览区域      │       │  ← 原图缩略图
│     │    (当前 EXIF 摘要)  │       │
│     │                    │       │
│     └────────────────────┘       │
│                                  │
│  ┌─ 选择模板 ──────────────────┐  │  ← 模板快速入口
│  │ [iPhone15p] [CanonR5] ...  │  │
│  └────────────────────────────┘  │
│                                  │
│  ┌─ 设备信息 ──────────────────┐  │
│  │ 品牌  [_____________]      │  │
│  │ 型号  [_____________]      │  │  ← EXIF 表单
│  │ 软件  [_____________]      │  │
│  ├─ 镜头信息 ──────────────────┤  │
│  │ 镜头  [_____________]      │  │
│  │ 焦距  [_____________]      │  │
│  ├─ 拍摄参数 ──────────────────┤  │
│  │ 光圈  [_____________]      │  │
│  │ 快门  [_____________]      │  │
│  │ ISO   [_____________]      │  │
│  ├─ 时间地点 ──────────────────┤  │
│  │ 时间  [___日期选择器___]    │  │
│  │ GPS   [___地图选点___]     │  │
│  └────────────────────────────┘  │
│                                  │
│  [清除EXIF]         [预览保存 →] │  ← 底部操作栏
└──────────────────────────────────┘
```

### 4.2 iPhone 相册保存体验对标

保存后的小程序反馈应确保与 iPhone 相册体验一致：

| 交互点 | iPhone 相册表现 | 小程序对标 |
|--------|---------------|----------|
| 保存动画 | 图片从底部缩放到右下角 | Toast + 沙漏/成功动画 |
| 提示文案 | 不打扰用户，静默保存 | "已保存到「照片」" |
| 保存位置 | "最近项目"相簿 | 系统相册（最新） |
| 格式保持 | HEIC 保持 HEIC | 保持 JPEG 输出（兼容性优先） |
| 时间戳 | 使用 EXIF 中的拍摄时间 | 保存后系统自动以当前时间排序（不可控） |

---

## 5. 错误码定义

| 错误码 | 含义 | 用户提示 |
|--------|------|----------|
| `IMG_TOO_LARGE` | 图片超过 20MB | "图片不能超过 20MB，请选择更小的图片" |
| `IMG_FORMAT_UNSUPPORTED` | 不支持的图片格式 | "暂不支持此图片格式，请选择 JPG 或 PNG 格式" |
| `PERMISSION_DENIED` | 相册/相机权限被拒 | "请在设置中允许小程序访问您的相册" |
| `EXIF_PARSE_FAILED` | EXIF 解析失败 | "无法读取此图片的 EXIF 信息" |
| `SAVE_FAILED` | 保存失败 | "保存失败，请检查手机存储空间" |
| `TEMPLATE_LIMIT` | 模板数量超限 | `最多保存 ${MAX_USER_TEMPLATES} 个模板` |
| `CANVAS_ERROR` | Canvas 处理异常 | "图片处理失败，请重试或选择其他图片" |
