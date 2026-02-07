# Three.js Visual Design: How to Make Things Look Good

## What

A practical breakdown of every technique that makes Three.js scenes look cinematic, from lightsaber glow effects to photorealistic rendering. This is not theory — it's a recipe book of specific settings, shaders, and post-processing passes that transform flat-looking 3D into something people actually want to look at.

The core insight: **80% of visual quality comes from 5 things** — tone mapping, environment lighting, bloom, color space, and shadows. Get these right, everything looks good. Miss any one, everything looks amateur.

## The "Cinematic Look" Recipe

### Step 1: Renderer Foundation

These 4 lines are the difference between "default Three.js" and "looks professional":

```javascript
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.8;
renderer.outputColorSpace = THREE.SRGBColorSpace;
```

| Setting | What It Does | Why It Matters |
|---------|-------------|----------------|
| `ACESFilmicToneMapping` | Compresses HDR to display range using the film industry's ACES standard | Rich blacks, soft highlights, no clipping — the "movie look" |
| `toneMappingExposure` | Controls how much light enters (like camera exposure) | Too low = dark/muddy, too high = washed out. Start at 1.0-2.0 |
| `SRGBColorSpace` | Applies gamma correction for human eye perception | Without it, colors look wrong — too bright midtones, crushed darks |
| `antialias` | Smooths jagged edges | Free visual upgrade, slight performance cost |

**Tone mapping comparison:**

| Algorithm | Character | Best For |
|-----------|-----------|----------|
| `NoToneMapping` | Raw HDR values, blown-out highlights | Never use in production |
| `LinearToneMapping` | Flat, even exposure | Technical visualization |
| `ReinhardToneMapping` | Soft, slightly washed | Outdoor scenes |
| `ACESFilmicToneMapping` | Rich contrast, cinematic | **Default choice for most scenes** |
| `AgXToneMapping` | Most accurate color preservation | When color fidelity matters most |

### Step 2: Environment Lighting (HDRI)

This is the single biggest visual upgrade. One HDRI map replaces dozens of manual lights and makes everything look real:

```javascript
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';

const rgbeLoader = new RGBELoader();
rgbeLoader.load('/studio.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.background = texture;    // 360° background
  scene.environment = texture;   // lights everything in scene
});
```

**Resolution guidelines:**
- Lighting only (no visible reflections): 256x256 is enough
- Metallic materials with visible reflections: 1024x1024 minimum
- Visible as background: 2048x2048 or higher

