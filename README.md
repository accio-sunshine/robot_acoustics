# robot_acoustics

Simulated 8-microphone robot-head array for studying speech capture on a robot:
sound source localization, beamforming, dereverberation, robot ego-noise, and
speech-recognition scoring. HARK (via PyHARK) is used as a comparison.

All code lives in [`robot-audio/`](robot-audio/). **Run every script from inside that folder**,
because they read and write paths relative to it.

## Setup

Requires Python 3.10+.

```bash
cd robot-audio
pip install -r requirements.txt
python check_setup.py        # checks libraries, files and internet, then runs the three simulations once
```

`check_setup.py` expects three voice recordings in `robot-audio/voices/`:
`male.wav`, `female.wav`, `baby.wav`. If they are missing, `sim_conversation.py` will
generate substitutes (TTS speech and a synthetic cry).

## Scripts

| Script | What it does | Output |
|---|---|---|
| `sim_8mic_room.py` | Core room simulation: reverberant room, 8-mic circular array (r = 5 cm), MUSIC / SRP-PHAT localization, delay-and-sum and MVDR (MPDR) beamforming, WPE dereverberation. Used by the experiments. | `results/exp01_echo_levels/` |
| `sim_8mic_robot_noise.py` | Adds synthetic robot ego-noise (fan, neck/shoulder servos, footsteps, brake clicks, 50 Hz hum) and writes `motor_state.csv` for ego-noise prediction. | `sim_noise_out/` |
| `sim_conversation.py` | Two-speaker conversation (male at 40°, female at 150°) with a baby crying at 250°, with and without robot noise. | `sim_conversation_out/` |
| `score_whisper.py` | Transcribes outputs with faster-whisper and computes word error rate (WER). | `results/wer_results.csv`, `results/transcripts.txt` |
| `make_hark_xml.py` | Writes mic and source position XML for HARKTOOL5 to build a geometric transfer function. | `hark/` |
| `pyhark_localize.py` | Runs HARK offline (MUSIC → SourceTracker → GHDSS) on a simulated recording. Needs `hark/tf_8mic.zip` from HARKTOOL5. | `hark_out/` |

Angle convention everywhere: 0° = mic 1 (+x), 90° = +y, counter-clockwise.

## Typical workflow

```bash
cd robot-audio
python sim_conversation.py           # simulate the scenes
python score_whisper.py              # score them (base.en; pass small.en for more accuracy)

# Optional HARK comparison
python make_hark_xml.py              # then build hark/tf_8mic.zip in HARKTOOL5-GUI
python pyhark_localize.py
```

## Notes

- Generated output folders (`results/`, `sim_*_out/`, `hark_out/`, `speech_cache/`) are git-ignored.
- Audio and other large binaries (`*.wav`, `*.zip`, ...) are tracked with Git LFS.
  Run `git lfs install` once before committing them.
