# 机房数字孪生 · Datacenter Digital Twin

用 three.js 构建的**机房数字孪生可视化**：全息线框 → 实体渲染 → 现代机房环境，三个独立页面记录了完整的效果演进。服务器为程序化建模的细节级 2U 机架式机型（硬盘仓、电源环钮、USB、指旋螺丝、型号/IP 显示屏），点击任意一台会弹出毛玻璃质感的科幻信息框，数据可通过配置文件接入真实数据源（数据库 / API）。

纯静态项目：**无构建工具、无 npm 依赖、可完全离线运行**，three.js 已随仓库本地化（`vendor/three/`），每个 HTML 都是自包含页面。

## 页面

| 文件 | 内容 |
|------|------|
| `ai-cluster.html` | **主页面**。现代机房环境（墙体、天花灯带、防静电地板、精密空调、走线桥架）+ 3 个异构机柜（昇腾 910B ×7 / 海光 K100-AI ×3 / NVIDIA T4 ×8）+ 点击弹窗 + 数据驱动接口 |
| `server-detail.html` | 单机特写。一台细节拉满的 2U 服务器，可"打开机盖"查看内部（风扇墙、CPU 散热器、DIMM、PCB），风扇实时旋转 |
| `index.html` | 早期全景。5 机柜 ×5 服务器的太空悬浮场景，自动环绕镜头 |

## 快速开始

```bash
cd datacenter-twin-3d
python3 -m http.server 8823
# 打开 http://localhost:8823/ai-cluster.html
```

需要通过 HTTP 访问（而非双击 file:// 打开）：浏览器禁止 file:// 页面用 `fetch` 读取本地模块和 config/data 文件。**无需联网**，three.js 已内置在 `vendor/` 目录。

## 数据接入

`ai-cluster.html` 是数据驱动的，场景完全由数据生成——机柜数量、每柜台数、U 位排布都来自数据行。

**`data.json`** 就是"表"，一行一台服务器，列名自定义：

```json
{ "hostname": "ascend-01", "rack": "R1", "u": 33, "vendor": "ascend",
  "ip": "10.20.1.11", "deployed_model": "DeepSeek-R1-70B",
  "util": 72, "temp": 58, "status": "ok" }
```

**`config.json`** 声明数据源和字段映射，模板用 `{列名}` 占位符：

```json
{
  "dataSource": { "type": "json", "url": "./data.json", "refreshInterval": 10 },
  "keyField": "hostname",
  "screen": { "line1": "{vendor_tag}", "line2": "{ip}" },
  "popup": { "title": "{hostname}", "rows": [
    { "label": "IP", "value": "{ip}" },
    { "label": "机柜位置", "value": "{rack} · U{u}" },
    { "label": "部署模型", "value": "{deployed_model}" }
  ]}
}
```

- `screen.line1/line2` → 每台服务器机身显示屏的两行文字
- `popup.rows` → 点击弹窗的每一行（label 任意，value 可拼接多个字段）
- `hud` → 底部状态栏字段
- `status` 列驱动状态灯：`ok` 绿慢闪 / `warn` 黄急闪 / `down` 红常亮
- `refreshInterval` 秒轮询数据源，机身屏幕、弹窗、HUD **不刷新页面**原地更新

**接数据库**：写一个把 SQL 结果转 JSON 数组的接口，把 `dataSource.url` 指过去即可：

```python
# FastAPI 示例
@app.get("/servers")
def servers():
    db = sqlite3.connect("cmdb.db"); db.row_factory = sqlite3.Row
    return [dict(r) for r in db.execute(
        "SELECT hostname, rack, u, vendor, ip, deployed_model, util, temp, status FROM servers")]
```

列名与模板 `{列名}` 对应，不一致用 SQL `AS` 改名。注意后端要开 CORS。

**降级策略**：config.json 缺失 → 用内置默认配置；数据源不可用/为空 → 自动切换内置随机演示数据（右上角标注"演示数据"），页面永不白屏。

## 关键技术点

### 1. 实体感的来源：PBR + 环境反射

金属材质（`MeshStandardMaterial` 高 metalness）在没有环境贴图时几乎是黑的——金属的观感本质是"反射了什么"。解法是用 three.js 内置的 `RoomEnvironment` 生成一张环境贴图：

```js
const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
```

配合 `envMapIntensity` 微调反射强度，机箱表面立刻出现金属光泽。这是"像实物"和"像塑料玩具"的分水岭。

布光用三灯法则：主光（DirectionalLight + 阴影）、背面补光（防止相机转到背面时一片死黑）、彩色点光源做氛围轮廓。

