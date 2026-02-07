# Three.js 視覺設計：如何讓東西好看

## What — 這是什麼

一份實用的技術拆解，涵蓋讓 Three.js 場景看起來像電影般的所有技術——從光劍發光效果到照片級渲染。這不是理論——而是一本具體設定、shader 和後處理 pass 的配方書，把平淡的 3D 變成人們真正想看的東西。

核心洞察：**80% 的視覺品質來自 5 樣東西** — tone mapping、環境光照、bloom、色彩空間和陰影。把這些搞對，一切都好看。漏掉任何一個，一切都業餘。

## 「電影感」配方

### 第一步：渲染器基礎

這 4 行是「預設 Three.js」和「看起來專業」的區別：

```javascript
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.8;
renderer.outputColorSpace = THREE.SRGBColorSpace;
```

| 設定 | 做什麼 | 為什麼重要 |
|------|--------|-----------|
| `ACESFilmicToneMapping` | 用電影業 ACES 標準壓縮 HDR 到顯示範圍 | 濃郁黑色、柔和高光、無裁切——「電影感」 |
| `toneMappingExposure` | 控制多少光進入（像相機曝光） | 太低 = 暗/渾濁，太高 = 泛白。從 1.0-2.0 開始 |
| `SRGBColorSpace` | 應用 gamma 校正適應人眼感知 | 沒有它色彩就不對——中間調太亮、暗部被壓 |
| `antialias` | 平滑鋸齒邊緣 | 免費的視覺升級，輕微性能開銷 |

**Tone mapping 對比：**

| 演算法 | 特性 | 最適合 |
|--------|------|--------|
| `NoToneMapping` | 原始 HDR 值，高光爆掉 | 永遠別用在正式場景 |
| `LinearToneMapping` | 平坦均勻曝光 | 技術視覺化 |
| `ReinhardToneMapping` | 柔和，略帶沖洗感 | 戶外場景 |
| `ACESFilmicToneMapping` | 豐富對比，電影感 | **大多數場景的預設選擇** |
| `AgXToneMapping` | 最精確的色彩保留 | 色彩保真度最重要時 |

### 第二步：環境光照（HDRI）

這是最大的單次視覺升級。一張 HDRI 貼圖取代幾十個手動燈光，讓一切看起來真實：

```javascript
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';

const rgbeLoader = new RGBELoader();
rgbeLoader.load('/studio.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.background = texture;    // 360° 背景
  scene.environment = texture;   // 照亮場景中的一切
});
```

**解析度指引：**
- 純光照（無可見反射）：256x256 就夠
- 有可見反射的金屬材質：至少 1024x1024
- 當作可見背景：2048x2048 或更高

**去哪找 HDRI：** [Poly Haven](https://polyhaven.com/) — 免費、高品質、CC0 授權。

**進階技巧：** 可以用低解析度 HDRI 做光照，高解析度做背景：
```javascript
scene.environment = lowResHDRI;    // 便宜的光照
scene.background = highResHDRI;    // 漂亮的背景
```

### 第三步：會對光反應的材質

預設材質看起來像塑膠。這些設定修正它：

**寫實金屬：**
```javascript
new THREE.MeshStandardMaterial({
  color: 0xcccccc,
  roughness: 0.2,    // 低 = 光亮, 高 = 霧面
  metalness: 1.0,    // 1 = 金屬, 0 = 非金屬
  envMapIntensity: 1.0
});
```

**玻璃/水晶：**
```javascript
new THREE.MeshPhysicalMaterial({
  roughness: 0,
  metalness: 0,
  transmission: 1,    // 1 = 完全透明
  ior: 1.5,          // 玻璃 = 1.5, 鑽石 = 2.42, 水 = 1.33
  thickness: 0.5     // 影響折射深度
});
```

**材質核心原則：**
- `roughness` 和 `metalness` 是兩個最重要的旋鈕——其他都是次要的
- 真實世界材質要嘛是金屬的（`metalness: 1`）要嘛是非金屬的（`metalness: 0`）——中間值很少見
- `MeshPhysicalMaterial` 擴展了 `MeshStandardMaterial`，加入清漆、透射、光澤——只在需要時用（更貴）

### 第四步：陰影

陰影讓物件「踩在地上」。沒有它們，一切都在飄：

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

const light = new THREE.DirectionalLight(0xffffff, 1);
light.castShadow = true;
light.shadow.mapSize.set(2048, 2048);
light.shadow.bias = -0.0001;         // 修復陰影痘（shadow acne）
light.shadow.normalBias = 0.02;      // 修復 peter-panning

mesh.castShadow = true;
ground.receiveShadow = true;
```

**假接觸陰影（更便宜，通常更好看）：**
在物件下方放一個半透明平面配徑向漸層貼圖，假造接觸陰影而不需要任何 shadow map 計算。很多獲獎網站都用這招。

### 第五步：後處理

後處理就是 3D 的電影調色。它拿到渲染好的圖像然後增強它。

**基本設定：**
```javascript
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/examples/jsm/postprocessing/OutputPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));       // 1. 渲染場景
composer.addPass(new UnrealBloomPass(                  // 2. 加 bloom
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.8,    // 強度 (0.5-2.0)
  0.5,    // 半徑 (0.5-1.0)
  0.85    // 亮度閾值 (0.8-1.0)
));
composer.addPass(new OutputPass());                     // 3. 色彩空間轉換

