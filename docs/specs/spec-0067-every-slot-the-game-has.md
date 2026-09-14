# spec-0067: Every slot the game has

- **Status**: Proposed
- **Ground**: written against engine `495fca44` (`origin/main`), read only, and
  against the pinned game data `crates/delvec/data/PROVENANCE.md` names: the
  `misode/mcmeta` `1.21.11-summary` at commit `c976eb3b`, whose
  `item_components/data.min.json` was fetched for this spec and read back at
  sha-256 `51b191e13f86813ca02f1498942e5bc235947edb71eb8105a78401670b3665c4` —
  the digest `PROVENANCE.md` pins — and the vendored
  `crates/dsl/data/entity-tags-1.21.11.json`. Every count below is computed
  from those two files by a script over the whole file, never a hand count;
  the wiki page cited is named where it is used. The shape of this document is
  spec-0062's.
- **What it is for**: a campaign dresses a body in anything the pinned game
  can put on it — a barded and saddled horse at the gate, a wolf in armour, a
  llama under a carpet, a ghast in a harness — and is refused, by name, when it
  puts a piece where the game will not show it.
- **Research**: the slot vocabulary is a fact of the pinned game, read from
  two of its own records: the `minecraft:equippable` item component's `slot`
  field (the wiki page *Data component format/equippable*, which lists the
  eight values) and the `allowed_entities` field of the same component in the
  pinned item data, which names entity types and entity-type tags the vendored
  tag file resolves. Every rule below is marked **cited** or **authored**.
- **Numbers**: no spec or ADR beyond this one. **One new DW code** (§5), to be
  allocated at implementation. **`dsl_version` moves**: `equipment` gains two
  fields and `drops[].slot` two values.
- **Non-goals**: a per-entity hitbox for the horse and the other mounts
  (`nav::entity_dims` has no row for them and falls back to the humanoid box;
  §7.1 names it as the adjacent gap it is); rideable mounts, a player on a
  saddle; a wave mob dropping its saddle (drops stay the `elite` / `boss`
  rule); trims, dyes and any other item component; the harness's or the
  saddle's runtime behaviour on a puppet.

## 1. The defect, in numbers

**Finding, from reading the tree and the pinned data.**

`MobEquipment` (`crates/dsl/src/stages.rs`) names six slots — `head`, `chest`,
`legs`, `feet`, `main_hand`, `off_hand` — and `MobEquipment::slots()` returns
a fixed array of six; `EquipSlot`, the enum a `drops[].slot` entry names, has
the same six; `emit::strip_drops_line` writes six `drop_chances` keys by hand.
Writing `body` or `saddle` into an actor's `equipment` is refused at `DW0100`
naming those six. The same `MobEquipment` is worn by a `WaveMob` and an
`Actor`; an `Npc` carries none.

The pinned game has eight. The `equippable` component's `slot` takes `head`,
`chest`, `legs`, `feet`, `body`, `mainhand`, `offhand`, `saddle` [cited — the
wiki page named above]. In the pinned item registry **84 items** carry the
component, over **seven** distinct slot values: `body` 44, `head` 16, `chest`
8, `feet` 7, `legs` 7, `saddle` 1, `offhand` 1 (the shield); no vanilla item
declares `mainhand`, because a hand takes anything. The two the DSL lacks are
the two a horse needs: `body` is where horse armour, wolf armour, a llama's
carpet, a nautilus's armour and a happy ghast's harness go; `saddle` is the
saddle.

The item data also says **who may wear what**: of the 84, 45 carry
`allowed_entities` — `#minecraft:can_wear_horse_armor` (6 items),
`#minecraft:can_equip_harness` (16), `#minecraft:can_wear_nautilus_armor`
(5), `#minecraft:can_equip_saddle` (1), `minecraft:wolf` (1) and
`[minecraft:llama, minecraft:trader_llama]` (16, the carpets); the 39 armour
pieces, heads, the pumpkin, the elytra and the shield carry none. The vendored
entity-tag file resolves every one of those tags:
`can_equip_saddle` is eleven types (camel, camel husk, donkey, horse, mule,
nautilus, pig, skeleton horse, strider, zombie horse, zombie nautilus),
`can_wear_horse_armor` is horse and zombie horse, `can_wear_nautilus_armor`
two, `can_equip_harness` one.

