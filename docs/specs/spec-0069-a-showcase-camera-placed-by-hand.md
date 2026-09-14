# spec-0069: A showcase camera placed by hand

- **Status**: Proposed
- **Ground**: written against the engine at `495fca44` (`origin/main`), read
  only, and against the branch `feature/a-shot-shows-the-best-side` at
  `30768527`, read only. The branch is where the agent's own showcase cameras
  are being built; at that revision it carries room cameras, standing views and
  the panorama's `--world`, and **no per-shot camera record yet** — so §4 lists
  the fields a hand-placed camera needs and names that branch as the schema's
  owner. The one live fact this spec rests on was taken from a recorded run,
  not re-run: `EULA=TRUE validation/rehearsal-flow.sh` on `495fca44` exits 0,
  which proves the overlay's trigger → storage → `say` stamp → `delvec
  harvest` chain works for a plain, un-opped player with no datapack reload.
  Nothing here was measured by building.
- **What it is for**: a released delve's storybook and front page carry a
  handful of renders, and the agent chooses their cameras. When no camera the
  agent finds is one a person is satisfied with, the person places the camera
  herself — standing in the running game where the shot should be taken — and
  **exactly that camera** is what the render uses. Her part is to stand and
  look; she types no coordinate and edits no file.
- **Research**: every rule below is marked **cited** (a vanilla fact, an ADR,
  a spec, or a document in the tree) or **authored** (this spec chooses). Three
  vanilla facts are stated from memory and marked as such; the implementing
  round pins each from the wiki page it reads, in the test that lands it.
- **Numbers**: `spec-0069` is the only number taken. No DW code, ADR or
  `dsl_version` is allocated; §7 counts what the design would need.
- **Non-goals**: the agent's own camera estimation (the branch's); any cutscene
  camera — spec-0019 stays the authority for those; a viewer-side "save this
  camera" control (§2 says why, and §9 puts it to the owner); a camera path or
  animation; any per-piece render; the implementation.

## 1. The defect, and the medium the judgement lives in

**Authored**, on the finding that opened this: the shipped showcase shots of a
released campaign were judged and rejected — the exterior too far and mostly
mountain, the interiors square-on to a compass direction and not the rooms at
their best.

The agent path answers this with estimation: a camera derived from the approved
concept image and refined by comparison. That is the right first answer and it
will sometimes be wrong, because the last judgement — *is this the picture* —
is one no gate makes. spec-0019 already states where that judgement belongs for
a cutscene: **in the running game**, where the person stands in the place and
looks. A showcase render is the same judgement about a still frame, and today
there is no path from the place she is standing to the scene Chunky renders:
the render plan is computed (`compiler::render_plan`), the panorama is fitted
(`delvec panorama`), and a camera she has found with her own eyes can reach
neither.

What exists and is one step short, at `495fca44`:

- The creator overlay captures a player's eye point on demand (`dw.mark`, as a
  block cell), raycasts along the view (`dw.aim`), and stamps a
  machine-readable line into the server log that `delvec harvest` turns into a
  versioned report. Live-verified, exit 0.
- `delvec snapshot --camera x,y,z,yaw,pitch[,fov]` renders a draft frame from
  any free camera in under a second (spec-0015).
- `delvec scene` turns every render-plan shot — a free `pos` + `yaw` + `pitch`
  + optional `fov` — into a Chunky scene, with the world path, the declared
  hour's sun, the horizon's water and the night-vision review policy all
  attached; `render-shots.sh` gates it on a real world save.

So the spec is small: **one trigger that captures a continuous pose, one
harvester arm that reads it, one record that carries it into the plan, and the
words on the page that make the loop walkable.**

## 2. Two ways a person places a camera, and which comes first

**Authored**, from what each surface can and cannot read.

| | in the running game | in `delvec viewer` |
|---|---|---|
| what she looks at | the assembled world: every piece seated, neighbours, terrain, the sky at the declared hour, the cast standing where it stands | one piece or one tiled zone, unlit, no neighbours, no sky |
| coordinates captured | **world** coordinates — the record's own | **piece-local**; a world camera needs the area's placement (the origin and rotation `compiler::plan` records per area), which she cannot see and which is ambiguous when one piece is placed twice |
| eye position | exact, to the millimetre: `execute anchored eyes` at the player's current eye, any pose (standing, sneaking, spectating) | exact, from the page's own camera state (`cameraEye()`), at the page's fixed eye height or its orbit |
| look direction | exact, continuous: the entity's `Rotation` (yaw, pitch) | exact, continuous: the page's yaw and pitch |
| field of view | **not readable**: a client option the server never receives; stated in the record (§4) | known: the page's own projection |
| moving freely | spectator: any position, through walls, no footing needed | the walk and orbit modes, no collision |
| where the render comes from | the same build's world save, the same coordinates | a per-piece GPU render, or a world camera only after the placement transform |
| what she has already done to be there | joined the step 9 server and walked the delve | opened a page the agent built for one piece |

**The game comes first, and it is the only route this spec builds.** The
picture she is rejecting is a picture of the assembled world; the game is the
one place she sees that world whole, and she is already standing in it at step
9. What the game cannot read — her field of view — is one number, stated once
(§4). What the viewer cannot read — where the piece stands in the world — is a
transform she has no way to check, and the moment it is wrong the camera is
silently in the wrong room. The viewer route is therefore **named and not
built**: a piece-level camera she wants is already a standing view on the
branch (`render piece --view stand=…`), and a viewer "save camera" for a world
render would owe the placement transform, the disambiguation of a twice-placed
piece, and a second capture format — three things for a case the game covers.
§9 puts the choice to the owner.

## 3. The loop, with her actions counted

**Authored.** From *the agent's shot is not good enough* to *the record is
committed*. She is in the game already (step 9, `tools/playtest-server.sh up`,
or the compose pair); the agent is at the terminal beside the same tree.

| # | who | does | she types |
|---|---|---|---|
| 1 | she | says which picture is wrong, in chat or in game | words |
| 2 | agent | names the shot in the record (a row with the concept image it answers, no camera yet) — or, when the row exists, says its roster number | — |
| 3 | she | `/trigger dw.free` to leave her body (spectator; through walls, any height); fires it again later to come back | one trigger, once per session |
| 4 | she | flies and looks until the frame is right | movement |
| 5 | she | `/trigger dw.cam set <n>` — `<n>` the roster number the overlay printed when she joined; `/trigger dw.cam` alone is shot 1 | one trigger |
| 6 | overlay | stamps `[DelveCamera] slot=<n> eye=<mm,mm,mm> yaw=<c°> pitch=<c°> in=<block at the eye>` to the server log — data, no verdict | — |
| 7 | agent | `delvec harvest <log> <layout.json>` → the camera report; writes the row's camera from it (§4), with `source: hand` | — |
| 8 | agent | `delvec snapshot … --shot showcase/<name>` — the draft frame, under a second — and posts it | — |
| 9 | she | looks: yes, or move and fire `dw.cam` again (back to 4) | yes / no |
| 10 | agent | `delvec build`; `render-shots.sh`; Chunky renders **that one scene** at a draft budget, then at the review budget on her yes | — |
| 11 | she | looks at the Chunky frame: yes, or back to 4 | yes / no |
| 12 | agent | commits the record and the frame; the storybook links the frame | — |

**Her count per camera: one trigger and two looks.** Once per session: the
`dw.free` toggle, and one question the agent asks in chat — *what does your FOV
slider say?* — because the server cannot read it (§4). No coordinate is typed
by anyone: the overlay reads it, the harvester parses it, the agent's tool
writes it.

Three things the loop relies on, each already true or cheap:

- **The server she is standing in already carries the overlay.**
  `tools/playtest-server.sh up` stages `creator-datapack/` beside the delve's
  datapack whenever the build emitted one; the compose `playtest` profile
  mounts it. Both are step 9's commands.
- **The overlay works for a plain player.** Every `dw.*` trigger is armed each
  tick for everyone; the rehearsal run proved it un-opped. `dw.free` needs no
  op either: a datapack function runs at the server's function permission
  level, which is enough for `gamemode` — **cited (vanilla `server.properties`
  `function-permission-level`, default 2), stated from memory; pin it.**
  `playtest-server.sh`'s `docker run` ops nobody, so without this toggle she
  could not leave the floor on the path the page prints first.
- **The draft is fast and the final is one scene.** The measured rate for one
  scene at the review budget is 299 s on ten threads (`visual-review.md`); a
  64-sample draft of one scene is about 40 s **by proportion, not measured**.
  The loop renders one scene, never the set.

## 4. What reaches the render

**Authored** for the fields and their precedence; **cited** for every check it
reuses.

### 4.1 The record, and who owns it

The branch `feature/a-shot-shows-the-best-side` has been told to state each
showcase camera as **one data record per shot**, read by the Chunky scene
emission. **That record is the only format; this spec defines no second one.**
What a hand-placed camera needs the record to hold, so the two meet at one
schema:

| field | meaning | who fills it |
|---|---|---|
| `name` | the shot, a filename fragment (`render::view`'s `check_name` rule) | agent |
| `answers` | the approved image, in the design record's own spelling (`concept/<stem>`), verified to exist under `design/` by the scan `DW0890` already performs | agent |
| `pos` | the eye, world coordinates, three floats | harvester → agent's tool |
| `yaw`, `pitch` | the look direction, degrees, in **the record's one convention** | harvester → agent's tool |
| `fov` | the field of view, degrees, with its axis stated (§4.3) | agent, from her answer |
| `source` | `estimated` or `hand` | the tool that wrote the camera |

Whatever else Chunky needs — frame size, sample target, world path, sun, water,
review policy — the scene emitter already derives from the plan and the build,
and the record does not restate it.

**The record goes through `render-plan.json`'s one door.** `Shots::push` is
the single constructor of a plan shot, and it does three things for every kind:
stands the camera up, writes the camera, and records the eye for the clear-eye
proof. A showcase camera is a shot of kind `showcase` pushed through it —
**never pulled in** (a hand-placed eye is where she stood, as a `pov` eye is
the player's), so a hand camera inside a block is a refusal, not a silent move.
Reading the record at build is also what lets the overlay print the roster
(§3 step 5) and lets `snapshot --shot` and `delvec scene` find the shot by
name with no new flag on either. If the branch reads its record elsewhere, the
roster is dropped and the harvester numbers cameras by capture order; the loop
still closes, one step less legible.

### 4.2 How the pose is validated

Two facts, both already computed by the tree, asked at build tier over the
final assembled world:

1. **The eye is not inside a block** — `DW0724`, `nav::verify_camera_eyes`,
   over the same `World::is_clear` every other camera is judged by. A hand
   camera's message names the record row and the one remedy — *stand somewhere
   else and fire `dw.cam` again* — never *move the geometry*, because the
   geometry is what she was photographing. Spectator mode makes this case
   ordinary rather than exotic: a body flying through a wall can fire the
   trigger from inside it, and the overlay's `in=` field says what block the
   eye was in so the agent can tell her before a build runs.
2. **The frame holds the loaded world** — the ray from the eye along the view
   direction meets the framed extent `scene::framed_extent(layout_aabb,
   horizon)`, the very box the scene's `chunkList` is cut from, and the eye's Y
   lies inside the world's build limits. A camera that fails this renders an
   empty sky at exit 0, the exact failure `render-shots.sh`'s world gate
   exists to refuse in another form. It is the second shape of the same rule
   (*a camera photographs the scene*) and takes no code of its own; §7.

Both run over estimated cameras too, so an estimate inside a block reds the
same way (§8.6 perturbs it).

### 4.3 The field of view

**Authored**, on two facts stated from memory and marked for pinning:

- A Minecraft client's FOV is an option (`options.txt` `fov`); the server is
  never sent it. The slider's *Normal* is 70°, the engine's own
  `PLAYER_FOV_DEG`. **Cited from memory; pin.**
- What she sees while flying is wider than her slider unless *FOV Effects* is
  off. **Cited from memory; pin.** The page tells her to set it off for the
  session, or the frame she approved in flight is not the frame she gets.

So the record's `fov` defaults to 70, and the agent asks once per session and
writes her number into every hand row of that session. One more measurement is
owed before the number means anything: **which axis Chunky's `fov` key is.**
`snapshot` documents its `fov` as vertical; `delvec scene` copies the plan's
`fov` into Chunky's key with no conversion, and nothing in the tree records
whether Chunky reads it as vertical or horizontal. The implementing round
settles it by rendering a scene with a subject of known angular size and writes
the answer, with the render's hash, into `docs/reference/tools.md` §4a; the
record's field is named `fov` only once that line exists. Until then a frame's
proportion is a guess, and the page says so.

### 4.4 Which camera wins, and how it is recorded

One row, one camera. A hand-placed camera **replaces** the estimate in the same
row and sets `source: hand`; the superseded estimate is in git, not in the
record. The tool that writes estimates **refuses to write over a `hand` row**
and names the row; deleting the row is her act, asked in chat. A row whose
camera was hand-placed against one concept image keeps that `answers` value —
a camera is an answer to a picture, and re-aiming it at another picture is a
new row.

## 5. Reach from the page

**Authored.** The rehearsal flow is live and green and **a creator cannot walk
it from the page** (`references/tools-by-symptom.md` names the triggers and
two CLI verbs and none of: how the overlay server comes up, where its port is,
that the log must be captured before harvest, where `layout.json` is). That is
the defect not to repeat, and the words that close it for this spec close it
for spec-0019 in the same edit, because the two share every one of those four
lines.

The page names this at **step 12** (the visual review — *a POV frame that is
the wrong picture*) and at **step 14** (the storybook — *the exterior or
starting-scene shot is not the one to ship*), in one line each, as a
human-in-the-loop tool is named: it exists, this is what it catches. The
procedure lives in `tools-by-symptom.md` under one symptom — *a render camera
nobody is satisfied with* — and carries, in this order, with no reference to
`docs/reference/`:

1. *The server she is in already has the overlay* — step 9's command; when it
   is not up, that command again. `owner-play.yaml` publishes
   `localhost:25565`; nothing else does.
2. *Tell her the two triggers*: `/trigger dw.free` to fly and to come back;
   `/trigger dw.cam set <n>` when the frame is right, `<n>` from the roster
   line the overlay printed at join. FOV Effects off; ask her slider's number.
3. *Read the log*: `docker logs dw-playtest > <file>` on the `playtest-server`
   path (the name `--name` set); `docker compose … --profile playtest logs
   --no-color > <file>` on the compose path. The log is read before `down`.
4. *Harvest*: `delvec harvest <file> <out>/creator-datapack/layout.json` —
   `<out>` is the `--out` given at step 9 — and the camera report it writes.
5. *Write the row, show the draft, render the one scene*: the three commands of
   §3 steps 7, 8 and 10, with the one-scene Chunky line and the draft budget.

The same five lines, with `dw.mark`/`dw.aim`/`dw.done` and `delvec calibrate`
in place of `dw.cam` and the row-writer, are the cutscene entry's procedure,
and the edit that lands one lands the other. `tools/check-skill-page.py` rule 4
already holds every `delvec` subcommand and flag the page names to the pinned
engine's clap surface; §8.8 adds the trigger names and the log and manifest
paths to what a page edit is held to.

## 6. What is reused, what is new, and what spec-0019 gives up

**Authored.**

Reused, by name, with nothing copied:

- **The trigger → `say` → harvester chain** (spec-0006 §3, spec-0019 §3–4):
  the same `dw.*` trigger family, armed the way `creator::tick` arms it (never
  reset by the tick — the recorded defect); the same `say` channel, because
  `tellraw` never reaches the log; the same `split_log_line` and the same
  single harvest pass, which gains a `[DelveCamera]` arm beside `[DelveShot]`
  and `[DelveNote]` and writes its report only when the log carries a stamp.
- **`dw.mark`'s capture mechanism**, generalised: `execute anchored eyes
  positioned ^ ^ ^` gives the eye point in any pose, and `execute store result
  … 1000` gives it in milli-blocks as an integer — the one NBT type a macro
  substitutes without a suffix, which is the whole reason `dw.mark` quantised
  to a cell. A continuous camera keeps the same rule at a finer grain: the
  stamp carries **fixed-point integers** (milli-blocks, centi-degrees from
  `Rotation` scaled by 100) and the harvester divides. No float crosses a
  macro.
- **`[DelveShotRoster]`'s join-time `say`**, for the camera roster.
- **`DW0724` and `Shots::push`**, for the eye proof and the one door (§4.1).
- **The yaw conversion `snapshot --shot` already owns** between Minecraft
  degrees and the plan's, run the other way; pinned by round trip (§8.5).
- **`snapshot`, `scene`, `render-shots.sh`, `world-save.sh`**, unchanged.

New:

- `dw.cam` and its handler; the `[DelveCamera]` stamp; the harvester arm and
  its report; the row-writer (the branch's tool, given a `hand` source); the
  `showcase` kind in the plan; the second shape of the clear-eye rule; the
  page words.

What spec-0019 gives up, and why it costs nothing:

- **`dw.free` is redefined.** spec-0019 §2 names `dw.free set 1` as *detach
  from the dolly for the next replay* — a flag on a replay that does not exist
  (§2 is unimplemented at `495fca44`, and the page says so). Here it is the
  general *leave the body, come back to it* toggle, which is what a creator
  watching a replay from outside also needs. One name, one meaning, no second
  vocabulary; when §2 lands it consumes this toggle rather than a flag.
- **Nothing else moves.** `dw.mark` stays a block cell — its consumer is
  `anchor + integer offset`, and spec-0019 §5's refusal of free coordinates in
  the DSL stands: a showcase camera is a render record, not a cutscene, and
  never enters a stage document's `cutscene`. `delvec calibrate` is untouched;
  a camera report never passes through it.

## 7. Numbers this spec does not take, and why

**Authored.**

- **DW codes.** The eye proof is `DW0724` with a new kind and a second shape;
  the design-image check is the scan `DW0890` already runs. The one code the
  design may need is the record's **document arm** — a `name` that is not a
  filename fragment, an `answers` naming no image under `design/`, an
  estimate written over a `hand` row — and that arm belongs to the record,
  which the branch owns; if the branch's record already carries a code for its
  refusals, the count is zero. **Count: at most one, and not this spec's to
  take.**
- **`dsl_version`.** Whether the record is a stage document under a
  `dsl_version` is the branch's question; this spec adds one value to one field
  (`source: hand`) and takes no number.
- **ADR.** ADR-0012 (the human gives ideas and plays the result) and ADR-0003
  (the overlay is tooling-side and never ships) are invoked, not moved.
- **Demo level.** `docs/demo-levels.md` queues mechanics a player meets; this is
  a creator tool a player never sees, confirmed on the rehearsal flow's own
  fixture (`crates/dsl/fixtures/valid/cutscene-shots`), never on a campaign.

## 8. Acceptance criteria

Machine-checkable; each names its instrument and was checked against the tree
at `495fca44` and the branch at `30768527` before being written. Every verdict
below is a debt: nothing of this exists on either ref.

1. **The triggers.** `dw.cam` and `dw.free` are emitted into
   `creator-datapack/` for every campaign that emits an overlay, are armed by
   `creator/tick` under the same never-reset rule
   (`rehearsal::the_tick_never_resets_a_trigger_it_arms` extended to them),
   ride the ADR-0006 determinism gate, and are absent from the shipped image
   (the existing exclusion check). *Debt.*
2. **The stamp is the pose.** PackTest: a body teleported to a known
   position and rotation, standing, sneaking and in spectator inside a solid
   block, fires `dw.cam set 1`; each `[DelveCamera]` line carries the eye
   within 0.001 block and the rotation within 0.01°, and the in-block case
   carries `in=<that block>` and refuses nothing. *Debt.*
3. **The toggle restores.** PackTest: `dw.free` from adventure enters
   spectator; `dw.free` again returns adventure at the prior position. *Debt.*
4. **One harvest pass.** A fixture log holding `[DelveNote]`, `[DelveShot]` and
   `[DelveCamera]` lines yields all three reports from one `delvec harvest`
   run; a log with no camera stamp writes no camera report; two stamps for one
   slot keep the last and count 2; unit-tested shape. *Debt.*
5. **One conversion, round-tripped.** The row-writer turns fixed-point
   Minecraft degrees into the record's convention through the function
   `snapshot --shot` uses, inverted; a property test takes a plan camera to
   Minecraft yaw/pitch and back within 0.001°; `delvec snapshot --shot
   showcase/<name>` of a written row frames the same direction as `--camera`
   with the harvested Minecraft values. *Debt.*
6. **The proofs bind the record.** Fixtures: a hand row inside a block is
   `DW0724` naming the row and *fire `dw.cam` again*, and the plan holds no
   `requested_pos` for it (never pulled in); a row whose view ray misses the
   framed extent is refused naming the extent; an **estimated** row inside a
   block reds identically (the perturbation toward the kind-scoped shape). *Debt.*
7. **Precedence.** Writing an estimate over a `hand` row is refused naming the
   row; after the row is deleted the same write succeeds; test. *Debt.*
8. **Reach from the page.** `tools-by-symptom.md` carries §5's five lines under
   one symptom; steps 12 and 14 name it in one line each;
   `tools/check-skill-page.py` gains a rule that every `dw.*` trigger the page
   names is one the overlay at the pinned `ref` emits, and that every log
   command and manifest path the page names resolves against the scripts and
   emitters at `ref` — red when a trigger is renamed or a path moves. *Debt —
   rule 4 covers `delvec` subcommands and flags today, not triggers or paths.*
9. **The live tier.** `validation/rehearsal-flow.sh` — extended, never a
   second flow — has the bot fire `dw.free`, move to a pose, fire `dw.cam set
   1`, and the harvested camera report equals the bot's pose within the
   tolerances of criterion 2; the existing `[DelveShot]` assertions still
   pass in the same run. *Debt.*
10. **The axis is measured.** `docs/reference/tools.md` §4a states whether the
    pinned Chunky core reads `fov` as vertical or horizontal, with the scene
    and the render's hash that settled it, before any page line calls the
    record's field `fov`. *Debt — no line in the tree says.*
11. **Byte identity.** Two builds of a campaign with a hand row produce
    byte-identical `render-plan.json`, scene JSON and shot index (ADR-0006).
    *Debt.*
12. **The record.** `docs/reference/compiler.md` carries the `showcase` kind,
    `DW0724`'s second shape and the overlay's two new triggers;
    `docs/reference/tools.md` carries the harvester's third report and the
    row-writer; spec-0019 §2's `dw.free` line is restated to §6's meaning —
    all in the pull request that lands the code. *Debt.*

## 9. Decisions for the owner

1. **Build the game route only, and name the viewer route out** (§2). What
   it commits her to: when she wants a picture of the assembled world, she
   joins the step 9 server; a viewer page never writes a world camera. The
   alternative — a "save this camera" control on the viewer — costs the
   placement transform, a rule for a twice-placed piece and a second capture
   format, for a case the game already covers.
2. **The field of view is hers to state, once per session** (§4.3). What it
   commits her to: answering one question in chat, and turning *FOV Effects*
   off while she frames. The alternative is a `dw.fov set <n>` trigger she
   types the number into, which is the same number typed by her instead of
   asked by the agent.
3. **A hand camera is final until she deletes it** (§4.4). What it commits
   her to: the agent may not re-estimate a shot she placed; when a later build
   moves the world under it, the row reds (`DW0724`) and she is asked, never
   overridden.
