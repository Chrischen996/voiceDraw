# voiceDraw

把一段"边说边画"的白板视频,交给一个 JSON 文件来描述。

你写一个脚本:说什么(`say`),在说到哪个词时画什么(`at` + `add`/`update`/...);voiceDraw 负责把旁白合成语音、按词触发 Excalidraw 画布上的每一笔,最后输出一个带配音的 MP4。全程本地渲染,不录屏。

```json
{ "voicedraw": 1, "steps": [ {
  "say": "Take a client and a server. Every page load begins with a request from one to the other.",
  "do": [
    { "at": "client", "add": { "id": "client", "type": "rectangle", "x": 140, "y": 280, "width": 220, "height": 110, "label": "Client" } },
    { "at": "server", "add": { "id": "server", "type": "rectangle", "x": 900, "y": 280, "width": 220, "height": 110, "label": "Server" } },
    { "at": "request", "add": { "id": "request", "type": "arrow", "from": "client", "to": "server", "label": "GET /" } }
  ]
} ] }
```

```sh
npx voicedraw render dns.voicedraw.json   # 得到 dns.mp4
```

## 三十秒上手

```sh
npx -y voicedraw@latest guide      # 完整的脚本格式参考(带例子)
npx voicedraw check  my.voicedraw.json    # 只查错, 不出片
npx voicedraw snapshot my.voicedraw.json  # 出一张 PNG 看布局
npx voicedraw render   my.voicedraw.json  # 合成配音并渲染 MP4
```

前几次渲染会自动下载 ~365MB 的语音模型(Kokoro,本地 CPU 运行)和一个 headless Chrome;`voicedraw setup` 可以提前把它们下好并逐项报告。环境要求:Node.js 20+、任何 Chromium 系浏览器(装了 Chrome/Edge 就用)。

## 五个示例,直接照着改

[examples/](examples/) 里有五个完整脚本,从 16 秒到 25 秒不等,覆盖了最常用的画法:

| 示例 | 内容 | 值得看的点 |
|---|---|---|
| [cache](examples/cache.voicedraw.json) | 缓存如何减少数据库查询 | `remove` + `camera: fit` 的组合 |
| [dns](examples/dns.voicedraw.json) | 浏览器输入 URL 之后发生了什么 | 箭头 `via` 绕行、`update` 改标题 |
| [binary-search](examples/binary-search.voicedraw.json) | 二分查找的过程动画 | 同一步里多个 `at` 依次触发 |
| [pythagoras](examples/pythagoras.voicedraw.json) | 勾股定理推导 | `math` 元素排版 LaTeX |
| [gaussian-integral](examples/gaussian-integral.voicedraw.json) | 高斯积分 | 公式一步步 `update` 替换 |

跑 `npx voicedraw check <示例路径>` 可以随时验证它们当前无错。

## 脚本格式(要点)

完整文档以 `voicedraw guide` 的输出为准([docs/format.md](docs/format.md)),这里只列骨架:

- **顶层**:`voicedraw: 1`、`steps`、可选 `title` / `canvas` / `voice` / `speed` / `captions` / `cover`。画布默认 1280×720,输出默认 1920×1080、30fps。
- **step**:`say`(旁白)+ `do`(动作列表)+ 可选 `pause`。一步讲一个意思。
- **动作**:`add` / `update` / `remove` / `camera` / `point`,每个动作用 `at` 绑定到旁白的一个词或短语(句首精确,句中误差约 0.25 秒),或直接用秒数。
- **元素**:`rectangle` `ellipse` `diamond` `text` `math` `arrow` `line` `freedraw` `image`,属性名沿用 Excalidraw。

几个容易踩的点:

