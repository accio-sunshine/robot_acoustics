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
├── experiments/              exp01–exp06, each a small settings file
├── voices/                   male.wav, female.wav, baby.wav (inputs for sim_conversation.py)
├── hark/                     mic_positions.xml, source_positions.xml
├── speech_cache/             CMU Arctic clips downloaded on first run
├── results/                  one folder per experiment (baseline outputs)
├── sim_noise_out/            robot-noise scenes + motor_state.csv
├── sim_conversation_out/     conversation scenes
└── web/                      robot_acoustic.html (robot's-eye listening lab) and array_bench.html
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
| `exp06_talker_distance` | Talker distance | 0.75, 1.5, 2.25 m |
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

### Experiment 6: talker distance

| Scene | MUSIC ° | SRP-PHAT ° | Mic 1 dB | DAS dB | MVDR dB |
|---|---|---|---|---|---|
| d075_close (0.75 m) | 33, 41 | 40, 147 | 3.7 | 4.3 | 6.3 |
| d150_medium (1.5 m) | 41, 332 | 37, 155 | 2.3 | 3.2 | 4.9 |
| d225_far (2.25 m) | 43, 335 | 38, 147 | 0.1 | 0.6 | 4.3 |

Moving the talkers away costs mic 1 and delay-and-sum about 3.6 dB from 0.75 m to 2.25 m. MVDR loses only 2 dB.
SRP-PHAT stays accurate at every distance.

## Interactive bench

`web/array_bench.html` is a browser version of `sim_8mic_room.py` with every scene setting as a slider:
RT60, SNR, room size, array radius, talker directions and distance, plus page-only knobs for the number of
mics and the DOA frequency band. Results update instantly, you can click the room plan to move a talker,
and each experiment scene is a preset that shows the Python result next to the page's estimate.
It also builds the `dict(...)` line to paste into a new experiment.

It uses a lighter room model (exact reflections up to 2 bounces, statistical late echo, measured speech
spectra). Against every experiment scene, mic-1 and delay-and-sum scores land within about 0.5 dB of
Python in most scenes, MVDR within about 1.5 dB, and SRP-PHAT within a few degrees. Use it to choose
scenes, then confirm in Python.

## Robot Acoustic page

**Live site:** https://accio-sunshine.github.io/robot_acoustics/ (8-Mic Array Bench at `/array_bench.html`).
It is rebuilt and published by `.github/workflows/pages.yml` on every push to `main`.

`web/robot_acoustic.html` is the conversation scene from the robot's point of view. The robot's head is in
the middle with its 8 mics, the man, woman and baby can be dragged around it, and every robot noise (fan,
neck servo, shoulder servos, brake clicks, 50 Hz hum, footsteps) has an on/off switch and a loudness knob,
along with the baby, room echo, noise floor and mic ring size. Each change re-renders all 8 microphones in
the browser. You can listen to any mics (stereo for headphones), to delay-and-sum, MVDR and learned-robot-noise
beamformers aimed at each speaker, and to each sound alone, with a timeline, waveform and spectrogram.
The Python recordings from `sim_conversation_out/` and `sim_noise_out/` can be played and viewed alongside.

The page uses the same voices, noise recipes, levels and beamformers as the scripts. The room is lighter
(exact reflections up to 3 bounces plus a statistical late echo). Beamformer scores land within about
0.5 dB of Python in the baby and fan scenes. Footsteps are approximate and start switched off.

Edit `web/src/engine.js` or `web/src/robot_acoustic.template.html`, then rebuild with
`cd robot-audio/web && python build.py`. To use the Python recordings locally, serve the `robot-audio`
folder (`python -m http.server`) and open `http://localhost:8000/web/robot_acoustic.html`.

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
