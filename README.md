# kinesis-edge

Keep a Kinesis Freestyle Edge RGB+ keyboard's onboard config in git, and get
it back onto the keyboard without the vendor app. Three commands: `status`,
`pull`, `push`.

    git clone https://github.com/interslice-systems/kinesis-edge ~/src/kinesis-edge
    ln -s ~/src/kinesis-edge/bin/kinesis-edge ~/.local/bin/kinesis-edge

`device/` holds the author's own layouts, lighting and settings. Fork it and
`pull` your own keyboard over them; the tooling does not care what is in
there. Built for one desk. Shared in case it's useful on yours. No warranty,
no promises, no roadmap — but if it breaks in an interesting way, an issue is
welcome.

## What this is

Git-tracked mirror of the keyboard's **v-Drive**, the 4 MB FAT volume the
firmware exposes over USB when you press **SmartSet + F8** (the gear key on the
right module). Everything the keyboard knows about remaps, macros, lighting and
global settings lives there as plain `.txt`. No driver, no vendor app.

Model `KB975-LIN`, firmware `1.0.121`. The `-LIN` suffix is the switch type
(linear), matching the `-BLU/RED/BRN/SIL` variants in Kinesis's own docs;
"RGB+" is the marketing name for this hotswap revision, not a separate
firmware line. The
[Direct Programming Guide v1.0.121](https://gaming.kinesis-ergo.com/wp-content/uploads/2020/09/Freestyle-Edge-Direct-Programming-Guide-Version-1.0.121-Sept-16-2020.pdf)
is authoritative for it.

## Use

    kinesis-edge status    # mount state, firmware, what differs
    kinesis-edge pull      # device -> repo
    kinesis-edge push      # repo -> device, then flush + unmount

Edit files under `device/`, `push`, then press SmartSet + F8 to close the
v-Drive (or **SmartSet + Layout** to apply without closing). `push` unmounts for
you; never toggle the shortcut on a still-mounted volume.

### Three states, not two

Ejecting in the file manager looks like "nothing happened" because it only
moves you from the third row to the second — the keyboard stays connected.

| State | Symptom | To leave it |
|---|---|---|
| closed | `29ea:0102`, no block device | `SmartSet + F8` |
| connected, unmounted | `29ea:0103`, `/dev/sdX` exists, no mount | `udisksctl mount` (or just run `pull`/`push`) |
| mounted | files readable under `/run/media/` | `udisksctl unmount` |

`pull` and `push` mount automatically from the middle state. Pressing
`SmartSet + F8` there sends you backwards to a closed drive.

`push` writes `layouts/`, `lighting/` and `settings/` only. `firmware/version.txt`
is device-generated and pull-only.

## Things that will bite

- **CRLF is load-bearing.** The firmware writes `\r\n` and its parser expects it.
  This repo stores LF; `kinesis-edge` converts at the boundary. Do not `cp` files
  onto the v-Drive by hand.
- **The keyboard changes USB identity.** Normally `29ea:0102` with three HID
  interfaces; with the v-Drive open it re-enumerates as `29ea:0103` with mass
  storage plus a single HID interface. NKRO and the consumer-control endpoints
  are absent while it is open, which is why onboard programming is disabled then.
  Any udev rule must match both PIDs, or match on `iProduct`.
- **`lsusb` misnames it** "Advantage2 Keyboard" — a stale `usb.ids` entry for
  `29ea:0102`. The device's own string descriptor is correct.
- **The guide's settings key names are wrong.** It prints `Led_mode` and
  `v-drive`; the real file uses `led_mode` and `v_drive`. Trust the file.
- **`nkro_mode` is absent** from `kbd_settings.txt` on this firmware even though
  the guide lists it. Set it with the onboard shortcut and re-`pull` to see
  whether the field appears; do not invent the line.
- Eject before `SmartSet + F8`, always. It is a real FAT volume with real dirty
  pages.

## Syntax, in brief

Square brackets remap, curly brackets macro, `>` splits position from action.
A leading `*` comments out a line. On conflicting lines, **the last one wins**.
Tokens are case-insensitive, and capitalising one does *not* produce the shifted
action. Prefix `fn ` (outside the brackets) addresses the embedded layer.

    [esc]>[caps]                                  Escape sends Caps Lock
    fn [lwin]>[rshft]                             Fn+Win sends Right Shift
    [caps]>[caps][t&h500][esc]                    tap Caps, hold 500ms for Esc
    {pause}>{-lshft}{h}{+lshft}{e}{l}{l}{o}       types "Hello" (- down, + up)
    {lctrl}{hk5}>{s9}{q}{w}{e}{r}{t}{y}           co-trigger, playback speed 9
    fn {tab}>{x1}{lmous}{d125}{lmous}             one double-click, 125ms gap

Lighting takes an effect plus parameters, or per-key RGB triples. A `[mono]` on
line 2 sets a base colour for the two-tone effects:

    [wave]>[spd5][dirright]
    [reactive]>[0][255][255][spd5]
    [mono]>[0][0][0]
    [w]>[255][0][0]                               per-key, Freestyle/Breathe

Limits: 9 profiles, 100 macros and 7,200 macro characters per layout, 10
tap-and-hold actions per layout, 300 keystrokes per macro.

## License

MIT. See `LICENSE`.
