---
name: logo-eraser-video-watermark
description: Use this skill when working on Logo Eraser, a single-file offline browser video logo/text watermark remover, or when implementing/debugging its video decomposition, per-reference-frame box selection, moving-logo tracking, text-watermark manual keyframes, per-frame text stroke erasure, temporal background restoration, edge feathering, Web Worker acceleration, WebCodecs MP4/WebM export, or local processing history. Trigger on requests about index.html, video watermark removal, logo or text watermark tracking, multi-reference template ensembles, frame restoration, worker pools, or this project.
---

# Logo Eraser 开发与维护 Skill

> 适用版本：v1.4

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
10. 必须支持文字水印模式：参考帧框选文字，分析笔画/颜色，并按帧擦除；文字轨迹允许任意跳变，绝不插值平滑。
11. 文字模式默认使用手动位置点；最新位置点应用到下一位置点之前的每一帧。自动检测只能作为实验功能。
12. 文字模式不做跨帧 donor 取回（相邻帧可能仍有同一水印）；使用笔画 mask + 局部扩散/小块纹理填充，复杂背景允许近似或平滑填充。

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
2. **逐帧框选参考帧**（每个参考帧都要单独框选，选区逐帧缓存）
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
| `#refGallery` | 参考帧卡片（固定间隔抽取，含首帧与末帧） |
| `#refCountSel` | 参考帧数量，1 / 4 / 8 / 10 / 12（默认）/ 16 / 20 / 30 / 40 / 60 |
| `#refSelectAllBtn` / `#refSelectAnchorBtn` | 全部参与 / 仅当前帧 |
| `#jumpUnboxedBtn` | 跳到下一个尚未框选的参考帧 |
| `#resetBoxesBtn` | 清空全部逐帧框选记录 |
| `#refBuildStatus` | 模板合成状态（框选数 / 集成数 / 相关度 / 耗时） |
| `#refBoxStatus` | 已框选帧数 |
| `#logoEditor` | 参考帧框选编辑器 |
| `#refPrevBtn` / `#refNextBtn` | **画面左右两侧的参考帧切换按钮**（也支持 ← → 键） |
| `#refNavLabel` / `#refNavDots` | 当前参考帧序号与可点击的帧指示点 |
| `#canvasFit` | **保持视频宽高比的画布外层** |
| `#logoCanvas` | 真正显示视频帧和绘制选框的画布 |
| `#canvasFrameInfo` | 显示原始/分析帧分辨率与显示缩放 |
| `#logoThumb` / `#thumbCanvas` | 放大后的模板预览，可直接拖动四角微调选框 |
| `#thumbZoomGroup` | 预览缩放 1× / 2× / 4× |
| `.thumb-tools [data-nudge]` | 宽度/高度 ±1px 微调 |
| `#confirmRectBtn` | 确认全部选区并开始定位 |
| `#watermarkModeSel` | `logo` / `text` 水印模式 |
| `#textColorInput` | 文字颜色（框选后可自动识别） |
| `#textTolerance` | 文字颜色容差 |
| `#textFillMode` | 文字擦除后的填充：`auto` / `color` |
| `#textFillColor` | 指定纯色填充颜色 |
| `#textManualMode` | 文字模式手动位置点开关，默认勾选 |
| `#sampleStep` | Logo 轨迹采样间隔，默认 6 |
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
  refs, finalIndex,           // refs: 参考帧池；finalIndex: 当前编辑帧
  refCount, refIndices,       // 参考帧数量 / 参与建模的帧
  refBoxes,                   // { [frameIndex]: {x,y,w,h} } 逐帧框选缓存
  refModels, templates,       // 逐帧模板摘要 / 集成模板金字塔
  refBuildMs, refBuildSignature, textRefs,
  finalBitmap, logoRect,      // 文字模式下为参考文字框
  watermarkMode,              // 'logo' | 'text'
  rawDetections, keyframes, rejectedKeyframes, trackSamples,
  trajectory,                 // 逐帧 Logo 位置和来源
  donorPlan,                  // 逐帧 donor 索引/覆盖率/短过渡
  textColor, textTolerance, textFillMode, textFillColor,
  textManualMode, textKeyframes, textDetections, textMask,
  thumbZoom,                  // 模板预览缩放 1/2/4
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
- 测试文件 `Logo.mp4/Logo.mov` 的旋转矩阵通常为 270° 方向；处理完的输出必须是正向竖屏。

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
- Logo 模板预览显示当前帧裁剪，并可二次微调（见 6.5）。
- 切换参考帧前必须先把当前帧选区写入 `state.refBoxes`。

### 6.4 回归测试

使用 `Logo.mp4` / `Logo.mov`：

