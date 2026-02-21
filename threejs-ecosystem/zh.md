# Three.js 生態系：從展示玩具到生產級 3D 平台

## What — 它是什麼

Three.js 是一個 JavaScript 函式庫，將 WebGL（現在還有 WebGPU）抽象為場景圖 API，用於在瀏覽器中渲染 3D。2025-2026 年，它跨過了一個關鍵門檻：WebGPU 在所有主流瀏覽器上達到生產可用，重負載性能提升 10-150 倍，生態系成熟為可以交付真實產品的完整技術棧——不再只是 Demo。

**數據（2026）：**
- NPM 週下載量 270 萬（是最近競爭者的 270 倍）
- GitHub 93K+ stars，每日都有貢獻
- WebGPU 支援 Chrome、Edge、Firefox、Safari（包括 iOS）
- 自 r171（2025 年 9 月）起 WebGPU 生產可用

## Why — 為什麼重要

Web 是最後一個沒有嚴肅 3D 能力的平台。原生應用有 Unity/Unreal，但瀏覽器被困在 WebGL 這個 2011 年代的 API 裡。三件事改變了這一切：

1. **WebGPU 全面登陸** — Safari 26（2025 年 9 月）補完了瀏覽器覆蓋。不再是「只能在 Chrome 上跑」。
2. **Compute Shader 解鎖了 GPU 通用計算** — 物理、ML 推理、粒子系統在 GPU 上跑，不用 CPU。
3. **TSL（Three Shading Language）消滅了 shader 痛點** — 用 JavaScript 寫 shader，自動編譯成 GLSL 或 WGSL。

結果：瀏覽器 3D 從「很酷但受限」變成「大多數場景能跟原生競爭」。

## How — 架構深度解析

### 核心：場景圖 + Object3D 層級

```
Scene (root)
├── Group
│   ├── Mesh (geometry + material)
│   ├── Mesh
│   └── Light
├── Camera
└── Group
    ├── Mesh
    └── InstancedMesh（上千個副本，一次 draw call）
```

**Object3D** 是一切的基類。每個物件維護：
- 局部變換：`position`（Vector3）、`rotation`（Euler）、`scale`（Vector3）
- 世界矩陣：沿層級向下乘算父級變換得到
- 法線矩陣：從世界矩陣推導，用於光照計算

**關鍵設計模式：**
- **延遲初始化** — shader 和 GPU buffer 在首次使用時才建立，不在構造時
- **Shader 快取** — 相同材質共用已編譯的 shader 程式
- **needsUpdate 標誌** — 追蹤哪些 GPU 資源需要重新上傳，跳過其餘
- **組合優於繼承** — 材質組合紋理和 shader 參數，而非深層類別樹

### 渲染管線

```
1. 更新矩陣      → 遍歷場景圖，計算世界變換
2. 視錐剔除      → 用包圍體測試攝影機視野
3. 收集可渲染物  → 聚合 geometry + material + transform 資料
4. 編譯 shader   → 從材質屬性生成 GLSL/WGSL（有快取）
5. 上傳 buffer   → 透過 WebGL/WebGPU 發送幾何到 GPU
6. 綁定狀態      → 設定深度測試、混合、模板
7. Draw call     → 執行索引/非索引繪製
```

渲染器維護 shader、紋理和幾何的 GPU 快取——避免跨幀重複上傳和編譯。

### WebGPU 整合（大升級）

**之前（WebGL）：**
- 單執行緒命令提交
- 無 compute shader
- 隱式狀態管理（容易出錯）
- 只有 GLSL

**之後（WebGPU）：**
- 多執行緒命令準備
- Compute shader 做 GPU 通用計算
- 顯式資源管理
- WGSL + 自動降級到 WebGL 2

**使用只需換一行：**
```javascript
// 舊的
const renderer = new THREE.WebGLRenderer();

// 新的——在舊瀏覽器上自動降級到 WebGL 2
import { WebGPURenderer } from 'three/webgpu';
const renderer = new WebGPURenderer();
```

**性能數據：**

| 指標 | WebGL | WebGPU | 提升 |
|------|-------|--------|------|
| 1 萬粒子更新 | 30ms | — | 基線 |
| 10 萬粒子更新 | 不可能 | <2ms | **150 倍** |
| Draw call 密集場景 | 基線 | 快 10 倍 | Render Bundles |
| 某真實遷移案例 | 基線 | 100 倍 | 平台特定 |

