# Luxembourgish Acoustic Model for Montreal Forced Aligner (MFA)

This repository provides a **ready-to-use acoustic model** and **pronunciation dictionary** for forced alignment of **Luxembourgish** speech using the [Montreal Forced Aligner](https://montreal-forced-aligner.readthedocs.io/en/latest/) (MFA). It is intended for linguists, computational linguists, and researchers working with Luxembourgish speech data.

The model and dictionary correspond to **training run 6** from our internal training pipeline and have been used successfully for word- and phone-level alignment of Luxembourgish corpora.

---

## What is forced alignment?

Forced alignment means matching a **transcript** (orthographic text) to an **audio recording** so that each word—and optionally each phone—gets precise start and end times. The result is typically a **TextGrid** (Praat format) with tiers for segments, words, and phones. Such alignments are the basis for phonetic analysis, pronunciation modelling, and building speech technology for Luxembourgish.

---

## Documentation and installation

- **MFA user guide and reference:**  
  [https://montreal-forced-aligner.readthedocs.io/en/latest/](https://montreal-forced-aligner.readthedocs.io/en/latest/)

We recommend installing MFA in a **Conda** environment so that dependencies stay isolated and reproducible.

### Install MFA with Conda

1. **Install Miniconda** (if you don’t have Conda yet):  
   [https://docs.conda.io/en/latest/miniconda.html](https://docs.conda.io/en/latest/miniconda.html)

2. **Create and activate an environment** (e.g. Python 3.10 or 3.11):

   ```bash
   conda create -n aligner python=3.11 -y
   conda activate aligner
   ```

3. **Install Montreal Forced Aligner** from conda-forge:

   ```bash
   conda install -c conda-forge montreal-forced-aligner
   ```

4. **Check the installation:**

   ```bash
   mfa version
   ```

For more options (e.g. other Python versions, GPU use), see the official [MFA installation documentation](https://montreal-forced-aligner.readthedocs.io/en/latest/getting_started/installation.html).

---

## Repository structure

```
mfa-luxembourgish-published/
├── README.md                    # This file
├── PIPELINE.md                  # Advanced pipeline (G2P, OOV, dictionary expansion)
├── environment.yml              # Optional: conda env spec for reproducibility
├── models/
│   └── lb_acoustic_model.zip    # Luxembourgish acoustic model (training run 6)
├── dictionary/
│   └── luxembourgish_mfa_run6.dict   # Pronunciation dictionary (trained, with probabilities)
├── config/                     # Optional MFA config used during training
│   ├── lb_rules.yaml           # Pronunciation rules (e.g. degemination)
│   └── lb_phone_groups.yaml   # Phone groups for training
├── g2p_models/                 # G2P models for OOV pronunciation generation
│   ├── model-8                 # Sequitur G2P (Luxembourgish)
│   └── lb_g2p.zip              # MFA Luxembourgish G2P model
├── sample/
│   ├── corpus/                 # Minimal sample
│   │   ├── kraider.wav, kraider.txt
│   │   └── RTL_1.wav, RTL_1.txt
│   └── output/                 # Example TextGrids (segment tier) 
│       └── kraider.TextGrid, RTL_1.TextGrid
├── phones.txt                  # List of phone symbols used in the dictionary
└── graphemes.txt               # List of grapheme characters (orthography)
```

- **`models/lb_acoustic_model.zip`** — Use this as the acoustic model in `mfa align` and `mfa segment`.
- **`dictionary/luxembourgish_mfa_run6.dict`** — Dictionary with pronunciation probabilities (trained on a Luxembourgish corpus). Use it as the dictionary in `mfa align`. Format: word, optional probability columns, then space-separated IPA phones.
- **`config/`** — YAML files used when *training* the acoustic model; you only need these if you retrain or adapt the model.
- **`g2p_models/`** — Grapheme-to-phoneme models for generating pronunciations for out-of-vocabulary (OOV) words: **model-8** (Sequitur) and **lb_g2p.zip** (MFA). See [PIPELINE.md](PIPELINE.md) and the section *G2P models for OOV conversion* below.
- **`sample/`** — Two recordings from the test corpus, each with `.wav` + `.txt`, plus example TextGrids (segment tier) produced by MFA.
- **`phones.txt`** and **`graphemes.txt`** — Reference lists of the phone set and grapheme set used in the dictionary.

---

## Quick start: align your corpus

Your corpus must be in **MFA format**: one folder (or hierarchy of folders) containing, for each recording, a **`.wav`** file and a **`.txt`** file with the same base name and the orthographic transcript (one file per recording).

Example layout:

```
my_corpus/
├── speaker1/
│   ├── recording1.wav
│   ├── recording1.txt
│   ├── recording2.wav
│   └── recording2.txt
└── speaker2/
    ├── session_a.wav
    └── session_a.txt
```

Paths below assume you have cloned this repo and you run commands from the repo root. Replace `PATH_TO_THIS_REPO` with the actual path.

### Minimal pipeline: segment, then align

We recommend a **two-step workflow**: first **segment** the audio+text corpus (VAD and transcript-based segmentation), then **align** the segmented corpus. This way alignment respects segment boundaries and is usually more stable.

1. **Segment** — From audio + transcript (`.wav` + `.txt`), MFA creates a **segmented corpus**: each recording gets a **TextGrid** with a **segment** tier (and the transcript text per segment). The segmented corpus is a folder with the same structure but **only `.wav` + `.TextGrid`** (no `.txt`).
2. **Align** — Run `mfa align` on that **segmented** corpus (wav + segment TextGrid only). MFA then adds **word** and **phone** tiers to the TextGrids.

So the minimal pipeline is:

```
your_corpus/ (wav + txt)  →  mfa segment  →  segmented_corpus/ (wav + segment TextGrid)
                                                      ↓
                                              mfa align  →  output/ (TextGrids with segments + words + phones)
```

### 1. Install the model and dictionary into MFA (one-time)

So that MFA can find the model and dictionary by name:

```bash
conda activate aligner

# Install the acoustic model (use a short name for convenience)
mfa model add acoustic lb_acoustic_lux PATH_TO_THIS_REPO/models/lb_acoustic_model.zip

# Install the dictionary
mfa dictionary add luxembourgish_run6 PATH_TO_THIS_REPO/dictionary/luxembourgish_mfa_run6.dict
```

### 2. Segment your corpus

**Note:** `mfa segment` requires [SpeechBrain](https://speechbrain.github.io/) (for VAD/segmentation). Install it if needed, e.g. `pip install speechbrain` or via the [MFA segmentation documentation](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/corpus_creation/create_segments.html).

```bash
mfa segment \
  /path/to/your/corpus \
  PATH_TO_THIS_REPO/dictionary/luxembourgish_mfa_run6.dict \
  PATH_TO_THIS_REPO/models/lb_acoustic_model.zip \
  /path/to/segmented_corpus \
  --num_jobs 4 \
  --clean --overwrite
```

Input: corpus with **`.wav`** and **`.txt`** per recording.  
Output: **`/path/to/segmented_corpus`** with the same layout but **`.wav`** + **`.TextGrid`** (segment tier only; no `.txt`). Use this folder as the input to `mfa align`.

### 3. Align the segmented corpus

```bash
mfa align \
  /path/to/segmented_corpus \
  PATH_TO_THIS_REPO/dictionary/luxembourgish_mfa_run6.dict \
  PATH_TO_THIS_REPO/models/lb_acoustic_model.zip \
  /path/to/output \
  --num_jobs 4
```

Input: **segmented corpus** (wav + segment TextGrid only).  
Output: **TextGrid** files with **segment**, **word**, and **phone** tiers.

If you installed the model and dictionary by name, you can use:

```bash
mfa align \
  /path/to/segmented_corpus \
  luxembourgish_run6 \
  lb_acoustic_lux \
  /path/to/output \
  --num_jobs 4
```

### 4. Try the sample corpus

From the repo root, run the full pipeline on the included sample:

```bash
conda activate aligner

# Step 1: Segment
mfa segment \
  sample/corpus \
  dictionary/luxembourgish_mfa_run6.dict \
  models/lb_acoustic_model.zip \
  sample/segmented \
  --num_jobs 2 --clean --overwrite

# Step 2: Align (input = segmented corpus: wav + TextGrid only, no .txt)
mfa align \
  sample/segmented \
  dictionary/luxembourgish_mfa_run6.dict \
  models/lb_acoustic_model.zip \
  sample/my_output \
  --num_jobs 2
```

Then open the generated `.TextGrid` files in [Praat](https://www.fon.hum.uva.nl/praat/). The tiers contain segments, words, and phones. 

---

## Sample files and TextGrids

- **`sample/corpus/`** — Two recordings from the test corpus. For each you have a **.wav** and a **.txt** with the orthographic transcript. Use this folder to run the minimal pipeline (segment → align) without preparing your own data.
- **`sample/output/`** — Example **TextGrid** files produced by MFA (segment tier). After running `mfa align` on the segmented corpus you get full TextGrids with **segment**, **word**, and **phone** tiers.

Example of an aligned TextGrid (segment, word, and phone tiers):

![Aligned TextGrid](sample/output/Aligned_TextGrid.png)

TextGrid format: [Praat TextGrid](https://www.fon.hum.uva.nl/praat/manual/TextGrid.html). MFA writes one tier per level (e.g. segments, words, phones) with intervals and labels.

---

## Grapheme and phone lists

- **`graphemes.txt`** — One character per line: the set of graphemes (letters and symbols) that appear in the dictionary’s orthographic forms. Useful for checking coverage of your orthography and for G2P or text normalization.
- **`phones.txt`** — One symbol per line: the set of **phones** (IPA-style symbols) used in the dictionary’s pronunciations. Useful for phonetic analysis, for building phoneme sets, and for checking that your dictionary and model use a consistent phone set.

Both lists are derived from `dictionary/luxembourgish_mfa_run6.dict`.

---

## G2P models for OOV conversion

This repo includes two **grapheme-to-phoneme (G2P)** models in **`g2p_models/`** so you can generate IPA pronunciations for **out-of-vocabulary (OOV)** words and add them to the dictionary:

| File | Description |
|------|--------------|
| **`model-8`** | **Sequitur G2P** model (Luxembourgish), trained on dictionary data. Use with the [Luxembourgish sequitur-g2p](https://github.com/PeterGilles/sequitur-g2p) tool, e.g. `g2p.py --model g2p_models/model-8 --apply oov_words.txt`. |
| **`lb_g2p.zip`** | **MFA Luxembourgish G2P** model. Install with `mfa g2p add lb_g2p g2p_models/lb_g2p.zip`, then run `mfa g2p oov_words.txt lb_g2p oov_pronunciations.txt`. |

Typical integration: (1) run `mfa find_oovs` on your corpus to get a list of OOV words; (2) convert that list to one word per line; (3) run either Sequitur or MFA G2P to get word → IPA; (4) merge the new entries into your dictionary; (5) optionally run `mfa train_dictionary` and then segment + align. Full details, command examples, and workflow options are in **[PIPELINE.md](PIPELINE.md)**.

---

## Links

- **MFA documentation:** [https://montreal-forced-aligner.readthedocs.io/en/latest/](https://montreal-forced-aligner.readthedocs.io/en/latest/)
- **Praat:** [https://www.fon.hum.uva.nl/praat/](https://www.fon.hum.uva.nl/praat/)
