# Kung Fu Mania — Full Architecture Plan

---

## Engine

**Unity** with the **2D URP (Universal Render Pipeline)** template.

Unreal's 2D support (Paper2D) is underdeveloped and largely unmaintained. Unity is the proven
choice for 2D metroidvanias (Hollow Knight, Dead Cells, Ori) and its tooling aligns directly
with the needs of this project: hand-drawn sprite rigging, Cinemachine for boss cinematics,
Timeline for cutscene sequencing, and a mature 2D physics pipeline for hitbox/hurtbox work.

---

## 1. Project Structure

```
Assets/
├── _Project/
│   ├── Scripts/
│   │   ├── Core/           GameManager, EventBus, SceneLoader, ServiceLocator
│   │   ├── Player/         PlayerController, PlayerStateMachine, PlayerCombat, PlayerAbilities, PlayerStats
│   │   ├── Combat/         HitboxController, HurtboxController, ComboSystem, ParrySystem,
│   │   │                   DodgeSystem, DamageCalculator, StaggerMeter, InputBuffer
│   │   ├── Buffs/          BuffManager, BuffVisualController, GameTickManager, ActiveBuff
│   │   ├── Enemies/        EnemyAIBase, EnemyStateMachine, Bosses/, Variants/
│   │   ├── World/          RoomManager, RoomTransition, WorldStateManager, AbilityGate
│   │   ├── Cinematics/     CinematicDirector, LetterboxController
│   │   ├── Input/          InputReader (ScriptableObject)
│   │   ├── Audio/          AudioManager, MusicController, SFXCatalog
│   │   └── UI/             HUDController, HealthBar, StaminaBar, BossHealthBar,
│   │                       MinimapController, AbilityIcons, ComboDisplay
│   ├── Data/
│   │   └── ScriptableObjects/
│   │       ├── Abilities/
│   │       ├── Buffs/
│   │       ├── Enemies/
│   │       ├── Combos/
│   │       ├── Rooms/
│   │       └── ParrySettings/
│   └── Art/, Audio/, Prefabs/, Animations/
```

### Conventions
- All game scripts live under `_Project/` to sort above Unity built-ins.
- Every major system has a corresponding ScriptableObject data layer.
- Scenes split into persistent (Core) and additive (rooms).
- Prefabs are kept separate from art so programmers can iterate without touching sprite atlases.

---

## 2. Core State Machines

### Top-Level Game States

```
BOOT → MAIN_MENU → LOADING → GAMEPLAY
                              ↕
                           PAUSED
                           CINEMATIC → BOSS_INTRO / BOSS_KILL
                           GAME_OVER
```

Transitions driven by EventBus events, not direct method calls. States are C# classes
implementing `IGameState` (Enter / Update / Exit interface).

---

### Player: Two Concurrent Layers

The player runs two parallel state machines — Locomotion and Combat. This allows attacks while
moving or jumping without combinatorial state explosion.

#### Locomotion Layer

```
IDLE → WALK → RUN → JUMP → FALL → WALL_SLIDE → CROUCH → BLOCKING → HURT
```

`BLOCKING` is a held locomotion state entered when the defensive button is held and no parry
triggered on the initial press (see Combat System). Player can move slowly while blocking
(configurable). Releasing exits back to the appropriate locomotion state.

#### Combat Layer

```
NONE
ATTACK_1 / ATTACK_2 / ATTACK_3 / AERIAL_ATTACK
PARRY_STARTUP → PARRY_ACTIVE         → PARRY_RECOVERY
              ↘ PERFECT_PARRY
              ↘ PERFECT_PARRY_COUNTER (auto-fires when hasPerfectParryCounter = true)
DODGE → PERFECT_DODGE (if timed within perfectDodgeWindowBefore of impactFrame)
      → DODGE_RECOVERY
STAGGER
```

Each combat state is authored as a ScriptableObject containing:
- Startup / active / recovery frame counts
- Cancellable frame (the point at which combo input buffer and defensive cancels are checked)
- Hitbox reference
- Stamina cost
- `isDefensiveCancellable` flag
- `isComboBufferCancellable` flag

