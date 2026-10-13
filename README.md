# launch-video

A Claude skill that turns a storyboard plus **real product screenshots** into a polished motion-design launch video: MP4, 1920×1080, 60 fps, real motion blur, and sound synced to the picture.

Claude doesn't screen-record anything or use a video editor. It writes one `index.html` that draws every frame as a pure function of time (`window.seek(t)`). Playwright captures the frames, ffmpeg blends and encodes them, and the audio is generated from the same beat timeline as the picture. A given frame renders identically on every run, so motion blur, sound sync and later fixes stay reliable.

```
storyboard + screenshots ──► index.html (seek(t)) ──► Playwright ──► ffmpeg tmix ──► clip.mp4
                                     ▲                                                  ▲
                                timeline.js (beats) ──────► audio generator ────────────┘
```

---

## What's in the package

`launch-video.skill` is a zip archive:

| File | Purpose |
|---|---|
| `launch-video/SKILL.md` | The instructions Claude follows: intake, studio setup, build, checks, delivery. |
| `launch-video/assets/brief-template.md` | Template of the brief (project parameters and storyboard). Copy it to `brief.md` to fill it in ahead of time. |
| `launch-video/references/look-template.md` | Visual and engine rules. Claude fills it in as `LOOK.md` for each project, and that file is binding for the whole job. |
| `launch-video/references/studio-setup.md` | One-time tooling check and the test clip that proves the pipeline works. |

To read or edit the source:

```bash
unzip launch-video.skill -d src/
# edit files under src/launch-video/, then repackage:
cd src && zip -r ../launch-video.skill launch-video && cd ..
```

---

## Installation

### Claude Code (recommended)

The pipeline runs Playwright and ffmpeg against folders on your machine, so it's built for Claude Code.

```bash
# Personal skill, available in every project
unzip launch-video.skill -d ~/.claude/skills/

# Or a project skill, shared with the repo
unzip launch-video.skill -d .claude/skills/
```

You should end up with `~/.claude/skills/launch-video/SKILL.md` (or `.claude/skills/launch-video/SKILL.md`). Restart Claude Code or open a new session, and the skill will show up in the skills list.

### Claude apps

You can also upload `launch-video.skill` as a custom skill in the Claude app settings. Keep in mind that the skill needs ffmpeg, Python, Playwright/Chromium and access to your brand and screenshot folders. Those are usually only available locally.

### Requirements

Claude checks these during studio setup and installs whatever is missing, but asks first if an install needs elevated permissions:

- `ffmpeg` with `libx264` and `aac`
- `python3`
- Python `playwright` plus Chromium (`playwright install chromium`)

---

## How to use it

### 1. Prepare your material

The skill shows only real material that you provide. Before you start, have these folders ready:

| Parameter | What it should contain |
|---|---|
| `BRAND_DIR` | Brand assets: the **typeface files** (required), brand colors, logo. |
| `PRODUCT_DIR` | The product itself: UI and the colors of states, alerts and components. |
| `KIT_DIR` | Real screenshots and crops. It must contain `shots/` and `crops.js` (crop positions), and optionally `site/` (the real pages, so new states can be screenshotted). Treated as **shared and read-only**. |
| `ASSETS_DIR` | Extra assets such as third-party logos, photos and icons. Can be `none`. |
| `DEMO_CMD` | A command that starts a demo or mock backend, if the product needs live data to look right. Can be `none`. |

A typical kit looks like this:

```
kit/
├── shots/          # full-page screenshots and crops (PNG)
├── crops.js        # positions of the crops inside the shots
└── site/           # the real pages, editable copies for new states
```

### 2. Start a new, empty working folder

Claude writes everything into the current folder and never into `KIT_DIR`. Use one folder per video:

```bash
mkdir my-launch-video && cd my-launch-video && claude
```

### 3. Ask for the video

Describe what you want. You don't need to mention the skill, ffmpeg or Playwright. The skill also triggers on Italian phrases like "video di lancio", "clip demo", "video prodotto". For example:

> Make a launch video for Acme Inbox. Storyboard: 1) the headline "Your inbox, finally quiet" rises in; 2) the camera zooms into the inbox and the cursor clicks "Snooze"; 3) the snoozed list appears; 4) logo and URL.

or, to call it explicitly:

> /launch-video

### 4. Answer the intake

Before building anything, Claude asks in one message for whatever is still missing:

- **Storyboard**: what happens, in order, in as much or as little detail as you like.
- **Project**: `PROJECT_NAME`, `PRODUCT_NAME`, `VIDEO_LANGUAGE`.
- **Directories**: `BRAND_DIR`, `PRODUCT_DIR`, `KIT_DIR`, `ASSETS_DIR`, `DEMO_CMD`.
- **Colors**: `CANVAS_COLOR`, `ACCENT_COLOR`, `CARD_BORDER`, `OTHER_COLORS`. A hex value and "from BRAND_DIR" are both valid answers.

Claude never fills these in from defaults or guesses. A guessed path breaks the build, and a guessed color puts the wrong brand on screen. Template colors are only offered as suggestions.

If you'd rather use a file, copy the template and fill it in first. Claude will then only ask about what's missing:

```bash
cp ~/.claude/skills/launch-video/assets/brief-template.md brief.md
```

No storyboard? Ask Claude to propose one. It drafts it only from what really exists in `KIT_DIR` and waits for your approval.

