# xmind 文件预览问题总结

## 1. 需求

在技能详情页的文件预览区域支持 `.xmind` 文件的在线预览。

## 2. 方案选型

选择 `xmind-embed-viewer` 库（npm 包），它是 xmind 官方提供的嵌入式查看器，通过 iframe 加载 `https://www.xmind.cn/embed-viewer/` 来渲染思维导图。

**优点**：改动最小，仅需前端改动，不依赖后端。
**缺点**：依赖外部 CDN，iframe 跨域限制，库内部有 SVG NaN bug。

## 3. 已完成的改动

### 3.1 前端代码改动（nuwax 仓库）

| 文件 | 改动 |
|------|------|
| `package.json` | 新增 `xmind-embed-viewer` 依赖 |
| `src/constants/appDevConstants.ts` | `SUPPORTEDENSIONS` 数组添加 `'xmind'` |
| `src/utils/appDevUtils.ts` | `isDocumentFile()` 的 `documentExtensions` 添加 `'xmind'`，返回 `{ isDoc: true, fileType: 'xmind' }` |
| `src/components/business-component/FilePreview/index.tsx` | 添加 xmind 类型定义、图标、预览逻辑 |

### 3.2 FilePreview 核心改动

```typescript
// 1. 类型定义
export type FileType = ... | 'xmind';

// 2. 扩展名映射
EXTENSION_MAP: { xmind: 'xmind' }

// 3. shouldShowContent / needsScroll 添加 'xmind'
// 4. ResizeObserver 重新初始化检查添加 'xmind'
// 5. initPreview 中添加 xmind 分支
case 'xmind': {
  // 直接操作 DOM 确保容器可见（不等 React 状态更新）
  container.style.display = 'block';
  container.style.visibility = 'visible';
  // double rAF 等待布局
  await new Promise(resolve => requestAnimationFrame(() => requestAnimationFrame(() => resolve())));
  // 创建 viewer 并加载
  const viewer = new XMindEmbedViewer({ el: container, region: 'cn' });
  await viewer.load(previewSrc);
  // 手动计算缩放（不用 setFitMap 避免 NaN）
  setTimeout(() => {
    const scale = Math.min(w / 1920, h / 1080, 1);
    viewer.setScale(scale);
  }, 2000);
}
```

## 4. 遗留问题：SVG NaN 错误

### 4.1 现象

打开 xmind 文件预览时，浏览器 console 输出大量 SVG 路径 NaN 错误：
```
Error: <path> attribute d: Expected number, "M NaN 0L NaN 0C NaN 0 NaN 0 NaN …".
Error: <g> attribute transform: Expected number, "scale(NaN NaN) transla…".
```

### 4.2 根因分析

`xmind-embed-viewer` 的工作原理：
1. 创建一个 iframe，加载 `https://www.xmind.cn/embed-viewer/`
2. iframe 内部运行一个 Vue 2 应用来渲染思维导图
3. 通过 `postMessage` 传递 xmind 文件数据

**NaN 的来源**：iframe 内部的 Vue 应用在初始化时会自动调用 `fitMap()` 函数，该函数读取 iframe 内部 body 的尺寸来计算缩放比例和 SVG 路径坐标。但此时 iframe 的 body 尺寸可能为 0（因为 iframe 刚创建，内部布局尚未完成），导致除以 0 产生 NaN。

**关键证据**（来自 stack trace）：
```
fitMap @ share-embed.76c2038161.js:35    ← iframe 内部的 fitMap
init @ share-embed.76c2038161.js:35      ← Vue 初始化阶段
dn @ vue@2-b0cd066675.7.14.min.js:11    ← Vue nextTick
```

这是 iframe 内部 Vue 应用的自动行为，发生在 `load()` 数据传输之前，我们无法从外部阻止。

### 4.3 尝试过的修复方案

| 方案 | 结果 |
|------|------|
| `setTimeout` 2s 后调 `setFitMap()` | ❌ `setFitMap` 本身也读取 iframe 内部尺寸，同样产生 NaN |
| 直接操作 DOM 设置容器 `display: block` + double rAF | ❌ 外部容器尺寸正确，但 iframe 内部 body 尺寸仍为 0 |
| `setScale(手动计算)` 替代 `setFitMap()` | ⚠️ 避免了我们调用产生的 NaN，但 iframe 内部初始化时的 NaN 无法消除 |
| ResizeObserver 等待容器有尺寸 | ❌ 容器有尺寸 ≠ iframe 内部有尺寸 |

### 4.4 结论

**这是 `xmind-embed-viewer` 库的 bug**，iframe 内部 Vue 应用在 `setup` 阶段过早调用 `fitMap()`，此时 iframe 内部布局未完成。该问题在 GitHub 上有相关 issue。

## 5. 后续建议

如果 xmind 预览效果不理想（NaN 导致 SVG 渲染混乱），可考虑以下替代方案：

### 方案 A：后端转换（推荐）
后端在文件上传时将 `.xmind` 转为 PNG/SVG 图片，前端直接用 `<img>` 预览。
- 优点：无 NaN 问题，加载快，不依赖外部 CDN
- 缺点：需要后端改动，需要安装 xmind CLI 或相关转换工具

### 方案 B：前端使用 xmind-parser + 自定义渲染
使用 `xmind-parser` 解析 xmind 文件结构，用 D3.js 或 AntV 自己绘制思维导图。
- 优点：完全可控，不依赖外部 CDN
- 缺点：改动大，需要自己实现思维导图渲染

### 方案 C：使用 xmind-embed-viewer 但接受 NaN
如果视觉上思维导图仍可辨识（只是 console 报错），可以接受这些 NaN 错误作为已知限制。
- 优点：零额外改动
- 缺点：console 有大量错误日志，视觉效果可能受影响
