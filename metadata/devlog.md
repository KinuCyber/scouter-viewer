# Project Scouter — Research Log

## 2026-08-20

### 🔍 Focus: Mouse Look & Input Hijacking Fixes

#### Session Overview

- **Project Focus:** Troubleshooting Engine-Level Mouse Look Failure & Player Controller Input Hijacking
- **Target Character:** Non-humanoid Cyclops Cat (`BP_Mushroom_Character`)

#### Completed Tasks, Diagnostics & Resolutions

##### 1. UE5 Mouse Look & Controller Diagnostics

**Symptom & Debugging:**
Player character camera was completely locked; mouse movements produced zero camera rotation. Hardware mouse input was verified registering via Unreal Engine debug overlay, but `Mouse XY 2D-Axis` output remained at $0$.

**Diagnostic Mistakes & False Leads:**

- *Premature Diagnostic Fallacy:* Spent significant time assuming a hardware or OS raw input failure rather than an engine Blueprint state conflict.
- *Blueprint Hierarchy Misjudgment:* Investigated `BP_Mushroom_Character` wiring and mapping modifiers (`Scalar`, `Negate` in `IMC_Default`) under the false assumption that context modifiers were suppressing axis values.
- *Overlooking Parent Controller Logic:* Failed to check `BP_ThirdPersonPlayerController` early. Its default `Event BeginPlay` was running UI/cursor code that silently intercepted Slate viewport focus and stripped mouse delta before the character event graph ever received it.
- *Mobile/Touch Template Interference:* Overlooked the default UE5 Third Person template's touch/mobile control logic.

**Root Cause:**
`BP_ThirdPersonPlayerController` was defaulting the mouse input mode to UI/Cursor mode on `BeginPlay`, blocking raw axis data from reaching `IA_Look`.

##### 2. Engine Fixes & Corrective Actions

- **Player Controller Override:** Modified `BP_ThirdPersonPlayerController`'s `Event BeginPlay` to force `Set Input Mode Game Only`.
- **Editor Settings:** Enabled `Game Gets Mouse Control` under `Editor Settings → Level Editor → Play`.
- **Sensitivity Boost:** Added a `Scalar (2.0, 2.0, 2.0)` modifier to `Mouse XY 2D-Axis` in `IMC_Default`.

#### Key Mistakes to Avoid

- **Controller Hierarchy First:** Always inspect `BP_PlayerController` logic *before* troubleshooting character pawn inputs. If the controller's input mode is wrong, no character-level input logic will execute correctly.
- **Never Trust Template Defaults:** Default template Blueprints are not clean slates. Template logic (e.g., touch input handlers) frequently causes silent overrides.

#### Next Steps

- Complete blueprint node fixes for hiding the cursor during gameplay.
- Transition to character pipeline setup (rigging, blending, interaction architecture).

---

## 2026-08-21

### 🔍 Focus: Blueprint Execution Wiring & Rigging Architecture

#### Session Overview

- **Project Focus:** Node Wiring Resolution, Execution Flow Mechanics, & Character Architecture / Rigging Design
- **Target Character:** Non-humanoid Cyclops Cat

#### Completed Tasks, Diagnostics & Resolutions

##### 1. Blueprint Node Wiring & Execution Line Fixes

**Symptom:** Could not find a `Target` pin on `Set Show Mouse Cursor` when connecting from `Get Player Controller`.

**Diagnostic Mistakes:**

- *Variable Setter vs. Function Call:* Selected a raw variable `SET` node instead of the blue Target Function node. Raw variable setters do not have target inputs.
- *Execution Line Ignorance:* Connected data wires (blue pins) while leaving the white execution arrow ($\triangleright$) disconnected — Unreal silently skipped the node at runtime.

**Resolutions:**

- Deleted the raw variable `SET` node.
- Pulled a wire from `Get Player Controller` (Return Value) and spawned the blue function node `Set Show Mouse Cursor`, automatically creating the correct `Target` pin.
- Connected the white execution arrow out of `Set Input Mode Game Only` into `Set Show Mouse Cursor`.
- Unchecked `Show Mouse Cursor` (value: `False`).
- *Alternative:* `Show Mouse Cursor` can also be toggled globally under `Class Defaults` in `BP_ThirdPersonPlayerController`.

##### 2. Character Locomotion & Animation Architecture

**Velocity-Based Speed Blending:**
Established a 1D Blend Space strategy ($0 \rightarrow \text{Max Speed}$) blending: Idle ($0$) → itty-bitty steps ($50$) → walk ($150$) → jog ($300$).

**Sprinting Mechanics:**
L3 (`IA_Sprint`) dynamically switches `Max Walk Speed` in the Character Movement Component from $300$ to $600$.

**Procedural Banking / Leaning:**

- Evaluated 2D Blend Spaces (requires manual lean clips in Blender) vs. UE5 Control Rig / `Transform (Modify) Bone`.
- **Selected:** Procedural spine roll via `Transform (Modify) Bone` driven by angular turning velocity — saves animation time while providing fluid procedural banking.

##### 3. Advanced Contextual Mechanics

**Parkour & Squeezing:** Root Motion Animations combined with UE5 **Motion Warping** for counter leaps and cabinet squeezing. Motion Warping dynamically adjusts animation landing targets to match trigger volume transforms.

**Facial System:** Approved Shape Keys / Blendshapes (Morph Targets) over bone-based facial rigs for expressions (blinking, squinting, brow drops).

**Eating Actions:** Driven via head-neck arc motions, pecking bite trajectories, head-roll chew cycles, and neck squash-and-stretch scaling — jaw does not open wide.

##### 4. Skeleton Blueprint & Helper Socket Architecture

- **Rig Lock Rule:** Freeze and lock skeleton hierarchies *before* producing animations — prevents corrupting animation tracks and skin weights.
- **Helper Sockets:** Non-deforming helper bones ($0.0$ vertex weight) serve as anchor points for cosmetics and mechanics without altering deformation skinning.

#### Key Mistakes to Avoid

- **Data Wires Do Not Execute Code:** Blue/green wires only pass data. Nodes with white execution arrows ($\triangleright$) **must** be wired into the execution chain or Unreal will silently ignore them.
- **Function Node vs Variable Setter:** Always drag off a target pin to spawn a Function Call — never a raw variable `SET`.
- **Never Modify Rig Hierarchies Post-Animation:** Adding/deleting bones after keyframing breaks bone transforms and invalidates existing FBX animations.

