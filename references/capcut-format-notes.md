# CapCut draft format — verified notes

Everything here was confirmed against a real, installed CapCut desktop app (9.3.0 at time of
writing on macOS) — not just by reading `pycapcut`'s source or docs. If you extend this pipeline
and discover something new, add it here (see the parent SKILL.md's "Keep this skill alive").

## Where drafts live, and how registration works

- Local project file is `draft_info.json` — **not** `draft_content.json`, despite that being the
  name some docs/libraries assume. The internal schema (`materials`/`tracks`/`canvas_config`) is
  what `pycapcut` targets either way.
- Drafts live at `~/Movies/CapCut/User Data/Projects/com.lveditor.draft/<DraftName>/` and are
  listed in that folder's `root_meta_info.json`, in an `all_draft_store` array. Each entry needs
  a `draft_json_file` path, a unique `draft_id`, and matching `draft_name`/`draft_fold_path`.
- `pycapcut`'s `DraftFolder(path).remove(name)` only deletes the draft's own folder — it does
  **not** clean up the corresponding `root_meta_info.json` entry. Do that yourself (filter
  `all_draft_store` by `draft_name` before appending the fresh entry) or stale entries pile up in
  CapCut's own project browser forever.

## Sandboxing

CapCut.app is sandboxed (`com.apple.security.app-sandbox`). A `VideoMaterial` (or any media
reference) pointed at an arbitrary absolute path outside CapCut's own managed folders shows
"File not accessible" or prompts a relink dialog when the project is opened — even though the
draft JSON itself is perfectly valid. Fix: copy every media asset into the draft's own folder
(e.g. `<draft>/local_materials/<subdir>/`) before referencing it, every single time, no
exceptions — this is what the builder's `stage()` helper is for.

## Positioning

`clip.transform.{x,y}` on any video or text segment is center-anchored, normalized to canvas
half-width/half-height. Positive x = right, positive y = **up** (screen Y is flipped from the
usual top-left-origin convention). Text is always center-anchored regardless of the `align`
field — align only changes multi-line justification *within* the text block, not where the block
itself is anchored. See the parent SKILL.md's "Positioning" section for the design pattern this
implies.

## Confirmed working

- **Transitions**: `segment.add_transition(TransitionType.X, duration=...)` — confirmed via the
  real Transition panel showing the correct name + duration when the segment is selected.
- **Custom keyframed animations**: `segment.add_keyframe(KeyframeProperty.scale_x / scale_y /
  position_x / ..., time_offset, value)` — confirmed with an unambiguous test (a static image
  keyframed between two scale values, checked by rendering the actual preview frame at two
  different timestamps and seeing a real visual difference).
  **Caveat**: the properties panel does **not** live-update as you scrub within a clip in this
  CapCut version — it can show a stale/end value regardless of playhead position. Don't use that
  panel to verify keyframes; always check the actual rendered preview frame instead.

## Confirmed NOT working

- **Preset text animations** — `add_animation()` with `TextIntro`/`TextOutro`/`TextLoopAnim` —
  show an "Animation loss" badge on the clip and silently fall back to no animation. Likely
  cause: these presets are downloadable cloud assets, and the specific resource ID `pycapcut`
  references may not resolve/download on a given install (general internet access working fine
  otherwise, so this reads as a resource-catalog/version mismatch, not a hard format
  limitation — but it's unconfirmed and not worth relying on either way). Use a custom
  `add_keyframe` on `alpha`/`position` for a manual fade or slide instead — that mechanism is
  proven to work.

## Export-blocking Pro paywall on transitions

`pycapcut`'s `add_transition(TransitionType.X, duration=...)` renders fine in the timeline and
preview, but the specific transition asset can be **Pro-gated on the account CapCut is signed
into** — and that gating shows up only at export time, not at build time or in the editor
preview. Symptom: clicking Export shows a "Get Pro and save X to unlock these features" dialog
listing every instance of the gated transition by timestamp, with only "Get free Pro" / "Get
discount" / "Back to edit" as options — no free-tier or watermarked export path. In one observed
case, `TransitionType.Flash` was gated, and checking the Transitions panel by hand (Basic and
Classic categories both) showed *every* transition thumbnail carrying the same purple "Pro" gem
badge — so this may not be a per-transition-type issue but a whole-catalog gate on some accounts.

**Practical rule**: don't assume a transition type is free because it's simple/generic-sounding.
Before relying on `add_transition()` in a build script meant to export without a paid
subscription, actually open the real Export dialog on a build that uses it and confirm no
paywall appears — checking the Transitions panel thumbnails for the Pro badge is a faster
sanity check than a full export attempt. If gated, the safe fallback is to drop the transition
and rely on `add_keyframe`-driven motion (scale/position zoom, fades via alpha) instead — those
are confirmed not to trigger this paywall, since they're plain segment properties rather than a
licensed effect asset.

## `VideoMaterial.duration` is the source of truth, not `ffprobe`

Building a Timerange from `ffprobe -show_entries format=duration` (rounded to microseconds) and
handing it to `pycapcut.VideoSegment` can fail with "截取的素材时间范围 ... 超出了素材时长" (source
timerange exceeds material duration) even though the numbers look like they should match — one
observed case was off by 46μs (ffprobe: 47.680000s container `format=duration`; pycapcut's own
probe: 47.634000s) on a plain h264+aac mp4. `format=duration` is the container-level duration
tag, which can run slightly past the actual last decodable frame. Always read `material.duration`
back off the `VideoMaterial` object itself (after constructing it) for a full-length segment's
`Timerange`, rather than trusting an independently-computed ffprobe value — the same "one source
of truth per boundary" rule the parent SKILL.md's "Timeline math" section already states for
scene-boundary math applies to whole-clip duration too.

## Local draft filename seen in practice

One real install (CapCut desktop, macOS) wrote `draft_content.json` as the per-draft JSON file,
not `draft_info.json` as noted below — check the actual folder contents rather than assuming
either name if you need to hand-edit a draft file directly; `pycapcut`'s own `Script_file` API
doesn't care either way since it targets the internal schema, not a specific filename.

## Timeline math

All start/duration values must be **integer microseconds**, derived from one running cursor per
scene boundary — never `float seconds → string → reparsed`. Two independently-rounded
float→string→microsecond conversions of "the same" boundary can differ by 1 microsecond, and
CapCut's segment-overlap check will (correctly, if confusingly) reject that as a genuine overlap
between adjacent segments.

## Verification quirks (these produce false readings if you don't know about them)

- **Stale window state**: CapCut restores its previous window/tab state on relaunch, which looks
  like a fresh open but can be showing stale in-memory content from before you regenerated the
  draft. After rebuilding: fully quit (`pkill -9 -f CapCut`), relaunch, and explicitly navigate
  from the project browser *into* the project — don't trust whatever's showing right after
  `open -a CapCut`.
- **Display sleep during long verification passes**: this can cause screenshots to render solid
  black, with the next click just "waking" the screen rather than registering as a real
  interaction. Run `caffeinate -d -t <seconds>` before an extended computer-use verification
  session.
- **Project tile order shifts** between sessions (CapCut sorts its browser by recency) — don't
  assume a screen coordinate that worked last time still points at the right project tile; read
  the label each time, zooming in if needed.
- **Preview panel sometimes doesn't refresh on the first click** after moving the playhead in
  this CapCut version. If the preview looks unchanged after clicking a new timeline position,
  click again at a slightly different x before concluding something is broken.