---

### Enemy State Machine

```
PATROL → DETECT → CHASE → ATTACK sub-states
                         → ATTACK_RECOVERY (after every attack, before next action)
                         → STAGGERED (stagger bar filled)
                         → GUARD / GUARD_BREAK  (canBlock enemies only)
                         → DEATH
```

`ATTACK_RECOVERY` is a mandatory post-attack state. The enemy cannot block, dodge, or chain
into another attack until it completes. Duration is defined per attack in `AttackPatternSO`
via `attackRecoveryFrames`. This prevents enemies from attacking and immediately retreating
into a guard, and is what makes parry counters always land cleanly.

**Blocking capability** is not universal. `EnemyDataSO` carries:

```csharp
bool canBlock    // default false — most standard enemies have no guard state
```

Only shield enemies, elites, and bosses set `canBlock = true`. Enemies where `canBlock = false`
never enter `GUARD` or `GUARD_BREAK` — those states are simply absent from their state machine.

#### Boss (extends standard)

```
PHASE_TRANSITION
PHASE_N_IDLE / PHASE_N_ATTACK_SET
ENRAGE
CINEMATIC_KILL
```

---

## 3. Combat System

### 3a. HitboxDataSO — Full Field List

```csharp
// Damage
float damage
float blockDamagePercent              // default 0.5 — damage dealt through block
bool  isUnblockable                   // bypasses block entirely, full damage, guard break
bool  piercesDodgeIFrames             // default false — i-frames don't protect; dodge movement still occurs
                                      // also prevents perfect dodge from triggering

// Dodge window (relative to impactFrame, only evaluated if !piercesDodgeIFrames)
int   perfectDodgeWindowBefore        // frames before impactFrame where a dodge counts as perfect

// Stagger
float staggerDamage                   // stagger buildup on a normal hit
float parryStaggerDamage              // stagger dealt when this attack is parried
float perfectParryStaggerMultiplier   // multiplier on parryStaggerDamage for perfect parry (e.g. 2.0)

// Parry windows (all relative to impactFrame)
// Window layout: [——standard——|——perfect——|IMPACT|——standard——]
int   impactFrame                     // frame the hitbox activates / damage is dealt
int   standardParryWindowBefore       // outer early window — standard parry (e.g. 7 frames)
int   perfectParryWindowBefore        // inner window closest to impact — perfect parry (e.g. 5 frames)
int   standardParryWindowAfter        // reaction window after impact — standard parry (e.g. 4 frames)

// Perfect parry counter (optional, configurable per attack)
bool          hasPerfectParryCounter          // default false — jabs just parry, haymakers counter
float         perfectParryCounterDamage       // damage dealt by the automatic counter hit
AnimationClip perfectParryCounterAnimation    // e.g. the 2-inch punch clip
bool          perfectParryCounterBulletTime   // default false — triggers slow-mo zoom on counter
float         perfectParryCounterTimeScale    // e.g. 0.2 (only used if bulletTime = true)
float         perfectParryCounterZoomAmount   // Cinemachine FOV delta (only used if bulletTime = true)

// Feel
float hitstunDuration
Vector2 knockbackVector
```

---

### 3b. Defensive System

One button. The timing of the **initial press** determines the outcome. Holding is the fallback.

| Press timing | Button released | Result |
|---|---|---|
| Within perfect window | Either | `PERFECT_PARRY` |
| Within standard window | Either | `PARRY_ACTIVE` |
| Outside window | Released quickly | Brief whiff, return to locomotion |
| Outside window | Held | `BLOCKING` (after 3–5 frame hold threshold) |

Pressing early and holding naturally falls into block — no punishment for trying to parry and
mistiming it, just a graceful fallback into a defensive stance.

#### Parry Recovery (Spam Deterrent)

After a **missed** parry attempt, a recovery window opens. The outcome depends on whether the
player holds the button:

```
Missed parry press — recovery timer starts
  ├─ Button held until timer completes → no lockout, clean exit to locomotion on release
  └─ Button released before timer completes → PARRY_RECOVERY (locked for remaining time)
```

Holding block through the recovery window waives the lockout entirely — committing to defense
is never punished, only spam-tapping is. Successful parries (PARRY_ACTIVE, PERFECT_PARRY) use
their own state durations as natural recovery; this system applies to **missed attempts only**.

`ParrySettingsSO`:
```csharp
float parryRecoveryDuration    // e.g. 0.4s — hold >= this = no lockout
```

#### DamageCalculator Resolution Order

```
if (isUnblockable && BLOCKING)
  → full damage + guard break animation

else if (PERFECT_PARRY && hasPerfectParryCounter)
  → 0 damage to player
  → trigger PERFECT_PARRY_COUNTER state (plays perfectParryCounterAnimation)
  → if perfectParryCounterBulletTime: drop timeScale, zoom camera, restore on counter end
  → deal perfectParryCounterDamage to attacker (attacker is in ATTACK_RECOVERY, cannot block)
  → apply parryStaggerDamage * perfectParryStaggerMultiplier to stagger bar

else if (PERFECT_PARRY)
  → 0 damage, counter window opens, parryStaggerDamage * perfectParryStaggerMultiplier

else if (PARRY_ACTIVE)
  → 0 damage, enemy staggered, parryStaggerDamage

else if (BLOCKING)
  → damage * blockDamagePercent

else
  → full damage, staggerDamage
```

---

### 3c. Parry Window — Impact Frame Based

Windows are relative to `impactFrame`, not the start of the attack animation. Each attack
defines its own window sizes — a slow telegraphed wind-up can have a generous perfect parry
window while a fast jab has a tight one.

Perfect parry is always **closest to impact**. Standard parry is the outer/earlier window.
Pressing early rewards reading the telegraph; pressing late (but before impact) rewards
reaction. The tightest, highest-reward zone is right before the hit lands.

```
Slow wind-up (standardBefore=7, perfectBefore=5, standardAfter=4):
[...startup animation...|——standard——|——perfect——|IMPACT|——std——|]
                          -12 to -6    -5 to -1           +1 to +4

Fast jab (standardBefore=3, perfectBefore=2, standardAfter=2):
[startup|——std——|—prf—|IMPACT|—std—|]
          -5 to -3  -2 to -1        +1 to +2
```

`ParrySystem` records the press timestamp. When a hit is registered it compares that timestamp
against the attacker's `impactFrame` timestamp to determine tier.

---

### 3d. Stagger Bar

Each enemy has a `StaggerMeter` component:

```csharp
float maxStagger
float currentStagger
float staggerDecayRate      // per second, decays when not being hit
float staggeredDuration     // how long STAGGERED state lasts
```

**Stagger sources:**
- Normal hit: `staggerDamage`
- Standard parry: `parryStaggerDamage`
- Perfect parry: `parryStaggerDamage * perfectParryStaggerMultiplier`
- Blocked hit: optional reduced stagger (e.g. `0.25x`), configurable

On `currentStagger >= maxStagger`: enemy enters `STAGGERED` (animation locked, takes bonus
damage), bar resets to 0.

---

### 3e. Combo System + Input Buffer

Combos are defined as `ComboSO` ScriptableObjects — an ordered list of `ComboStep` entries.

#### ComboStep Fields

```csharp
// Input
InputType requiredInput              // light, heavy, directional modifier
float     inputBufferWindow          // how early before the cancellable frame a press is accepted (e.g. 0.2s)
float     inputExpireWindow          // how long after the cancellable frame an input stays valid (e.g. 0.15s)

// State
string    nextStateName              // combat state to transition into
AnimationClip animationOverride      // combo-specific animation variant

// Cancels
bool isDefensiveCancellable          // default true — parry/block press immediately interrupts
bool isComboBufferCancellable        // default true — buffered combo input fires at cancellable frame

// Polish
bool hasCinematicFlourish            // brief zoom / impact freeze on this step
float staminaCost
```

#### Input Buffer