### TSL：Three.js Shading Language

TSL 是架構上最有意思的部分。它是**用 JavaScript 寫的節點圖 shader 系統**，能編譯成 GLSL 和 WGSL。

**工作原理：**

```javascript
// 你寫 JavaScript——不是 shader 字串
const material = new THREE.MeshStandardNodeMaterial();

material.colorNode = texture(map).mul(color(0xff0000));
material.roughnessNode = float(0.5);
material.emissiveNode = color(0x00ffff)
  .mul(sin(time))
  .mul(normalView.dot(positionView).abs());
```

**底層機制：**

1. 每個函式呼叫（`sin()`、`mul()`、`texture()`）建立一個 **Node 物件**——不是立即計算
2. Node 形成**抽象語法樹**，描述資料流
3. **NodeBuilder** 分析圖、最佳化（死碼消除、變數復用、uniform 去重）
4. **WGSLNodeBuilder** 或 **GLSLNodeBuilder** 生成目標 shader 程式碼
5. 同一份 JavaScript → 不同 GPU 語言，開發者零介入

**為什麼這很重要：**

| 面向 | 舊方式（ShaderMaterial） | 新方式（TSL） |
|------|--------------------------|--------------|
| 語言 | JS 裡的 GLSL 字串 | JavaScript 函式 |
| IDE 支援 | 無（它是字串） | 完整自動補全、重構 |
| 錯誤訊息 | 難懂的 GPU 錯誤 | JavaScript 堆疊追蹤 |
| 光照/陰影 | 從零重建 | 自動——擴展標準材質 |
| 跨平台 | 只有 GLSL | 自動編譯為 GLSL 或 WGSL |
| 可組合性 | 複製貼上字串 | Import/export JS 模組 |

**頂點位移範例：**
```javascript
material.positionNode = Fn(() => {
  const pos = positionLocal;
  const norm = normalLocal;
  const displacement = sin(time.mul(3.0).add(pos.y.mul(5.0))).mul(0.075);
  return pos.add(norm.mul(displacement));
})();
```

## 生態系技術棧

### 核心層

| 函式庫 | 角色 | 大小 |
|--------|------|------|
| **three** | 場景圖、渲染器、材質 | ~600KB min |
| **WebGPURenderer** | WebGPU 後端（r171 起內建） | 已包含 |
| **TSL** | 節點式 shader 系統 | 已包含 |

### React 層（pmndrs 生態系）

| 函式庫 | 角色 |
|--------|------|
| **react-three-fiber (R3F)** | Three.js 的 React 渲染器——宣告式 JSX 場景圖 |
| **@react-three/drei** | ~100 個即用 helper（OrbitControls、Text、Environment 等） |
| **@react-three/postprocessing** | Bloom、SSAO、DOF、色差——性能最佳化 |
| **@react-three/rapier** | Rapier 物理引擎綁定 |
| **leva** | 參數調整的 GUI 控制器 |
| **zustand** | 狀態管理（同團隊） |

### R3F vs 原生 Three.js 選擇

| 場景 | 選擇 | 原因 |
|------|------|------|
| React 應用加 3D | R3F | 整合 React 生命週期、狀態、路由 |
| 自定義渲染管線 | 原生 | 完全控制，無抽象開銷 |
| 複雜 shader 遊戲 | 原生 | 直接 WebGPU 存取，自定義渲染迴圈 |
| 產品配置器/儀表板 | R3F | 宣告式、元件化、快速原型 |
| 極致性能 | 原生 | 零 React reconciliation 開銷 |

## 人們實際在做什麼

### 瀏覽器遊戲（包括 MMO）

| 專案 | 類型 | 亮點 |
|------|------|------|
| stein.world | 瀏覽器 MMORPG | 「世界首個全功能瀏覽器即時 MMORPG」 |
| mal-war.com | MMORPG | 完全用 Three.js |
| Ironbane | 瀏覽器 MMO | Three.js + Node.js + GLSL |
| Star Defenders 3D | 多人 FPS | 可破壞體素世界 |
| riseintime.com | 策略多人遊戲 | Three.js |

### 獲獎級互動體驗