#### Next Steps

- Finalize custom cat bone hierarchy and lock in Blender armature setup.
- Execute $0,0,0$ transform freezes, UV mapping, and vertex weight painting.
- Export FBX mesh, morph targets, and ORM textures for UE5 Master Material setup.

---

## 2026-08-22

### 🔍 Focus: Rig Spec Consolidation & Dev Log Automation

#### Session Overview

- **Project Focus:** Rig Specification Consolidation, Ecosystem Integration Realities, and Automated Dev Log Generation
- **Target Character:** Non-humanoid Cyclops Cat

#### Finalized Bone Blueprint

Comprehensive bone hierarchy for a single-eyed quadruped cat:

- **Core/Spine:** `Root` $(0,0,0)$ → `Pelvis` → `Spine_01..03` → `Neck` → `Head`
- **Limbs:** Clavicle/Thigh down to Paws with front socket anchors (`Paw_Front_Socket_L/R`)
- **Tail & Ears:** 5-segment tail (`Tail_01..05`), `Tail_Tip_Socket`, base-to-tip ear joints
- **Facial/Snout:** `Eye_Center` (single cyclops eye), `Bone_Eye_Socket` (2D pupil anchor), `Jaw` (micro-opening), `Bone_Mouth_Socket`, `Bone_Snout_FX`, 3x whiskers per side
- **Gear Helpers (0.0 Weight):** `Bone_Hat`, `Bone_Glasses`, `Bone_Neck_Collar`, `Bone_Spine_Attach`

#### System Capabilities & Misconceptions

**Google Keep / AI Access Reality:** AI assistants cannot natively read/write to personal Google Keep or Workspace accounts without an active real-time API integration. Resolved via manual copy-paste markdown workflow.

#### Key Mistakes to Avoid

- **Distinguish Template vs. Content:** Use reference `.md` files as structural schemas — not as content to summarize.
- **Raw Formatting Requests:** Wrap output in explicit code blocks for external transfer to Notion/Obsidian.

---

### 🔍 Focus: Cozy Life-Sim Scope & Hybrid VR Possession Architecture

#### Scope Definition

- **Genre:** Cozy Life-Sim / Observation Game
- **Core Loop:** Ambient exploration → owner observation → bonding → unlocking machine control
- **Perspective Swap:** Third-person quadruped cat → VR first-person humanoid robot

#### Technical Framework (UE5 Hybrid VR)

**Possession Flow:**

1. Player controls `BP_Mushroom_Character` via standard third-person flat inputs.
2. On activating the robot interface: `PlayerController` unpossesses the cat, possesses `BP_HumanoidVR`.
3. `AIController` takes over `BP_Mushroom_Character`, running an idle Behavior Tree (wandering, sleeping, responding to VR hand triggers).

**Optimization:** Maintain $1\text{ unit} = 1\text{ cm}$ real-world scale across all Maya assets for spatial accuracy when switching from low-angle cat view to VR eye level.

---

### 🔍 Focus: Touch Gestures & AnimBP State Transitions

#### Dual-Point Touch Gesture Detection

- **Collision Setup:** Dual trigger volumes (`Trig_Spine_L`, `Trig_Spine_R`) along the spine mesh.
- **Sequence Logic:** Measured delta time ($\le 0.4\text{s}$) between left-to-right or right-to-left overlap events to evaluate swipes.
- **Animation Responses:**
  - *Left → Right Swipe:* Plays fall-to-right animation (`LyingRight` state).
  - *Right → Left Swipe:* Triggers roll-over to left side (`LyingLeft` state).

#### Auto Stand-Up Timer

- $10.0\text{s}$ countdown timer on entering any lying pose.
- Interrupted and restarted on new touch events.
- On expiry: triggers `Anim_StandUp`, resets pose to `Standing`.

---

### 🔍 Focus: Character Architecture & VR Possession Persistence

#### Controller Swap Architecture

- **Pawn Retention:** `BP_Mushroom_Character` remains active in the level; only controller ownership shifts between `PlayerController` and `AIC_Mushroom_Cat`.
- **Stat Continuity:** `Hunger`, `Energy`, `Affection` persist natively on `BP_Mushroom_Character` — no save-game serialization needed during runtime swaps.

#### AI & VR Interoperability

- **Behavior Tree:** `AIC_Mushroom_Cat` reads real-time stats from `BP_Mushroom_Character` to drive autonomous states (e.g., resting when `Energy` is low).
- **VR Gameplay Loop:** VR hand interactions directly modify the cat's core stats, ensuring continuity when reverting to third-person control.

---

### 🔍 Focus: Scope Management & Pre-Production Lock

#### Technical Standards

- Standardized $1\text{ unit} = 1\text{ cm}$ metric scale and Y-Up / X-Forward world alignment across Maya and UE5.
- Confirmed DirectX normal map format and ORM packed texture pipeline.

#### Architectural Decisions Locked

- MoSCoW framework for MVP scope: movement, swipe gestures, basic stats, possession swap.
- Variable persistence directly on `BP_Mushroom_Character`.
- Custom skeleton structure and socket helper bone naming locked before skin binding.

---

### 🔍 Focus: AI Attention & Look-At Head Tracking

#### Environment Query System (EQS)

- Periodic EQS queries scoring nearby actors tagged `Interest.PlayerHead`, `Interest.PlayerHand`, or `Interest.Prop`.
- Filters: distance ($\le 300\text{cm}$), forward vision cone ($120^\circ$), hand motion velocity.

#### AnimBP Head Tracking

- `Neck`, `Head`, and `Eye_Center` driven via AnimBP `Look At` node.
- Angular clamps: $45^\circ$ max yaw/pitch.
- Liveliness: gaze timeouts ($2\text{s}$–$5\text{s}$), eye-blink Morph Target triggers on target swaps, random micro-saccade offsets.

---

### 🔍 Focus: Tail Physics & VR Hand Collision

#### Procedural Tail Physics

- Bone chain `Tail_01` through `Tail_05` via Kawaii Physics / Anim Dynamics in AnimBP.
- PhAT sequential capsule colliders on tail joints to register physical impulses.