A timestamped ring buffer (16 entries) on the player stores inputs ahead of execution.

- **Press during non-cancellable frames** → stored in buffer, executes at the first valid
  (cancellable) frame as long as it hasn't expired
- **Press too early** → buffered and held until the cancellable frame (responsive feel,
  rewards rhythm without requiring frame-perfect timing)
- **Press too late / expired** → dropped, player must press again

This gives the "slightly sloppy but still rewarded" feel that makes combos feel frenetic
rather than clinical.

#### Cancel Priority (each frame at cancellable point)

```
During any attack:
  ├─ Parry/block pressed + isDefensiveCancellable = true  → immediate defensive state
  ├─ Parry/block pressed + isDefensiveCancellable = false → ignored, attack plays out
  ├─ Buffered attack input present + isComboBufferCancellable = true → fire next combo step
  └─ No valid input → attack plays to end, Combat Layer → NONE
```

Heavy finishers, grab attacks, and special moves set `isDefensiveCancellable = false` —
committing to them is a deliberate risk. Standard light/medium attacks default to cancellable,
keeping the flow frenetic: **attack → attack → parry → counter → attack**.

---

### 3f. Hit Pause

On any hit: `Time.timeScale = 0` for 2–6 frames (configurable per attack weight in
`HitboxDataSO`), paired with Cinemachine impulse shake and a chromatic aberration pulse.

---

### 3g. Dodge

**`DodgeSO`** full field list:

```csharp
// I-frames
int   iFrameStartFrame
int   iFrameEndFrame

// Movement
float totalDuration
AnimationCurve speedCurve
float staminaCost

// Recovery
int   recoveryFrames

// Perfect dodge
bool   perfectDodgeBulletTime          // default true
float  perfectDodgeTimeScale           // e.g. 0.2
float  perfectDodgeZoomAmount          // Cinemachine FOV delta
BuffSO perfectDodgeBuff                // buff applied on perfect dodge (assigned in Inspector)
```

- I-frames disable the player `HurtboxController` for the configured window. Attacks with
  `piercesDodgeIFrames = true` bypass this check entirely — the player can still dodge away
  but is not protected by i-frames.
- Position delta via direct transform + collision check (not physics) for precise frame control.
- `DODGE_RECOVERY` window after the dodge prevents spam.

**Perfect dodge** triggers when:
1. Dodge input fires within `perfectDodgeWindowBefore` frames of an incoming attack's `impactFrame`
2. The attacker's hitbox would have overlapped the player hurtbox (i-frames actually saved them)
3. `piercesDodgeIFrames = false` on the incoming attack

On perfect dodge:
```
PERFECT_DODGE state entered
  → timeScale drops to perfectDodgeTimeScale (if perfectDodgeBulletTime = true)
  → Cinemachine zooms by perfectDodgeZoomAmount
  → BuffManager.ApplyBuff(perfectDodgeBuff) on player
  → dodge animation completes at reduced timescale
  → timescale and camera restore
  → DODGE_RECOVERY
```

The buff's own `duration` governs the follow-up window. Any unlocked ability that requires
the buff checks `BuffManager.HasBuff()` independently — the dodge system has no knowledge
of what consumes it.

---

### 3h. Buff / Debuff System

Buffs and debuffs share the same architecture — a `BuffSO` with positive or negative effects,
applied to any entity carrying a `BuffManager` component (player, enemy, boss).

### BuffSO — Base Fields

```csharp
string    buffId                 // unique identifier, used for HasBuff() checks
string    displayName
Sprite    icon
float     duration               // seconds; -1 = permanent until explicitly removed
bool      stackable              // can multiple instances apply simultaneously
int       maxStacks
bool      isDebuff               // cosmetic flag for UI tinting, not logic
BuffVisualEffectSO visualEffect  // null = no visual change
```

Each `BuffSO` is self-contained. It subscribes to EventBus events **when applied** and
unsubscribes **when removed**. `BuffManager` owns the active list and lifetime — it does
not need to know what any individual buff does.

