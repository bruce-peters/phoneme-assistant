# Phoneme Assistant

Early research prototype for the reading tutor that became [Word Wiz AI](https://wordwizai.com).
A reader says a sentence aloud. wav2vec2 models transcribe the audio into IPA phonemes and into
words, and the result is aligned against the sentence's expected phonemes (grapheme-to-phoneme).
That alignment yields a phoneme error rate (PER) plus the specific missed, added, and substituted
sounds, which GPT turns into child-friendly feedback and a next practice sentence.
`phoneme_assistant.py` can also read the feedback aloud with ElevenLabs text-to-speech.

Currently, this is just a development repository meant for experimenting

If you would like to test the phoneme assistant MVP follow these steps:
1. Clone the repository
2. Create a .env file containing the following content:
```
ELEVENLABS_API_KEY = [your api key here]
OPENAI_API_KEY= [your api key here]
```
4. Run the phoneme_assistant_app.py file (install necessary pip modules based on errors)
# Future App
This project is simply here to show an MVP of this system
In the future, this functionality will be in an actual app rather than just a repository

## Run it

Prerequisites:
- Python 3.12 (tested). Bytecode in the repo shows that 3.12 and 3.13 were both used.
- About 3.6 GB of disk for the two Hugging Face models, which download on first run:
  `speech31/wav2vec2-large-TIMIT-IPA` (phonemes) and
  `jonatasgrosman/wav2vec2-large-xlsr-53-english` (words)
- **Linux only:** the PortAudio library for `sounddevice` (`sudo apt install libportaudio2`),
  and `python3-tk` for the app. The macOS and Windows `sounddevice` wheels bundle PortAudio.
  Some scripts import the recording module even when they only read audio files, so you need
  it for those too.

```bash
git clone https://github.com/bruce-peters/phoneme-assistant
cd phoneme-assistant
python3.12 -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install torch --index-url https://download.pytorch.org/whl/cpu   # or plain `pip install torch` for CUDA
pip install -r requirements.txt
cp .env.example .env                 # only needed for the GPT / ElevenLabs parts
```

Run everything **from the repo root**, because paths such as `dataset/` and `gpt_prompts/` are
relative to the working directory.

| Command | Needs API keys? | What it does |
|---|---|---|
| `python phoneme_extractor.py` | No | Transcribes `audio_samples/test.wav` to IPA phonemes and prints them next to the expected phonemes for "zero three five one" |
| `python measure_accuracy.py` | No | Quantitative evaluation on the committed dataset (see below) |
| `python qualitative_analysis.py` | **Yes** | Interactive evaluation: runs the full assistant on 20 random clips and asks *you* to score accuracy and feedback quality from 1 to 5 |
| `python phoneme_assistant_app.py` | **Yes** | The tkinter MVP app. Records from the microphone and shows the feedback and next sentence |

API keys go in `.env` (see `.env.example`). OpenAI (`gpt-4o-mini`) and ElevenLabs are both
paid services. The two keyed scripts were not run in this review.

## Reproduce the evaluation

`dataset/` holds 53 recordings (`dataset/audio/<n>.wav`). Each has a matching
`dataset/transcriptions/<n>.json` that lists an *original* sentence, an *altered* sentence with
one word changed by a sound (for example "blue cup" → "blue cub"), both phoneme strings, and the
ground-truth PER between them.

```bash
MPLBACKEND=Agg python measure_accuracy.py    # drop MPLBACKEND=Agg to see the two plots
```

The script runs the audio → phoneme → alignment pipeline on 50 clips drawn at random (with
replacement, no seed). For each clip it compares the model's sentence PER against the dataset's
ground-truth PER, then prints the correlation coefficient, mean squared error, mean absolute
error, and average processing time per clip. It writes no result files. Because the sample is
unseeded, the numbers change from run to run. If the draw includes `dataset/audio/45.wav`, it
prints `Skipping index 44 … no speech` and leaves that clip out. On a CPU-only machine the full
run took 5–6 minutes after the models were downloaded.

## Results files in the repo

These files come from `qualitative_analysis.py`. That evaluation needs the paid APIs and a human
scorer, so it was not re-run here.

- `evaluation_results.csv` and `evaluation_results.txt` hold the same 19 scored samples:
  `speed_score`, `accuracy_score`, `feedback_quality_score` (1–5, rubric in
  `qualitative_analysis.py`), and `elapsed_time` in seconds. `elapsed_time` ranges from 5.8 to
  10.4 s, so every sample got speed score 1 (rubric: >5 s).
- `evaluation_summary.txt` reports average speed 0.95, accuracy 4.30, and feedback quality 4.35.
  Those averages divide by the 20 attempted samples, but only 19 were scored. Averaged over
  the 19 rows in the CSV they are 1.00, 4.53, and 4.58. The script now divides by the number of
  scored samples.
- `baseline-model-test.md` is a note on the baseline (no preprocessing, dynamic-programming
  alignment only).

## Project structure

| Path | What it is |
|---|---|
| `phoneme_assistant.py` / `phoneme_assistant_app.py` | `PhonemeAssistant` (models + GPT + TTS) and its tkinter UI |
| `phoneme_extractor.py`, `word_extractor.py` | wav2vec2 phoneme and word recognizers |
| `process_audio.py` | Word/phoneme alignment, PER, and the per-word analysis table |
| `grapheme_to_phoneme.py` | Expected phonemes via `eng_to_ipa` |
| `speech_problem_classifier.py` | Groups errors by phoneme class (vowels, plosives, …) |
| `audio_recording.py`, `audio_preprocessing.py` | Microphone capture with voice activity detection; noise reduction |
| `measure_accuracy.py`, `qualitative_analysis.py`, `evaluation/` | Evaluation scripts and the PER metric |
| `self_dataset_loader.py`, `data_creation.py`, `recorder*.py`, `dataset/` | The recorded dataset and the tools used to build it |
| `gpt_prompts/` | System prompts for the feedback model |
| `dataset_loader.py` | Loader for the public `mispeech/speechocean762` dataset (early experiments) |
| `montreal_forced_alignment.py`, `mfa_output/`, `visualize_spectrogram.py` | Montreal Forced Aligner experiment (needs a separate `mfa` install) |
| `phonecodes/`, `phonecode_conversions/` | IPA ↔ ARPAbet conversion helpers |
| `main.ipynb`, `testing/` | Exploration notebooks |