### 2. 科幻辉光：Bloom 后处理

所有"发光"元素（LED、光条、光环、灯带）并不真的发光，而是用高亮度 `MeshBasicMaterial`（不受光照影响，颜色始终饱和），再由 `UnrealBloomPass` 对整帧画面中超过亮度阈值的像素做泛光：

```js
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(resolution, 0.75 /*强度*/, 0.45 /*半径*/, 0.55 /*阈值*/));
composer.addPass(new OutputPass());  // 负责色调映射与色彩空间
```

阈值是关键旋钮：调低整个画面糊成一团光，调高辉光消失。0.5~0.6 配合 ACES 色调映射效果最稳。

### 3. 程序化建模：不用任何 3D 模型文件

所有几何体都由参数化的 `BoxGeometry / CylinderGeometry / TorusGeometry` 组合而成。服务器是一个工厂函数，按机箱高度自适应布局——高机箱 3 排硬盘仓、中等 2 排、薄款 1 排，同一份代码生成三种机型：

```js
if(h > 0.9){ rows = [0.10, -0.14, -0.38]; }      // 4U：3 排硬盘仓
else if(h > 0.4){ rows = [0.09, -0.10]; }         // 2U：2 排
else { rows = [0.01]; }                            // 1U：1 排
```

好处：零资源加载、可以被数据驱动（多一行数据多一台服务器）、每个细节（LED 位置、把手宽度）都是可调参数。

### 4. Canvas 动态贴图：屏幕、冲孔板、地板

机身的"型号 + IP 显示屏"是一张 256×96 的 `<canvas>` 画出来的 `CanvasTexture`。数据刷新时重画 canvas 并置 `texture.needsUpdate = true`，3D 世界里的屏幕就更新了——这是 2D 数据进入 3D 场景最廉价的通道：

```js
ctx.fillText(tpl(CFG.screen.line2, row), 10, 74);  // 模板渲染 IP
texture.needsUpdate = true;
```

同样的技术做了：硬盘仓冲孔网（圆点阵）、PSU 同心圆格栅、PCB 走线、防静电地板方格（含通风穿孔砖）。两个细节参数：`colorSpace = SRGBColorSpace`（否则颜色发灰）、`anisotropy = 8`（斜视角下文字不糊）。

### 5. HTML 与 3D 融合：CSS2DRenderer

机柜顶的标签牌和点击弹窗是真正的 HTML 元素，由 `CSS2DRenderer` 把 DOM 节点投影到 3D 坐标上，每帧跟随镜头。这比在 3D 里渲染文字清晰得多，且可以用全部 CSS 能力：

- 弹窗毛玻璃 = `backdrop-filter: blur(8px)` + 半透明深蓝底 + 1px 青色描边 + `::before/::after` 画四角括号
- 入场动画 = CSS transition（scale + opacity），注意用 `setTimeout` 而非 `requestAnimationFrame` 触发 class，避免后台标签页节流时动画卡住
- **视口钳制**：每帧检查弹窗 `getBoundingClientRect()`，超出屏幕边缘时用 margin 拉回来，保证永远完整可见
- 弹窗与服务器之间的引线是一条 3D `THREE.Line`，两端点由服务器世界坐标计算

### 6. 交互拾取：Raycaster + 点击/拖拽区分

点击检测用 `Raycaster.intersectObjects(pickables)`，只对机箱主体做拾取（不含小零件，省性能）。关键细节是区分"点击"和"旋转拖拽"：记录 pointerdown 位置，pointerup 时位移超过 5px 判为拖拽、不触发选中。选中态用 `EdgesGeometry` 描边（默认透明，选中时 opacity 拉起）。

### 7. 机房环境：一个反转的盒子

房间就是一个大 `BoxGeometry` 配 `side: THREE.BackSide`（只渲染内表面），three.js 会自动翻转法线让内壁正确受光。天花灯带 = 发光长条（吃 bloom）+ 同位置的真实 `PointLight`（负责照亮机柜顶面），视觉和光照分离处理。相机 `maxDistance` 限制在房间内，避免穿墙。

### 8. 氛围层：让画面"活着"

- **尘埃粒子**：300 个 `THREE.Points` 加法混合，缓慢上浮循环
- **扫描光片**：一块加法混合的半透明薄板上下往复，扫过机柜时像全息扫描切片
- **地面光环**：`RingGeometry` 四段弧，两圈反向旋转
- **LED 群闪**：每颗灯随机相位/频率的 `sin` 开关，避免整齐划一的假感
- **CRT 覆盖层**：纯 CSS 的 `repeating-linear-gradient` 扫描线 + 径向渐变暗角，罩在整个画面上，一行代码的"科幻滤镜"

