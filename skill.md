---
name: logo-eraser-video-watermark
description: Use this skill when working on Logo Eraser, a single-file offline browser video logo/watermark remover, or when implementing/debugging its video decomposition, mouse box selection, moving-logo tracking, temporal background restoration, edge feathering, Web Worker acceleration, WebCodecs MP4/WebM export, or local processing history. Trigger on requests about index.html, video watermark removal, logo tracking, frame restoration, worker pools, or this project.
---

# Logo Eraser 开发与维护 Skill

> 适用版本：v1.2  
> 项目路径：`/home/lyorn/Code/AIProjects/video-remove-logo`  
> 主交付物：`index.html`（单文件、离线、双击运行）  
> 使用说明：`README.md`

## 1. 任务目标

这是一个**完全在浏览器本地运行**的 Web 视频 Logo / 水印移除工具，最终必须满足：

1. 直接双击 `index.html` 就能运行，不需要服务器、npm、网络或外部资源。
2. 视频、帧、音频和导出文件都不上传到任何服务器。
3. 不引入 CDN，不新增外部字体或图片资源。
4. 用户先选择视频；工具按顺序完成拆帧、参考帧选择、Logo 框选、轨迹定位、修复预览、导出视频。
5. 支持移动 Logo 的跨帧背景还原；无法跨帧时使用内容感知修补。
6. Logo 修复边缘必须做足够宽的像素羽化，避免明显的矩形边界线。
7. 处理过程使用 `navigator.hardwareConcurrency` 检测逻辑 CPU 核数，并启动对应数量的 Web Worker（上限 32），并行拆帧编码、模板匹配和空间修补。
8. 成功导出后写入本地历史记录；历史只记录参数、轨迹和文件元信息，不缓存视频或成品。
9. 界面语言为中文，深色、克制、工具感设计；保持现有视觉语言和交互结构。

## 2. 交付物与硬约束

- 最终交付必须仍以单个 `index.html` 为主，所有 JS/CSS/第三方库都内联。
- 允许使用临时开发文件，但打包时必须回到单文件。
- 不能把视频上传、发送到 WebSocket、外部请求或通过远程 AI 服务处理。
- 不能把原始视频、帧数据、输出 blob 写进 `localStorage`。
- 不能在没有失败回退的情况下强依赖 Worker、WebCodecs 或 OffscreenCanvas。
- 第三方库必须保留许可证声明：
  - mp4box.js 0.5.3 — BSD-3-Clause
  - mp4-muxer 5.2.2 — MIT
  - webm-muxer 5.1.4 — MIT
- 推荐浏览器：最新版 Chrome / Edge。其他 Chromium 浏览器也可，但不能假设全部支持 H.264 编码或 AAC 编码。
- 保持 `file://` 下的可行性：不要使用 `fetch()` 读取本地文件，不要依赖 Service Worker 或本地服务器；必须使用 `<input type="file">` / Blob / Blob URL / Web Worker Blob URL。

## 3. 当前单文件结构

`index.html` 内部大致顺序：

1. `<style>`：全部 UI 样式。
2. `<body>`：顶部、上传页、工作区步骤、历史页、忙碌遮罩、Toast、隐藏文件输入。
3. `<script id="workerSource" type="text/js-worker">`：Worker 源码，浏览器不会当作普通脚本执行，主线程读取 `.textContent` 后用 Blob URL 创建 Worker。
4. 三个内联第三方库：mp4box.js、mp4-muxer、webm-muxer。
5. 主应用 IIFE：所有业务逻辑。
6. 第三方许可证 HTML 注释。

### 修改源码的推荐方式

由于当前仓库只有打包后的 `index.html`，直接编辑大文件容易出错。推荐：

1. 从 `index.html` 中提取：
   - `<style>` → 临时 CSS
   - `<script id="workerSource" type="text/js-worker">` → 临时 worker 源码
   - 主应用 IIFE → 临时 app.js
2. 用外部临时文件开发和测试。
3. 最后重新内联回单个 `index.html`。
4. 如果保留了构建脚本，必须保证 Worker 内的 inpainting 函数和主线程 inpainting 函数同步；目前两者是手动复制关系。
5. 搜索关键函数时注意 `contentAwareInpaint` 会出现**两次**：一次在 Worker 源码，一次在主应用。

