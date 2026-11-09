# Super Mario Bros. 2 (NES) Game Genie codes

These codes were made from PRG0, the first USA release. Each code's bytes
were also checked in PRG1 (Rev A) and Super Mario USA. PRG1 and Super Mario
USA move some code in banks 0-3, C, E and F, so a code works on another
revision only if its entry says so. Every code is the 8-letter form with a
compare byte. All effects come from the disassembly. They have not been
tested in play unless an entry says so.

## Lift an enemy while you already carry one

**Code:** `XOULOEKO`
**Raw form:** `B8B1:9A:9C` (address:new-byte:compare-byte, bank 3)
**Recorded:** 2026-10-02 18:35:27

**Mechanism:** When the player stands on an enemy and presses B,
`CheckCollisionWithPlayer_StandingOnHead` (`prg-2-3.asm`) checks if a lift is
allowed:

```
	; bail if we already have an item or are ducking
	LDA HoldingItem
	ORA PlayerDucking
	BNE CheckCollisionWithPlayer_NoLift
```

The code changes the operand of `LDA HoldingItem` (`$9C`) to `$9A`, so it
reads `PlayerDucking` instead. The check becomes
`PlayerDucking OR PlayerDucking`. Holding an item no longer stops the lift.
Ducking still does. A is still `$00` after the check, so the stores after it
(`EnemyCollision`, `ObjectShakeTimer`) get the same value as in the normal
game.

**Evidence:** Patches the operand of `LDA HoldingItem` in
`CheckCollisionWithPlayer_StandingOnHead`. The source and the built PRG0 ROM
both have `A5 9C` at `$B8B0`. No other bank has `$9C` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Only enemies. Lifting from the ground (vegetables, mushroom blocks and so
  on) goes through a different path and still checks `HoldingItem`.
- The new lift overwrites `ObjectBeingCarriedIndex` and increments
  `HoldingItem` again, to 2. The first object is no longer the carried one.
  Expect it to float in place or act oddly. After one throw, `HoldingItem` is
  still 1, so the game may think you still carry something.

## Pull up vegetables while you carry an item

**Code:** `XPSEXTSN`
**Raw form:** `8E5A:9A:F5` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-04 00:57:24

**Mechanism:** B does two things in the same frame, in this order:

1. `loc_BANK0_8E05` (after `sub_BANK0_8DC0`) throws the item you carry. It
   does `DEC HoldingItem`, then clears `PlayerDucking` and
   `Player1JoypadPress` with `STA`, so nothing later in the frame sees the B
   press.
2. `TileBehavior_CheckJar` runs later, from `PlayerTileCollision`. It does
   `LDY HoldingItem` / `BNE`, then `TileBehavior_CheckPickUp` checks for B
   and pulls the tile you stand on.

The code changes the operand of `STA Player1JoypadPress` (`$F5`) to `$9A`.
The instruction now stores 0 to `PlayerDucking` again, which the previous
instruction already did. The B press is kept, and `HoldingItem` is already
0 after the throw, so step 2 pulls up the tile.

When you carry an item and press B on top of a vegetable, you throw the item
and pull up the vegetable at the same time.

**Evidence:** `loc_BANK0_8E42` and `TileBehavior_CheckJar` in
`src/prg-0-1.asm`. The byte matches the source and each built ROM. No other
bank in any of the three ROMs has `$F5` at this offset.

**Revisions:** Same code for PRG0, PRG1 and Super Mario USA.

**Caveats:**

- You don't keep the first item: it is thrown. Only one item is carried at a
  time, so `HoldingItem` and `ObjectBeingCarriedIndex` stay correct.
- If the item can't be thrown (for example, a wall is in front of you),
  nothing is thrown and nothing is pulled up.
- This works for anything you can pull or dig from the ground: vegetables,
  mushroom blocks and sand.
- The other B checks later in the frame also see the press. If you throw
  while standing on an enemy, you also lift the enemy.
- The effect in play was not checked.

## Lift an enemy from any side

**Code:** `AEOLUELA`
**Raw form:** `B893:00:0B` (address:new-byte:compare-byte, bank 3)
**Recorded:** 2026-10-02 18:35:27

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** When the player touches an enemy that does not always do
damage, `CheckCollisionWithPlayer_NotInvincible` (`prg-2-3.asm`) calls
`DetermineCollisionFlags` and tests the result:

```
	LDA byte_RAM_F
	AND #$0B
	BEQ CheckCollisionWithPlayer_StandingOnHead

CheckCollisionWithPlayer_HurtPlayer:
	JMP DamagePlayer
```

`$0B` is `CollisionFlags_Up | CollisionFlags_Left | CollisionFlags_Right`. If
the player touches from any side but the top, the player is hurt. The code
changes the mask to `$00`, so the `BEQ` always goes to
`CheckCollisionWithPlayer_StandingOnHead`. Any touch counts as standing on the
enemy's head, and B lifts it.

**Evidence:** Patches the operand of `AND #$0B` in
`CheckCollisionWithPlayer_NotInvincible`. The source and the built PRG0 ROM
both have `29 0B` at `$B892`. No other bank has `$0B` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Enemies no longer hurt you by touch from the side or below. Enemies with
  `SpriteFlags46E_Damage` set (for example Porcupo) and exploding bombs still
  hurt you, because those checks come first.
- Each touch also clears `PlayerInAir` and sets `CollisionFlags_PlayerOnTop`
  on the enemy. Expect odd movement when you walk into an enemy.
- Unliftable enemies still can't be lifted.

## Toad floats like the Princess

