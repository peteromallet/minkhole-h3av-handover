# Brief: Make H3_AV Editing Repeatable, Reliable, and Easy to Delegate

## Handover files and starting point

This brief and two portable Astrid timeline archives are available in the [public handover repository](https://github.com/peteromallet/minkhole-h3av-handover). Download the archives from its [timeline-v1 release](https://github.com/peteromallet/minkhole-h3av-handover/releases/tag/timeline-v1). The `rough-cut.astrid` archive is the current edited timeline and machine-voice reference; `original-matrix-scene-reference.astrid` is the separate source-scene reference. They contain embedded media, not just timeline JSON.

For a machine with GitHub CLI installed:

```sh
gh release download timeline-v1 --repo peteromallet/minkhole-h3av-handover --dir ./minkhole-inputs
shasum -a 256 ./minkhole-inputs/*.astrid
```

Expected SHA-256:

```text
50e7bf1262f0b89bd95dcce5ee0edebf80eea7d446f6f8234c52f022a4110e2e  rough-cut.astrid
7a19fb1fc2253d8b6a39d8bd603d8b88bb0a6117d1cc1ac1f38ebf685658f8fb  original-matrix-scene-reference.astrid
```

Import the archives using Astrid's timeline archive import and verify they open and their media resolves. Import them into an appropriate project without overwriting an existing timeline. The project and timeline IDs below identify the original workspace; imported copies can have different IDs. Existing local paths and prototype worktrees in this brief describe evidence on the original Mac and may not exist on the receiving machine. The archives do not necessarily include project history, unreferenced assets, code, credentials, dependencies or all supporting files; inspect their contents and report any gaps rather than assuming a full workspace backup.

**Current first task:** make timeline clip appearance explicit and consistent across preview, export and visualization, then validate the Matrix clip's Fit/Fill framing as described in section 0. After that, proceed to the H3_AV editing and reusable workflow described in sections 1–8.

## Objective

**Use the Astrid project *Into the Minkhole* as a real-world test case to develop, improve, and document an end-to-end workflow for AI-powered video editing with the H3_AV pack.**

*Into the Minkhole* is a Matrix-inspired parody of going "down the rabbit hole."

The immediate creative goal is to transform the existing footage so that the characters deliver the intended dialogue and visual changes, using the target script already represented by the machine-voiced version of the timeline.

The broader goal is more important: **make this class of editing so well-tooled, structured, and documented that a substantially less capable agent can execute it reliably on future projects.**

You are the **orchestrator and creative tastemaker**. Delegate most implementation, investigation, and repetitive work to Haiku sub-agents, escalating genuinely difficult tasks to Sonnet. Retain responsibility for creative judgment, architectural decisions, quality, and the final result.

## 0. First: make timeline appearance explicit and consistent

Before beginning the H3_AV creative work below, fix the timeline appearance and verification problems uncovered in *Into the Minkhole*. Use this project as a real example, but solve the underlying problems in reusable Astrid tooling. Then continue with the creative task; this is the first phase, not a replacement for the rest of the brief.

### The requirement

**Everything that influences how a clip looks should be authored, visible and understandable from the timeline.** A user or agent should be able to inspect and change fit, frame, crop, position, scale, opacity and effects without discovering hidden behavior. If a shot or track contributes a transform or effect, show its effective influence and where it comes from, and let it be edited at the appropriate scope. Show source dimensions and output aspect ratio as context. Do not duplicate inherited settings onto every clip merely to make them visible.

The timeline is the creative source of truth. Preview, export, the timeline visualizer and editing handles must interpret the same authored settings consistently; there should not be a separate creative “render settings” layer. A derived appearance plan is fine, provided it is calculated from the timeline rather than maintained as another editable state.

### What happened and why this matters

- The current project is **Matrix — Into the Minkhole**, rough-cut timeline `350c1f5e08f54e7d8f3a34c49227afe5` in project `f282e58fcca14f42909b20d96859d7d6`. Its separate reference timeline is `c2e752fef9384483a6927ef9b2d89304`. Re-read current state before editing; these are references, not permission to overwrite newer work.
- The supplied shortened movie is `/Users/peteromalley/Documents/matrix_shot.mp4` (1920 × 800). The higher-quality aligned replacement is now in both timelines, and the three relevant rough-cut source excerpts use it. The managed replacement removes baked-in bars while retaining the original source audio and end screen. The third excerpt's authored out-point was shortened by approximately 0.001 seconds to satisfy an existing occurrence-boundary guard; the displayed endpoint remains on the same frame. Voiceover and other clip records were checked unchanged. The full movie is `/Users/peteromalley/Downloads/The.Matrix.1999.mp4`.
- A first attempt to fill the 1280 × 720 timeline enlarged the clip rectangle to 135%. It looked much too zoomed because positioned media already invokes **cover** inside that rectangle, compounding the enlargement. That change was reverted. The current framing is the pre-zoom state; do not assume a desired fill is already applied.
- A 1920 × 800 picture fitted into a 16:9 canvas leaves space above and below. A minimum centered Fill necessarily crops the sides: at 1280 × 720, the picture is 1728 × 720 and approximately 224 pixels are cropped from each side. A canvas-sized frame with **Fill** and authored zoom **1.0** is the intended starting point—not an additional 135% zoom. Validate visually and ask the user for creative review if the resulting crop is undesirable.
- Existing Astrid behavior implicitly changes fit based on the presence of geometry fields, even a neutral `x: 0`. The Release 14 preview and the shipped compositor also differ in how they apply track scale to positioned clips. These hidden, duplicated rules make preview and export liable to disagree.
- A black preview was observed during intermittent local request delays. The media file decoded correctly and the browser requested the correct object. Astra reproduced a delay through the local bridge, but the exact internal cause was not proven. Do not assume a black preview means the video is corrupt, or claim a network root cause without evidence.

### Proposed architecture and boundaries to fix

Astra and two independent helpers recommended **one source-owned timeline appearance evaluator**, shared by the editor, compositor/export, composed visualization and editing handles. Resolve clip settings, shot/track ancestors, media dimensions, canvas and animation into a derived, immutable appearance plan: viewport, media bounds, clipping and ordered transforms. Python and Runtime should preserve and validate the authored contract rather than independently invent fitting semantics.

1. Make Fit/Fill explicit and independent of position or size for new edits. A Fit/Fill action must change fitting only—not secretly change authored geometry or zoom. Remove duplicated geometry, fit and track-scale calculations in favor of the shared evaluator.
2. Preserve existing projects through a **versioned compatibility normalizer**, including the existing track-scale migration markers. Preview and export have already disagreed for some positioned track-scaled clips, so both historical outputs cannot always be preserved: choose and document an authored/versioned baseline, test the difference, and avoid silently rewriting project intent.
3. Do not promote a clip's `fit` to the whole track: mixed Fit and Fill clips must remain independent. The current prototype in `astrid-fit-schema-agent/astrid/packs/rendering/actions/render/managed_timeline.py` contains such an adapter and needs revision when the compositor supports per-clip fit.
4. Keep media admission, provenance, concurrency and renderer capability checks. Normalize valid authored media references into a verified renderable form at the appropriate boundary; do not silently skip unresolved clips or bypass safeguards. Unsupported appearance must fail with the relevant clip and property rather than export a different image.
5. The shipped `@banodoco/timeline-composition` 0.0.17 is available as readable compiled JavaScript, but its original TypeScript source and source maps were not found. The older source located cannot rebuild it. Astra recommends preserving the exact hash-pinned shipped artifact as a compatibility oracle and, if source cannot be recovered, restoring its readable JavaScript as a clearly labelled, source-controlled baseline **in the composition package's owning repository**, with provenance and license checked. Make narrow changes there, publish a new package version, and pin App and Agent coherently. This is source recovery and ongoing maintenance, not a claim that the original TypeScript was found. Reassess this recommendation against the current repository state before implementation; do not ship an app-only renderer or an undocumented patched tarball.

### Existing work, issues and required checks

Unpublished, isolated prototypes exist at `/Users/peteromalley/Documents/astrid-timeline-fit-fix`, `/Users/peteromalley/Documents/astrid-fit-schema-runtime`, `/Users/peteromalley/Documents/astrid-fit-schema-agent`, and `/Users/peteromalley/Documents/astrid-render-fit-guard`. They are **not a finished or deployed solution**. Review rather than blindly applying them; in particular, the clip-to-track fit adapter is not the intended final design. The recovered compositor distribution in the app worktree is an unmodified investigation copy. Check Git status and preserve unrelated user work.

The composed timeline visualizer failed with `installed registry discovery refuses explicit import or pack roots`; inputs mode worked but did not produce a rendered frame. This separate issue is [astrid-dev #748](https://github.com/banodoco/astrid-dev/issues/748). Repair it without disabling the installed-provenance guard, and use the composed visualizer to validate the Matrix crop. The multi-fork validator issue encountered during replacement is [astrid-dev #745](https://github.com/banodoco/astrid-dev/issues/745); individual exact-head publications were used as a guarded workaround. The settings URL issue is [astrid-dev #736](https://github.com/banodoco/astrid-dev/issues/736), whose fix was merged to Release 14. Check current issue and branch state before duplicating work.

Before calling this phase complete:

- Verify old project appearances with rendered-frame comparisons: positioned and unpositioned media, crop, held images, track scale, nested shots and legacy migration markers.
- Test mixed Fit and Fill clips on the same track and show inherited appearance influences in the inspector. Verify a Fit/Fill change does not also alter position, crop or authored scale.
- Compare the **same candidate** in timeline preview, exported frames and composed visualization; inspect the Matrix shots and confirm the smallest intended fill without compounded zoom.
- Verify source/package provenance and matching App/Agent compositor pins. Exercise unsupported paths and ensure they fail clearly rather than silently changing output.
- Preserve project backups and use guarded current-head edits. Do not claim full export verification from preview alone. Previous export attempts encountered both the visualizer/worker environment issue and a separate existing asset-metadata validation issue; diagnose current state rather than stripping unrelated project metadata.

Only after these foundations are verified should the agent proceed to the H3_AV creative workflow below. If a larger issue must be deferred, document the concrete limitation and an actionable issue rather than concealing preview/export disagreement.

## 1. Start with the actual creative task

Inspect the *Into the Minkhole* project in Astrid, including its timeline, source clips, target dialogue, and existing machine-voice reference.

For every segment requiring modification:

- Understand what happens visually and what is currently said.
- Identify the desired dialogue, visuals, and timing in the final output.
- Determine which parts of the original footage should be preserved.
- Decide the best editing technique and where the edit should begin and end.
- Identify the source material, voice references, masks, and other inputs required.

Consider techniques including:

- **Clip continuation:** Preserve an initial portion of a clip, then generate subsequent video and audio.
- **Dialogue replacement:** Change what a character says while preserving their appearance, performance, and surrounding footage.
- **Spatial video inpainting:** Modify a selected region, such as a mouth, face, or object, while maintaining the rest of the shot.
- **Temporal and multi-region masking:** Apply masks across selected frames or multiple independently controlled regions, potentially using SAM-based segmentation.
- **Voice-conditioned generation:** Use appropriate reference footage or audio to preserve a character's voice when generating new dialogue.

Choose techniques based on their creative suitability, not merely on what is easiest to invoke.

## 2. Enrich the clips with structured information

Before editing, establish what information an agent needs to understand, plan, execute, and evaluate each transformation.

In particular, investigate:

- Word-level transcription with timestamps for both source and generated clips, potentially using Whisper through ComfyUI.
- Storage and retrieval of transcriptions associated with the correct clips, segments, and timeline positions.
- Source-to-target dialogue alignment, including the reference machine-voice track.
- Clear records of desired changes, preserved regions, edit boundaries, voice references, masks, and generation settings.
- A way to compare intended dialogue with actual generated dialogue.

**Sense-check Astrid's existing data structures.** Determine whether these annotations and relationships can already be represented cleanly. Where necessary, improve the structures rather than relying on fragile, one-off notes.

The goal is to give future agents a persistent, inspectable understanding of what each clip contains, what should change, and what has already been attempted.

## 3. Improve the tooling while using it

For each editing technique required by the project, investigate the existing H3_AV capabilities and identify what is missing or unnecessarily difficult.

Consider both missing functionality and avoidable orchestration complexity.

Examples include:

- Generating and applying spatial or temporal masks.
- Preparing video/audio inputs and reference voices.
- Selecting precise edit boundaries using transcription timestamps.
- Invoking ComfyUI workflows without manually assembling complicated inputs.
- Maintaining continuity across generated and preserved footage.
- Inspecting, comparing, retrying, and accepting results.

**Do not assume a capability is missing before testing what already exists.**

Prefer reusable, composable tools with simple interfaces over project-specific workarounds. The ideal agent interaction should describe *what edit is needed*, with the tooling handling as much of the mechanical preparation and execution as possible.

## 4. Build creative knowledge into the H3_AV skill

Alongside technical improvements, develop practical skill documentation within the H3_AV pack.

This should teach an agent not just how to invoke tools, but **how to make good creative decisions when using them**.

Cover questions such as:

- How do you choose the right source footage and voice reference?
- When should you continue a clip versus inpaint or regenerate it?
- How do you choose a boundary that preserves natural speech and motion?
- How do you align newly generated dialogue with the desired script and performance?
- How do you detect problems with lip sync, voice consistency, continuity, pacing, or unwanted changes?
- How do you compare the generated output against both the intended result and the original?
- When should you retry, modify the approach, or ask the user for creative review?
- How should candidate results be presented to the user for approval?

Document useful heuristics, failure modes, examples, and decision rules discovered through actual editing.

The documentation should be sufficiently procedural that a less intelligent agent can follow it without reinventing the reasoning.

## 5. Run a continuous build–use–learn loop

Treat the editing project as both a production task and a test of Astrid's capabilities.

For each segment:

1. **Understand:** Inspect and annotate the source material and intended result.
2. **Plan:** Choose an editing approach and identify required inputs.
3. **Assess:** Determine whether existing tools and skills can execute it reliably.
4. **Improve:** Fix missing or awkward capabilities where justified.
5. **Execute:** Generate candidate outputs.
6. **Evaluate:** Compare against the target script, original footage, and creative intent.
7. **Review:** Present meaningful options or uncertainties for user judgment.
8. **Codify:** Feed lessons into reusable tools, data structures, and skill documentation.

Avoid treating a successful one-off generation as sufficient. The question is always: **could another agent reproduce this result, or a similar one, without our intervention?**

## 6. Fix friction opportunistically; capture the rest

Throughout the work, actively identify friction in Astrid, H3_AV, agent workflows, data structures, and documentation.

Apply a pragmatic rule:

- **Fix it now:** If an improvement is reasonably straightforward, materially reduces friction, and can be handled by one or two sub-agents without significantly distracting from the main task, implement and validate it.
- **Create an issue:** If an improvement is larger, uncertain, or outside the immediate scope, create a GitHub issue in `astrid-dev` rather than allowing it to derail the creative work.
- **Avoid duplication:** Check existing issues and plans before opening new ones. Link related work where appropriate.

Issues should explain the observed problem, why it matters, how it was encountered, a concrete reproduction or example where possible, and the desired outcome.

Do not merely accumulate a list of complaints. Aim to produce an actionable backlog of improvements grounded in actual experience.

**Keep thinking at the meta level.** Regularly examine not only whether an individual tool works, but whether the entire process is unnecessarily complicated.

Ask:

- Why did this require so many steps?
- Why did an agent need to reason about this manually?
- Could a better interface, skill, or data structure eliminate that reasoning?
- Is the problem specific to this clip, or will future projects encounter it?
- Could we restructure the workflow so a less capable model succeeds more reliably?

The objective is to progressively eliminate unnecessary complexity while preserving creative flexibility.

## 7. Orchestration and delegation

Act as the overall orchestrator, maintaining a coherent understanding of the project and its creative objectives.

Use Haiku sub-agents for most execution, such as clip analysis, transcription, workflow inspection, implementation, testing, and documentation. Use Sonnet selectively for difficult engineering or reasoning problems.

Your responsibilities include:

- Breaking work into independently executable tasks.
- Maintaining the relationship between individual segment edits and the complete timeline.
- Choosing between alternative technical and creative approaches.
- Reviewing delegated work rather than accepting it blindly.
- Avoiding redundant investigation or unnecessary complexity.
- Opportunistically fixing small sources of friction and recording larger ones as GitHub issues.
- Ensuring improvements become reusable parts of Astrid and H3_AV.

**Optimize the process for future agents, not for demonstrating the intelligence of the current orchestrator.**

If something requires sophisticated judgment today, investigate whether that judgment can be translated into an explicit decision rule, a better tool, a structured data field, or a targeted user-review step.

## 8. Definition of success

There are two equally important deliverables.

**A. The finished creative project**

The *Into the Minkhole* timeline contains the intended dialogue and visual transformations, with results reviewed for intelligibility, timing, performance, visual consistency, and overall creative quality.

**B. A reusable editing system**

Astrid and H3_AV have demonstrably improved through the process:

- Source and output clips can carry useful transcription and editing metadata.
- Required editing techniques have straightforward, reusable tooling.
- The H3_AV skill contains practical, tested instructions for planning, executing, and evaluating edits.
- A repeatable runbook covers the complete process, from inspecting a timeline to delivering reviewed results.
- Straightforward improvements discovered during editing have been implemented where practical.
- Larger opportunities and unresolved problems have been captured as actionable GitHub issues in `astrid-dev`.
- A less capable agent can manage comparable projects with minimal supervision, escalating only genuinely ambiguous creative decisions.

**The ultimate test:** Give a future agent a different timeline, a target script, and access to Astrid and H3_AV. It should be able to determine what needs changing, prepare the necessary inputs, select and execute appropriate editing techniques, evaluate its results, and request meaningful user review—without needing to rediscover the workflow.

The finished video is the proof that the system works. The reusable system is the lasting deliverable.