| 創作者 | 作品 | 獎項 |
|--------|------|------|
| Samsy | 賽博朋克世界，WebGPU，120+ FPS，第一人稱 | 年度作品集候選 |
| Jordan Breton | 浮島 + 草地/瀑布/火焰/蝴蝶 | FWA Site of the Day（2025.10） |
| JReyes MC | Minecraft 風格 3D 作品集 | Awwwards Honorable Mention |
| Bruno Simon | 開車探索 3D 世界 | Three.js 經典標杆 |
| WoraWork | 薩爾達/動物森友會風格可探索世界 | 社群最愛 |

### 電商 & 產品配置器

- **IKEA** — 3D + AR 家具視覺化
- **Lolo Chateney** — 奢侈品牌 3D 包包定制器
- **Thömus** — 3D 自行車配置器
- **眼鏡零售商** — 瀏覽器內 AR 試戴

### 創意編碼 & 生成藝術

- **Genuary 2026** — 年度生成藝術挑戰，Three.js 是主力工具
- 自定義 GLSL shader 程式化視覺效果（光標拖尾、Bloom、膠片顆粒）
- WebGPU compute 互動粒子模擬（10 萬+粒子）
- InstancedMesh + GLSL 做上千物件的物理反應

## Tradeoffs — 取捨

### Three.js 做得好的

- **生態系統統治地位** — 下載量是競爭者的 270 倍，社群龐大
- **靈活性** — 它是函式庫，不是框架。按需組合
- **WebGPU 優先** — 生產可用的 WebGPU + 自動 WebGL 降級
- **TSL** — Web 上最好的 shader 撰寫體驗
- **React 整合** — R3F 是現存最好的宣告式 3D 框架

### Three.js 做得差的

| 弱點 | 細節 |
|------|------|
| **不是遊戲引擎** | 無內建物理、動畫狀態機、ECS、音訊系統——需要第三方函式庫 |
| **陰影系統僵硬** | 單一 `scene.shadowMap`——不像 Babylon.js 有逐物件陰影生成器 |
| **後處理需要外掛** | 無內建 bloom、SSAO、DOF——需要 pmndrs/postprocessing |
| **學習曲線** | 需要理解 3D 圖學概念（矩陣、shader、GPU 管線） |
| **打包大小** | 核心 ~600KB min。Babylon.js 2MB+ 但什麼都有 |
| **無視覺編輯器** | 無內建場景編輯器。Babylon.js 有 Playground，PlayCanvas 有完整雲端編輯器 |

### 與替代方案比較

| 特性 | Three.js | Babylon.js | PlayCanvas |
|------|----------|------------|------------|
| 哲學 | 函式庫（組合） | 引擎（全包） | 引擎 + 雲端編輯器 |
| 打包大小 | ~600KB | ~2MB+ | ~150KB 引擎 |
| 內建物理 | 否（用 Rapier/Cannon） | 是（Havok） | 是（ammo.js） |
| 視覺編輯器 | 否 | Playground/Inspector | 完整雲端 IDE |
| WebGPU | 生產可用（r171+） | 生產可用 | 進行中 |
| 社群規模 | 最大（93K stars） | 大（23K stars） | 較小（9K stars） |
| React 整合 | 優秀（R3F） | 有限 | 無 |
| NPM 週下載 | 270 萬 | ~10 萬 | ~1 萬 |
| 最適合 | 創意 Web、配置器、React 應用 | 複雜遊戲、企業級 | 團隊協作、快速原型 |

**核心洞察：** Three.js 以生態系和靈活性取勝。Babylon.js 以「全包」在遊戲領域取勝。PlayCanvas 以雲端編輯器在團隊工作流程上取勝。根據你要做的東西選，不要看合成跑分。

## 生產性能模式

### 必做最佳化

1. **共用材質和幾何** — 每個獨特材質都要編譯 shader，每個獨特幾何都要上傳 buffer
2. **用 InstancedMesh** — 1 萬棵樹？一次 draw call，不是 1 萬次
3. **用 glTF + Draco 壓縮** — 網格大小降到 OBJ/COLLADA 的 <10%
4. **縮小攝影機視錐** — 更小的 near/far 範圍 = 更好的深度精度和剔除
5. **按需渲染** — 靜態場景不需要 60fps 渲染迴圈
6. **用 `powerPreference: "high-performance"`** — 提示瀏覽器在筆電上使用獨立顯卡

### 常見陷阱

