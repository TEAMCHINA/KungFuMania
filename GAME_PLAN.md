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
│   │   ├── Player/         PlayerController, PlayerStateMachine, PlayerCombat, PlayerAbilities, PlayerStats,
│   │   │                   AnimatorSpeedSync
│   │   ├── Combat/         HitboxController, HurtboxController, ComboSystem, ParrySystem,
│   │   │                   DodgeSystem, DamageCalculator, StaggerMeter, InputBuffer,
│   │   │                   MotionInputBuffer, MotionInputDetector
│   │   ├── Auras/          AuraManager, AuraVisualController, GameTickManager, ActiveAura
│   │   ├── Stats/          StatSheet, StatRegistry
│   │   ├── Abilities/      AbilityBase, AbilityExecutionContext, AbilityModifierSO, EquipmentManager
│   │   ├── MovementAbilities/ DoubleJumpAbility, AirDashAbility, GrappleAbility
│   │   ├── Cosmetics/      CharacterCustomizationController, CosmeticRegistry
│   │   ├── Enemies/        EnemyAIBase, EnemyStateMachine, Bosses/, Variants/
│   │   ├── World/          RoomManager, RoomTransition, WorldStateManager, AbilityGate, GrapplePoint
│   │   ├── Cinematics/     CinematicDirector, LetterboxController
│   │   ├── Input/          InputReader (ScriptableObject)
│   │   ├── Audio/          AudioManager, MusicController, SFXCatalog
│   │   └── UI/             HUDController, HealthBar, StaminaBar, BossHealthBar,
│   │                       MinimapController, AbilityIcons, ComboDisplay
│   ├── Data/
│   │   └── ScriptableObjects/
│   │       ├── Abilities/
│   │       ├── AbilityModifiers/
│   │       ├── MotionInputs/       MotionPatternSO assets, StickInputConfigSO
│   │       ├── Auras/
│   │       ├── Cosmetics/      CosmeticSlotSO assets, CosmeticOptionSO assets, CosmeticRegistry
│   │       ├── Enemies/
│   │       ├── Combos/
│   │       ├── Rooms/
│   │       ├── Stats/              StatConfigSO (dexterity coefficients, level scaling curve, etc.)
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
                                                                   → DOWN → DOWN_RECOVERY
```

`BLOCKING` is a held locomotion state entered when the defensive button is held and no parry
triggered on the initial press (see Combat System). Player can move slowly while blocking
(configurable). Releasing exits back to the appropriate locomotion state.

`DOWN` is entered when a knockdown attack lands (see 3a `onHitTargetState` and 3m).
`DOWN_RECOVERY` is the get-up animation; input during DOWN can trigger an early tech roll
that skips to a brief neutral recovery stance instead.

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
                         → DOWN → DOWN_RECOVERY (knockdown attacks)
                         → GUARD / GUARD_BREAK  (canBlock enemies only)
                         → DEATH
```

`ATTACK_RECOVERY` is a mandatory post-attack state. The enemy cannot block, dodge, or chain
into another attack until it completes. Duration is defined per attack in `AttackPatternSO`
via `attackRecoveryFrames`. This prevents enemies from attacking and immediately retreating
into a guard, and is what makes parry counters always land cleanly.

#### AttackPatternSO — Key Fields

```csharp
HitboxDataSO hitboxData              // damage, parry windows, stagger values
AnimationClip attackAnimation
int     attackRecoveryFrames         // mandatory cooldown after the attack completes
bool    superArmor                   // default false — taking damage does not interrupt;
                                     // full damage applies, stagger builds normally;
                                     // stagger bar filling is the ONLY interrupt — see 3d
float   usageWeight                  // relative probability in weighted random selection
float   lastUsedCooldown             // seconds before this attack can be selected again
```

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
// Damage formula (Option 2 — multi-stat weighted)
// FinalDamage = ((Strength * strengthWeight) + (Chi * chiWeight) + (WeaponDamage * weaponWeight))
//               * baseMultiplier * levelScale * contextModifiers
float strengthWeight                  // physical contribution (e.g. punch = 0.9, chi blast = 0.1)
float chiWeight                       // chi/energy contribution (e.g. punch = 0.0, chi blast = 1.2)
float weaponWeight                    // weapon contribution (e.g. sword strike = 0.8, kick = 0.0)
float baseMultiplier                  // per-attack scalar (e.g. light = 0.8, flying kick = 1.4, finisher = 2.2)
float armorPenetration                // 0.0–1.0, default 0 — fraction of target Armor ignored
                                      // e.g. 0.5 = ignores half of target's armor

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

// Target state precondition — checked before damage resolution; whiffs if not satisfied
TargetStateRequirement targetRequirement  // default None — no restriction
                                          // see enum below; use flags for multi-state attacks

// On-hit result — determines which hurt state the target enters
OnHitTargetState onHitTargetState        // default Hurt — see enum below