### 9. 性能预算

单页约 1000+ mesh，流畅的前提：`renderer.setPixelRatio(Math.min(devicePixelRatio, 2))` 防止 Retina 屏 4 倍像素开销；材质全局共享（同款金属只创建一次）；阴影只给一盏主光、2048 贴图；粒子用 BufferGeometry 原地改写。

## 文件结构

```
datacenter-twin-3d/
├── ai-cluster.html      # 主页面（数据驱动，内置降级演示数据）
├── config.json          # 可选：数据源 + 字段映射
├── data.json            # 可选：服务器清单示例（18 台）
├── server-detail.html   # 单机 2U 特写（可开盖）
├── index.html           # 5 机柜太空全景（早期版本）
├── vendor/three/        # three.js 0.160.0 本地化（离线部署依赖，勿删）
└── README.md
```

---

# 交接说明（Handover）

> 本节面向接手维护的同事，读完应该能独立定位代码、完成常见修改并部署上线。

## 1. 项目定位与现状

- **是什么**：机房数字孪生的前端可视化 Demo，核心页面 `ai-cluster.html` 已具备接入真实数据的能力（轮询 JSON/API），另外两个页面是效果演进过程中的独立成品，无数据接口。
- **技术栈**：原生 HTML/CSS/JS + three.js 0.160（ES Module，CDN 引入），无框架、无构建、无 node_modules。改完刷新浏览器即生效。
- **当前状态**：功能完整可用；数据为示例数据（`data.json`），尚未对接真实 CMDB/监控系统。

## 2. 各文件职责

| 文件 | 行数 | 职责 | 是否依赖其他文件 |
|------|------|------|------|
| `ai-cluster.html` | ~720 | 主页面，全部逻辑在一个 `<script type="module">` 里 | 可选读取 config.json、data.json，读不到自动降级 |
| `config.json` | ~40 | 数据源地址、轮询间隔、字段映射模板、品牌样式、状态色 | 被 ai-cluster.html 读取 |
| `data.json` | ~22 | 服务器清单示例，模拟数据库表 | 被 config.json 的 dataSource.url 指向 |
| `server-detail.html` | ~600 | 独立页面：单台 2U 服务器解剖展示，含开盖动画、旋转风扇 | 无 |
| `index.html` | ~390 | 独立页面：5 机柜全景，自动环绕镜头 | 无 |

三个 HTML **互不引用**，删除任何一个不影响其余。

## 3. ai-cluster.html 代码结构导览

代码按注释分块，从上到下依次是（搜索注释即可跳转）：

1. **`<style>`**：HUD 样式。`.popup` 系列是点击弹窗（毛玻璃 + 四角括号），`#stats` 是底部状态栏，`#overlay` 是全屏扫描线滤镜。
2. **`数据层`**：`loadJSON`、`DEFAULT_CFG`（内置默认配置，与 config.json 结构完全一致）、`genDemoRows()`（内置演示数据生成器）、config/data 加载与降级逻辑。**改字段映射的默认值在 `DEFAULT_CFG` 里**。
3. **`场景基础`**：renderer、相机、OrbitControls（`maxDistance 8.6` 限制在房间内）、灯光（key 主光带阴影 / back 背面补光 / fill 氛围点光）。
4. **`材质与工具`**：共享材质（`matBody` 机身 / `matDark` 深色 / `matFin` 银色金属件…）、`box/glow/cylZ` 三个建模辅助函数、`canvasTex`（canvas→贴图，注意 `anisotropy=8`）、`perfMat`（冲孔板材质）。
5. **`机房环境`**：地板贴图、`room`（BackSide 反转盒子）、天花灯带、墙脚灯、CRAC 空调 ×2、走线桥架。
6. **`机柜与服务器`**：`makeScreen(row)` 生成机身 IP 屏（返回 `{mat, draw}`，`draw()` 可重复调用刷新文字）；`caddy()` 单个硬盘仓；`makeServer(row, h)` 服务器工厂函数（**按高度 h 自动决定 1/2/3 排硬盘仓**）；机柜循环（由 `racksData` 驱动，机柜 X 坐标自动均布）。
7. **`弹窗 + HUD`**：`renderPopup/showPopup/hidePopup`、引线 `leader`、底部状态栏由 `CFG.hud` 动态生成。
8. **`数据刷新`**：`setInterval` 轮询，按 `keyField` 匹配后 `Object.assign` 合并数据行，再触发 `screen.draw()` / 弹窗 / HUD 重绘。
9. **`交互`**：Raycaster 拾取（只检测 `pickables` 数组里的机箱主体）、点击 vs 拖拽的 5px 判定、hover 光标。
10. **`animate()`**：渲染循环——光环旋转、LED 群闪、扫描光片、粒子上浮、弹窗视口钳制（margin 拉回）都在这里逐帧执行。