#### VR Interaction Mechanics

- VR hand sphere colliders drive skeletal deformation on overlap — spring-tensioned bending along the displacement vector.
- Configurable spring stiffness and damping constants restore tail to baseline keyframed state on collision exit.

---

### 🔍 Focus: Phased Roadmap & VR Staging Validation

#### Pre-Production Integrity

- Delaying VR features causes zero changes to Maya skeletal hierarchies, socket helpers, or $1\text{ unit} = 1\text{ cm}$ scale standards.
- Third-person character variables and AnimBP dynamic physics will port directly into VR possession modes without structural refactoring.

#### Phase 1 Milestone

Flat third-person quadruped locomotion, 1D speed blending, swipe gestures, and basic item handling — all before any VR systems.

---

### 🔍 Focus: Auto-Idle Observation & Shared AI Control

#### Input Inactivity Detection

- `TimeSinceLastInput` float tracker on `BP_Mushroom_Character` reset by any player movement delta.
- Reaching $15.0\text{s}$ of inactivity automatically passes control to `AIC_Mushroom_Cat`.

#### Shared Behavior Tree Architecture

- Reuses VR NPC Behavior Tree for autonomous idling (sunspot seeking, grooming, stat-driven resting).
- Any registered player input immediately cancels AI execution and restores direct third-person control.

---

### 🔍 Focus: Mobile VR Performance Budgets (Pico 4)

#### Geometry & Mesh Budgets

- **Character Target:** `BP_Mushroom_Character` → 10,000–15,000 triangles (~12,000 tris baseline in Maya).
- **Scene Horizon:** Hard limit of 150,000 total visible triangles for 72–90 FPS stability.

#### Mobile Shader & Rig Constraints

- Max 4 bone influences per vertex; active bone palette well below 75 joints.
- 1–2 master shader materials using DirectX ORM texture packing.

---

### 🔍 Focus: PCVR / SteamVR Pipeline & High-Fidelity Specs

#### Expanded PCVR Budgets

- **Character Target:** 30,000–60,000 triangles.
- **Scene:** 1,000,000+ triangles.

#### Shading & Visual Fidelity

- Subsurface Scattering (SSS) on ear/skin meshes, complex eye shaders, dynamic shadow maps.
- High-poly PCVR base mesh as primary source asset; auto-LODs (~12,000 tris) for mobile ports.

---

### 🔍 Focus: Mid-Range PCVR Target (RTX 3060)

#### Geometry Budgets

- **Character:** 25,000–35,000 triangles (~30,000 baseline in Maya).
- **Scene:** 500,000–800,000 triangles for stable 72–90 FPS per eye.

#### VR Rendering Engine

- Lumen GI disabled for VR runtime; Baked GPU Lightmass / optimized dynamic lights.
- Forward Shading with MSAA for clean image resolution over streaming hardware.

---

### 🔍 Focus: Pico-First Baseline & Universal Targets

#### Final Geometry Policy

- **Locked Baseline:** ~12,000 tris for `BP_Mushroom_Character`; 150,000 total visible scene tris.
- **Universal Asset Model:** Mobile-first optimization as the default across all development phases.
- **Materials:** 1–2 master shader slots using DirectX ORM texture maps.

---

### 🔍 Focus: Stylized Pupil Rig & 4-Tier Gaze Hierarchy

#### Pupil Geometry & Shader Setup

- **Rigging Method:** Geometric overlay (`pPupil_Geo` over `pIris_Geo`) driven by `Bone_Pupil_Control`.
- **Material Slot Optimization:** Single material slot across iris and pupil geometry = 1 Draw Call for the eye assembly.
- **Z-Fighting:** $0.1\text{cm}$ outward offset along eye normals to prevent z-fighting on mobile VR hardware.

#### 4-Tier Gaze Tracking

| Tier | Angle | Behavior |
|------|-------|----------|
| 1 | $0^\circ$–$15^\circ$ | Eye-only glance — drives `Bone_Pupil_Control` translation/rotation |
| 2 | $15^\circ$–$45^\circ$ | Pupil max offset + incremental `Head` bone rotation |
| 3 | $45^\circ$–$70^\circ$ | `Neck` & `Head` rotate; pupil interpolates back to neutral center |
| 4 | $>70^\circ$ | Triggers `Anim_Turn_L_90` or `Anim_Turn_R_90`; re-aligns body forward vector |

#### Maya Animation Requirements

`Anim_Turn_L_90` and `Anim_Turn_R_90` keyframed in-place with zero Root Motion displacement. Start and end poses matched to `Anim_Idle_Base`.

---

### 🔍 Focus: Locomotion State Machine & Pose Transitions

#### Animation Deliverables (Maya)

- **Base Stances:** `Anim_Idle_Stand`, `Anim_Idle_Sit`, `Anim_Idle_Lie`
- **Pose Transitions:** `Anim_Stand_To_Sit`, `Anim_Sit_To_Stand`, `Anim_Stand_To_Lie`, `Anim_Lie_To_Stand`
- **Pivot Locomotion:** `Anim_Turn_L_90`, `Anim_Turn_R_90` (standing pivots only)

#### Engine State Rules (UE5 AnimBP)

- `Stand` state must be active before playing 90° body turns.
- Additive spine blending for small rotational adjustments ($\le 45^\circ$) while seated — avoids unnecessary stand-up cycles.

---

### 🔍 Focus: Foot Sliding Mitigation & Procedural IK

#### Foot Sliding Countermeasures

- UE5 Control Rig: procedural raycast ground checks + Two-Bone IK on paw joints.
- Root motion data for turn pivots ties animation translation directly to world space, eliminating slip.

---

### 🔍 Focus: IK Deferral & Scope Protection

#### Scope Containment Decision

- All UE5 Foot IK, Control Rig line traces, and ground-adaptation setups deferred to **post-alpha polish phases**.
- Strict FK skeletal leg structure in Maya — no embedded custom IK constraints.

#### Immediate Objective

Proceed directly to Maya mesh finalization, UVs, basic FK skinning, and initial FBX export.

---

### 🔍 Focus: Pre-Export Rig & Mesh Validation

#### Mesh & Topology Lock

