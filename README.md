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
├── sample/
│   ├── corpus/                 # Minimal sample: audio + transcripts
│   │   ├── *.wav
│   │   └── *.txt
│   └── output/                 # Example TextGrids from MFA (segment tier)
│       └── *.TextGrid
├── phones.txt                  # List of phone symbols used in the dictionary
└── graphemes.txt               # List of grapheme characters (orthography)
```

- **`models/lb_acoustic_model.zip`** — Use this as the acoustic model in `mfa align` and `mfa segment`.
- **`dictionary/luxembourgish_mfa_run6.dict`** — Dictionary with pronunciation probabilities (trained on a Luxembourgish corpus). Use it as the dictionary in `mfa align`. Format: word, optional probability columns, then space-separated IPA phones.
- **`config/`** — YAML files used when *training* the acoustic model; you only need these if you retrain or adapt the model.
- **`sample/`** — A tiny corpus (wav + txt) and example TextGrids produced by MFA on a small test set (`mfa-test-small`).
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

### 1. Install the model and dictionary into MFA (one-time)

So that MFA can find the model and dictionary by name:

```bash
conda activate aligner

# Install the acoustic model (use a short name for convenience)
mfa model add acoustic lb_acoustic_lux PATH_TO_THIS_REPO/models/lb_acoustic_model.zip

# Install the dictionary
mfa dictionary add luxembourgish_run6 PATH_TO_THIS_REPO/dictionary/luxembourgish_mfa_run6.dict
```

### 2. Run alignment

```bash
mfa align \
  /path/to/your/corpus \
  PATH_TO_THIS_REPO/dictionary/luxembourgish_mfa_run6.dict \
  PATH_TO_THIS_REPO/models/lb_acoustic_model.zip \
  /path/to/output \
  --num_jobs 4
```

If you installed the model and dictionary as above, you can use their names:

```bash
mfa align \
  /path/to/your/corpus \
  luxembourgish_run6 \
  lb_acoustic_lux \
  /path/to/output \
  --num_jobs 4
```

Output: in `/path/to/output` you get the same folder structure as the input, with **TextGrid** files (and optional `.lab`), containing word- and phone-level alignments.

### 3. Try the sample corpus

From the repo root:

```bash
conda activate aligner

mfa align \
  sample/corpus \
  dictionary/luxembourgish_mfa_run6.dict \
  models/lb_acoustic_model.zip \
  sample/my_output \
  --num_jobs 2
```

Then open the generated `.TextGrid` files in [Praat](https://www.fon.hum.uva.nl/praat/) or any tool that reads TextGrids. The tiers contain segments/words and phones.

---

## Sample files and TextGrids

- **`sample/corpus/`** — A few **.wav** and **.txt** files from the internal test corpus `mfa-test-small`. Each `.txt` contains the orthographic transcript for the corresponding `.wav`. You can use this folder to test `mfa align` without preparing your own data.
- **`sample/output/`** — Example **TextGrid** files produced by MFA on the same small corpus. They contain at least a **segment** tier (and, when running full alignment, **word** and **phone** tiers). Use them as a reference for the output format and for checking alignment quality.

TextGrid format: [Praat TextGrid](https://www.fon.hum.uva.nl/praat/manual/TextGrid.html). MFA writes one tier per level (e.g. segments, words, phones) with intervals and labels.

---

## Grapheme and phone lists

- **`graphemes.txt`** — One character per line: the set of graphemes (letters and symbols) that appear in the dictionary’s orthographic forms. Useful for checking coverage of your orthography and for G2P or text normalization.
- **`phones.txt`** — One symbol per line: the set of **phones** (IPA-style symbols) used in the dictionary’s pronunciations. Useful for phonetic analysis, for building phoneme sets, and for checking that your dictionary and model use a consistent phone set.

Both lists are derived from `dictionary/luxembourgish_mfa_run6.dict`.

---

## Going further: G2P, OOV words, and dictionary expansion

In real corpora, **out-of-vocabulary (OOV)** words often appear. The supplied dictionary is large but will not contain every possible word. A typical workflow is:

1. Run **OOV detection** (e.g. `mfa find_oovs`) on your corpus.
2. Add pronunciations for OOVs, e.g. by **grapheme-to-phoneme (G2P)** prediction, manual transcription, or a mix.
3. **Merge** these new entries into the dictionary (and optionally **train dictionary** with `mfa train_dictionary` to get pronunciation probabilities).
4. Run **alignment** (and optionally **segmentation** with `mfa segment` first).

We use a more complex pipeline that includes:

- Number-to-words conversion (e.g. *12* → *zwielef*),
- G2P (Sequitur or MFA’s Luxembourgish G2P) for OOV words,
- Manual review and correction of OOV pronunciations,
- Dictionary training for probabilities,
- Optional segmentation before alignment.

A high-level description of this pipeline and the main steps is in **[PIPELINE.md](PIPELINE.md)**. It is aimed at researchers who want to adapt the workflow or integrate G2P and OOV handling into their own scripts.

---

## Citation and licence

If you use this model or dictionary in academic work, please cite the Montreal Forced Aligner and, if applicable, this repository.

- **Montreal Forced Aligner:**  
  McAuliffe et al., *Montreal Forced Aligner: Trainable Text-Speech Alignment Using Kaldi*, Interspeech 2017.

For licence and reuse of this repository’s contents, see the licence file in the repository (if present); otherwise, please contact the repository maintainers.

---

## Links

- **MFA documentation:** [https://montreal-forced-aligner.readthedocs.io/en/latest/](https://montreal-forced-aligner.readthedocs.io/en/latest/)
- **MFA pretrained models:** [https://montreal-forced-aligner.readthedocs.io/en/latest/pretrained_models.html](https://montreal-forced-aligner.readthedocs.io/en/latest/pretrained_models.html)
- **Praat:** [https://www.fon.hum.uva.nl/praat/](https://www.fon.hum.uva.nl/praat/)