**Code:** `GUNAPLAE`
**Raw form:** `83F1:3C:00` (address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 18:35:27

**Mechanism:** `CharacterStats` (`prg-a-b.asm`) holds 23 bytes per character.
`CopyCharacterStatsAndStuff` copies the current character's row to
`CharacterStatsRAM`. The "Floating Time" byte goes to `JumpFloatLength`. It is
`$3C` (60 frames) for the Princess and `$00` for the other three.
`PlayerStartJump` copies `JumpFloatLength` to `JumpFloatTimer` at each jump.
`PlayerGravity` lets the player float while A is held and the timer is not
zero. This code sets the Floating Time byte of `ToadStats` to `$3C`.

**Evidence:** Patches the "Floating Time" byte of `ToadStats`. The source and
the built PRG0 ROM have `$00` at `$83F1` in bank A. The Princess's byte at
`$841F` is `$3C`. Bank A is mapped at `$8000` only. No other even bank has
`$00` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The new value takes effect when the stats are copied, for example at the
  character select screen. It does not change a game that is already running.
- **There is no safe code for Mario.** His byte is at `$83DA`. Bank 0 also
  has `$00` there, and bank 0 is mapped at `$8000` during play. In bank 0
  that byte is the operand of `LDX #$00` before `JSR sub_BANK0_8478`, in the
  attribute update for upward scrolling. With a `$3C` there, that routine
  writes to `$011D` (in the stack page) instead of `byte_RAM_E1`. The compare
  byte can't tell the two banks apart, so the Mario code is left out.
- For Mario, see "All characters float, including Mario".
- Use with `GLEEAGAE` (Luigi) to give both characters the float.


## Luigi floats like the Princess

**Code:** `GLEEAGAE`
**Raw form:** `8408:3C:00` (address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 18:35:27

**Mechanism:** `CharacterStats` (`prg-a-b.asm`) holds 23 bytes per character.
`CopyCharacterStatsAndStuff` copies the current character's row to
`CharacterStatsRAM`. The "Floating Time" byte goes to `JumpFloatLength`. It is
`$3C` (60 frames) for the Princess and `$00` for the other three.
`PlayerStartJump` copies `JumpFloatLength` to `JumpFloatTimer` at each jump.
`PlayerGravity` lets the player float while A is held and the timer is not
zero. This code sets the Floating Time byte of `LuigiStats` to `$3C`.

**Evidence:** Patches the "Floating Time" byte of `LuigiStats`. The source and
the built PRG0 ROM have `$00` at `$8408` in bank A. The Princess's byte at
`$841F` is `$3C`. Bank A is mapped at `$8000` only. No other even bank has
`$00` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The new value takes effect when the stats are copied, for example at the
  character select screen. It does not change a game that is already running.
- Luigi falls slower than the others, so his float may look different.
- Use with `GUNAPLAE` (Toad) to give both characters the float.
- There is no safe code for Mario. See the Toad entry.


## Climb in open air by pressing Up

**Codes:** `AGUAKLYP` + `AAUPAEGE` (use both)
**Raw form:** `8B34:40:17` and `9030:08:0C`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:35:27

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `PlayerTileCollision_CheckClimbable` (`prg-0-1.asm`) compares
the tile at the player's position with the 10 entries in `ClimbableTiles`.
`PlayerTileCollision_CheckCherryAndClimbable` calls it each frame. If the tile
is climbable, `PlayerTileCollision_Climbable` starts a climb when Up or Down is
held. `HandlePlayerState_Climbing` calls the same check to keep the climb going.

- `AGUAKLYP` changes the last `ClimbableTiles` entry from
  `BackgroundTile_ChainStandable` (`$17`) to `BackgroundTile_Sky` (`$40`).
  The empty background now counts as a vine.
- `AAUPAEGE` changes `AND #ControllerInput_Down | ControllerInput_Up`
  (`$0C`) in `PlayerTileCollision_Climbable` to `AND #ControllerInput_Up`
  (`$08`). Only Up starts a climb.

The second code keeps Down free, so you can still duck, charge a jump, and go
down jars while you stand in open air. Without it, pressing Down in the open
would start a climb.

**Evidence:** `ClimbableTiles` has `$17` at `$8B34`. `AND` with `$0C` is at
`$902F`, so the operand is at `$9030`. Both match the source and the built
PRG0 ROM. No other bank has the same byte at either offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The climb starts only when the player is lined up with the tile grid. The
  check is `(PlayerXLo + 4) AND $0F < 8`, so it works on about half of each
  16-pixel column. It also does not start while you carry something.
- This should not block doors. A door is not a sky tile, so pressing Up in a
  doorway should still enter the door. Not tested.
- Chain tops (`BackgroundTile_ChainStandable`) can no longer be climbed.
- Down no longer starts a climb on real vines and ladders. Up still does.
- Other background tiles that look like empty space (if an area uses one) are
  not climbable.

## Float time never runs out

**Code:** `SXNEXKVK`
**Raw form:** `8CFA:AD:CE` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:35:27

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** While the player floats, `PlayerGravity` (`prg-0-1.asm`) counts
the float timer down:

```
	LDY JumpFloatTimer
	BEQ PlayerGravity_Falling

	DEC JumpFloatTimer
	LDA byte_RAM_10
```

The code changes the `DEC` opcode (`$CE`, absolute) to `LDA` (`$AD`,
absolute), so the line becomes `LDA JumpFloatTimer`. The next line loads A
again, so this has no other effect. The timer stays at its start value, so
the float lasts while A is held.

**Evidence:** Patches the opcode of `DEC JumpFloatTimer` in `PlayerGravity`.
The source and the built PRG0 ROM both have `CE C9 04` at `$8CFA`. No other
bank has `$CE` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Only characters with a float can use it: the Princess, or Toad and Luigi
  with the code above.
- The float still starts only near the top of a jump (Y velocity `$FC` or
  more).
- In the normal game, `PlayerGravity` sets the timer to zero once it is no
  longer full. With this code it stays full, so you can also float after you
  walk off a ledge.

## Jump again in mid-air

**Code:** `AAXAEGZG`
**Raw form:** `8C20:00:42` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:55:15

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `HandlePlayerState_Normal` calls `sub_BANK0_8C1A`
(`prg-0-1.asm`) each frame. It starts a jump when A is pressed, but first
skips the jump code if the player is in the air:

```
	LDA PlayerInAir
	BNE ResetPartialCrouchJumpTimer
```

The code changes the `BNE` offset (`$42`) to `$00`, so the branch goes to
the next instruction either way. The jump check then runs in the air too.
Each A press calls `PlayerStartJump`, which sets a new Y velocity, and plays
the jump sound. You can jump again as many times as you like.

**Evidence:** Patches the operand of `BNE ResetPartialCrouchJumpTimer` at the
start of `sub_BANK0_8C1A`. The source and the built PRG0 ROM both have
`A5 99 D0 42` at `$8C1D`. No other bank has `$42` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The jump height comes from the same table as a ground jump. It uses your
  X speed and whether you carry something, as on the ground.
- In the air, the Down button now ducks, as it does on the ground. In the
  normal game, the duck state from the ground stays the same during a jump.
- Each jump in the air does `INC PlayerInAir`, so it can go above 1. The
  tile collision code sets it back to 0 or 1 when you fall or land. All
  reads of `PlayerInAir` only test for zero, so this should not matter.
- Not checked with the flying carpet. On the carpet, the code may now take
  a different path in the air.

## Crouch jump charges at once

**Code:** `VGSEVKVV`
**Raw form:** `8C5E:CE:EE` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:57:32

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** While the player ducks on the ground, `sub_BANK0_8C1A`
(`prg-0-1.asm`) charges the crouch jump:

```
	LDA CrouchJumpTimer ; check if crouch jump is charged
	CMP #$3C
	BCS loc_BANK0_8C92

	INC CrouchJumpTimer ; increment crouch jump charge
	BNE loc_BANK0_8C92
```

The jump is charged when `CrouchJumpTimer` is `$3C` (60 frames) or more.
The code changes `INC` (`$EE`) to `DEC` (`$CE`). On the first frame of
ducking, the timer goes from `$00` to `$FF`. That is more than `$3C`, so the
charge is full after one frame. The `BNE` still branches, because `$FF` is
not zero. All other reads of the timer use `CMP #$3C` and `BCS`/`BCC`, so
they treat `$FF` as charged: the jump height in `PlayerStartJump`, the reset
check, and the flashing palette in bank F.

**Evidence:** Patches the opcode of `INC CrouchJumpTimer` in
`sub_BANK0_8C1A`. The source and the built PRG0 ROM both have `EE CA 04` at
`$8C5E`. No other bank has `$EE` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The player flashes as soon as you duck, as with a full charge.
- Any duck followed by A is now a crouch jump, even a short one.
- `PlayerStartJump` sets the timer back to `$00` at each jump, as in the
  normal game.

## Ice is not slippery

**Code:** `AANAUNYA`
**Raw form:** `8F73:00:0F` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:58:15

**Mechanism:** When the player lands on ground, `PlayerTileCollision`
(`prg-0-1.asm`) checks if the ground is ice:

```
PlayerTileCollision_CheckSlipperyTile:
	LSR byte_RAM_C
	BCC PlayerTileCollision_CheckJar

	LDA #$0F
	STA GroundSlipperiness
```

`GroundSlipperiness` is set to `$00` at the start of each frame. A value of
`$0F` makes the player speed up only every 4th frame and slow down only
every 16th frame, and stops the walk animation. The code changes `LDA #$0F`
to `LDA #$00`, so ice sets the value to `$00`, the same as normal ground.

**Evidence:** Patches the operand of `LDA #$0F` in
`PlayerTileCollision_CheckSlipperyTile`. The source and the built PRG0 ROM
both have `A9 0F 8D 24 06` at `$8F72`. No other bank has `$0F` at this
offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Only the player. This code does not change how enemies or objects move on
  ice.
- After the patch, A is `$00` instead of `$0F` when the code calls
  `TileBehavior_CheckJar`. That routine loads A before it reads it, so this
  should not matter.
- Do not use with "All ground is slippery" (`AANAOYIA`). That code skips
  this check in a different way, so the two codes cancel each other.

## All ground is slippery like ice

**Code:** `AANAOYIA`
**Raw form:** `8F71:00:05` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:58:15

Tags: `demo-ready` `play-start`

**Mechanism:** When the player lands on ground, `PlayerTileCollision`
(`prg-0-1.asm`) checks if the ground is ice:

```
PlayerTileCollision_CheckSlipperyTile:
	LSR byte_RAM_C
	BCC PlayerTileCollision_CheckJar

	LDA #$0F
	STA GroundSlipperiness
```

`byte_RAM_C` is set when the tile under the player is
`BackgroundTile_JumpThroughIce`. The code changes the `BCC` offset (`$05`)
to `$00`, so the branch goes to the next instruction either way.
`GroundSlipperiness` is set to `$0F` on all ground. The player speeds up
only every 4th frame and slows down only every 16th frame, as on ice.

**Evidence:** Patches the operand of `BCC PlayerTileCollision_CheckJar` in
`PlayerTileCollision_CheckSlipperyTile`. The source and the built PRG0 ROM
both have `46 0C 90 05` at `$8F6E`. No other bank has `$05` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Only the player. This code does not change how enemies or objects move.
- It applies to any ground the player lands on, including enemies and
  objects you stand on, if the player tile collision code handles them.
  Not checked.
- The walk animation stops on all ground, as it does on ice.
- Do not use with "Ice is not slippery" (`AANAUNYA`).

## All characters flutter their legs like Luigi

**Code:** `AANEKIGL`
**Raw form:** `8D7C:00:34` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:58:59

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `PlayerWalkJumpAnim` (`prg-0-1.asm`) runs the walk animation
in the air only for Luigi:

```
	LDA CurrentCharacter ; does this character get to flutter jump?
	CMP #Character_Luigi
	BNE ExitPlayerWalkJumpAnim

	LDA PlayerWalkFrameCounter
	BNE UpdatePlayerAnimationFrame ; maintain current frame

	LDA #$02 ; fast animation
	BNE NextPlayerWalkFrame
```

The code changes the `BNE` offset (`$34`) to `$00`, so the branch goes to
the next instruction either way. Every character runs the fast walk
animation (2 frames per step) in the air.

**Evidence:** Patches the operand of `BNE ExitPlayerWalkJumpAnim` after
`CMP #Character_Luigi`. The source and the built PRG0 ROM both have
`C9 03 D0 34` at `$8D79`. No other bank has `$34` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- This is the animation only. Luigi's high, floaty jump comes from his
  values in `CharacterStats`, and this code does not change them.
- Don't use with the one-character flutter codes below. They patch the
  byte before this one, and this code already covers every character.

## Mario flutters his legs like Luigi

**Code:** `AANEXILA`
**Raw form:** `8D7A:00:03` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:58:59

**Mechanism:** `PlayerWalkJumpAnim` (`prg-0-1.asm`) runs the walk animation
in the air only when `CurrentCharacter` is `Character_Luigi` (`$03`). See
the "All characters" entry above. The code changes `CMP #Character_Luigi`
to `CMP #Character_Mario` (`$00`).

**Evidence:** Patches the operand of `CMP #Character_Luigi` in
`PlayerWalkJumpAnim`. The source and the built PRG0 ROM both have
`A5 8F C9 03` at `$8D77`. No other bank has `$03` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Luigi no longer flutters. Only one character can be named in the `CMP`.
  Use `AANEKIGL` for every character.
- Animation only. The jump physics do not change.

## Princess flutters her legs like Luigi

**Code:** `PANEXILA`
**Raw form:** `8D7A:01:03` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:58:59

**Mechanism:** Same as the Mario entry, but the `CMP` value becomes
`Character_Princess` (`$01`).

**Evidence:** Same byte as the Mario entry: `$03` at `$8D7A`. No other bank
has `$03` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Luigi no longer flutters. Use `AANEKIGL` for every character.
- Animation only. The animation also runs while she floats.

## Toad flutters his legs like Luigi

**Code:** `ZANEXILA`
**Raw form:** `8D7A:02:03` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 18:58:59

**Mechanism:** Same as the Mario entry, but the `CMP` value becomes
`Character_Toad` (`$02`).

**Evidence:** Same byte as the Mario entry: `$03` at `$8D7A`. No other bank
has `$03` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Luigi no longer flutters. Use `AANEKIGL` for every character.
- Animation only. The jump physics do not change.

## Thrown vegetables and enemies fly flatter

**Code:** `AEUEKSAO`
**Raw form:** `8DBC:08:18` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:00:07

**Mechanism:** When the player throws, the end of `sub_BANK0_8DC0`
(`prg-0-1.asm`) sets the object's start speed from two tables:

```
ThrowXVelocity:
	...
	.db $D0 ; standing, left (projectiles)
	.db $30 ; standing, right (projectiles)
	...

ThrowYVelocity:
	.db $18 ; standing (blocks)
	.db $00 ; moving (blocks)
	.db $18 ; standing (projectiles)
	.db $F8 ; moving (projectiles)
```

"Projectiles" are object types up to `Enemy_Coin`: enemies, vegetables,
shells and coins. A throw while standing or walking slowly (X speed less
than `$08`) starts at Y speed `$18` (down) and X speed `$30`, a path about
27 degrees below horizontal. The code changes the "standing (projectiles)"
Y speed to `$08`, so the throw starts about 9 degrees below horizontal.
Gravity still pulls the object down after that.

**Evidence:** Patches the third byte of `ThrowYVelocity`. The source and the
built PRG0 ROM both have `18 00 18 F8` at `$8DBA`. No other bank has `$18`
at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Only the slow throw of projectiles. The fast throw (`$F8`, a little
  upward) and the throws of blocks, bombs, POW blocks and keys do not
  change.
- For a different angle, change the new byte. `$00` starts the throw fully
  flat (`AEUEKSAP`). Values from `$01` to `$17` give angles in between.
- Not tested. The object may fly farther before it lands, so it may reach
  enemies that a normal throw would miss.

## Climb faster

**Codes:** `KNVAEZIX` + `SSVAOZXY` (use both)
**Raw form:** `8AE0:FC:25` and `8AE1:D5:F2`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:01:22

**Mechanism:** `HandlePlayerState_Climbing` (`prg-0-1.asm`) sets the climb
speed each frame from a table in bank F:

```
loc_BANK0_8ADF:
	LDA ClimbSpeed, Y
	STA PlayerYVelocity
```

`ClimbSpeed` (`$F225`) is `$00`, `$20` (down, 2 pixels per frame) and `$F0`
(up, 1 pixel per frame). The two codes change the operand of the `LDA` from
`$F225` to `$D5FC`. That address is in fixed bank E, inside the DPCM sample
data, and holds `$00 $40 $DB`. Down becomes `$40` (4 pixels per frame, 2x).
Up becomes `$DB` (about 2.3 pixels per frame, 2.3x).

The `ClimbSpeed` table itself does not change, because the vine area
transition also uses it. That transition stops when the player is at an
exact Y position, and a faster speed could skip past it. With these codes,
the transition still runs at normal speed.

**Evidence:** Patches both operand bytes of `LDA ClimbSpeed, Y` at `$8ADF`.
The source and the built PRG0 ROM both have `B9 25 F2` there. No other bank
has `$25` or `$F2` at these offsets. Bank E is always mapped at `$C000`
(MMC3 PRG mode 0), and `$D5FC` holds `00 40 DB` in all three revisions.

**Revisions:** Same bytes, same data at `$D5FC`, and no other bank match in
PRG0, PRG1 and Super Mario USA.

**Caveats:**

- Use both codes. With only one, the `LDA` reads from a wrong address and
  the climb speeds will be random values.
- The vine area transition (climbing off the top or bottom of the screen)
  runs at the normal speed.
- Up + Down together reads the next byte, as in the normal game's "climb
  zip" bug. That byte is `$A2` here (`$A5` in the normal game), so the zip
  is about the same.

## Speed up 3x faster when you walk or run

**Code:** `GEEANKAP`
**Raw form:** `8C87:04:18` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:02:17

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** When Left or Right is held, `sub_BANK0_8C1A`
(`prg-0-1.asm`) adds a step to the X speed:

```
	LDA PlayerXVelocity
	CLC
	ADC PlayerControlAcceleration, Y
	STA PlayerXVelocity
```

`PlayerControlAcceleration` (`$8C18`) is `$FE` (left) and `$02` (right).
The code changes the low byte of the `ADC` operand from `$18` to `$04`, so
it reads from `$8C04` instead. `$8C04` is in the same bank and holds
`$F6 $06`: the operand of an `STA` in `HandlePlayerState_ChangingSize`. The
step becomes -6 or +6, three times the normal value. You reach top speed in
about a third of the time.

The table itself can't be changed. Bank 6 has `$FE` at the same offset
(`$8C18`), inside an `LDA` operand in
`CreateObject_GreenJumpthroughPlatformTile`, so a code on that byte would
also break green platforms.

**Evidence:** Patches the operand low byte of `ADC PlayerControlAcceleration,
Y`. The source and the built PRG0 ROM both have `79 18 8C` at `$8C86` and
`F6 06` at `$8C04`. No other bank has `$18` at `$8C87`. The code and the
data it reads are both in bank 0, which is mapped while this code runs.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Only speeding up. Slowing down when you let go uses
  `PlayerXDeceleration` and does not change.
- Top speed does not change. `sub_BANK0_8DC0` still limits the X speed.
- The same code runs in the air, so you also change direction faster in a
  jump.
- On ice, the step is still applied only every 4th frame.

## All solid blocks are jump-through for the player

**Code:** `PPVPGEAE`
**Raw form:** `9064:19:08` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:03:48

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `CheckTileUsesCollisionType` (`prg-0-1.asm`) tells the
player collision code whether a tile is solid in a given way. Y is the
collision type: 0 = solid to mushroom blocks, 1 = solid on top
(jump-through), 2 = solid on all sides. `JumpthroughTileCollisionTable`
uses type 1 when the player moves down and type 2 for up, left and right.

```
	; add the offset for the type of collision we're checking
	ADC TileGroupTable, Y
	TAY

	; check which side of the tile ID pivot we're on
	PLA
	CMP TileSolidnessTable, Y
```

`TileGroupTable` is `$00`, `$04`, `$08`. Each picks 4 bytes in
`TileSolidnessTable` (`$F64E`), one per tile group (`$00-$3F`, `$40-$7F`,
`$80-$BF`, `$C0-$FF`). A tile is solid if its ID is at least the byte for
its group. The code changes the type 2 entry from `$08` to `$19`, so the
"all sides" check reads 4 bytes at `$F667` instead: `$7E $F6 $C4 $F5` (code
bytes in `UpdateJoypads`). No tile in any group reaches its new limit,
except the unused tiles `$F5-$FF`. Nothing is solid on all sides any more,
but tiles that are solid on top still are. The player can jump up through
blocks and walk through walls, and still lands on top of blocks.

**Evidence:** Patches the third byte of `TileGroupTable`. The source and the
built PRG0 ROM both have `00 04 08` at `$9062`. No other bank has `$08` at
this offset. `$F667` is in fixed bank F and holds `7E F6 C4 F5` in all three
revisions.

**Revisions:** Same bytes, same data at `$F667`, and no other bank match in
PRG0, PRG1 and Super Mario USA.

**Caveats:**

- Only the player. Enemies use `TileGroupTable_Bank3`, which does not
  change.
- The throw check in `sub_BANK0_8DC0` also uses type 2. You can now throw
  bombs, POW blocks and other "blocks" into a wall.
- Inside a wall, the player's feet may land on the top of a block in the
  wall. Expect jumpy movement when you walk through thick walls. Not
  tested.

## All ground is a conveyor belt to the left

**Code:** `AAVASNAA`
**Raw form:** `8F65:00:08` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:04:32

**Mechanism:** When the player stands on ground, `PlayerTileCollision`
(`prg-0-1.asm`) checks for a conveyor tile:

```
PlayerTileCollision_CheckConveyorTile:
	LSR byte_RAM_A
	BCC PlayerTileCollision_CheckSlipperyTile

	LDX byte_RAM_A
	LDA ConveyorSpeedTable, X
	STA PlayerXAcceleration
```

`byte_RAM_A` is `$00` on normal ground, `$01` on a left conveyor and `$03`
on a right conveyor. After the `LSR`, X is `$00` (left) or `$01` (right).
`ConveyorSpeedTable` is `$F0` (left) and `$10` (right). The code changes the
`BCC` offset (`$08`) to `$00`, so the branch goes to the next instruction
either way. On normal ground X is `$00`, so the ground pushes the player
left like a left conveyor.

**Evidence:** Patches the operand of `BCC PlayerTileCollision_CheckSlipperyTile`.
The source and the built PRG0 ROM both have `46 0A 90 08` at `$8F62`. No
other bank has `$08` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Right conveyors still push right.
- Only the player. Enemies and objects use `ObjectConveyorSpeedTable` in
  bank 3, which does not change.
- Only when standing on background tiles, not on enemies or objects.

## All ground is a conveyor belt to the right

**Codes:** `AAVASNAA` + `AONEETEY` (use both)
**Raw form:** `8F65:00:08` and `8EF8:10:F0`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:04:32

Tags: `demo-ready` `play-start`

**Mechanism:** `AAVASNAA` makes all ground act as a conveyor (see "All
ground is a conveyor belt to the left"). On normal ground it reads
`ConveyorSpeedTable` entry 0, the left speed (`$F0`). `AONEETEY` changes
that entry to `$10`, the right speed. All ground and both kinds of conveyor
now push the player right.

**Evidence:** `BCC` operand `$08` at `$8F65` and `ConveyorSpeedTable` entry
0 (`$F0`) at `$8EF8`. Both match the source and the built PRG0 ROM. No other
bank has the same byte at either offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- `AONEETEY` alone makes left conveyors push right, and does nothing else.
- Only the player, and only when standing on background tiles.

## Every pulled item is the same item

One shared code plus one code per item (use both):

| Item | Codes | Raw form of the second code |
|------|-------|-----------------------------|
| Rocket | `EXXPZAKZ` + `YEXPLEPA` | `90A3:07:09` |
| Shell | `EXXPZAKZ` + `AEXPLEPE` | `90A3:08:09` |
| Bomb | `EXXPZAKZ` + `PEXPLEPE` | `90A3:09:09` |
| POW block | `EXXPZAKZ` + `LEXPLEPA` | `90A3:03:09` |
| Magic potion | `EXXPZAKZ` + `ZEXPLEPE` | `90A3:0A:09` |
| 1-up mushroom | `EXXPZAKZ` + `LEXPLEPE` + `AOKOGAEY` | `90A3:0B:09`, `90CC:10:F0` |

**Raw form of the shared code:** `90A2:A0:A4`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:05:37

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** When the player pulls grass or lifts a block from the
background, the code puts an index in `byte_RAM_9` and jumps to
`loc_BANK0_9074` (`prg-0-1.asm`). That code finds a free object slot and
sets the object type from `PickUpToEnemyTypeTable`:

```
	LDA #EnemyState_Alive
	LDY byte_RAM_9
	CPY #$0E
	BNE loc_BANK0_90AE
	...
loc_BANK0_90AE:
	STA EnemyState, X
	LDA PickUpToEnemyTypeTable, Y
	STA ObjectType, X
```

The shared code changes `LDY byte_RAM_9` (`A4 09`) to `LDY #imm`
(`A0 09`). The second code sets the immediate value, which is the index in
`PickUpToEnemyTypeTable`: `$07` rocket, `$08` shell, `$09` bomb, `$03` POW
block, `$0A` potion, `$0B` 1-up. Every pulled vegetable, coin, mushroom
block, POW block and item becomes that item.

**Evidence:** Patches `LDY byte_RAM_9` at `$90A2`. The source and the built
PRG0 ROM both have `A4 09 C0 0E` there. No other bank has `$A4` at `$90A2`
or `$09` at `$90A3`.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Use the shared code with exactly one item code. The shared code alone
  gives index `$09`, the bomb, because the old operand stays.
- Digging sand also gives the item. Index `$0E` (sand) is no longer seen, so
  the sand dig no longer uses `EnemyState_Sand`.
- The rest of the code still runs. A bomb gets a full fuse.
- Without `AOKOGAEY`, only the first 1-up works. When you collect a 1-up,
  `Award1upMushroom` does `INC Mushroom1upPulled`. After that,
  `loc_BANK0_90C5` turns each pulled 1-up into a small vegetable. The flag
  is cleared in `ResetAreaAndProcessGameMode_NotTitleCard` with the other
  once-per-level flags. `AOKOGAEY` changes the `BEQ` after
  `LDA Mushroom1upPulled` (`F0 1C` at `$90CC`) to `BPL`. The flag is only
  incremented, so it stays below `$80` and the branch is always taken. No
  other bank has `$F0` at `$90CC` in any of the three ROMs.
- The rocket flies to the area that the level data sets for the current
  page. On a page with no rocket destination, where you land is not known.
  Not tested.
- Lifting enemies uses a different path and does not change.

## Mario jumps and falls like Luigi

**Code:** `TXKEPLAE`
**Raw form:** `83C9:2E:00` (address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 19:07:15

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `CopyCharacterStatsAndStuff` (`prg-a-b.asm`) copies 23 bytes
of `CharacterStats` to `CharacterStatsRAM`. The start of the row comes from
`StatOffsets`:

```
	LDX CurrentCharacter
	LDY StatOffsets, X
	LDX #$00
loc_BANKA_8458:
	LDA CharacterStats, Y
	STA CharacterStatsRAM, X
```

`StatOffsets` is `$00` (Mario), `$45` (Princess), `$17` (Toad) and `$2E`
(Luigi). This code changes Mario's entry to `$2E`, so Mario gets Luigi's
whole row. Luigi's jump speeds are lower (`$D6` standing instead of `$B0`),
and his gravity is much lower (`$01` with A held instead of `$04`, `$02`
without A instead of `$07`). His jump is a little higher and much slower,
and he falls slowly.

**Evidence:** Patches the first byte of `StatOffsets`. The source and the
built PRG0 ROM both have `00 45 17 2E` at `$83C9` in bank A. No other even
bank has `$00` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The whole row changes, not only the jump. Mario also gets Luigi's pick-up
  speed (the same as Mario's) and his run speed while carrying (`$16`
  instead of `$18`).
- The stats are copied at the character select screen. The code does not
  change a game that is already running.
- The flutter animation is a separate code (`AANEXILA` for Mario, or
  `AANEKIGL` for everyone).
- Luigi's float code (`GLEEAGAE`) now also applies to Mario, because it
  patches Luigi's row. This is a way to give Mario a float, but he also gets
  Luigi's jump.

## Toad jumps and falls like Luigi

**Code:** `TXKELLYO`
**Raw form:** `83CB:2E:17` (address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 19:07:15

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** Same as "Mario jumps and falls like Luigi", but changes
Toad's `StatOffsets` entry from `$17` to `$2E`. Toad gets Luigi's whole
stats row: Luigi's jump speeds and his low gravity.

**Evidence:** Patches the third byte of `StatOffsets`. The source and the
built PRG0 ROM both have `00 45 17 2E` at `$83C9` in bank A. No other even
bank has `$17` at `$83CB`.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The whole row changes. Toad loses his fast pick-up speed and his faster
  run speed while carrying.
- The Toad float code (`GUNAPLAE`) no longer works, because Toad no longer
  reads his own row. Luigi's float code (`GLEEAGAE`) applies instead.
- The stats are copied at the character select screen.
- The flutter animation is a separate code (`ZANEXILA` for Toad, or
  `AANEKIGL` for everyone).

## Princess jumps and falls like Luigi

**Code:** `TXKEZLIK`
**Raw form:** `83CA:2E:45` (address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 19:07:15

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** Same as "Mario jumps and falls like Luigi", but changes the
Princess's `StatOffsets` entry from `$45` to `$2E`. She gets Luigi's whole
stats row: Luigi's jump speeds and his low gravity.

**Evidence:** Patches the second byte of `StatOffsets`. The source and the
built PRG0 ROM both have `00 45 17 2E` at `$83C9` in bank A. No other even
bank has `$45` at `$83CA`.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- **She loses her float.** Luigi's Floating Time is `$00`. Add Luigi's float
  code (`GLEEAGAE`) to get it back.
- The whole row changes. She also gets Luigi's faster pick-up speed and his
  run speed while carrying.
- The stats are copied at the character select screen.
- The flutter animation is a separate code (`PANEXILA` for the Princess, or
  `AANEKIGL` for everyone).

## The flying carpet never runs out

**Code:** `SLNZZLVI`
**Raw form:** `A372:B5:D6` (address:new-byte:compare-byte, bank 3)
**Recorded:** 2026-10-02 19:08:26

**Mechanism:** `CreateFlyingCarpet` (`prg-2-3.asm`) sets the carpet's life
in `EnemyArray_B1` to `$A0`. `EnemyBehavior_FlyingCarpet` counts it down
once every 4 frames, and removes the carpet at zero:

```
	LDA byte_RAM_10
	AND #$03
	BNE loc_BANK3_A37C

	DEC EnemyArray_B1, X
	BNE loc_BANK3_A37C

	STA PlayerRidingCarpet
	JMP EnemyDestroy
```

The code changes `DEC EnemyArray_B1, X` (`$D6`) to `LDA EnemyArray_B1, X`
(`$B5`). The life no longer goes down. A is the life value (`$A0`), which is
not zero, so the `BNE` always branches and the carpet is never removed.

**Evidence:** Patches the opcode of `DEC EnemyArray_B1, X` in
`EnemyBehavior_FlyingCarpet`. The source and the built PRG0 ROM both have
`D6 B1 D0 06` at `$A372`. No other odd bank has `$D6` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- You still need to get the carpet from a Pidgit first. This does not put
  you on a carpet.
- The carpet is kept even when it goes offscreen (the game already does
  this). Not checked whether it carries over to a new area.
- Jumping off still works. Land on the carpet again to ride it.

## Every jar you can enter is a warp jar

**Code:** `AAVASLTA`
**Raw form:** `8B65:00:06` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:09:33

**Mechanism:** When the player has gone down into a jar,
`HandlePlayerState_GoingDownJar` (`prg-0-1.asm`) picks what to do from
`InJarType` ($00 = warp, $01 = regular, $02 = pointer):

```
	LDA InJarType
	BNE HandlePlayerState_GoingDownJar_NonWarp

	LDA #GameMode_Warp
	STA GameMode
	RTS
```

The code changes the `BNE` offset (`$06`) to `$00`, so the branch goes to
the next instruction either way. Every jar sets `GameMode_Warp`, and the
game shows the warp screen.

**Evidence:** Patches the operand of
`BNE HandlePlayerState_GoingDownJar_NonWarp`. The source and the built PRG0
ROM both have `AD EE 04 D0 06` at `$8B61`. No other bank has `$06` at this
offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Only jars you can already enter. Jars that can't be entered still can't.
  (In the normal game, those become enterable in Subspace and are the real
  warp jars.)
- The destination comes from `WarpDestinations`: World 1 goes to 4, 3 to 5,
  4 to 6, 5 to 7. Worlds 2, 6 and 7 "warp" to the start of the same world.

## Dig through any floor like sand

**Code:** `AAVOPPGA`
**Raw form:** `9169:00:04` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:10:13

Tags: `demo-ready` `play-start`

**Mechanism:** When the player stands on ground and presses B,
`TileBehavior_CheckPickUp` (`prg-0-1.asm`) checks the tile under the
player's feet:

```
	LDA byte_RAM_0
	CMP #BackgroundTile_DiggableSand
	BNE loc_BANK0_916E

	LDA #$0E
	BNE loc_BANK0_9177
```

Index `$0E` makes the lift code create a sand object in `EnemyState_Sand`
and replace the tile with `BackgroundTile_Sky`. The code changes the `BNE`
offset (`$04`) to `$00`, so the branch goes to the next instruction either
way. Every floor tile is dug like sand.

**Evidence:** Patches the operand of `BNE loc_BANK0_916E` after
`CMP #BackgroundTile_DiggableSand`. The source and the built PRG0 ROM both
have `C9 93 D0 04` at `$9166`. No other bank has `$04` at this offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Any floor: bricks, ground, jar tops, bridges. You can dig a hole to the
  bottom of the area and fall out of it.
- Mushroom blocks, POW blocks and other liftable tiles also turn into sand,
  so you can't lift them from the ground.
- **Pulling grass mostly stops working.** The dig runs whenever B is pressed
  and the player is lined up with the tile grid (12 of every 16 pixels).
  Grass is pulled only in the other 4 pixels.
- The dig only runs on tiles you stand on, not on enemies or objects.

## Lift any floor tile as a mushroom block

**Codes:** `AAVOPPGA` + `EXXPZAKZ` + `AEXPLEPA` (use all three)
**Raw form:** `9169:00:04`, `90A2:A0:A4` and `90A3:00:09`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:10:42

Tags: `demo-ready` `play-start`

**Mechanism:** This joins two cheats above.

- `AAVOPPGA` ("Dig through any floor like sand") makes B dig any floor
  tile. In the normal game, this gives index `$0E`, a sand object that
  can't be thrown.
- `EXXPZAKZ` + `AEXPLEPA` change `LDY byte_RAM_9` in the lift code to
  `LDY #$00` (see "Every pulled item is the same item"). Index `$00` in
  `PickUpToEnemyTypeTable` is `Enemy_MushroomBlock`. Because Y is not
  `$0E`, the object starts in `EnemyState_Alive`, not `EnemyState_Sand`.

Together, B lifts the floor tile under you as a mushroom block that you can
throw. The tile becomes `BackgroundTile_Sky`.

**Evidence:** See the two entries above. All three bytes match the source
and the built PRG0 ROM, and no other bank has the same byte at any of the
three offsets.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- For another object, swap `AEXPLEPA` for one of the item codes in "Every
  pulled item is the same item" (for example `LEXPLEPA` for a POW block).
- Pulled grass also gives a mushroom block, and grass is pulled only in 4
  of every 16 pixels (see the dig entry).
- You can dig a hole to the bottom of the area and fall out of it.

## Start every area on a flying carpet

**Codes (horizontal areas):** `SKEEYAGE` + `PKXATEAL` + `AOUELAEN` +
`PEOZPUPX` (use all four)
**Codes (all areas):** the four above + `SXUALASP` + `TOUAGAIO` (use all
six)
**Raw form:** `808F:CD:04`, `80A6:41:38`, `80BB:18:F0` (bank 2),
`A391:09:29` (bank 3), and for all areas also `80B3:A5:95`, `80B4:1E:15`
(bank 2)
(address:new-byte:compare-byte)
**Recorded:** 2026-10-02 19:14:48

**Mechanism:** Each area starts with the player on a flying carpet. The
player can't get off until the carpet runs out, about 4.3 seconds later.

- `SKEEYAGE`: `AreaInitialization_CheckObjectCarriedOver` starts with
  `LDA ObjectCarriedOver` (`AD AF 04`). This changes it to `LDA $CDAF`. That
  byte is `$41` (`Enemy_FlyingCarpet`) in fixed bank E. Every area now
  starts as if the player carried a carpet in, in object slot 5.
- `PKXATEAL`: changes `CMP #Enemy_Rocket` to `CMP #Enemy_FlyingCarpet`, so
  the carpet uses the rocket set-up code, not the "player carries it" code.
  That code sets the carpet's life (`EnemyArray_B1`) to `$41` and puts it
  at X `$78`, Y `$E0` on page 0.
- `AOUELAEN`: changes the rocket's `LDA #$F0` to `LDA #$18`. The same value
  sets Y velocity and, after `ASL A`, the Y position. The carpet now starts
  at Y `$30` (top of the screen) and moves down a little before it stops.
- `PEOZPUPX`: in `EnemyBehavior_FlyingCarpet`, changes `AND #$20` to
  `ORA #$20` in the test for "player stands on carpet". The result is
  never zero, so `PlayerRidingCarpet` is always set. Each frame the code
  moves the carpet to the player's X and the player to the carpet's Y. The
  player rides from the first frame, wherever they are.
- The carpet's life is `$41`. `EnemyBehavior_FlyingCarpet` decrements it
  every 4 frames, so it lasts 260 frames (about 4.3 seconds). At 0 the code
  clears `PlayerRidingCarpet` and destroys the carpet. The player falls
  and can use doors, jars and vines again.
- `SXUALASP` + `TOUAGAIO`: change `STA ObjectXHi, X` to `LDA PlayerYHi`, so
  the next `STA ObjectYHi, X` puts the carpet on the player's Y page, not
  page 0. In a vertical area, page 0 is the top of the whole area. With
  these codes, the player starts near the top of their own page. The
  carpet's X page is not set, but the carpet moves to the player on the
  first frame. In horizontal areas, leave these out: if the player enters
  from above the screen, their Y page is `$FF`, and the carpet would start
  above the screen.

The game keeps flying carpets alive when they are off screen
(`HandleEnemyScreenBounds`), so the carpet is not removed before it reaches
the player.

**Evidence:** `AreaInitialization` and `EnemyBehavior_FlyingCarpet` in
`src/prg-2-3.asm`. All bytes match the source and the built PRG0 ROM. No
other bank has the same byte at any of these offsets. (`80B1`, the
`LDA #$00` opcode, was not used: bank 0 has `A9` at the same offset in
scroll code.)

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA. `$CDAF` is `$41` in all three.

**Caveats:**

- Jumping does not get you off the carpet. The carpet catches the player
  again on the next frame.
- With `SLNZZLVI` ("Flying carpet never runs out") added, the carpet never
  runs out, so you can never get off. Then you can't go through doors.
- Anything carried into an area (a key, a vegetable, the rocket ride) is
  replaced by the carpet. A key no longer brings Phanto with it.
- The carpet's graphics are in the enemy CHR set. In worlds where Pidgit's
  graphics are not loaded, the carpet may look wrong.
- Doors, jars, vines and pits with a forced carpet ride were not checked.

## Doors all take you to Bonus Chance

**Codes:** `EXNOGPEG` + `AENOYPIA` (use both)
**Raw form:** `91FC:A0:C0` and `91FF:00:05`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:16:00

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** When the player presses up at a door, `loc_BANK0_91EB` checks
`CPY #$04` (`BackgroundTile_LightDoorEndLevel`, the door at the end of a
level). If it matches, it does `DEY` and `STY GameMode`, which sets
`GameMode` to `$03` (Bonus Chance).

- `EXNOGPEG` changes `CPY #$04` to `LDY #$04`, so every door counts as the
  end-of-level door.
- `AENOYPIA` changes the `BNE` operand to `$00`. `LDY #$04` clears the zero
  flag, so without this the `BNE` would still go to the normal door code.

Up at any door now goes to Bonus Chance. It does not end the level. After
Bonus Chance, `GoToNextLevel` calls `FollowCurrentAreaPointer`, which reads
the area pointer for the page you are on (`AreaPointersByPage`). At the
real end-of-level door, that pointer goes to the next level. At other doors,
it usually goes to an area of the same level.

**Evidence:** `loc_BANK0_91EB` and `DoorTiles` in `src/prg-0-1.asm`. Both
bytes match the source and the built PRG0 ROM, and no other bank has the
same byte at either offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- This includes locked doors (with or without a key), dark doors and the
  subspace doors made by a potion. You can't enter subspace.
- Holding anything other than a key still stops you from going through a
  door. Doors in jars are not checked by this code.
- After Bonus Chance, you may start the area in the middle of a wall and
  get stuck. Seen in testing. Probable cause (not checked): the area starts
  from the character select screen with `TransitionType` 0, so the player
  is placed as at the start of a level, not at a door.

## Open locked doors without a key

**Code:** `TLOPAXAX`
**Raw form:** `9210:3E:28` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:16:37

**Mechanism:** `DoorHandlingPointers` entry 1 (locked door) points to
`DoorHandling_LockedDoor` (`$9228`). That code checks that you carry the
key, sets `KeyUsed`, turns the key into smoke, plays the locked-door
animation and goes through the door. This changes the low byte of the
pointer so it points to `$923E`, the `JSR DoorAnimation_Locked` near the
end. The key checks are skipped, and the door opens.

**Evidence:** `DoorHandlingPointers` and `DoorHandling_LockedDoor` in
`src/prg-0-1.asm`. The byte matches the source and the built PRG0 ROM, and
no other bank has the same byte at this offset.

**Revisions:** Same byte and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- `KeyUsed` is not set, so the door is still locked when you come back, and
  the key still appears. The code opens it again.
- If you carry the key through, it is not used up. It goes with you, and
  Phanto follows.
- Holding anything other than a key still stops you from going through a
  door.
- The jump does not start at `INC KeyUsed` (`$9236`) because the code after
  it uses Y (the door type, here 1) as the key's object slot and would turn
  object 1 into smoke.

## Phanto chases you without the key

**Codes (PRG0):** `NOVEIGEX` + `PPXPAUZO`
**Codes (PRG1 and Super Mario USA):** `NOVEIGEX` + `PPXPGLSU`
**Raw form:** `84ED:9F:A0` (all), `9320:19:1A` (PRG0), `9324:19:1A` (PRG1
and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:17:38

**Mechanism:**

- `NOVEIGEX`: `EnemyInit_Phanto` sets `PhantoActivateTimer` to `$A0`, the
  value that holds Phanto asleep until you pick up the key. This makes it
  `$9F`, so the timer counts down at once: Phanto flashes, shakes and
  plays its sound, then wakes up after about 2.7 seconds.
- `PPXPAUZO` / `PPXPGLSU`: `EnemyBehavior_Phanto` loads `LDY #$01` (move
  away from the player), then `BEQ Phanto_Movement` if you hold nothing.
  This changes the branch operand from `$1A` to `$19`, so it goes to
  `Phanto_AfterStartTimer` instead. That is the `DEY` that sets Y to `$00`
  (move toward the player).

**Evidence:** `EnemyInit_Phanto` and `EnemyBehavior_Phanto` in
`src/prg-2-3.asm`. The bytes match the source and each built ROM. The
behavior code is 4 bytes later in PRG1 and Super Mario USA (`REV_A` code
earlier in bank 2), so the branch needs a different code there. No other
bank in any of the three ROMs has `$1A` at `$9320` or `$9324`, or `$A0` at
`$84ED`, so both branch codes can be on together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- Phanto exists only in rooms with a key, and in areas you carry the key
  into. This does not add Phanto to other areas.
- If you hold an item that is not the key, Phanto still moves away, as in
  the normal game.

## Every area starts with a rocket ride

**Codes:** `ESEETENX` + `SKEEYAGE` + `SEXELESO` + `ALOAXPPK` (use all four)
**Raw form:** `808E:D8:AF`, `808F:CD:04`, `80AB:8D:9D` and `8912:38:41`
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:18:46

**Mechanism:**

- `ESEETENX` + `SKEEYAGE`: `AreaInitialization_CheckObjectCarriedOver`
  starts with `LDA ObjectCarriedOver`. This changes it to `LDA $CDD8`. That
  byte is `$38` (`Enemy_Rocket`) in fixed bank E. Every area now runs
  `AreaInitialization_Rocket`, as if the player had ridden a rocket in.
- `SEXELESO`: `AreaInitialization_Rocket` has `STA PlayerInRocket, X` with
  X = 5 (the source marks it "Bug?"). It writes `ObjectXAcceleration`
  instead. In the normal game this does not matter, because
  `PlayerInRocket` is still set from the launch. This changes it to
  `STA PlayerInRocket`, so the player is in the rocket. The rocket drops
  the player off near the top of the screen.
- `ALOAXPPK`: `ScreenSpriteClipping_Vertical` keeps a few object types
  alive when they are off screen. This changes `CPY #Enemy_FlyingCarpet`
  to `CPY #Enemy_Rocket`. The rocket is always placed at X `$78` on page 0.
  If the area starts far from there, the rocket would be removed before it
  drops the player off, and the player would stay stuck "in the rocket".

**Evidence:** `AreaInitialization` and `ScreenSpriteClipping_Vertical` in
`src/prg-2-3.asm`, and `EnemyBehavior_Rocket_DroppingOff`. All bytes match
the source and the built PRG0 ROM. No other bank has the same byte at any
of these offsets.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA. `$CDD8` is `$38` in all three.

**Caveats:**

- The rocket is not placed at the player. It flies on page 0 (off screen
  in most areas), and the player appears at their own X position when it
  drops them off. The player's Y page does not change.
- Anything carried into an area (a key, a vegetable) is replaced by the
  rocket. A key no longer brings Phanto with it.
- While `ALOAXPPK` is on, flying carpets (Pidgit's) are removed when off
  screen.
- `SKEEYAGE` is also in "Start every area on a flying carpet". Don't use
  the two cheats together.

## Enemies spawn anywhere on the next page (horizontal areas)

**Codes:** `SEUATZSG` + `AEUEPZNT` (use both)
**Raw form:** `82B6:85:C5` and `82B9:00:E7`
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:20:11

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** In a horizontal area, `CheckObjectHorizontalSpawnBoundaries`
finds the column just off the screen edge you move toward
(`SpawnBoundaryOffsets`) and stores it in `byte_RAM_5` (X column) and
`byte_RAM_6` (page). It then reads the enemy data for that page, and spawns
an enemy only if its X column equals `byte_RAM_5`.

- `SEUATZSG` changes `CMP byte_RAM_5` to `STA byte_RAM_5`. The enemy's own
  X column is stored there, and the spawn code later uses it as the
  enemy's X position.
- `AEUEPZNT` changes the `BNE` operand after it to `$00`, so the column
  test never skips the enemy.

Every enemy on the page at the spawn boundary now spawns at its own
position, one per frame, while there is a free enemy slot. Enemies on other
pages still wait until the spawn boundary reaches their page.

**Evidence:** `CheckObjectHorizontalSpawnBoundaries_InitializePage_InitializeObject`
and `_CreateObject` in `src/prg-2-3.asm`. Both bytes match the source and
the built PRG0 ROM, and no other bank has the same byte at either offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Vertical areas use separate code and are not changed.
- An enemy can spawn far off screen (up to about one screen past the
  edge). If it is past the despawn range, it may be removed and spawn
  again over and over until you get closer.
- Enemies behind you on that page can also spawn.
- A swarm or generator object anywhere on that page also starts at once.

## Thrown objects hit enemies up to 128 pixels away

**Codes:** `OVNLLIGE` + `EEONYEEY` + `EEUYLEEY` + `NNKYYAAX` + `NNSNLAGX`
(use all five)
**Raw form:** `B5F3:E9:04` (bank 3), `F09F:80:F8`, `F0B3:80:F8`,
`F0C7:FF:20`, `F0DB:FF:24` (bank F)
(address:new-byte:compare-byte)
**Recorded:** 2026-10-02 19:22:30

Tags: `demo-ready` `play-start` `curated`

**Mechanism:**

- `OVNLLIGE`: in `CheckObjectCollision`, a thrown object
  (`ObjectProjectileTimer` not zero) takes `BNE
  CheckObjectCollision_CheckObjectBeingCarried`, then loads its own hitbox.
  This changes the branch to go back to the `LDY #$06` used for a bomb
  explosion, so thrown objects use hitbox `$06`.
- The other four codes change hitbox `$06` in the
  `ObjectCollisionHitbox*` tables: left `$F8` → `$80`, top `$F8` → `$80`,
  width `$20` → `$FF`, height `$24` → `$FF`. The box now reaches from 128
  pixels left of and above the object to 127 pixels right of and below it.

`CheckHitboxCollisionDimension` compares 16-bit positions, but the width
and height are 8 bits, so a box can't be bigger than this. A thrown object
hits every enemy within about 128 pixels, which is most of the screen.

**Evidence:** `CheckObjectCollision`, `CheckHitboxCollisionDimension` in
`src/prg-2-3.asm` and the `ObjectCollisionHitbox*` tables in
`src/prg-e-f.asm`. All bytes match the source and the built PRG0 ROM, and
no other bank has the same byte at any of these offsets.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Bomb explosions use the same hitbox, so they also reach about 128
  pixels. An exploding bomb can hurt you from most of the screen away.
- The hitbox tables are copied to RAM when a level is loaded. If you turn
  the codes on during a level, the bigger box starts at the next level.
- After a thrown enemy has flown for 32 frames, it can collide with the
  player again. With the big box, it may hurt you if it is still on
  screen.

## Spawn multiple copies of each enemy

**Codes:** `AEOEALEA` + `GSOEGUKS` (use both)
**Raw form:** `8398:00:80` and `839C:5C:DC`
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:23:27

Tags: `demo-ready` `play-start` `curated` `challenge`

**Mechanism:** When an enemy spawns,
`CheckObjectSpawnBoundaries_InitializePage_MarkEnemyData` sets bit 7 of its
byte in the area's enemy data (`ORA #%10000000`). The spawn code skips
enemies with bit 7 set, so each enemy spawns once.

- `AEOEALEA` changes `ORA #%10000000` to `ORA #$00`, so the enemy is never
  marked. It spawns again on each check while it is at the spawn boundary.
- `GSOEGUKS`: the next instruction is `CMP #%10000000 | Enemy_BossBirdo`,
  which tests for a boss so the boss can make the exit door. Without bit
  7, no enemy would pass. This changes it to `CMP #Enemy_BossBirdo`.

**Evidence:** `CheckObjectSpawnBoundaries_InitializePage_MarkEnemyData` in
`src/prg-2-3.asm`. Both bytes match the source and the built PRG0 ROM, and
no other bank has the same byte at either offset. (`STA (RawEnemyData), Y`
at `$8399` was not used: bank 4 has `$91` at the same offset.)

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Copies spawn about once per frame until all 5 enemy slots are full, so
  you mostly get a stack of the same enemy at one spot. Other enemies wait
  for a free slot.
- This applies to every spawned object, including bosses, keys, potions
  and other items placed in the enemy data.
- Without `GSOEGUKS`, bosses don't make the exit door.

## Enemies use horizontal physics in place of vertical physics

**Code (PRG0):** `AAKOVVZA`
**Code (PRG1 and Super Mario USA):** `AASPXVZA`
**Raw form:** `9E4E:00:0A` (PRG0), `9E52:00:0A` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:24:06

Tags: `demo-ready` `play-start`

**Mechanism:** Object physics use one routine for both axes.
`ApplyObjectPhysicsY` does `TXA` / `CLC` / `ADC #$0A` / `TAX` and falls into
`ApplyObjectPhysicsX`. The Y arrays (`ObjectYVelocity`, `ObjectYLo`, ...)
are `$0A` bytes after the X arrays, so index X + `$0A` reads the Y values.
This changes the index offset to `$00`.

When an object asks for vertical physics, it gets horizontal physics
instead. Objects that run both now move twice as fast left and right, and
never move up or down from their own physics: they don't fall and can't
jump.

**Evidence:** `ApplyObjectPhysicsY` in `src/prg-2-3.asm`. The byte matches
the source and each built ROM. The routine is 4 bytes later in PRG1 and
Super Mario USA (the `REV_A` change earlier in bank 2), so each needs its
own code. No other bank in any of the three ROMs has `$0A` at `$9E4E` or
`$9E52`, so both codes can be on together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- Other offsets were not used. With `$09` or `$0B`, each object's vertical
  physics would run on the slot before or after it, and the first or last
  slot would reach RAM outside the object arrays (`$09` reaches the
  player's Y values).
- Some objects set their Y position directly and still move up and down.
- Objects that rely on falling (thrown items, bombs, Bob-ombs walking off
  ledges) float in the air.

## Enemies use vertical physics in place of horizontal physics

**Code (PRG0):** `LGNEIIAS`
**Code (PRG1 and Super Mario USA):** `YGNEIIGS`
**Raw form:** `857D:4B:50` (PRG0), `857D:4F:54` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 23:34:03

Tags: `demo-ready` `play-start`

**Mechanism:** `ApplyObjectMovement` does `JSR ApplyObjectPhysicsX` and
then `JSR ApplyObjectPhysicsY`. This changes the low byte of the first
`JSR` so that it also calls `ApplyObjectPhysicsY`.

Objects that move with `ApplyObjectMovement` (most enemies) now get
vertical physics twice and no horizontal physics. They fall and jump twice
as fast, and never move left or right from their own physics.

This is the opposite of "Enemies use horizontal physics in place of
vertical physics". That code changes the shared routine, so it affects
every caller. This code changes only one call. The vertical entry falls
into the horizontal code, so one code can't change the horizontal entry
for every caller.

**Evidence:** `ApplyObjectMovement` and `ApplyObjectPhysicsY` in
`src/prg-2-3.asm`. The byte matches the source and each built ROM. The
`JSR` is at the same address in all three ROMs, but its target is 4 bytes
later in PRG1 and Super Mario USA (the `REV_A` change earlier in bank 2),
so each needs its own code. No other bank in any of the three ROMs has
`$50` or `$54` at `$857D`, so both codes can be on together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- About 25 other places call `ApplyObjectPhysicsX` directly (for example
  Sparks, the flying carpet, Wart's bubbles, and some bosses). Those still
  move left and right.
- Objects that shake (`ObjectShakeTimer`) skip the first `JSR`, so they
  get vertical physics once, as normal.
- The effect on each enemy was not checked. Walking enemies will probably
  stand still and fall faster.

## Enemies move the next enemy left and right in place of themselves

**Codes:** `ETNAYSSU` + `PZNEASZK` + `AANEPIGA` (use all three)
**Raw form:** `8577:E8:BD`, `8578:29:4A` and `8579:00:04`
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 23:41:53

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `ApplyObjectMovement` starts with `LDA ObjectShakeTimer, X`
and `BNE ApplyObjectMovement_Vertical`, then calls `ApplyObjectPhysicsX`
and `ApplyObjectPhysicsY`. These codes change the first 3 bytes to `INX` /
`AND #$00`. `AND #$00` sets the zero flag, so the `BNE` is never taken.
`ApplyObjectPhysicsX` then runs on object slot X + 1.

Each object that moves with `ApplyObjectMovement` (most enemies) now
applies its horizontal physics to the next slot. It does not move left or
right itself, unless the slot before it does the same.

**Why only horizontal:** `ApplyObjectPhysicsX` ends with `LDX
byte_RAM_12`, so the index is back to the object's own slot before the
`JSR ApplyObjectPhysicsY` and the gravity step. Each of those would need its
own `INX`, and there is no room for them. Changing `byte_RAM_12` instead
would hang the game: `AreaMainRoutine_ObjectLoop` counts down from it, so
it would run the same object again and again.

**Evidence:** `ApplyObjectMovement` and `ApplyObjectPhysicsX` in
`src/prg-2-3.asm`. The bytes match the source and each built ROM. No other
bank in any of the three ROMs has the same byte at any of the three
offsets.

**Revisions:** Same codes for PRG0, PRG1 and Super Mario USA.

**Caveats:**

- Slot 8 is the last slot, so its next index is 9. Index 9 of the X arrays
  is the player's Y values (`PlayerYVelocity`, `PlayerYLo`, ...). An object
  in slot 8 applies the player's vertical physics a second time, so the
  player may fall or jump faster. In horizontal areas this also changes
  `PlayerYHi`.
- For slot 8 the code also writes 0 to `ObjectNonSticky` + 9, which is
  `EnemiesKilledForHeart` (`$04AD`). This may delay the heart that appears
  after you defeat enough enemies.
- Shaking objects (`ObjectShakeTimer`) now also move the next slot. Before,
  they skipped horizontal physics.
- About 25 other places call `ApplyObjectPhysicsX` directly, for example
  Sparks, the flying carpet and Wart's bubbles. These still move
  themselves.
- The effect in play was not checked.

## Enemies move the next enemy up and down in place of themselves

**Codes:** `ETNAYSSU` + `OZNEASZK` (use both)
**Raw form:** `8577:E8:BD` and `8578:A9:4A`
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 23:44:23

**Mechanism:** `ApplyObjectMovement` starts with `LDA ObjectShakeTimer, X`
and `BNE ApplyObjectMovement_Vertical`. These codes change the first 3
bytes to `INX` / `LDA #$04`. `LDA #$04` clears the zero flag, so the `BNE`
is always taken. It skips `JSR ApplyObjectPhysicsX` and goes to `JSR
ApplyObjectPhysicsY`, which now runs on object slot X + 1.

Each object that moves with `ApplyObjectMovement` (most enemies) now
applies its vertical physics to the next slot. It does not move up or down
itself, unless the slot before it does the same. `ApplyObjectPhysicsY`
ends with `LDX byte_RAM_12`, so the gravity step after it still changes
the object's own `ObjectYVelocity`.

**Why horizontal physics are lost:** the `INX` has to come before `JSR
ApplyObjectPhysicsY`, and `JSR ApplyObjectPhysicsX` is just before it. That
call resets X to the object's own slot, so it must be skipped. These
objects don't move left or right from `ApplyObjectMovement` at all.

**Evidence:** `ApplyObjectMovement` and `ApplyObjectPhysicsY` in
`src/prg-2-3.asm`. The bytes match the source and each built ROM. No other
bank in any of the three ROMs has the same byte at either offset.

**Revisions:** Same codes for PRG0, PRG1 and Super Mario USA.

**Caveats:**

- `ETNAYSSU` is also in "Enemies move the next enemy left and right in
  place of themselves", but the second byte is different. Don't use both
  cheats together.
- Slot 8 is the last slot, so it reaches index `$13` of the X arrays. This
  reads `PlayerState` as the velocity and `QuicksandDepth` as the
  acceleration. When both are 0 nothing changes. In other cases it adds to
  `PlayerLock` (`$041B`), and the carry can change `PlayerXVelocity` and
  `PlayerXLo`. This may lock the player or move them.
- Shaking objects (`ObjectShakeTimer`) now also move the next slot.
- Other places that call `ApplyObjectPhysicsY` directly still move the
  object itself.
- The effect in play was not checked.

## Lift any enemy

**Codes:** `AEXUOAZA` + `AEEUXAPA` (use both; see caveats for using only
the first)
**Raw form:** `B8A9:00:02` and `B88A:00:01`
(address:new-byte:compare-byte, bank 3)
**Recorded:** 2026-10-02 19:25:04

**Mechanism:** Both codes are in `CheckCollisionWithPlayer` (`prg-2-3.asm`).

- `AEXUOAZA`: `CheckCollisionWithPlayer_StandingOnHead` tests
  `AND #SpriteFlags46E_Unliftable` before it checks B. This changes the
  mask to `$00`, so no enemy counts as unliftable.
- `AEEUXAPA`: before that, the game tests `AND #SpriteFlags46E_Damage` and
  hurts the player if the enemy is spiky or hot (for example Porcupo,
  Spark, Panser and several bosses). Those enemies are also unliftable, so without
  this code you can't stand on them long enough to lift them. This changes
  the mask to `$00`, so you can stand on them.

**Evidence:** `CheckCollisionWithPlayer` and
`CheckCollisionWithPlayer_StandingOnHead` in `src/prg-2-3.asm`. Both bytes
match the source and the built PRG0 ROM, and no other bank has the same
byte at either offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- `AEXUOAZA` alone lets you lift unliftable enemies that are safe to
  stand on, and keeps spiky enemies harmful.
- With `AEEUXAPA`, landing on any enemy from above no longer hurts you.
  Touching them from the side still does.
- Some objects were never meant to be carried (bosses, flying carpets,
  Hawkmouth, Phanto). Lifting them may look wrong or break the area. Use
  with "Lift an enemy from any side" to lift without standing on top.

## Thrown objects never hit the player

**Code:** `AEKAUOYP`
**Raw form:** `89C3:00:1F` (address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:26:01

**Mechanism:** A thrown object gets `ObjectProjectileTimer` = `$01`.
`HandleEnemyState_Alive` then counts it up by 1 each frame until it
reaches `$20`:

```
	LDY ObjectProjectileTimer, X
	DEY
	CPY #$1F
	BCS loc_BANK2_89C9
	INC ObjectProjectileTimer, X
```

This changes `CPY #$1F` to `CPY #$00`, so the `INC` never runs and the
timer stays at its lowest value, `$01`. The object is still a projectile.
`CheckObjectCollision` skips the player while the timer is below `$20`
(the "post-throw grace period"), so thrown objects never touch the player.

**Evidence:** `HandleEnemyState_Alive` and
`CheckObjectCollision_PlayerCollisionEnabled` in `src/prg-2-3.asm`. The byte
matches the source and the built PRG0 ROM, and no other bank has the same
byte at this offset.

**Revisions:** Same byte and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Some objects clear the timer when they land (for example in
  `loc_BANK3_A6ED`). Any object that keeps it can't touch the player, so
  you can't stand on it or lift it.
- Thrown objects still hit other enemies.

## Walking enemies move 4x as fast

**Codes:** `AXNAZSAA` + `EVNALSEY` (use both)
**Raw form:** `85F2:20:08` and `85F3:E0:F8`
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:26:52

Tags: `demo-ready` `play-start` `curated` `challenge`

**Mechanism:** `EnemyInit_BasicMovement` sets an enemy's X velocity from
`EnemyInitialAccelerationTable`: index 1 (`$08`, right) or index 2 (`$F8`,
left). `EnemyBehavior_TurnAround` only negates the velocity, so the speed
stays the same for the enemy's life. These codes change the two entries to
`$20` and `$E0`, 4 times the normal speed. Enemies with
`SpriteFlags46E_DoubleSpeed` still double it, to `$40` / `$C0`.

**Evidence:** `EnemyInit_BasicMovement`, `EnemyInitialAccelerationTable`
and `EnemyBehavior_TurnAround` in `src/prg-2-3.asm`. Both bytes match the
source and the built PRG0 ROM, and no other bank has the same byte at
either offset.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Every enemy that uses `EnemyInit_BasicMovement` is faster, not only
  Shyguys and other basic walkers.
- The table is shared with `ExplosionTileYOffsets` (the source notes
  this). Two tiles of each bomb explosion are drawn 24 pixels further up
  or down.

## Bullets and eggs move at 1/4 speed

**Codes (PRG0):** `AAUOGGAX` + `EYUOIGEV`
**Codes (PRG1 and Super Mario USA):** `AAKPAGAX` + `EYKPPGEV`
**Raw form:** `943C:08:20` + `943D:F8:E0` (PRG0), `9440:08:20` +
`9441:F8:E0` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:27:32

**Mechanism:** `CreateBullet_WithSlotInY` sets the new object's X velocity
from `BulletProjectileXSpeeds` (`$20` right, `$E0` left). These codes change
the two entries to `$08` and `$F8`, 1/4 of the normal speed.

The same routine makes every shot in the game:

- Snifit bullets (`EnemyBehavior_Snifit`, and gray Snifits that shoot when
  they jump)
- Birdo eggs and fireballs (`BirdoBehavior_SpitProjectile`)
- Cobrat shots from jars (`EnemyBehavior_CobratJar_Shoot`)
- Autobomb fire

**Evidence:** `CreateBullet` and `BulletProjectileXSpeeds` in
`src/prg-2-3.asm`. The bytes match the source and each built ROM. The table
is 4 bytes later in PRG1 and Super Mario USA (the `REV_A` change earlier in
bank 2), so each needs its own codes. No other bank in any of the three
ROMs has the compare bytes at these offsets, so all four codes can be on
together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- Birdo's fireballs and Autobomb fire are slow too.
- An egg you throw back moves at normal throw speed.

## Swarm enemies appear almost constantly

**Code (PRG0):** `AKKANLLA`
**Code (PRG1 and Super Mario USA):** `AKKEULLA`
**Raw form:** `8BC7:40:03` (PRG0), `8BCB:40:03` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:28:56

Tags: `demo-ready` `play-start` `curated` `challenge`

**Mechanism:** While a swarm is active (Albatoss carrying Bob-ombs, or
diving Beezos), `Swarm_CreateEnemy` adds `$03` to `SwarmCounter` each frame
and makes a new enemy when the counter overflows, about every 85 frames.
This changes the `ADC #$03` to `ADC #$40`, so a new enemy comes every 4
frames while there is a free enemy slot.

**Evidence:** `Swarm_CreateEnemy` in `src/prg-2-3.asm`. The byte matches the
source and each built ROM. The routine is 4 bytes later in PRG1 and Super
Mario USA (the `REV_A` change earlier in bank 2), so each needs its own
code. No other bank in any of the three ROMs has `$03` at `$8BC7` or
`$8BCB`, so both codes can be on together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- `CreateEnemy` uses only the first 6 enemy slots, so the swarm stops when
  they are full and starts again when one is free.

## A swarm is always active

**Albatoss carrying Bob-ombs:** `KSNAYPGA` + `AONATOOU`
**Diving Beezos:** `KSNAYPGA` + `AENATOOL`
**Wart's vegetable thrower:** `KSNAYPGA` + `LKNATOOU`
**Raw form:** `81F7:D4:04` and `81F6:18:B9` (Albatoss), `81F6:00:B9`
(Beezo) or `81F6:4B:B9` (vegetable thrower)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:28:56 (vegetable thrower: 2026-10-03 00:02:55)

Tags: `demo-ready` `play-start` `curated` `challenge`

**Mechanism:** After the object loop, the game does `LDA SwarmType`
(`AD B9 04`). If it is zero, there is no swarm. If not, `InitializeSwarm`
subtracts `$47` (the first generator type) and jumps through
`GeneratorInitializationTable`: `$47` is Albatoss, `$48` is Beezo, `$4A`
is the vegetable thrower from Wart's room.

These codes change the address to a byte in fixed bank E that already
holds the value: `KSNAYPGA` sets the high byte to `$D4`, and the low-byte
code picks `$D418` (`$47`), `$D400` (`$48`) or `$D44B` (`$4A`). The game
now reads the same swarm type every frame.

**Evidence:** `InitializeSwarm` and `GeneratorInitializationTable` in
`src/prg-2-3.asm`. All bytes match the source and the built PRG0 ROM, and
no other bank has the same byte at either offset. `$D418` is `$47`,
`$D400` is `$48` and `$D44B` is `$4A` in all three ROMs.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The swarm runs in every area, including vertical areas, jars, subspace
  and boss rooms.
- A swarm set by the area's enemy data is ignored, and `Swarm_Stop` (the
  enemy data object that ends a swarm) no longer works.
- Use only one of the low-byte codes at a time.
- The vegetable thrower (`Generator_VegetableThrower`) makes one large
  vegetable every 256 frames (about 4.3 seconds), and only when you are not
  holding an item. It always puts the vegetable on page 2, at the positions
  for Wart's room. In other areas it may be off screen or in a wall.
- Works with "Swarm enemies appear almost constantly" (Albatoss and Beezo
  only; the vegetable thrower uses the frame counter, not the swarm
  counter).

## Stationary enemies walk

**Code (PRG0):** `SLNAOISP` + `SUUOPGSP` + `SUKPTLSP`
**Code (PRG1 and Super Mario USA):** `SLNASISP` + `SUUOIGSP` + `SUKOZLSP`
**Raw form:** `8D71:B5:95` + `94B9:B5:95` + `93C6:B5:95` (PRG0),
`8D75:B5:95` + `94BD:B5:95` + `93CA:B5:95` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:31:08

**Mechanism:** Gray Snifits already use `EnemyBehavior_BasicWalker`, and
red and gray Pansers already call `ApplyObjectMovement`. They stand still
only because `EnemyInit_Stationary` calls `EnemyInit_Basic` (which sets an
X velocity toward the player) and then does `STA ObjectXVelocity, X` with
`A = $00`. The first code changes that `STA` (`$95`) to `LDA` (`$B5`),
so the enemy keeps the walking velocity from `EnemyInit_Basic`.

`EnemyBehavior_BasicWalker` also sets a gray Snifit's X velocity to 0 each
frame unless it was thrown. The second code changes that `STA` to `LDA`.

Jumping Ninjis also use `EnemyBehavior_BasicWalker`, but they set their X
velocity to 0 each time they land. The third code changes that `STA` to
`LDA` too, so they walk between jumps.

**Evidence:** `EnemyInit_Stationary`, `EnemyBehavior_NinjiJumping`,
`EnemyBehavior_BasicWalker`, `EnemyBehavior_PanserRedAndGray` and
`EnemyBehaviorPointerTable` in `src/prg-2-3.asm`. The bytes match the source and each built ROM. The code
is 4 bytes later in PRG1 and Super Mario USA (the `REV_A` change earlier in
bank 2), so each needs its own codes. No other bank in any of the three ROMs
has `$95` at these addresses, so all six codes can be on together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- The first code alone makes red and gray Pansers walk. Gray Snifits need
  the first two codes. The third code is only for jumping Ninjis.
- `EnemyInit_Stationary` is also used for mushrooms, Hawkmouths, keys,
  Trouters and falling logs. These also get an X velocity now. Mushrooms
  and keys may slide, and Trouters may drift sideways.
- Falling logs store `A` in `ObjectStunTimer` right after this call. With
  the first code, that value is the X velocity, not `$00`, so falling logs
  may act differently.

## Basic walker enemies move like Trouters

**Code (PRG0):** `GGVOIGAX` + `NAVOTKSY` + `VAVOYGKU`
**Code (PRG1 and Super Mario USA):** `GGNPPGAX` + `UANPZKSN` + `VANPLGKU`
**Raw form:** `946D:4C:20` + `946E:87:FD` + `946F:8E:B4` (PRG0),
`9471:4C:20` + `9472:8B:FD` + `9473:8E:B4` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:32:24

Tags: `demo-ready` `play-start`

**Mechanism:** `EnemyBehavior_BasicWalker` starts with
`JSR ObjectTileCollision` (`20 FD B4`). These codes change it to
`JMP EnemyBehavior_Trouter` (`4C 87 8E` in PRG0, `4C 8B 8E` in PRG1 and
Super Mario USA), so every enemy that uses the basic walker code runs the
Trouter code in its place.

The Trouter code does no tile collision. The enemy falls through the
ground until it reaches the Trouter's lowest point (`TrouterMaxY`: `$EA`
in horizontal areas, `$92` in vertical areas). When `ObjectTimer1` is 0,
it jumps up with a velocity from `TrouterJumpVelocityY`, then waits
`$C0` frames before the next jump. The table index (`EnemyArray_B1`) is
0 for these enemies, because they did not run `EnemyInit_Trouter`, so
they all use the highest jump (`$AC`).

**Evidence:** `EnemyBehavior_BasicWalker`, `EnemyBehavior_Trouter`,
`EnemyInit_Trouter` and `EnemyBehaviorPointerTable` in
`src/prg-2-3.asm`. The bytes match the source and each built ROM. The code
is 4 bytes later in PRG1 and Super Mario USA (the `REV_A` change earlier in
bank 2), so each needs its own codes. No other bank in any of the three
ROMs has the same byte at these addresses, so all six codes can be on
together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- This affects Shyguys, Tweeters, Porcupos and Snifits (enemy types `$01`
  to `$07`), and also Ninjis and walking Bob-ombs, which jump to the same
  code.
- The enemies use the Trouter's palette and flip upside down when falling,
  like a Trouter.
- They keep their X velocity, so they also move left or right. Without
  tile collision they do not turn at walls.
- The first code is the same as in the bomb and Ostro cheats. Use only
  one of these three cheats at a time.

## Basic walker enemies move like bombs

**Code (PRG0):** `GGVOIGAX` + `UGVOTKSY` + `EPVOYGKL`
**Code (PRG1 and Super Mario USA):** `GGNPPGAX` + `NGNPZKSY` + `EPNPLGKL`
**Raw form:** `946D:4C:20` + `946E:C3:FD` + `946F:90:B4` (PRG0),
`9471:4C:20` + `9472:C7:FD` + `9473:90:B4` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:33:47

**Mechanism:** Same method as "Basic walker enemies move like Trouters".
The `JSR ObjectTileCollision` at the start of `EnemyBehavior_BasicWalker`
becomes `JMP EnemyBehavior_Bomb` (`4C C3 90` in PRG0, `4C C7 90` in PRG1
and Super Mario USA).

The bomb code bounces the object when it lands fast, slows it when it hits
a wall, and sets its X velocity to 0 when it is on the ground. So the
enemies drop to the ground and stop there, like a bomb or a vegetable.

**Evidence:** `EnemyBehavior_BasicWalker`, `EnemyBehavior_Bomb` and
`EnemyBehavior_Vegetable` in `src/prg-2-3.asm`. The bytes match the
source and each built ROM. No other bank in any of the three ROMs has the
same byte at these addresses, so the PRG0 and PRG1 codes can be on
together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- They do not explode. The fuse code runs only when the object type is
  `Enemy_Bomb`.
- The bomb code does not call `EnemyBehavior_CheckDamagedInterrupt`, so
  hit enemies may not react normally.
- This affects Shyguys, Tweeters, Porcupos and Snifits, and also Ninjis
  and walking Bob-ombs.
- The first code is the same as in the Trouter and Ostro cheats. Use only
  one of these three cheats at a time.

## Basic walker enemies move like Ostros

**Code (PRG0):** `GGVOIGAX` + `IYVOTKSN` + `VZVOYGKL`
**Code (PRG1 and Super Mario USA):** `GGNPPGAX` + `IYNPZKSN` + `VZNPLGKL`
**Raw form:** `946D:4C:20` + `946E:7D:FD` + `946F:A6:B4` (PRG0),
`9471:4C:20` + `9472:7D:FD` + `9473:A6:B4` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:33:47

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** Same method as "Basic walker enemies move like Trouters".
The `JSR ObjectTileCollision` at the start of `EnemyBehavior_BasicWalker`
becomes `JMP EnemyBehavior_Ostro` (`4C 7D A6`, bank 3, which is mapped
at the same time as bank 2).

The enemies hop each time they land, turn at walls, and turn toward the
player every 64 frames, like an Ostro.

**Evidence:** `EnemyBehavior_BasicWalker`, `EnemyBehavior_Ostro` and
`CreateEnemy` in `src/prg-2-3.asm`. The bytes match the source and each
built ROM. No other bank in any of the three ROMs has the same byte at
these addresses, so the PRG0 and PRG1 codes can be on together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- The Ostro code uses `EnemyArray_B1 = 0` to mean "an Ostro with a Shyguy
  on it". These enemies have `EnemyArray_B1 = 0`, so they act like that:
  - When hit, the enemy drops a new red Shyguy (from `CreateEnemy`). That
    Shyguy also uses this code, so it can drop another one.
  - When lifted, the enemy turns into a red Shyguy and an Ostro appears in
    its place.
- This affects Shyguys, Tweeters, Porcupos and Snifits, and also Ninjis
  and walking Bob-ombs.
- The first code is the same as in the Trouter and bomb cheats. Use only
  one of these three cheats at a time.

## Bombs and mushrooms stop in mid-air

**Code (PRG0):** `AESOIEIP`
**Code (PRG1 and Super Mario USA):** `AEVPPEIP`
**Raw form:** `90DD:00:1D` (PRG0), `90E1:00:1D` (PRG1 and Super Mario
USA) (address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:34:31

**Mechanism:** `EnemyBehavior_CheckGround` (part of `EnemyBehavior_Bomb`)
does `AND #CollisionFlags_Down` and then `BEQ` past the ground code if the
object is not on the ground. This code sets the `BEQ` offset to 0, so the
object is always treated as on the ground.

On the ground, an object that falls at `$09` or faster bounces. If not,
`ResetObjectYVelocity` sets its Y velocity to 0 and moves it up to a
16-pixel row, and its X velocity is set to 0. So a dropped or thrown object
stops where it is, or bounces in the air if it was falling fast.

**Evidence:** `EnemyBehavior_Bomb`, `EnemyBehavior_CheckGround`,
`EnemyBehavior_Mushroom` and `ResetObjectYVelocity` in
`src/prg-2-3.asm`. The byte matches the source and each built ROM. The
code is 4 bytes later in PRG1 and Super Mario USA (the `REV_A` change
earlier in bank 2), so each needs its own code. No other bank in any of
the three ROMs has `$1D` at these addresses, so both codes can be on
together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- `EnemyBehavior_Bomb` is used by bombs, and by mushrooms, 1-up mushrooms
  and stopwatches when you are not holding them.
- With "Basic walker enemies move like bombs", those enemies also stop in
  mid-air.
- Bombs still explode when their timer runs out.

## Enemies face and walk away from the player

**Code (PRG0):** `EPKPNTEL`
**Code (PRG1 and Super Mario USA):** `EPKOUTEL`
**Raw form:** `9E47:90:B0` (PRG0), `9E4B:90:B0` (PRG1 and Super Mario
USA) (address:new-byte:compare-byte, bank 2)
**Recorded:** 2026-10-02 19:35:12

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `EnemyFindWhichSidePlayerIsOn` subtracts the enemy's X
position from the player's. It returns `Y = 0` if the player is to the
right and `Y = 1` if the player is to the left, using `BCS` to skip the
`INY`. This code changes the `BCS` (`$B0`) to `BCC` (`$90`), so the
result is reversed.

`EnemyInit_BasicMovementTowardPlayer` uses this result to set
`EnemyMovementDirection` and the X velocity, so enemies spawn facing and
walking away from the player.

**Evidence:** `EnemyFindWhichSidePlayerIsOn` and
`EnemyInit_BasicMovement` in `src/prg-2-3.asm`. The byte matches the
source and each built ROM. The routine is 4 bytes later in PRG1 and Super
Mario USA (the `REV_A` change earlier in bank 2), so each needs its own
code. No other bank in any of the three ROMs has `$B0` at these
addresses, so both codes can be on together.

**Revisions:** Checked in PRG0, PRG1 and Super Mario USA (see above).

**Caveats:**

- 16 places in banks 2 and 3 call this routine, and all of them get the
  reversed side. This includes Birdo, Albatoss, Ninji, Hoopstar and Cobrat
  code, so some of these may aim or turn the wrong way too.
- `byte_RAM_F` (the X distance) is not changed, so distance checks still
  work.
- Enemies that turn at walls or cliffs still turn normally after they
  spawn.

## All characters float, including Mario

**Codes:** `ZSKEVGLI` + `ZIOEKILI` (use both)
**Raw form:** `8CCE:52:53` and `8D1C:52:53`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 19:37:24

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** The Toad and Luigi codes patch the "Floating Time" byte in
`CharacterStats`. That does not work for Mario (see "Toad floats like the
Princess"). These codes change the code that reads the float time.

`PlayerStartJump` does `LDA JumpFloatLength` (`$0553`) and stores it in
`JumpFloatTimer`. `PlayerGravity` does `CMP JumpFloatLength` to see if the
float has started. Both codes change the low byte of the address from
`$53` to `$52`, so both read `JumpHeightQuicksand` (`$0552`) in place
of `JumpFloatLength`. `JumpHeightQuicksand` is `$E0` for all four
characters, so every character gets a float of `$E0` (224) frames, about
3.7 seconds.

**Evidence:** `PlayerStartJump` and `PlayerGravity` in
`src/prg-0-1.asm`, and `CharacterStats` in `src/prg-a-b.asm`. The source
and the built ROMs have `AD 53 05` at `$8CCD` and `CD 53 05` at `$8D1B`
in bank 0. No other even bank has `$53` at these offsets.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Use both codes. With only the first one, the `CMP` in `PlayerGravity`
  sees that the timer is not equal to `JumpFloatLength` and clears it, so
  there is no float.
- The float is 224 frames, not the Princess's 60 frames. The Princess's
  float is also 224 frames with these codes.
- Unlike the Toad and Luigi codes, this works in a game that is already
  running.
- The float still starts only near the top of a jump (Y velocity `$FC` or
  more).
- Works with "Float time never runs out".

## Mario walks and runs faster

**Codes:** `AXSETUAO` (right) + `ESVAPUEV` (left)
**Raw form:** `83DE:28:18` and `83E1:D8:E8`
(address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 22:31:33

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `CharacterStats` (`prg-a-b.asm`) holds the running speeds for
each character. `CopyCharacterStatsAndStuff` copies the current
character's row to `RunSpeedRight` and `RunSpeedLeft` in RAM. The speed cap
in bank 0 (`sub_BANK0_8DC0`) uses the "no object" speed (`$18`, or `$E8` to
the left) as the walking speed. When B is held, it adds half again, so the
run speed is `$24`.

These codes change Mario's "no object" speeds in `MarioStats` to `$28`
and `$D8` (−`$28`). Walking is `$28` and running is `$28 + $14 = $3C`,
about 1.67x the normal speed in both cases.

The cap check is `CMP PlayerXVelocity` followed by `BPL` or `BMI`, which
is not a true signed compare. The game also adds the walk step (2, or 6
with "Speed up 3x faster when you walk or run") before the cap runs, so the
speed can go past the cap for one frame. The check works only while the
right speed plus the left speed plus the step is `$7F` or less. If not, a
player running left suddenly gets full speed to the right. `$3C` + `$3C` +
6 = `$7E`, so these values are safe with or without the 3x cheat. A walk
speed of `$2A` (run `$3F`) breaks when running left.

**Evidence:** `MarioStats` in `src/prg-a-b.asm` and `sub_BANK0_8DC0` in
`src/prg-0-1.asm`. The source and the built PRG0 ROM have `$18` at `$83DE`
and `$E8` at `$83E1` in bank A. No other even bank has the same byte at
the left address.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA (except the bank 6 match below).

**Caveats:**

- The new value takes effect when the stats are copied, for example at the
  character select screen. It does not change a game that is already running.
- The speeds when carrying a heavy item and in quicksand do not change.
- Either code works alone, for one direction only.
- Bank 6 also has `$18` at `$83DE`, so the right code also changes that
  byte when bank 6 is mapped. That byte is in `World7BackgroundPalettes`,
  in the palette marked "Castle (unused)" (`$0F, $28, $18, $08` becomes
  `$0F, $28, $28, $08`). The game does not use that palette, so this should
  have no effect.

## Toad walks and runs faster

**Codes:** `AXNAIUAO` (right) + `ESNEAUEV` (left)
**Raw form:** `83F5:28:18` and `83F8:D8:E8`
(address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 22:31:33

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `CharacterStats` (`prg-a-b.asm`) holds the running speeds for
each character. `CopyCharacterStatsAndStuff` copies the current
character's row to `RunSpeedRight` and `RunSpeedLeft` in RAM. The speed cap
in bank 0 (`sub_BANK0_8DC0`) uses the "no object" speed (`$18`, or `$E8` to
the left) as the walking speed. When B is held, it adds half again, so the
run speed is `$24`.

These codes change Toad's "no object" speeds in `ToadStats` to `$28`
and `$D8` (−`$28`). Walking is `$28` and running is `$28 + $14 = $3C`,
about 1.67x the normal speed in both cases.

The cap check is `CMP PlayerXVelocity` followed by `BPL` or `BMI`, which
is not a true signed compare. The game also adds the walk step (2, or 6
with "Speed up 3x faster when you walk or run") before the cap runs, so the
speed can go past the cap for one frame. The check works only while the
right speed plus the left speed plus the step is `$7F` or less. If not, a
player running left suddenly gets full speed to the right. `$3C` + `$3C` +
6 = `$7E`, so these values are safe with or without the 3x cheat. A walk
speed of `$2A` (run `$3F`) breaks when running left.

**Evidence:** `ToadStats` in `src/prg-a-b.asm` and `sub_BANK0_8DC0` in
`src/prg-0-1.asm`. The source and the built PRG0 ROM have `$18` at `$83F5`
and `$E8` at `$83F8` in bank A. No other even bank has the same byte at either
address.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The new value takes effect when the stats are copied, for example at the
  character select screen. It does not change a game that is already running.
- The speeds when carrying a heavy item and in quicksand do not change.
- Either code works alone, for one direction only.

## Luigi walks and runs faster

**Codes:** `AZEEGKAO` (right) + `EIEEYKEV` (left)
**Raw form:** `840C:28:18` and `840F:D8:E8`
(address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 22:31:33

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `CharacterStats` (`prg-a-b.asm`) holds the running speeds for
each character. `CopyCharacterStatsAndStuff` copies the current
character's row to `RunSpeedRight` and `RunSpeedLeft` in RAM. The speed cap
in bank 0 (`sub_BANK0_8DC0`) uses the "no object" speed (`$18`, or `$E8` to
the left) as the walking speed. When B is held, it adds half again, so the
run speed is `$24`.

These codes change Luigi's "no object" speeds in `LuigiStats` to `$28`
and `$D8` (−`$28`). Walking is `$28` and running is `$28 + $14 = $3C`,
about 1.67x the normal speed in both cases.

The cap check is `CMP PlayerXVelocity` followed by `BPL` or `BMI`, which
is not a true signed compare. The game also adds the walk step (2, or 6
with "Speed up 3x faster when you walk or run") before the cap runs, so the
speed can go past the cap for one frame. The check works only while the
right speed plus the left speed plus the step is `$7F` or less. If not, a
player running left suddenly gets full speed to the right. `$3C` + `$3C` +
6 = `$7E`, so these values are safe with or without the 3x cheat. A walk
speed of `$2A` (run `$3F`) breaks when running left.

**Evidence:** `LuigiStats` in `src/prg-a-b.asm` and `sub_BANK0_8DC0` in
`src/prg-0-1.asm`. The source and the built PRG0 ROM have `$18` at `$840C`
and `$E8` at `$840F` in bank A. No other even bank has the same byte at either
address.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The new value takes effect when the stats are copied, for example at the
  character select screen. It does not change a game that is already running.
- The speeds when carrying a heavy item and in quicksand do not change.
- Either code works alone, for one direction only.

## Princess walks and runs faster

**Codes:** `AZXALKAO` (right) + `EIXATKEV` (left)
**Raw form:** `8423:28:18` and `8426:D8:E8`
(address:new-byte:compare-byte, bank A)
**Recorded:** 2026-10-02 22:31:33

Tags: `demo-ready` `play-start` `curated`

**Mechanism:** `CharacterStats` (`prg-a-b.asm`) holds the running speeds for
each character. `CopyCharacterStatsAndStuff` copies the current
character's row to `RunSpeedRight` and `RunSpeedLeft` in RAM. The speed cap
in bank 0 (`sub_BANK0_8DC0`) uses the "no object" speed (`$18`, or `$E8` to
the left) as the walking speed. When B is held, it adds half again, so the
run speed is `$24`.

These codes change Princess's "no object" speeds in `PrincessStats` to `$28`
and `$D8` (−`$28`). Walking is `$28` and running is `$28 + $14 = $3C`,
about 1.67x the normal speed in both cases.

The cap check is `CMP PlayerXVelocity` followed by `BPL` or `BMI`, which
is not a true signed compare. The game also adds the walk step (2, or 6
with "Speed up 3x faster when you walk or run") before the cap runs, so the
speed can go past the cap for one frame. The check works only while the
right speed plus the left speed plus the step is `$7F` or less. If not, a
player running left suddenly gets full speed to the right. `$3C` + `$3C` +
6 = `$7E`, so these values are safe with or without the 3x cheat. A walk
speed of `$2A` (run `$3F`) breaks when running left.

**Evidence:** `PrincessStats` in `src/prg-a-b.asm` and `sub_BANK0_8DC0` in
`src/prg-0-1.asm`. The source and the built PRG0 ROM have `$18` at `$8423`
and `$E8` at `$8426` in bank A. No other even bank has the same byte at either
address.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- The new value takes effect when the stats are copied, for example at the
  character select screen. It does not change a game that is already running.
- The speeds when carrying a heavy item and in quicksand do not change.
- Either code works alone, for one direction only.

## Background objects are solid for the player

**Codes:** `AAVPGEAA` + `AAVPLAGA` (use both)
**Raw form:** `9064:00:08` and `9063:00:04`
(address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-02 23:17:35

Tags: `demo-ready` `play-start`

**Mechanism:** `CheckTileUsesCollisionType` (`prg-0-1.asm`) uses
`TileGroupTable` (`$00 $04 $08`) to pick 4 limits in
`TileSolidnessTable` (`$F64E`), one per tile group (`$00-$3F`,
`$40-$7F`, `$80-$BF`, `$C0-$FF`). A tile is solid in that way if its
ID is at least the limit for its group:

| Type | Offset | Limits | Meaning |
|---|---|---|---|
| 0 | `$00` | `$01 $43 $80 $C0` | solid to mushroom blocks |
| 1 | `$04` | `$12 $60 $91 $CA` | solid on top |
| 2 | `$08` | `$18 $69 $98 $D5` | solid on all sides |

The player uses type 1 when moving down and type 2 for up, left and right.
Tiles from the type 0 limit up to the type 1 or type 2 limit are
background objects: the player passes through them, but mushroom blocks
don't.

- `AAVPGEAA` changes the type 2 offset from `$08` to `$00`. Background
  objects now block the player from the sides and from below.
- `AAVPLAGA` changes the type 1 offset from `$04` to `$00`. The player
  can now also stand on them.

Only tile `$00` and tiles `$40-$42` (sky) stay empty.

**Evidence:** `CheckTileUsesCollisionType` and `TileGroupTable` in
`src/prg-0-1.asm`, `TileSolidnessTable` in `src/prg-e-f.asm`, and the
`BackgroundTile_` names in `src/defs.asm`. The source and the built ROMs
have `00 04 08` at `$9062`. No other even bank has `$04` at `$9063` or
`$08` at `$9064`.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Not tested. Many things are background objects. These now become solid
  for the player:
  - Background clouds, houses, trees, palm trunks, stars, pyramids and
    background bricks
  - Water and waterfalls
  - Grass you can pull (all kinds), and cherries
  - Doors, the light door and its trail, and dark doors
  - Ladders, vines, chains and `BackgroundTile_ClimbableSky`
  - Quicksand
- So the player may be unable to walk past grass, collect cherries, go
  through doors, or climb. Areas that fill the sky with
  `ClimbableSky` may trap the player.
- Only the player. Enemies use `TileGroupTable_Bank3`, which does not
  change.
- The throw check in `sub_BANK0_8DC0` also uses type 2, so you can't throw
  some items next to a background object.
- `AAVPGEAA` ($9064) is the same byte as "All solid blocks are
  jump-through for the player" (`PPVPGEAE`). Don't use both.

## No music

**Code:** `ATVAPUSZ`
**Raw form:** `8361:60:AD` (address:new-byte:compare-byte, bank 4)
**Recorded:** 2026-10-12 22:57:37

**Mechanism:** Each frame, `StartProcessingSoundQueue` (`prg-4-5.asm`)
calls `ProcessMusicQueue` once, from `ProcessOnlyMusicQueue2`. That routine
starts any queued song and then plays the current song's notes:

```
ProcessMusicQueue:
	LDA MusicQueue2
	BMI ProcessMusicQueue_StopMusic
```

The code changes the first opcode, `LDA` (`$AD`, absolute), to `RTS`
(`$60`). `ProcessMusicQueue` now returns at once, so no song ever starts and
`MusicPlaying1` and `MusicPlaying2` stay `$00`. The music queues are still
cleared after the call, as in the normal game. Sound effects and DPCM
samples are played by their own routines before this call, so they still
play.

**Evidence:** Patches the first opcode of `ProcessMusicQueue`. The source
and the built ROMs have `AD 03 06` at `$8361`. No other bank has `$AD` at
this offset. `ProcessMusicQueue` has only one caller.

**Revisions:** Same bytes and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Not tested.
- The rocket and POW rumble noise in `ProcessSoundEffectQueue3` reads
  bytes from `ProcessMusicQueue` as "random" noise values. It reads with
  Y = 1 to `$7F`, never Y = 0, so it never reads the patched byte. Those
  sounds don't change.
- The game checks `MusicPlaying2` for `Music2_BossClearFanfare` to lock
  the player during the boss clear fanfare (`RunFrame_Horizontal`,
  `RunFrame_Vertical` and `sub_BANKF_F0F9`). With no music this value is
  never set, so the player can move during that time.
- `RespawnPlayer` checks `MusicPlaying1` for the invincibility song. That
  check never matches now, which has no effect without music.

## The game runs at half speed

**Code (PRG0):** `PEUVXZAA` + `GSEVKUTS`
**Code (PRG1 and Super Mario USA):** `PEKTEZAA` + `ZVOTXLGT`
**Raw form:** `EABA:01:00` + `EB8C:5C:5E` (PRG0),
`EAC0:01:00` + `EB92:62:64` (PRG1 and Super Mario USA)
(address:new-byte:compare-byte, bank F)
**Recorded:** 2026-10-12 22:59:08

**Mechanism:** The game loop syncs with the NMI through `WaitForNMI`
(`prg-e-f.asm`). It sets `NMIWaitFlag` to `$00` and waits until the flag
is negative:

```
	LDA #$00
	STA NMIWaitFlag ; Start waiting for NMI to finish
WaitForNMILoop:
	LDA NMIWaitFlag ; Has the NMI routine set the flag yet?
	BPL WaitForNMILoop ; If no, wait some more
```

During gameplay, `NMI` does the full frame update only if the flag is
`$00`. Then it does `DEC NMIWaitFlag`, so the flag becomes `$FF` and the
game loop goes on. If the flag is not `$00` (the game loop is still busy),
it goes to `NMI_Waiting`, which only restores `PPUMASK` and jumps to
`NMI_DoSoundProcessing`. That is a normal lag frame.

The first code changes `LDA #$00` in `WaitForNMI` to `LDA #$01`, so the
flag starts at `$01`. The second code changes the target of the `JMP` in
`NMI_Waiting` from `NMI_DoSoundProcessing` to the `DEC NMIWaitFlag` just
before it. Each game frame now takes two NMIs:

1. The flag is `$01`, so the NMI takes the `NMI_Waiting` path (a lag
   frame). It decrements the flag to `$00`. `WaitForNMILoop` still waits,
   because `$00` is not negative.
2. The flag is `$00`, so the NMI does the full update and decrements the
   flag to `$FF`. The game loop goes on.

The pause/menu path (`NMI_PauseOrMenu`) and `NMI_Transition` also end with
`DEC NMIWaitFlag`, so they also take two NMIs per game frame.

**Evidence:** `WaitForNMI`, `NMI`, `NMI_Waiting` and the comment block above
`NMI`. The source and the built ROMs have `A9 00 85 EB` at `$EAB9` (PRG0)
and `$EABF` (PRG1 and Super Mario USA), and `4C 5E EC` at `$EB8B` (PRG0) or
`4C 64 EC` at `$EB91` (PRG1 and Super Mario USA). `DEC NMIWaitFlag`
(`C6 EB`) is at `$EC5C` (PRG0) and `$EC62` (PRG1 and Super Mario USA).
Bank F is fixed at `$E000`, so no other bank can match.

**Revisions:** PRG1 and Super Mario USA move this code by 6 bytes, so each
has its own codes. Each set's compare bytes don't match in the other
revisions, so the wrong set does nothing.

**Caveats:**

- Not tested.
- Use both codes. The first code alone freezes the game: the flag never
  reaches `$00`, so the NMI never does the full update or the `DEC`. The
  second code alone has no effect, except that lag frames now also
  decrement the flag. `WaitForNMI` sets the flag again each frame, so this
  does no harm.
- The audio engine runs in every NMI, so music and sound effects play at
  normal speed. Only the game logic and the screen update slow down.
- Every other frame is a lag frame, so the screen updates at 30 fps.
- The title screen, the ending and the Mario sleeping scene have their own
  wait loops (`WaitForNMI_TitleScreen`, `WaitForNMI_Ending` and
  `WaitForNMI_MarioSleeping`). They still set the flag to `$00`, so they run
  at normal speed.
- Pause should still work, at half speed. It uses `WaitForNMI` too.

## All characters pick up items instantly

**Code:** `SXUEEZSA`
**Raw form:** `8AB8:A5:85` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-25 18:49:25

**Mechanism:** `HandlePlayerState_Lifting` (`src/prg-0-1.asm`) runs the
lifting animation for every character. It waits until `PlayerStateTimer`
is 0, then moves to the next step of `ObjectBeingCarriedTimer` and sets the
wait for that step:

```
	LDA PlayerLiftTimer - 2, Y
	BPL loc_BANK0_8AB8

loc_BANK0_8AB5:
	LDA PickupSpeedAnimation - 2, Y

loc_BANK0_8AB8:
	STA PlayerStateTimer
	RTS
```

`PickupSpeedAnimation` is the character's pick-up speed (copied from the
character stats), and `PlayerLiftTimer` is the shared table for small
vegetables and objects in state 6. Both paths end at the same `STA`. The
code changes `STA PlayerStateTimer` (`$85`) to `LDA PlayerStateTimer`
(`$A5`). The routine only gets here when the timer is already 0, so it
stays 0, and the animation moves one step every frame for every character
and every object. The stats tables are not changed.

**Evidence:** The source and the built PRG0 ROM both have
`B9 C0 8A 10 03 B9 44 05 85 82 60` at `$8AAD` (`PlayerStateTimer` is
`$82`). `nes_gg.py` finds no other even bank with `$85` at `$8AB8`.

**Revisions:** Same byte and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Not tested.
- The lift still has its 6 steps (`ObjectBeingCarriedTimer` 7 down to 2),
  so it takes 6 frames instead of none. That is about 1/10 of a second.
- The pull sound still plays at the start of each lift.
- Toad's speed advantage is gone, since every character uses the same
  1-frame wait.

## The same tuft of grass can be uprooted multiple times

**Code:** `GXNPLAAX`
**Raw form:** `90F3:2C:20` (address:new-byte:compare-byte, bank 0)
**Recorded:** 2026-10-25 18:51:08

**Mechanism:** When the player pulls something out of the ground,
`loc_BANK0_90EA` (`src/prg-0-1.asm`) sets up the new object and then
clears the tile it came from:

```
	LDA #BackgroundTile_Sky
	JSR ReplaceTile_Bank0
```

`ReplaceTile_Bank0` writes the sky tile into the decoded level data and
queues the PPU update, so the grass is gone. The code changes the `JSR`
(`$20`) to `BIT` (`$2C`), so the call is skipped and the tile stays as it
was. The player can throw the item and pull the same tuft again. Each
pull spawns the item from `PickUpToEnemyTypeTable` for that tile, as the
first pull did.

**Evidence:** The source and the built PRG0 ROM both have
`20 EB B9 A9 04 95 5B A9 40 20 4F 93 A9 07 95 A8` at `$90EA`
(`BackgroundTile_Sky` is `$40`, `ReplaceTile_Bank0` is at `$934F`).
`BIT $934F` only reads ROM and sets flags; the next instruction is
`LDA #$07`. `nes_gg.py` finds no other even bank with `$20` at `$90F3`.

**Revisions:** Same byte and no other bank match in PRG0, PRG1 and Super
Mario USA.

**Caveats:**

- Not tested.
- This is the shared pick-up path, so every liftable tile stays: mushroom
  blocks, POW blocks and the other pull-out items, not only grass.
- Digging in sand also uses this path, so sand no longer goes away. The
  player can't dig down through sand. Turn the code off in the World 2
  and other areas where you have to dig to go on.
- Big vegetables still count toward `BigVeggiesPulled`, so the fifth pull
  from any tuft gives a Stopwatch, as before.
- A tuft with a 1-up mushroom gives a small vegetable after the first
  time, since `Mushroom1upPulled` is set.

## Short square notes play with the electric piano instrument

**Codes:** `ZLKATTIZ` + `TZKEZTPX` (use both)
**Raw form:** `8646:32:25` + `864A:2E:21`
(address:new-byte:compare-byte, bank 4)
**Recorded:** 2026-11-01 19:48:16

**Mechanism:** This is the SMB2 version of the SMB1 and Zelda codes
"Music uses different instruments" and "All music plays with a thin,
nasal instrument". Each square note has a patch (instrument) in its
high nibble. Every frame, `LoadSquareInstrumentDVE` (`src/prg-4-5.asm`)
picks the instrument's duty/volume table by patch:

```
LoadSquareInstrumentDVE:
	CPX #$90
	BEQ LoadSquareInstrumentDVE_90_E0

	CPX #$E0
	BEQ LoadSquareInstrumentDVE_90_E0

	CPX #$A0
	BEQ LoadSquareInstrumentDVE_A0
```

Patches `$90` and `$E0` share the "short square" (25% duty with a
pronounced decay). Patch `$A0` is the "electric piano" (12.5% duty with
a pronounced decay). The codes change the first two `BEQ` offsets so both
go to `LoadSquareInstrumentDVE_A0` (`$8679`): `$25` to `$32` and `$21` to
`$2E`. `$90` is the most-used patch in the music data, and `$E0` is the
title song's patch. So the title, character select, overworld, star and
subspace songs now use the thin electric piano for most of their notes.

**Evidence:** The source and the built PRG0 ROM both have
`E0 90 F0 25 E0 E0 F0 21 E0 A0 F0 2A` at `$8643`, and the `A0` branch goes
to `$8679`. `nes_gg.py` finds no other bank with `$25` at `$8646`. Bank A
also has `$21` at `$864A`, in `UnusedText_Blank214D`, which the game
never uses.

**Revisions:** PRG0 only; PRG1 and Super Mario USA not checked.

**Caveats:**

- Tested by the user: works. The overworld song starts with the
  electric piano anyway (patch `$A0`) and then plays patch `$80`, so the
  change takes a while to hear in 1-1.
- Square 2 still plays slightly detuned while Square 1 uses patch `$E0`
  (`PlayNote_CheckSquareChorus`). That check reads the patch, not the
  instrument, so the "honky tonk" effect stays.
- The other patches (`$80`, `$B0`, `$C0`, `$D0`, `$F0`) don't change, so
  the ending, the underground song and parts of other songs sound as
  before.
- Sound effects don't use this routine and don't change.

## Music and sound effects play an octave higher

**Code:** `ZESEITPA`
**Raw form:** `86DD:02:01` (address:new-byte:compare-byte, bank 4)
**Recorded:** 2026-11-01 19:48:16

Tags: `demo-ready` `play-start` `curated` `video-ready`

**Mechanism:** `PlayNote` (`src/prg-4-5.asm`) keeps only one octave of
periods in `NoteFrequencyData`. It works out the octave from the note
number and halves the period once per octave:

```
PlayNote_NotRest:
	LDY #$01
	STY NextOctave
	...
PlayNote_FrequencyOctaveLoop:
	LSR NextFrequencyHi
	ROR NextFrequencyLo
	DEC NextOctave
	BNE PlayNote_FrequencyOctaveLoop
```

The code changes the starting count from `$01` to `$02`, so every period
is halved one more time. Every note plays one octave higher. Rests
(`$7E`) return before this and are still silent.

**Evidence:** The source and the built PRG0 ROM both have
`A0 01 8C F4 05 48 A8` at `$86DC`. `nes_gg.py` finds no other bank with
`$01` at `$86DD`.

**Revisions:** PRG0 only; PRG1 and Super Mario USA not checked.

**Caveats:**

- Tested by the user: works.
- `PlayNote` plays the music's squares and triangle, and also the square
  sound effects that go through `PlaySquare1Sweep` and
  `ProcessSoundEffectQueue1` (jump, coin, growing, shrinking and so on).
  Those are an octave higher too. Noise and DPCM sounds don't change.
- The highest notes get very short periods. A square note with a period
  under 8 is silent on the NES, so a few very high notes may drop out.

## Every song plays the next song in the list

**Code:** `AEOAZUNY`
**Raw form:** `8392:00:FF` (address:new-byte:compare-byte, bank 4)
**Recorded:** 2026-11-01 20:15:37

Tags: `demo-ready` `play-start`

**Mechanism:** When a song is queued in `MusicQueue1`,
`ProcessMusicQueue_MusicQueue1` (`src/prg-4-5.asm`) turns its bit
into an index into the song pointer tables:

```
	LDY #$FF

ProcessMusicQueue_FirstPointerLoop:
	INY
	LSR A
	BCC ProcessMusicQueue_FirstPointerLoop

ProcessMusicQueue_ReadFirstPointer:
	LDA MusicPointersFirstPart, Y
```

The code changes `LDY #$FF` to `LDY #$00`, so every index is one
higher. `MusicPointersFirstPart`, `MusicPointersEndPart` and
`MusicPointersLoopPart` (`src/music/music-pointers.asm`) all use the
same index, so each song plays the next song in the table, start to end:

| Queued song | Plays |
|---|---|
| Overworld | Character select |
| Character select | Underground (inside) |
| Inside | Boss |
| Boss | Star |
| Star | Subspace |
| Subspace | Wart |
| Wart | Title |
| Title | Ending |

**Evidence:** The source and the built PRG0 ROM both have
`8D 09 06 A0 00 8C 06 06 A0 FF C8 4A 90 FC B9 03 91` at `$8389`, and
`MusicPointersFirstPart` (`$9103`) has 9 entries, so the title's new
index (8) is still in the table. `nes_gg.py` reports that banks 7, 11
and 13 also have `$FF` at `$0392`. All three are unused `$FF`
padding there, so the code doesn't change anything else.

**Revisions:** PRG0 only; PRG1 and Super Mario USA not checked.

**Caveats:**

- Tested by the user: works.
- The title song doesn't loop (`MusicPointersLoopPart` is 0), so the
  Wart fight plays the title song once and then goes silent.
- `MusicPlaying1` still holds the queued song's bit, so the game still
  thinks the original song is playing. The inside and star songs still
  play their drums as DPCM samples, so the boss and subspace songs get
  sampled drums while the other songs get noise drums. The star timer and
  `RespawnPlayer` still find the star song.
- The death jingle, game over and other `MusicQueue2` songs don't
  change.

## Long square and electric piano notes play with the strings instrument

**Codes:** `EASAGTEG` + `GGKETVZZ` (use both)
**Raw form:** `8654:80:C0` + `864E:44:2A`
(address:new-byte:compare-byte, bank 4)
**Recorded:** 2026-11-01 20:44:26

**Mechanism:** This is a companion to "Short square notes play
with the electric piano instrument", for the patches that code doesn't
touch.
`LoadSquareInstrumentDVE` (`src/prg-4-5.asm`) compares the square
note's patch against `$90`, `$E0`, `$A0`, `$B0`, `$C0`, `$D0` and
`$F0` in turn, and patch `$80` (the "long square", 50% duty) is what's
left when none match.

- `EASAGTEG` changes `CPX #$C0` to `CPX #$80`, so patch `$80` now
  goes to `LoadSquareInstrumentDVE_C0` (`$8693`), the "strings" (25%
  duty with a slow attack). Patch `$C0` no longer matches anything and
  falls through to the long square.
- `GGKETVZZ` changes the `BEQ` offset after `CPX #$A0` from `$2A` to
  `$44`, so patch `$A0` (the electric piano) also goes to the strings.

The overworld song's intro is patch `$A0` and the melody right after it
is patch `$80` (`src/music/segments/overworld/`), so the change is
heard from the first note of 1-1. The underground song and the ending
also use patch `$80`, and the character select song uses `$A0` and
`$C0`.

**Evidence:** The source and the built PRG0 ROM both have
`E0 A0 F0 2A E0 B0 F0 33 E0 C0 F0 3C` at `$864B`. The `BEQ` at `$864D`
goes to `$864F + $2A = $8679` (`LoadSquareInstrumentDVE_A0`), and
`$864F + $44 = $8693` is `LoadSquareInstrumentDVE_C0`. `nes_gg.py`
finds no other bank with either compare byte at these addresses.

**Revisions:** PRG0 only; PRG1 and Super Mario USA not checked.

**Caveats:**

- Tested by the user: works. The change is subtle.
- The strings swell in slowly, so short notes may sound quieter.
- Parts that used patch `$C0` (part of the overworld song and of the
  character select song) now play with the long square instead.
- Patches `$90` and `$E0` don't change, so later parts of the overworld
  song sound as before. This code can be used together with the
  electric piano code, since they change different bytes.
- Sound effects don't use this routine and don't change.

## All music plays with the strings instrument

**Codes:** `UOUAIGLG` + `UPXEGILG` (use both)
**Raw form:** `84B5:93:43` + `852C:93:43`
(address:new-byte:compare-byte, bank 4)
**Recorded:** 2026-11-01 20:49:34

Tags: `demo-ready` `play-start`

**Optional, the pluck instrument instead:** `SXUAIGLK` + `SZXEGILK`
(`84B5:AD:43` + `852C:AD:43`)

**Optional, the electric piano instrument instead:** `PNUAIGLK` + `PYXEGILK`
(`84B5:79:43` + `852C:79:43`)

**Mechanism:** Every frame, the music engine calls
`LoadSquareInstrumentDVE` (`$8643`, `src/prg-4-5.asm`) once for each
square channel, at `ProcessMusicQueue_LoadSquare2Instrument` and
`ProcessMusicQueue_Square1AfterDecrementInstrumentOffset`. That routine
picks the instrument's duty/volume table by the note's patch, and each
patch's handler (`LoadSquareInstrumentDVE_80` and so on) is in the same
page. The codes change the low byte of both `JSR` operands, so both calls
skip the patch check and go straight to one handler. Every square note in
every song then uses that instrument, whatever its patch.

| Low byte | Handler | Instrument (`src/music/instruments.asm`) |
|---|---|---|
| `$93` | `LoadSquareInstrumentDVE_C0` (`$8693`) | Strings: 25% duty with a slow attack |
| `$AD` | `LoadSquareInstrumentDVE_D0` (`$86AD`) | Pluck: 50%, 25% and 12.5% duty with a fast decay to a quiet tone |
| `$79` | `LoadSquareInstrumentDVE_A0` (`$8679`) | Electric piano: 12.5% duty with a pronounced decay |

**Evidence:** The source and the built PRG0 ROM both have `20 43 86` at
`$84B4` and `$852B`, and these are the only two `JSR`s to
`LoadSquareInstrumentDVE` in bank 4. All the handlers take the note
length in A and the table offset in Y, so any of them can be called
here. `nes_gg.py` finds no other bank with `$43` at either address.

**Revisions:** PRG0 only; PRG1 and Super Mario USA not checked.

**Caveats:**

- Tested by the user: works. The pluck version also works. The electric
  piano version works, but doesn't sound great.
- The strings swell in slowly, so short, fast notes may sound quieter.
  With the pluck, long notes fade to a soft tone after a moment.
- Square 2 still plays slightly detuned while Square 1 is on patch
  `$E0` (the title song and part of the overworld song), since that
  check reads the patch.
- The triangle bass, the drums and the sound effects don't change.
