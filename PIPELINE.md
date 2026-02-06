# Advanced alignment pipeline: G2P, OOV handling, and dictionary expansion

This document describes a **more complex workflow** for aligning Luxembourgish speech when your corpus contains **out-of-vocabulary (OOV) words**—i.e. words that are not in the pronunciation dictionary. 

---

## Why is this needed?

The dictionary shipped in this repo (`luxembourgish_mfa_run6.dict`) is large (tens of thousands of entries) but cannot list every possible word. In practice you will encounter:

- **Rare or dialectal forms**
- **Proper nouns and neologisms**
- **Numbers** (if they appear as digits in the transcript)
- **Spelling variants** or typos

If MFA encounters a word that is not in the dictionary, alignment can fail or produce poor results for that utterance. So we need a way to:

1. **Find** which words in the corpus are OOV.
2. **Assign pronunciations** to those words (e.g. via G2P or manual IPA).
3. **Add** them to the dictionary (and optionally train pronunciation probabilities).
4. **Run alignment** (and optionally segmentation) with the expanded dictionary.

---

## Overview of the pipeline

A typical high-level workflow looks like this:

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Corpus         │     │  OOV detection    │     │  OOV list       │
│  (wav + txt)    │ ──► │  (mfa find_oovs)  │ ──► │  word, count    │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                           │
                                                           ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Segmented      │     │  mfa segment      │     │  G2P + manual    │
│  corpus         │ ◄── │  (recommended)    │     │  review for OOV   │
│  (wav+TextGrid) │     └──────────────────┘     └────────┬────────┘
└────────┬────────┘                                        │
         │                                                   ▼
         │              ┌─────────────────┐     ┌─────────────────┐
         │              │  Expanded       │     │  Merge OOV      │
         │              │  dictionary    │ ◄── │  into dictionary│
         │              │  (base + OOV)   │     └─────────────────┘
         │              └────────┬────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌──────────────────┐
│  Aligned        │     │  mfa align        │
│  TextGrids      │ ◄── │  (on segmented    │
└─────────────────┘     │   corpus)         │
                         └──────────────────┘
