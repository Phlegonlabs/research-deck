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
