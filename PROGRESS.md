# Progress

## [2026-02-07] Parallel Coding Agents — Architecture Guide
- Consolidated research: how to run N coding agents in parallel in the cloud
- Covers Cloudflare (Workers + Containers + Sandbox SDK), Warp (Namespace + Ambient Agents), E2B, Daytona, Modal, Sprites, Northflank, GitHub Codespaces
- Universal pattern: Trigger → Orchestrate → Execute (N sandboxes) → Observe
- Key insight: sandbox layer is commoditizing (Rivet SDK), value moves to orchestration and intent specification
- Warp's "Workers" = generic concept (Namespace containers), NOT Cloudflare Workers
- Cloudflare CAN run parallel agents via Containers + Sandbox SDK (official Claude Code tutorial exists)
- 5 architecture patterns: ephemeral-per-task, fork-and-explore, persistent workstation, Cloudflare stack, Warp ambient
- 8 stealable patterns: orchestrator≠executor, sidecar volumes, fork-snapshot, shared cache, scoped credentials, progressive autonomy, friction-point activation, TOEO
- Decision framework with cost analysis (4 agents × 8hr/day × 20 days: $11-$115/mo depending on platform)
- Files: `parallel-coding-agents/`
- Next steps: prototype parallel agent system with Cloudflare Sandbox SDK or Daytona

## [2026-02-07] Three.js Visual Design
- Deep analysis of how to make Three.js scenes look cinematic — from lightsaber glow to photorealistic rendering
- Key insight: 80% of visual quality from 5 things — tone mapping, HDRI environment, bloom, color space, shadows
- Lightsaber effect breakdown: emissive core + glow shader + Polyboard trail + selective bloom via Layers
- Complete "cinematic recipe": 4-line renderer setup → HDRI → materials → shadows → post-processing
- Visual effects cookbook: volumetric light (radial blur), dissolve (noise discard), god rays, fog, color grading/LUT
- 5 layers of visual quality: Default → Foundation (5min) → Lighting (15min) → Post-processing (30min) → Polish (hours) → WebGPU
- Files: `threejs-visual-design/`
- Next steps: build a lightsaber demo to validate the full stack

## [2026-02-07] Three.js Ecosystem
- Deep analysis of Three.js in 2025-2026: from demo toy to production 3D platform
- Key findings: WebGPU production-ready (r171+, all browsers), 10-150x performance gains, TSL node-graph shader system
- Architecture: scene graph + Object3D hierarchy, lazy init, shader caching, needsUpdate flags
- TSL deep dive: JavaScript → Node AST → GLSL/WGSL compilation, automatic cross-platform shaders
- Ecosystem: R3F + drei + rapier + postprocessing + zustand + leva = complete 3D app stack
- Compared with Babylon.js (batteries-included engine) and PlayCanvas (cloud editor)
- Real-world: browser MMORPGs, award-winning portfolios, IKEA-level product configurators, generative art
- Files: `threejs-ecosystem/`
- Next steps: build something with R3F + WebGPU to validate patterns

## [2026-02-07] VoxYZ Autonomous AI Company
- Researched how VoxYZ built 6 autonomous AI agents with OpenClaw + Vercel + Supabase
- Key patterns: closed-loop orchestration, gate-before-queue, trigger/reaction matrix, self-healing heartbeat
- Files: `docs/voxyz-autonomous-agents/`
- Next steps: explore more agentic coding patterns

## [2026-02-07] Claude Code Agent Team
- Researched Claude Code's native multi-agent orchestration system (Agent Teams / Swarm Mode)
- Source: @YukerX Twitter thread + official docs + multiple deep-dive articles
- Key patterns: parallel specialists, competing hypotheses, self-organizing swarm, plan-approve-execute, file ownership boundaries
- Architecture: TeammateTool (13 ops), file-based coordination (JSON on disk), task dependency auto-unblocking, hook-based quality gates
- Compared with LangGraph, CrewAI, OpenAI Swarm, claude-flow
- Files: `claude-code-agent-team/`
- Next steps: try Agent Team on a real multi-file refactoring task