// Feel
float hitstunDuration
Vector2 knockbackVector
```

**`TargetStateRequirement`** — C# flags enum, checked against target's current locomotion state:

```csharp
[System.Flags]
enum TargetStateRequirement {
    None      = 0,       // no restriction — hitbox lands against any state
    Grounded  = 1 << 0,  // target must be grounded (not in JUMP/FALL/airborne)
    Airborne  = 1 << 1,  // target must be airborne (JUMP or FALL)
    Crouching = 1 << 2,  // target must be in CROUCH
    Standing  = 1 << 3,  // target must be in IDLE/WALK/RUN (upright, not crouching, not airborne)
    Downed    = 1 << 4,  // OTG — target must be in DOWN state; normal hits whiff vs. downed targets
}
// Flags can combine: Grounded | Standing means standing (not crouching, not airborne)
// Omitting Downed from any Grounded attack means it won't auto-OTG — intentional default
```

**`OnHitTargetState`** — determines which state the target enters when hit:

```csharp
enum OnHitTargetState {
    Hurt,       // standard hitstun — target enters HURT, plays pain animation
    Down,       // knockdown — target enters DOWN (floor tumble/slide), then DOWN_RECOVERY
    Launched,   // juggle starter — target enters FALL with strong upward knockback vector
}
// New states (Crumple, WallBounce, etc.) are added here without touching HitboxDataSO logic
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
// Precondition check — runs before any damage or state resolution:
if (!attack.targetRequirement.Satisfies(target.currentLocomotionState))
  → whiff; hitbox overlap is silently ignored; no damage, no stagger, no state change
  // e.g. command throw whiffs vs. airborne; low sweep whiffs vs. jumping; OTG whiffs vs. standing

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
  → RawDamage = damage * blockDamagePercent

else
  → RawDamage = full damage, staggerDamage

// Armor applied last, after all parry/block resolution, for both player and enemy targets:
EffectiveArmor = targetStats.Armor * (1 - attack.armorPenetration)
FinalDamage    = max(1, RawDamage - EffectiveArmor)
// max(1) ensures at least 1 damage always lands — attacks never feel completely negated

// Interrupt resolution — runs after damage and stagger are applied:
if (staggerBarJustFilled)
  → STAGGERED — overrides super armor; deactivate all hitboxes immediately (no trade)
else if (target.currentAttack.superArmor)
  → animation continues; hurt state suppressed; damage and stagger already applied above
else
  → target enters attack.onHitTargetState:
      Hurt     → HURT (standard hitstun)
      Down     → DOWN (knockdown floor animation, then DOWN_RECOVERY)
      Launched → FALL with upward knockback vector (juggle)
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

On `currentStagger >= maxStagger`: entity enters `STAGGERED` (animation locked, takes bonus
damage), bar resets to 0.

#### Super Armor and Stagger

An attack flagged `superArmor = true` (on `ComboStep` or `AttackPatternSO`) absorbs
incoming damage without interrupting the animation. Every hit taken during the armor window
still builds stagger normally. When the bar fills:

1. `STAGGERED` fires immediately — armor breaks mid-attack with no delay
2. All active hitboxes deactivate instantly — the hit that caused the stagger break does
   **not** trade; the staggered entity cannot land their pending hit
3. The capitalize window opens clean — punish without eating a counter-hit

Super armor is a commitment mechanic, not a get-out-of-jail card. Opponents can race to
fill the stagger bar rather than retreating.

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
bool canCancelIntoSpecial            // default false — attack can 2-in-1 cancel into a motion input
                                     // special ON HIT only; requires advancedCombatActive naturally
                                     // since specials already gate on it

// Armor
bool superArmor                      // default false — taking damage does not interrupt this attack
                                     // full damage still applies; stagger still builds normally
                                     // stagger bar filling is the ONLY interrupt — see 3d

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
  ├─ Damage received + superArmor = false → HURT state (attack cancelled immediately)
  ├─ Damage received + superArmor = true  → damage/stagger applied; animation continues
  │   (stagger bar filling overrides super armor → immediate STAGGERED, hitboxes off)
  ├─ Parry/block pressed + isDefensiveCancellable = true  → immediate defensive state
  ├─ Parry/block pressed + isDefensiveCancellable = false → ignored, attack plays out
  ├─ Buffered attack input present + isComboBufferCancellable = true → fire next combo step
  └─ No valid input → attack plays to end, Combat Layer → NONE
```

Heavy finishers, grab attacks, and special moves set `isDefensiveCancellable = false` —
committing to them is a deliberate risk. Standard light/medium attacks default to cancellable,
keeping the flow frenetic: **attack → attack → parry → counter → attack**.

#### 2-in-1 Cancel (SF2-Style)

When `canCancelIntoSpecial = true` on a `ComboStep`, the attack can be cancelled into a
motion input special **on hit only**. This is the Street Fighter 2 "2-in-1" cancel — the
special fires during the active or early recovery frames of the attack, but only if the
hit actually connected.

**How it works:**

```
Attack with canCancelIntoSpecial = true swings:
  ├─ Hit connects (HurtboxController overlap confirmed)
  │   → hitConnected flag set for this ComboStep
  │   → MotionInputDetector checks buffer for a valid motion pattern
  │     ├─ Pattern matched + advancedCombatActive = true → special fires, cancels recovery
  │     └─ No match → attack plays out normally
  └─ Whiff (no hit confirmed)
      → canCancelIntoSpecial is ignored; attack plays to end
```

The `advancedCombatActive` requirement is **not an additional gate** — it is already
enforced naturally, since only motion input specials are valid cancel targets, and those
already require `advancedCombatActive` to detect. Players without `ADVANCED_COMBAT`
unlocked simply cannot perform the cancel, with no extra logic needed.

**Design intent:**
- Whiff punishing is preserved — specials cannot be thrown out safely without landing
- Rewards players who learn which normals cancel into which specials
- Baked-in combo chains (`isComboBufferCancellable`) chain naturally and do not use this
  system; 2-in-1 is exclusively for chaining a normal into a motion special mid-combo

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
AuraSO perfectDodgeAura                // aura applied on perfect dodge (assigned in Inspector)
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
  → AuraManager.ApplyAura(perfectDodgeAura) on player
  → dodge animation completes at reduced timescale
  → timescale and camera restore
  → DODGE_RECOVERY
```

