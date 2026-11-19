# live-tryon

浏览器端实时虚拟试衣 —— 上传一张平铺的衣服图，摄像头把衣服实时贴到你身上。

零依赖、零构建、无后端。双击 `index.html` 就能跑。

> Real-time virtual try-on in the browser. One flat garment photo + your webcam, no backend, no build step.

---

## 这是做什么的

用 MediaPipe Pose 提取人体肩、髋关键点，构建躯干坐标系，把衣服切成三角网格后逐块做仿射纹理映射。所以衣服会跟着肩膀转动、随身高缩放、腰部略微收窄，而不是一张硬纸板跟着飘。

## 这不是什么

**这不是生成式 AI 试衣。** 它不做重新生成，只做几何形变。所以：

- 衣服的褶皱、材质、垂坠感都是原图里的，不会重新渲染
- 转身会穿帮（一张平面图没有背面信息）
- 手臂会被衣服盖住（没有人体解析）
- 光照只是整体亮度匹配，不是真的重新打光

如果你要的是"逼真"，那需要扩散模型，代价是单张 2–10 秒，接不进摄像头。两条路线的取舍见文末。

## 特性

- **摄像头实时**：约 30 fps，纯前端运行，不传任何画面到服务器
- **照片模式**：没有摄像头也能用，上传一张人像照即可（也方便截图）
- **自动去白底**：从四边洪泛填充，只删和边缘连通的白色，不会误删衣服上的白图案
- **网格形变**：3×4 网格拆成 24 个三角形，逐块仿射映射
- **光照融合**：采样躯干区域的环境亮度，把衣服亮度校到同一个量级
- **关键点平滑**：指数滑动平均，静止时衣服不抖
- **网格可视化**：勾选后可以看到形变网格，方便调试

## 快速开始

### 方式一：直接打开

下载仓库，双击 `index.html`。

Chrome 把 `file://` 视为安全上下文，摄像头可以正常调用。如果遇到权限问题，用方式二。

### 方式二：本地起个静态服务

```bash
# Python
python -m http.server 8000

# 或 Node
npx serve .
```

然后打开 http://localhost:8000

### 用法

1. 点「开启摄像头」或「用照片试试」
2. 上传一张平铺的正面衣服图（白底或透明底 PNG 效果最好）
3. 站到画面中间，让上半身完整入镜
4. 用右侧滑杆微调位置

首次打开需要联网：姿态模型（约 5 MB）从 Google 的模型仓库加载，WASM 运行时从 jsDelivr 加载。

## 工作原理

```
衣服图上传
   ├─ 缩放到 640px
   ├─ 洪泛填充去白底（可选）
   └─ 算平均亮度，作为光照基准
                    ↓
每一帧
   ├─ MediaPipe Pose 提取肩(11/12)、髋(23/24)
   ├─ 构造躯干坐标系：axis = 肩中点 → 髋中点
   │                  edge = axis 的法线
   ├─ 3×4 网格：每行按 swell（下摆略宽）和 pinch（腰部收窄）计算半宽
   ├─ 每个网格顶点 → 目标位置
   ├─ 每个三角形：解 3 点仿射，clip + setTransform + drawImage
   └─ 采样环境亮度，用 ctx.filter 校正衣服亮度
```

几个关键细节：

**镜像要在绘制时做，不能用 CSS。** 如果用 `transform: scaleX(-1)` 翻转 canvas，关键点坐标和画面会反向，衣服会跑到人的另一边。

**躯干长度必须夹紧。** 髋部经常在画面外，模型只能靠猜，不夹的话衣服会被拉成两米长的布。代码里限制在肩宽的 0.75–1.7 倍。

**相邻三角形要向外扩 0.7px。** 否则相邻块之间会露出抗锯齿造成的细缝。

## 参数说明

| 滑杆 | 作用 | 什么时候调 |
|---|---|---|
| 衣服宽度 | 缩放网格横向尺寸 | 衣服比人窄/宽过头 |
| 衣服长度 | 缩放网格纵向尺寸 | 下摆位置不对 |
| 上下位置 | 整体沿躯干轴平移 | 领口位置不对 |
| 融合强度 | 整体透明度 | 想让衣服更"融"进去 |
| 光照校正 | 亮度匹配的程度 | 环境很暗/很亮时 |

## 已知局限

这些是 2D 形变路线的固有问题，不是 bug：

1. **没有背面** —— 转身超过 60° 就会明显穿帮。产品上建议限制用户正面站立，或者做 3D 重建。
2. **没有遮挡关系** —— 手臂、长发压不住衣服。需要人体解析（human parsing）按部位分层渲染。
3. **宽松衣服形变差** —— 网格只有 24 个三角形，大摆裙、oversize 卫衣会显得很硬。
4. **光照是假的** —— 只是整体亮度相乘，没有方向光和阴影投射。
5. **衣服图质量决定一切** —— 平铺白底正面图效果最好；模特穿着图、带背景的图会明显变差。

## Roadmap

- [ ] 单应变换（homography）替代仿射，侧身时透视更准
- [ ] 3D 衣服模型模式（GLTF + 布料模拟），解决背面和垂坠感
- [ ] 接入生成模型做"站定后精修"，实时层打底 + 离线层出高质量图
- [ ] 人体解析分层，处理手臂遮挡

## 相关开源项目

做这个领域之前值得先看一眼这些，能省很多弯路：

**实时方向**

- [TechAngelX/softwear](https://github.com/TechAngelX/softwear) — React + Three.js + MediaPipe，3D 衣服模型 + WASM 布料物理，MIT
- [jeeliz/jeelizGlassesVTOWidget](https://github.com/jeeliz/jeelizGlassesVTOWidget) — 商业级 WebGL 实时眼镜 VTO，工程细节值得学
- [VIKHYATH-KS/Virtual-Trial-Room](https://github.com/VIKHYATH-KS/Virtual-Trial-Room) — MediaPipe + homography，思路和本项目接近

**生成式方向**（脱机，不能接摄像头）

- [IDM-VTON](https://github.com/yisol/IDM-VTON) — ECCV 2024，细节最强，CC BY-NC-SA（非商用），≥16 GB 显存
- [CatVTON](https://github.com/Zheng-Chong/CatVTON) — ICLR 2025，最轻，<8 GB 显存，CC BY-NC-SA（非商用）
- [Leffa](https://github.com/franciszzj/Leffa) — CVPR 2025，MIT 代码，但训练数据是非商用数据集，商用属灰区
- [FASHN VTON v1.5](https://huggingface.co/fashn-ai) — Apache-2.0，少数明确可商用的开源模型

**能力组件**

- [IC-Light](https://github.com/lllyasviel/IC-Light) — 重打光，Apache-2.0
- [awesome-virtual-try-on](https://github.com/minar09/awesome-virtual-try-on) — 这个领域最全的论文/项目汇总

⚠️ **许可证提醒**：上面生成式模型里多数是 CC BY-NC-SA，禁止商用。用之前务必确认，包括它们背后的训练数据集（VITON-HD、DressCode 都是非商用）。

## 技术栈

- [MediaPipe Tasks Vision](https://developers.google.com/mediapipe)（Apache-2.0）— 姿态关键点
- Canvas 2D — 三角网格纹理映射
- 无框架、无构建工具、无 npm 依赖

## License

MIT