## 4. 核心数据流

```
config.json（可选，缺省用 DEFAULT_CFG）
   │  dataSource.url / 字段映射模板
   ▼
data.json 或 API（缺省用 genDemoRows 随机数据）
   │  ROWS: 一行 = 一台服务器
   ▼
racksData（按 rack 列分组、按 u 列降序排列）
   │
   ▼
构建 3D 场景：每行 → makeServer(row) → 机身屏幕 makeScreen(row)
   │
   ▼  每 refreshInterval 秒
轮询数据源 → 按 keyField 匹配 → Object.assign 更新 row
   → screen.draw() 重绘机身屏幕
   → renderPopup / renderHud 更新弹窗和状态栏
```

关键约定：
- **`keyField`（默认 hostname）是唯一主键**，轮询靠它匹配服务器，值必须稳定；
- **拓扑（机柜数/台数/U位）只在页面加载时构建**，轮询只更新显示内容。增删机器需刷新页面；
- 模板占位符 `{列名}` 直接取数据行字段，另有两个派生字段：`{vendor_tag}`、`{vendor_name}`（来自 config.vendors）。

## 5. 常见修改任务速查

| 任务 | 做法 |
|------|------|
| 加一台服务器 | data.json 加一行（rack 写已有机柜名），刷新页面 |
| 加一个机柜 | data.json 中使用新的 rack 名即可，机柜会自动出现并均布 |
| 弹窗加一行（如"负责人"） | 数据行加列 `owner`，config.json 的 `popup.rows` 加 `{"label":"负责人","value":"{owner}"}` |
| 换机身屏幕显示内容 | 改 `config.screen.line1/line2` 模板 |
| 新增硬件品牌 | config.json 的 `vendors` 加一项（tag/name/accent），数据行 vendor 填新 key |
| 改状态灯颜色/新增状态 | config.json 的 `statusColors`；闪烁频率在 `makeServer` 的 `stSpeed` 处 |
| 接真实数据库 | 后端出一个返回 JSON 数组的接口（见上文 FastAPI 示例），改 `dataSource.url`，注意 CORS |
| 调初始镜头 | `camera.position.set(...)` 与 `controls.target.set(...)` |
| 房间尺寸 | `room` 的 BoxGeometry 和地板 PlaneGeometry，同时记得调 `controls.maxDistance` |
| 关掉演示数据降级 | 把 `catch` 里 `usingDemo=true` 分支改回抛错即可（不建议） |

## 6. 部署

任何静态托管都行（nginx / 宝塔 / OSS / GitHub Pages），把仓库文件原样放上去即可。**支持完全离线/内网部署**——three.js 已本地化在 `vendor/three/` 目录（主库 + 13 个插件模块，共 2.1 MB），页面通过相对路径 importmap 加载，运行时不发起任何外网请求。注意：

1. `vendor/` 目录必须与 HTML 一起部署，保持相对路径结构不变；
2. three.js 版本**锁死 0.160.0**（vendor 内文件即此版本），不要随意升级 —— examples/jsm 的模块路径和 API（如 OutputPass）跨大版本常有破坏性变更。如需升级，重新下载对应版本的同名 14 个文件替换 vendor 目录；
3. 若数据接口与页面不同源，后端必须返回 `Access-Control-Allow-Origin` 响应头。

## 7. 已知限制

- 拓扑变更（增删机器/机柜）不支持热更新，需刷新页面（轮询只改显示值）；
- 机身屏幕 canvas 为 256×96，IP/型号超过约 15 个字符会溢出裁切；
- 机柜均布逻辑按 3.3 间距硬编码，超过 ~6 个机柜会超出房间宽度，需同步调房间尺寸；
- `file://` 直接打开时 fetch 被浏览器禁止，会自动进入演示数据模式（属预期行为，不是 bug）；
- 移动端未做专门适配（可旋转缩放，但 HUD 布局按桌面设计）。

## 8. 本地开发

```bash
python3 -m http.server 8823        # 任何静态服务器都行
# http://localhost:8823/ai-cluster.html
```

改代码 → 刷新浏览器，没有编译步骤。调试建议开着 DevTools Console：数据源加载失败、轮询失败都会打 warning 并注明原因。

## License

MIT
