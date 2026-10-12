---
name: uefi_sound_output
description: 'How to output sound from a UEFI application: PC-speaker beep via PIT/port 0x61, and real WAV playback via OpenCore AudioDxe EFI_AUDIO_IO_PROTOCOL; no standard UEFI audio protocol exists.'
tags:
- uefi
- edk2
- audio
- sound
- opencore
- hda
- pc-speaker
created_at: '2026-10-01T13:16:54.478147885+02:00'
---

# Sound output from a UEFI application

## Key fact
The UEFI specification defines **no** audio protocol. There is no standard way to play sound. Options below.

## 1. PC speaker beep (simplest, no driver)
Uses PIT channel 2 + port 0x61. Works on many desktops, NOT on most laptops (no speaker) and not in VMs without speaker emulation. Single tone only.

```c
#include <Uefi.h>
#include <Library/IoLib.h>
#include <Library/TimerLib.h>

STATIC VOID
PcSpeakerBeep (UINT32 Frequency, UINT32 DurationMs)
{
  UINT32 Divisor;
  if (Frequency == 0) return;
  Divisor = 1193182 / Frequency;          // PIT base clock 1.193182 MHz
  IoWrite8 (0x43, 0xB6);                  // channel 2, mode 3 (square), lobyte/hibyte
  IoWrite8 (0x42, (UINT8)(Divisor & 0xFF));
  IoWrite8 (0x42, (UINT8)((Divisor >> 8) & 0xFF));
  IoWrite8 (0x61, IoRead8 (0x61) | 0x03); // bits 0+1: speaker gate + data enable
  MicroSecondDelay (DurationMs * 1000);
  IoWrite8 (0x61, IoRead8 (0x61) & ~0x03);
}
```
Requires IoLib + TimerLib. Pure port I/O, works in any DXE app.

## 2. EFI_AUDIO_IO_PROTOCOL (OpenCore / Acidanthera) — real WAV playback
- `AudioDxe.efi` initializes Intel HDA controllers/codecs and exposes one `EFI_AUDIO_IO_PROTOCOL` per codec.
- Header: `OpenCorePkg/Include/Acidanthera/Protocol/AudioIo.h`
- Methods: `StartPlayback`, `StartPlaybackAsync`, `StopPlayback`; takes a WAV (PCM) buffer.
- NOT part of the UEFI spec — an Acidanthera extension. App only works if AudioDxe.efi was loaded first (e.g. OpenCore `UEFI/Drivers`).

```c
EFI_AUDIO_IO_PROTOCOL *AudioIo;
if (!EFI_ERROR (gBS->LocateProtocol (&gEfiAudioIoProtocolGuid, NULL, (VOID **)&AudioIo))) {
  AudioIo->StartPlayback (AudioIo, WavBuffer, WavSize);
}
```

## 3. Own driver
- Intel HDA: MMIO + codec verb commands.
- USB audio via `EFI_USB_IO_PROTOCOL`.
- VirtIO sound for VMs (edk2 discussions around `EFI_AUDIO_OUTPUT_PROTOCOL`).
All hardware-specific and significant effort.

## Recommendation
- Signal tone / feedback -> PC speaker (option 1).
- Music/WAV -> OpenCore AudioDxe + EFI_AUDIO_IO_PROTOCOL, document the driver dependency.
- Fully portable across all hardware -> not feasible; sound belongs in the OS.

## References
- OpenCorePkg (Acidanthera) AudioDxe / OcAudioLib
- Goldfish64/AudioPkg (archived, read-only, HDA stack for UEFI)