The aura's own `duration` governs the follow-up window. Any unlocked ability that requires
the aura checks `AuraManager.HasAura()` independently — the dodge system has no knowledge
of what consumes it.

---

### 3h. Aura System

Auras share the same architecture — a `AuraSO` with positive or negative effects,
applied to any entity carrying a `AuraManager` component (player, enemy, boss).

### AuraSO — Base Fields

```csharp
string    auraId                 // unique identifier, used for HasAura() checks
string    displayName
Sprite    icon
float     duration               // seconds; -1 = permanent until explicitly removed
bool      stackable              // can multiple instances apply simultaneously
int       maxStacks
bool      isNegative               // cosmetic flag for UI tinting, not logic
AuraVisualEffectSO visualEffect  // null = no visual change
```

Each `AuraSO` is self-contained. It subscribes to EventBus events **when applied** and
unsubscribes **when removed**. `AuraManager` owns the active list and lifetime — it does
not need to know what any individual aura does.

**Trigger examples:**
```
PoisonAuraSO          → OnGameTick — deals damage every 10 ticks
DamageBoostAuraSO       → OnPlayerAttackLanded — applies damage multiplier for duration
PerfectDodgeReadyAuraSO → expires by duration; abilities poll HasAura() each frame
BlockBreakAuraSO      → OnPlayerHit — increases damage taken temporarily
```

### AuraManager

Component on any entity that can receive auras (player, enemies, bosses):

```csharp
List<ActiveAura> activeAuras    // runtime: AuraSO + remaining duration + stack count

void ApplyAura(AuraSO)          // adds instance, aura subscribes to its triggers
void RemoveAura(string auraId)  // removes instance, aura unsubscribes
bool HasAura(string auraId)     // queried by abilities, DamageCalculator, AI, etc.
```

Broadcasts via EventBus:
- `OnAuraApplied(auraId)` — for UI, audio, visual controller
- `OnAuraRemoved(auraId)` — for UI, visual controller cleanup
- `OnAuraStackChanged(auraId, stacks)` — for UI stack indicators

### GameTickManager

Lightweight singleton that fires `OnGameTick` at a fixed interval. Tick-based auras subscribe
here rather than `Update`, keeping effects deterministic and frame-rate independent.

```csharp
int tickIntervalFrames    // FixedUpdate frames per game tick (e.g. 4)
event Action OnGameTick   // broadcast via EventBus
```

### AuraVisualEffectSO

Assigned per aura in the Inspector. `AuraVisualController` (component on the player and any
enemy that can receive visible auras) listens to `OnAuraApplied` / `OnAuraRemoved` and
drives the visual response.

```csharp
Color      spriteColorTint       // e.g. green for poison, red for enrage
Material   materialOverride      // optional shader swap (glow, outline, dissolve)
GameObject particlePrefab        // optional looping particle effect, spawned as child
float      pulseSpeed            // > 0 = tint pulses rather than being static
```

**`AuraVisualController`** on apply:
- Lerps `SpriteRenderer.color` to `spriteColorTint`
- Swaps material if `materialOverride` is set (via `material.SetColor()` at runtime for
  shader-driven glow intensity)
- Spawns `particlePrefab` as a child GameObject

On remove:
- Lerps color back to default
- Restores original material
- Destroys particle instance

Multiple simultaneous auras are handled by priority order or additive blending (configurable
per `AuraVisualEffectSO`).

---

### 3i. Stat System

Stats are split into two categories: **primary stats** (innate to the character, grow via
levelling and upgrades) and **equipment stats** (contributed by equipped items). Both feed
into derived values used at runtime by the damage formula, health system, and chi pool.

#### Primary Stats

| Stat | Intent |
|---|---|
| `Strength` | Scales physical attack damage — punches, kicks, weapon strikes |
| `Chi` | Scales chi/energy attack damage; also governs max chi pool size |
| `Dexterity` | Raw agility stat — drives AttackSpeedMultiplier and MovementSpeedMultiplier as computed properties |
| `Constitution` | Governs max health pool size |

#### Equipment Stats (player only — sourced from equipped items, not innate)

| Stat | Intent |
|---|---|
| `WeaponDamage` | Flat damage contribution from the equipped weapon; 0 when unarmed |
| `Armor` | Flat damage reduction applied to incoming hits; 0 when no armor equipped |

> **Enemies** do not have an inventory. Both `WeaponDamage` and `Armor` are native stats
> on `EnemyDataSO`, authored directly in the Inspector alongside Strength, Chi, etc.

#### Derived Stats (computed, never set directly)

| Derived | Formula (intent — coefficients TBD during balancing) |
|---|---|
| `MaxHealth` | f(Constitution, Level) |
| `MaxChiPool` | f(Chi, Level) |
| `AttackSpeedMultiplier` | f(Dexterity) — read by `AnimatorSpeedSync` to set `animator.speed`; also scales recovery frame durations |
| `MovementSpeedMultiplier` | f(Dexterity) — read by `PlayerController` to scale movement velocity |
| `LevelScale` | f(Level) — global damage scalar applied to all attacks |

#### StatSheet