## 4. 用户流程与状态

### 4.1 页面步骤

1. **导入与拆帧**
2. **选择参考帧**
3. **跟踪 Logo**
4. **修复与导出**

历史页是一个独立页面，不属于四个步骤。

### 4.2 关键 UI 选择器

| 选择器 | 用途 |
|---|---|
| `#fileInput` | 普通视频文件输入 |
| `#historyFileInput` | 历史恢复时重新选择原视频 |
| `#dropzone` | 视频拖放区 |
| `#extractBtn` | 开始拆帧 |
| `#refGallery` | 随机参考帧卡片 |
| `#logoEditor` | 参考帧框选编辑器 |
| `#canvasFit` | **保持视频宽高比的画布外层** |
| `#logoCanvas` | 真正显示视频帧和绘制选框的画布 |
| `#canvasFrameInfo` | 显示原始/分析帧分辨率与显示缩放 |
| `#confirmRectBtn` | 确认 Logo 选区 |
| `#sampleStep` | 轨迹采样间隔，默认 6 |
| `#thresholdRange` | 模板匹配阈值，默认 0.30 |
| `#interpolationSel` | `smooth` / `linear` |
| `#trackCanvas` | 轨迹预览画布 |
| `#keyframeList` | 关键帧 chip 列表 |
| `#goRestoreBtn` | 进入修复页 |
| `#beforeCanvas` / `#afterCanvas` | 修复前后对比 |
| `#compareSlider` | 对比分割线 |
| `#previewScrub` | 选择预览帧 |
| `#exportBtn` | 导出视频 |
| `#historyView` | 历史记录页面 |
| `#historyBtn` | 顶部历史入口 |
| `#historyFooterBtn` | 底部历史入口 |
| `#historyList` | 历史记录列表 |

### 4.3 主要 state 字段

```js
state = {
  file, meta,                 // 输入文件与 {width,height,duration}
  frames,                     // [{ index, time, sourceTime, blob, size }]
  workW, workH,               // 当前分析分辨率（偶数、绝不放大）
  sourceFrameCount, stride,   // 源帧数与均匀跳帧步长
  sourceAudio,                // MP4 可直接复用的 AAC 样本
  sourceHasAudio,
  refs, finalIndex,
  finalBitmap, logoRect,
  rawDetections, keyframes, rejectedKeyframes, trackSamples,
  trajectory,                 // 逐帧 Logo 位置和来源
  workerPool, coreCount,
  previewReady, previewIndex, previewStats,
  restoreLog,                 // 修复来源/像素数记录
  outputBlob, outputUrl
}
```

## 5. 拆帧管线

### 5.1 MP4 快速路径

使用 mp4box.js + WebCodecs `VideoDecoder`：

1. 读取完整文件到 ArrayBuffer。
2. `MP4Box.createFile()` 解析 meta。
3. 读取第一条视频轨和可选的音频轨。
4. 提取视频样本，按 decode order 喂给 `VideoDecoder`。
5. 通过 `selectedKeys` 选择保留的展示时间点。
6. 对每个输出 VideoFrame：
   - 若旋转为 90/180/270，按显示方向绘制。
   - 主线程绘制到 OffscreenCanvas。
   - `transferToImageBitmap()` 后把 ImageBitmap transfer 给 Worker 池。
   - Worker 负责缩放/JPEG 编码并返回 Blob。
7. 最终按 `sourceTime` 排序，归一化时间轴到首帧 0。

关键点：

- 不能直接放大原始分辨率。`chooseTargetSize(maxSide)`：
  ```js
  const scale = Math.min(1, maxSide / Math.max(sourceW, sourceH));
  workW = even(sourceW * scale);
  workH = even(sourceH * scale);
  ```
- 默认处理最长边 1280。
- 最多保留 2400 个分析帧；超过时 `stride = ceil(sourceFrameCount / 2400)`，均匀跳帧。
- 保留时间戳，不要强行按固定整数帧率重建时间。
- 竖屏视频必须读取转矩阵并旋转，不能只交换宽高。
- 测试文件 `Logo.mp4` 的旋转矩阵通常为 270° 方向；处理完的输出必须是正向竖屏。