- **折线要直就写 `"smooth": false`**。`line`/`freedraw` 默认把控制点连成平滑曲线(适合签名、手绘);画数据折线(价格、轨迹)时关掉它,不然 V 形跌回升会被拉成 S。
- **公式用 `math` 元素**(`"tex": "\\frac{a}{b}"`,反斜杠要写两遍),旁白里念不出来就写成文字("x 的平方"),画面里再排版。
- **视频第一帧是封面**(默认取第 1 步结束时的画面),信息流截图不会看到空画布;`"cover": 3` 可换,`"cover": false` 关掉。
- `check` 除了查引用和重叠,还会报两条最毁画面的布局问题:**笔画穿过已有标签**、**两条开放笔画端点接近却没接上**(容差按圆头半径算,真接上了不报)。

## 配音

默认行为:脚本不指定 `voice` 时,用旁白语言的在线语音(中文 `edge:zh-CN-XiaoxiaoNeural`,英文 `edge:en-US-EmmaMultilingualNeural`,需要本机装有 [`edge-tts`](https://github.com/rany2/edge-tts));拿不到时整个脚本退回内置语音,保证一条视频只有一个声音。

- 内置语音:`lin`(女/中)、`chen`(男/中)、`maple`(英)、`sol`(英),`voicedraw voices` 可列出,也可以用模型自带名(如 `zf_001`)。中文音色就是模型自带的普通话说话人,不再混入英文音色:混入会让英文词更准一点,但会把元音往英语腔上拉、把声调磨平,中文反而不自然。
- 不想发任何数据到网上:显式指定内置语音,如 `--voice lin`。
- 旁白是中文却指定了英文语音(或反过来),`check` 会警告:那种组合念出来的另一门语言带口音、且很多词会念错。去掉 `voice` 让它按旁白语言自动选,或换成对应语言的语音。
- 接自己的 TTS:`--tts-command "python my_tts.py"`,程序从 stdin 读一句、把 WAV 写到 `$VOICEDRAW_OUT`。
- 已知限制:中文仅普通话;英文混在中文句子里,内置语音识别率偏低(在线语音好得多);句内单词时间是估算(±0.25s),句首是精确的。

## 让 coding agent 直接出片

这个仓库同时是一个 agent skill,所以最常见的用法不是自己敲命令,而是对 agent 说一句:

```text
Install the voicedraw skill with `npx skills add Chrischen996/voiceDraw -g -y`,
then make me a video explaining how DNS works.
```

skill([skills/voicedraw/SKILL.md](skills/voicedraw/SKILL.md))只写流程(读 guide → 写脚本 → check → snapshot → render),格式细节始终从已安装的 CLI 里取,CLI 升级格式时 skill 自动跟手。Claude Code 用户也可以用 plugin 方式安装:

```
/plugin marketplace add Chrischen996/voiceDraw
/plugin install voicedraw
```

## 开发者

```sh
npm install && npm run build && npm test   # 55 个单元测试, 覆盖 check/配音流水线/H.264 分段/token 计数
```

- 架构:画布是"时间的纯函数"——每个元素一条动画轨道,任意时刻可单独出帧;配音按步落定,画帧与编码并行进行,总耗时约等于视频时长。
- 编码优先用浏览器 WebCodecs 的 H.264(出片前会自检:试编、回解、逐像素比对),不通过时退回"PNG 帧 + ffmpeg x264"慢路径;`VOICEDRAW_ENCODER=ffmpeg` 可强制慢路径。
- CI(`.github/workflows/`):`test.yml` 每次 push 跑全量测试;`publish.yml` 在推 `v*` tag 时自动发 npm(需要仓库 Secret `NPMPUBLISH`)。发版流程见 [CONTRIBUTING.md](CONTRIBUTING.md)。
- 产物:CLI 是 esbuild 打的单文件 `dist/cli.js`(Node 端),播放页 `dist/player/` 打进 Excalidraw、MathJax、Excalifont 字体。

## 许可

MIT。依赖:Excalidraw(MIT)、MathJax(Apache-2.0)、sherpa-onnx(Apache-2.0)、Kokoro v1.1(Apache-2.0)、ffmpeg-static(GPL,独立进程运行)。