// 渲染迴圈中：composer.render() 代替 renderer.render()
```

**管線順序很重要：**
```
RenderPass → [Effect Passes] → OutputPass
     ↑              ↑               ↑
  渲染 3D      Bloom/SSAO 等    sRGB + tone map
```

## 光劍效果：完整拆解

這就是激發這次調研的東西——以下是確切做法：

### 劍刃發光

劍刃本身用**兩層**：

1. **核心：** 圓柱/膠囊 mesh + 明亮自發光材質（`emissive: 0xffffff, emissiveIntensity: 2`）
2. **外層光暈：** 相同幾何放大 1.2-1.5 倍 + 自定義 glow shader

**Glow shader 技巧：**
```javascript
// 頂點 shader 概念
// 光暈強度 = dot(視角方向, 表面法線) 的反值
// 邊緣背對攝影機 → 高光暈
// 中心面對攝影機 → 低光暈
float intensity = pow(1.0 - dot(viewDir, normal), 2.0);
```

原理是球體/圓柱的邊緣法線垂直於視角——所以 dot product 很低，而 `1 - 低 = 高`，邊緣光暈強。跟真實霓虹燈管邊緣最亮是同樣道理。

### 動態拖尾（揮劍軌跡）

**Polyboard 技術**：

1. 每幀追蹤劍尖位置 → 建立折線（3D 點列表）
2. 在每個點垂直於運動方向展開 → 建立有寬度的三角帶
3. 沿帶狀從 alpha 1.0（最新）衰減到 0.0（最舊）
4. 套用同樣的 glow shader + bloom 後處理
5. 每幀移除舊點保持拖尾長度恆定

**現成方案：** [TrailRendererJS](https://github.com/mkkellogg/TrailRendererJS) — 指定拖尾形狀和目標 Object3D 就能附加動態拖尾。

### 選擇性 Bloom（只讓該亮的亮）

問題：bloom 影響所有亮的東西。你只想讓光劍發光，不是白色地板。

**Layer 技術：**
```
1. 把發光物件分配到 Layer 1
2. Bloom composer：只渲染 Layer 1 物件 + bloom
3. Main composer：渲染完整場景
4. 合併：把 bloom 層合成到主渲染之上
```

在 bloom pass 期間，非發光物件暫時設為黑色材質避免貢獻 bloom。然後在最終渲染前恢復材質。

**替代方案：** 把 `UnrealBloomPass.threshold` 調高到只有自發光材質能觸發 bloom。更簡單但控制較少。

## 視覺效果配方書

### 體積光 / 上帝光線（God Rays）

**徑向模糊技術：**
1. 把光源渲染為白色、所有遮擋物渲染為黑色 → 遮擋紋理
2. 從光源位置向外套用徑向模糊 shader
3. 加法混合結果到正常場景渲染之上

**便宜替代：** 用 `ConeGeometry` + 加法混合 + 低透明度 → 從任何點光源即時產生「光束」。

### 大氣霧

```javascript
scene.fog = new THREE.FogExp2(0x000000, 0.02);  // 指數霧
// 或
scene.fog = new THREE.Fog(0x000000, 10, 100);    // 線性霧（近, 遠）
```

霧增加景深感、隱藏遠處 LOD 切換、營造氛圍。深色霧 = 神秘。淺色霧 = 夢幻。

### 溶解 / 碎裂效果

用噪聲紋理搭配自定義 shader：
1. 在每個 fragment 採樣噪聲
2. 把噪聲值和 `progress` uniform（0→1）做比較
3. 如果 noise < progress，`discard` 該 fragment
4. 在邊緣（noise ≈ progress）加上明亮自發光色 + bloom

### 調色 / LUT

要電影感色彩：
- **分離調色：** 陰影染藍/青，高光染橙/暖色 → 即刻「好萊塢」感
- **S 曲線對比：** 暗的更暗、亮的更亮 → 更豐富的圖像
- **3D LUT（.cube 檔）：** 從 DaVinci Resolve/Photoshop 輸出的專業調色，作為後處理套用

## 視覺品質的 5 層級

從「預設灰」到「獲獎級」：

```
Layer 0：預設 Three.js
├── 平光、無陰影、錯誤色彩空間
├── 看起來像 2005 年 Flash 遊戲
│
Layer 1：基礎（5 分鐘工作量）
├── + ACESFilmic tone mapping
├── + sRGB 輸出色彩空間
├── + 抗鋸齒
├── 已經好 3 倍了
│
Layer 2：光照（15 分鐘）
├── + HDRI 環境貼圖
├── + 陰影（PCFSoft + bias 調整）
├── + 正確的材質 roughness/metalness
├── 現在看起來「真實」了
│
Layer 3：後處理（30 分鐘）
├── + Bloom（選擇性、閾值調整）
├── + SSAO（微妙的環境光遮蔽）
├── + 暗角（Vignette）
├── 看起來「有電影感」了
│
Layer 4：打磨（數小時）
├── + 調色 / LUT
├── + 景深
├── + 自定義 shader（glow、trails、dissolve）
├── + 體積光效果
├── 獲獎水準
│
Layer 5：WebGPU（前沿）
├── + Compute shader 粒子（10 萬+）
├── + TSL 自定義材質
├── + 即時光線追蹤提示
├── 次世代瀏覽器 3D
```

## Tradeoffs — 取捨

| 技術 | 視覺衝擊 | 性能開銷 | 複雜度 |
|------|---------|---------|--------|
| Tone mapping | 高 | 零 | 1 行 |
| HDRI 環境光 | 非常高 | 低 | 5 行 |
| 陰影 | 高 | 中 | 需要調參 |
| Bloom | 高 | 中 | 用 pmndrs 很簡單 |
| SSAO | 中 | 中高 | 用 pmndrs 很簡單 |
| 選擇性 bloom | 高 | 高（2 次渲染） | 複雜 |
| 體積光 | 非常高 | 高 | 自定義 shader |
| 景深 DOF | 中 | 中 | 用 pmndrs 很簡單 |
| 動態拖尾 | 高（動作場景） | 低中 | TrailRendererJS |
| 調色/LUT | 中 | 低 | 需要設計眼光 |

## Steal — 可直接復用的模式

### 1. 四行基礎
```javascript
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.5;
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.shadowMap.enabled = true;
```
先做這個。永遠。沒得商量。

### 2. HDRI = 即刻真實感
一個 `.hdr` 檔取代一堆燈。用 Poly Haven。同時設為 `scene.background` 和 `scene.environment`。

### 3. 光劍技術棧
`自發光核心 mesh + 放大 glow shader mesh + bloom 後處理 + 動態拖尾（Polyboard）` = 完整武器發光效果。

### 4. 透過 Layer 做選擇性 Bloom
把自發光物件分到獨立 Layer。只在那個 layer 渲染 bloom pass。合成到主場景之上。這是每個發光物件 demo 都用的標準技術。

### 5. pmndrs 後處理棧
別自己搭 EffectComposer 鏈——用 `@react-three/postprocessing` 或 `pmndrs/postprocessing`。它們把效果合併到更少的 pass 中以獲得更好的性能。

### 6. 先假裝再做真的
- 假接觸陰影（漸層平面）> 真 shadow map（簡單場景）
- 假光暈（放大 mesh + glow shader）> 完整 bloom pass（單個物件）
- 假體積光（圓錐 + 加法混合）> 真 god rays（大多數情況）
- 烘焙光照貼圖 > 即時光源（靜態場景）

最好看的 Three.js 場景，通常是假裝最多東西的那些。
