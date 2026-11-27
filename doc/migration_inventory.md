# Migration inventory: LiveLily-part on LiveLily's core, with JUCE

LiveLily-part shows the part of one instrument (or a few) of LiveLily's score, for a performer to sight-read, and plays the click track.
Until now it was an openFrameworks app with copies of LiveLily's parser, instruments and score (`ofApp.cpp`, `instrument.cpp`, `score.cpp`),
which drifted from LiveLily's (about 400 lines in `score.cpp` alone). After the migration it runs LiveLily's own code:
a `Session` (the parser and the instruments) and the score of `Views`, with only the instruments LiveLily sends it, in a JUCE window.
A bar is then parsed and drawn by the same code in both programs, and looks the same in both.

Each step is one commit (in LiveLily, in LiveLily-part, or in both), checked by the user before it is committed.

## Progress

- [ ] Step 1 (LiveLily): the score's area, not the workspace, in `Views`
- [ ] Step 2 (LiveLily): CMake usable from another project, and the JUCE canvas as a library
- [ ] Step 3 (LiveLily-part): CMake, with LiveLily fetched or from a local clone
- [ ] Step 4 (LiveLily-part): the app: a `Session`, the score of `Views`, and the OSC messages of LiveLily
- [ ] Step 5 (LiveLily-part): the playback: the beat, the bar that plays, the countdown
- [ ] Step 6 (LiveLily-part): the click track
- [ ] Step 7 (LiveLily-part): openFrameworks goes, and CI builds LiveLily-part

## How LiveLily-part gets LiveLily's code

CMake fetches LiveLily at a fixed commit (or tag), so a fresh clone builds with nothing else to do,
and `-DLIVELILY_DIR=~/Applications/LiveLily` builds with a local clone instead, as `-DLIVELILY_JUCE_DIR` does for JUCE in LiveLily:
a change to LiveLily is in LiveLily-part at its next build, with no commit or push between them.
JUCE comes from LiveLily's CMake (fetched by it, or from `LIVELILY_JUCE_DIR`), so both use the same JUCE.

What LiveLily-part uses of LiveLily: `livelily_core` (with `LIVELILY_PYO=OFF`, and FluidSynth, which the core links), the platform of JUCE
(OSC, audio for the click track), and the JUCE canvas of the desktop app. Not the editor's panes, nor the sequencer's playback,
as the beats come from LiveLily.

## The OSC messages of LiveLily, and what they become

LiveLily sends them to the host and port given to an instrument (`\instname.sendto ...`, port 9000 by default), which LiveLily-part listens to.

| Message | Arguments | Today in LiveLily-part | After the migration |
| --- | --- | --- | --- |
| `/initinst` | index, name | an instrument of its own | `\insts.add name` (or `\insts.init` for the first), with a map from LiveLily's index to the part's |
| `/bar` | name | starts a bar | starts the text of `\bar name {` |
| `/line` | the expanded line, e.g. `\violin c'4 d' e'` | parsed with its copy of the parser | a line of the bar's text |
| `/bar` | int 0 | ends the bar, parses it | ends the text (`}`), which the session parses, as the editor runs a bar |
| `/loop` | name, index, bars | a loop of its own | the session's loop (`\loop name { ... }`, by the names of its bars) |
| `/group` | names | groups the instruments | `\group` |
| `/clef` | name, clef | the clef of the last bar | the clef of the instrument's last bar, as `\clef` sets it |
| `/size` | staff lines' distance | the size of the score | the size of the score in `Views` |
| `/numbars` | number | bars shown | the number of bars of the score (horizontal view) |
| `/accoffset` | float | accidentals' offset | the same, on the instruments |
| `/scorechange` | bool | change the score on the last bar | the same, in `Views` |
| `/beatcolor`, `/beatviztype`, `/beatbright` | | the beat visualization | the same, in `Views` |
| `/fullscreen`, `/cursor` | bool | full screen, mouse cursor | the window |
| `/seq` | bool | the sequencer plays | the state of playback (step 5) |
| `/beatinfo`, `/beat` | steps per ms, ramp start / beat | the beat visualization | `Views::beat()`, and a click (step 6) |
| `/loopndx`, `/thisloopndx` | index | the loop and the bar that play | the session's loop index and bar (step 5) |
| `/update` | loop index | the score changes at the next bar | the same (step 5) |
| `/countdown`, `/nocountdown` | number | the countdown | the same (step 5) |
| `/exit` | | quits | quits |

The indexes of bars and loops that LiveLily sends (`/loop`, `/loopndx`, `/update`) are LiveLily's; the part's are its own, as it gets only
the bars sent to it, so they are mapped by name (step 4 checks whether they always match, e.g. the empty bar `\insts.init` makes).

## Steps, in order

1. **(LiveLily) The score's area in `Views`.** `Views` reads the workspace in about 20 places, mostly for the bottom of the area the score
   may fill (`belowBase`), and to lay out the panes again when the score moves. A small interface gives `Views` its area and what it asks
   of the panes; LiveLily answers it with the workspace, and LiveLily-part with its whole window. LiveLily doesn't change (the screenshots are the same).
2. **(LiveLily) CMake usable from another project.** Included by another project (not the top level), LiveLily's CMake builds only its libraries,
   not the desktop app, the headless runner and the tests. The JUCE canvas (`desktop/juce_canvas.*`) becomes a library both apps link.
   LiveLily and its CI build as before.
3. **(LiveLily-part) CMake.** A `CMakeLists.txt` that fetches LiveLily (or uses `LIVELILY_DIR`), and builds an empty JUCE window, on Linux, macOS and Windows.
4. **(LiveLily-part) The app.** A `Session` and the score of `Views` (in the orientation of the old part, with its number of bars), the front end
   the session needs (most of `FrontendHost` is about panes, and does nothing here), and the OSC messages above, up to `/exit`, except the playback.
   Checked by sending a part from LiveLily to localhost and comparing it with the same instrument in LiveLily's score.
5. **(LiveLily-part) The playback.** The beat visualization, the bar that plays, the change of the score at the next bar, and the countdown,
   from `/seq`, `/beatinfo`, `/beat`, `/loopndx`, `/thisloopndx`, `/update` and `/countdown`.
6. **(LiveLily-part) The click track.** The audio output (JUCE), and LiveLily's `ClickTrack`, triggered by `/beat` (the first beat of a bar an octave higher).
   To decide: how a part's click track is turned on (a message from LiveLily, e.g. with `\instname.clicktrack on`, or a setting of the part),
   and whether a click on the arrival of `/beat` is on time over a network, or the part should schedule its clicks from the tempo (`/beatinfo`).
7. **(LiveLily-part) openFrameworks goes.** The makefiles, `config.make`, `addons.make`, `livelily_part.qbs`, `ofApp.*`, and the copies
   of `instrument.*` and `score.*`; the README tells how to build it; CI builds it on Linux, macOS and Windows.