## 2. The row's general form, corrected

**Authored.** The row proposes deriving the slot set from the `equippable`
`slot` values the pinned item registry uses. That derivation gives **seven**
slots, not eight: it cannot see `mainhand`, which no item declares and every
body has. A slot set derived that way would drop the hand the DSL already
models, or keep it by a hand-written exception — the second authority the row
was trying to remove. The fix is to name the right authority and use the
registry as its cross-check:

1. **The vocabulary is the game's equipment-slot set**, eight values, held as
   data in `crates/dsl` with `VanillaRule` provenance naming the wiki page it
   was pinned from, and pinned by the test that lands it (spec-0062 §3's
   pattern for the hurting-block set). It is the type `EquipSlot` deserialises
   into and enumerates from; nothing else lists slots.
2. **The registry is the cross-check, not the source.** A test asserts every
   `equippable.slot` value the pinned item data uses is in the vocabulary,
   with the counts of §1 as the numbers it prints — so a slot the game gains
   and an item uses reds the test the day the data is re-pinned, which is the
   property the row wanted, and `mainhand` is in the set because the game says
   so rather than because a script happened to find it.
3. **The item data is also the fit authority** (§4), which the row did not
   ask for and which is the half that catches the mistake nobody sees: a horse
   armour in `chest`, a saddle on a villager. Both are structurally perfect
   NBT the server stores and never shows.

## 3. The surface

**Authored.**

```json
{ "id": "actor/destrier", "entity": "minecraft:horse", "anchor": "anchor/gate",
  "equipment": {
    "body":   "minecraft:iron_horse_armor",
    "saddle": "minecraft:saddle"
  } }
```

- `MobEquipment` gains `body` and `saddle`; each takes an `EquipItem` (bare id,
  or `{item, enchantments}`) exactly as the six do. The DSL keeps its own
  spellings for the hands (`main_hand`, `off_hand`) and the NBT keys stay
  where they are (`EquipSlot::nbt`).
- `MobEquipment::slots()` is derived from `EquipSlot::ALL`, never a literal
  array; `strip_drops_line` writes one `drop_chances` key per slot of the same
  enumeration. A ninth slot, should the game grow one, is one enum arm.
- `drops[].slot` accepts the eight; a horse elite that leaves its saddle is
  `{"slot": "saddle"}` under the same `DW0490` / `DW0491` rules.
- Emission is unchanged in form: the component-era `equipment:{…}` /
  `drop_chances:{…}` summon NBT, with the two new keys and chance `0.0f` on
  every undeclared slot. The generated gear PackTest, which already asserts
  the summoned body holds what the campaign dressed it in, asserts the new
  keys the same way; that assertion is the live proof that the pinned server
  reads `saddle` and `body` off `/summon` NBT, and the spec states it as the
  proof rather than assuming the keys.

## 4. Fit — what the registry says an item is for

**Cited** for the facts (the pinned item data); **authored** for the rule.

A declared piece is held to the item data at validation, where the `DW0143`
item check already runs:

1. **An item that declares an `equippable.slot` goes in that slot.** A
   `diamond_helmet` in `legs`, an `iron_horse_armor` in `chest`, a `saddle` in
   `body`: refused. The hands are exempt — `main_hand` and `off_hand` take any
   item, because the game renders a held helmet as a held helmet; the shield's
   own `offhand` declaration is the one registry value that names a hand, and
   it is satisfied trivially.
2. **An item that declares `allowed_entities` goes on a body the list admits.**
   The body's entity id is the one the puppet actually wears
   (`nav::actor_body_entity` — a skinned actor is a mannequin), resolved
   against the ids and the entity-type tags the item names, through the
   vendored tag file. A saddle on a zombie, a horse armour on a mule, a wolf
   armour on a fox: refused, naming the item, the body, and the entities the
   item admits.
3. **An item that declares neither is not judged.** A carved pumpkin on a
   guard's head, a stone block on a zombie's head, a diamond block in a hand —
   the game accepts them, several are idioms, and the registry states no
   contrary fact. The rule refuses only what the game's own data contradicts.