**Trigger examples:**
```
PoisonDebuffSO          → OnGameTick — deals damage every 10 ticks
DamageBoostBuffSO       → OnPlayerAttackLanded — applies damage multiplier for duration
PerfectDodgeReadyBuffSO → expires by duration; abilities poll HasBuff() each frame
BlockBreakDebuffSO      → OnPlayerHit — increases damage taken temporarily
```

### BuffManager

Component on any entity that can receive buffs (player, enemies, bosses):

```csharp
List<ActiveBuff> activeBuffs    // runtime: BuffSO + remaining duration + stack count

void ApplyBuff(BuffSO)          // adds instance, buff subscribes to its triggers
void RemoveBuff(string buffId)  // removes instance, buff unsubscribes
bool HasBuff(string buffId)     // queried by abilities, DamageCalculator, AI, etc.
```

Broadcasts via EventBus:
- `OnBuffApplied(buffId)` — for UI, audio, visual controller
- `OnBuffRemoved(buffId)` — for UI, visual controller cleanup
- `OnBuffStackChanged(buffId, stacks)` — for UI stack indicators

### GameTickManager

Lightweight singleton that fires `OnGameTick` at a fixed interval. Tick-based buffs subscribe
here rather than `Update`, keeping effects deterministic and frame-rate independent.

```csharp
int tickIntervalFrames    // FixedUpdate frames per game tick (e.g. 4)
event Action OnGameTick   // broadcast via EventBus
```

### BuffVisualEffectSO

Assigned per buff in the Inspector. `BuffVisualController` (component on the player and any
enemy that can receive visible buffs) listens to `OnBuffApplied` / `OnBuffRemoved` and
drives the visual response.

```csharp
Color      spriteColorTint       // e.g. green for poison, red for enrage
Material   materialOverride      // optional shader swap (glow, outline, dissolve)
GameObject particlePrefab        // optional looping particle effect, spawned as child
float      pulseSpeed            // > 0 = tint pulses rather than being static
```

**`BuffVisualController`** on apply:
- Lerps `SpriteRenderer.color` to `spriteColorTint`
- Swaps material if `materialOverride` is set (via `material.SetColor()` at runtime for
  shader-driven glow intensity)
- Spawns `particlePrefab` as a child GameObject

On remove:
- Lerps color back to default
- Restores original material
- Destroys particle instance

Multiple simultaneous buffs are handled by priority order or additive blending (configurable
per `BuffVisualEffectSO`).

---

## 4. Metroidvania Map / Scene Management

### Scene Structure

- **Persistent Core scene** always loaded: GameManager, AudioManager, InputReader, EventBus,
  UI Canvas
- **Room scenes** loaded additively via `Addressables`; old room unloads after transition

```
Naming convention:
  Room_Zone01_Entrance
  Room_Zone01_Temple_A
  Boss_Zone01_TigerSensei
  Cinematic_Boss_TigerSensei_Kill
```

### Room Transitions

1. Player enters `RoomTransition` trigger zone (BoxCollider2D trigger)
2. EventBus fires `OnRoomTransitionRequest(transitionData)`
3. Game state → `LOADING`, ink-splat wipe plays
4. New scene loads additively, player repositioned to matching entrance point
5. Old scene unloads, game state → `GAMEPLAY`

`RoomTransitionData`: target scene name, target entrance ID, transition animation type,
music crossfade flag.

### World State Persistence

`WorldStateManager` persists across loads, maintains:

```csharp
Dictionary<string, RoomState>   // dead enemies, pickups, open doors per room
HashSet<string>                 // unlockedAbilities
Dictionary<string, bool>        // worldFlags
PlayerPersistentData            // health, position, current room
```

Serialized to JSON via `Newtonsoft.Json` on: room transition, boss death, respawn point
activation.

On room load, `WorldStateManager` broadcasts `OnRoomStateRestored` and individual room
objects self-configure via their own EventBus listeners.

### Ability Gates

Each `AbilityGate` holds a `RequiredAbilitySO`. Queries `WorldStateManager.HasAbility()` on
room load and on `OnAbilityUnlocked` events. Opens with an animation and disables its collider.