1. 选择参考帧 #0。
2. 猫 Logo 的视觉范围约为 `x=30, y=88, w=172, h=176`（以视频帧像素计）。
3. 用鼠标围绕猫框选，检查 `state.logoRect` 和数值输入。
4. 结果必须接近上述坐标，不能出现几十像素的系统偏移或明显尺寸比例错误。
5. 在 1440×1100、1366×768、DPR=1.25、页面滚动后各测一次。
6. 画布元素尺寸必须与 bitmap 宽高比一致，不能有内部 letterbox。

### 6.5 多参考帧：逐帧框选（当前实现）

参考帧不是"选一帧、其余自动定位"，而是**每一帧都由用户框选**。自动定位在淡色/半透明水印上误检率高，逐帧框选把定位误差交给用户，模板质量显著更高。

**抽取规则（固定间隔）**

```js
function pickReferenceFrames(total, count) {
  // count=1 → [0]
  // total<=count → [0..total-1]
  // 否则 step=(total-1)/(count-1)，i*step 四舍五入
  // 强制 out[0]=0、out[last]=total-1
}
```

- 必须是**确定性的固定间隔**，不再随机抖动；同一视频 + 同一数量永远得到同一组帧。
- 必须包含第 0 帧和最后一帧（时间轴两端唯一可用的 donor 来源）。
- 中间帧是离散等距取整，允许因取整少一帧，但不允许重复或越界。

**逐帧缓存**

- `state.refBoxes[frameIndex] = {x,y,w,h}`，切换帧时先 `cacheCurrentRefBox()`。
- 切回某帧必须恢复该帧自己的选区（不能清空）。
- `refBoxFor(index)`：当前编辑帧优先用 `state.logoRect`，其余读 `refBoxes`。
- 清空按钮只删当前帧的缓存；"重置全部框选"清空整个 `refBoxes`。

**帧切换**

- 画面左右两侧各一个按钮（`#refPrevBtn` / `#refNextBtn`），位于 `.canvas-wrap` 内绝对定位，不遮挡画布中心。
- 到首/末帧时对应按钮 disabled，不循环。
- 支持键盘 ← →；输入框聚焦时不拦截。
- 底部 `#refNavLabel` 显示 `参考帧 n / N · #0042`，`#refNavDots` 提供可点击的帧指示点（已框选为亮色，当前帧为青色）。

**模板合成（buildReferenceModels）**

1. 收集 `refIndices ∪ {finalIndex}` 中有合法选区（≥8×8）的帧；没有选区的帧保留在 `refModels` 里但 `boxed:false, dropped:true`，不参与合成。
2. 对每个已框选帧：`createImageBitmap` → `buildFeaturePyramid` → **`alignRefToAnchor` 对齐** → `makeTemplates(features, alignedRect, mask)`。
3. `corr = templateCorrelation(anchorTemplates[0], templates[0])`，anchor 取"当前编辑帧（若有选区）否则第一个已框选帧"。
4. **只有 `corr >= REF_LOGO_CORR(0.20) / REF_TEXT_CORR(0.10)` 的帧进入集成**。相关度低说明该帧框到的不是同一内容，混进去只会让模板变糊；这些帧在卡片上显示"偏差大"。
5. 只有与 anchor **像素尺寸完全一致**的模板能平均（`t.length === anchorLen`），不一致的标 `尺寸不一致` 并 dropped，保证"集成 N"不虚报。
6. `state.templates = buildEnsembleTemplates(list)`，`REF_MAX_ENSEMBLE = 32`。
7. 状态栏文案：`框选 6 / 8 · 集成 6 · 相关 0.70 · 0.1s`，缺失/偏差帧追加 `N 帧未框选`、`N 帧偏差大` 并标 warn 色。

**必须先对齐再平均（关键修复）**

模板是 Sobel 边缘图，手画框在帧间**不可能**像素一致。实测（Logo.mov，172×176 框）：
- 框误差 ±4px 时，同一 Logo 的 `corr` 从 `0.70` 掉到 `0.17` —— 低于 0.20 门限，于是 6 个参考帧只剩 2 个进集成，表现和"只有一个参考帧"一样；
- 侥幸过门限的那几个是**错位**裁剪，平均后模板变糊，NCC 峰值变平，跟踪反而更差。这正是"多参考帧还不如单帧"的根因。

修复：`alignRefToAnchor(features, anchorTemplates, refRect, radiusPx)` 在 `start=2`（scale 4）层用 anchor 模板滑窗找最佳位置，再 `refineCandidate` 回到 0 层；随后**一律用 anchor 的 w/h 裁剪**。修复后同一组 ±4px 手画框得到 `corr 0.67/0.73/0.78`、`集成 4`，与像素级精确的框结果完全一致。

- 对齐搜索半径取 `max(10, 0.14 * max(w,h))`，足以覆盖手画误差，又不会跑到画面别处。
- `refModels[i].rect` 存**对齐后**的位置（不是用户原始框），文本模式的搜索中心与卡片状态都用它。
- 对齐后仍要做 `corr` 门控：框完全画错（不同内容）时对齐也救不回来，必须继续排除。

**为什么参考帧越多越准**