- **Vertex Order:** Hard-locked final topology before Morph Target (blend shape) generation.
- **Material Slots:** Single material assignment (`M_CyclopsCat`) for 1 Draw Call rendering.

#### Rigging & Skinning Parameters

- Max 4 bone influences per vertex (mobile GPU compatibility).
- Uniform joint axis alignments and zeroed bind pose channels verified.
- FBX Export: Centimeter units, Y-Up alignment, Skins/Blend Shapes enabled.

---

### 🔍 Focus: Phase 1 Animation Deliverables & Blending

#### Core Animation Clips

| Clip | Description |
|------|-------------|
| `Anim_Idle_Base` | Looping breathing idle; base pose for AnimBP state machine |
| `Anim_Walk_Loop` | In-place walk cycle, $0$–$150\text{ cm/s}$ velocity |
| `Anim_Turn_L_90` / `_R_90` | In-place $90^\circ$ pivot steps for Tier 4 gaze alignment |
| `Anim_Pose_Test` | Extreme joint rotation clip for skinning verification |

#### Pipeline Constraints

All locomotion loops keyframed in-place with `Root` locked at $(0,0,0)$ origin — zero Root Motion displacement.

---

### 🔍 Focus: Multi-Tiered Inactivity & Super-Idle Transitions

#### Idle Inactivity Milestones

| Tier | Duration | State |
|------|----------|-------|
| 1 | 0s–30s | `Anim_Idle_Stand` (default active standing) |
| 2 | 30s–60s | `Anim_Stand_To_Sit` → `Anim_Idle_Sit` loop |
| 3 | >60s | `Anim_Sit_To_Lie` → `Anim_Idle_Lie` sleeping loop |

Added `Anim_Idle_Sit`, `Anim_Stand_To_Sit`, `Anim_Idle_Lie`, and `Anim_Sit_To_Lie` to pre-production deliverables.

---

### 🔍 Focus: Layered Look-At for Resting Stances

#### Additive Animation Architecture

- **Layered Blend per Bone:** `Spine_02` through `Bone_Pupil_Control` isolated for procedural EQS tracking during resting states.
- **Stance Clamping:** Stricter angular limits ($30^\circ$ max pitch/yaw) for seated/lying look-at nodes to preserve mesh volume.

#### Proximity State Overrides

VR player proximity within $100\text{cm}$ activates Tier 1–3 tracking. Exceeding angular vision boundaries triggers `Anim_Sit_To_Stand` or `Anim_Lie_To_Stand` transition.

---

### 🔍 Focus: Hype / Engagement Meter Architecture

#### HypeLevel Variable Dynamics

- `HypeLevel` float ($0.0 \rightarrow 1.0$) stored on `BP_Mushroom_Character`.
- **Accumulation:** VR hand interactions +0.2 per hit.
- **Decay:** −0.02/s continuous; waking from `Anim_Idle_Lie` caps starting Hype at 0.0.

#### Behavioral Gating

| Range | Behavior |
|-------|----------|
| < 0.3 | Blocks Tier 3/4 gaze and stand-up transitions; minor ear/eye twitches only |
| 0.3–0.7 | Standard EQS tracking execution |
| > 0.7 | Immediate tracking, gaze snaps, instant stance break to investigate |

---

### 🔍 Focus: Hype System Effort & Implementation Complexity

- **Complexity Assessment:** Trivial — implemented via native UE5 Blueprint nodes (`Clamp`, `FInterp To`, `Branch`).
- **Resource Budget:** 1.0 Manday total across character variables, AI decision gating, and AnimBP tuning.

---

### 🔍 Focus: Ambient VR Teaser Asset Integration

#### Baron Steelfist Preview Character

- *Baron Steelfist* (*Direct & Dominate* series) as a passive ambient NPC in the room level.
- Uninteractable via direct UI/inputs; configured for simple physical proximity overlap with `BP_Mushroom_Character` (triggering subtle cat ear/tail twitches on contact).
- Budget: $\le 10,000$ triangles and 1 Material Slot.

---

### 🔍 Summary — Key Takeaways (0937hrs)

Consolidated recap across all sessions from 2026-08-20 through 2026-08-22:

- **Input & camera debugging:** PlayerController hierarchy first — UI/cursor input modes silently kill mouse delta before the Pawn sees it. Fix: `Set Input Mode Game Only` + ensure viewport focus settings.
- **Blueprint execution discipline:** Data wires don't run logic. Always verify the white execution chain. Spawn function nodes by dragging off a valid target reference — avoid raw `SET` variable nodes.
- **Locomotion plan:** 1D velocity blend space (idle→walk→jog), sprint via runtime `Max Walk Speed`, procedural banking via bone transforms instead of extra animation clips.
- **Interaction & traversal:** Root Motion + Motion Warping for context actions (counter jumps, squeezing) so animations adapt to target transforms.
- **Rigging rules:** Freeze/lock skeletal hierarchy before animation. Use 0-weight helper bones for attachments to prevent deformation/track corruption.
- **Facial/performance:** Blendshapes for expressions/blinks. Limited-jaw eating conveyed via head/neck arcs and squash/stretch — not wide jaw opens.
- **Hybrid architecture:** Cat pawn unpossessed/possessed into VR pawn. Stats persist on cat pawn. AIController runs cat during VR mode.
- **Touch gestures:** Dual trigger volumes + short time-window sequencing enables swipe detection with auto-stand recovery.
- **AI attention:** EQS feeding AnimBP look-at for neck/head/eye with clamps, timeouts, blinks, micro-saccades.
- **Secondary motion:** Tail dynamics via Anim Dynamics/Kawaii Physics + PhAT. VR hand collision drives reactive bending with spring recovery.
- **Scope protection:** Third-person locomotion + core gestures/stats ship first. Foot IK/Control Rig deferred to polish.
- **Performance budgets:** Clear triangle/material/bone-influence targets for Pico/XR2 with scalable PCVR headroom.

---

### 🔍 DevLog: Maya Pipeline & macOS Engine Setup (1253hrs)

#### 3D Asset Pipeline & Workflow Pivots

**Software Pivot:**
Initially attempted modeling in Blender. Experienced efficiency bottlenecks and UI friction. Pivoted to **Autodesk Maya** to leverage existing muscle memory and maximize production speed.

