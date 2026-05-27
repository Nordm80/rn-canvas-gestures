# rn-canvas-gestures

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that gives Claude a battle-tested recipe for building **interactive canvas UIs in React Native**: pinch + pan + zoom of a background image with **draggable, resizable, rotatable items** on top — running on **iOS, Android, and web** from the same codebase.

When loaded, Claude can:

- Generate the gesture composition correctly the first time
- Audit existing canvas code and flag the violations (broken Android pinch via `ScrollView`, `Gesture.Race` with `runOnJS`, normalized-space rotation math, missing web DOM listeners, etc.)
- Fix bugs in cross-platform canvas code without reproducing the 6+ iteration cycles of the original session

## Install

Clone this repo directly into your Claude Code user-level skills directory:

```bash
git clone https://github.com/Nordm80/rn-canvas-gestures \
  ~/.claude/skills/rn-canvas-gestures
```

That's it. Next time you start Claude Code, the skill auto-loads when you (or your code) match its triggers.

To update later:

```bash
cd ~/.claude/skills/rn-canvas-gestures && git pull
```

## What's inside

- `SKILL.md` — the full recipe, with frontmatter, architecture decisions, exact dependency setup (Reanimated 4 + worklets babel plugin), the gesture composition pattern, the `bgEnabled` shared-value gate, source-space hit-testing under bg transforms, web-only DOM listeners for wheel + mouse-drag, 2-finger scale+rotate+translate, 12 specific gotchas with symptoms and fixes, a verification checklist per platform, and a refactor guide for existing code that violates the pattern.

## When Claude loads it

The skill's frontmatter description triggers auto-load when work involves:

- React Native + `react-native-gesture-handler` + `react-native-reanimated` worklets
- `Gesture.Pinch` / `Gesture.Pan` / `Gesture.Manual` composition
- Symptoms like "Android pinch broken", "ScrollView pinch", "bg moves during item drag", or the `setGestureState in non-worklet function` warning
- Building image editors, floor plan editors, draggable marker UIs, photo annotation tools — anything where the canvas needs to work cross-platform

## Background

This skill was extracted from a Phase 4 spike in [Framanfor-AB/cctv-survey-tool](https://github.com/Framanfor-AB/cctv-survey-tool) where we unified two separate canvas implementations (photo editor + floor plan viewer) under one engine. Six iterations of failure modes on Android, web mouse-drag, finger-timing races, and rotation math on non-square canvases all baked in.

## License

MIT — see `LICENSE`.
