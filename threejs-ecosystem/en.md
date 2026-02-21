# Three.js Ecosystem: From Demo Toy to Production 3D Platform

## What

Three.js is a JavaScript library that abstracts WebGL (and now WebGPU) into a scene-graph API for 3D rendering in the browser. In 2025-2026, it crossed a critical threshold: WebGPU went production-ready across all major browsers, performance improved 10-150x for heavy workloads, and the ecosystem matured into a full stack for shipping real products — not just demos.

**By the numbers (2026):**
- 2.7M weekly NPM downloads (270x more than nearest competitor)
- 93K+ GitHub stars with daily contributions
- WebGPU support in Chrome, Edge, Firefox, Safari (including iOS)
- Production-ready WebGPU since r171 (September 2025)

## Why This Matters

The web was the last platform without serious 3D capabilities. Native apps had Unity/Unreal, but browsers were stuck with WebGL's 2011-era API. Three things changed:

1. **WebGPU landed everywhere** — Safari 26 (Sep 2025) completed browser coverage. No more "works on Chrome only."
2. **Compute shaders unlocked GPU general-purpose computing** — physics, ML inference, particle systems on the GPU, not CPU.
3. **TSL (Three Shading Language) eliminated the shader pain** — write shaders in JavaScript, compile to GLSL or WGSL automatically.

The result: browser 3D went from "cool but limited" to "competitive with native for most use cases."

## Architecture Deep Dive

### Core: Scene Graph + Object3D Hierarchy

```
Scene (root)
├── Group
│   ├── Mesh (geometry + material)
│   ├── Mesh
│   └── Light
├── Camera
└── Group
    ├── Mesh
    └── InstancedMesh (thousands of copies, one draw call)
```

**Object3D** is the base class for everything. Each object maintains:
- Local transform: `position` (Vector3), `rotation` (Euler), `scale` (Vector3)
- World matrix: computed by multiplying parent transforms down the hierarchy
- Normal matrix: derived from world matrix for lighting calculations

**Key design patterns:**
- **Lazy initialization** — shaders and GPU buffers created on first use, not construction
- **Shader caching** — identical materials share compiled shader programs
- **needsUpdate flags** — track which GPU resources need re-upload, skip the rest
- **Composition over inheritance** — materials compose textures and shader params, not deep class trees

### Rendering Pipeline

```
1. Update matrices     → traverse scene graph, compute world transforms
2. Frustum culling     → test bounding volumes against camera view
3. Collect renderables → gather geometry + material + transform data
4. Compile shaders     → generate GLSL/WGSL from material properties (cached)
5. Upload buffers      → send geometry to GPU via WebGL/WebGPU
6. Bind state          → configure depth testing, blending, stencil
7. Draw calls          → execute indexed/non-indexed drawing
```

The renderer maintains GPU caches for shaders, textures, and geometry — avoiding redundant uploads and compilations across frames.

### WebGPU Integration (The Big Upgrade)

**Before (WebGL):**
- Single-threaded command submission
- No compute shaders
- Implicit state management (error-prone)
- GLSL only

**After (WebGPU):**
- Multi-threaded command preparation
- Compute shaders for general GPU work
- Explicit resource management
- WGSL + automatic fallback to WebGL 2

**Usage is a one-line swap:**
```javascript
// Old
const renderer = new THREE.WebGLRenderer();

// New — falls back to WebGL 2 on old browsers automatically
import { WebGPURenderer } from 'three/webgpu';
const renderer = new WebGPURenderer();
```

**Performance numbers:**

| Metric | WebGL | WebGPU | Improvement |
|--------|-------|--------|-------------|
| 10K particles update | 30ms | — | baseline |
| 100K particles update | impossible | <2ms | **150x** |
| Draw-call-heavy scenes | baseline | 10x faster | Render Bundles |
| One real-world migration | baseline | 100x | platform-specific |

### TSL: Three.js Shading Language

TSL is the most architecturally interesting part. It's a **node-graph shader system written in JavaScript** that compiles to both GLSL and WGSL.

**How it works:**

```javascript
// You write JavaScript — NOT shader strings
const material = new THREE.MeshStandardNodeMaterial();

material.colorNode = texture(map).mul(color(0xff0000));
material.roughnessNode = float(0.5);
material.emissiveNode = color(0x00ffff)
  .mul(sin(time))
  .mul(normalView.dot(positionView).abs());
```

**Under the hood:**

