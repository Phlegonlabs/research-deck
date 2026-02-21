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

## Latest Updates (2026)

### WebGPU Is Now Production-Ready Everywhere

The most significant shift in Three.js visual quality tooling since 2025 is **universal WebGPU browser support**. In September 2025, Apple shipped WebGPU in Safari 26 (macOS, iOS, iPadOS, visionOS) — the last major holdout. Combined with Chrome/Edge (since v113, May 2023) and Firefox (v139+, June 2025), you can now ship WebGPU to every user with automatic WebGL 2 fallback. Since Three.js r171, a single import line is all it takes:

```javascript
import * as THREE from 'three/webgpu';
// Automatically falls back to WebGL 2 on older browsers
```

Performance gains of **2-10x** are common for draw-call-heavy scenes, with some data visualization workloads seeing up to 100x improvement. This directly impacts visual quality: more GPU budget means more post-processing passes, higher shadow map resolutions, and denser particle systems without frame drops.

### TSL (Three Shader Language) Replaces Raw GLSL

Three.js introduced **TSL** — a JavaScript-based, node-based shader authoring system that compiles to both WGSL (WebGPU) and GLSL (WebGL) from a single codebase. Instead of writing shader strings, you compose typed JavaScript functions:

```javascript
import { uniform, sin, positionLocal } from 'three/tsl';

const time = uniform(0);
const material = new THREE.MeshStandardNodeMaterial();
material.colorNode = sin(time).mul(0.5).add(0.5);
material.positionNode = positionLocal.add(sin(time).mul(0.1));
```