### 5.2 备用路径

非 MP4、mp4box 解析失败、解码器不支持时：

- 使用 `<video>` 元素逐帧 seek，画到 OffscreenCanvas。
- 默认备用帧率 30，可在 UI 选择 25/30/60。
- 同样限制 2400 帧；超过时均匀跳帧。
- 这条路径较慢，但必须保留。

### 5.3 音频

MP4 中常见 AAC 音轨：

- 从 mp4box 的 sample entries 提取 `esds` 里的 DecoderSpecificInfo 作为 AAC description。
- 导出 MP4 时直接 `addAudioChunkRaw` 复制原始 AAC 数据，不重新编码。
- WebM 导出时用 OfflineAudioContext 解码原音轨，转 48kHz 双声道，用 WebCodecs `AudioEncoder` 编码 Opus。
- 如果浏览器无 AAC 编码器且源不是可直通的 AAC，MP4 无法带音轨；UI 必须禁用音轨复选框并说明原因，建议改用 WebM。
- 源视频没有音轨时，UI 必须显示“源视频没有音轨”，而不是误报浏览器无法写音轨。

## 6. 参考帧编辑器：必须保持的坐标正确性

### 6.1 曾出现的严重 Bug

旧实现给画布固定 `width: 100%`、`height: auto`、`max-height: 68vh`、`object-fit: contain`。

结果是：

- 竖屏视频的画布元素是宽 735 × 高 748，但 bitmap 以 `contain` 内置黑边显示为约 420 × 748。
- 鼠标坐标却按整个 735 × 748 元素换算，选框位置和大小全部错误。
- 用户感觉“帧分辨率被放大/和原视频不一样”。

### 6.2 当前正确实现

**不要重新引入 `object-fit: contain` 的画布交互。**

当前结构：

```html
<div class="canvas-wrap">
  <div class="canvas-fit" id="canvasFit">
    <canvas id="logoCanvas"></canvas>
  </div>
</div>
```

CSS 原则：

```css
.canvas-fit {
  width: 100%;
  aspect-ratio: 16 / 9;      /* JS 会改写为真实视频宽高比 */
  margin: 0 auto;
  position: relative;
}
#logoCanvas {
  display: block;
  width: 100%;
  height: 100%;
  touch-action: none;
}
```

JS 在选中参考帧后：

```js
fit.style.aspectRatio = `${workW} / ${workH}`;
fit.style.maxWidth = `min(100%, calc(68vh * ${workW / workH}), ${workW}px)`;
```

- `aspect-ratio` 保证显示内容不拉伸。
- `maxWidth` 中的 `${workW}px` 防止视频帧被 CSS 放大到超过原始像素宽度。
- 指针换算仍使用画布真实 `getBoundingClientRect()`：
  ```js
  x = (clientX - rect.left) / rect.width * canvas.width;
  y = (clientY - rect.top) / rect.height * canvas.height;
  ```
- 必须显示帧分辨率和显示缩放，例如：
  `原始 720×1280 · 分析帧 720×1280 · 当前显示 58%（等比缩放，不改变帧像素）`

### 6.3 框选交互要求

- `pointerdown/move/up` + `setPointerCapture`。
- 支持新建矩形、移动矩形、拖动 8 个 handle 调整。
- 选区最小 8×8；实际保存前 clamp 在画面内。
- 选区数值输入与画布双向同步。
- Logo 模板预览应显示最终参考帧的裁剪。
- 确认后保存 `state.logoRect` 和 `state.finalIndex`。

### 6.4 回归测试

使用 `/home/lyorn/Movies/Logo.mp4`：

1. 选择参考帧 #0。
2. 猫 Logo 的视觉范围约为 `x=30, y=88, w=172, h=176`（以视频帧像素计）。
3. 用鼠标围绕猫框选，检查 `state.logoRect` 和数值输入。
4. 结果必须接近上述坐标，不能出现几十像素的系统偏移或明显尺寸比例错误。
5. 在 1440×1100、1366×768、DPR=1.25、页面滚动后各测一次。
6. 画布元素尺寸必须与 bitmap 宽高比一致，不能有内部 letterbox。

## 7. Logo 跟踪算法

### 7.1 特征