The table this reads is a vendored derivation,
`crates/delvec/data/item-equippable-1.21.11.json`: item id → `{slot,
allowed_entities?}`, extracted from `item_components/data.min.json` by a
script beside the existing extractors, which pins the source digest and the
counts of §1 (84 items, seven slot values, 45 with an allowed list) and
refuses a source whose digest or counts differ. `PROVENANCE.md` gains its row.

## 5. The refusal

**Authored.** One new code, validation tier (exit 1), `dsl::validate`, two
shapes of one rule — *a piece is declared where the game will show it*:

- **the wrong slot** — names the item, the slot written, and the slot the
  item declares; the remedy is the slot;
- **the wrong body** — names the item, the body's entity, and the entities
  the item admits (ids and tag members spelled out, so the creator does not
  open the tag file); the remedy is the entity or the piece.

Why validation tier: every fact is in the documents and the pinned data; a
build is not needed to learn that a saddle does not go on a villager. Why one
code: both shapes are the registry contradicting a declaration, and the two
remedies never conflict. Every build prints `equipment binding: B body(ies)
dressed, P piece(s) declared over S slot(s) in use, F piece(s) with a
registry-declared slot, A with an allowed-entity list, R refused` — zeroes
included.

## 6. What the gallery, the record and the skill owe

**Authored.**

- **The gallery element** (spec-0039). A horse actor in the annex —
  `minecraft:horse`, `body: minecraft:iron_horse_armor`, `saddle:
  minecraft:saddle` — spawned by the existing staging beat; bound by
  perturbation: changing the armour's id moves the `summon` line's `equipment`
  compound. Units: `MobEquipment.body`, `MobEquipment.saddle`, and
  `EquipSlot::body` / `EquipSlot::saddle` through a tiered horse's
  `drops[]` or through the probe. Vanilla item and entity ids are data, never
  units.
- **The probe.** One committed probe for the new code: the horse's `saddle`
  slot given `minecraft:iron_chestplate` (shape 1), refused by `validate`;
  the second shape is a unit test (the same saddle on `actor/sergeant`, a
  zombie), since one probe carries one code and the shape is the same code.
- **The record.** `docs/reference/compiler.md`: the `equipment` surface row
  (eight slots, the fit rule), the new code's row, `DW0490`'s row (the slot
  list it enumerates), and the PackTest section for the gear assertion;
  `crates/delvec/data/PROVENANCE.md` and `docs/reference/tools.md` gain the
  extractor's rows — in the pull request that lands the code.
- **The skill.** A creator reads it from `references/quest-capabilities.md`
  under *Items, containers and loot* (the equipment shape, the eight slots,
  and that a piece is refused where the game would not show it) and under
  *Bodies* (a horse is dressed through `body` and `saddle`; its footprint is
  the humanoid default until §7.1 is closed, so a mounted set piece is posted,
  not routed).
- **The demo level.** A row in `docs/demo-levels.md` when the code lands: a
  stable yard — a barded and saddled horse, a harnessed ghast overhead, a
  wolf in armour at the gate — each dressed piece visible from the arrival
  cell.

## 7. Scope, and what is named out of it

**Authored.**

### 7.1 The mount's body

`nav::entity_dims` lists twenty-odd entities and falls back to `0.6 × 1.95`
for the rest; `horse`, `donkey`, `mule`, `camel`, `pig`, `strider` and `llama`
are not in it. A horse actor is therefore routed and clearance-checked as a
humanoid. This spec dresses the horse and does not size it; the gap is
recorded as a ledger row against `entity_dims` with the binding computed
over the actors and wave mobs whose entity falls back, and the skill page
says a mount is posted, not walked, until that row closes.

### 7.2 The rest

- Riding, mounting, a player in the saddle — vanilla behaviour no puppet has.
- Item components beyond `equippable`: trims, dyes, custom model data.
- Drops from ordinary mobs — the no-grind rule stands.

## 8. Acceptance criteria

Machine-checkable; each names its instrument, and each was checked against the
tree at `495fca44` before being written. Where the tree cannot yet satisfy a
criterion the verdict is recorded as a debt.