| 錯誤 | 修正 |
|------|------|
| 在迴圈中建立物件 | 用 `vector.set()` 復用 |
| 啟用 `preserveDrawingBuffer` | 只在需要 canvas 截圖時啟用 |
| 每個 mesh 用新 Material | 相似 mesh 共用材質 |
| OBJ/COLLADA 格式 | 改用 glTF + Draco |
| 靜態場景每幀渲染 | 按需渲染（攝影機移動、動畫時） |
| 巨大紋理 | 用 KTX2/basis 壓縮紋理 |

## 最新動態 (2026)

### 發布節奏：r170 到 r183

Three.js 在 2025 至 2026 年初持續保持激進的月度發布節奏。關鍵里程碑版本：

- **r170**（2024 年 10 月）— 萬聖節版本，持續穩定 WebGPU。
- **r171**（2025 年 9 月）— 里程碑版本：全域代碼分割 WebGL/WebGPU 入口點，引入 `three.tsl.js`，零配置 `import { WebGPURenderer } from 'three/webgpu'`。
- **r175**（2025 年 3 月）— 增強 TSL：材質內 compute、`RaymarchingBox`、`debug()` 函式、while 迴圈支援，改進 WebXR Layers 整合。
- **r177**（2025 年中）— 新增 `maskNode`、`toJSON()`/`fromJSON()` 場景序列化、改進 `shapeCircle()`。
- **r178**（2025 年 6 月）— 完成 112 個目標。新增渲染器 Float16Array 支援，修正所有後端的混合公式。
- **r181**（2025 年 11 月）— 渲染品質大躍進：**螢幕空間全域光照（SSGINode）**、**螢幕空間陰影（SSSNode）**、DFG LUT 取代解析近似、GGX VNDF 重要性採樣、多散射能量補償、移除 WebGL 1.0 程式碼、WebGPU Inspector 除錯工具。
- **r182**（2025 年 12 月）— ESLint 9 遷移、現代化陰影映射、WebGPU 多渲染目標（MRT）支援。
- **r183**（2026 年 2 月）— `BezierInterpolant` 動畫、雙渲染器反向深度緩衝區、`BatchedMesh` 逐實例透明度和線框支援、USDC/USD 格式載入、GLTFLoader 中 `KHR_meshopt_compression`、TSL `retroPass` 和 `exponentialHeightFogFactor()`、棄用 `Clock` 改用 `Timer`。

### GitHub 和 NPM 增長

Three.js 已突破 **110K GitHub stars**（從 2025 年中的 93K 增長），現在記錄 **每週 540 萬+ NPM 下載量** — 不到一年從 270 萬翻倍。生態系統統治地位更加明顯：大約是 Babylon.js 下載量的 **500 倍**。

### Vibe Coding 革命

「Vibe coding」— 使用自然語言提示配合 AI 工具（Claude、ChatGPT、Cursor）生成程式碼 — 被評為 Collins 辭典 2025 年度詞彙。Three.js 成為這一工作流程的首選函式庫，原因：

1. **簡單 API** — 場景設置約 10 行程式碼，非常適合 LLM 生成。
2. **即時視覺回饋** — 結果在瀏覽器中即刻渲染。
3. **龐大訓練語料** — 15 年以上的範例、教程和 StackOverflow 回答。
4. **無需安裝引擎** — 從 CDN 或 `npm install three` 即可運行。

獨立開發者報告使用 AI 配對程式設計在數天內構建完整的 3D 瀏覽器遊戲。Rosebud AI 發布了一系列完全用 vibe coding 製作的 Three.js 遊戲，多位開發者從 AI 生成的遊戲原型中獲得了可觀收入。

### WebGPU：從實驗到預設

自 Safari 26 於 2025 年 9 月支援 WebGPU（macOS、iOS、iPadOS、visionOS），WebGPU 在**所有現代瀏覽器上 100% 可用**。Three.js 現在將 WebGPU 視為主要路徑：

- `WebGPURenderer` 自動降級到 WebGL 2 — 不需要功能檢測。
- 首次渲染前通過 `await renderer.init()` 非同步初始化是唯一的新要求。
- Compute shader 解鎖了瀏覽器中的 GPU 端物理、粒子系統和 ML 推理。
- 真實世界遷移報告 draw-call 密集場景 **2-10 倍** 提升，邊界情況在百萬點數據集上達到 **100 倍**。

### 進階渲染功能（內建）