### 5. Confirm the recap

Claude checks that every folder exists and holds the right things, then shows a recap of every parameter (defaults included) and one line per scene. **Nothing renders until you give an explicit go.** A full render takes minutes, so it's cheaper to fix a color here. The confirmed brief is saved as `brief.md`.

### 6. Let it build

Then Claude does the following:

1. **Studio setup (once per machine).** It renders a 3-second test clip (a dot that springs into a pill reading "Studio ready." with a beep at 1.0 s) and proves three things with evidence: the render is **deterministic** (pixel-identical frame across two runs), **motion blur** works, and the **beep lands on time** within one frame in the final MP4. Result: `test.png`.
2. **`LOOK.md`.** It fills in the look and engine rules from your brief.
3. **Build.** It writes `index.html`, `timeline.js`, a render script and an audio generator, then renders in parallel segments with cached frames.
4. **Check.** It renders one frame per beat into `contact.png`, fixes the three worst problems, and scans the video for single-frame pops.
5. **Report.** It delivers `clip.mp4` (H.264, yuv420p, AAC) and tells you what it fixed, what it would still change, and how it read any ambiguous parts of the storyboard.

### 7. Iterate

Ask for changes in plain language from the same folder, for example *"make scene 3 shorter"* or *"slow down the zoom on the Snooze button"*. For fixes to an already confirmed video, Claude skips the intake and works from `brief.md`. Cached frames mean only the affected segments re-render.

For the next video in the same folder, Claude starts from the saved `brief.md`, shows the recap and asks only what changes.

---

## Files produced in the working folder

| File | What it is |
|---|---|
| `brief.md` | Confirmed parameters and storyboard. |
| `LOOK.md` | Binding look and engine rules for this project. |
| `index.html` | The scene. Every frame is drawn by `window.seek(t)`. |
| `timeline.js` | Every time in the video, written in beats. Picture and sound both read it. |
| render script / audio generator | The capture, blend and encode pipeline. |
| `test.png` | First, middle and last frames of the studio test clip. |
| `contact.png` | One frame per beat, used for the visual check. |
| `clip.mp4` | The final video (`OUTPUT_FILE`). |

---

## The default look

Unless your storyboard says otherwise, the video follows these rules (from `LOOK.md`):

- **Screen-recording style with a smart camera.** The whole real product screen sits on the canvas as one big card, and the camera zooms to wherever the cursor is working. The camera is one transform on one container, with eased moves and zoom interpolated in log space.
- **Real UI only.** Every product pixel comes from `KIT_DIR/shots`. No redrawn UI and no invented numbers, names or prices in product scenes. Illustrative scenes you explicitly ask for may use made-up content.
- **Light canvas** (`CANVAS_COLOR`) with a soft 5% accent tint behind whatever is in focus. No dark scenes.
- **Cards**: white, radius 20, `CARD_BORDER` border, soft shadow.
- **Text** rises out of a mask line, one word per beat. Nothing fades or blurs in.
- **Shape morphs instead of cuts**: a dot grows into a button, a button stretches into a card.
- **Banned**: 3D, glows, particles, gradients on UI, crossfades, any hold longer than 1.5 beats.

If the storyboard contradicts a rule, the storyboard wins **for that scene only**.

---

## Settings with defaults

You can change any of these in the brief. Otherwise they keep these values:

| Group | Setting | Default |
|---|---|---|
| Render | `OUTPUT_FILE` | `clip.mp4` |
| | `WIDTH` × `HEIGHT` @ `FPS` | 1920 × 1080 @ 60 |
| | `SUBFRAMES` / `SUBFRAMES_FAST` | 4 / 8 (used where motion exceeds `FAST_THRESHOLD_PX`) |
| | `FAST_THRESHOLD_PX` | 15 px per frame |
| Test clip | `TEST_DURATION`, `TEST_TEXT`, `TEST_BEEP_AT`, `TEST_FRAME` | 3 s, "Studio ready.", 1.0 s, 90 |
| Look | `ACCENT_TINT` | 5% |
| | `CARD_RADIUS` / `CARD_SHADOW` | 20 / `0 12px 32px rgba(21,32,27,.08)` |
| | `MAX_HOLD` | 1.5 beats |
| | `TEXT_ENTRY` | rises out of a mask line, one word per beat |

---

## Tips and troubleshooting

- **Missing logo or asset.** Claude stops and asks instead of redrawing it from memory. Add it to `ASSETS_DIR`.
- **A state that isn't in the screenshots** (e.g. a filled form, an alert). Claude edits a *copy* of the real page in `KIT_DIR/site` and screenshots it. If the product needs real data, provide `DEMO_CMD`.
- **First frames in the wrong font.** Fonts weren't awaited before capture. The build waits for `document.fonts.ready` and image decoding. Make sure `BRAND_DIR` contains the actual font files.
- **Audio out of sync.** Audio must be generated from `timeline.js`, not from rounded frame times. AAC adds an encoder delay, which is measured and compensated in the studio test.
- **Visible steps in fast motion blur.** Subframes are raised only on fast moments (`SUBFRAMES_FAST`), not everywhere.
- **Jerky motion when a value changes target twice.** It's animated as a sum of springs, one per change.
- **Renders are slow.** 60 fps × 4–8 subframes means thousands of captures. That's expected. Segments run in parallel and are cached, so later fixes are fast.