每个参考帧贡献一个零均值/单位方差模板，按 `clamp(corr,.25,1)` 加权平均。背景噪声在 N 帧之间不相关，平均后幅度约降到 `1/√N`，而水印结构在所有帧里一致、被保留。因此集成模板的 NCC 峰值更尖锐，弱水印也不容易被背景边缘抢走。代价是 `createImageBitmap` + 金字塔的耗时与内存随 N 线性增长。

**模板预览（放大 + 二次微调）**

- `#thumbCanvas` 高度 208px（旧版 104px），显示"选区 + 上下文"。
- 缩放是**上下文比例**：`pad = 0.9 / zoom`，1× 留很多环境、4× 紧贴选区，然后整体缩放填满预览。
- 四角有 7px handle：拖角改尺寸、拖内部平移，写回 `state.logoRect` 并同步数值输入。
- 拖动期间必须冻结预览变换（`thumbFreeze`），否则预览会跟随选框重新居中，导致指针增量被反复放大。
- `data-nudge="w:±1|h:±1"` 提供 1px 步进微调。
- 文字模式下预览叠加笔画先验（红色）。

### 6.6 多参考帧回归测试

1. `refCount=8`、2250 帧 → `refs === [0,321,643,964,1285,1606,1928,2249]`。
2. 点第 1 帧框选 → `refBoxes[0]` 写入；点"下一个"→ 新帧 `logoRect === null`；点"上一个"→ 原选区恢复。
3. 只框选 1 帧就确认 → 状态栏出现 `5 帧未框选`，跟踪仍可用（单参考帧等价旧行为）。
4. 用同一坐标框选所有参考帧（静态 Logo 视频）→ 6 帧 `corr ≈ 1.00`，`集成 6`。
5. 用错误坐标框选其余帧 → 这些帧 `corr ≈ 0`，被标"偏差大"并排除，集成数降为 1。
6. `Logo.mov`：仅框 #0 的轨迹与"按轨迹逐帧框选全部参考帧"的轨迹应逐帧一致（≤1px）。
7. **手画误差回归（必须）**：把每个参考帧的框在真值位置上随机偏移 ±4px 后重跑，`corr` 与"精确框"一致（约 0.67–0.78），`集成` 数量不下降。若 `corr` 掉到 0.1x，说明 `alignRefToAnchor` 被绕过或失效。
8. 集成模板的 `tw/th` 必须等于 anchor 选区的尺寸；`集成 N` 只能统计真正参与平均的帧数。

### 6.7 手动轨迹（logo 模式）

自动整帧搜索在背景复杂或水印会跳变的素材上会跑偏。手动轨迹把"位置"交还用户，自动匹配只做**局部**微调。

- 开关：`#trajManualMode`；细化开关 `#trajRefineMode`（默认开）；清除按钮 `#clearTrajBtn`。
- 打开时若还没有关键帧，自动用 `{index: finalIndex, x: logoRect.x, y: logoRect.y}` 播一个种子关键帧，并立刻 `applyManualTraj({noRefine:true})` 生成轨迹（否则轨迹画布不可见，用户无法点击）。
- 在 `#trackCanvas` 上 `pointerdown` 记点：`x = clickX - logoRect.w/2`，同一帧再次点击是**替换**该帧的关键帧。
- 点完立刻重建轨迹但**不做细化**（保证点击即时响应）；细化只在点"开始定位"时跑一次。

**轨迹生成（applyManualTraj）**

```js
// i <= first.index      → 保持第一个关键帧
// i >= last.index       → 保持最后一个关键帧
// 之间                   → 按 index 做线性插值
// w/h 恒等于 logoRect 的尺寸（严格按用户框的范围）
```

- 尺寸**不允许**在关键帧之间变化：用户框多大，轨迹框就多大。
- 细化：`radius = clamp(max(w,h)*0.09, 8, 28)`，以插值点为搜索中心做局部 NCC；只有 `score >= trackerThreshold*0.75` 才采纳，否则保留插值位置。因此自动匹配**不可能**跑到别的位置。
- 细化走 Worker：`pool.run('matchFrame', {index, blob, centers:[{x,y}], centerRadius})`，`matchFrameTask` 优先用 **payload 里的 centers**，其次才是全局 `cfg.centers`（文本模式用）。
- `source` 标记：`anchor` / `detected`（细化成功）/ `manual`（插值或关键帧），轨迹预览据此上色与显示标签。

**回归**

1. 打开手动开关 → `trajKeyframes.length === 1`、`trajectory.length === frames.length`、`#trackResults` 可见。
2. 在 #160 与 #320 各点一次 → 两个关键帧；`trajectory[80]` 必须等于两点的中点（线性），范围外等于端点。
3. `trajectory[i].w/h` 恒等于参考框尺寸。
4. 点"开始定位" → 只有 `score` 达标的帧变成 `detected`，其余仍是 `manual`，且没有任何帧偏离插值路径超过 `radius`。
5. 关闭开关 → 轨迹清空、结果区隐藏，回到自动模式。

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

