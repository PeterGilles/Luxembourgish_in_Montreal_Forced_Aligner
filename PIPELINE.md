# Advanced alignment pipeline: G2P, OOV handling, and dictionary expansion

This document describes a **more complex workflow** for aligning Luxembourgish speech when your corpus contains **out-of-vocabulary (OOV) words**—i.e. words that are not in the pronunciation dictionary. It is aimed at linguists and computational linguists who want to understand or adapt such a pipeline.

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
2. **Number conversion** — If your transcripts contain digits (e.g. *12*, *2024*), convert them to words (e.g. *zwielef*, *zweedausendvéieranzwanzeg*) using a number-to-words tool (e.g. for Luxembourgish). Then treat those word forms as normal vocabulary (add them to the dictionary if they are still OOV).
3. **G2P for OOVs** — For each OOV word, generate a candidate pronunciation (IPA) using one of the **G2P models** included in this repo: **model-8** (Sequitur) or **lb_g2p.zip** (MFA). See the section *G2P models: integrating model-8 and lb_g2p.zip* below.
4. **Manual review** — Inspect the OOV list with suggested IPA. Correct mispronunciations, dialectal variants, and proper nouns. Save the result as word–IPA pairs (or word–replacement–IPA if you normalize spelling).
5. **Dictionary expansion** — Merge the base dictionary with the new OOV entries (word + IPA). Optionally run **`mfa train_dictionary`** on a corpus to estimate **pronunciation probabilities** for multiple variants (e.g. *a* → [aː] vs [ɑ]).
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

### Dictionary training (pronunciation probabilities)

```bash
mfa train_dictionary CORPUS_DIR DICTIONARY_PATH ACOUSTIC_MODEL OUTPUT_DIR [--num_jobs N] [--clean] [--overwrite]
```

- **DICTIONARY_PATH** — Base dictionary (word + IPA, possibly with multiple pronunciations per word).
- **ACOUSTIC_MODEL** — The Luxembourgish acoustic model (e.g. `models/lb_acoustic_model.zip`).
- **OUTPUT_DIR** — Directory where MFA writes the trained dictionary (with probabilities). You can then use this trained dictionary in `mfa align`.

See the [MFA documentation on training dictionaries](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/workflows/training_dictionary.html).

### Segmentation (optional)

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
- **Dependency:** You need [sequitur-g2p](https://github.com/sequitur-g2p/sequitur-g2p) installed (e.g. `pip install sequitur-g2p` or clone the repo and use its `g2p.py`).

**How to use it to convert OOVs:**

1. Extract your OOV words into a text file, **one word per line** (e.g. `oov_words.txt`).
2. Run Sequitur’s `g2p.py` with this repo’s model:

   ```bash
   # From repo root; PATH_TO_REPO = path to this repo
   g2p.py --model PATH_TO_REPO/g2p_models/model-8 --apply oov_words.txt > oov_pronunciations.txt
   ```

   If `g2p.py` is not on your PATH, use the full path to the script inside a sequitur-g2p clone:

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

- **Sequitur (model-8):** No MFA dependency for the G2P step; works with any environment where sequitur-g2p is installed. Useful in batch scripts that only need word → IPA.
- **MFA (lb_g2p.zip):** Fits naturally if you already use MFA for alignment and dictionary training; same toolchain and phone set as the rest of the pipeline.

You can also run both on the same OOV list and compare or merge pronunciations (e.g. prefer one model for certain word types or use MFA output as fallback when Sequitur fails).

---

## G2P and OOV list format

- **G2P** takes a word (graphemes) and returns one or more pronunciation(s) in IPA (phone sequence). Training a G2P model requires a pronunciation dictionary (word–IPA pairs); Sequitur and MFA’s G2P trainer are common options.
- In our pipeline, the **OOV list** is often stored as:  
  `word TAB replacement TAB IPA TAB frequency`  
  so that you can correct spelling (replacement) and IPA in one place. The “replacement” is the form that gets added to the dictionary; “word” is the original form in the corpus (for reference).

After editing, you merge only the **replacement + IPA** (or word + IPA if no replacement) into the main dictionary.

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