**Planned abilities:**
- `WALL_JUMP`
- `DOUBLE_JUMP`
- `DASH_UPGRADE`
- `CHI_BLAST`
- `HIGH_KICK` (breaks cracked ceilings)
- `IRON_BODY` (withstands environmental hazards)
- `WIRE_FU` (grapple to ceiling anchor rings)

---

## 5. Boss System

### Phase Controller

Each boss holds a `List<BossPhaseDataSO>`. Each phase defines:

```csharp
float             hpThreshold                  // e.g. 0.66, 0.33
List<AttackPatternSO> attackPool
MovementBehavior  movementBehavior
float             musicPhaseParameter
bool              playCinematicOnEntry
```

Attack selection: weighted random from pool filtered by phase + player conditions (proximity,
airborne state), with last-used cooldown to prevent repetition.

### Phase / Kill Cinematic Flow

```
HP threshold hit
  → Boss enters PHASE_TRANSITION
    → BossCinematicTrigger fires OnCinematicStart
      → Input locked, letterbox animates in
        → PlayableDirector runs Timeline
          (camera blends, character animations, audio cue)
            → OnCinematicComplete → new phase begins

HP = 0
  → CINEMATIC_KILL state (suppresses standard death)
    → Slow-mo final blow
      → Freeze frame (timeScale = 0, ~0.5s)
        → Victory pose hold
          → Ability drop, WorldState update, respawn point set
```

---

## 6. Boss Health Bar

**Position:** top-center of screen.

**Structure:** one full-width bar per phase. No segments. The bar represents current phase HP
only, draining as the player deals damage within that phase.

**Two stacked `Image` fills:**
- `backgroundImage` — always full width, tinted the **next** phase color
- `foregroundImage` — Fill Origin: Left, fill amount = `currentPhaseHP / maxPhaseHP`,
  tinted the **current** phase color

As the foreground drains left-to-right, the background color is revealed beneath. When the
phase threshold is hit the bar is entirely the background color — that color naturally becomes
the next phase's foreground with no separate animation needed.

**On `OnBossPhaseChange(int newPhase)`:**
1. Swap foreground color to old background color
2. Set new background color to next-next phase color
3. Reset foreground fill to 1.0

| Phase | Foreground (current HP) | Background (next phase preview) |
|---|---|---|
| Phase 1 | Red | Yellow |
| Phase 2 | Yellow | Green |
| Phase 3 | Green | Dark / black |

---

## 7. Input System

`InputReader` is a **ScriptableObject** wrapping `PlayerInputActions` (`.inputactions` asset).
Exposes C# events (not UnityEvents). Passed via Inspector — no singleton coupling.

```csharp
event Action          OnAttackLight
event Action          OnAttackHeavy
event Action          OnDefensivePress      // parry evaluation happens here
event Action          OnDefensiveRelease    // exits BLOCKING
event Action          OnDodge
event Action          OnJump
event Action          OnJumpCancelled
event Action<Vector2> OnMove
event Action          OnInteract
event Action          OnPause
event Action          OnAbility1
event Action          OnAbility2
```

`OnDefensiveHold` is not needed — block state is entered when press is outside the parry
window and the button remains held past the 3–5 frame threshold.

Control remapping via `InputActionRebindingExtensions` API. Bindings saved as JSON in the
save file.

---

## 8. Audio

**FMOD Studio** (`com.fmod.unity`) strongly preferred over Unity's Audio Mixer:
- Parameter-driven adaptive music reacts to combat intensity in real time
- Snapshot system handles per-zone reverb (temple halls, caves, outdoors) with crossfading
- Stinger events for phase transitions without audio seams

### Music Parameters

```
Combat_Intensity   float 0.0–1.0   ramps up on combat start, decays ~8s after combat ends
Phase              int   0–3        set on boss phase change / room entry
```

### SFX Categories

- Combat: sword swings, fist impacts (cloth / flesh / bone variants), parry clang,
  chi blast, kick impacts