```

Steps in words:

1. **OOV detection** — Run `mfa find_oovs` on your corpus with the current dictionary. MFA reports which words in the transcripts are not in the dictionary (and often how often they occur).
2. **Number conversion** — If your transcripts contain digits (e.g. *12*, *2024*), convert them to Luxembourgish words (e.g. *zwielef*, *zweedausendvéieranzwanzeg*) using the [Luxembourgish num2words](https://github.com/PeterGilles/num2words) package. Then treat those word forms as normal vocabulary (add them to the dictionary if they are still OOV).
3. **G2P for OOVs** — For each OOV word, generate a candidate pronunciation (IPA) using one of the **G2P models** included in this repo: **model-8** (Sequitur) or **lb_g2p.zip** (MFA). See the section *G2P models: integrating model-8 and lb_g2p.zip* below.
4. **Manual review** — Inspect the OOV list with suggested IPA. Correct mispronunciations, dialectal variants, and proper nouns. Save the result as word–IPA pairs (or word–replacement–IPA if you normalize spelling).
5. **Dictionary expansion** — Merge the base dictionary with the new OOV entries (word + IPA). Optionally run **`mfa train_dictionary`** on a corpus to estimate **pronunciation probabilities** for multiple variants.
6. **Segmentation (recommended)** — Run **`mfa segment`** on your corpus (wav + txt). This produces a **segmented corpus** (wav + segment TextGrid only). Alignment then uses this folder so that MFA respects segment boundaries.
7. **Alignment** — Run **`mfa align`** on the **segmented** corpus with the expanded (and optionally trained) dictionary and the Luxembourgish acoustic model. Output: TextGrids with segment, word, and phone tiers.
8. **Merge (optional)** — If you need a single TextGrid per file that combines segment boundaries with word/phone tiers from a separate align run, use a small script (e.g. with `praatio`) to merge them.

---

## Main tools and commands

### OOV detection

```bash
mfa find_oovs CORPUS_DIR DICTIONARY_PATH OUTPUT_DIR [--num_jobs N] [--clean] [--overwrite]
```

- **CORPUS_DIR** — Your corpus (wav + txt).
- **DICTIONARY_PATH** — Path to the dictionary (or its name if installed).
- **OUTPUT_DIR** — Where MFA writes OOV lists (e.g. `oov_counts_*.txt` with word and count).

See the [MFA documentation on finding OOVs](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/workflows/finding_oovs.html).

### Number conversion (Luxembourgish num2words)

If your transcripts contain digits, dates, times, or currency (e.g. *12*, *2024*, *10:34*, *1,50 EUR*), convert them to Luxembourgish words **before** OOV detection or alignment. The **[num2words](https://github.com/PeterGilles/num2words)** package provides Luxembourgish support (language code `lb`), including a text normalizer that converts numbers and related expressions in place.

**Install** (this Luxembourgish-enhanced version is not on PyPI; install from GitHub):

```bash
pip install git+https://github.com/PeterGilles/num2words.git
```

Or clone and install in development mode:

```bash
git clone https://github.com/PeterGilles/num2words.git
cd num2words
pip install -e .
```

**Usage for corpus text:** The repo includes `luxembourgish_normalizer.py`, which converts numbers, dates, times, percentages, currency, etc. in a text file to Luxembourgish word forms:

```bash
python luxembourgish_normalizer.py input_file.txt
# or pipe text
echo "Den 12. Abrëll 2024" | python luxembourgish_normalizer.py
```

**In Python** (e.g. in a custom script):

```python
from num2words import num2words
num2words(42, lang='lb')           # 'zweeavéierzeg'
num2words(2024, lang='lb', to='year')  # 'zweedausendvéieranzwanzeg'
```

Run number conversion on your corpus `.txt` files (with backups) before running `mfa find_oovs` or `mfa segment`, so that digit forms are replaced by words that can be looked up in the dictionary.

### Dictionary training (pronunciation probabilities)

```bash
mfa train_dictionary CORPUS_DIR DICTIONARY_PATH ACOUSTIC_MODEL OUTPUT_DIR [--num_jobs N] [--clean] [--overwrite]
```

- **DICTIONARY_PATH** — Base dictionary (word + IPA, possibly with multiple pronunciations per word).
- **ACOUSTIC_MODEL** — The Luxembourgish acoustic model (e.g. `models/lb_acoustic_model.zip`).
- **OUTPUT_DIR** — Directory where MFA writes the trained dictionary (with probabilities). You can then use this trained dictionary in `mfa align`.

See the [MFA documentation on training dictionaries](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/workflows/training_dictionary.html).

### Segmentation (optional)

**Requirement:** `mfa segment` uses [SpeechBrain](https://speechbrain.github.io/) for VAD and segmentation. Install it if needed (e.g. `pip install speechbrain`). See the [MFA segmentation documentation](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/corpus_creation/create_segments.html) for details.

```bash
mfa segment CORPUS_DIR DICTIONARY_PATH ACOUSTIC_MODEL OUTPUT_DIR [--num_jobs N] [--single_speaker] [--clean] [--overwrite]
```

- **CORPUS_DIR** — Corpus with wav + txt (or symlinks). MFA will run VAD and split by transcript.
- **OUTPUT_DIR** — Segmented corpus: wav (or links) + TextGrids with a **segment** tier. For the next step, you use only wav + TextGrid (no `.txt`) so that `mfa align` aligns within segments.

See the [MFA documentation on segmentation](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/corpus_creation/create_segments.html).

### Alignment

```bash
mfa align CORPUS_DIR DICTIONARY_PATH ACOUSTIC_MODEL OUTPUT_DIR [--num_jobs N] [--beam 100] [--retry_beam 400]
```

- Use the **expanded** (and optionally **trained**) dictionary and the **Luxembourgish acoustic model** from this repo.

---

## G2P models: integrating model-8 and lb_g2p.zip

This repository includes two G2P models in **`g2p_models/`** so you can generate IPA pronunciations for OOV words and add them to your dictionary. You can use either or both.

### 1. Sequitur G2P: **model-8**

- **File:** `g2p_models/model-8`
- **Description:** A Sequitur G2P model trained on Luxembourgish dictionary data (grapheme → IPA). It outputs one pronunciation per word (space-separated phones).
- **Dependency:** Use the **Luxembourgish-adapted** [sequitur-g2p](https://github.com/PeterGilles/sequitur-g2p) repository, which includes the pre-trained `model-8` and supports Luxembourgish characters and conventions. Clone it and use its `g2p.py`:

   ```bash
   git clone https://github.com/PeterGilles/sequitur-g2p.git
   cd sequitur-g2p
   pip install -e .
   ```

   Alternatively, use the `model-8` from this repo’s `g2p_models/` with that installation (see below).

**How to use it to convert OOVs:**

1. Extract your OOV words into a text file, **one word per line** (e.g. `oov_words.txt`).
2. Run Sequitur’s `g2p.py` with the Luxembourgish model (either from the [PeterGilles/sequitur-g2p](https://github.com/PeterGilles/sequitur-g2p) clone or this repo’s `g2p_models/model-8`):

   ```bash
   # PATH_TO_REPO = path to this (MFA Luxembourgish) repo
   g2p.py --model PATH_TO_REPO/g2p_models/model-8 --apply oov_words.txt > oov_pronunciations.txt
   ```

   If `g2p.py` is not on your PATH, use the full path to the script inside the sequitur-g2p clone:

   ```bash
   python /path/to/sequitur-g2p/g2p.py --model PATH_TO_REPO/g2p_models/model-8 --apply oov_words.txt > oov_pronunciations.txt
   ```

3. The output is **word TAB space-separated IPA** (one line per word). You can then merge these lines into your MFA dictionary (word + IPA only; no probability columns for new entries).

**Integration in a script:** Read `mfa find_oovs` output (e.g. `oov_counts_*.txt`), extract the word column into a one-word-per-line file, run the command above, then append the resulting lines to your dictionary or merge them into a dedicated OOV dictionary file.

### 2. MFA G2P: **lb_g2p.zip**

- **File:** `g2p_models/lb_g2p.zip`
- **Description:** The Montreal Forced Aligner’s pretrained Luxembourgish G2P model. It can output one or several pronunciations per word and writes in a format suitable for MFA dictionaries.

**How to use it to convert OOVs:**

1. **Install the G2P model** into MFA (one-time):

   ```bash
   mfa g2p add lb_g2p PATH_TO_REPO/g2p_models/lb_g2p.zip
   ```

2. Prepare an OOV word list, **one word per line** (e.g. `oov_words.txt`).

3. Run MFA’s G2P:

   ```bash
   mfa g2p oov_words.txt lb_g2p oov_pronunciations.txt --num_pronunciations 1
   ```

   Output: `oov_pronunciations.txt` in MFA dictionary format (word TAB IPA). Merge these lines into your main dictionary.

**Integration in a script:** After `mfa find_oovs`, take the list of OOV words (one per line), run `mfa g2p ... lb_g2p ...` as above, then merge the generated file with your base dictionary (e.g. concatenate, or use a small script to avoid duplicate words).

### Choosing between the two

- **Sequitur (model-8):** No MFA dependency for the G2P step; use the [Luxembourgish sequitur-g2p](https://github.com/PeterGilles/sequitur-g2p) fork for full Luxembourgish support. Useful in batch scripts that only need word → IPA.
- **MFA (lb_g2p.zip):** Fits naturally if you already use MFA for alignment and dictionary training; same toolchain and phone set as the rest of the pipeline.

You can also run both on the same OOV list and compare or merge pronunciations (e.g. prefer one model for certain word types or use MFA output as fallback when Sequitur fails).

---

## Practical tips

- **Start small** — Run OOV detection on a subset of the corpus to see how many OOVs you have and how many are numbers, names, or rare words.
- **Prioritize by frequency** — Add pronunciations first for the most frequent OOVs; they have the largest impact on alignment quality.
- **Re-run OOV after expansion** — After adding new words, run `mfa find_oovs` again to see if anything is still missing (e.g. after number conversion).
- **Segmentation for long files** — If you have long recordings with many utterances, segment first so that alignment is more stable and respects utterance boundaries.
- **Conda environment** — Keep MFA, G2P, and any scripts in a dedicated conda environment (e.g. `aligner`) so that dependencies do not conflict.

---

## References

- [Montreal Forced Aligner — User guide](https://montreal-forced-aligner.readthedocs.io/en/latest/)
- [MFA — Finding OOVs](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/workflows/finding_oovs.html)
- [MFA — Training dictionary](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/workflows/training_dictionary.html)
- [MFA — Creating segments](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/corpus_creation/create_segments.html)
- [Luxembourgish num2words](https://github.com/PeterGilles/num2words) — Number, date, time, and currency conversion to Luxembourgish.
- [Luxembourgish Sequitur G2P](https://github.com/PeterGilles/sequitur-g2p) — Grapheme-to-phoneme conversion for Luxembourgish.