`StatSheet` is a component on the player (and on enemies for their own damage formulas).
It holds the raw primary stat values and exposes computed derived values as properties.
Equipment bonuses are aggregated into a separate `equipmentBonuses` struct and added
on top at read time — equipping and unequipping items never mutates base stats.

```csharp
// Primary (base values, modified by level-up and permanent upgrades)
int strength
int chi
int dexterity
int constitution
int level

// Equipment overlay (player only — aggregated from all equipped items by EquipmentManager)
// Enemies leave this zeroed out; their WeaponDamage and Armor are set directly as native stats
StatBonus equipmentBonuses     // additive on top of base stats
                               // contains: weaponDamage, armor (and future equipment stats)

// Derived (computed properties — never set directly, always calculated from raw stats)
float MaxHealth                  // => f(constitution, level)
float MaxChiPool                 // => f(chi, level)
float AttackSpeedMultiplier      // => 1f + (dexterity * attackSpeedCoefficient)
float MovementSpeedMultiplier    // => 1f + (dexterity * movementSpeedCoefficient)
float LevelScale                 // => f(level)
int   WeaponDamage               // player: from equipmentBonuses; enemy: native stat, set directly
int   Armor                      // player: from equipmentBonuses; enemy: native stat, set directly
// coefficients are configurable on a StatConfigSO, not hardcoded
```

`StatSheet` broadcasts `OnStatsChanged` via EventBus whenever any value changes, allowing
health bars, chi bars, and UI to react without polling.

---

### 3j. Ability Execution Pipeline

Abilities are fully data-driven and unaware of how equipment or upgrades might modify them.
At execution time, a default context is built from the ability's own data, passed through
a modifier pipeline registered by `EquipmentManager`, and then executed against the
final mutated values.

#### AbilityExecutionContext

Plain mutable C# class (not a ScriptableObject — created and discarded per execution):

```csharp
// Damage formula inputs (defaults from AbilitySO / HitboxDataSO, freely mutable)
float  strengthWeight
float  chiWeight
float  weaponWeight
float  baseMultiplier
int    hitCount                 // default 1 — AddHitModifier increments this
float  armorPenetration         // 0.0–1.0, default from HitboxDataSO — equipment can raise this

// Timing
float  hitstunDuration          // can be extended or reduced by modifiers

// On-hit effects
List<AuraSO> onHitAuras       // applied to target on each hit
AuraSO       selfAuraOnHit      // applied to caster on hit (null = none)

// Read-only references (modifiers may read these to compute values)
StatSheet    casterStats
string       abilityId
```

#### AbilityModifierSO

Abstract ScriptableObject. Item and upgrade designers subclass this — one class per modifier
type. The ability pipeline calls `Modify(ctx)` on each registered modifier in order:

```csharp
public abstract void Modify(AbilityExecutionContext ctx);
```

**Example concrete modifier types:**

| Class | What it does to context |
|---|---|
| `DamageMultiplierModifierSO` | `ctx.baseMultiplier *= multiplier` |
| `AddHitModifierSO` | `ctx.hitCount += count` |
| `ApplyAuraOnHitModifierSO` | `ctx.onHitAuras.Add(aura)` |
| `ShiftStatWeightModifierSO` | adjusts chi/strength/weapon weights (e.g. make flying kick chi-scaled) |
| `ExtendHitstunModifierSO` | `ctx.hitstunDuration += seconds` |
| `ApplySelfAuraOnHitModifierSO` | `ctx.selfAuraOnHit = aura` |
| `ChargeCountModifierSO` | contributes `bonusCharges` to `EquipmentManager.GetChargeBonus(abilityId)` — affects charge-based abilities (double jump, air dash) at reset time, not execution time |

New modifier behaviors never require touching ability code — create a new subclass and assign
it to an item asset.

#### EquipmentManager

Singleton (or ServiceLocator-resolved) component on the player. Owns the modifier registry
and handles equip/unequip lifecycle:

```csharp
// Registry: abilityId → modifiers contributed by currently equipped items
Dictionary<string, List<AbilityModifierSO>> modifiersByAbilityId

// Charge bonus registry: abilityId → total bonus charges from all equipped items
// Updated incrementally on equip/unequip — not recomputed each frame
Dictionary<string, int> chargeBonuses

void RegisterModifiers(EquipmentSO item)          // called on equip; also adds ChargeCountModifierSO contributions to chargeBonuses
void UnregisterModifiers(EquipmentSO item)         // called on unequip; subtracts contributions

// Called by ability at execution time
AbilityExecutionContext BuildContext(string abilityId, AbilityExecutionContext defaults)

// Called by charge-based abilities when restoring charges (e.g. on landing)
int GetChargeBonus(string abilityId)              // returns 0 if no contributions registered
```

`BuildContext` clones `defaults`, iterates registered modifiers for `abilityId` in order,
calls `Modify(ctx)` on each, and returns the final context. The ability never sees the
modifier list — it just receives the finished context.

#### Execution Flow

```
Player activates ability
  → ability builds default AbilityExecutionContext from its own AbilitySO/HitboxDataSO data
  → EquipmentManager.BuildContext(abilityId, defaults) called
      → each registered AbilityModifierSO mutates context in order
  → ability executes using final context values
  → DamageCalculator.Compute(ctx, casterStats, targetStats) called per hit
      → BaseDamage     = (Strength * ctx.strengthWeight)
                       + (Chi * ctx.chiWeight)
                       + (WeaponDamage * ctx.weaponWeight)
      → RawDamage      = BaseDamage * ctx.baseMultiplier * LevelScale * blockModifier
      → EffectiveArmor = targetStats.Armor * (1 - attack.armorPenetration)
      → FinalDamage    = max(1, RawDamage - EffectiveArmor)
      // applies identically whether target is player or enemy
  → onHitAuras applied to target, selfAuraOnHit applied to caster
```