**Maya Export Strategy:**

- Modeled and rigged base character mesh ("Freaky Cat").
- Structured Walk, Idle, Attack animations on a single unified timeline.
- Configured Maya **Game Exporter** (`File → Game Exporter`) to split timeline clip ranges into discrete, UE5-ready `.fbx` files in a single export pass.

#### macOS Engine Environment Configuration

**Problem:** Unreal Engine blocked on launch — missing Metal Graphics Compiler dependencies required for shader compilation on Apple Silicon.

**Investigation:**

- Tested `xcode-select --install` (Command Line Tools ~500MB).
- *Outcome:* Rejected by `xcodebuild` — Metal Toolchain binaries are packaged exclusively inside full Xcode builds.

**Resolution:**

1. Audited Epic Games documentation to identify version compatibility (avoid Xcode 26.4).
2. Downloaded **Xcode 26.0 (Apple Silicon)** from Apple Developer archives (`.xip` ~2.08 GB).
3. Moved to `/Applications` and linked paths in Terminal:

```bash
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
```

4. Verified with `xcode-select -p`.
5. Initialized first-run components; skipped 2GB Predictive Code Completion model to save storage.

---

### 🔍 Focus: Volumetric Shell Fur Shading (1631hrs)

#### Session Overview

- **Target:** `BP_AlphaCat` / `skm_alphacat`
- **Objective:** Lightweight mobile-VR-compliant volumetric fuzz silhouette that deforms with skeletal animations — no geometry shaders, no traditional strand grooming

#### Shading Model & Node Logic

**Problem:** Initial Default Lit material conveyed no structural depth.

**Solution:** Upgraded `M_Fur_Base` to the **Cloth** shading model to open fuzz scattering channels.

**Diagnostic Mistakes:**

- Introduced a circular feedback loop routing a Subtract node back into an upstream Multiply node — bleached color parameters into a floating gray grid.
- Routing a scalar gradient from FuzzyShading directly into a binary Opacity Mask under Masked Blend Mode made the mesh 100% transparent.

**Fixes:**

- Broke the circular connection on the top Multiply node.
- Rerouted `BaseColor` parameter through `FuzzyShading` (BaseColor V3) input first, then bridged Result pin into both Base Color and Fuzz Color slots.
- Restored Noise block to its correct dual pathway (color blend sequence + Subtract A port).

#### Component Architecture & Leader Pose

**Problem:** Duplicating raw mesh files in the World Outliner caused fur layers to float independently, failing to track animation bones.

**Solution:**

- Wrapped setup into `BP_AlphaCat` Blueprint class.
- Nested `Fur_Layer_01`, `Fur_Layer_02`, `Fur_Layer_03` as child items under main `Mesh (CharacterMesh0)`.

**Base Layer Override (`MI_Layer_0`):**
`ShellDistance = 0.0`, `LayerIndex = -1.0` — forces subtraction math to ignore transparency cutouts, creating a 100% solid color backing layer that hides inner hollowing.

**Leader Pose Construction Script:**
`Set Leader Pose Component` node: main `Mesh` → `New Leader Bone Component`; all three fur layers → `Target`. Passes bone transform matrices to child components every frame.

**Blueprint Architecture:**

```text
[BP_AlphaCat]
  │
  ├── Construction Script ──► Set Leader Pose Component
  │                              ├── Leader: Mesh (CharacterMesh0)
  │                              └── Targets: [Fur_Layer_01, 02, 03]
  │
  └── Components Hierarchy
        └── Mesh (CharacterMesh0) [MI_Layer_0 | Offset: 0.0 | Index: -1.0]
              ├── Fur_Layer_01    [MI_Layer_1 | Offset: 0.1 | Index: 0.2]
              ├── Fur_Layer_02    [MI_Layer_2 | Offset: 0.2 | Index: 0.4]
              └── Fur_Layer_03    [MI_Layer_3 | Offset: 0.3 | Index: 0.6]
```

**Master Material Logic:**

```text
[BaseColor Param] ──► [Multiply A]
                            ▲
[Noise Node] ───────────────┤──► [Multiply B] ──► [FuzzyShading (BaseColor V3)]
                            │                               │
                            ▼                              ▼
                      [Subtract A]                  [Fuzzy Result]
                            ▲                       ├──► [Base Color Slot]
[LayerIndex] ──► [Subtract B]                       ├──► [Fuzz Color Slot]
                            │                       └──► [Cloth Input (1.0)]
                            ▼
                    [Opacity Mask Slot]

[VertexNormalWS] ──► [Multiply A]
                          ▲
[ShellDistance] ──► [Multiply B] ──► [World Position Offset Slot]
```

#### Key Mistakes to Avoid

- **Components Panel vs. Graph Nodes:** Manage structural properties (textures, physics, material slots) from the Components hierarchy window — not the blue reference variable node in the graph.
- **Raw Skeletal Mesh vs. Blueprint Class:** Raw Skeletal Mesh files (crimson identifier bar) hold only single-layer reference shapes. For custom scripts or layered component groupings, always drag the Blueprint Class (blue bar with gear icon) into the level.

#### Next Steps

1. Switch parent component to *Use Animation Asset* and assign walk/run loops to verify fur shell deformation under active bone loads.
2. Tune `ShellDistance` and `LayerIndex` per Material Instance for fur density and silhouette length.
3. Extend vertex network with a sine-wave wind vector for procedural fur sway during idle states.

---

## 2026-08-24

### 🔍 Focus: Modern Input Architecture & State-Driven Locomotion

#### Session Overview

- **Project Focus:** Local Project Migration, Enhanced Input System, Dynamic Animation Blueprint
- **Target Character:** `BP_AlphaCat` / `skm_alphacat`
- **Objective:** Hardware-agnostic camera orbit and movement; responsive state-driven animation blending without engine-side hacks

#### Local Development Migration

**Problem:** Testing over remote desktop broke camera navigation — remote cursor data uses absolute screen coordinates, but Unreal Engine requires raw relative mouse delta.

**Solution:**

- Trimmed project by deleting `DerivedDataCache`, `Intermediate`, and `Saved` — cut file size up to 90% without breaking assets.
- Renamed root folder and core activation file to `project_scooter`.
- Updated `ProjectName=project_scooter` inside `DefaultEngine.ini`.
- Booted natively on local hardware to restore absolute mouse tracking.