## 7A. 文字水印模式

### 7A.1 交互与状态

- 在 `#watermarkModeSel` 选择 `text` 后，参考帧编辑器仍然用于框选文字框。
- 默认 `textManualMode=true`。确认选区后进入轨迹页，用户滚动到 `#trackCanvas`，点击文字中心记录位置点。
- `#trackCanvas` 的 pointerdown：
  - 取得当前 `#trackScrub` 帧号；
  - 把点击坐标减去 `logoRect.w/2`、`logoRect.h/2` 得到左上角；
  - 写入 `state.textKeyframes`，同一帧重复点击会替换，按帧号排序。
- `applyManualTextKeyframes()` 为每一帧选择“帧号不大于当前帧的最新位置点”，生成 `textDetections` 和 `trajectory`。**不得在这些点之间插值或平滑**。
- 自动文字检测仅实验：全画面搜索，低置信度帧保留原样。默认不要把手动模式改回自动而不加提示。

### 7A.2 参考笔画先验

`buildTextMaskFromBitmap(bitmap, rect, {updateColor})` 在每个**已框选**的参考帧文字框内：

1. 计算灰度、4px box blur 和高通 `hp = gray - blur`。
2. 根据自动/手动文字颜色判断 bright/dark，并统计高响应像素。
3. 核心笔画条件：
   - 高响应方向正确（`hp` 或 `-hp` 超过阈值 `max(6, min(18, maxHp*0.18))`）；
   - 亮度不与文字颜色方向矛盾（bright 时跳过 `gray<=55`，dark 时跳过 `gray>=200`）；
   - **饱和度不超过框内背景**（见下）。
4. 核心过密时提高阈值重算，最多两次（>26%、>40% 框面积）。
5. **封闭空洞填充**：高通只标记边缘，粗笔画的内部不会被命中，擦除后会留下亮块。用"从 ROI 边界对非核心像素做洪泛"找出所有被核心完全包围的区域，把面积 ≤ `clamp(coreCount*0.9, 24, 900)` 的封闭区并入核心（字形内部与字母封闭区一起补上）。
6. 对核心做 2px 膨胀，形成 `textMask.data`。
7. 保存 `anchorX/anchorY`、`alignW/alignH`、`satLimit/boxSat`，以及笔画核心的高通特征 `alignX/alignY/alignF`，用于逐帧局部对齐。

**饱和度必须相对背景，不能写死绝对值**：

```js
boxSat = 框内饱和度的中位数;
satLimit = bright ? clamp(boxSat - 0.05, 0.20, 0.46)
                  : clamp(boxSat - 0.02, 0.26, 0.52);
```

早期实现用 `satLimit = 0.16`（bright）。在暖色/高饱和背景上，半透明白字被背景染色后饱和度约 0.2–0.35，结果 `coreCount === 0`，整条文字擦除链路静默失效（`textPixels: 0`，画面完全没变化）。绝对阈值只在纯中性背景上成立，必须避免。

不要退回“按颜色距离接近文字颜色”的绝对阈值：半透明白字在深色背景上会变成灰色，绝对距离会漏掉大部分笔画。

**多参考帧合并**：`mergeTextMasks(anchorMask, textRefs)` 在各帧自己的框坐标里投票，某像素需要 ≥40% 的参考帧支持才保留；只有"零票"的锚点像素会被移除。这样既能去掉单帧压缩噪声，又不会因为一帧模糊而删掉真实笔画。

### 7A.3 逐帧位置/尺度微调

`refineTextRect(targetData, det, opts)`：

- 以手动/自动检测框中心为基准做搜索。用参考笔画高通特征与目标帧局部高通做 NCC。
- 采用 2px 偏移网格 + 粗尺度网格，再细化；3px 粗网格可能只锁到两行文字中的一行而向下错 10px。
- 对偏移和尺度加正则，避免两个相近峰值时离开用户点击太远。
- 只有最佳分数 ≥0.12 且比原地分数高 0.025 才应用。
- 输出检测可带 `scale`、`w/h` 会被同步缩放；mask 采样围绕中心按 scale 映射。

**手动点必须严格（strict）**

手动点击本身已经是"水印就在这里"的明确声明，通用搜索窗口（±14px、0.75–1.25 倍）在淡色水印上会把好框拖到邻近边缘，表现为"框和我画的不一致"。

| 参数 | 自动检测 | 手动点（strict） |
| --- | --- | --- |
| 偏移上限 | ±14px | ±4px |
| 尺度范围 | 0.75–1.25 | 0.90–1.10 |
| 偏移惩罚 | 0.0025/px | 0.02/px |
| 尺度惩罚 | 0.2 | 0.9 |

