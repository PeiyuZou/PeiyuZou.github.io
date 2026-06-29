# UGUI 重建（Rebuild）与重绘（Repaint）及列表优化

## 题目

UGUI 的「重建 Rebuild」与「重绘/合批 Repaint」分别在什么情况下发生？一个 UGUI 列表滑动卡顿，你会从哪些角度排查与优化？（高频）

## 参考答案

### Rebuild（重建）

当 UI 元素的**内容/布局变化**时，会被标记为脏（`SetVerticesDirty` / `SetLayoutDirty`）。在 `Canvas.SendWillRenderCanvases`（Update 之后）阶段，Canvas 批量重建——重新计算顶点、UV、三角网格、布局。属于 **CPU 开销**。触发原因：文字内容变化、Image 的 Sprite/Color 变化、子元素增删、LayoutGroup 内子项变化等。

### Repaint（重绘）

把重建后的网格按材质合批后提交 DrawCall 实际绘制。即便内容没变，只要 Canvas 在动（如摄像机/元素移动），每帧仍会 Repaint。一个 Canvas 至少一次 Repaint。

> 关键区别：**Rebuild 是「网格/布局重新计算」，Repaint 是「绘制提交」**。位置移动通常不触发 Rebuild（网格内容不变），但每帧 Repaint。

### 列表滑动卡顿的排查与优化

1. **对象池复用可见项**：用 Loop/EnhancedScroller 这类只实例化可见几行 + 复用的方案，避免成百上千 item 实例化。
2. **拆分 Canvas**：动态列表与静态背景分到不同 Canvas，避免一个动态元素让整个大 Canvas 每帧 Rebuild。
3. **减少 Rebuild**：滑动时不要每帧改 Text/Image 内容；Item 内容变化时只对单项 dirty。
4. **关闭无用 RaycastTarget**：不需要交互的 Image/Text 关闭 `RaycastTarget`，降低 EventSystem 每帧射线检测。
5. **慎用 LayoutGroup**：Vertical/Horizontal/Grid LayoutGroup 布局计算贵，复杂列表用手动定位或 ContentSizeFencer + 限制层级深度。
6. **用 TextMeshPro 替代 Text**：避免动态字体图集每帧重建；TMP 用 SDF、图集稳定。
7. **Mask 选择**：`RectMask2D`（裁剪，轻）优先于 `Mask`（模板缓冲，会打断合批）。
8. **图集合批**：列表项打成同一图集，减少 DrawCall。
9. **Profiler 定位**：看 `Canvas.SendWillRenderCanvases` 与 `Canvas.BuildBatch` 的耗时，定位是 Rebuild 还是合批开销。

> 答题亮点：强调「Rebuild vs Repaint 的区分 + Canvas 拆分 + 对象池」三件套，并用 Profiler 证实。
