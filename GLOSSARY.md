# callcontrol

A Stream Deck plugin for macOS that mirrors and toggles the state of a Microsoft Teams call. It exists because Teams no longer exposes a call control API, so call state has to be read from the client itself.

## Language

**Call**:
A Teams meeting or call James is currently in. There is at most one at a time.
_Avoid_: meeting, session

**Call state**:
The facts about the current call as Teams' own controls show them: whether a call is in progress, whether the mic is muted, whether the camera is on, whether the hand is raised. Teams is the source of truth; the macOS microphone and camera are not.
_Avoid_: status, mute status

**Control**:
Something a deck key can do to the call: toggle mute, toggle camera, toggle the raised hand, send a reaction.
_Avoid_: command, action (reserved for the Stream Deck meaning)

**Raised hand**:
The Teams "raise hand" indicator. A toggle with readable state, unlike a reaction.

**Reaction**:
A transient emoji Teams shows to other participants (like, love, applause, laugh, surprised). Fire-and-forget; it has no readable state.

**Deck**:
The Stream Deck hardware and its desktop software. The only consumer of call state in v1.

**Action**:
A Stream Deck key type the plugin offers, in Elgato's sense of the word. Each action renders one piece of call state and triggers one control.

**Adapter**:
The part that knows how one particular call app exposes its call state and controls. v1 has one: the Teams adapter.

**Helper**:
The native macOS process that reads call state from, and performs controls on, the call app on the plugin's behalf.
