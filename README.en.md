# CHUWI CoreBook X on Arch Linux: only one speaker works (left channel only) (slowed by AI)

[Русская версия](README.md)

**Search terms:** CHUWI CoreBook X, Arch Linux, PipeWire, Realtek ALC269VC, only
one speaker works, sound only from one speaker, left channel only, right speaker
silent, no stereo, Windows 11 audio works, `line_outs=1`, HDA pin `0x1b`.

## Problem

On a **CHUWI CoreBook X** running Arch Linux, audio played from only one internal
speaker: the left channel was audible and the other speaker was silent. The same
laptop works correctly in Windows 11.

PipeWire balance and volume were normal. Both `front-left` and `front-right`
were at 100%, ALSA `Speaker`, `PCM`, and `Master` were unmuted, and
`speaker-test` produced both channels. The missing output was not an application
or balance problem; the second HDA speaker output was disabled by the BIOS pin
configuration.

## Hardware

| Item | Value |
|---|---|
| Laptop | CHUWI CoreBook X |
| PCI audio | Intel Ice Lake-LP Smart Sound Technology, `8086:34c8` |
| PCI SSID | `1e50:7036` |
| Codec | Realtek ALC269VC, `10ec:0269` |
| Codec subsystem | `1e50:7036` |
| OS | Arch Linux |
| Kernel | `7.2.9-arch1-1` |
| Audio stack | PipeWire 1.6.9 + WirePlumber |

## Root cause

The BIOS reports HDA pin `0x1b` as not connected (`0x411111f0`). The Linux HDA
auto-parser therefore configures only one speaker output:

```text
autoconfig for ALC269VC: line_outs=1 (0x14/0x0/0x0/0x0/0x0) type:speaker
```

Pin `0x1b` is wired to the second internal speaker but remains disabled. The
Windows Realtek driver applies a board-specific configuration, while the Linux
auto-parser does not.

Current upstream Linux already contains a quirk for this SSID:

```c
SND_PCI_QUIRK(0x1e50, 0x7036, "Acer Gadget E10 ETBook",
              ALC233_FIXUP_WUJIE_SPEAKERS),
```

Source:
<https://github.com/torvalds/linux/blob/master/sound/hda/codecs/realtek/alc269.c>

The tested Arch kernel did not apply it automatically yet, so the pin needs to
be overridden manually.

## Fix

Override pin `0x1b` with the same value used by the upstream quirk:

```text
0x1b: 0x90170150
```

Create the early firmware patch:

```bash
sudo tee /lib/firmware/hda-jack-retask.fw > /dev/null <<'EOF'
[codec]
0x10ec0269 0x1e507036 0

[pincfg]
0x1b 0x90170150
EOF
```

Tell `snd-hda-intel` to load it:

```bash
echo 'options snd-hda-intel patch=hda-jack-retask.fw' | \
  sudo tee /etc/modprobe.d/hda-jack-retask.conf
```

Reboot:

```bash
reboot
```

Applying the pin override at runtime through
`/sys/class/sound/card0/hwC0D0/reconfig` is unreliable while PipeWire is
running and can fail with:

```text
The codec is being used, can't reconfigure.
```

The early firmware patch is applied before PipeWire starts and survives reboots.

## Verify

```bash
journalctl -k -b --no-pager | grep -E 'autoconfig for ALC269|speaker_outs'
```

Expected:

```text
autoconfig for ALC269VC: line_outs=2 (0x14/0x1b/0x0/0x0/0x0) type:speaker
```

Check the second speaker control:

```bash
amixer -c 0 scontents | grep -A5 -i 'Bass Speaker'
```

Test both channels:

```bash
speaker-test -D default -c 2 -t wav -l 1
```

Also verify that `Speaker`, `Bass Speaker`, `PCM`, and the active PipeWire sink
are not muted and not at zero volume.

## Rollback

```bash
sudo rm -f /etc/modprobe.d/hda-jack-retask.conf \
           /lib/firmware/hda-jack-retask.fw
sudo reboot
```

## Notes

The upstream `ALC233_FIXUP_WUJIE_SPEAKERS` quirk may make this workaround
unnecessary once it reaches the installed distro kernel. Keep the pin override
until `line_outs=2` appears without it.

## Files

- [`hda-jack-retask.fw`](hda-jack-retask.fw) — HDA early firmware patch;
- [`hda-jack-retask.conf`](hda-jack-retask.conf) — kernel module option.