不要用原始 RGB 相关性作为主匹配特征。半透明 Logo、背景变化时容易失败。

当前使用：

1. 灰度化。
2. **Sobel 边缘幅值图**。
3. 构造 1×、2×、4×、8× 多尺度金字塔。
4. 模板同样从最终参考帧裁剪，并计算零均值模板和方差。
5. 使用归一化互相关 NCC：
   ```text
   score = Σ(I - meanI)(T - meanT) /
           sqrt(Σ(I - meanI)^2 * Σ(T - meanT)^2)
   ```

### 7.2 搜索

- 从最粗尺度开始全图搜索 top-K 候选。
- 逐级细化到 1/2、1/4、1/8，并做小窗口局部搜索。
- Logo 尺寸固定，不搜索缩放；缩放 Logo 是已知限制。
- 匹配阈值默认 0.30，可在 UI 调整。
- 最终帧是用户提供的 anchor，score=1，不参与匹配。

### 7.3 低置信度与异常值

- `score < threshold` → 标记 rejected。
- 再做连续性过滤：
  - 用前后有效关键帧做线性期望；
  - 若位置误差大，且置信度未明显更高，则移除。
- 被剔除的点不参与最后轨迹，但 UI 中保留展示。
- 关键帧 list 应继续显示低置信/已剔除状态。

### 7.4 插值

- 默认使用 **PCHIP 单调三次 Hermite 插值**。
- 可切换为线性。
- 逐帧计算每个 frame 的 x/y，w/h 使用用户框选尺寸。
- 所有坐标 clamp 在画面内。

### 7.5 Worker 并行匹配

- 主线程只负责：
  - 解码最终参考帧；
  - 构造最终帧特征和模板；
  - 广播匹配配置；
  - 汇总结果。
- `pool.broadcast('matchConfig', { workW, workH, templates })`。
- 每个 sampled keyframe 发送：
  ```js
  pool.run('matchFrame', { index, blob: state.frames[index].blob })
  ```
- Worker 自己解码 JPEG → Sobel 金字塔 → `findTemplate` → 返回 `{index,x,y,score}`。
- 如果 Worker 匹配失败，必须回退到主线程逐帧匹配。
- 测试视频按 step=6 约产生 121 个采样点、111 个有效关键帧（允许少量差异）。

## 8. 修复 / 去 Logo

### 8.1 核心常量

```js
const EDGE_PAD = 4;        // Logo 紧邻边缘扩展
const BLEND_MARGIN = 16;   // 外圈羽化宽度
```

当前修复矩形：

```js
targetRect = paddedRect(trajectory[index], EDGE_PAD + BLEND_MARGIN);
```

即 Logo 四周额外扩展 20px。最后对最外圈 **16px** 做 smootherstep alpha 融合。

### 8.2 跨帧 donor 规则

**只使用完整干净帧，不使用部分遮挡帧拼补丁。**

原因：部分 donor 容易把不同时间的背景块混在一起，造成明显块状接缝。

当前策略：

1. 对每个目标帧找出 Logo mask 与 targetRect 完全不相交的 donor。
2. 有多个 full-clean donor 时，比较目标框周围一圈背景的像素差，选最接近当前帧周围背景的 donor。
3. 从 donor 把整个 targetRect 拷贝到目标帧，跳过 watermarked core。
4. **没有 full-clean donor 时，不要使用部分 donor**；直接让内容感知修补填充整个 targetRect。
5. 记录 donor 来源、矩形和复制像素数，写入 `restoreLog`。

### 8.3 内容感知修补

主修复流程：

1. 用双线性/区域平均建立工作 ROI。
2. 对 residual mask 做 BFS 扩散初始化。
3. 构造多尺度图像层级。
4. `exemplarFill`：从边界向内做基于块的 exemplar 填充。
5. 最终将 exemplar 结果与扩散底图按约 `textureStrength = 0.48` 混合，避免块状拼接。
6. 只回写 residual mask 内的像素。
7. 修复后应用 **16px smootherstep 羽化**：
   - alpha=0 位于 targetRect 外边缘；
   - alpha=1 位于向内 16px 处；
   - 框选 Logo 位于 alpha=1 的区域，不能被原 Logo 污染；
   - 靠近视频边框时，不向画面外羽化。