---

### 3k. Movement Ability Plugin System

Movement abilities follow the same philosophy as `AbilityModifierSO` — they are
self-contained components; the base player systems have no knowledge of them. Each ability
is a `MonoBehaviour` added to the player prefab. The ability checks its own unlock condition,
manages its own state, and calls exposed hooks on `PlayerController` to affect movement.

Adding a new movement ability never requires touching `PlayerStateMachine` or
`PlayerController`.

#### PlayerController Hooks

`PlayerController` exposes a minimal set of hooks. It has no knowledge of which ability
components are installed:

```csharp
void    ApplyImpulse(Vector2 force)             // adds velocity — double jump, air dash, grapple pull
void    ForceLocomotionState(string stateId)    // overrides current locomotion state
Vector2 GetVelocity()                           // read current velocity
bool    IsGrounded()                            // ground contact query
bool    IsAirborne()                            // convenience inverse
```

#### Double Jump (`DoubleJumpAbility`)

```csharp
int   maxExtraJumps       // base value, default 1
int   remainingJumps      // runtime counter; restored on landing
float jumpImpulseForce
```

- Listens to `OnJump` from `InputReader`
- Only acts when `IsAirborne()` and `remainingJumps > 0`
- Calls `playerController.ApplyImpulse(Vector2.up * jumpImpulseForce)`
- Decrements `remainingJumps`; on `OnPlayerLanded`:
  `remainingJumps = maxExtraJumps + EquipmentManager.GetChargeBonus("DOUBLE_JUMP")`

#### Air Dash (`AirDashAbility`)

```csharp
int     maxAirDashes        // base value, default 1
int     remainingAirDashes  // runtime counter; restored on landing
DodgeSO airDashData         // reuses existing DodgeSO — same i-frames, speed curve, recovery
```

- Listens to `OnDodge` from `InputReader`
- Only acts when `IsAirborne()` and `remainingAirDashes > 0`
- Delegates entirely to `DodgeSystem` using `airDashData`
- Same i-frame and perfect dodge logic applies unchanged
- On `OnPlayerLanded`:
  `remainingAirDashes = maxAirDashes + EquipmentManager.GetChargeBonus("AIR_DASH")`

#### Grapple / Hookshot (`GrappleAbility` — `WIRE_FU`)

Implemented as a strong directional pull — no pendulum physics, no new locomotion states.
`JUMP` and `FALL` handle the player throughout; the ability stays a pure plugin.

```csharp
float     maxGrappleRange
LayerMask grapplePointLayer
float     launchImpulse          // strength of the pull toward the anchor point
float     releaseRedirectStrength // optional velocity nudge on release based on approach angle
```

On `OnGrapple` input:
```
1. Raycast/overlap for nearest active GrapplePoint within maxGrappleRange
2. If found → ApplyImpulse(directionToPoint * launchImpulse)
              player flies toward anchor through normal JUMP/FALL states
3. On reaching anchor (proximity check) or second OnGrapple press
   → optionally ApplyImpulse(redirectVector * releaseRedirectStrength)
   → normal aerial locomotion resumes with carried momentum
```

No joint physics, no state overrides, no locomotion lock. The feel is a snappy wuxia wire
launch rather than a pendulum swing — fast, directional, releases cleanly into aerial attacks.

**`GrapplePoint`** — simple component on world anchor rings:

```csharp
bool isActive    // can be toggled (e.g. a boss destroys anchor rings as a phase attack)
```

---

### 3l. Motion Input System

Fighting game-style directional sequences (quarter circles, half circles, etc.) as learnable
abilities. Detection is entirely separate from the combo system — different buffer, different
SO type, different detector. Once a motion is matched it fires through the existing
`AbilityExecutionContext` pipeline identically to any other ability.

#### Advanced Combat Mode

Holding the assigned button (e.g. left bumper) sets one flag on `PlayerController`:

```csharp
bool advancedCombatActive
// in movement update: if (!advancedCombatActive) { UpdateFacing(moveInput); }
```

While active:
- Facing direction is locked — the character won't turn around during stick rotations
- `MotionInputDetector` processes action button presses against motion patterns
- Movement speed is unchanged

The `ADVANCED_COMBAT` ability must be unlocked before the button does anything.
Pressing it before unlock has no effect — the flag simply never sets.

#### Analog Stick → Zone Mapping

The raw stick `Vector2` is converted to a single integer zone every frame before reaching
the buffer. Two steps:

**1. Dead zone check**
If `stick.magnitude < zoneDeadzone` → zone 5 (neutral). Prevents drift registering as input.

**2. Angle-based sector snap**
Beyond the dead zone, `atan2(y, x)` maps the stick to one of 8 sectors:

```
     8 (90°)
  7     9
4    5    6
  1     3
     2 (270°)
```

Each sector is centered on its angle. Default sector width is **45°** per zone.
Setting cardinal zones to **50°** (diagonals shrink to 40°) makes quarter circles more
forgiving — a tuning value on `StickInputConfigSO`, not a code change.

The decimal stick position is consumed here and never reaches the buffer. All downstream
motion detection reasons purely in integers.

#### MotionInputBuffer

