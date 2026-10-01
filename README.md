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

## Folder layout

```
robot-audio/
├── sim_8mic_room.py          core room simulation, used by every experiment
├── sim_8mic_robot_noise.py   adds robot ego-noise
├── sim_conversation.py       two talkers + baby crying, with/without robot noise
├── score_whisper.py          Whisper transcription + word error rate
├── make_hark_xml.py          mic/source geometry for HARKTOOL5
├── pyhark_localize.py        HARK localization + separation
├── check_setup.py            Phase 0 environment check
├── experiments/              exp01–exp05, each a small settings file
├── voices/                   male.wav, female.wav, baby.wav (inputs for sim_conversation.py)
├── hark/                     mic_positions.xml, source_positions.xml
├── speech_cache/             CMU Arctic clips downloaded on first run
├── results/                  one folder per experiment (baseline outputs)
├── sim_noise_out/            robot-noise scenes + motor_state.csv
└── sim_conversation_out/     conversation scenes
```

## Scripts

| Script | What it does | Output |
|---|---|---|
| `sim_8mic_room.py` | Reverberant room, 8-mic circular array (r = 5 cm), MUSIC / SRP-PHAT localization, delay-and-sum and MVDR (MPDR) beamforming, WPE dereverberation. Experiments call its `run_experiment()`. Run directly, it reproduces exp01. | `results/<experiment>/` |
| `sim_8mic_robot_noise.py` | Synthetic robot ego-noise (fan, neck/shoulder servos, footsteps, brake clicks, 50 Hz hum); also writes `motor_state.csv` (fan RPM, joint velocities, walking) for ego-noise prediction. | `sim_noise_out/` |
| `sim_conversation.py` | Male at 40° and female at 150° (1.5 m), baby crying at 250° (2.5 m); scenes: baby only, + fan, + all robot noise. | `sim_conversation_out/` |
| `score_whisper.py` | Transcribes conversation outputs with faster-whisper and computes WER. | `results/wer_results.csv`, `results/transcripts.txt` |
| `make_hark_xml.py` | Mic and candidate-source positions (every 5°) for a geometric transfer function in HARKTOOL5. | `hark/` |
| `pyhark_localize.py` | HARK offline: MultiFFT → LocalizeMUSIC → SourceTracker → GHDSS. Needs `hark/tf_8mic.zip` built in HARKTOOL5-GUI. | `hark_out/` |

Angle convention everywhere: 0° = mic 1 (+x), 90° = +y, counter-clockwise.

## Experiments

Each file in `experiments/` changes one thing from the default scene
(RT60 0.5 s, radius 5 cm, SNR 15 dB, talker A at 40°, talker B at 150°, 1.5 m, 6×5×3 m room).

```bash
cd robot-audio
python experiments/exp01_echo_levels.py
```

| Experiment | Varies | Scenes |
|---|---|---|
| `exp01_echo_levels` | Room echo (RT60) | 0.2, 0.5, 0.8 s |
| `exp02_array_size` | Array radius | 3, 5, 10 cm |
| `exp03_background_noise` | Background SNR | 25, 15, 5 dB |
| `exp04_talker_spacing` | Talker B direction | one talker, 110°, 50°, 30° apart |
| `exp05_test_set` | Fixed test set reused by every later phase. **Don't change it**; add a new experiment instead. | one_talker, two_far, two_close, dry_room, echoey_room |

Each scene folder holds the 8-channel mix, mic-1 input, delay-and-sum, MVDR, WPE and
WPE→MVDR outputs, clean references for scoring, and a DOA spectrum plot.
Each experiment has `summary.txt` / `summary.csv`.

## Baseline results

Talker A vs everything else, in dB (higher = cleaner). DAS = delay-and-sum. From `results/exp05_test_set/summary.txt`:

| Scene | True ° | MUSIC ° | SRP-PHAT ° | Mic 1 dB | DAS dB | MVDR dB |
|---|---|---|---|---|---|---|
| one_talker | 40 | 28 | 38 | 15.0 | 23.1 | 8.9 |
| two_far | 40, 150 | 41, 332 | 37, 155 | 2.3 | 3.2 | 4.9 |
| two_close | 40, 70 | 42, 53 | 46, 162 | 0.8 | 1.1 | 2.3 |
| dry_room | 40, 150 | 20, 32 | 40, 147 | 2.2 | 3.7 | 6.4 |
| echoey_room | 40, 150 | 39, 323 | 32, 161 | 2.2 | 3.0 | 3.3 |

What the baseline shows:
- SRP-PHAT finds talker A within about 8° in every scene; MUSIC often puts the second source in the wrong place (e.g. 332° instead of 150°).
- MVDR beats delay-and-sum whenever there are two talkers; its gain drops from 6.4 to 3.3 dB as echo goes from 0.2 to 0.8 s.
- Talkers 30° apart are hard to separate with a 5 cm array (MVDR only 2.3 dB).
- With a single talker, MVDR (8.9 dB) is far below delay-and-sum (23.1 dB), which points to self-cancellation in the MPDR form and is worth investigating.

## Typical workflow

```bash
cd robot-audio
python experiments/exp05_test_set.py   # fixed test set
python sim_conversation.py             # conversation scenes
python score_whisper.py                # WER (base.en; pass small.en for more accuracy)

# Optional HARK comparison
python make_hark_xml.py                # then build hark/tf_8mic.zip in HARKTOOL5-GUI
python pyhark_localize.py
```

## Notes

- Outputs in `results/`, `sim_noise_out/` and `sim_conversation_out/` are committed as the
  reference baseline. Re-running the scripts overwrites them, so commit new results deliberately.
- `hark_out/` and Python caches are git-ignored.