### 8.4 Worker 修补

- 主线程完成目标帧解码和 donor 复制。
- 若存在 residual，将 full RGBA buffer transfer 给 Worker：
  ```js
  { buffer, width, height, rect, mask }
  ```
- Worker 执行 `contentAwareInpaint`，返回 transfer 后的新 buffer 和 meta。
- 主线程重建 `ImageData`，执行 16px 羽化并编码。
- Worker 内 inpainting 代码必须与主线程实现一致。
- Worker 修补失败时要报明确错误，不能悄悄继续。

### 8.5 边界线验收

- 不能看到明显矩形硬边。
- 用 `Logo.mp4` 检查 0s、10s、20s、29s 附近的成片。
- 猫 Logo 的移动路径位置应完整移除，不应残留边缘。
- 修复区域边缘应有至少 8px 以上的渐变；当前为 16px。

## 9. 导出

### 9.1 编码路径

- **MP4**：H.264 via VideoEncoder + mp4-muxer。
- **WebM**：VP9 via VideoEncoder + webm-muxer。
- 如果所选格式不可用，自动切到另一个可用格式或禁用导出按钮。
- 编码配置必须通过 `VideoEncoder.isConfigSupported()` 探测。

### 9.2 时间轴

- 使用 `frame.time`（相对于首帧 0）的秒数 → 微秒时间戳。
- 保持单调递增，必要时 +1000µs。
- `VideoFrame` 的 duration 由下一帧时间差得出，最后一帧使用帧率倒数。
- **mp4-muxer / webm-muxer 的 `frameRate` 必须是正整数**。
  - 之前出现过的 bug：`fps=30.0003` 直接传 muxer 导致导出失败。
  - 正确做法：`const muxFps = Math.max(1, Math.round(plan.video.fps))`。
- MP4 设置 `fastStart: 'in-memory'`、`firstTimestampBehavior: 'cross-track-offset'`。
- WebM 使用 `firstTimestampBehavior: 'offset'`。

### 9.3 导出并行流水线

为了利用 Worker 和主线程编码：

- 先创建 `N` 个帧修复任务（lookahead）。
- 主线程按顺序等待对应帧完成，再写入 VideoEncoder。
- 每编码完一帧，再补一个新的修复任务。
- lookahead 数量需受内存限制：
  ```js
  const frameBytes = W * H * 4;
  const memoryLookahead = floor(180MB / (frameBytes * 2.2));
  const movingLogo = !movementIsSmall();
  const desired = movingLogo
    ? min(workerCount, 8)
    : workerCount;
  const lookahead = clamp(min(N, desired, memoryLookahead), 1, N);
  ```
- 不要一次性把所有帧都放进 Promise 队列，避免 4K 内存爆掉。
- 输出编码顺序必须保持原始 frame index 顺序。
- 导出完成后对 `restoreLog` 按 frame 排序。
- 下载链接使用 Blob URL；文件名 `${原文件名}-clean.${format}`。

### 9.4 音频

- 源 MP4 + AAC：直接复用 samples，不重新编码。
- 源其他格式：OfflineAudioContext 解码 → 48kHz 2ch → Opus（WebM）。
- AAC 编码器是否可用因浏览器/平台而异。
- 源无音轨时禁用音轨选项并显示正确原因。
- 输出音轨起点以音轨自身首个样本为基准，不要错误地用视频首帧的 MP4 raw CTS 偏移去裁剪音频。

## 10. Web Worker 池

### 10.1 数量

```js
const coreCount = clamp(round(navigator.hardwareConcurrency || 4), 1, 32);
```

- 每个逻辑核心启动一个 Worker。
- 上限 32。
- UI 中应显示检测到的核心数和当前 Worker 数，例如 `WEBCODECS · 22T`。
- 启动前做 `ping` 握手；失败则回退主线程。

### 10.2 任务协议

| type | payload | transfer | result |
|---|---|---|---|
| `ping` | none | none | `"pong"` |
| `extractBitmap` | `{bitmap,targetW,targetH,quality,sourceTime}` | `[bitmap]` | `{blob,size,sourceTime}` |
| `matchConfig` | `{workW,workH,templates}` | 无 | 无响应，Worker 保存配置 |
| `matchFrame` | `{index,blob}` | 无 | `{index,x,y,score}` |
| `inpaint` | `{buffer,width,height,rect,mask}` | `[buffer]` | `{buffer,meta}` |