- 由 `textRestoreFrame` 传入：`refineTextRect(targetData, coarse, { strict: !!coarse.manual })`。
- 实测（Logo-Text.mp4，框 `468,398,112,42`）：strict 下 #1775/#1790/#1800/#1810 的偏移为 `-2/+0/+0/+2`、`+1/0/0/+3`，`textPixels` 2471–2865；改动前同样输入会漂到 ±14px。
- 手动模式下若仍觉得框在动，先确认 `det.manual` 是否为 true（`applyManualTextKeyframes` 写入），再检查 `textMask.alignF.length >= 24`（笔画点太少时 `refineTextRect` 直接返回原框）。

### 7A.4 擦除与填充

`textRestoreFrame()`：

1. 解码目标帧，先对检测做局部微调。
2. 把参考 mask 的坐标围绕参考框中心，按目标框中心和 `scale` 映射到当前帧。
3. `mask` 为笔画先验；`dilated` 只额外扩 1px。mask 内 alpha=1，额外一圈 alpha≈0.55。
4. 不使用 temporal donor。相邻帧仍可能包含同一文字；复制会把文字带回来。
5. `textFillMode='color'`：用指定纯色填充。
6. `auto`：
   - 在文字框 ROI 内把已知像素做 `fillHolesDiffusion`，得到平滑底图；
   - 只有当空洞面积 ≤ ROI 的 22% 时才运行小型 `patchMatchInpaint`（2 iterations, refine）：空洞大时可用源纹理不足，PatchMatch 会凭空造出亮块，实测最差误差 64（扩散仅 27）；
   - 混合时再加一道守卫：仅当 `|textured - diffused|` 均值 ≤16 时按 0.62 权重采用纹理，否则该像素退回纯扩散。没有这道守卫，粗笔画水印擦完会留下明显的亮条。
   - 只回写 mask/扩边像素。
7. 记录 `textPixels`、`rect`、`fillMode` 到 `restoreLog`。

### 7A.5 历史与导出

- 历史记录保存 `mode`、文字颜色/容差/填充设置、`textKeyframes` 和完整 trajectory。
- 文字历史恢复时重新拆帧、重建参考 bitmap、重新 `prepareTextReference(false)`，再用 `applyManualTextKeyframes()` 重建检测；不依赖保存的 mask 像素。
- 文字模式导出仍使用同一 `startExport()` 和 WebCodecs；`restoreFrame()` 在文字模式分派到 `textRestoreFrame()`。
- 长文字视频同样受 2400 帧上限限制；尚未实现分段导出合并。

## 8. 修复 / 去 Logo

### 8.1 核心常量

```js
const EDGE_PAD = 4;        // Logo 紧邻边缘扩展
const BLEND_MARGIN = 16;   // 外圈羽化宽度
const RESTORE_PAD = 20;    // targetRect = Logo + 20
const CORE_PAD = 8;        // 允许 inpaint 的 Logo 核心
const DONOR_PAD = 8;       // donor 自身 Logo 的排除边距
const DONOR_FADE = 6;      // donor 边缘软融合
const DONOR_LAG = 36;      // 固定 donor 时间偏移（逐帧推进）
```

当前修复矩形：

```js
targetRect = paddedRect(trajectory[index], RESTORE_PAD);
coreRect   = paddedRect(trajectory[index], CORE_PAD);
```

即 Logo 四周额外扩展 20px；但只有 `coreRect` 内、donor 未覆盖的像素才进入 inpaint。周围本来干净的像素保留当前帧。最后对最外圈 **16px** 做 smootherstep alpha 融合，并使用圆角外轮廓。

### 8.2 跨帧 donor 规则

**使用固定时间偏移的单个 donor，让它逐帧推进；绝不恢复“多个部分遮挡 donor 拼接”。**

原因：旧实现只接受 full-clean donor，而最近完整干净帧常常在 20–70 帧之外，并且会被连续多帧复用。结果是暂停看单帧正常，播放时补丁内容冻结几帧再跳变；动态背景上非常明显。

当前策略：

1. 目标帧 `t` 使用 `donor = t + DONOR_LAG`（默认 36，接近片尾时改为 `t - DONOR_LAG`）。donor 随目标逐帧 +1，因此补丁内容是连续运动的。
2. donor 可以是**部分遮挡**的：只复制其 Logo mask（`DONOR_PAD=8`）之外的像素，并对 donor 边缘做 6px 软融合。
3. 只有 `coreRect`（Logo+8）内、donor 未覆盖的像素才标记为 residual 并 inpaint；targetRect 内、core 外的干净像素保留当前帧，不送进修补器。
4. 如果固定偏移的 donor 对核心覆盖率 < 0.7，在 ±`[DONOR_MIN_LAG, DONOR_MAX_LAG]` 内寻找覆盖率更高的 donor；仍然不足时回退到 nearest full-clean donor，最后才接受核心 inpaint。
5. 仅在片尾/方向切换处做最多 3 帧的短交叉淡化，避免明显的冻结或跳变；不要做长时间 dissolve，会造成补丁模糊。
6. 记录 donor 来源、矩形、覆盖率和复制像素数，写入 `restoreLog`。