1. **The vocabulary.** `EquipSlot::ALL` has eight arms; a test asserts the
   set equals the eight the wiki page lists, with `VanillaRule` provenance
   naming the page; `MobEquipment::slots()` and `strip_drops_line` derive
   their keys from it, asserted by a test that counts `drop_chances` keys in
   the emitted strip line against `EquipSlot::ALL.len()`. *Tree: debt — six
   arms, a fixed array, six hand-written keys.*
2. **The cross-check.** A test over `item-equippable-1.21.11.json` asserts
   every `slot` value it holds is in `EquipSlot::ALL`, prints the per-slot
   counts, and asserts them equal to §1's (`body` 44, `head` 16, `chest` 8,
   `feet` 7, `legs` 7, `saddle` 1, `offhand` 1; 84 items; 45 with an allowed
   list). *Tree: debt — no such file.*
3. **The extractor.** `tools/extract-item-equippable.py` pins the source
   digest `51b191e1…` and the counts, refuses a mismatch by exit status, and
   is reproducible byte-for-byte (two runs, one digest); `PROVENANCE.md`
   carries the row. *Tree: debt.*
4. **The surface.** `delvec schema --stage all` exports `body` and `saddle`
   on `MobEquipment` and the two variants on `EquipSlot`, under the
   `dsl_version` the implementing round is handed. *Tree: debt.*
5. **Emission.** A test dresses a horse actor in armour and saddle and asserts
   the summon NBT carries `equipment:{body:{…},saddle:{…}}` and
   `drop_chances` with `body:0.0f,saddle:0.0f`; the generated gear PackTest
   for that campaign asserts both keys on the live body; two builds are
   byte-identical (ADR-0006). *Tree: debt.*
6. **Fit, shape 1.** A test declares `minecraft:diamond_helmet` in `legs` and
   asserts the new code naming `head`; the same helmet in `main_hand` is
   green. *Tree: debt — the code is on no ref.*
7. **Fit, shape 2.** A test declares `minecraft:saddle` on a `minecraft:zombie`
   and asserts the new code naming the eleven admitted types; the same saddle
   on `minecraft:horse` is green; a skinned actor is judged as
   `minecraft:mannequin`. *Tree: debt.*
8. **Fit, silence.** A test declares `minecraft:carved_pumpkin` in `head` on a
   zombie and `minecraft:stone` in `head` and asserts neither is refused.
   *Tree: debt.*
9. **Drops.** A test declares `{"slot": "saddle"}` on a `boss` horse that
   wears one and asserts the emitted drop chance on `saddle`; the same drop on
   a horse without a saddle is `DW0490`. *Tree: debt.*
10. **The binding line.** Every build prints §5's line; the gallery primary
    reports `F ≥ 1` and `A ≥ 1`. *Tree: debt.*
11. **The gallery.** §6's horse builds green; perturbing the armour id moves
    the summon line; the probe is refused at `validate` with the new code;
    `tools/check-gallery-coverage.py` reports 0 units in neither state. *Tree:
    debt — no gallery body wears `body` or `saddle`.*
12. **The ledger row for §7.1** exists in `docs/playtest-findings.json` with a
    binding computed over the bodies whose entity falls back to the default
    box. *Tree: not yet due.*
13. **The record and the skill.** The rows and pages of §6, in the pull
    request that lands the code. *Tree: debt.*
14. A demo-level row is queued when the code lands. *Tree: not yet due.*

## 9. Decisions for the owner

- The slot vocabulary is **the game's eight**, held as pinned data and
  cross-checked against the item registry — the alternative is the ledger
  row's derivation from the registry alone, which yields seven and loses the
  main hand.
- A piece declared **where the game will not show it is refused** (wrong slot,
  wrong body; one new DW code) — the alternative is emitting it and letting a
  saddled villager ship with nothing visible.
- An item the registry says nothing about (a pumpkin, a block) is **not
  judged** — the alternative is refusing every non-armour item outside the
  hands, which would forbid an idiom the game supports.
- The mounts' **hitbox is not sized here** — a horse stays a humanoid to the
  router until a separate row closes it; the alternative is widening this
  spec into the dims table.
- **`dsl_version` moves**; no ADR.