- Worker 内部按消息 id 返回 `{id, ok, result}` / `{id, ok:false, error}`。
- 主线程 `WorkerPool` 维护 `idle` / `active` / `queue`。
- 一个 Worker 同一时间只执行一个任务。
- Worker 运行异常时应重建 Worker 或让调用方回退。
- 提取任务必须向 Worker transfer ImageBitmap；修补任务必须 transfer ArrayBuffer，注意 transfer 后主线程原 buffer 会 detach。

### 10.3 使用范围

当前 Worker 池用于：

1. 分析帧 JPEG 编码。
2. 关键帧 Logo 模板匹配。
3. 空间内容修补。

导出编码仍在主线程。主线程负责：
- 解码目标帧；
- donor 选择与时间一致性；
- 最终边缘羽化；
- VideoEncoder 编码和封装。

### 10.4 回退

- Worker 不可用 → 主线程 JPEG 转换 / 主线程模板匹配 / 主线程 inpainting。
- Worker 初始化失败 → 弹 toast，不能阻塞流程。
- Worker 匹配异常 → 回退主线程，并清理 partial detections。
- Worker 修补异常 → 抛错，由上层 toast。

## 11. 本地历史

### 11.1 存储要求

- localStorage key：`logoEraserHistory.v1`
- 上限 12 条。
- 只保存 JSON 元信息、设置和轨迹坐标。
- **禁止**存储 File、Blob、ImageData、视频帧、输出成品。
- 大记录保存失败时：
  1. 去掉较早记录的 trajectory；
  2. 再失败则只保留最近 5 条的轻量元数据；
  3. 再失败则提示存储空间不足。
- 浏览器禁用 localStorage 时，历史页显示提示，但其他功能不受影响。

### 11.2 记录 schema

```js
{
  version: 1,
  id,
  savedAt,
  fileName, fileSize, fileLastModified,
  sourceWidth, sourceHeight,
  workWidth, workHeight,
  frameCount, sourceFrameCount, stride,
  duration, fps, extractMethod,
  processWidth,
  sampleStep, threshold, interpolation,
  finalIndex,
  logoRect: { x, y, w, h },
  workerCount,
  trajectory: [[x, y], ...],
  output: {
    format: "mp4" | "webm",
    quality: "standard" | "high",
    audio: boolean,
    name: "xxx-clean.mp4",
    size: number
  }
}
```

### 11.3 写入时机

只在导出成功、Blob 已经生成后写入。  
预览、跟踪完成但没有导出时不写入。

### 11.4 恢复流程

浏览器不允许自动读取本地文件路径，因此：

1. 用户点击「历史」。
2. 点击某条记录的「恢复处理」。
3. 打开文件选择器，要求选择**同名同大小**的原视频。
4. 校验文件名 + 文件大小。
5. 自动重新拆帧。
6. 校验 `frameCount/workWidth/workHeight` 与记录一致。
7. 恢复：
   - sampleStep；
   - threshold；
   - interpolation；
   - format/quality/audio；
   - finalIndex/logoRect；
   - 完整逐帧 trajectory。
8. 直接显示修复页，并自动生成一次预览。
9. 用户可重新导出；历史本身不提供上一次输出文件下载。

如果无法匹配 frameCount 或工作分辨率，或没有 trajectory：

- 回退到参考帧页；
- 恢复 finalIndex/logoRect；
- 让用户重新定位。
- 不能把历史轨迹强行用到不同分辨率的帧上。

### 11.5 历史 UI

- 顶部历史按钮、底部「历史记录」入口。
- 每条记录显示：文件名、导出格式、时间、分辨率、时长、帧数、采样间隔、线程数、输出文件名/大小、是否含音轨。
- 支持单条删除、清空全部。
- 页面必须明确说明：
  - 不缓存视频或成片；
  - 恢复需要重新选择同一个原视频；
  - 记录只存储在本地浏览器。
- 文件名必须通过 `textContent` 写入，禁止拼接为未转义 HTML。

## 12. UI / 视觉约定