**Where to get HDRIs:** [Poly Haven](https://polyhaven.com/) — free, high-quality, CC0 licensed.

**Pro tip:** You can use a low-res HDRI for lighting and a different high-res one for background:
```javascript
scene.environment = lowResHDRI;    // cheap lighting
scene.background = highResHDRI;    // pretty background
```

### Step 3: Materials That React to Light

Default materials look plastic. These settings fix that:

**Realistic metal:**
```javascript
new THREE.MeshStandardMaterial({
  color: 0xcccccc,
  roughness: 0.2,    // low = shiny, high = matte
  metalness: 1.0,    // 1 = metal, 0 = dielectric
  envMapIntensity: 1.0
});
```

**Glass/crystal:**
```javascript
new THREE.MeshPhysicalMaterial({
  roughness: 0,
  metalness: 0,
  transmission: 1,    // 1 = fully transparent
  ior: 1.5,          // glass = 1.5, diamond = 2.42, water = 1.33
  thickness: 0.5     // affects refraction depth
});
```

**Key material principles:**
- `roughness` and `metalness` are the two most important knobs — everything else is secondary
- Real-world materials are either metallic (`metalness: 1`) or non-metallic (`metalness: 0`) — values in between are rare
- `MeshPhysicalMaterial` extends `MeshStandardMaterial` with clearcoat, transmission, sheen — use only when needed (it's more expensive)

### Step 4: Shadows

Shadows ground objects in the scene. Without them, everything floats:

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

const light = new THREE.DirectionalLight(0xffffff, 1);
light.castShadow = true;
light.shadow.mapSize.set(2048, 2048);
light.shadow.bias = -0.0001;         // fixes shadow acne
light.shadow.normalBias = 0.02;      // fixes peter-panning

mesh.castShadow = true;
ground.receiveShadow = true;
```

**Fake contact shadows (cheaper, often better looking):**
A semi-transparent plane with a radial gradient texture underneath objects fakes contact shadows without any shadow map computation. Used by many award-winning sites.

### Step 5: Post-Processing

Post-processing is the film color grading of 3D. It takes a rendered image and enhances it.

**The setup:**
```javascript
import { EffectComposer } from 'three/examples/jsm/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/examples/jsm/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/examples/jsm/postprocessing/OutputPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));       // 1. render scene
composer.addPass(new UnrealBloomPass(                  // 2. add bloom
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.8,    // intensity (0.5-2.0)
  0.5,    // radius (0.5-1.0)
  0.85    // luminance threshold (0.8-1.0)
));
composer.addPass(new OutputPass());                     // 3. color space conversion

// In render loop: composer.render() instead of renderer.render()
```

**The pipeline order matters:**
```
RenderPass → [Effect Passes] → OutputPass
     ↑              ↑               ↑
  Renders 3D    Bloom/SSAO/etc   sRGB + tone map
```

## The Lightsaber Effect: Complete Breakdown

This is what inspired the research — here's exactly how it's done:

### Blade Glow

The blade itself uses **two layers**:

1. **Core:** A cylinder/capsule mesh with bright emissive material (`emissive: 0xffffff, emissiveIntensity: 2`)
2. **Outer glow:** Same geometry scaled 1.2-1.5x with a custom glow shader

**The glow shader trick:**
```javascript
// Vertex shader concept
// Glow intensity = inverse of dot(viewDirection, surfaceNormal)
// Edges face away from camera → high glow
// Center faces camera → low glow
float intensity = pow(1.0 - dot(viewDir, normal), 2.0);
```

This works because edges of a sphere/cylinder have normals perpendicular to the view — so the dot product is low, and `1 - low = high` glow at edges. It's the same reason real neon tubes glow brightest at edges.

### Motion Trail (The Sword Swing)

The **Polyboard technique**:

1. Track the blade tip position every frame → create a polyline (list of 3D points)
2. Expand each point perpendicular to the motion direction → create a triangle strip with width
3. Fade alpha from 1.0 (newest) to 0.0 (oldest) along the strip
4. Apply the same glow shader + bloom post-processing
5. Remove old points each frame to keep trail length constant

**Ready-made solution:** [TrailRendererJS](https://github.com/mkkellogg/TrailRendererJS) — attach motion trails to any Object3D by specifying the trail shape and target.

### Selective Bloom (Only Glow What Should Glow)

The problem: bloom affects everything bright. You want only the lightsaber to glow, not the white floor.

**The Layers technique:**
```
1. Assign glowing objects to Layer 1
2. Bloom composer: render only Layer 1 objects + bloom
3. Main composer: render full scene
4. Merge: composite bloom layer on top of main render
```

During the bloom pass, non-glowing objects are temporarily set to black material so they don't contribute to bloom. Then materials are restored for the final render.

**Alternative:** Set `UnrealBloomPass.threshold` high enough that only emissive materials trigger bloom. Simpler but less control.

## Visual Effect Cookbook

### Volumetric Light / God Rays

The **radial blur technique**:
1. Render light source as white, all occluders as black → occlusion texture
2. Apply radial blur shader from light source position outward
3. Additively blend result over normal scene render

**Cheap fake:** Use `ConeGeometry` with additive blending and low opacity → instant "light shaft" from any point light.

### Atmospheric Fog

```javascript
scene.fog = new THREE.FogExp2(0x000000, 0.02);  // exponential fog
// OR
scene.fog = new THREE.Fog(0x000000, 10, 100);    // linear fog (near, far)
```

Fog adds depth, hides distant LOD transitions, and sets mood. Dark fog = mysterious. Light fog = dreamy.

### Dissolve / Disintegration Effect

Uses a noise texture in a custom shader:
1. Sample noise at each fragment
2. Compare noise value against a `progress` uniform (0→1)
3. If noise < progress, `discard` the fragment
4. At the edge (noise ≈ progress), add bright emissive color + bloom

### Color Grading / LUT

For cinematic color:
- **Split toning:** Tint shadows blue/teal, highlights orange/warm → instant "Hollywood" look
- **S-curve contrast:** Darken darks, brighten brights → richer image
- **3D LUT (.cube file):** Professional color grading from DaVinci Resolve/Photoshop, applied as post-processing

## The 5 Layers of Visual Quality

From "default gray" to "award-winning":

```
Layer 0: Default Three.js
├── Flat lighting, no shadows, wrong color space
├── Looks like a 2005 Flash game
│
Layer 1: Foundation (5 minutes of work)
├── + ACESFilmic tone mapping
├── + sRGB output color space
├── + Antialias
├── Already 3x better
│
Layer 2: Lighting (15 minutes)
├── + HDRI environment map
├── + Shadows (PCFSoft + bias tuning)
├── + Proper material roughness/metalness
├── Looks "real" now
│
Layer 3: Post-Processing (30 minutes)
├── + Bloom (selective, threshold-tuned)
├── + SSAO (subtle ambient occlusion)
├── + Vignette
├── Looks "cinematic"
│
Layer 4: Polish (hours)
├── + Color grading / LUT
├── + Depth of field
├── + Custom shaders (glow, trails, dissolve)
├── + Volumetric effects
├── Award-worthy
│
Layer 5: WebGPU (cutting edge)
├── + Compute shader particles (100K+)
├── + TSL custom materials
├── + Real-time ray tracing hints
├── Next-gen browser 3D
```

## Tradeoffs

| Technique | Visual Impact | Performance Cost | Complexity |
|-----------|--------------|-----------------|------------|
| Tone mapping | High | Zero | 1 line |
| HDRI environment | Very high | Low | 5 lines |
| Shadows | High | Medium | Needs tuning |
| Bloom | High | Medium | Easy with pmndrs |
| SSAO | Medium | Medium-High | Easy with pmndrs |
| Selective bloom | High | High (2 render passes) | Complex |
| Volumetric light | Very high | High | Custom shader |
| DOF | Medium | Medium | Easy with pmndrs |
| Motion trail | High (for action) | Low-Medium | TrailRendererJS |
| Color grading/LUT | Medium | Low | Needs design eye |

## Steal: Actionable Patterns

### 1. The 4-Line Foundation
```javascript
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 1.5;
renderer.outputColorSpace = THREE.SRGBColorSpace;
renderer.shadowMap.enabled = true;
```
Do this first. Always. Non-negotiable.

### 2. HDRI = Instant Realism
One `.hdr` file replaces a dozen lights. Use Poly Haven. Set as both `scene.background` and `scene.environment`.

### 3. The Lightsaber Stack
`Emissive core mesh + scaled glow shader mesh + bloom post-processing + motion trail (Polyboard)` = complete weapon glow effect.

### 4. Selective Bloom via Layers
Assign emissive objects to a separate Layer. Render bloom pass on that layer only. Composite over main scene. This is the standard technique used by every glowing-object demo.

### 5. The pmndrs Post-Processing Stack
Don't build your own EffectComposer chain — use `@react-three/postprocessing` or `pmndrs/postprocessing`. They merge effects into fewer passes for better performance.

### 6. Fake It Before You Make It
- Fake contact shadows (gradient plane) > real shadow maps for simple scenes
- Fake glow (scaled mesh + glow shader) > full bloom pass for single objects
- Fake volumetric light (cone + additive blend) > real god rays for most cases
- Baked lightmaps > real-time lights for static scenes

The best-looking Three.js scenes are usually the ones that fake the most things.