r181-r183 週期將以前只能通過後處理外掛獲得的渲染能力直接帶入核心：

| 功能 | 版本 | 影響 |
|------|------|------|
| SSGINode（螢幕空間全域光照） | r181 | 無需烘焙貼圖的即時色彩溢出和光線反彈 |
| SSSNode（螢幕空間陰影） | r181 | 逐像素接觸陰影，不需要陰影貼圖 |
| 反向深度緩衝區 | r183 | 消除大距離下的 z-fighting（太空、地形） |
| MRT（多渲染目標） | r182 | 一次 pass 寫入多張紋理 — 啟用延遲渲染 |
| BatchedMesh 透明度/線框 | r183 | 複雜場景的逐實例視覺控制 |

### TSL 成熟化

TSL 已從實驗性質演變為推薦的 shader 撰寫路徑。2025-2026 關鍵新增：

- **材質內 Compute**（r175）— 直接從材質節點運行 compute 操作。
- **RaymarchingBox / raymarchingTexture3D**（r174）— 通過 TSL 實現體積渲染。
- **TSL Transpiler 擴展** — 矩陣類型、varying 支援、discard 操作。
- **`debug()` 函式**（r175）— 開發時檢查中間節點值。
- **`retroPass`**（r183）— 復古/CRT 視覺效果作為內建後處理節點。
- **`exponentialHeightFogFactor()`**（r183）— 基於高度的霧效，無需自定義 shader。

### WebXR 與空間計算

Three.js WebXR 支援隨新硬體一起成熟：

- **Samsung Galaxy XR**（2025 年 10 月）推出時即原生支援 WebXR。
- **Apple Vision Pro** 持續通過 Safari 支援 WebXR。
- r175 新增更深入的 **WebXR Layers** 整合，提升 VR/AR 覆蓋層性能。
- r174 新增 XR 渲染的色調映射和輸出色彩空間支援。
- 使用場景擴展到 VR 手術訓練、房地產虛擬導覽、瀏覽器內 AR 產品試用。

### React Three Fiber：穩定期

R3F 在 2025 年末進入穩定期 — 沒有重大破壞性變更，專注於可靠性。關鍵生態系發展：

- **@react-three/drei** 達到 v10.7+，擁有 100+ 輔助元件。
- **Threlte**（Svelte + Three.js）作為 R3F 的 Svelte 等價物獲得了關注，正在向 Svelte 5 相容性努力。
- R3F 正從「網站 3D」演進為支援複雜即時應用、即時模擬和瀏覽器原生遊戲。
- 通過 React Server Components 和 Next.js 整合進行 3D 場景的伺服器端渲染現已可行，改善 SEO 和初始載入性能。

## Steal — 可直接復用的模式

### 1. WebGPU 自動降級
```javascript
import { WebGPURenderer } from 'three/webgpu';
// 零設定——在舊瀏覽器上自動降級到 WebGL 2
```
**模式：** 永遠瞄準更好的 API，優雅降級。不需要功能檢測。

### 2. TSL 跨平台 Shader
用 JavaScript 寫一次 shader 邏輯，編譯到瀏覽器支援的任何格式。這是正確的抽象層級——不是原始 GLSL 字串，不是視覺節點編輯器，而是有型別的 JavaScript 函式。

### 3. InstancedMesh 批量物件
一份幾何 + 一份材質 + 實例變換 = 上千物件一次 draw call。這個模式適用於任何渲染系統，不只 Three.js。

### 4. R3F 的宣告式 3D 模式
```jsx
<Canvas>
  <mesh position={[0, 1, 0]}>
    <boxGeometry />
    <meshStandardMaterial color="orange" />
  </mesh>
</Canvas>
```
**模式：** 把 3D 場景當元件樹。狀態驅動渲染。React reconciler 處理更新。這是 3D UI 開發的未來。

### 5. 按需渲染
不動就不渲染。監聽互動事件，只在需要時動畫。這一招就能讓半靜態場景的 GPU 功耗降低 90%。

### 6. pmndrs 技術棧
`R3F + drei + rapier + postprocessing + zustand + leva` = 完整 3D 應用棧，零自建基礎設施。這種「精選的可組合函式庫生態系」模式，在 Web 開發中打敗了「一個巨型引擎」。

## References

### 官方