- 深色背景 `#090b0f`，卡片 `rgba(17,22,30,.82)`，主色亮柠檬绿 `#c9ff5c`，辅助青色 `#6fe7ff`。
- 中文文案简洁、专业，不要出现“AI 魔法”“一键完美”等夸大。
- 主按钮绿色；危险/警告使用橙色；历史徽章绿色。
- 顶部 sticky header 显示品牌、隐私提示、历史入口。
- 移动端：
  - 隐藏隐私 pill；
  - 历史按钮收成图标；
  - 所有步骤可横向适配；
  - 画布等比缩放，不横向溢出。
- 任何耗时操作必须有进度或忙碌遮罩。
- 忙碌遮罩不要使用大范围 `backdrop-filter` 模糊；实测会显著拖慢视频解码/JPEG 编码。
- Toast 必须 `pointer-events: none`，避免挡住导出按钮。
- 不可缓存或上传提示必须清楚。

## 13. 已知限制（必须如实告知用户）

1. 不是生成式 AI 去水印模型。
2. 静止 Logo 下方复杂纹理没有真实参考帧，只能空间修补，可能留下模糊或纹理不一致。
3. 强烈运镜或背景运动可能导致接缝；16px 羽化只能减弱边缘，不保证完全消失。
4. Logo 尺寸变化、旋转、严重形变不支持；当前跟踪是固定框尺寸。
5. 单次最多 2400 个分析帧；长视频自动均匀跳帧，输出帧率降低。
6. 浏览器内存限制可能限制高分辨率长视频。
7. MP4 音轨可直通仅适用于可访问的 AAC 源；否则依赖平台 AAC 编码器。
8. Safari / Firefox 支持程度不如 Chrome / Edge，尤其是 WebCodecs 编码、AAC、Worker 内 OffscreenCanvas。
9. 历史记录需要原文件重新选择；不缓存成品。
10. 选择参考帧显示是等比缩放，不代表视频帧被重采样；分辨率信息栏必须说明这一点。

## 14. 回归测试清单

### 14.1 启动

- 直接双击 `index.html`，`file://` 下无外部请求。
- 控制台无错误。
- 右下角/底部版本显示 `v1.2`。
- 顶部显示 CPU 线程数，例如 `WEBCODECS · 22T`。

### 14.2 拆帧

用 `/home/lyorn/Movies/Logo.mp4`：

- 输出 720×1280、720 帧、24fps。
- Worker 数 = 当前逻辑核数（测试环境为 22）。
- 拆帧约 3–5 秒；比单线程快数倍。
- 帧 blob 不是黑帧。
- 首帧时间归一化为 0。
- 输出工作分辨率不高于源视频。
- 竖屏画面方向正确。

### 14.3 框选

- 选择参考帧 #0。
- 鼠标框选猫 Logo，结果约为 `x=30,y=88,w=172,h=176`。
- `#canvasFrameInfo` 形如：
  `原始 720×1280 · 分析帧 720×1280 · 当前显示 58%（等比缩放，不改变帧像素）`。
- 在横屏视频、竖屏视频、DPR 1/1.25、不同窗口宽度下各测一次。
- 拖拽后选框必须紧贴鼠标轨迹和 Logo 边缘。

### 14.4 跟踪

- 采样间隔 6 → 约 121 个 sampled keyframes（含 anchor 和最后一帧）。
- 关键帧匹配应在 1 秒内完成；置信度合理。
- 猫 Logo 轨迹向下移动，允许最后几帧因靠近边缘/出画被剔除。
- 调整阈值/插值方式不应重新解码帧，只应重算轨迹。
- 低置信度展示、keyframe chip、轨迹概览图正常。

### 14.5 修复与边界

- 进入修复页后自动生成预览。
- 移动 Logo 应从干净 donor 恢复，不出现块状补丁。
- 边缘 16px 羽化，不能看到明显矩形硬边。
- `restoreLog` 中记录 temporal/inpaint 像素量、donor、rect。
- 静态 Logo 走空间修补，边缘仍应尽量平滑。
- 对 Logo.mp4 检查 0s、10s、20s、29s 成片，不能有猫残留。

### 14.6 导出

