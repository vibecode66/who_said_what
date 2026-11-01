# who_said_what

Speaker-attributed meeting transcripts: **who said what, and when**. It transcribes a multi-speaker recording, separates the voices, lets a person put names to them by ear, and runs a second, more careful pass on the part of the meeting that matters.

## What this code does

You give it **one audio recording of a meeting**, such as an M4A from a phone or a Teams/Zoom export. You get back a **written transcript with each line labeled by who said it, and when**.

```
[00:26:01] Speaker A: We need to know which devices are for the packet core domain...
[00:26:41] Speaker B: Do you think we can get those details?
[00:26:48] Speaker C: It's not going to be straightforward...
```

It does this in five steps:

1. **Converts the audio** to a standard format (16 kHz mono WAV). It makes two copies: one lightly cleaned for transcription, and one untouched for voice recognition.
2. **Turns speech into text** with OpenAI's Whisper, recording a timestamp for every word.
3. **Works out who spoke when.** It recognizes the different voices and groups them as SPEAKER_00, SPEAKER_01, and so on, without knowing anyone's name yet.
4. **Lets a person name the voices.** It plays a short sample clip of each speaker in the notebook. You listen, type the real names once, and they're applied to the whole transcript. This human step is what makes the names reliable.
5. **Runs a second, more careful pass** on the part of the meeting you care about, or the whole meeting. It saves one audio clip per line so any name can be checked by ear and corrected in a single step.

**Outputs**

| File | Use |
|---|---|
| `focus_refined.md` | Readable transcript with speaker names |
| `focus_refined.csv` | Same transcript, opens in Excel to filter or sort by speaker |
| `focus_refined.srt` | Subtitles: play alongside the audio in VLC |
| `transcript.md` | Full first-pass transcript |
| MP4 (optional) | The audio with speaker-labeled subtitles burned in, which plays in Windows Media Player |

Lines are flagged **⚠** where people talk over each other and **?** where the speech was hard to make out, so you know which lines to double-check.

**Good for:** meeting minutes, finding who answered a specific question, or feeding a clean "who said what" transcript to an LLM for summaries and action items.

## Customize for your meeting

- **Starting prompt** (`DEFAULT_PROMPT` in `who_said_what.py`, or `--prompt`): list the attendees' names and any technical terms, so Whisper spells them correctly.
- **Keywords** (`DEFAULT_KEYWORDS`, or `find --keywords`): words that mark the discussion you want to locate.
- **Speaker count**: `--min-speakers` / `--max-speakers` for the first pass, and the exact number with `--speakers` for the refined pass.

## Files

| File | What it is |
|---|---|
| `who_said_what_colab.ipynb` | Google Colab notebook. Runs the whole process step by step, with no Google Drive needed |
| `who_said_what.py` | The script the notebook runs. It can also be run from the command line |

## Quick start (Colab)

1. Open `who_said_what_colab.ipynb` in Colab (**File → Upload notebook**).
2. **Runtime → Change runtime type → T4 GPU → Save**.
3. Create a Hugging Face **Read** token, and click **Agree** on the pages for `pyannote/speaker-diarization-community-1` and `pyannote/wespeaker-voxceleb-resnet34-LM`.
4. Run the cells **one at a time, from top to bottom**. After the install cell, use **Runtime → Restart session**, then continue from cell 2.
5. Stage 2 and Stage 5 stop for you to listen to the clips and type in the names.
6. Download the results in Stage 6 before you close the tab.

## Stages

| Stage | Command | What it does |
|---|---|---|
| 1 | `run` | M4A → WAV → Whisper transcript with word timestamps → speaker separation → one sample clip per speaker |
| 2 | (notebook) | **You** listen to each speaker's clip and fill in `NAMES` |
| 3 | `find` / `render` | Searches for keywords to find the discussion, then writes the full transcript and an excerpt for the chosen window |
| 4 | `refine` | Runs separation and transcription again on the chosen window with the exact number of speakers, and carries the names over by voice |
| 5 | `refine --rerender` | **You** check each turn's audio clip and correct names in the `FIX` cell |
| 6 | (notebook) | Downloads `.md`, `.csv` and `.srt`, plus an optional MP4 with the speaker names burned in as subtitles |

