---
name: task-completion-chime
description: Configure, diagnose, and adjust the local Codex task-completion chime, including the Stop hook, PulseAudio/PipeWire playback, and system-volume compensation. Use when the user asks to create, repair, tune, or explain this completion notification. Do not use for general media playback or unrelated application notifications.
---

# Task Completion Chime

Maintain the user's short, distinctive Codex completion sound without changing the user's system volume.

## Local layout

- Hook configuration: `/home/guga/.codex/hooks.json`
- Notification directory: `/home/guga/tool/chatgpt/notification`
- Playback entrypoint: `/home/guga/tool/chatgpt/notification/notify`
- Primary audio: `/home/guga/tool/chatgpt/notification/completed.wav`
- Optional matching copy: `/home/guga/tool/chatgpt/notification/completed.mp3`
- Local documentation: `/home/guga/tool/chatgpt/notification/README.md`

Inspect these files before editing. Preserve unrelated hooks and configuration. The intended hook event is `Stop`, which means “停止/结束” and fires when a Codex turn stops. The hook command should call the `notify` entrypoint rather than invoking an audio player directly.

## Desired behavior

- Play a distinctive three-note chime of about 0.7 seconds when a Codex turn completes.
- Keep the chime at the original 100%-system-volume loudness while system volume is between 50% and 100%.
- Below 50%, reduce the final signal level linearly toward silence.
- Respect system mute and never change the user's sink volume or mute state.
- Wait for playback to finish and leave no background process.

The current sound is intentionally short and has a source peak near `-20.5 dB`, leaving headroom for playback compensation.

## PulseAudio compensation

Read the first percentage from `pactl get-sink-volume @DEFAULT_SINK@`. PulseAudio applies the same nonlinear, approximately cubic curve to sink and stream volumes, so ordinary amplitude multiplication is incorrect.

Let `p` be the sink percentage as a fraction and `t = 0.5` be the boundary:

- For `p >= t`, set the stream scalar ratio to `s = 1 / p`.
- For `0 < p < t`, set `s = ((1 / t) / p^2)^(1/3)`. This makes the final signal decrease linearly below the boundary.
- At zero volume, do not attempt division; system output remains silent.

PulseAudio's 100% stream-volume integer is `65536`. On this machine, `paplay` accepts values above `65536`, which are required for inverse compensation. If the playback backend changes, verify this capability with a short silent file before using an over-100% value. Never test it with a loud signal first.

Expected checkpoints with the 50% boundary:

| System volume | Stream volume |
| ---: | ---: |
| 100% | 100% |
| 75% | 133.3% |
| 50% | 200% |
| 40% | 232.1% |
| 25% | 317.5% |
| 10% | 584.8% |

## Hook shape

Merge this shape into `hooks.json` when the hook is missing; do not overwrite other events:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/home/guga/tool/chatgpt/notification/notify",
            "timeout": 15,
            "statusMessage": "播放任务完成提示音"
          }
        ]
      }
    ]
  }
}
```

Use a new chat or restart Codex when a newly created hook is not picked up by the active session.

## Audio changes

When the user asks to redesign the chime, preserve the filenames so the hook does not need to change. Prefer PCM mono WAV at 24 kHz and keep the MP3 copy synchronized. Use a temporary directory, inspect duration and peak level, install the validated files, and clean the temporary directory. Avoid clipping and do not normalize away the headroom needed by compensation.

## Validation

Validate only what changed:

1. Run `python3 -m json.tool /home/guga/.codex/hooks.json` after hook edits.
2. Run `bash -n /home/guga/tool/chatgpt/notification/notify` after script edits.
3. Run `notify --check` to verify the audio service without playing sound.
4. Check WAV duration and peak level with `ffprobe` and FFmpeg `volumedetect` after audio edits.
5. Perform one actual playback when useful, tell the user first, and confirm no `paplay` process remains.

Do not change the system volume merely to test checkpoints. Test calculations independently or inspect the chosen stream volume with shell tracing. Keep test output and temporary files cleaned up.

For remote execution, follow the SSH guidance already maintained in the notification directory's `README.md`; do not invent a host address or assume passwordless SSH is configured.
