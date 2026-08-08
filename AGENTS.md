# AGENTS.md

本文件供 AI 编码代理阅读,目的是让不了解本项目的读者快速掌握项目全貌。阅读者被假定对本项目一无所知。

## 项目概述

这是一个关于傅里叶变换的可视化教学项目,基于 3Blue1Brown 的科普视频《But what is the Fourier Transform?》(https://www.youtube.com/watch?v=spUNpyF58BY)。

核心思路(视频中的"质心法"):把输入信号按不同的采样频率 f 缠绕到圆上(极坐标形式 amp·e^(i·2πft)),然后跟踪缠绕图形的质心 x 坐标。当 f 恰好等于信号中的某个频率时,质心会明显偏向一侧,由此可以识别信号中包含的频率成分。

项目包含三个交付物:

| 文件 | 说明 |
|------|------|
| `Fourier Transform - A Visual Introduction.ipynb` | 主 Jupyter Notebook,用 Python(numpy / matplotlib / seaborn / ipywidgets)完整实现质心法,并带 ipywidgets 逐帧动画 |
| `fourier_transform.html` | 独立可运行的静态网页版(原生 HTML + JavaScript + mathjs),无需任何 Python 环境,浏览器直接打开即可 |
| `README.md` | 简要说明与依赖安装指引 |

默认输入信号是指数衰减信号 f(t) = e^(-2t)(t ∈ [0, 2]),频率扫描范围 0–10 Hz、步长 0.1 Hz。注意文件名带空格。

## 技术栈

- **Notebook 版**:Python 3(nbformat 4 / nbformat_minor 2,notebook 由 Python 3.11.8 环境生成)
  - numpy(数值计算)
  - matplotlib(`%matplotlib widget` 交互后端)
  - seaborn(美化绘图,`sns.set()`)
  - ipywidgets / ipympl(动画控件:Play、IntSlider、HBox/VBox、jslink、interactive_output)
  - IPython.display(嵌入 YouTube 视频)
- **网页版**:纯静态 HTML/CSS/JavaScript,零构建步骤、无包管理器
  - mathjs 12.4.1(CDN 加载:cdnjs,失败时回退到 unpkg)
  - KaTeX 0.16.9(CDN 加载:cdnjs,失败时回退到 unpkg),用于第 5 节的 LaTeX 公式渲染
  - HTML5 Canvas 绘图
  - 单文件、深色主题、页面语言 zh-CN

## 目录结构与模块划分

```
Fourier_Transform/
├── Fourier Transform - A Visual Introduction.ipynb   # 主 Notebook
├── fourier_transform.html                            # 网页版交互实现
└── README.md
```

Notebook 共 4 个章节:

1. **生成信号** — 定义指数衰减信号 `exp_signal = np.exp(-alpha * t)`,`alpha = 2.0`,t 从 0 到 2、步长 0.001。
2. **缠绕信号** — 对每个采样频率 sf,构造极坐标点 (幅值, 角度):
   - `r_cord[l] = [(exp_signal[i], t[i]*sf*2*np.pi) for i in range(len(t))]`
   - 再转直角坐标 `x_cord`(amp·cosθ)、`y_cord`(amp·sinθ)
   - 绘制全部子图(约 100 个、每行 4 个),红色圆点标记质心
   - `mean_list` 保存每个频率下 x 坐标之和(供第 4 节画质心曲线)
3. **动态交互动画** — 用 ipywidgets 的 Play / IntSlider / jslink 实现"播放",逐帧绘制缠绕曲线,红色点实时显示质心。
4. **质心 vs 采样频率** — 折线图、平滑图、柱状图:
   - 平滑阈值取最大值的一定比例:`smoothed = [i if i>0 and i>0.2*max(mean_list) else 0 for i in mean_list]`

网页版 `fourier_transform.html` 的 JavaScript 结构(无模块化,全部是全局函数,自上而下依次为):

- 全局常量/状态:`NUM_POINTS = 2000`、`t`、`signal`、`sfList`、`xCord`、`yCord`、`meanList`,以及各 canvas 的 2d context。
- `generate()` — 读取输入参数,用 mathjs `math.compile(funcStr)` 编译用户函数,生成时间序列与信号,计算 xCord/yCord/meanList,最后统一调用各绘图函数;计算放在 `setTimeout(..., 30)` 里以便先渲染 loading 提示。
- `drawSignal()` — 第 1 节:信号曲线。
- `drawGrid()` — 第 2 节:10 列子图网格,每格画缠绕曲线与质心红点。
- `computeGlobalRange()` / `drawWindingCell(ctx, k, numPoints, rect, opts)` — 第 2、3 节共用的缠绕绘图:先对所有频率计算统一的全局范围 `gRange`,再按该范围绘制单个缠绕图(坐标轴穿过数据原点 (0,0));`drawGrid` 逐格调用、`drawAnim` 每帧调用(传 `frame+1` 只画到当前帧)。
- `drawAnim(freqIdx, frame)` / `togglePlay()` / `animStep()` — 第 3 节:逐帧播放动画(每 40ms 前进 10 帧)。
- `drawCOMPlots()` / `drawLinePlot()` / `drawBarPlot()` — 第 4 节:三个并排图(原始质心、平滑后、柱状)。
- 第 5 节为静态 HTML 讲解(英文,标题 "Understanding the Geometric Meaning of Fourier Transform & Spectral Density"),数学公式用 KaTeX auto-render 渲染:页面末尾独立的 `<script>` 在 DOMContentLoaded 时对 `#section5` 调用 `renderMathInElement(...)`,支持内联 `$...$` 与独立 `$$...$$`。
- `onMathReady()` / `checkMath()` — 轮询等待 mathjs 就绪后自动 `generate()`;8 秒超时在页面上提示 math.js 加载失败。
- 事件监听:频率/帧滑块 `input` 事件、输入框回车触发 `generate()`。

## 构建与运行

本项目**没有构建步骤,也没有任何配置文件**(无 pyproject.toml / package.json / requirements.txt)。

- **Notebook**:安装依赖后启动 Jupyter:
  ```bash
  pip3 install --user numpy matplotlib seaborn ipywidgets ipympl
  jupyter notebook "Fourier Transform - A Visual Introduction.ipynb"
  ```
  注意:第 3 节动画单元格依赖 `%matplotlib widget`(ipympl),首次使用需安装 `pip install ipywidgets ipympl`(notebook 第 3 节说明中已注明)。notebook 内嵌大量 base64 图像输出,文件很大,直接手工编辑 JSON 极易出错,应尽量用 Jupyter 界面编辑。
- **网页版**:用任意浏览器直接打开 `fourier_transform.html` 即可(需联网加载 mathjs CDN;加载失败时页面会在约 8 秒后给出网络错误提示)。

## 代码风格约定

- 注释语言混用:2026 年新增/修改的代码与说明使用中文(例如 `# 指数衰减信号`、`# 衰减系数`,第 2 节后的 markdown 详解、第 3 节标题均为中文),2019 年原始内容为英文。新增代码请沿用所在单元格/段落附近已有的语言习惯。
- 网页版页面语言为 zh-CN,界面文案用中文;代码标识符统一用英文 camelCase(如 `drawAnim`、`meanList`、`funcInput`、`sfList`)。
- Notebook 变量命名习惯:`r_cord`(极坐标点列表)、`x_cord` / `y_cord`(直角坐标)、`mean_list`(质心 x 之和)、`sf_list`(采样频率)。
- Notebook 绘图参数习惯:`plt.rcParams["figure.figsize"]` 常用 (12,4);第 2 节子图大图用 (15,110)。

## 测试

- 项目没有单元测试、没有 CI、没有自动化测试脚本。
- 验证以人工为主:
  - Notebook:按顺序执行各单元格,确认四节图形与第 3 节动画正常;
  - 网页版:浏览器打开,确认四节渲染与动画/滑块交互正常。
- notebook 每次重新运行后,会产生 `execution_count`、widget `model_id`、内嵌 base64 图像等大量 diff(当前分支的未提交改动正是这类产物),这是正常现象,不代表代码出错。

## 版本控制

- 分支:`master`(上游)与 `feature/use_exponential_function_as_input`(当前分支,HEAD,工作区有未提交改动)。
- 历史:2019 年原仓库由 thatSaneKid 创建;2026 年 7 月由 hushpro 扩展,新增动画与网页版。未提交改动只是 notebook 重跑产物(execution_count / widget model_id / 内嵌图像),提交时通常一并带上。

## 安全注意事项

- 网页版通过 `math.compile` 在浏览器端执行用户输入的函数表达式,这是功能设计(等价于让访问者运行自定义数学函数),仅作用于其本人浏览器会话、不涉及任何服务器;但若要把该 HTML 部署到共享环境,应知晓它会在访问者浏览器中执行任意数学表达式。
- 页面依赖外部 CDN(mathjs),存在网络依赖;离线环境不降级,仅提示加载失败。
- Notebook 可执行任意 Python 代码,与普通 Jupyter 行为一致——不要打开或运行不可信来源的 notebook。
