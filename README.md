# 视觉传输（Cimbar PWA）

电脑屏幕 → 手机摄像头，纯视觉通道的文件 / 文本 / 图片传输。无需网络、蓝牙、NFC，所有数据都通过"屏幕显示 + 摄像头拍摄"完成。

- **发送端**：电脑浏览器打开 `sender.html`，选择文件 / 文本 / 图片，屏幕播放彩色编码动画。
- **接收端**：手机浏览器打开 `receiver.html`，摄像头对准屏幕自动解码；文件可保存、文本可复制、图片可预览保存；大文件按段接收，中断后同浏览器内可续传。
- **PWA**：手机端可"添加到主屏幕"当作 App 使用，支持离线缓存。
- **语言**：入口 / 发送 / 接收页面均支持 中文 / English 切换，默认英文，选择会记住在浏览器本地。

## 目录结构

```
cimbar-pwa/
├── index.html            # 入口页（发送/接收二选一）
├── sender.html           # 发送端（文件/文本/图片 + 分段可调 + 颜色模式 + 帧率/全屏）
├── receiver.html         # 接收端（扫码解码 + 分段续传 + 历史记录 + 静态图测试）
├── sender-legacy.html    # 旧版单文件发送端（内嵌 WASM，可双击本地打开；与本接收端同源格式）
├── manifest.webmanifest  # PWA 清单
├── sw.js                 # Service Worker（预缓存 + 运行时缓存）
├── index.js            # cimbar 运行库（qt-works/file-transfer-codec）
├── index.wasm          # cimbar 解码 WASM（约 3.8MB）
├── worker.js           # 解码 Worker
├── icon-180.png / icon-192.png / icon-512.png  # PWA 图标
├── LICENSE               # Apache-2.0（qt-works/file-transfer-codec）
├── NOTICE
├── THIRD_PARTY_NOTICES.md
└── README.md
```

## 快速开始 A：部署到 GitHub Pages（推荐，手机端可用）

1. 在 GitHub 新建一个**公开仓库**（免费版 GitHub Pages 只支持公开仓库；隐私要求高可用 Cloudflare Pages 等替代）。
2. 把本目录**所有文件**上传到仓库根目录（扁平结构，无需子目录）。
3. 仓库 `Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: main / (root)` → Save。
4. 等 1–2 分钟，页面会显示站点地址：`https://<你的用户名>.github.io/<仓库名>/`。
5. 电脑打开 `<站点>/sender.html`；手机打开 `<站点>/receiver.html`（首次会请求摄像头权限，允许即可）。
6. 手机 Safari/Chrome 里"添加到主屏幕"，可全屏离线使用。

> 注意：必须用站点域名（HTTPS）打开，`file://` 或局域网 `http://192.168.x.x` 会被浏览器禁止摄像头。

## 快速开始 B：本机 localhost 测试（电脑端）

```bash
# 在本目录下
python -m http.server 8000
```

然后电脑浏览器打开：

- 发送端：`http://localhost:8000/sender.html`
- 接收端：`http://localhost:8000/receiver.html`

`localhost` 属于安全上下文，摄像头可用。手机端无法访问你电脑的 localhost，手机测试请用方案 A（GitHub Pages）。

## 使用说明

### 发送端（sender.html）

| 模式 | 操作 |
|---|---|
| 文件 | 拖放或点击选择文件；大文件会自动分段（段大小可调，默认 512KB）；超过 30MB 会弹窗提醒耗时 |
| 文本 | 粘贴文字 → 点击"开始传输文本" |
| 图片 | 选择 / 拖入 / 粘贴（Ctrl+V）图片 → 自动压缩（最长边 1600px、JPEG 0.85）→ 开始传输 |

- **颜色模式**：默认 `4C · 彩色`（密度最高、速度最快）；如手机黑白屏或环境光干扰明显，可切 `B · 黑白`（更稳）。
- **段大小**：256KB / 512KB / 1MB / 2MB 可选（默认 512KB）。段越小单段播放越快、中断损失越小，但段头开销比例略高。
- **帧率**：15/20/30 fps 可调。手机解码速度快时可调高；画面不稳请调回 15。
- **全屏**：放大编码画面，手机更容易对准。
- 传输中请保持屏幕全亮度、画面稳定，手机尽量正对屏幕、避免反光。

### 接收端（receiver.html）

1. 点击"开始扫描"，摄像头对准电脑屏幕上的编码动画，等待进度走满。
2. 完成后自动识别类型：
   - **文件** → 直接保存；
   - **文本** → 显示原文 + 一键复制；
   - **图片** → 预览 + 保存。
3. 点"继续扫描"可接收下一个文件。
4. **分段续传**：大文件按段接收，每段立即存入浏览器 IndexedDB；顶部"接收中的传输"显示各文件段进度。中断（关页/刷新）后回到本页继续扫描，已收到的段不会重复丢失，补齐剩余段即可完成拼装。
5. **已接收记录**保存在浏览器 IndexedDB 中，可随时重新下载 / 删除（同一浏览器内有效）。
6. **解码图片（测试）**：可导入一张保存下来的编码画面截图进行离线解码，便于排查对齐问题。

## 技术说明与限制（重要）

- **编码格式**：基于 libcimbar 的彩色图标矩阵视觉编码（屏幕→摄像头）。发送端内嵌官方 libcimbar 引擎（单文件、离线可用）；接收端运行库为 `@qt-works/file-transfer-codec`（Apache-2.0，底层 libcimbar MIT）。
- **数据封装（CTXF v2）**：数据流前附加 `CTXF` 头部（版本 + 类型 + 标志 + 文件名 + 负载长度）。大文件按段发送，每段带段序号 / 总段数 / 原始大小，接收端据此**跳段去重 + 断点续传**；文本 / 图片走单段。旧版无头部的原始数据流（如官方 cimbar.org、`sender-legacy.html` 发出的）也能按文件接收，但无法识别类型。
- **单向视觉通道的物理限制**：
  - 现实吞吐量级约 10–60 KB/s（取决于颜色模式与手机解码速度），**大文件建议 ≤ 30MB**，越大越耗时（超过 30MB 发送端会弹窗提醒）；
  - 分段续传只在**同一浏览器 + 同一站点**内有效（数据存在该浏览器的 IndexedDB 中），换设备 / 换浏览器无法接续。
- **兼容性**：`sender-legacy.html`（旧版官方发送端）与本接收端的兼容性需要实测确认；两端都基于 libcimbar，格式同源，预期可用。
- **摄像头权限**：接收端必须 HTTPS 或 localhost；iOS Safari 需 `playinline`（已内置）。首次授权后若误拒绝，请在浏览器设置中重新允许。

## 下一步（可选）

1. **快通道**：屏幕显示"握手二维码"（会话 ID + 局域网地址），手机扫码后大文件改走局域网/WebRTC（MB/s 级、真断点续传），视觉通道作为离线兜底。
2. **双向视觉续传**：PC 有摄像头时，手机屏幕显示"缺失帧位图"，PC 读回后只补缺失部分。
3. **压缩传输**：文件启用 Deflate 压缩（浏览器 CompressionStream）进一步缩短扫描时间。

## 许可

- 本项目页面代码与封装逻辑：Apache-2.0（见 `LICENSE`，来自 `@qt-works/file-transfer-codec`）。
- 底层视觉编码 `libcimbar` 及 OpenCV 等第三方组件：见 `THIRD_PARTY_NOTICES.md` 与 `NOTICE`。
- `sender-legacy.html` 来自官方 libcimbar 发布包（MIT）。
