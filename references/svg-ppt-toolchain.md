# SVG → PPT 工具链骨架

本文件是阶段二的**可复用逻辑与代码骨架**，配套 `references/academic-ppt.md` 使用。

- 只给骨架与关键函数签名，不附完整脚本（避免连字体路径一起维护）。
- 依赖仅限：Python 标准库 + PIL + PyMuPDF。**不引入** cairosvg、视觉模型、Edge 截图等外部依赖。

---

## 1. 坐标系换算

```python
CANVAS_W, CANVAS_H = 1280, 720          # SVG 画布（px，无单位）
DESIGN_W, DESIGN_H = 960, 540           # PowerPoint 页面（pt）
SCALE = CANVAS_W / DESIGN_W             # = 4/3

def pt2svg(pt: float) -> float:
    """设计 pt → SVG 坐标/字号"""
    return pt * SCALE

def svg2pt(v: float) -> float:
    """SVG 坐标/字号 → 落地 pt（仅供复核，不应写进 SVG）"""
    return v / SCALE
```

**自检**：`pt2svg(24) == 32`、`pt2svg(9.5) == 12.666...`、`pt2svg(60) == 80`。

---

## 2. 字体度量（em = 1000）

```python
METRICS = {
    # name: (ascent_em, descent_em)
    "Microsoft YaHei": (1.0590, 0.2620),
    "Arial":           (0.9060, 0.2120),
}
LATIN_BASELINE_SHIFT_EM = METRICS["Microsoft YaHei"][0] - METRICS["Arial"][0]  # 0.1530

def latin_dy(font_size_svg: float) -> float:
    """同一行内，拉丁文相对中文应额外下移的 SVG 距离"""
    return LATIN_BASELINE_SHIFT_EM * font_size_svg
```

**换字体必须重测**：上面两个度量是 YaHei + Arial 的实测值，不能直接套到其他字体组合。

自检示例（9.5 pt 行）：`font_size_svg = 12.6667`，`latin_dy ≈ 1.938`；若中文 y=236.00，数字 y≈237.94（按实际缩放复核）。

---

## 3. 折行与包围盒

```python
def measure_text_width(text: str, font, font_size: float) -> float:
    """用真实字体度量算整行宽度（不要用 len(text) * 常数估算）"""

def wrap_lines(text: str, font, font_size: float, max_width: float) -> list[str]:
    """按视觉宽度折行；中英文混排时在同宽度下比较，不按字符数折"""

def text_bbox(lines: list[str], x: float, y: float,
              font_size: float, ascent_em: float, descent_em: float,
              line_height: float, anchor: str = "start") -> tuple[float, float, float, float]:
    """返回 (x0, y0, x1, y1)：按真实 ascent/descent 与行高计算，y 为首行基线"""
```

要点：
- 折行必须用**真实字体度量**，不能按字符数硬切（中英混排宽度差异极大）。
- `maxlines` 一旦截断正文，**必须记录并报警**，不能静默丢弃。
- 顺便检查中英文之间是否缺空格（可读性与排版问题）。

---

## 4. 几何校验五件套

```python
TOL = {
    "margin":   30.0,   # pt：左右边距下限
    "overlap":   1.6,   # pt：两两重叠上限
    "container": 3.0,   # pt：卡片溢出上限
    "truncate":  0,     # 处：截断必须为 0
    "over_image":0,     # 处：压图必须为 0
}

def check_margins(boxes, page_w, page_h, tol=TOL["margin"]) -> list: ...
def check_overlaps(boxes, tol=TOL["overlap"]) -> list: ...
def check_containers(boxes, cards, tol=TOL["container"]) -> list: ...
def check_truncation(truncated_log) -> list: ...
def check_over_images(boxes, image_rects) -> list: ...
```

全部返回**问题列表**（空列表 = 通过）。任一非空即未达标，改了源文件再跑。

阈值可放宽，但**必须记录放宽了哪一项、放宽到多少、为什么**。

---

## 5. 渲染后回读复核

```python
def readback_positions(svg_or_pdf_path) -> list[dict]:
    """用 PDF 引擎回读每段文字的真实坐标，与估算 bbox 对照"""
```

目的：防止「估算通过、渲染后却出界」。回读是最终裁决，估算只用于快速迭代。

---

## 6. PDF 抽取

```python
def extract_text_with_eq_flags(page) -> list[dict]:
    """get_text()；识别 inline-eq-IEqNN 占位符 → 标记「公式，值未取到」"""

def detect_rotated_table(words) -> bool:
    """若 x 分布远多于 y，疑似 90° 旋转表格 → 行列互换后交叉验算"""

def figure_band_from_caption(page, caption_y0: float,
                             prev_text_y1: float, pad: float = 8.0):
    """按 caption 的 y 与上一段正文的 y 反推图区带（矢量图 get_images() 为空也适用）"""
```

红线：
1. 公式对象取不到值 → 标「未取到」，**禁止猜**。
2. 旋转表格 → 必须用「相对提升」等衍生行交叉验算。
3. `get_images()` 为空 ≠ 没图 → 按 caption 反推。

---

## 7. 图裁切与内嵌

```python
def crop_whitespace(img) -> tuple:
    """按非白像素自动裁白边"""

def is_edge_clipped(img, edge: int = 3) -> bool:
    """裁完仍贴边 → 说明被切断，扩大范围重裁"""

def embed_base64(img, display_width_pt: float) -> str:
    """缩放到 显示宽度 × 2 像素，转 base64 data URI 内嵌"""
```

内嵌要求：`<image xlink:href="data:image/png;base64,...">`，**不依赖外部文件路径**，保证 SVG 移动到其他目录后图片仍显示。

---

## 8. 自检清单（交付前逐项过）

- [ ] 画布 `width=1280 height=720 viewBox="0 0 1280 720"`
- [ ] 全部字号/坐标 = 设计 pt × 4/3
- [ ] 每行拉丁文 y 比中文 y 多 `0.153 × 字号`
- [ ] 五项几何校验全归零，或已记录放宽容差
- [ ] 渲染后回读复核无出界/重叠
- [ ] 无逐字 `<text>`、无 text-to-path、无整页截图
- [ ] 每幅图是一个完整内嵌 `<image>`
- [ ] 截断记录为 0，或已报警说明
