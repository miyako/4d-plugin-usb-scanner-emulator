# 4d-plugin-usb-scanner-emulator

A 4D plugin that types text into whatever window currently has keyboard focus, as if it came from a keyboard or a USB barcode scanner in "keyboard wedge" mode. It works by sending synthetic key-down/key-up events to the operating system (`CGEventPost` on macOS, `SendInput` on Windows), so the receiving application cannot tell the input apart from real typing. It was written for testing: you can exercise a scan-entry field in your own application without owning a scanner.

The plugin exposes one command and returns nothing.

| Command | Returns | Purpose |
|---|---|---|
| [POST TEXT](#post-text) | — | Types a text value, one key press per character, into the focused window |

**Platforms:** macOS and Windows.

---

## Requirements & platform notes

- **The text goes to the focused window, not to 4D.** The keystrokes are injected at operating-system level. Whichever application and field has focus when the command runs receives them. If that is not the field you intended, the text lands somewhere else.
- **Failure is silent.** `POST TEXT` never raises a 4D error and never returns a status. If the operating system refuses the input, or a character can't be typed, nothing tells you. See [Error handling & troubleshooting](#error-handling--troubleshooting).
- **macOS permission.** Synthetic keyboard events are only delivered if the host application (4D, 4D Server, or your merged application) has been allowed to control the computer. Grant it under **System Settings → Privacy & Security → Accessibility**. Depending on your macOS version, an additional input-related entry may be listed there; if events are ignored, check that section for the 4D application. The command itself does not check or request the permission.
- **Windows elevation.** Windows blocks injected input into windows that run with higher privileges than the sender. If the target application runs as administrator and 4D does not, the keystrokes are dropped.
- **One mandatory parameter.** `POST TEXT` takes exactly one text parameter. There is no optional form.
- **The command blocks the calling process until every key has been queued.** It returns when the events are handed to the OS, not when the target application has finished processing them. Long texts take proportionally longer, and there is no built-in length limit.
- **Where it runs matters.** On 4D Server, a stored procedure or server-side method that calls `POST TEXT` types on the server machine's desktop, not on a client's.
- **Security.** This command can type into any application, including password prompts and terminals. Keep it out of production builds.

**Which characters can be typed differs by platform:**

| | macOS | Windows |
|---|---|---|
| Letters | `a`–`z`, `A`–`Z` (upper case is sent as the key plus Shift) | Anything the current keyboard layout can produce |
| Digits | `0`–`9`, sent as numeric-keypad keys | Sent as normal digit keys, per the keyboard layout |
| Symbols | `= - + *` (keypad keys), `( ) : ; \ , / .`, space | Anything the keyboard layout can produce, including Shift/Ctrl/Alt combinations the layout requires |
| Carriage return (`Char(13)`) | Return key | Enter key |
| Line feed (`Char(10)`) | Keypad Enter key | Enter key |
| Tab (`Char(9)`) | Tab key | Tab key |
| Anything else | **Skipped** (not typed) | **Skipped** if the layout has no key for it |

On macOS the keys are sent as physical key positions based on a US (ANSI) keyboard. On a non-US layout, the character that appears can differ from the one you passed. On Windows, the key lookup uses the keyboard layout of the 4D process, which is not necessarily the layout of the window receiving the keys.

A line ending of `Char(13)+Char(10)` produces two Enter presses. 4D text normally uses `Char(13)` alone.

**Version note.** The skipping of unmapped characters, the Shift handling, and the Enter/Tab handling described here apply to the revised source (`4DPlugin.cpp` delivered with this document). Earlier builds behaved differently: on macOS an unmapped character sent a bare Command key press, upper case was sent as lower case, and `(`, `)` and `:` produced `[`, `]` and `'`; on Windows, characters needing Shift were sent incorrectly. Rebuild the plugin to get the behavior documented here. This was verified only against test stubs, not on real macOS or Windows hardware.

---

## POST TEXT

### Syntax

```
POST TEXT ( text )
```

| Parameter | Type | Description |
|---|---|---|
| `text` | Text | The characters to type. Each character becomes one key press (down then up). An empty string does nothing. |
| Result | — | None. The command returns no value and sets no error. |

### Description

`POST TEXT` walks through `text` one character at a time and sends a key-down and key-up event for each one to the operating system. The events go to the application that has keyboard focus at that moment.

Characters with no mapping on the current platform are **skipped**: the remaining characters are still typed, and nothing is reported. Check the character table in [Requirements & platform notes](#requirements--platform-notes) before relying on punctuation or non-ASCII text.

**On macOS**, digits and the arithmetic characters `=`, `-`, `+`, `*` are sent as numeric-keypad keys, which is how many scanners present themselves. Upper-case letters, `(`, `)` and `:` are sent with Shift. If the operating system can't create a keyboard event, the command stops typing at that point and returns without an error.

**On Windows**, each character is looked up on the keyboard layout and sent with whatever Shift, Ctrl or Alt state that layout needs. The events are sent to Windows in batches, and if Windows refuses a batch (for example because the target is an elevated window), the command stops and returns without an error.

To emulate a scanner that ends each code with Enter, append `Char(13)`.

The command does not pause between keys. If the target application can't keep up with very fast input, split the text and delay between the pieces.

### Example

Type a code followed by Enter, from a separate process so you have time to click into the target field first. The `DELAY PROCESS` value is in ticks (1/60 s), so 180 is about three seconds.

```4d
// Project method: Scan_Type
DELAY PROCESS(Current process; 180)
POST TEXT("4006381333931"+Char(13))
```

```4d
// Run it
$p:=New process("Scan_Type"; 0; "Scan_Type")
```

Simulate a burst of scans, one per second:

```4d
C_LONGINT($i)
DELAY PROCESS(Current process; 180)
For ($i; 1; 5)
	POST TEXT("ITEM"+String(1000+$i)+Char(13))
	DELAY PROCESS(Current process; 60)
End for
```

If your text uses line feeds, convert them so Enter is sent once per line on both platforms:

```4d
C_TEXT($t)
$t:="LINE1"+Char(10)+"LINE2"+Char(10)
$t:=Replace string($t; Char(10); Char(13))
POST TEXT($t)
```

---

## Error handling & troubleshooting

- **Nothing was typed on macOS.** The most likely cause is that the 4D application hasn't been granted the macOS permission to control the computer (see [Requirements & platform notes](#requirements--platform-notes)). Events are dropped with no error. Restart the application after changing the setting.
- **Nothing was typed on Windows.** If the focused window belongs to an application running as administrator, Windows discards the input. Run 4D at the same privilege level, or test against a non-elevated window.
- **The text arrived in the wrong place.** The keystrokes go to the focused window at the moment the command runs. If 4D itself has focus, they go to the 4D window. Run the command from a separate process after a delay, as in the example, and click into the target first.
- **Some characters are missing.** Characters with no mapping are skipped without any message. On macOS only the characters listed in the table can be typed. On Windows a character is skipped when the current keyboard layout has no key for it.
- **The wrong characters appear.** The macOS key codes assume a US layout, and the Windows lookup uses 4D's keyboard layout rather than the target window's. Test with the keyboard layout you will actually use.
- **Two Enter presses per line.** The text contains `Char(13)+Char(10)`. Use `Char(13)` alone (or convert as shown in the example).
- **The target drops or garbles keys on long text.** The command sends keys without pausing. Split the text into pieces and add `DELAY PROCESS` between them.
- **The calling process is busy for a long time.** The command blocks until all keys are queued, and there is no size limit. Avoid passing very large texts, particularly from a user-interface process.
- **Unexpected 4D errors.** The plugin catches internal exceptions and ignores them. If the command appears to do nothing at all, treat that as a failure to deliver input rather than an error you can trap.

---

## Quick reference

```4d
// Type a code and press Enter (US layout characters are the safest)
POST TEXT("4006381333931"+Char(13))

// Delayed, from its own process, so you can focus the target field first
$p:=New process("Scan_Type"; 0; "Scan_Type")

// Line feeds -> Enter, once per line, on both platforms
POST TEXT(Replace string($t; Char(10); Char(13)))

// Several scans with a pause in between
For ($i; 1; 5)
	POST TEXT("ITEM"+String(1000+$i)+Char(13))
	DELAY PROCESS(Current process; 60)
End for
```