- MP4 导出播放正常，分辨率/帧数/时长与状态一致。
- WebM 导出播放正常。
- 有音轨的源（可另造 MP4+AAC 测试）MP4 导出保留音轨；WebM 转 Opus。
- 源无音轨时导出视频无声，UI 不误导。
- 文件可点击下载。
- 历史记录写入成功。

### 14.7 历史

- 完成一次导出。
- 顶部/底部打开历史页，能看到记录。
- 刷新页面后记录仍在。
- 点击「恢复处理」会打开文件选择器。
- 重新选择同名同大小原视频后：
  - 自动拆帧；
  - 恢复轨迹和设置；
  - 直接进入修复页；
  - 自动生成预览；
  - 可以再次导出。
- 选择不匹配文件时应提示并取消。
- 删除/清空记录正常。
- 历史 page 不得发出任何非本地网络请求。

### 14.8 测试环境记录（本机，仅供参考）

- 测试视频：`/home/lyorn/Movies/Logo.mp4`，720×1280，24fps，30s，720帧，无音轨。
- 22 逻辑核心：
  - 720 帧拆帧约 3–4s；
  - 模板跟踪约 0.45–0.8s；
  - 静态 Logo 120 帧导出约 2–4s。
- 这些数字不是性能保证，不应写死进测试断言。

## 15. 常见回归 / 禁止操作

1. **不要把 Logo 编辑器画布改回 `object-fit: contain` + 固定容器宽高**。这会重新引入鼠标坐标偏移。
2. **不要把显示尺寸设得大于帧本身分辨率**，否则用户会误以为视频被放大。
3. **不要在 template / canvas 内部再加 letterbox 后仍使用元素外框做坐标映射**。
4. **不要让 `#canvasFit` 缺少真实宽高比**。每次选中参考帧或 resize 都要设置 aspect ratio。
5. **不要为了边界平滑把修复区域扩大到覆盖 Logo 核心外仍使用原 Logo 像素羽化**。外圈羽化必须在 Logo 选框外侧。
6. **不要恢复“多个部分遮挡 donor 拼接”策略**，除非实现了可靠的无缝融合与重叠裁剪。之前的实现会产生明显补丁块。
7. **不要把 muxer frameRate 传为小数**。
8. **不要在未确认 Worker 可用时移除主线程回退**。
9. **不要在主线程和 Worker 中只改一份 inpainting 代码**。
10. **不要在 Worker 源码中写 `</script>` 字符串**。
11. **不要把视频帧、ImageData 或输出 Blob 写入 localStorage**。
12. **不要引入外部 CDN 依赖**。
13. **不要忽略视频旋转矩阵**。
14. **不要让历史恢复直接应用轨迹而跳过文件/分辨率一致性校验**。
15. **不要发送任何视频数据到网络。**

## 16. 可选后续改进

按优先级：

1. 把主应用和 Worker 共用的 inpainting 核心抽成可构建的单一源，避免双份实现漂移。
2. 增加轨迹手动修正（拖拽关键帧 bbox、增加/删除关键帧）。
3. 用光流/全局位移对齐 donor，改善运镜和相机运动场景。
4. 引入更强的经典 inpaint（Poisson / multiband / PatchMatch EM 多轮）或小型本地模型，改善静止 Logo。
5. 使用 IndexedDB 暂存处理过程中的中间帧，突破单次 2400 帧内存限制。
6. 使用 File System Access API 保存文件句柄，让历史恢复无需重新选择原视频（需用户授权，且不再是纯 file:// 通用方案）。
7. 更细粒度的性能遥测：解码、转换、匹配、修补、编码分段计时。

## 17. Definition of Done

一个任务只有满足以下条件才能算完成：

- 单文件 `index.html` 双击可运行，无外部请求。
- 没有控制台异常。
- 鼠标框选位置、大小、比例与视频帧完全一致。
- 分辨率信息显示正确，帧没有被 CSS 放大或篡改。
- Worker 数量检测与并行路径正常；失败可回退。
- 移动 Logo 能从干净跨帧 donor 恢复。
- 边缘有 16px 羽化，无明显矩形硬边。
- MP4/WebM 导出可播放；音轨场景正确。
- 历史可写入、刷新后可读、可恢复、可删除。
- README 和本 skill 中的限制说明与实际行为一致。