#### Enhanced Input System Setup

**Problem:** Right-click searches for input action events returned nothing — tokens not instantiated yet.

**Solution:**

1. Created `IA_Look` and `IA_Move` data assets (Axis2D / Vector2D profile).
2. Built `IMC_CatContext` mapping sheet:
   - `IA_Look` → Mouse XY 2D-Axis
   - `IA_Move` WASD with modifiers:
     - **W (Forward):** Swizzle Input Axis Values (YXZ)
     - **S (Backward):** Swizzle (YXZ) + Negate
     - **A (Left):** Negate
     - **D (Right):** No modifiers (default positive)
3. `BP_AlphaCat` Event Graph: `Event BeginPlay` → `Cast To PlayerController` → `Get Enhanced Input Local Player Subsystem` → `Add Mapping Context: IMC_CatContext`.

#### Directional Movement Math

**Problem:** Raw camera angles fed into movement nodes caused the character to burrow into the floor when the camera tilted downward.

**Fix:**

- Pull `Get Control Rotation` → `Break Rotator` (separate Pitch/Roll/Yaw).
- `Make Rotator` using **Yaw only** (Roll = 0, Pitch = 0).
- Multiply by `Get Forward Vector` and `Get Right Vector`.
- Result: flat 2D movement disk, always horizontal regardless of camera tilt.

#### State-Driven Locomotion Brain (`ABP_Cat`)

**Problem:** Hardcoded animation clip caused the cat to walk in place regardless of velocity. Shortcut fallbacks produced frozen mid-stride joints.

**Solution:**

- Exported `anim_cat_walk` and `anim_cat_idle` (40-frame breathing cycle) from Blender NLA editor with Fake User shield locked on both actions.
- Imported targeting existing `sm_alphacat_Skeleton` with *Import Mesh* unchecked.
- AnimBP Event Graph: `Try Get Pawn Owner` → `Is Valid?` → `Get Velocity` → `Vector Length` → `SET Speed`.
- Anim Graph: both clips into `Blend Poses by Bool`, toggled by `Speed > 1.0`.

#### Physics & Camera Calibration

- `SpringArm`: Target Arm Length `180.0`–`200.0`, Target Offset Z `55.0`.
- `CharacterMovement`: Max Walk Speed `200.0`, Rotation Rate Yaw `250.0`.
- Camera pitch inversion via Negate (Y-Axis) modifier in `IMC_CatContext`.

**Blueprint Architecture:**

```text
[BP_AlphaCat]
  │
  ├── Event BeginPlay
  │     └── Cast To PlayerController
  │           └── Get Enhanced Input Subsystem
  │                 └── Add Mapping Context: IMC_CatContext
  │
  └── Component Hierarchy
        └── CapsuleComponent (origin, collision, gravity)
              ├── SpringArm (Length: 180-200, Offset Z: 55)
              │     └── Camera (third-person over-tail)
              ├── CharacterMovement (Max Walk: 200, Yaw Rate: 250)
              └── Mesh [Transform: Z -90, Yaw -90] [Anim: ABP_Cat]
                    ├── Fur_Layer_01
                    ├── Fur_Layer_02
                    └── Fur_Layer_03
```

**Animation System (`ABP_Cat`):**

```text
EVENT GRAPH:
[Blueprint Update Animation]
  └── Try Get Pawn Owner ──► Is Valid? ──► Get Velocity ──► Vector Length ──► SET Speed

ANIM GRAPH:
[anim_cat_idle (Loop: ON)] ──► [False Pose]
                                      │
[anim_cat_walk (Loop: ON)] ──► [True Pose] ──► Output Pose
                                      │
                  [Speed > 1.0] ──────┘  (Active Value toggle)
```

#### Key Mistakes to Avoid

- **Remote Desktop Input Wall:** Never test fine 3D camera controls over streaming software — remote tools translate inputs as absolute coordinates, blocking relative mouse delta.
- **Promoting Pins vs. Background Searching:** Right-click the specific colored data pin to *Promote to Variable* — not the blank gray grid background.
- **Blender's Asset Trashing System:** Always press the **Shield Icon (Fake User)** on animation action blocks to prevent Blender from garbage-collecting them on file close.

#### Next Steps

1. Tune Rotation Rate (Yaw) ~`250.0`; test figure-eight patterns for weighted arc turns.
2. Verify fur child layers stay parented under movement loads (no gaps or tearing).
3. Add SpringArm Pitch limits to prevent camera clipping through the floor.

---

## 2026-08-26

### 🔍 Focus: Shell Fur, Accessory Rigging & Cross-Blueprint Input

#### Session Overview

- **Target Characters:** `BP_AlphaCat` (possessed player) and `BP_ScoutTheCat` (accessory-wearing companion)
- **Objective:** Volumetric fuzz + modular skeletal accessories (cape, cap, scarf) with real-time visibility toggle via cross-blueprint input architecture

#### Leader Pose for Skeletal Accessories

Built `Set Leader Pose Component` chain on `BeginPlay`:

- Leader: main `Mesh` (cat skeleton)
- Targets: `Scarf Mesh`, `Cape Mesh`, `Cap Mesh`
- **`In Follower Should Tick Pose: TRUE`** — required for continuous bone matrix evaluation

#### Live Simulation Accessory Freeze Bug

**Problem:** Toggling visibility in the Blueprint Details panel during PIE caused accessory meshes to freeze permanently.

**Cause:** Details panel edits during live simulation force component reconstruction, severing runtime `Set Leader Pose` bindings.

**Fix:** Use runtime Blueprint node execution (`Toggle Visibility`) for all toggles — never the Details panel during PIE.

#### Cross-Blueprint Input Architecture

**Problem:** Keys 1/2/3 failed to toggle visibility on `BP_ScoutTheCat`.

**Root Cause:** `BP_ScoutTheCat` was unpossessed. Enhanced Input Actions in unpossessed actor graphs are ignored by default. Also, wiring from the `Triggered` pin caused continuous per-frame firing.

**Solution:**

