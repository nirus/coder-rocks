**TL;DR** — CMUX uses libghostty, which speaks the *kitty keyboard protocol*. That protocol encodes `Ctrl+C` as the escape sequence `\e[99;5u` instead of the single byte `0x03`. Zsh has no idea what `\e[99;5u` means, so it dumps it on screen as `^[[99;5u`. The fix is a three-part defence: pop the keyboard mode CMUX pushes on surface init, drain the residual init bytes from the input buffer, and register `bindkey` calls that teach zsh the CSI u vocabulary.

---

## The Symptom

You're in CMUX. You press `Ctrl+C`. Instead of aborting the current line, you get:

```
^[[99;5u^[[99;5u^[[99;5u%
❯ 9;5u9;5u9;5u
```

Every `Ctrl`+key combo produces a similar garbled string. In standalone Ghostty the same keys work perfectly.

---

## Background: How Terminals Send Keystrokes

### The Legacy Model (VT100 era → today)

Traditional terminals encode modified keys by mangling them into a single byte. `Ctrl+C` becomes `0x03` (ASCII ETX). The kernel's tty line discipline watches for that byte and fires `SIGINT`. It's simple, battle-tested, and ambiguous — there is no way to distinguish `Ctrl+I` from `Tab`, or `Ctrl+M` from `Enter`, or `Escape` from `Alt+[`.

### The Kitty Keyboard Protocol (CSI u)

The [kitty keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/) solves the ambiguity by encoding every key press as:

```
CSI <unicode-codepoint> ; <modifiers> u
```

Where:

| Component | Meaning |
|-----------|---------|
| `CSI`     | `\e[` — the Control Sequence Introducer |
| codepoint | Unicode code point of the key (e.g. `99` = `c`) |
| modifiers | Bitmask: `2`=Shift, `3`=Alt, `5`=Ctrl, `6`=Ctrl+Shift, `7`=Ctrl+Alt, `8`=Ctrl+Alt+Shift |
| `u`       | Literal `u` — the sequence terminator |

So `Ctrl+C` becomes:

```
\e[99;5u
     │  │
     │  └─ modifier 5 = Ctrl
     └──── 99 = Unicode code point for 'c'
```

Applications opt in by sending a **push** escape (`\e[>flags u`) and opt out with a **pop** (`\e[<u`). The protocol is *progressive enhancement* — the terminal only sends CSI u sequences after the application asks for them.

### What Ghostty (Standalone) Does

Ghostty implements the full kitty keyboard protocol. But it only sends CSI u sequences to applications that **explicitly push** the keyboard mode. When your shell hasn't pushed, Ghostty falls back to legacy encoding. `Ctrl+C` → `0x03` → kernel fires `SIGINT`. Everything just works.

### What CMUX Does Differently

CMUX is a terminal multiplexer built on **libghostty** — the same rendering and input engine as standalone Ghostty, extracted as a library. CMUX sits between the outer terminal (Ghostty, iTerm2, whatever) and your shell:

![CMUX flow: Ghostty sends keys to CMUX (libghostty) which sends CSI u sequences to zsh](cmux-flow.svg)

The issue: CMUX's terminal emulation layer (libghostty) writes modified key presses to the inner pty using CSI u encoding — even when the child application (zsh) never requested it. This is likely because libghostty defaults to advertising the kitty keyboard protocol to the *outer* terminal but doesn't gate the *inner* encoding on an explicit push from the child shell.

The result: zsh receives `\e[99;5u` on its stdin, doesn't have a matching keybinding, and falls through to ZLE's (Zsh Line Editor) default behavior — printing the raw bytes.

---

## Why the Obvious Fix Doesn't Work

A common workaround floating around is to send the **pop** sequence on every prompt:

```zsh
_reset_kitty_kb() { printf '\e[<u' 2>/dev/null; }
add-zsh-hook precmd _reset_kitty_kb
```

The `\e[<u` sequence tells the terminal to pop one level of keyboard mode. In theory, this should revert to legacy encoding.

In practice, it doesn't help because:

1. **CMUX may not honour the pop on the inner pty.** libghostty's keyboard mode is managed by the terminal surface, not the application stream. A pop sent *to* the inner pty may be interpreted as an *output* escape (drawn/ignored) rather than a mode change instruction.
2. **Even if the pop succeeds, CMUX may re-push on the next keypress.** The push/pop model assumes the *application* controls the mode. If the *terminal multiplexer* is the one enabling it, pops from the child are fighting an upstream force.
3. **Timing.** The pop runs on `precmd` — before the prompt. If the mode is re-enabled before the next keypress, you're back to square one.

---

## The Real Fix: A Three-Layer Defence

Keybindings alone solve most presses — but the **very first `Ctrl+C`** in a fresh CMUX surface still prints junk. Here's why, and the full fix.

### Why the First Press Leaks

When CMUX creates a terminal surface, libghostty sends a **push keyboard mode** escape (`\e[>1u`) into the pty. That sequence arrives as bytes on zsh's stdin. By the time the shell finishes initialising and ZLE starts reading input, those bytes are sitting in the input buffer:

```
input buffer (before first keypress):
  \e [ > 1 u            ← residual push from CMUX surface init

user presses Ctrl+C:
  \e [ > 1 u \e [ 9 9 ; 5 u
              ↑ actual Ctrl+C

ZLE tries to match from the start:
  \e [ > 1 u  → no binding → printed as junk
```

The first Ctrl+C flushes the residual bytes as garbage. The *second* press arrives on a clean buffer and matches `\e[99;5u` → `send-break` cleanly.

### The Fix: Pop + Drain + Bind

The complete solution has three layers executed during `.zshrc` init, before ZLE ever reads a keypress:

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: Pop                                                    │
│   printf '\e[<u'                                                │
│   Tell CMUX to drop the pushed keyboard mode.                   │
│   Works if CMUX honours the pop; harmless if it doesn't.        │
├─────────────────────────────────────────────────────────────────┤
│ Layer 2: Drain                                                  │
│   while read -t 0.01 -k 1 _discard; do :; done                 │
│   Consume any residual bytes (the push sequence, or fragments   │
│   of it) still sitting in stdin before ZLE takes over.           │
├─────────────────────────────────────────────────────────────────┤
│ Layer 3: Bind                                                   │
│   bindkey '\e[99;5u' send-break   # … and all other Ctrl+keys  │
│   Safety net: if CMUX continues sending CSI u despite the pop,  │
│   ZLE knows how to interpret every sequence.                     │
└─────────────────────────────────────────────────────────────────┘
```

Layer 1 alone doesn't reliably work (covered in the previous section). Layer 3 alone leaves the first-press bug. All three together eliminate the junk on every press, including the first.

### The Encoding Table

Every `Ctrl+<key>` combination has a deterministic CSI u encoding. The codepoint is the lowercase ASCII value; the modifier is `5` (Ctrl):

| Key     | Codepoint | CSI u Sequence   | Zsh Widget                               |
|---------|-----------|-------------------|------------------------------------------|
| Ctrl+A  | 97        | `\e[97;5u`        | `beginning-of-line`                      |
| Ctrl+B  | 98        | `\e[98;5u`        | `backward-char`                          |
| Ctrl+C  | 99        | `\e[99;5u`        | `send-break`                             |
| Ctrl+D  | 100       | `\e[100;5u`       | `delete-char-or-list`                    |
| Ctrl+E  | 101       | `\e[101;5u`       | `end-of-line`                            |
| Ctrl+F  | 102       | `\e[102;5u`       | `forward-char`                           |
| Ctrl+H  | 104       | `\e[104;5u`       | `backward-delete-char` (Backspace)       |
| Ctrl+K  | 107       | `\e[107;5u`       | `kill-line`                              |
| Ctrl+L  | 108       | `\e[108;5u`       | `clear-screen`                           |
| Ctrl+N  | 110       | `\e[110;5u`       | `down-line-or-history`                   |
| Ctrl+P  | 112       | `\e[112;5u`       | `up-line-or-history`                     |
| Ctrl+R  | 114       | `\e[114;5u`       | `history-incremental-search-backward`    |
| Ctrl+S  | 115       | `\e[115;5u`       | `history-incremental-search-forward`     |
| Ctrl+T  | 116       | `\e[116;5u`       | `transpose-chars`                        |
| Ctrl+U  | 117       | `\e[117;5u`       | `kill-whole-line`                        |
| Ctrl+W  | 119       | `\e[119;5u`       | `backward-kill-word`                     |
| Ctrl+Y  | 121       | `\e[121;5u`       | `yank`                                   |
| Ctrl+Z  | 122       | `\e[122;5u`       | custom: `kill -TSTP 0`                   |

### The Code

Add this to `~/.zshrc`:

```zsh
# ── CSI u / Kitty keyboard protocol fix (CMUX + libghostty) ──
# cmux sends modified keys as \e[<code>;<modifier>u (CSI u format).
# Three-part fix:
#   1. Pop the kitty keyboard mode pushed by CMUX on surface init
#   2. Drain any residual init bytes sitting in the input buffer
#   3. Register ZLE bindings so zsh understands CSI u sequences
if [[ -n "$CMUX_SOCKET_PATH" ]]; then
  # (1) Pop kitty keyboard mode — tell the terminal to stop CSI u encoding
  printf '\e[<u' 2>/dev/null
  # (2) Drain leftover init sequences from CMUX surface startup
  while read -t 0.01 -k 1 _cmux_discard 2>/dev/null; do :; done
  unset _cmux_discard
  # (3) ZLE keybindings for CSI u encoded Ctrl+key sequences
  bindkey '\e[99;5u'  send-break                          # Ctrl+C
  bindkey '\e[100;5u' delete-char-or-list                  # Ctrl+D
  bindkey '\e[97;5u'  beginning-of-line                    # Ctrl+A
  bindkey '\e[101;5u' end-of-line                          # Ctrl+E
  bindkey '\e[102;5u' forward-char                         # Ctrl+F
  bindkey '\e[98;5u'  backward-char                        # Ctrl+B
  bindkey '\e[107;5u' kill-line                             # Ctrl+K
  bindkey '\e[117;5u' kill-whole-line                       # Ctrl+U
  bindkey '\e[119;5u' backward-kill-word                    # Ctrl+W
  bindkey '\e[108;5u' clear-screen                          # Ctrl+L
  bindkey '\e[114;5u' history-incremental-search-backward   # Ctrl+R
  bindkey '\e[115;5u' history-incremental-search-forward    # Ctrl+S
  bindkey '\e[112;5u' up-line-or-history                    # Ctrl+P
  bindkey '\e[110;5u' down-line-or-history                  # Ctrl+N
  bindkey '\e[116;5u' transpose-chars                       # Ctrl+T
  bindkey '\e[121;5u' yank                                  # Ctrl+Y
  bindkey '\e[104;5u' backward-delete-char                  # Ctrl+H (backspace)
  # Ctrl+Z: send SIGTSTP to the foreground process group
  _cmux_ctrl_z() { kill -TSTP 0; }
  zle -N _cmux_ctrl_z
  bindkey '\e[122;5u' _cmux_ctrl_z                          # Ctrl+Z
fi
```

Then reload:

```bash
source ~/.zshrc
```

### Why This Works

1. **Layer 1 — Pop (`printf '\e[<u'`).** Runs during `.zshrc` init, before ZLE starts. Sends the kitty protocol pop sequence to CMUX's inner pty. If CMUX honours it, subsequent key presses revert to legacy encoding and the `bindkey` mappings become a dormant safety net. If CMUX ignores it, no harm done — layers 2 and 3 pick up the slack.

2. **Layer 2 — Drain (`read -t 0.01 -k 1` loop).** Consumes any bytes already in the input buffer — specifically the `\e[>1u` push sequence that CMUX wrote during surface creation. This eliminates the residual-buffer problem that caused the first-press junk. The `0.01` second timeout ensures we only drain what's already buffered without blocking shell startup.

3. **Layer 3 — Bind (`bindkey` mappings).** Registers every `Ctrl+key` CSI u sequence as a ZLE widget. Even if CMUX continues sending CSI u encoding (because the pop was ignored or re-pushed), ZLE now matches `\e[99;5u` → `send-break` on the first byte, every time.

4. **`send-break` does the right thing.** The ZLE widget `send-break` cancels the current input line and sends `SIGINT` to the shell's process group — identical to what happens when the tty driver sees `0x03`.

5. **The guard clause keeps it scoped.** The `if [[ -n "$CMUX_SOCKET_PATH" ]]` check ensures these bindings only activate inside CMUX sessions. Regular Ghostty, iTerm2, or SSH sessions are unaffected.

6. **Ctrl+Z needs special handling.** Unlike other Ctrl keys, `Ctrl+Z` is handled by the tty driver (it sends `SIGTSTP`), not by ZLE. Since the tty driver never sees `0x03`/`0x1a` in CSI u mode, we create a custom ZLE widget that manually sends `SIGTSTP` (signal 18) to the process group `0` (the foreground group).

---

## What About Running Processes?

There's a subtlety: `bindkey` only works when ZLE is active (i.e. you're at the prompt typing). When a foreground process is running (`sleep 100`, `npm run dev`, etc.), ZLE is not reading input — the process is.

In the legacy model, the kernel's tty line discipline intercepts `0x03` *before* it reaches the process and fires `SIGINT`. With CSI u encoding, the tty driver sees `\e[99;5u` — which isn't the configured `intr` character — so no signal is sent.

This is a deeper architectural issue in CMUX/libghostty. The terminal emulator should translate `Ctrl+C` back to `0x03` when writing to the inner pty if the child hasn't explicitly opted into the kitty keyboard protocol via the push escape. This is how standalone Ghostty handles it correctly.

Until CMUX addresses this at the emulator level, the ZLE-level bindings solve the most common case (prompt interaction), and `Ctrl+C` during running processes should still work if CMUX's own input layer translates the key before writing to the pty — which it does for most foreground programs.

---

## Appendix: The CSI u Modifier Bitmask

If you need to extend this for other modifier combos:

| Modifier Value | Keys Held          |
|----------------|--------------------|
| 2              | Shift              |
| 3              | Alt (Option)       |
| 4              | Shift + Alt        |
| 5              | Ctrl               |
| 6              | Ctrl + Shift       |
| 7              | Ctrl + Alt         |
| 8              | Ctrl + Alt + Shift |

For example, `Ctrl+Shift+C` would be `\e[99;6u`. To bind it:

```zsh
bindkey '\e[99;6u' some-widget  # Ctrl+Shift+C
```

---

*Fix tested on: CMUX (libghostty) with Ghostty shell, zsh 5.9, macOS Sequoia.*