已知取舍：`DONOR_LAG` 越大，长遮挡时真实像素越多、补丁越清晰，但 donor 动画相位与当前帧的差距越大。`36` 是测试视频（动态壁纸、猫 Logo 长遮挡）的折中值；调整时只改这一个常量。

### 8.3 内容感知修补

主修复流程：

1. 用双线性/区域平均建立工作 ROI。
2. 对 residual mask 做 BFS 扩散初始化。**residual mask 只覆盖 coreRect 内 donor 未覆盖的像素，不再覆盖整个 targetRect。**
3. 构造多尺度图像层级。
4. `exemplarFill`：从边界向内做基于块的 exemplar 填充。
5. 最终将 exemplar 结果与扩散底图按约 `textureStrength = 0.48` 混合，避免块状拼接。
6. 只回写 residual mask 内的像素。
7. 将 donor 颜色按 donor alpha（0–1，6px 软边）叠加到 inpaint 底图上，避免 donor/inpaint 之间的硬缝。
8. 修复后应用 **16px smootherstep 羽化**：
   - alpha=0 位于 targetRect 外边缘；
   - alpha=1 位于向内 16px 处；
   - 框选 Logo 位于 alpha=1 的区域，不能被原 Logo 污染；
   - 靠近视频边框时，不向画面外羽化；
   - 不贴边时使用圆角矩形距离场，避免明显直角矩形。

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
- 逐帧检查 `state.donorPlan`：正常区间内 donor 应基本满足 `plan[i].index = plan[i-1].index + 1`，不能出现长时间相同 donor 或每几帧大跳。
- 动态背景视频：暂停时单帧正常但播放时仍能看出区域，属于动画相位缺失；不要用继续加宽羽化来“解决”，那只会扩大补丁足迹。

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
  mode: "logo" | "text",
  finalIndex,
  logoRect: { x, y, w, h },
  refCount,
  refIndices: [0, 321, ...],
  refBoxes: { "0": { x, y, w, h }, "321": { x, y, w, h } },
  refModelCount,
  workerCount,
  textManual, textColor, textTolerance, textFillMode, textFillColor,
  textKeyframes: [{ index, x, y, w, h }],
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
   - refCount/refIndices/refBoxes（逐帧框选缓存，key 为帧号字符串）；
   - 完整逐帧 trajectory。
8. 直接显示修复页，并自动生成一次预览。
9. 用户可重新导出；历史本身不提供上一次输出文件下载。
10. `mode:'text'` 时：恢复颜色/容差/填充设置，重新建参考 mask，调用 `applyManualTextKeyframes()` 恢复位置点和逐帧检测；不要走 Logo 的插值/ donor 路径。

`restoreRefSetup(record, finalIndex)` 的容错要求：

- `refBoxes` 的 key 必须校验为 `[0, frameCount)` 内的整数，非法 key 直接丢弃。
- 每个 box 经 `normalizeRect` clamp，`w/h < 8` 的丢弃。
- 旧记录（无 `refBoxes`）用 `record.logoRect` 作为 `finalIndex` 的选区种子，保证至少一帧可建模。
- `finalIndex` 必须同时在 `refs` 与 `refIndices` 里。

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
3.1. **动态背景（重要）**：如果背景本身持续动画（动态壁纸、AI 动画、头发/水波/花瓣），被 Logo 挡住的动画相位在源视频中不存在。固定偏移 donor 可以保证补丁逐帧运动、不冻结，但补丁内容与周围仍可能存在约 `DONOR_LAG` 帧的相位差；这是信息缺失，不是羽化/调参能完全消除的。
4. Logo 尺寸变化、旋转、严重形变不支持；当前跟踪是固定框尺寸。
5. 单次最多 2400 个分析帧；长视频自动均匀跳帧，输出帧率降低。
6. 浏览器内存限制可能限制高分辨率长视频。
7. MP4 音轨可直通仅适用于可访问的 AAC 源；否则依赖平台 AAC 编码器。
8. Safari / Firefox 支持程度不如 Chrome / Edge，尤其是 WebCodecs 编码、AAC、Worker 内 OffscreenCanvas。
9. 历史记录需要原文件重新选择；不缓存成品。
10. 选择参考帧显示是等比缩放，不代表视频帧被重采样；分辨率信息栏必须说明这一点。
11. 文字模式必须如实说明：位置点之间不插值；漏点会擦错位置或留下文字。
12. 文字自动检测是实验功能，低透明度、文字/背景颜色接近、前景遮挡时可能漏检。
13. 文字局部对齐只覆盖约 0.75–1.25 倍尺寸变化；明显缩放需要重新选参考帧。
14. 文字模式不做跨帧 donor；复杂/动态背景只能近似填充，可能留下局部平滑痕迹。
15. 文字模式目前没有分段导出/合并，超过 2400 帧仍会均匀跳帧。

## 14. 回归测试清单

### 14.1 启动