1. Created Custom Events in `BP_ScoutTheCat`: `ToggleCapEvent`, `ToggleScarfEvent`, `ToggleCapeEvent` — each wired into `Toggle Visibility`.
2. In `BP_AlphaCat` (possessed): `Get Actor of Class` → save as `ScoutRef` on `BeginPlay`.
3. Enhanced Input Actions wired via the **`Started`** pin (not `Triggered`) → call Scout's Custom Events via `ScoutRef`.

**Architecture:**

```text
[BP_AlphaCat — Possessed Player]
  │
  ├── BeginPlay ──► Get Actor of Class: BP_ScoutTheCat ──► [ScoutRef]
  │
  └── Enhanced Input (Started pin)
        ├── IA_ToggleCap   ──► ScoutRef ──► ToggleCapEvent
        ├── IA_ToggleScarf ──► ScoutRef ──► ToggleScarfEvent
        └── IA_ToggleCape  ──► ScoutRef ──► ToggleCapeEvent

[BP_ScoutTheCat — Unpossessed Companion]
  │
  ├── BeginPlay ──► Set Leader Pose Component
  │                   ├── Leader: Mesh (Cat Skeleton)
  │                   ├── Targets: [Scarf Mesh, Cape Mesh, Cap Mesh]
  │                   └── Tick Pose: TRUE
  │
  └── Custom Events
        ├── ToggleCapEvent   ──► Toggle Visibility (Cap Mesh)
        ├── ToggleScarfEvent ──► Toggle Visibility (Scarf Mesh)
        └── ToggleCapeEvent  ──► Toggle Visibility (Cape Mesh)
```

#### Key Mistakes to Avoid

- **Live Editing During PIE:** Never check/uncheck component properties in the Details panel while simulating — it breaks `Set Leader Pose` bindings.
- **Unpossessed Actor Input Isolation:** Capture inputs inside the possessed character; delegate via reference variables and Custom Events to unpossessed targets.
- **`Triggered` vs. `Started` Pins:** `Triggered` fires every frame while held — causes high-speed flickering. Always use **`Started`** for single-press toggle actions.

#### Next Steps

1. Validate accessories toggle cleanly during active walk/run animation without pose desync.
2. Fine-tune `ShellDistance` and `LayerIndex` on `BP_AlphaCat` fur Material Instances.
3. Transition test keys 1/2/3 into a UMG equipment menu with persistent visibility state.

---

## 2026-08-27

### 🔍 Focus: Procedural Aim-Point Head Tracking & AnimGraph Integration

#### Session Overview

- **Target:** `ABP_ScoutTheCat` (Scout)
- **Objective:** Scout tracks the player's camera aim point — not the camera itself — with soft interpolation, hemisphere gating, and angular clamps

#### Aim-Point Projection & Hemisphere Gating

**Problem:** Scout tracked raw camera position, causing his head to turn backward to stare into the lens.

**Diagnostic Mistakes:**

- *Vector Multiply Axis Flattening:* `Multiply` node defaulted to $(X=0, Y=2000, Z=0)$ — stripped depth and height, locking tracking to a single axis.
- *Reversed Select Logic:* `Select Float` had $A = 0.0$ (inactive), $B = 1.0$ (active) — tracking was ON when aim was behind Scout and OFF when in front.
- *Target Vector Mismatch:* `Subtract` node's input A was wired to raw `Get Camera Location` instead of the projected aim point, causing hemisphere validation to fail.

**Fixes:**

- Converted `Multiply` pin to Float scalar `2000.0`: $\text{Target} = \text{Cam Location} + [\text{Cam Forward} \times 2000]$.
- Swapped `Select Float` to $A = 1.0$ (active) and $B = 0.0$ (inactive).
- Re-routed `Subtract` input A to the `Add` node output (the projected aim point).

#### Null Pointer Prevention

**Error:** `Blueprint Runtime Error: Accessed None trying to read property CallFunc_GetPlayerCameraManager_ReturnValue` — triggered before camera manager initialization on world start.

**Fix:** Placed `Is Valid` execution node after `Get Player Camera Manager` to guard downstream `SET Camera Location` threads during early frame initialization.

#### AnimGraph Integration

- `Look At` skeletal control node targeting the `neck` bone.
- `Look at Location` ← `Camera Location` variable.
- `Alpha` ← `Look at Strength` variable.
- `Look at Clamp`: reduced from $65^\circ$ to $20^\circ$–$25^\circ$ for subtle, natural motion.
- `FInterp To` speed: reduced from $5.0$ to $1.5$–$2.0$ for soft, frame-rate-independent tracking.

**Architecture:**

```text
[ABP_ScoutTheCat]
  │
  ├── EventGraph
  │     └── Try Get Pawn Owner ──► Is Valid
  │                                    │
  │                                    ▼
  │                      Get Player Camera Manager ──► Is Valid
  │                                                        │
  │                                          [Aim Vector Calculation]
  │                                          [Hemisphere Gate + FInterp]
  │
  └── AnimGraph
        └── scoutthecat_cycle_walk (Loop: ON)
              └── Local To Component
                    └── Look At Node (bone: neck)
                          ├── Look At Location: Camera Location
                          ├── Alpha: Look at Strength
                          └── Clamp: 20°–25°
                          └── Component To Local ──► Output Pose
```

**Event Graph Logic:**

```text
[Get Camera Rotation] ──► [Get Forward Vector] ──► [× 2000.0]
[Get Camera Location] ─────────────────────────────────────────► [Add] ──► SET Camera Location
                                                                              │
[Camera Location] ──┐                                                         │
                    ├──► [Subtract] ──► [Normalize] ──► [Dot Product]         │
[Actor Location]  ──┘                                        │                │
                                                             ▼                │
[Actor Forward Vector] ──────────────────────────────► [> 0.1 Check]         │
                                                             │                │
                                                             ▼                ▼
                                                      [Select Float]──►[FInterp To]──► SET Look at Strength
                                                      (A=1.0, B=0.0)
```

#### Key Mistakes to Avoid

- **Pin Type Conversions:** Convert operand pins on math nodes to Float explicitly when scaling directional vectors — manual $(0, Y, 0)$ vector entries collapse 3D trajectories to a single axis.
- **Target Vector Consistency:** The projected aim point used for `Look At Location` must be identically fed into the hemisphere validation pipeline. Never mix raw camera origins with projected aim vectors.
- **Unvalidated Subsystem Managers:** Always route `Get Player Camera Manager` through `Is Valid` inside AnimBPs to avoid frame-0 null pointer exceptions.

