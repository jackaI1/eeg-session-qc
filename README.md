# EEG session quality check

I built a recording quality report because I needed a way to tell whether an EEG recording was worth analysing before using it. EEG measures small electrical changes through sensors on the scalp. A loose sensor, a blink, muscle tension or electrical interference can hide the signal I want to study.

My tool reads a recording, marks suspicious sensors and times, and gives it a score with a reason: **usable**, **usable with caveats** or **not usable**. I can see what needs attention before deciding which parts to keep. It screens the recording without changing it.

**[Open a saved one-page report (PDF)](reports/S004R01.pdf)**. No installation needed. The report explains the decision, shows a timeline and suggests what to inspect. I also saved [standalone HTML reports](reports/) that open in a browser after downloading, alongside the full measured evidence in JSON.

This is a research screening project, **not a diagnostic tool**. A flag suggests interference; it does not prove its cause. “Usable” does not guarantee that a recording is suitable for every analysis.

## Try it

I use [uv](https://docs.astral.sh/uv/getting-started/installation/) to install Python and the locked dependencies:

```sh
git clone https://github.com/jackaI1/eeg-session-qc.git
cd eeg-session-qc
uv sync --locked --python 3.12
uv run eeg-qc demo
```

The demo downloads public PhysioNet recordings, saves the reports in `reports/`, and runs the injection study. Open `reports/S004R01.html` in a browser. I do not need an account or any private recordings.

To check a different public EDF file (a common recording format):

```sh
uv run eeg-qc report path/to/public-recording.edf --output reports/my-session
# For a recording collected where mains electricity is 50 Hz:
uv run eeg-qc report path/to/public-recording.edf --line-hz 50 --output reports/my-session
```

The command writes both `.html` and `.json`. For other EDFs, I check that auxiliary sensors have standard type prefixes such as `EOG` or `EMG` followed by a space; MNE excludes those sensors. Untyped channels are assumed to be EEG, so an EDF with unlabelled auxiliary sensors needs its channel types corrected before use. It rejects recordings that are too short, contain invalid values or cannot resolve the electrical frequency being checked. Missing forehead sensors produce an explicit caveat because blink screening cannot run.

## What I checked

I used the first four eyes-open baseline recordings from the public EEG Motor Movement/Imagery dataset. Here, a “session” means one EDF recording, not a participant's entire visit. No substitute or synthetic recording was used for these session reports.

These are the saved results, reproduced by `uv run eeg-qc demo` and printed by `uv run eeg-qc summary`:

| Recording | Score | Decision |
| --- | ---: | --- |
| S001R01 | 30.1/100 | Not usable |
| S002R01 | 6.1/100 | Not usable |
| S003R01 | 49.4/100 | Usable with caveats |
| S004R01 | 9.1/100 | Not usable |

Each recording has 64 sensors, samples at 160 Hz and lasts 61 seconds. The conservative rules flag substantial interference in these recordings, including flat signal near the end. A low score means I should inspect the data before use, not that the participant was unwell.

## How I tested the flags

I added known interference to copies of quieter parts of the real EEG, leaving the originals unchanged. I tried different strengths of sensor dropout, random sensor noise, a repeating electrical signal, a slow forehead pulse resembling a blink, and a burst of fast interference resembling muscle tension. A success requires the right type, target sensor and time window.

The reference screen found 2 qualifying segments out of 28 candidates. Both came from S004R01, with 55 quiet sensors retained in each; excluded sensors are listed in [the trial evidence](reports/validation.json). These are low-amplitude references, not independently certified artifact-free EEG. I did not filter or synthesize the reference signals.

| Added interference | Detected / trials | Detection rate |
| --- | ---: | ---: |
| Flat sensor | 6 / 6 | 100.0% |
| Noisy sensor | 4 / 6 | 66.7% |
| Electrical line noise | 6 / 6 | 100.0% |
| Blink-like pulse | 6 / 6 | 100.0% |
| Muscle-like burst | 6 / 6 | 100.0% |

The weakest noisy-sensor injections were missed in both reference segments. The untreated references also had line-noise flags elsewhere, so I record untreated flags and paired detections separately. None of the injection targets was already flagged for that artifact at that time. These repeated trials share the same two segments, so the rates are a small controlled check, not an estimate of performance across people or recording systems. I have no expert-labelled blink or muscle events here, so I cannot claim real-world sensitivity or specificity.

All numbers above come from `uv run eeg-qc summary`; [provenance](reports/provenance.json) records the source hashes and library versions. [The methods notes](docs/methods.md) explain the rules, injection strengths and limitations.

## How the code is arranged

I use MNE-Python to read EDF signals in volts and estimate their frequency content. I inspect short time windows for nearly constant sensors, unusually large signals, mains-frequency peaks, slow forehead changes and excess fast activity. The last two checks are proxies for blinks and muscle activity because these recordings do not include dedicated eye or muscle sensors.

- `src/eeg_session_qc/quality.py` measures the signals and explains the score.
- `src/eeg_session_qc/report.py` builds a self-contained, printable summary.
- `src/eeg_session_qc/validation.py` runs paired artifact injections and untreated controls.
- `src/eeg_session_qc/data.py` downloads only pinned public files and checks their SHA-256 fingerprints.
- `src/eeg_session_qc/sources.json` lists each exact public download URL.

The score combines unflagged sensor-time with time free of electrical, blink-like and muscle-like flags. The decision also uses explicit rejection thresholds. It is a rule-based triage score, not a learned probability. Thresholds live in the JSON report; I have not tuned them against expert labels.

## Sources and reuse

I used public data only. The EDF downloads remain in the ignored `data/` folder; saved reports contain public sensor measurements and public recording IDs. The code is MIT licensed; the dataset retains its own attribution licence.

- Schalk, G. [EEG Motor Movement/Imagery Dataset, PhysioNet](https://physionet.org/content/eegmmidb/1.0.0/) ([dataset DOI](https://doi.org/10.13026/C28G6P)). The dataset page describes collection and the Open Data Commons Attribution licence. Every file used is linked in `sources.json`.
- Schalk and colleagues, [BCI2000: A General-Purpose Brain-Computer Interface System](https://doi.org/10.1109/TBME.2004.827072), the original recording-system publication requested by the dataset.
- Pollard and colleagues, [PhysioNet as a global platform for biomedical research](https://doi.org/10.1038/s44360-026-00096-z), the platform citation requested by the dataset page.
- [MNE-Python EDF import documentation](https://mne.tools/stable/auto_tutorials/io/20_reading_eeg_data.html) and [Welch spectrum documentation](https://mne.tools/stable/generated/mne.time_frequency.psd_array_welch.html). My screening thresholds and injection rules are project choices, not published diagnostic criteria.

## Tests and reproducibility

```sh
uv run pytest -q
uv run ruff check src tests scripts
uv run ruff format --check src tests scripts
uv run eeg-qc demo --output /tmp/eeg-qc-reproduced
uv run python scripts/check_evidence.py /tmp/eeg-qc-reproduced
uv build
```

CI runs these checks on pull requests and on main. It tests the detector, scoring, invalid inputs, download integrity, standalone reports and validation, then downloads the pinned public EDFs and reproduces the saved session decisions and injection evidence. It uploads regenerated reports as a CI artifact. The unit tests work offline; the public-data integration test runs once the demo data are available.
