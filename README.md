# [StoneAC](https://modrinth.com/plugin/stoneac)

**A companion anticheat for [GrimAC](https://github.com/GrimAnticheat/Grim) that covers the checks GrimAC intentionally does not.**

GrimAC is a simulation-based anticheat that predicts client movement with extreme accuracy. It is exceptional at movement and combat geometry checks. However it deliberately does not check things like click rate, inventory speed, reaction timing, or bot-like behaviour patterns. StoneAC fills that gap — 40 additional checks that run alongside GrimAC without duplicating its work.

---

## Requirements

| Dependency | Type | Notes |
|---|---|---|
| [PacketEvents](https://github.com/retrooper/packetevents) | **Required** | StoneAC uses PacketEvents for low-level packet interception. GrimAC also uses PacketEvents so it is likely already on your server. |
| [GrimAC](https://github.com/GrimAnticheat/Grim) | Optional (recommended) | StoneAC works standalone but is designed to complement GrimAC. When GrimAC is present, StoneAC routes its alerts through GrimAC's staff chat pipeline and auto-disables duplicate admin features. |
| [Floodgate / Geyser](https://geysermc.org) | Optional | If present, Bedrock players are automatically exempted from all checks. |

**Requires Paper 1.21.4+ and Java 21.**

---

## How it works with GrimAC

StoneAC and GrimAC are two separate plugins that run in parallel. They do not conflict.

**When GrimAC is detected on first startup:**
- StoneAC hooks into GrimAC's alert pipeline using reflection — flags appear in GrimAC's staff chat alongside GrimAC's own flags, in the same format, respecting `/grim alerts` toggles
- Duplicate admin commands (`/sac freeze`, `/sac spectate`, `/sac gui`, standalone alert broadcast) are automatically disabled since GrimAC already provides them
- All 40 detection checks remain fully enabled regardless

**Permission bridging:**
- Players with `grim.staff` automatically have full StoneAC admin access — no extra permission nodes needed
- Players with `grim.alerts` automatically receive StoneAC alert messages in their feed

**Without GrimAC:**
StoneAC functions completely independently with its own alert broadcast, freeze, spectate, and GUI system.

---

## Installation

1. Drop `StoneAC-1.0.0.jar` into your `plugins/` folder
2. Make sure `packetevents` is also in `plugins/` (required)
3. Restart the server — StoneAC generates `config.yml` automatically
4. If GrimAC is present, duplicate features are auto-disabled on first run and logged to console

---

## The 40 Checks

### Combat

| Check | What it detects | How |
|---|---|---|
| **AutoClicker** | Click rates above the configured max, and suspiciously consistent intervals (catches jitter-mode autoclickers) | Timestamps every attack packet in nanoseconds. Flags raw CPS > max, or CPS > 8 with standard deviation below 2ms |
| **KillAura** | Six sub-checks on every attack | See below |
| **AimAssist** | Rotation snapping toward nearby entities without attacking | On every rotation packet, if yaw/pitch delta exceeds threshold, schedules a main-thread check to see if any entity is within 3° of the new aim direction |
| **SilentAim** | Attacking entities while not looking at them | Computes angle between the player's sent yaw/pitch and the target's center hitbox. Flags if > 90° |
| **Triggerbot** | Attacking impossibly fast after crosshair lands on a target | Uses INTERACT_AT packets as "crosshair on entity" proxy, measures reaction time. Flags sub-50ms reactions and statistically consistent patterns |
| **AntiKnockback** | Ignoring server-sent velocity packets | Records velocity direction/magnitude when sent, waits ping-compensation ticks, then checks if player moved in the knockback direction using position history |
| **Criticals** | Impossible critical hits | Detects critical hit conditions (falling, not on ground/water/ladder) using `EntityDamageByEntityEvent` with Paper 1.21.4 compatible detection |
| **Reach** | Attacking beyond maximum reach distance | Measures eye-to-target-center distance on every attack, flags above configured max + ping buffer |
| **FightBot** | Perfectly timed automated combat AI | Tracks millisecond intervals between attacks on the Netty thread. Flags if standard deviation across 10+ intervals is below 20ms — humans vary widely, bots hit exactly on cooldown every time |
| **KillPotion** | Throwing harming splash potions at superhuman speed or simultaneously with attacking | Flags throws within 200ms of an attack, and throw rates below 300ms apart |
| **TPAura** | Teleporting to attack targets | Checks if the player's last two positions show a jump > 3 blocks in a single tick while ending within 4 blocks of the target |

**KillAura sub-checks:**
- **RotSnap** — yaw/pitch jumped by more than the threshold in the same tick as an attack, while already on target
- **OutsideFOV** — attacked an entity outside the configured field of view angle
- **OmniAura** — attacked an entity more than 160° behind the player
- **NoSwing** — attack packet arrived without a preceding arm-swing packet
- **DeadHit** — attacked while the player's own health was zero
- **TargetSwitch** — switched attack targets faster than the configured minimum milliseconds

---

### Inventory

| Check | What it detects | How |
|---|---|---|
| **ChestStealer** | Clicking through a chest at superhuman speeds | Two sub-checks: per-click delay below 50ms, and more than 20 clicks/second in any container |
| **AutoArmor** | Equipping multiple armor pieces faster than humanly possible | Tracks armor equip sequences. Flags if average time per piece drops below 100ms |
| **AutoTotem** | Re-equipping a totem of undying within milliseconds of it popping | Starts a timer on `EntityResurrectEvent`, flags if totem is moved to offhand within 80ms |
| **FastInventory** | Bulk inventory transaction spam | Counts all window transactions per second across any inventory type. Flags above 30/s |
| **AutoEat** | Eating while attacking, or eating faster than vanilla allows | Flags eating within 400ms of an attack, and eating faster than once per 1500ms |
| **AutoPotion** | Throwing splash/lingering potions at superhuman rate | Flags throw intervals below 200ms |
| **FastEat** | Consuming food faster than the 32-tick vanilla minimum | Tracks eat start (from packet) to finish (from FoodLevelChangeEvent). Flags below 1400ms |

---

### World

| Check | What it detects | How |
|---|---|---|
| **Nuker** | Breaking too many blocks per second or hitting too many block faces | Flags > 5 blocks/second or > 3 distinct block faces within the same second |
| **InstaBreak** | Breaking blocks faster than hardness allows | Simulates Minecraft's full break-time formula (hardness, tool speed, Efficiency, Haste, Mining Fatigue, Aqua Affinity, floating penalty) and flags if actual time was shorter |
| **VeinMiner** | Breaking many of the same ore/log type in rapid succession | Tracks same-material breaks per 1-second window. Flags > 4 of the same ore/log/valuable block per second |
| **AutoFarm** | Breaking crops at superhuman speeds | Flags > 8 crop blocks (wheat, carrots, potatoes, beetroot, nether wart, sugar cane, bamboo, melon, pumpkin) per second |
| **AutoFish** | Reeling in within milliseconds of a fish bite | Measures gap between `PlayerFishEvent.BITE` and reel. Flags reactions below 100ms |
| **AutoRespawn** | Respawning instantly after death | Measures gap between death and respawn. Flags below 150ms |
| **AutoSwitch** | Switching hotbar slots impossibly fast, or attacking immediately after a switch | Two sub-checks: slot switches below 50ms apart, and attack within 50ms of a slot switch (AutoSword) |
| **AutoMine** | Mining blocks the player is not looking at | Computes angle between player's look direction and block center. Flags above 60° |

---

### Movement

| Check | What it detects | How |
|---|---|---|
| **Jesus** | Walking on water | Every tick: checks if player is standing on water surface (`isOnGround()` + water block at feet). Exempts Frost Walker and Levitation. Flags after 10 consecutive ticks |
| **Strafe** | Perfectly circular orbit around a target (KillAura movement pattern) | Tracks yaw delta per rotation packet. During combat, flags if coefficient of variation of last 40 deltas is below 5% — perfectly consistent circular movement |
| **Blink** | Packet-burst blink hacks | On every position packet: measures gap since last position packet and horizontal distance jumped. Flags if gap > 2s and jump > 5 blocks |
| **InvWalk** | Moving while a container inventory is open | On every movement event: checks if a real container is open (not CRAFTING or PLAYER type). Flags horizontal movement above 0.1 blocks |
| **AntiAFK** | Looping rotation macros | Accumulates yaw deltas per rotation packet. Flags if last 20+ samples are all within 0.01° of each other |
| **HighJump** | Jumping higher than vanilla allows | On the first tick leaving the ground: flags Y velocity above 0.42 + (0.1 × Jump Boost level) + 0.1 buffer |
| **FastLadder** | Climbing ladders faster than allowed | On movement while on LADDER or VINE: flags upward Y delta above 0.13 blocks/tick |
| **NoWeb** | Moving through cobweb at normal speed | On movement while inside COBWEB: flags horizontal speed above 0.04 blocks/tick |
| **Regen** | Health regenerating faster than any legal source | Every tick: calculates HP gain since last check, converts to HP/second, flags if above max possible from natural regen + Regeneration potion |
| **AntiHunger** | Sprinting without hunger draining | Every tick during survival sprint: flags after 200 consecutive ticks without food level decreasing (eating while sprinting correctly resets the counter) |
| **AntiCactus** | Taking no damage from cactus contact | Every tick while touching cactus: after 20 ticks, checks if HP dropped since contact started. Flags if not |
| **AntiPotion** | Moving at full speed while under Slowness | On movement with Slowness effect: calculates max legal speed (base × (1 - 0.15 × amplifier)) and flags if exceeded |
| **Parkour** | Always jumping at the perfect last tick before a block edge | On every jump: checks if player is within 0.3 blocks of a block boundary AND the adjacent block is air. Flags after 8 consecutive perfect edge jumps |
| **SneakSpam** | Sneak packet spam | Counts sneak toggles per second. Flags above 8/s |

---

## Commands

All commands require `stoneac.admin` **or** `grim.staff`.

| Command | Description |
|---|---|
| `/sac violations <player>` | Show all violation counts for a player across every check |
| `/sac freeze <player>` | Freeze a player in place (they cannot move until unfrozen) |
| `/sac unfreeze <player>` | Unfreeze a player |
| `/sac spectate <player>` | Teleport to a player to observe them |
| `/sac gui` | Open an in-game GUI showing all online players and their violation counts. Click a player to teleport to them |
| `/sac info` | Show plugin version and active check count |
| `/sac reload` | Reload config.yml without restarting |

---

## Permissions

| Permission | Default | Description |
|---|---|---|
| `stoneac.admin` | OP | Full access to all `/sac` commands. Automatically granted to anyone with `grim.staff` |
| `stoneac.alerts` | OP | Receive StoneAC flag alerts in chat. Automatically granted to anyone with `grim.alerts` or `grim.staff` |
| `stoneac.bypass` | false | Bypass all StoneAC checks |
| `stoneac.check.<name>` | false | Bypass a specific check (e.g. `stoneac.check.autoclicker`) |

---

## Configuration

StoneAC generates a fully documented `config.yml` on first run. Every check can be individually enabled or disabled:

```yaml
AutoClicker:
  enabled: true
  max-cps: 20
  min-variance: 2.0
  max-violations: 10
  punish: true
```

Key sections:

- **alerts** — prefix, cooldown, console logging, broadcast toggle
- **logging** — log to file (`logs/stoneac.log`) and/or SQLite/MySQL database
- **bedrock** — Bedrock player exemption via Floodgate or name prefix
- **admin** — freeze message, and enable/disable flags for freeze, spectate, GUI
- **punishments** — commands to run when a player exceeds max violations

### GrimAC auto-disable

On first startup with GrimAC present, StoneAC writes the following keys to `config.yml` if they are not already set:

```yaml
admin:
  freeze-enabled: false      # GrimAC has /grim freeze
  spectate-enabled: false    # GrimAC has /grim spectate
  gui-enabled: false         # GrimAC has its own GUI
alerts:
  broadcast: false           # alerts route through GrimAC's pipeline
```

To re-enable any of these, set the value to `true` and run `/sac reload`.

---

## Bedrock Support

If Floodgate or Geyser is installed, StoneAC automatically detects Bedrock players and exempts them from all checks. Bedrock clients move differently and would produce false flags. Detection uses the Floodgate API if available, with a name-prefix fallback (default prefix `.`).

---

## Violation Logging

StoneAC logs every flag to `plugins/StoneAC/logs/stoneac.log` by default. Optional database logging (SQLite or MySQL) can be enabled in config:

```yaml
logging:
  log-to-database: true
database:
  type: mysql
  mysql:
    host: localhost
    port: 3306
    database: stoneac
    username: root
    password: secret
```

---

## Bug Reports

Found a bug? Open an issue on [GitHub Issues](https://github.com/stone380380alt-arch/StoneAC/issues) with:
- Your server version (Paper build number)
- StoneAC version
- GrimAC version (if applicable)
- What happened vs what you expected
- Relevant server log output

---

## Building from Source

```bash
git clone https://github.com/stone380380alt-arch/StoneAC.git
cd StoneAC
mvn clean package
```

Output JAR: `target/StoneAC-1.0.0.jar`

Requires Java 21 and Maven. PacketEvents and Paper API are pulled automatically as provided dependencies.
