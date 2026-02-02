# xpra IME (Input Method Editor) Support - Implementation Plan

## 1. Problem

When using xpra remotely (e.g. `xpra start :100 --start-child=gnome-terminal` on Ubuntu,
`xpra attach ssh:<host>:100` from Windows), IME-based input (Japanese, Chinese, Korean, etc.)
does not work at all. Neither client-side nor server-side input methods are functional.

### Test Environment

| Component | Version |
|-----------|---------|
| Server (Ubuntu 24.04) | xpra v6.4.3 (PPA) |
| Client (Windows 11) | xpra v6.3.2 (winget) |
| Development base | master branch |

## 2. Root Cause Analysis

### 2.1 Three Structural Issues

#### A. Client-side: GTK event handlers consume all key events

`xpra/client/gtk3/window/keyboard.py` L82-L90:

```python
def handle_key_press_event(self, _window, event) -> bool:
    key_event = self.parse_key_event(event, True)
    self._client.handle_key_action(self, key_event)
    return True  # prevents GTK IMContext from receiving the event
```

In GTK, `return True` means "event consumed" - no further propagation to IMContext.

#### B. Server-side: XTest bypasses the IM pipeline

Event flow:
```
Client → Network → Server → XTest fake_key() → X11 app
```

`XTestFakeKeyEvent` injects key events directly into X11, completely bypassing
the IM layer (XIM/fcitx/ibus). Even if fcitx is running on the server, it never
sees these events.

#### C. No IMContext integration

Xpra windows do not use `GtkIMContext`. There is no protocol for preedit
(composition display), commit (finalized string), or surrounding text.

### 2.2 Existing IBus Code

The following files contain IBus-related code, but only for daemon management
and layout querying - not for IME event processing:
- `xpra/keyboard/ibus.py` - IBus engine/layout queries
- `xpra/x11/subsystem/keyboard.py` - IBus daemon start/stop, IM env vars

## 3. Branch Strategy

### PR Target: `master`

| Reason | Detail |
|--------|--------|
| New features target master | v6.3.x / v6.4.x are bugfix-only branches |
| `packet_type.py` | Only exists on master; required for new packet types |
| Packet system redesigned | master uses `keyboard-event` + `BACKWARDS_COMPATIBLE` branching |
| GTK3 `keyboard.py` | Identical between master and v6.4.x - easy to backport |

### master vs v6.4.x Diff Summary (IME-relevant files)