A separate circular buffer from `InputBuffer` (which handles combo chaining). Runs
continuously, recording one entry per zone change:

```csharp
struct MotionZoneEntry {
    int   zone
    float enteredAt    // Time.time when this zone was entered
    float exitedAt     // Time.time when zone was exited (0 if still active)
}

int   bufferSize     // entries to retain (e.g. 32)
float zoneDeadzone   // magnitude threshold for neutral (zone 5)
```

An entry is only written when the zone changes — prevents flickering between adjacent
zones from polluting the buffer.

#### MotionPatternSO

Each step carries a zone group rather than a single zone, and an optional minimum hold
duration for charge inputs. Defines one learnable ability as a data asset — no code per ability:

```csharp
struct MotionStep {
    int[]  zones             // any zone in this array satisfies the step
    float  minHoldDuration   // 0 = transition (pass-through); > 0 = charge (must hold)
}

MotionStep[] sequence        // ordered steps
InputAction  confirmButton   // OnAttackLight, OnAttackHeavy, OnAbility1, etc.
float        maxStepInterval // max seconds allowed between each step
string       abilityId       // passed into AbilityExecutionContext pipeline on match
```

**Zone groups decouple intent from exact position.** `zones=[1,4,7]` means "any back
direction" — the player can hold straight back, down-back, or up-back interchangeably.
Diagonal zones naturally carry both their axis components, matching Guile-style flexibility
at no extra cost.

**Common patterns:**

| Name | Steps | Shorthand |
|---|---|---|
| Quarter Circle Forward | [2,3]→[3,6]→[6] | QCF |
| Dragon Punch | [6]→[2,3]→[3,6] | DP |
| Quarter Circle Back | [2,1]→[1,4]→[4] | QCB |
| Half Circle Back | [6,3]→[3,2]→[2,1]→[1,4]→[4] | HCB |
| Sonic Boom (charge) | [1,4,7] hold 1.2s → [6,3,9] | CB→F |
| Flash Kick (charge) | [1,2,3] hold 1.2s → [7,8,9] | CD→U |
| Back Back Forward | [4,1,7]→[4,1,7]→[6,3,9] | BBF |

#### MotionInputDetector

Component on the player. Only evaluates when `advancedCombatActive = true`.

**Pattern matching uses skip mode** — when scanning the buffer backwards for a step, any
entry whose zone is not in `step.zones` is skipped transparently. Only entries that satisfy
the current step advance the match. This means intermediate directions between repeated
steps (e.g. `4→2→4→6` satisfying `BBF`) are ignored naturally — the only requirement is
that matching zones appear in order within `maxStepInterval`.

**Charge detection** sums the `exitedAt - enteredAt` durations of all contiguous buffer
entries where `zone ∈ step.zones`. This handles stick drift between valid charge zones
(e.g. wandering between zone 4 and zone 1 while holding back) without breaking the charge.

On any action button press:
```
1. Check advancedCombatActive — if false, skip entirely
2. Filter registered MotionPatternSOs by confirmButton
3. For each candidate, walk buffer backwards in skip mode:
   a. For each step, scan backwards skipping non-matching entries
   b. For charge steps: sum durations of contiguous same-group entries,
      verify total >= minHoldDuration
   c. Verify time between matched entries <= maxStepInterval
4. Match found → fire ability through AbilityExecutionContext pipeline, consume input
5. No match → input falls through to ComboSystem as a normal attack
```

Fall-through is important — pressing punch without a valid motion still performs a normal
attack. Players who haven't unlocked `ADVANCED_COMBAT`, or who aren't holding the button,
experience zero difference from the base combat system.

---

### 3m. Knockdown State (DOWN)

`DOWN` is a locomotion state entered when a hit resolves with `onHitTargetState = Down`.
It replaces HURT entirely for the duration — the character is floored, not just reeling.

#### State Flow

```
Hit with onHitTargetState = Down
  → DOWN (floor tumble / slide animation plays, entity cannot act)
    ├─ Tech input detected (any direction + jump during DOWN)
    │    → DOWN_RECOVERY (brief neutral get-up stance, shorter than full recovery)
    └─ No tech input → DOWN_RECOVERY (full get-up animation on timer)
         → IDLE (resumes normal locomotion)
```

#### DOWN State Behavior

- Entity is completely invulnerable to **normal** hits while downed — attacks with
  `targetRequirement` that does not include `Downed` whiff even if hitboxes overlap
- OTG attacks explicitly include `Downed` in their `targetRequirement` — they hit on the
  ground but whiff against standing/airborne targets
- Blocking, parrying, and dodging are unavailable during DOWN and DOWN_RECOVERY
- Stagger bar does not decay during DOWN (enemy is already maximally vulnerable)

#### Per-Entity Tuning

DOWN duration and tech window are configurable per-entity, not per-attack:

```csharp
// On PlayerDataSO / EnemyDataSO:
float downDuration          // total time on the floor before forced DOWN_RECOVERY begins
float techWindowStart       // earliest frame a tech input is accepted (prevents mashing out instantly)
float techWindowEnd         // last frame a tech is accepted (= downDuration; missed = full recovery)
float downRecoveryDuration  // get-up animation length (tech = shortened version)
```

The attack's `knockbackVector` determines slide distance and direction while downed —
a sweep sends the target sliding backward, a launcher with Down result goes straight down.

#### Distinction from STAGGERED

