# 机房数字孪生 · Datacenter Digital Twin

用 three.js 构建的**机房数字孪生可视化**：全息线框 → 实体渲染 → 现代机房环境，三个独立页面记录了完整的效果演进。服务器为程序化建模的细节级 2U 机架式机型（硬盘仓、电源环钮、USB、指旋螺丝、型号/IP 显示屏），点击任意一台会弹出毛玻璃质感的科幻信息框，数据可通过配置文件接入真实数据源（数据库 / API）。

纯静态项目：**无构建工具、无 npm 依赖**，three.js 从 CDN 加载，每个 HTML 都是自包含的单文件页面。

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

需要通过 HTTP 访问（而非双击 file:// 打开），原因有二：`fetch` 读取本地 config/data 文件受浏览器安全限制；不过即使 file:// 打开，页面也会自动降级为内置演示数据，不会白屏。需联网加载 jsdelivr CDN 上的 three.js。

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
└── README.md
```

## License

MIT