| File | Difference |
|------|-----------|
| `client/gtk3/window/keyboard.py` | **Identical** |
| `server/subsystem/keyboard.py` | Packet processing refactored (key-action → keyboard-event) |
| `client/gui/keyboard_helper.py` | BACKWARDS_COMPATIBLE branching, send_config() added |
| `net/packet_type.py` | **New in master only** |
| `x11/bindings/keyboard.pyx` | `_parse_keysym` moved to module-level function |
| `client/win32/window.py` | **New in master only** (native Win32 client #921, WIP) |

### Client Architecture Note

- Windows clients on v6.3.x / v6.4.x use the **GTK3-based** client
- The native Win32 client (`client/win32/window.py`) is a master-only WIP feature
- This implementation targets the **GTK3 client (shared across all platforms)**
- Native Win32 IMM32/TSF support is deferred to a future PR

## 4. Implementation Design

### 4.1 Architecture

```
[Client Side]                           [Server Side]

Key Input
  |
  v
GtkIMContext.filter_keypress()
  |
  +-- True (IME handled it)
  |     |
  |     v
  |   "commit" signal
  |     |
  |     v
  |   IME_COMMIT packet ----> Network ----> Unicode character injection
  |                                         (XChangeKeyboardMapping
  |                                          + XTest fake_key)
  +-- False (regular key)
        |
        v
      keyboard-event packet ----> Normal key processing (unchanged)
```

### 4.2 Capability Negotiation

Both client and server advertise IME support via the existing capability exchange:

- **Server** `get_caps()`: adds `"ime-commit": True`
- **Client** `get_caps()`: adds `"ime-commit": True`
- Client checks server caps before sending `IME_COMMIT` packets
  (prevents sending unknown packets to older servers)
- Server checks client caps before sending IME-related packets
  (for future preedit extensions)

This follows the existing pattern in `server/subsystem/keyboard.py` `get_caps()`
(L108-115) and `parse_hello()` (L117-119).

### 4.3 Files to Modify

#### Protocol Layer
- `xpra/net/packet_type.py` - Add `IME_COMMIT` packet type constant

#### Client Side (GTK3)
- `xpra/client/gtk3/window/keyboard.py`
  - Create and initialize `Gtk.IMMulticontext` instance
  - Call `im_context.filter_keypress(event)` before processing key events
  - Handle `commit` signal to capture finalized strings
- `xpra/client/gui/keyboard_helper.py`
  - Add method to send IME commit strings to server

#### Server Side
- `xpra/server/subsystem/keyboard.py`
  - Add `_process_ime_commit` handler
  - Register packet handler in `init_packet_handlers` with `main_thread=True`
  - Implement Unicode string injection logic

### 4.4 Server-Side Unicode Character Injection

XTest only accepts **keycodes** (`XTestFakeKeyEvent`).
Japanese characters are not in the default keymap, so the following steps are needed:

#### Injection Algorithm

For a commit string (e.g. "こんにちは", 5 characters):

1. Compute Unicode keysyms for all characters (`0x01000000 + codepoint`)
2. For each keysym, check if a keycode already exists via `KeysymToKeycodes()`
3. Collect unmapped keysyms that need temporary keycode assignment
4. Find unused keycodes by scanning with `get_keysym_mappings()` (keyboard.pyx:687-696)
   - Keycodes where all keysym slots are `NoSymbol` are considered unused
   - Pool up to 8 unused keycodes for batch assignment
5. Suppress keymap change notifications by setting `keymap_changing_timer`
   (reuses existing pattern from `x11/subsystem/keyboard.py` L325-338)
6. Batch-assign all unmapped keysyms to unused keycodes via `XChangeKeyboardMapping`
   (**single call**, not per-character)
7. Call `XFlush` to ensure the mapping change reaches the X server
8. Loop through all characters: `XTestFakeKeyEvent` press/release for each
9. Restore original keymap via `XChangeKeyboardMapping` (**single call**)
10. Re-enable keymap change notifications via timer (existing pattern)

This results in at most **2 `XChangeKeyboardMapping` calls** regardless of string length.
If all characters already have keycodes (e.g. ASCII), no keymap changes are needed.

#### Unused Keycode Discovery

Dynamic scan approach using public APIs:
- `get_keysym_mappings()` returns `{keysym: [keycodes]}` for all mapped keysyms
- `get_minmax_keycodes()` (keyboard.pyx:532-535) returns the valid keycode range (typically 8-255)
- Keycodes not appearing in any keysym mapping are unused
- Pool size of 8 covers typical IME commits (a few to ~10 characters)
- If more than 8 unmapped characters exist, process in multiple rounds

Note: `_get_raw_keycode_mappings()` is a `cdef` method (not callable from Python).
Use the public `get_keycode_mappings()` (keyboard.pyx:698-709) or
`get_keysym_mappings()` (keyboard.pyx:687-696) instead.

#### Keymap Change Race Condition Handling

`XChangeKeyboardMapping` triggers asynchronous `MappingNotify` events, which
would cause xpra's `keymap_changed()` handler (server/subsystem/keyboard.py:75-78)
to fire and propagate unnecessary keymap updates to clients.

Mitigation (reusing existing pattern from `x11/subsystem/keyboard.py:325-338`):
1. Set `keymap_changing_timer` to a non-zero value before injection
   - While non-zero, `_keys_changed()` (L378-383) is suppressed
2. Perform the injection within an `xsync` block for X11 error handling
3. Call `XFlush` (keyboard.pyx:354) after keymap changes to flush the X11 request buffer
   - Note: `XSync` is not directly exposed in xpra's Cython bindings;
     `XFlush` is sufficient as we only need to ensure the server processes
     the mapping change before we inject key events
4. After injection, restore the keymap and schedule `keymap_changing_timer`
   to re-enable change notifications (via `GLib.timeout_add`, same as `set_keymap()`)

#### Thin Wrapper vs Reusing xmodmap_setkeycodes()

`xmodmap_setkeycodes()` (keyboard.pyx:589-646) wraps `XChangeKeyboardMapping`
but has a complex interface designed for full keymap configuration (handles
`new_keysyms` redistribution, missing keysym reporting, etc.).

For IME injection, a simpler dedicated wrapper is preferred:
- Takes a `{keycode: keysym}` mapping
- Calls `XChangeKeyboardMapping` directly
- No modifier or redistribution logic needed

#### Thread Safety

The IME commit handler must be registered with `main_thread=True` in
`init_packet_handlers()`, matching all other keyboard packet handlers.
This ensures `XChangeKeyboardMapping` and `XTestFakeKeyEvent` execute on
the same GLib main thread that owns the X11 connection.

### 4.5 Out of Scope (not included in this PR)

- **Preedit display**: Rendering IME composition candidates on the remote screen.
  The client-side OS IME candidate window will display locally as-is.
- **Surrounding text**: IME reading text around the cursor
- **Native Win32 client**: IMM32/TSF API integration
- **Server-side IME**: Input through server-side fcitx/ibus

## 5. Testing Strategy

### Phase 0: Verify Unicode Character Injection via XTest (Ubuntu only)

Verify that XTest can inject Japanese characters independently of IME.
This is the highest technical risk and should be validated first.

```
Steps:
1. Start Xvfb :99
2. Start xterm -display :99
3. Run a test Python script that:
   - Connects to X11 display
   - Finds an unused keycode
   - Maps Unicode keysym (0x01000000 + ord('あ')) via XChangeKeyboardMapping
   - Calls XFlush
   - Presses/releases via XTestFakeKeyEvent
   - Verify 'あ' appears in xterm
```

If this fails, the X11 server does not support Unicode keysym injection.
This is unlikely on modern X servers but would mean IME support requires
a fundamentally different approach (out of scope for this PR).

### Phase 1: Server-Side IME_COMMIT Handler (Ubuntu)

Changes:
- `packet_type.py`: add packet definition
- `server/subsystem/keyboard.py`: add handler with `main_thread=True`

Test:
```
1. xpra start :100 --start-child=xterm
2. Connect to server with a test script
3. Send IME_COMMIT packet: ("ime-commit", wid, "こんにちは")
4. Verify text appears in xterm
```

### Phase 2: GTK3 Client IMContext Integration

Changes:
- `client/gtk3/window/keyboard.py`: add IMContext

Test:
```
Step 2a: Add IMContext initialization and commit signal logging only
         → Verify commit strings appear in log when using IME on client
Step 2b: Send commit strings as IME_COMMIT packets
         → Verify Japanese text appears in server-side xterm
```

### Phase 3: End-to-End Test (Windows → Ubuntu)

```
1. Build xpra from master on both Ubuntu and Windows
2. Ubuntu: xpra start :100 --start-child=gnome-terminal
3. Windows: xpra attach ssh:<host>:100
4. Type Japanese via Windows IME → verify it appears in gnome-terminal
```

## 6. Key File Reference

### Client Side
| File | Role |
|------|------|
| `xpra/client/gtk3/window/keyboard.py` | GTK key event handlers (IMContext target) |
| `xpra/client/gtk3/window/base.py` | GTK window base class |
| `xpra/client/gui/keyboard_helper.py` | Key event send helper |
| `xpra/client/subsystem/keyboard.py` | Client keyboard initialization |
| `xpra/keyboard/common.py` | KeyEvent data structure |

### Server Side
| File | Role |
|------|------|
| `xpra/server/subsystem/keyboard.py` | Packet processing pipeline |
| `xpra/x11/subsystem/keyboard.py` | X11 keyboard subsystem, keymap change suppression pattern |
| `xpra/x11/server/keyboard_config.py` | X11 keycode translation |
| `xpra/x11/server/xtest_keyboard.py` | XTest device wrapper |
| `xpra/x11/bindings/keyboard.pyx` | X11 keyboard bindings (Cython) |
| `xpra/x11/bindings/test.pyx` | XTest C bindings |

### Protocol
| File | Role |
|------|------|
| `xpra/net/packet_type.py` | Packet type constants |
| `xpra/net/common.py` | Packet class, BACKWARDS_COMPATIBLE flag |

### Tests
| File | Role |
|------|------|
| `tests/unittests/unit/x11/keyboard_test.py` | Unicode keysym test (reference pattern) |
| `tests/xpra/keyboard/test_get_keycode_mappings.py` | Key mapping test |