- 直接双击 `index.html`，`file://` 下无外部请求。
- 控制台无错误。
- 右下角/底部版本显示 `v1.4`。
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

多参考帧逐帧框选：

- 卡片徽章依次为 `编辑中`（青）/ `已框选`（绿）/ `待框选`（橙）/ `偏差大`。
- 画面左右两侧的 ← → 按钮可切换参考帧，首/末帧时对应按钮 disabled。
- 切换后：新帧无框选时 `logoRect === null` 且确认按钮 disabled；返回上一帧必须恢复原选区。
- `#refBoxStatus` 与 `#refCountLabel` 的计数随框选实时更新。
- 模板预览高度约 208px；1×/2×/4× 切换时选框占预览的比例明显变化。
- 在预览里拖四角 → 尺寸变化；拖内部 → 平移；拖动过程中预览不应"追着选框跑"（1:1 跟手）。
- `−W/+W/−H/+H` 每次精确改变 1px。
- 只框 1 帧就确认 → 提示"还有 N 个参考帧未框选"，仍可继续。

### 14.4 跟踪

- 采样间隔 6 → 约 121 个 sampled keyframes（含 anchor 和最后一帧）。
- 关键帧匹配应在 1 秒内完成；置信度合理。
- 猫 Logo 轨迹向下移动，允许最后几帧因靠近边缘/出画被剔除。
- 调整阈值/插值方式不应重新解码帧，只应重算轨迹。
- 低置信度展示、keyframe chip、轨迹概览图正常。
- 参考帧相关度：同一 Logo 位置框选多帧时 `corr` 应 > 0.5；坐标框错时 `corr ≈ 0`，该帧必须被排除出集成（`集成` 计数下降）。

### 14.5 修复与边界

- 进入修复页后自动生成预览。
- 移动 Logo 应从固定偏移 donor 恢复，不出现块状补丁；连续帧的 donor 索引应基本 +1，而不是冻结/大跳。
- 只有 Logo 核心进入 inpaint；`targetRect` 内、核心外的干净像素必须保留当前帧。
- 边缘 16px 羽化 + 圆角外轮廓，不能看到明显矩形硬边。
- `restoreLog` 中记录 temporal/inpaint 像素量、donor、覆盖率、rect。
- 静态 Logo 走空间修补，边缘仍应尽量平滑。
- 对 Logo.mp4 检查 0s、10s、20s、29s 成片，不能有猫残留。
- 动态背景视频：暂停正常但播放仍有区域感时，先确认 donor 路径是否逐帧推进，再判断是否为动画相位缺失；不要靠继续加宽羽化掩盖。

### 14.5A 文字水印模式

测试视频：`Logo-Text.mp4`，2250 帧，720×1280，30fps，75.1s。

- 参考帧按固定间隔抽取（12 帧时为 `0,204,409,…,2249`），逐帧框选文字水印。
- 「MERGE COOKING」非常淡（笔画与背景只差约 6–12 灰阶），直接框选前先在 4× 模板预览里确认能看到笔画；框在空白处会得到 `coreCount = 0`（这是正确行为，不是 bug）。
- 若 `coreCount === 0`：先怀疑框选位置，再怀疑阈值。不要用调大阈值来"修复"。
- **该片是多段拼接的广告合集**，同一个「MERGE COOKING」在不同片段的**位置不同**。实测（720×1280 坐标）：
  - 约 60s（拼图关卡，水印压在深色横条上）：`x≈468, y≈398, w≈112, h≈42`。已验证笔画先验准确落在字形上：`coreCount ≈ 835 / 4704 px ≈ 18%`，`threshold ≈ 12.8`，`boxSat ≈ 0.19` → `satLimit = 0.20`（下限被夹住），`bright = true`。下半行「COOKING」被游戏 UI 面板压住一部分，属正常。
  - 同一帧跑完整擦除链路（`buildTextMaskFromBitmap` → `textRestoreFrame`）：`coreCount 879`、`textPixels 2672`（57% 选框面积）、文字**完全消失**；但框底边切到了紫色 UI 面板的顶边，扩散填充会把面板颜色向上抹开成一片紫色渐变（`meanDiff ≈ 24`）。这是「框内包含高对比度边缘 + 第二行被面板遮挡」的固有限制：被遮住的那半行本来就不存在，只能凭空生成。
  - 结论：文字框应尽量贴紧文字、**避开面板/字幕条等强边缘**；被 UI 压掉一半的水印，请接受局部平滑痕迹。
  - 已尝试过的无效方案（不要重复）：在 `buildTextMaskFromBitmap` 里"剔除长而细的强阶跃边缘"（按行/列找 ≥`max(12,0.15*rw)` 的连续 core 段 + 厚度 ≤3 + 上下 ±4 行灰度差 ≥20）。在干净画面上能剔掉 24 px，但在**实际拆帧的 JPEG 帧上完全无效**（`runs: []`，边缘被 `satLimit` 的饱和度门控打碎成短段），对 `coreCount` 与成片毫无影响，已回退。
  - 约 20s（篝火场景）：位置不同，约 `x≈218, y≈842, w≈90, h≈52`。但背景是**明亮火焰**，高通响应被火花和火苗主导，先验会混入明显误检（`coreCount ≈ 492`，含框右侧背景噪点）。**这类高亮/高频背景不适合做文字模式的验证样本**，别用它判断算法好坏。
  因此**单个参考框不可能覆盖全片**：整片回归会在大跨度场景切换处丢失目标，这是素材本身的限制，不是跟踪 bug。整片验证请改用下面的合成水印自检；真实片只适合做「同一片段内」的定点抽查（例如只取 55s–70s 这段连续镜头）。