1. Each function call (`sin()`, `mul()`, `texture()`) creates a **Node object** — not an immediate computation
2. Nodes form an **abstract syntax tree** describing data flow
3. **NodeBuilder** analyzes the graph, optimizes it (dead code elimination, variable reuse, uniform deduplication)
4. **WGSLNodeBuilder** or **GLSLNodeBuilder** generates target shader code
5. Same JavaScript → different GPU languages, zero developer intervention

**Why this is a big deal:**

| Aspect | Old (ShaderMaterial) | New (TSL) |
|--------|---------------------|-----------|
| Language | GLSL strings in JS | JavaScript functions |
| IDE support | None (it's a string) | Full autocomplete, refactoring |
| Error messages | Cryptic GPU errors | JavaScript stack traces |
| Lighting/shadows | Rebuild from scratch | Automatic — extends standard materials |
| Cross-platform | GLSL only | Auto-compiles to GLSL or WGSL |
| Composability | Copy-paste strings | Import/export JS modules |

**Vertex displacement example:**
```javascript
material.positionNode = Fn(() => {
  const pos = positionLocal;
  const norm = normalLocal;
  const displacement = sin(time.mul(3.0).add(pos.y.mul(5.0))).mul(0.075);
  return pos.add(norm.mul(displacement));
})();
```

## The Ecosystem Stack

### Core Layer

| Library | Role | Size |
|---------|------|------|
| **three** | Scene graph, renderer, materials | ~600KB min |
| **WebGPURenderer** | WebGPU backend (built-in since r171) | included |
| **TSL** | Node-based shader system | included |

### React Layer (pmndrs ecosystem)

| Library | Role |
|---------|------|
| **react-three-fiber (R3F)** | React renderer for Three.js — declarative JSX scene graph |
| **@react-three/drei** | ~100 ready-made helpers (OrbitControls, Text, Environment, etc.) |
| **@react-three/postprocessing** | Bloom, SSAO, DOF, chromatic aberration — performance-optimized |
| **@react-three/rapier** | Rapier physics engine bindings |
| **leva** | GUI controls for tweaking parameters |
| **zustand** | State management (same team) |

### When to use R3F vs vanilla Three.js

| Scenario | Choice | Why |
|----------|--------|-----|
| React app adding 3D | R3F | Integrates with React lifecycle, state, routing |
| Custom rendering pipeline | Vanilla | Full control, no abstraction overhead |
| Game with complex shaders | Vanilla | Direct WebGPU access, custom render loops |
| Product configurator / dashboard | R3F | Declarative, component-based, fast to prototype |
| Maximum performance | Vanilla | Zero React reconciliation overhead |

## What People Are Actually Building

### Browser Games (Including MMOs)

| Project | Type | Notable |
|---------|------|---------|
| stein.world | Browser MMORPG | "World's first full-fledged real-time browser MMORPG" |
| mal-war.com | MMORPG | Entirely Three.js |
| Ironbane | Browser MMO | Three.js + Node.js + GLSL |
| Star Defenders 3D | Multiplayer FPS | Destructible voxel worlds |
| riseintime.com | Strategy multiplayer | Three.js |

### Award-Winning Interactive Experiences

| Creator | What | Award |
|---------|------|-------|
| Samsy | Cyberpunk world, WebGPU, 120+ FPS, first-person | Portfolio of the Year contender |
| Jordan Breton | Floating island with grass/waterfall/fire/butterflies | FWA Site of the Day (Oct 2025) |
| JReyes MC | Minecraft-style 3D portfolio | Awwwards Honorable Mention |
| Bruno Simon | Drive a car through 3D world | Classic Three.js showcase |
| WoraWork | Zelda/Animal Crossing style explorable world | Community favorite |

### E-Commerce & Product Configurators

- **IKEA** — 3D + AR furniture visualization in your space
- **Lolo Chateney** — luxury bag 3D configurator
- **Thömus** — 3D bike configurator
- **Eyewear retailers** — AR try-on in browser

### Creative Coding & Generative Art

- **Genuary 2026** — annual generative art challenge, Three.js is primary tool
- Procedural visual effects with custom GLSL shaders (cursor trails, bloom, film grain)
- Interactive particle simulations (100K+ particles with WebGPU compute)
- InstancedMesh + GLSL for thousands of physics-reactive objects

## Tradeoffs

### What Three.js Does Well

- **Ecosystem dominance** — 270x more downloads than competitors, massive community
- **Flexibility** — it's a library, not a framework. Compose what you need
- **WebGPU-first** — production-ready WebGPU with automatic WebGL fallback
- **TSL** — the best shader authoring experience on the web
- **React integration** — R3F is the best declarative 3D framework that exists

### What Three.js Does Poorly

| Weakness | Detail |
|----------|--------|
| **Not a game engine** | No built-in physics, animation state machines, ECS, or audio system — need third-party libs |
| **Shadow system is rigid** | Single `scene.shadowMap` — no per-object shadow generators like Babylon.js |
| **Post-processing requires add-ons** | No built-in bloom, SSAO, DOF — need pmndrs/postprocessing |
| **Learning curve** | Requires understanding 3D graphics concepts (matrices, shaders, GPU pipeline) |
| **Bundle size** | ~600KB min for core. Babylon.js is 2MB+ but includes everything |
| **No visual editor** | No built-in scene editor. Babylon.js has Playground, PlayCanvas has full cloud editor |

### vs. Alternatives

| Feature | Three.js | Babylon.js | PlayCanvas |
|---------|----------|------------|------------|
| Philosophy | Library (compose) | Engine (batteries included) | Engine + cloud editor |
| Bundle size | ~600KB | ~2MB+ | ~150KB engine |
| Built-in physics | No (use Rapier/Cannon) | Yes (Havok) | Yes (ammo.js) |
| Visual editor | No | Playground/Inspector | Full cloud IDE |
| WebGPU | Production (r171+) | Production | In progress |
| Community size | Largest (93K stars) | Large (23K stars) | Smaller (9K stars) |
| React integration | Excellent (R3F) | Limited | None |
| NPM downloads | 2.7M/week | ~100K/week | ~10K/week |
| Best for | Creative web, configurators, React apps | Complex games, enterprise | Team collaboration, rapid prototyping |

**Key insight:** Three.js wins on ecosystem and flexibility. Babylon.js wins on "batteries included" for games. PlayCanvas wins on team workflow with its cloud editor. Choose based on what you're building, not synthetic benchmarks.

## Production Performance Patterns

### Must-Do Optimizations

1. **Share materials and geometries** — every unique material compiles a shader, every unique geometry uploads buffers
2. **Use InstancedMesh** — 10,000 trees? One draw call, not 10,000
3. **Use glTF + Draco compression** — reduces mesh size to <10% of OBJ/COLLADA
4. **Shrink camera frustum** — smaller near/far range = better depth precision and culling
5. **Only render when needed** — static scenes don't need 60fps render loops
6. **Use `powerPreference: "high-performance"`** — hints browser to use discrete GPU on laptops

### Common Pitfalls

| Mistake | Fix |
|---------|-----|
| Creating objects in loops | Reuse with `vector.set()` |
| Enabling `preserveDrawingBuffer` | Only enable if you need canvas screenshots |
| New Material per mesh | Share materials across similar meshes |
| OBJ/COLLADA format | Switch to glTF + Draco |
| Rendering every frame for static scenes | Render on demand (camera move, animation) |
| Huge textures | Use KTX2/basis compressed textures |

## Latest Updates (2026)

### Release Cadence: r170 through r183

Three.js maintained its aggressive monthly release cycle throughout 2025 into early 2026. Key milestone releases:

- **r170** (October 2024) — Halloween release with continued WebGPU stabilization.
- **r171** (September 2025) — The landmark release: global codesplit WebGL/WebGPU entrypoints, introduction of `three.tsl.js`, zero-config `import { WebGPURenderer } from 'three/webgpu'`.
- **r175** (March 2025) — Enhanced TSL with compute-in-materials, `RaymarchingBox`, `debug()` function, while loop support, and improved WebXR Layers integration.
- **r177** (mid-2025) — Added `maskNode`, `toJSON()`/`fromJSON()` for scene serialization, improved `shapeCircle()`.
- **r178** (June 2025) — 112 goals completed. Added Float16Array support in renderers, corrected blending formulas across all backends.
- **r181** (November 2025) — Major rendering quality leap: **Screen-Space Global Illumination (SSGINode)**, **Screen-Space Shadows (SSSNode)**, DFG LUT replacing analytical approximations, GGX VNDF importance sampling, multi-scattering energy compensation, WebGL 1.0 code removal, and WebGPU Inspector for debugging.
- **r182** (December 2025) — ESLint 9 migration, modernized shadow mapping, Multiple Render Target (MRT) support in WebGPU.
- **r183** (February 2026) — `BezierInterpolant` for animation, reversed depth buffer across both renderers, `BatchedMesh` per-instance opacity and wireframe support, USDC/USD format loading, `KHR_meshopt_compression` in GLTFLoader, TSL `retroPass` and `exponentialHeightFogFactor()`, and deprecated `Clock` in favor of `Timer`.

### GitHub & NPM Growth

Three.js has surged past **110K GitHub stars** (up from 93K in mid-2025) and now records **5.4M+ weekly NPM downloads** — more than doubling from 2.7M in under a year. The ecosystem dominance is even more pronounced: roughly **500x** the download volume of Babylon.js.

### Vibe Coding Revolution

"Vibe coding" — using natural language prompts with AI tools (Claude, ChatGPT, Cursor) to generate code — was named Collins Dictionary's Word of the Year 2025. Three.js became the de facto library for this workflow because:

1. **Simple API** — scene setup is ~10 lines, ideal for LLM generation.
2. **Immediate visual feedback** — results render in-browser instantly.
3. **Massive training corpus** — 15+ years of examples, tutorials, and StackOverflow answers.
4. **No engine install** — runs from a CDN or `npm install three`.

Indie developers reported building complete 3D browser games in days using AI pair-programming. Rosebud AI released a series of fully vibe-coded Three.js games, and multiple developers have reached significant revenue from AI-generated game prototypes.

### WebGPU: From Experimental to Default

Since Safari 26 shipped WebGPU support in September 2025 (macOS, iOS, iPadOS, visionOS), WebGPU is available on **100% of modern browsers**. Three.js now treats WebGPU as the primary path:

- `WebGPURenderer` auto-falls back to WebGL 2 — no feature detection needed.
- Async initialization via `await renderer.init()` before first render is the only new requirement.
- Compute shaders unlocked GPU-side physics, particle systems, and ML inference in the browser.
- Real-world migrations report **2-10x** improvements for draw-call-heavy scenes, with edge cases hitting **100x** on million-point datasets.

### Advanced Rendering Features (Built-In)

The r181-r183 cycle brought rendering capabilities previously only available via post-processing add-ons directly into core:

| Feature | Release | Impact |
|---------|---------|--------|
| SSGINode (Screen-Space GI) | r181 | Real-time color bleeding and light bounces without baked maps |
| SSSNode (Screen-Space Shadows) | r181 | Per-pixel contact shadows, no shadow map required |
| Reversed Depth Buffer | r183 | Eliminates z-fighting at large distances (space, terrain) |
| MRT (Multiple Render Targets) | r182 | Write to multiple textures in one pass — enables deferred rendering |
| BatchedMesh opacity/wireframe | r183 | Per-instance visual control for complex scenes |

### TSL Maturation

TSL has evolved from experimental to the recommended shader authoring path. Key 2025-2026 additions:

- **Compute-in-materials** (r175) — run compute operations directly from material nodes.
- **RaymarchingBox / raymarchingTexture3D** (r174) — volumetric rendering via TSL.
- **TSL Transpiler expansion** — matrix types, varying support, discard operations.
- **`debug()` function** (r175) — inspect intermediate node values during development.
- **`retroPass`** (r183) — retro/CRT visual effect as a built-in post-processing node.
- **`exponentialHeightFogFactor()`** (r183) — height-based fog without custom shaders.

### WebXR & Spatial Computing

Three.js WebXR support matured alongside new hardware:

- **Samsung Galaxy XR** (October 2025) launched with native WebXR support.
- **Apple Vision Pro** continues to support WebXR through Safari.
- r175 added deeper **WebXR Layers** integration for better performance in VR/AR overlays.
- r174 added tone-mapping and output color space support for XR rendering.
- Use cases expanding into VR surgical training, real estate virtual tours, and in-browser AR product try-on.

### React Three Fiber: Stability Phase

R3F entered a period of stability in late 2025 — no major breaking changes, focused on reliability. Key ecosystem developments:

- **@react-three/drei** reached v10.7+ with 100+ helper components.
- **Threlte** (Svelte + Three.js) gained traction as the Svelte equivalent of R3F, working toward Svelte 5 compatibility.
- R3F is evolving beyond "website 3D" toward supporting complex real-time apps, live simulations, and browser-native video games.
- Server-side rendering of 3D scenes via React Server Components and Next.js integration is now possible, improving SEO and initial load performance.

## Steal: Patterns to Reuse

### 1. WebGPU with Automatic Fallback
```javascript
import { WebGPURenderer } from 'three/webgpu';
// Zero config — falls back to WebGL 2 on older browsers
```
**Pattern:** Always target the better API, degrade gracefully. No feature detection needed.

### 2. TSL for Cross-Platform Shaders
Write shader logic once in JavaScript, compile to whatever the browser supports. This is the right abstraction level — not raw GLSL strings, not a visual node editor, but typed JavaScript functions.

### 3. InstancedMesh for Mass Objects
One geometry + one material + instance transforms = one draw call for thousands of objects. This pattern applies to any rendering system, not just Three.js.

### 4. R3F's Declarative 3D Pattern
```jsx
<Canvas>
  <mesh position={[0, 1, 0]}>
    <boxGeometry />
    <meshStandardMaterial color="orange" />
  </mesh>
</Canvas>
```
**Pattern:** Treat 3D scenes as component trees. State drives rendering. React reconciler handles updates. This is the future of 3D UI development.

### 5. Demand-Based Rendering
Don't render at 60fps if nothing is moving. Listen for interaction events, animate only when needed. This alone can cut GPU power consumption by 90% for semi-static scenes.

### 6. The pmndrs Stack
`R3F + drei + rapier + postprocessing + zustand + leva` = complete 3D application stack with zero custom infrastructure. This pattern of "curated ecosystem of composable libraries" beats "one monolithic engine" for web development.

## References

### Official

- [Three.js Official Site & Examples](https://threejs.org/examples/)
- [Three.js WebGPU Docs](https://threejs.org/docs/pages/WebGPU.html)
- [Three.js Shading Language (TSL) Wiki](https://github.com/mrdoob/three.js/wiki/Three.js-Shading-Language)
- [React Three Fiber (R3F) GitHub](https://github.com/pmndrs/react-three-fiber)
- [R3F Documentation](https://docs.pmnd.rs/react-three-fiber)
- [pmndrs/postprocessing GitHub](https://github.com/pmndrs/postprocessing)
- [react-three-rapier GitHub](https://github.com/pmndrs/react-three-rapier)

### WebGPU & TSL Deep Dives

- [What's New in Three.js (2026): WebGPU, New Workflows & Beyond](https://www.utsubo.com/blog/threejs-2026-what-changed)
- [Field Guide to TSL and WebGPU — Maxime Heckel](https://blog.maximeheckel.com/posts/field-guide-to-tsl-and-webgpu/)
- [TSL: A Better Way to Write Shaders — Three.js Roadmap](https://threejsroadmap.com/blog/tsl-a-better-way-to-write-shaders-in-threejs)
- [Why TSL is So Interesting — Three.js Forum](https://discourse.threejs.org/t/why-tsl-three-js-shading-language-is-so-interesting/56306)
- [WebGL vs WebGPU Explained — Three.js Roadmap](https://threejsroadmap.com/blog/webgl-vs-webgpu-explained)
- [Moving from WebGL to WebGPU in Three.js — Sude Nur Çevik](https://medium.com/@sudenurcevik/upgrading-performance-moving-from-webgl-to-webgpu-in-three-js-4356e84e4702)
- [Three.js Introduction to WebGPU and TSL — Forum](https://discourse.threejs.org/t/three-js-introduction-to-webgpu-and-tsl/78205)
- [GPU Acceleration in Browsers: WebGPU Benchmarks](https://www.mayhemcode.com/2025/12/gpu-acceleration-in-browsers-webgpu.html)

### Architecture & Internals

- [Scene Graph & Object System — DeepWiki](https://deepwiki.com/mrdoob/three.js/2.3-scene-graph-and-object-system)
- [How THREE.js Abstracts 3D Graphics — Evan Perry](https://medium.com/@evmaperry/how-three-js-abstracts-away-the-complexities-of-programming-3d-graphics-c3b74dbf051c)
- [Three.js Renderer Docs](https://threejs.org/docs/pages/Renderer.html)
- [R3F Performance Pitfalls](https://r3f.docs.pmnd.rs/advanced/pitfalls)

### Comparisons & Alternatives

- [React Three Fiber vs Three.js in 2026 — GraffersID](https://graffersid.com/react-three-fiber-vs-three-js/)
- [Three.js vs Babylon.js — LogRocket](https://blog.logrocket.com/three-js-vs-babylon-js/)
- [Why We Use Babylon.js Instead of Three.js — Spot Virtual](https://www.spotvirtual.com/blog/why-we-use-babylonjs-instead-of-threejs-in-2022)
- [Babylon.js vs Three.js — Slant](https://www.slant.co/versus/11077/11348/~babylon-js_vs_three-js)
- [PlayCanvas vs Three.js — Slant](https://www.slant.co/versus/5149/11348/~playcanvas_vs_three-js)
- [Overview of 3D Web Rendering Engines — VIVERSE](https://docs.viverse.com/optimization/overview-of-3d-web-rendering-engines)
- [JS Game Rendering Benchmark — GitHub](https://github.com/Shirajuki/js-game-rendering-benchmark)

### Showcases & Projects

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

### Performance & Optimization

- [Building Efficient Three.js Scenes — Codrops](https://tympanus.net/codrops/2025/02/11/building-efficient-three-js-scenes-optimize-performance-while-maintaining-quality/)
- [The Big List of Three.js Tips and Tricks — Discover Three.js](https://discoverthreejs.com/tips-and-tricks/)
- [Performance Tips — Three.js Journey](https://threejs-journey.com/lessons/performance-tips)
- [Three.js vs WebGPU for 500MB+ Models — AlterSquare](https://altersquare.io/three-js-vs-webgpu-construction-3d-viewers-scale-beyond-500mb/)
- [100 Three.js Tips That Actually Improve Performance (2026)](https://www.utsubo.com/blog/threejs-best-practices-100-tips)

### Tutorials & Learning

- [Three.js Journey (Paid Course)](https://threejs-journey.com/)
- [Discover Three.js (Free Book)](https://discoverthreejs.com/)
- [Wawa Sensei — R3F + WebGPU/TSL Course](https://wawasensei.dev/courses/react-three-fiber/lessons/webgpu-tsl)
- [Three.js WebGPU Renderer Tutorial — SBCode](https://sbcode.net/threejs/webgpu-renderer/)
- [TSL Getting Started — SBCode](https://sbcode.net/tsl/getting-started/)
- [From Websites to Games: Future of R3F — Kris Baumgartner](https://gitnation.com/contents/from-websites-to-games-the-future-of-react-three-fiber)
- [Building 3D Web Apps in 2025 — Bela Bohlender](https://gitnation.com/contents/building-3d-web-apps-in-2025-react-xr-and-ai)

### Release Notes (2025-2026)

- [Three.js Releases — GitHub](https://github.com/mrdoob/three.js/releases)
- [Release r175 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r175)
- [Release r178 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r178)
- [Release r181 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r181)
- [Release r183 — GitHub](https://github.com/mrdoob/three.js/releases/tag/r183)
- [Migration Guide — GitHub Wiki](https://github.com/mrdoob/three.js/wiki/Migration-Guide)
- [Migrate Three.js to WebGPU (2026) — Complete Checklist](https://www.utsubo.com/blog/webgpu-threejs-migration-guide)

### Vibe Coding & AI Integration

- [5 Three.js Game Examples Vibe Coded with AI — Rosebud AI](https://lab.rosebud.ai/blog/three-js-game-examples-vibe-coded)
- [Ultimate Guide to Vibe Coding Games with Three.js and Cursor](https://webtech.tools/the-ultimate-guide-to-vibe-coding-games-in-2025)
- [Getting AI to Write TSL That Works — Three.js Roadmap](https://threejsroadmap.com/blog/getting-ai-to-write-tsl-that-works)
- [Three.js MCP Server — AI Control of 3D Scenes](https://skywork.ai/skypage/en/3d-worlds-ai-threejs-mcp-server/1980470680631943168)

### WebXR & Spatial Computing

- [Top VR/AR/XR Use Cases 2026 — Three.js Resources](https://threejsresources.com/vr/blog/top-vr-ar-xr-use-cases-in-2026-building-immersive-experiences-that-deliver-real-value)
- [Best VR Headsets for WebXR & Three.js Development 2026](https://threejsresources.com/vr/blog/best-vr-headsets-with-webxr-support-for-three-js-developers-2026)
- [SSGI WebGPU Demo — Anderson Mancini](https://ssgi-webgpu-demo.vercel.app/)
- [Interactive Text Destruction with Three.js, WebGPU, and TSL — Codrops](https://tympanus.net/codrops/2025/07/22/interactive-text-destruction-with-three-js-webgpu-and-tsl/)

### Ecosystem & Frameworks

- [Threlte — 3D Framework for Svelte](https://threlte.xyz/)
- [React Three Fiber: Stability in Late 2025 — Needle Radio](https://needle.tv/episode/react-three-fiber-a-quiet-period-of-stability-in-late-2025-852)
- [Drei Documentation](https://drei.docs.pmnd.rs/)
- [Three.js NPM Stats](https://www.npmjs.com/package/three)
- [Three.js NPM Trends](https://npmtrends.com/three)