## 1. Hardware and platform

| Item | What we used |
|---|---|
| Platform | Google Colab, free tier |
| GPU | **NVIDIA Tesla T4** with 15 GB memory, selected under *Runtime → Change runtime type → T4 GPU* |
| Runtime | Python 3 (Python 3.13), runtime version *Latest (recommended)* |
| Storage | Colab session disk at `/content/MDE`, with no Google Drive. Files are deleted when the session ends |
| GPU drivers and libraries | NVIDIA **CUDA 12**, **cuBLAS** and **cuDNN 9**, added for faster-whisper |

## 2. Python libraries

| Category | Library | Role |
|---|---|---|
| **Deep learning framework** | PyTorch | Runs the speaker models on the GPU |
| **Speech-to-text** | faster-whisper | Runs Whisper and produces word timestamps |
| | CTranslate2 | Fast inference engine underneath faster-whisper |
| **Speaker separation and voice matching** | pyannote.audio | Splits speakers and creates voice fingerprints |
| **Math and matching** | NumPy | Audio arrays and similarity scores |
| | SciPy | One-to-one pairing of voices with names |
| **Audio input and output** | soundfile | Reads and writes WAV files |
| **Notebook interface** | IPython.display, google.colab | Audio players, file upload and download, token prompt |
| **Standard library** | json, pathlib, argparse, subprocess, re | Settings files, file paths, command-line stages |

## 3. AI models

| Task | Model | Source |
|---|---|---|
| **Voice → text** | Whisper **large-v3** (float16 on GPU) | OpenAI, via faster-whisper |
| Skipping silence | Silero VAD | Built into faster-whisper |
| **Who spoke when** | **speaker-diarization-community-1** | pyannote, on Hugging Face |
| **Voice fingerprints** | **WeSpeaker ResNet34** (256-number embedding) | pyannote, on Hugging Face |

## 4. Other tools

| Tool | Role |
|---|---|
| **FFmpeg** | Converts M4A to 16 kHz mono WAV, cleans up the audio, and builds the MP4 with subtitles |
| **Hugging Face token** (Read) | Gives access to the pyannote models |

## 5. How each step is done

| Step | How |
|---|---|
| **Recognize voices** | pyannote finds speech, takes a fingerprint of each voice, and groups them into SPEAKER_00 to SPEAKER_05 |
| **Convert voice to text** | Whisper large-v3 with word timestamps and a starting prompt of names and terms |
| **Label speakers** | **You** listen to the clips and fill in `NAMES`. The refined pass carries the names over by voice similarity |
| **Validate** | Similarity table, ⚠ and ? flags, audio clip for every turn, the `FIX` cell, and playback with the SRT or MP4 |

## Known issues and fixes

- **`libcublas.so.12 is not found`**: Colab doesn't include the CUDA 12 libraries faster-whisper needs. The install cell adds `nvidia-cublas-cu12` and `nvidia-cudnn-cu12==9.*`, and cell 2 points the system to them. Rerun cell 2 after any restart.
- **Red "dependency conflicts" warnings during install** are harmless.
- **One person split across two labels**: give both labels the same name in `NAMES` or `FIX`.
- **Overlapping speech** (⚠) can split one sentence between two speakers. Check those lines by ear.

## Command line (outside Colab)

```bash
pip install faster-whisper pyannote.audio soundfile scipy   # plus ffmpeg on PATH
export HF_TOKEN=hf_...
python who_said_what.py run    "meeting.m4a" --min-speakers 5 --max-speakers 8
# listen to <analysis>/speaker_samples/*.wav, fill in speaker_map.json
python who_said_what.py find   "meeting.m4a"
python who_said_what.py render "meeting.m4a" --focus 00:05:30-00:19:00
python who_said_what.py refine "meeting.m4a" --start 00:05:30 --end 00:19:00 --speakers 4
```

> Recordings, transcripts and audio clips are meeting data. Keep them out of this repo; `.gitignore` excludes common audio and output files.