- [Three.js 官方網站 & 範例](https://threejs.org/examples/)
- [Three.js WebGPU 文件](https://threejs.org/docs/pages/WebGPU.html)
- [Three.js Shading Language (TSL) Wiki](https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language)
- [React Three Fiber (R3F) GitHub](https://github.com/pmndrs/react-three-fiber)
- [R3F 文件](https://docs.pmnd.rs/react-three-fiber)
- [pmndrs/postprocessing GitHub](https://github.com/pmndrs/postprocessing)
- [react-three-rapier GitHub](https://github.com/pmndrs/react-three-rapier)

### WebGPU & TSL 深度解析

- [What's New in Three.js (2026): WebGPU, New Workflows & Beyond](https://www.utsubo.com/blog/threejs-2026-what-changed)
- [Field Guide to TSL and WebGPU — Maxime Heckel](https://blog.maximeheckel.com/posts/field-guide-to-tsl-and-webgpu/)
- [TSL: A Better Way to Write Shaders — Three.js Roadmap](https://threejsroadmap.com/blog/tsl-a-better-way-to-write-shaders-in-threejs)
- [Why TSL is So Interesting — Three.js Forum](https://discourse.threejs.org/t/why-tsl-three-js-shading-language-is-so-interesting/56306)
- [WebGL vs WebGPU Explained — Three.js Roadmap](https://threejsroadmap.com/blog/webgl-vs-webgpu-explained)
- [Moving from WebGL to WebGPU in Three.js — Sude Nur Çevik](https://medium.com/@sudenurcevik/upgrading-performance-moving-from-webgl-to-webgpu-in-three-js-4356e84e4702)
- [Three.js Introduction to WebGPU and TSL — Forum](https://discourse.threejs.org/t/three-js-introduction-to-webgpu-and-tsl/78205)
- [GPU Acceleration in Browsers: WebGPU Benchmarks](https://www.mayhemcode.com/2025/12/gpu-acceleration-in-browsers-webgpu.html)

### 架構與內部機制

- [Scene Graph & Object System — DeepWiki](https://deepwiki.com/mrdoob/three.js/2.3-scene-graph-and-object-system)
- [How THREE.js Abstracts 3D Graphics — Evan Perry](https://medium.com/@evmaperry/how-three-js-abstracts-away-the-complexities-of-programming-3d-graphics-c3b74dbf051c)
- [Three.js Renderer Docs](https://threejs.org/docs/pages/Renderer.html)
- [R3F Performance Pitfalls](https://r3f.docs.pmnd.rs/advanced/pitfalls)

### 比較與替代方案

- [React Three Fiber vs Three.js in 2026 — GraffersID](https://graffersid.com/react-three-fiber-vs-three-js/)
- [Three.js vs Babylon.js — LogRocket](https://blog.logrocket.com/three-js-vs-babylon-js/)
- [Why We Use Babylon.js Instead of Three.js — Spot Virtual](https://www.spotvirtual.com/blog/why-we-use-babylonjs-instead-of-threejs-in-2022)
- [Babylon.js vs Three.js — Slant](https://www.slant.co/versus/11077/11348/~babylon-js_vs_three-js)
- [PlayCanvas vs Three.js — Slant](https://www.slant.co/versus/5149/11348/~playcanvas_vs_three-js)
- [Overview of 3D Web Rendering Engines — VIVERSE](https://docs.viverse.com/optimization/overview-of-3d-web-rendering-engines)
- [JS Game Rendering Benchmark — GitHub](https://github.com/Shirajuki/js-game-rendering-benchmark)

### 展示 & 專案

- [Best Three.js Websites — Awwwards](https://www.awwwards.com/websites/three-js/)
- [Three.js Forum Showcase](https://discourse.threejs.org/c/showcase/7)
- [10 Award-Winning Three.js Projects — Orpetron](https://orpetron.com/blog/10-award-winning-projects-showcasing-three-js-innovation/)
- [Best Three.js Portfolio Examples 2025 — CreativeDevJobs](https://www.creativedevjobs.com/blog/best-threejs-portfolio-examples-2025)
- [20 Best Three.js Examples 2025 — UI Cookies](https://uicookies.com/threejs-examples/)
- [142 Three.js Examples — FreeFrontend](https://freefrontend.com/three-js/)
- [50+ Three.js Examples — DevSnap](https://devsnap.me/three-js-examples)
- [Three.js Demos — Codrops](https://tympanus.net/codrops/hub/tag/three-js/)
- [Top Games Made with Three.js — itch.io](https://itch.io/games/made-with-threejs)
- [Genuary 2026 — Three.js Forum](https://discourse.threejs.org/t/genuary-2026-generative-january/89632)

### 性能 & 最佳化

- [Building Efficient Three.js Scenes — Codrops](https://tympanus.net/codrops/2025/02/11/building-efficient-three-js-scenes-optimize-performance-while-maintaining-quality/)
- [The Big List of Three.js Tips and Tricks — Discover Three.js](https://discoverthreejs.com/tips-and-tricks/)
- [Performance Tips — Three.js Journey](https://threejs-journey.com/lessons/performance-tips)
- [Three.js vs WebGPU for 500MB+ Models — AlterSquare](https://altersquare.io/three-js-vs-webgpu-construction-3d-viewers-scale-beyond-500mb/)
- [100 Three.js Tips That Actually Improve Performance (2026)](https://www.utsubo.com/blog/threejs-best-practices-100-tips)

### 教程 & 學習

- [Three.js Journey (Paid Course)](https://threejs-journey.com/)
- [Discover Three.js (Free Book)](https://discoverthreejs.com/)
- [Wawa Sensei — R3F + WebGPU/TSL Course](https://wawasensei.dev/courses/react-three-fiber/lessons/webgpu-tsl)
- [Three.js WebGPU Renderer Tutorial — SBCode](https://sbcode.net/threejs/webgpu-renderer/)
- [TSL Getting Started — SBCode](https://sbcode.net/tsl/getting-started/)
- [From Websites to Games: Future of R3F — Kris Baumgartner](https://gitnation.com/contents/from-websites-to-games-the-future-of-react-three-fiber)
- [Building 3D Web Apps in 2025 — Bela Bohlender](https://gitnation.com/contents/building-3d-web-apps-in-2025-react-xr-and-ai)

### 發布說明（2025-2026）

- [Three.js Releases — GitHub](https://github.com/mrdoob/three.js/releases)
- [Release r175 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r175)
- [Release r178 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r178)
- [Release r181 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r181)
- [Release r183 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r183)
- [Migration Guide — GitHub Wiki](https://github.com/mrdoob/three.js/wiki/Migration-Guide)
- [Migrate Three.js to WebGPU (2026) — Complete Checklist](https://www.utsubo.com/blog/webgpu-threejs-migration-guide)

### Vibe Coding & AI 整合

- [5 Three.js Game Examples Vibe Coded with AI — Rosebud AI](https://lab.rosebud.ai/blog/three-js-game-examples-vibe-coded)
- [Ultimate Guide to Vibe Coding Games with Three.js and Cursor](https://webtech.tools/the-ultimate-guide-to-vibe-coding-games-in-2025)
- [Getting AI to Write TSL That Works — Three.js Roadmap](https://threejsroadmap.com/blog/getting-ai-to-write-tsl-that-works)
- [Three.js MCP Server — AI Control of 3D Scenes](https://skywork.ai/skypage/en/3d-worlds-ai-threejs-mcp-server/1980470680631943168)

### WebXR & 空間計算

- [Top VR/AR/XR Use Cases 2026 — Three.js Resources](https://threejsresources.com/vr/blog/top-vr-ar-xr-use-cases-in-2026-building-immersive-experiences-that-deliver-real-value)
- [Best VR Headsets for WebXR & Three.js Development 2026](https://threejsresources.com/vr/blog/best-vr-headsets-with-webxr-support-for-three-js-developers-2026)
- [SSGI WebGPU Demo — Anderson Mancini](https://ssgi-webgpu-demo.vercel.app/)
- [Interactive Text Destruction with Three.js, WebGPU, and TSL — Codrops](https://tympanus.net/codrops/2025/07/22/interactive-text-destruction-with-three-js-webgpu-and-tsl/)

### 生態系與框架

- [Threlte — Svelte 3D 框架](https://threlte.xyz/)
- [React Three Fiber: Stability in Late 2025 — Needle Radio](https://needle.tv/episode/react-three-fiber-a-quiet-period-of-stability-in-late-2025-852)
- [Drei 文件](https://drei.docs.pmnd.rs/)
- [Three.js NPM 統計](https://www.npmjs.com/package/three)
- [Three.js NPM 趨勢](https://npmtrends.com/three)