#### Next Steps

1. Import Scout's base looping Idle animation and swap AnimGraph base pose from walk to idle.
2. Add velocity blending: `Try Get Pawn Owner` → `Get Velocity` → `Vector Length` → drive Blend Space 1D or State Machine.
3. Implement pupil/iris glance offsets beyond the $20^\circ$ neck clamp.

---

## 2026-08-28

### 🔍 Focus: Aim Tracking Calibration & Locomotion State Machine

#### Session Overview

- **Target:** `ABP_ScoutTheCat`
- **Objective:** Robust procedural head-tracking + modular locomotion framework (Idle ↔ Walk driven by velocity)

#### Tracking Fixes (Recap)

All vector math issues from 2026-08-27 resolved: flattened multiply fixed, inverted Select Float corrected, mismatched Subtract input re-routed. `Is Valid` guard added for Camera Manager null pointer.

#### Ground Speed Calculation Pipeline

Added `GroundSpeed` (Float) variable. Extracted every tick:

```text
Try Get Pawn Owner ──► Get Velocity ──► Vector Length ──► SET GroundSpeed
```

#### Locomotion State Machine

**Problem:** Empty `Locomotion` State Machine produced compiler warnings (`Entry node is not connected to state`).

**Solution:**

- Added **`Idle`** state (looping `scoutthecat_idle`) linked to `Entry`.
- Added **`Walk`** state (looping `scoutthecat_cycle_walk`).
- Transition rules: `GroundSpeed > 10.0` (Idle → Walk) and `GroundSpeed <= 10.0` (Walk → Idle).

**Full AnimBP Architecture:**

```text
[ABP_ScoutTheCat]
  │
  ├── EventGraph
  │     ├── Velocity pipeline ──► SET GroundSpeed
  │     └── Camera Manager ──► Is Valid ──► Aim Vector ──► FInterp ──► SET Look at Strength
  │
  └── AnimGraph
        └── [Locomotion State Machine]
              ├── Entry ──► Idle (scoutthecat_idle, looping)
              ├── Idle ──► Walk  (GroundSpeed > 10.0)
              └── Walk ──► Idle  (GroundSpeed ≤ 10.0)
              │
              ▼
        Local To Component
        Look At Node (bone: neck) ◄── Camera Location | Look at Strength | Clamp 20°–25°
        Component To Local ──► Output Pose
```

#### Key Mistakes to Avoid

- **Unconnected Math Outputs:** Always wire `Vector Length` output explicitly into a `SET` node — unwired pins evaluate to default `0.0`.
- **Unconnected State Machine Entry:** Wire the internal `Entry` node to a valid starting state before compiling.
- **State Machines vs. Blend Spaces:** State Machines = discrete behavioral logic (Idle/Walk/Jump). Blend Spaces = continuous multi-speed locomotion along a value axis.

#### Next Steps

1. PIE validation: Idle ↔ Walk transitions with concurrent head tracking.
2. Pupil/iris glance mechanics beyond $20^\circ$ neck clamp.
3. Optional Blend Space 1D upgrade if trot/sprint clips are introduced.

---

## Reference: Export Workflows

### 🔍 Scarf & Cape Export — Skeletal Mesh / Leader Pose

Scarf and cape are rigged and animated in Maya — they must deform with the character body. Use the **Modular Skeletal Mesh** workflow.

#### Key Difference from the Cap

- **Cap:** Rigid object attached to a single bone point. Does not bend.
- **Scarf & Cape:** Skinned to joints. Must deform with neck/spine rotation and play back Maya-authored animation.

#### Maya Export Requirements

- **Keep World Position:** Do NOT move scarf/cape to $0,0,0$. Keep them exactly where they are on the character.
- **Skeleton Match:** Export with the **exact same skeleton hierarchy** (or a subset) as the main character.
- **Selection Export:** Select scarf mesh + bound joints → `Export Selection` as `SK_Scarf.fbx`. Repeat for cape.

#### UE5 Setup — Leader Pose Workflow

1. **Import:** Import scarf and cape as **Skeletal Meshes**. In the import dialog, select the **Cat's Skeleton Asset** in the Skeleton dropdown.
2. **Add Components:** Open Cat Character Blueprint → add `ScarfMesh` and `CapeMesh` Skeletal Mesh Components.
3. **Assign Assets:** Set Skeletal Mesh assets on each component.
4. **Construction Script:** `Set Leader Pose Component` — Target: `ScarfMesh` + `CapeMesh`; New Leader Bone Component: main `Mesh (CharacterMesh0)`.

#### Performance Note

`Set Leader Pose Component` eliminates the cost of running three separate Animation Blueprints — the leader calculates bone positions once and accessories copy those positions.

#### Optional: Chaos Cloth

- **Maya Animations:** Scarf moves exactly as animated — full authorial control.
- **Chaos Cloth:** Paint physics-enabled mesh regions. Engine calculates dynamic draping (e.g., cape falls naturally when a character is knocked down).

---

### 🔍 Cap / Hat Export — Static Mesh / Socket

#### Option A: Socket Method (Recommended for Rigid Hats)

**Maya Setup:**

- Move hat to origin **(0,0,0)** — the world center becomes the pivot point in Unreal.
- Place pivot at the base of the hat (where it touches the head) for natural rotation.
- Do NOT bind to a skeleton.

**Export:** Select only the hat → export as `SM_Hat.fbx`.

**UE5 Setup:**

1. Open Skeletal Mesh asset → Skeleton Tree → right-click `head` bone → **Add Socket** → rename `S_HatSocket`.
2. Open Character Blueprint → add a **Static Mesh Component** parented to the Mesh.
3. In Component Details → set **Parent Socket** to `S_HatSocket`.
4. Set Mesh to `SM_Hat` — hat snaps to head bone. Fine-tune location/rotation in the Socket editor.

#### Coordinate Note

- **Option A (Socket):** Hat must be at $0,0,0$ — the socket acts as the new world zero for that object.
- **Option B (Skeletal):** Hat must stay in its natural world position — it shares the same coordinate space and origin as the body mesh.