| | STAGGERED | DOWN |
|---|---|---|
| Source | Stagger bar fills | Attack `onHitTargetState = Down` |
| Posture | Upright reel animation | Floor tumble/slide |
| Interrupt | Hitboxes deactivate immediately | Invulnerable to non-OTG hits |
| Exit | Timer (staggeredDuration) | Timer or player tech input |
| Can be hit | Yes — bonus damage window | Only by OTG-flagged attacks |

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
- `ADVANCED_COMBAT` (unlocks advanced combat mode + motion input detection)
- Additional motion input abilities defined via `MotionPatternSO` — no code required per ability

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
event Action          OnGrapple                    // WIRE_FU — fire/release grapple line
event Action          OnAdvancedCombatPressed      // hold to enter advanced combat mode
event Action          OnAdvancedCombatReleased     // release to exit
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

**Movement abilities as plugins.** Movement abilities (double jump, air dash, grapple) are
self-contained `MonoBehaviour` components that call hooks on `PlayerController`. The base
player systems have no knowledge of installed abilities. Adding a new movement ability is
adding a component — no changes to `PlayerStateMachine` or `PlayerController`.

**Motion inputs are detection-layer only.** `MotionInputBuffer` and `MotionInputDetector`
are entirely separate from `InputBuffer` and `ComboSystem`. A matched motion fires through
the same `AbilityExecutionContext` pipeline as every other ability. New motion abilities are
`MotionPatternSO` assets — no code required. Unmatched inputs fall through to the combo
system transparently.

---

## 13. Character Customization

Cosmetic-only sprite layer swapping. All options are pre-drawn assets; there is no procedural
generation. The system supports any number of slots and options per slot — initial scope is two
options per slot to prove the pipeline works.

### Technical Backbone: Unity Sprite Library

`com.unity.2d.animation` (already in the package list) ships a Sprite Library system designed
for exactly this:

- **`SpriteLibraryAsset`** — ScriptableObject that organizes sprites by **Category** (slot
  name, e.g. "Hair") and **Label** (option name, e.g. "Topknot"). One asset for the whole
  player character.
- **`SpriteLibrary`** component on the player root — holds the active `SpriteLibraryAsset`.
- **`SpriteResolver`** component on each layer child GameObject — references a Category and
  resolves to one Label at runtime. Swapping is one call:
  `spriteResolver.SetCategoryAndLabel("Hair", "Topknot")`

The animation rig drives all layers equally. Swapping a sprite label doesn't touch the
skeleton, blend trees, or any animation clip — it is purely a texture swap at the data level.

### Defined Slots (initial)

| Slot ID | Category (Sprite Library) | Initial options |
|---|---|---|
| `hair` | "Hair" | Topknot, Ponytail |
| `face` | "Face" | Default, Marked (face paint / scar) |
| `outfit` | "Outfit" | Robe, Wrapped (fighter wraps) |

New slots are added by authoring new Category entries in the `SpriteLibraryAsset` and a
corresponding `CosmeticSlotSO`. No code changes required.

### Data Layer

#### CosmeticSlotSO

Represents one swappable slot. Registered in `CosmeticRegistry`:

```csharp
string   slotId             // matches Sprite Library Category name exactly (e.g. "Hair")
string   displayName        // shown in customization UI
Sprite   slotIcon           // UI icon for this slot
List<CosmeticOptionSO> options
```

#### CosmeticOptionSO

One selectable option within a slot:

```csharp
string optionId             // matches Sprite Library Label name exactly (e.g. "Topknot")
string displayName
Sprite previewThumbnail     // shown in the option carousel
bool   isUnlockedByDefault  // false = must be unlocked via WorldStateManager
```

#### CosmeticRegistry

ScriptableObject (one asset, assigned in GameManager):

```csharp
List<CosmeticSlotSO> slots  // ordered — defines UI display order
```

Single source of truth for all available slots and options. Adding a new slot is adding an
entry here and drawing the sprites — nothing else.

#### PlayerAppearanceData

Plain serializable C# struct stored inside the save data (`WorldStateManager`):

```csharp
Dictionary<string, string> selections   // slotId → optionId
// e.g. { "hair": "Ponytail", "face": "Default", "outfit": "Robe" }
```

Missing entries fall back to the first option in the slot's list — new slots added in
patches won't break existing saves.

### CharacterCustomizationController

MonoBehaviour on the player prefab root. Holds a reference to one `SpriteResolver` per slot:

```csharp
// Inspector-assigned — one SpriteResolver child per slot, keyed by slotId
Dictionary<string, SpriteResolver> resolvers

void ApplyAppearance(PlayerAppearanceData data)
    // for each slot: resolvers[slotId].SetCategoryAndLabel(slotId, data.selections[slotId])

void ApplySlot(string slotId, string optionId)
    // single-slot live update — called by CustomizationUI during preview
```

Listens to `OnAppearanceChanged` via EventBus to reapply after a scene load (the player
prefab is always in the persistent Core scene, so in practice this fires once on boot).

### Unlock Integration

`CosmeticOptionSO.isUnlockedByDefault = false` options are locked until
`WorldStateManager.UnlockCosmetic(slotId, optionId)` is called. The customization UI
queries `WorldStateManager.IsCosmeticUnlocked(slotId, optionId)` per option and greys out
locked entries with a lock icon. Unlocks can be wired to boss kills, collectibles, or any
other `WorldState` flag — the customization system has no opinion on unlock conditions.

### Customization UI (CustomizationScreen)

Accessible from the pause menu or a hub NPC. Not available mid-combat.