- 定点抽查时，参考帧也要落在同一片段内；跨片段的参考帧会因为 `corr` 低于阈值被标「偏差大」并排除（这是预期的门控行为）。
- 手动位置点必须覆盖每个**片段边界**（示例 `0/150/750/900/1800` 只是历史用例，实际以场景切换帧为准）；最新点应用到下一帧点之前。
- 位置点不得被插值平滑；在跳变帧附近检查 `textDetections` 的 x/y 是否逐帧保持点击值。
- 同一片段内的预览检查：文字应消失，背景只留下局部平滑痕迹，不应有完整文字或大矩形硬边。
- `coreCount / 选框面积` 的量级参考：真实淡字约 **10%–20%**；合成粗体白字约 **45%–55%**。低于 5% 基本说明框没对准，或该处水印几乎不可见。
- `textPixels` 不应等于整个文字框面积。
- 自动文字检测只作为实验对比；不要在回归中要求它稳定覆盖全部帧。
- 文字模式导出 MP4 后应能播放，分辨率/时长正确，源音轨保留。
- 历史记录应保存 `mode:'text'` 和位置点；重新选择原视频后恢复到文字修复页。

**合成水印自检（推荐，可复现）**：真实水印太淡时，用 `_test.buildTextMaskFromBitmap` / `_test.textRestoreFrame` 在真实帧上叠一行白色粗体文字来验证链路：

1. 背景取 `frames[0]`，在 `{x:250,y:900,w:220,h:60}` 内画 `bold 34px` 白字。
2. `buildTextMaskFromBitmap(bmp, box, {updateColor:true})`：
   - `recall`（真值像素被 mask 覆盖比例）必须为 1.0；
   - `coreCount` 应接近真值像素数（字形内部已被封闭空洞填充补上）；
   - `satLimit` 由 `boxSat` 推出（本图约 `0.35`），不应是固定 0.16。
3. `textRestoreFrame(0)`：框内平均误差 ≤3（0–255 尺度），不应出现残留亮条；
   早期 `patchMatch` 无守卫时最差误差 64，加上守卫后约 28。
4. 框外的文字（例如故意把框画窄切掉最后一个字母）必须保持原样——工具只擦框内。

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
6. **不要恢复“多个部分遮挡 donor 拼接”策略**，除非实现了可靠的无缝融合与重叠裁剪。之前的实现会产生明显补丁块。当前允许的是“单个部分 donor + 核心 inpaint + donor 软边”，不是把多帧部分像素拼成一个补丁。
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
8. 长视频分段处理：按时间切成若干段顺序修复/编码，再合并音视频；当前仍未实现，超过 2400 帧只能跳帧。
9. 文字模式可增加更强的时间一致性：在位置点之间不做插值，但可在同一位置段内寻找真实干净背景 donor，并用环带一致性校验避免复制回水印。

## 17. Definition of Done

一个任务只有满足以下条件才能算完成：

- 单文件 `index.html` 双击可运行，无外部请求。
- 没有控制台异常。
- 鼠标框选位置、大小、比例与视频帧完全一致。
- 分辨率信息显示正确，帧没有被 CSS 放大或篡改。
- Worker 数量检测与并行路径正常；失败可回退。
- 移动 Logo 能从固定偏移、逐帧推进的 donor 恢复；`donorPlan` 正常区间基本 +1，不出现长时间冻结/大跳。
- 只有 Logo 核心进入 inpaint，周围干净像素保留当前帧。
- 边缘有 16px 羽化 + 圆角外轮廓，无明显矩形硬边。
- 动态背景的相位缺失已在 README / UI 限制中如实说明。
- 文字模式支持手动位置点，点与点之间不插值；跳变后不补点时必须明确提示可能残留。
- 文字参考笔画先验低饱和过滤有效，不把彩色 UI 整块当文字；逐帧 NCC 可修正小范围点击误差和尺寸变化。
- 文字擦除只影响笔画 mask 及软边，填充使用局部扩散/纹理，不重新引入相邻帧水印。
- 文字模式 MP4/WebM 导出可播放；音轨场景正确。
- 历史可写入、刷新后可读、可恢复、可删除；文字历史能重建位置点。
- README 和本 skill 中的限制说明与实际行为一致。
