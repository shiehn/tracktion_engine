# Tracktion Engine: the Signals & Sorcery fork

This is a fork of [Tracktion Engine](https://github.com/Tracktion/tracktion_engine), maintained for
[Signals & Sorcery](https://signalsandsorcery.com), a free Generative Audio Workstation for Mac and Windows. In
Signals & Sorcery you define a musical contract, generate MIDI and audio layers within it, and then shape, arrange
and master those layers into songs.

Tracktion Engine is the audio engine inside Signals & Sorcery's native engine process. The fork exists for two
reasons:

1. **Source availability.** Every Signals & Sorcery release is built from this exact code, so the source behind the
   shipped engine is public.
2. **A few small patches** for things Signals & Sorcery needs that upstream doesn't do (yet).

Looking for Tracktion Engine itself? Start with the
[upstream repository](https://github.com/Tracktion/tracktion_engine) and its
[README](https://github.com/Tracktion/tracktion_engine/blob/develop/README.md). Everything below is about this fork.

## How Signals & Sorcery uses the engine

Signals & Sorcery drives Tracktion Engine differently from a typical DAW, and the patches follow from that.

- **A headless engine behind an app.** The engine runs as its own native process, controlled by the desktop app over
  a local JSON-RPC socket. There is no Tracktion UI; every edit arrives as a message.
- **One long-lived Edit.** Opening a project swaps its content into an Edit that is already running, rather than
  starting a fresh engine for each project.
- **Loops first, songs second.** A composition is a set of scenes. Each track holds one looping MIDI or audio clip,
  and scenes play from two decks. Songs are arranged from those scenes afterwards and rendered offline into stems
  and masters.
- **Plugins in and out of process.** Third-party VST3 instruments and effects run in a sandboxed plugin-host process;
  a few, such as Surge XT, run in-process. Instrument patches change at runtime (preset shuffles, sound history), so
  plugin state is restored often while the engine is running.

## The patches

All three sit on top of upstream `develop` (v3.2.0 plus later upstream commits) and total 46 lines.

| Commit | Area | What and why |
|---|---|---|
| `d5dbd51f` | Edit | Adds `Edit::resetNextItemID()`. After a loaded project replaces a live Edit's state, the item ID counter could hand out IDs that already exist in the project, which made new tracks collide. Signals & Sorcery calls it after every load, so the next ID is re-scanned. |
| `6ce32314` | External plugins | Some VST3 plugins (Surge XT among them) defer `setStateInformation` until the message loop runs. Signals & Sorcery applies patches from its own polling loop, which is not a dispatch loop, so a deferred patch could silently never land. `restorePluginStateFromValueTree` now reads the state once and pumps the message loop briefly, so the new patch commits right away. |
| `8e31600c` | Sampler | Upstream only chokes a same-note voice on retrigger when the sound is not open-ended. Signals & Sorcery plays drums and one-shots as open-ended samples, so fast same-note rolls stacked voices. Same-note voices are now choked on retrigger even when open-ended; note-off still lets open-ended voices ring out. |

These are small and general enough to offer upstream.

## Branches

| Branch | What |
|---|---|
| `sas-patches` (default) | Upstream `develop` plus the patches above. This is what Signals & Sorcery builds against, pinned as a git submodule of its engine. |

Updating to a newer upstream means re-applying the patches onto it (a rebase), not a plain version bump.

## Licence

Tracktion Engine is dual-licensed: GPL-3.0-or-later, or a commercial licence from
[Tracktion](https://www.tracktion.com/develop/tracktion-engine). Signals & Sorcery uses it under the GPL, and the
patches here are released under the same terms. JUCE, bundled in `modules/juce`, is available under AGPLv3 or a
commercial licence. See [LICENSE.md](https://github.com/shiehn/tracktion_engine/blob/sas-patches/LICENSE.md).

Using Tracktion Engine in closed-source software still requires Tracktion's commercial licence; this fork doesn't
change that.

## Links

- Signals & Sorcery: [signalsandsorcery.com](https://signalsandsorcery.com)
- Upstream Tracktion Engine: [github.com/Tracktion/tracktion_engine](https://github.com/Tracktion/tracktion_engine)
- JUCE: [juce.com](https://juce.com)