```
┌─ Slot tabs: [Hair] [Face] [Outfit] ... ─────────────────────────────────┐
│                                                                         │
│   Live preview (player idle animation plays with current selections)   │
│                                                                         │
│   Option carousel: ← [Topknot ✓] [Ponytail] [🔒 ???] →               │
│                                                                         │
│                          [Confirm]  [Cancel]                            │
└─────────────────────────────────────────────────────────────────────────┘
```

- Selecting an option calls `ApplySlot()` immediately for live preview; selection is not
  saved until Confirm
- Cancel reverts to `WorldStateManager`-stored selections by calling `ApplyAppearance()`
- Locked options show a lock icon and are not selectable; their `displayName` can be replaced
  with a hint ("Defeat the Tiger Sensei")

### Art Pipeline Constraint

All options within a slot **must be drawn on the same canvas with identical pivot points**.
If a hair option is drawn 3px higher than the default, it will float off the head at runtime.
This is a sprite authoring rule, not a code safeguard — establish it as a template layer in
the art file before drawing any options.

Recommended: keep a master reference layer (body silhouette) in every art file and lock it
while drawing cosmetics, so alignment is always relative to the same anchor.

---

## 12. Build Order (Critical Path)

| Order | File | Why first |
|---|---|---|
| 1 | `EventBus.cs` | Everything communicates through this |
| 2 | `PlayerStateMachine.cs` | Gates all combat work |
| 3 | `HitboxController.cs` / `HurtboxController.cs` | Damage pipeline |
| 4 | `InputBuffer.cs` | Required before combo system |
| 5 | `StaggerMeter.cs` | Required before combat tuning |
| 6 | `StatSheet.cs` | Required before damage formula, health system, or chi pool |
| 7 | `GameTickManager.cs` | Required before any tick-based aura |
| 8 | `AuraManager.cs` + `AuraVisualController.cs` | Required before dodge, abilities, or status effects |
| 9 | `EquipmentManager.cs` + `AbilityExecutionContext.cs` | Required before any ability executes damage |
| 10 | `PlayerController` hooks (`ApplyImpulse`, `ForceLocomotionState`, etc.) | Required before any movement ability component |
| 11 | `MotionInputBuffer.cs` + `MotionInputDetector.cs` | Required before any motion input ability |
| 12 | `WorldStateManager.cs` | Room persistence and ability unlocks |
| 13 | `CharacterCustomizationController.cs` | Requires WorldStateManager for unlock queries and save/load |
| 14 | `CinematicDirector.cs` | Required before any boss content |

---

## TODO — Systems Not Yet Planned

These systems need full design sections before any code is written. Listed in priority order —
earlier entries block more downstream work.

| Priority | System | Why it's missing / what's needed |
|---|---|---|
| 1 | **Hitbox / Hurtbox runtime pipeline** | `HitboxDataSO` is fully designed but the runtime mechanics are never described: how hitboxes are authored on prefabs (child colliders?), how `HitboxController.OnTriggerEnter2D` routes to `DamageCalculator`, and critically — multi-hit prevention (same hitbox must not register twice on the same target per swing). Blocks all combat implementation. |
| 2 | **Physics layer matrix** | Which Unity physics layers exist and which collide with which (player hitbox → enemy hurtbox, environment, projectiles, etc.). Short section but must be decided before any collider is placed. Blocks the hitbox pipeline. |
| 3 | **Enemy AI** | State machine *states* are defined but no behavior logic exists: detection (vision cone? radius? LOS?), attack decision-making beyond "weighted random," aggression/spacing model, ranged vs. melee differences, boss AI phase navigation. Blocks any enemy implementation. |
| 4 | **`EquipmentSO` definition** | `EquipmentManager`, `AbilityModifierSO`, and the loot system all reference `EquipmentSO` but its fields are never defined. Needs: equipment slot types (weapon, armor, accessory), stat contribution fields, modifier list, display data. Blocks all equipment/loot work. |
| 5 | **Death & respawn flow** | Completely absent. What happens when HP reaches 0: death animation, which respawn point is selected, what state is restored (health, position, auras, enemy state), whether there is a death penalty. Needed before the health system or player controller can be considered complete. |
| 6 | **Save system detail** | `WorldStateManager` mentions JSON serialization but the flow is vague: what triggers a save (room transition? checkpoint activation? manual?), checkpoint placement rules, save slot management, and how death-respawn integrates with the save state. Closely related to #5 but a separate design concern. |
| 7 | **Loot & item drops** | Drop tables (enemy-specific? room-specific? global pool?), item pickup interaction, and a minimal inventory/equipment slot UI. The `AbilityModifierSO` pipeline is ready to consume items — nothing yet produces them. |
| 8 | **Level up / XP system** | `LevelScale` exists in the damage formula and stats reference `level`, but there is no XP gain, level-up event, stat growth on level-up, or player-facing progression loop. |
| 9 | **Camera system** | Cinemachine is planned for combat effects (impulse shake, bullet time zoom, boss cinematics) but the *gameplay* camera is never designed: room-based confinement, player follow behavior (lookahead? deadzone?), camera transition between rooms, and lock-on framing during boss fights. |
| 10 | **NPC / dialogue** | Most metroidvanias require at least a minimal system: dialogue trigger volumes, a text display component, branching or sequential lines, and integration with `WorldStateManager` for state-gated dialogue. Needed for lore delivery, merchants, and ability teachers. Lowest priority — doesn't block combat or movement. |