- Footsteps: surface material parameter (stone, wood, grass, bamboo)
- Voice: grunts, exertion, pain — supports multiple voice packs
- Ambient: wind, temple bells, insects, distant crowds
- Cinematic: dramatic sting, whoosh reveal, freeze-frame impact boom

`SFXCatalog` ScriptableObject maps string keys to FMOD event paths. All calls go through
`AudioManager.PlaySFX(string key)`.

---

## 9. UI / HUD

### Canvas Architecture

- `WorldSpaceCanvas` (Screen Space - Camera): floating damage numbers, enemy health bars,
  stagger bars — follow world positions
- `ScreenSpaceCanvas` (Screen Space - Overlay): player HUD, fixed to screen

### HUD Elements

| Element | Notes |
|---|---|
| Health bar | Segmented ink-brush style, ghost-health trail lags behind on damage |
| Stamina / Chi | Arc-shaped, flashes when empty, recharges via fill shader |
| Boss bar | Top-center, full-width, phase color system described above |
| Stagger bar | World-space above enemy, fills on hits and parries |
| Minimap | Data-driven from `RoomDataSO` polygons, fog of war, current room pulses |
| Combo counter | Brushstroke font, white → gold → red color tiers, fades 2s after last hit |
| Ability icons | Cooldown clock fill when unavailable, unlock slide-in animation |

### Screen Effects

- Letterbox bars animate in during cinematics (`LetterboxController`)
- Persistent film grain (mild during gameplay, intensified on kill cinematic)
- Chromatic aberration pulse on hard impacts
- Ink-splat full-screen wipe on room transitions

---

## 10. Recommended Unity Packages

| System | Package |
|---|---|
| Input | `com.unity.inputsystem` |
| Camera + shake | `com.unity.cinemachine` |
| Cutscene sequencing | `com.unity.timeline` |
| Sprite rigging | `com.unity.2d.animation` |
| Tilemaps | `com.unity.2d.extras` |
| Post-processing | URP built-in |
| Audio | FMOD for Unity |
| Save serialization | `com.unity.nuget.newtonsoft-json` |
| Async scene loading | `com.unity.addressables` |
| Enemy pathfinding | A* Pathfinding Project (Aron Granberg) |

---

## 11. Key Architectural Principles

**EventBus over direct coupling.** Systems communicate by broadcasting typed events.
`EventBus` is a static generic class: `Subscribe<T>`, `Unsubscribe<T>`, `Publish<T>`.
A boss death should not require `BossAI` to know about `AudioManager`.

**ScriptableObject data layer.** All tunable values live in ScriptableObjects. Designers
iterate on hitbox data, combo definitions, attack patterns, and parry windows without touching
code.

**Additive scene loading.** The persistent Core scene never unloads. Room scenes load and
unload additively. AudioManager and GameManager survive room transitions without
`DontDestroyOnLoad` hacks.

**Frame-data-driven combat.** Startup, active, recovery, cancellable frames, and parry windows
are authored as data. The combo and parry systems have a single source of truth. Balancing
happens in the Inspector, not in code.

**Cinematic state isolation.** When `CINEMATIC` game state is active, non-cinematic gameplay
events are blocked or queued. `CinematicDirector` is the sole authority and signals completion
explicitly.

---

## 12. Build Order (Critical Path)

| Order | File | Why first |
|---|---|---|
| 1 | `EventBus.cs` | Everything communicates through this |
| 2 | `PlayerStateMachine.cs` | Gates all combat work |
| 3 | `HitboxController.cs` / `HurtboxController.cs` | Damage pipeline |
| 4 | `InputBuffer.cs` | Required before combo system |
| 5 | `StaggerMeter.cs` | Required before combat tuning |
| 6 | `GameTickManager.cs` | Required before any tick-based buff |
| 7 | `BuffManager.cs` + `BuffVisualController.cs` | Required before dodge, abilities, or status effects |
| 8 | `WorldStateManager.cs` | Room persistence and ability unlocks |
| 9 | `CinematicDirector.cs` | Required before any boss content |