TSL eliminates the GLSL/WGSL split entirely. The compiler handles temporary variable optimization, uniform reuse, and dead code elimination. For visual effects work, this means custom glow shaders, dissolve effects, and procedural materials can be written once and run on both renderers. A growing library of [TSL textures](https://github.com/boytchev/tsl-textures) provides ready-made procedural noise, patterns, and effects as composable nodes.

### Native WebGPU Post-Processing with TSL Nodes

For WebGPU projects, Three.js now offers **built-in post-processing via TSL nodes** — replacing the need for pmndrs/postprocessing in WebGPU contexts. The native solution supports compute shaders for effects like ambient occlusion, bloom, denoising, and depth of field, all defined in a few lines of TSL. The pmndrs/postprocessing library remains the best choice for WebGL-only projects, but for new WebGPU work, the native TSL pipeline is the recommended path.

### 3D Gaussian Splatting Integration

**Gaussian splatting** — rendering photorealistic scenes from point-cloud captures — has moved from research curiosity to production-ready Three.js integration. [Spark](https://sparkjs.dev/) is the actively developed renderer, supporting multiple file formats, integration with standard Three.js meshes, and fast rendering on all devices. It enables mixing traditional 3D geometry with photogrammetry-captured environments, opening up hybrid real/synthetic visual design. 2025 was the year 3DGS moved from research to production; 2026 is the inflection point where it becomes a standard tool.

### React Three Fiber v9 + WebGPU

React Three Fiber (R3F) now supports WebGPU through an **async `gl` prop factory**. The `Canvas` component accepts an async function that creates and initializes a `WebGPURenderer`, making the React ecosystem fully compatible with the new rendering pipeline. Combined with TSL node materials, R3F projects can now access compute shaders, native post-processing, and all WebGPU performance benefits without leaving the React paradigm.

### "Vibe Coding" and AI-Assisted 3D

In February 2025, Andrej Karpathy coined the term **"vibe coding"** — using AI tools to generate code by describing what you want. Three.js turned out to be the ideal library for this workflow: simple setup (just JavaScript, no servers), immediate visual feedback, and a massive training corpus. "Vibe coding" became Collins Dictionary's Word of the Year 2025, and Three.js is the library of choice for AI-assisted 3D development. This has dramatically lowered the barrier to creating visually impressive 3D web experiences.

### Three.js Ecosystem Scale

Three.js now exceeds **2.7 million weekly NPM downloads** — 270x higher than competing 3D web libraries. The ecosystem has expanded beyond websites into museum installations, retail displays, data visualization dashboards, and physical interactive venues. The library's dominance means visual techniques developed for Three.js are effectively the standard for 3D on the web.

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

## References

### Lightsaber & Weapon Effects

- [WebGL Lightsaber — Polyboard trail technique breakdown](https://glampert.com/2015/06-07/webgl-lightsaber/)
- [Slow-Motion Sword Trail Effect in Three.js — 80.lv](https://80.lv/articles/slow-motion-sword-trail-effect-recreated-with-three-js)
- [TrailRendererJS — 3D motion trail library for Three.js](https://github.com/mkkellogg/TrailRendererJS)
- [Star Wars Blaster Demo — Three.js Forum](https://discourse.threejs.org/t/blaster-demo-inspired-by-star-wars-blaster/14190)
- [X-Wing Star Wars WebGL Game](https://amilajack.com/xwing/)
- [My Star Wars Game — Three.js Forum](https://discourse.threejs.org/t/my-starwars-game/61887)

### Glow & Bloom

- [Selective Unreal Bloom — Wael Yasmina](https://waelyasmina.net/articles/unreal-bloom-selective-threejs-post-processing/)
- [Three.js Official Selective Bloom Example](https://threejs.org/examples/webgl_postprocessing_unreal_bloom_selective.html)
- [UnrealBloomPass Docs](https://threejs.org/docs/pages/UnrealBloomPass.html)
- [SelectiveUnrealBloomPass Library — GitHub](https://github.com/VisualSource/selectiveUnrealBloomPass)
- [Fake Glow Material for Three.js — GitHub](https://github.com/ektogamat/fake-glow-material-threejs)
- [Geometric Glow Extension — GitHub](https://github.com/jeromeetienne/threex.geometricglow)
- [Shader Glow Demo — Lee Stemkoski](https://stemkoski.github.io/Three.js/Shader-Glow.html)
- [Best Way to Achieve Glow — Three.js Forum](https://discourse.threejs.org/t/whats-the-best-way-to-achieve-a-glow-effect/59724)

### Post-Processing

- [pmndrs/postprocessing Library — GitHub](https://github.com/pmndrs/postprocessing)
- [Post-Processing with Three.js — Wael Yasmina](https://waelyasmina.net/articles/post-processing-with-three-js-the-what-and-how/)
- [Post-Processing Tutorial — Sangil Lee](https://sangillee.com/2025-01-15-post-processing/)
- [SSAO Example — Three.js Official](https://threejs.org/examples/webgl_postprocessing_ssao.html)
- [Bloom — React Postprocessing Docs](https://react-postprocessing.docs.pmnd.rs/effects/bloom)
- [SSAO — React Postprocessing Docs](https://react-postprocessing.docs.pmnd.rs/effects/ssao)
- [Dissolve Effect with Shaders and Particles — Codrops](https://tympanus.net/codrops/2025/02/17/implementing-a-dissolve-effect-with-shaders-and-particles-in-three-js/)

### Lighting & Environment

- [How to Create Ultra-Realistic Scenes — Wael Yasmina](https://waelyasmina.net/articles/how-to-create-ultra-realistic-scenes-in-three.js/)
- [Realistic Render Lesson — Three.js Journey](https://threejs-journey.com/lessons/realistic-render)
- [Environment Map Lesson — Three.js Journey](https://threejs-journey.com/lessons/environment-map)
- [HDR Environment Mapping — Three.js Official Example](https://threejs.org/examples/webgl_materials_envmaps_hdr.html)
- [Live Envmaps for Realistic Studio Lighting — Three.js Forum](https://discourse.threejs.org/t/live-envmaps-and-getting-realistic-studio-lighting-almost-for-free/35627)
- [FastHDR Environment Maps — Needle](https://cloud.needle.tools/articles/fasthdr-environment-maps)
- [HDR Lighting in Three.js — PixelCapture](https://pixel-capture.com/tutorials/hdr-lighting-threejs-article)

### Tone Mapping & Color

- [Tone Mapping Overview — Three.js Forum](https://discourse.threejs.org/t/tone-mapping-overview/75204)
- [Tone Mapping Example — Three.js Official](https://threejs.org/examples/webgl_tonemapping.html)
- [Color Management in Three.js — Don McCurdy](https://www.donmccurdy.com/2020/06/17/color-management-in-threejs/)
- [Color Grading Techniques in Three.js — MoldStud](https://moldstud.com/articles/p-an-in-depth-look-at-color-grading-techniques-in-threejs-post-processing)
- [AgX Tone Mapping Support — GitHub Issue](https://github.com/mrdoob/three.js/issues/27362)

### Volumetric & Atmospheric Effects

- [Volumetric Light Rays — Codrops](https://tympanus.net/codrops/2022/06/27/volumetric-light-rays-with-three-js/)
- [Volumetric Lighting with Raymarching — Maxime Heckel](https://blog.maximeheckel.com/posts/shaping-light-volumetric-lighting-with-post-processing-and-raymarching/)
- [God Rays Post-Processing Tutorial — Red Stapler](https://redstapler.co/godrays-three-js-post-processing-tutorial/)
- [Volumetric Light Example — GitHub](https://github.com/netpraxis/volumetric_light_example)
- [Volumetric Lights with Radial Blur — TheFrontDev](https://www.thefrontdev.co.uk/creating-volumetric-lights-with-radial-blur-in-three.js-using-layers/)
- [God Rays Forum Discussion](https://discourse.threejs.org/t/help-with-persistent-volumetric-light-god-rays-light-shafts-sunbeam-sunburst-for-underground-cave-scene/79085)

### Trails & Particles

- [Particle Trail Effect — Three.js Forum](https://discourse.threejs.org/t/particle-trail-effect/31642)
- [Creating a Mouse Trail — Leanne Werner](https://medium.com/@leannewerner/creating-a-mouse-trail-in-three-js-fb15346ce784)
- [Three Ways to Create 3D Particle Effects — Varun Vachhar](https://varun.ca/three-js-particles/)
- [Interactive Particle Simulation — Three.js Forum](https://discourse.threejs.org/t/interactive-particle-simulation/67581)

### General Visual Quality & Best Practices

- [100 Three.js Best Practices (2026) — Utsubo](https://www.utsubo.com/blog/threejs-best-practices-100-tips)
- [Building Efficient Scenes: Quality + Performance — Codrops](https://tympanus.net/codrops/2025/02/11/building-efficient-three-js-scenes-optimize-performance-while-maintaining-quality/)
- [Design Fundamentals with Three.js — Zero to Mastery](https://zerotomastery.io/blog/design-fundamentals-with-threejs/)
- [How Can I Make This Scene More Beautiful — Three.js Forum](https://discourse.threejs.org/t/how-can-i-make-this-game-scene-more-beautiful/56620)
- [Building a Vaporwave Scene — Maxime Heckel](https://blog.maximeheckel.com/posts/vaporwave-3d-scene-with-threejs/)

### Learning Resources

- [Three.js Journey (Paid Course)](https://threejs-journey.com/)
- [Poly Haven — Free HDRIs, Textures, Models](https://polyhaven.com/)
- [Awwwards Three.js Websites — Inspiration](https://www.awwwards.com/websites/three-js/)
- [Codrops Three.js Demos](https://tympanus.net/codrops/hub/tag/three-js/)

### 2026 Updates Sources

- [What Changed in Three.js 2026? WebGPU, Vibe Coding & Beyond — Utsubo](https://www.utsubo.com/blog/threejs-2026-what-changed)
- [Migrate Three.js to WebGPU (2026) — The Complete Checklist — Utsubo](https://www.utsubo.com/blog/webgpu-threejs-migration-guide)
- [Field Guide to TSL and WebGPU — Maxime Heckel](https://blog.maximeheckel.com/posts/field-guide-to-tsl-and-webgpu/)
- [TSL: A Better Way to Write Shaders in Three.js — Three.js Roadmap](https://threejsroadmap.com/blog/tsl-a-better-way-to-write-shaders-in-threejs)
- [Three.js Shading Language Wiki — GitHub](https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language)
- [TSL Textures Collection — GitHub](https://github.com/boytchev/tsl-textures)
- [Spark — Advanced 3D Gaussian Splatting Renderer for Three.js](https://sparkjs.dev/)
- [GaussianSplats3D — Three.js Gaussian Splatting](https://github.com/mkkellogg/GaussianSplats3D)
- [3D Gaussian Splatting Complete Guide (2026) — Utsubo](https://www.utsubo.com/blog/gaussian-splatting-guide)
- [React Three Fiber v9 Migration Guide](https://r3f.docs.pmnd.rs/tutorials/v9-migration-guide)
- [R3F WebGPU Starter — Anderson Mancini](https://github.com/ektogamat/r3f-webgpu-starter)
- [Interactive Text Destruction with Three.js, WebGPU, and TSL — Codrops](https://tympanus.net/codrops/2025/07/22/interactive-text-destruction-with-three-js-webgpu-and-tsl/)
- [Three.js BatchedMesh and Post Processing with WebGPURenderer — Codrops](https://tympanus.net/codrops/2024/10/30/interactive-3d-with-three-js-batchedmesh-and-webgpurenderer/)
- [Three.js Releases — GitHub](https://github.com/mrdoob/three.js/releases)
