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

### 2.1 keyboard-sync is unrelated

`--keyboard-sync` only controls key repeat behavior:
- `sync=True`: server holds key-down state and manages repeat
- `sync=False`: client manages repeat; server does immediate press/release

It has no involvement in the IME input pipeline.
Relevant code: `xpra/server/subsystem/keyboard.py` `_handle_key()`

### 2.2 Three Structural Issues

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

### 2.3 Existing IBus Code

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

### 4.2 Files to Modify

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
  - Register packet handler in `init_packet_handlers`
  - Implement Unicode string injection logic

### 4.3 Server-Side Unicode Character Injection

XTest only accepts **keycodes** (`XTestFakeKeyEvent`).
Japanese characters are not in the default keymap, so the following steps are needed:

1. Look up Unicode keysym (`0x01000000 + codepoint`) via `KeysymToKeycodes()`
2. If not found, find an unused keycode
3. Temporarily map the Unicode keysym via `XChangeKeyboardMapping`
4. Press/release the keycode via `XTestFakeKeyEvent`
5. Restore the keymap

Existing infrastructure:
- `xpra/x11/bindings/keyboard.pyx` `xmodmap_setkeycodes()` already wraps
  `XChangeKeyboardMapping`
- `xpra/x11/bindings/keyboard.pyx` `KeysymToKeycodes()` provides
  keysym → keycode reverse lookup

### 4.4 Out of Scope (not included in this PR)

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
   - Presses/releases via XTestFakeKeyEvent
   - Verify 'あ' appears in xterm
```

Success → proceed to Phase 1.
Failure → fall back to xdotool subprocess approach.

### Phase 1: Server-Side IME_COMMIT Handler (Ubuntu)

Changes:
- `packet_type.py`: add packet definition
- `server/subsystem/keyboard.py`: add handler

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
