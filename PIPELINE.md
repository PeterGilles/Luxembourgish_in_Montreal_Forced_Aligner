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
│  Aligned        │     │  MFA align       │     │  Expanded       │
│  TextGrids      │ ◄── │  (optional:      │ ◄── │  dictionary     │
└─────────────────┘     │   segment first) │     │  (base + OOV)   │
                         └────────┬─────────┘     └────────┬────────┘
                                  │                        │
                                  │                        │
                         ┌────────▼─────────┐     ┌────────▼────────┐
                         │  Optional:       │     │  G2P + manual    │
                         │  train_dictionary│     │  review for OOV │
                         └─────────────────┘     └─────────────────┘
```

Steps in words:

1. **OOV detection** — Run `mfa find_oovs` on your corpus with the current dictionary. MFA reports which words in the transcripts are not in the dictionary (and often how often they occur).
2. **Number conversion** — If your transcripts contain digits (e.g. *12*, *2024*), convert them to words (e.g. *zwielef*, *zweedausendvéieranzwanzeg*) using a number-to-words tool (e.g. for Luxembourgish). Then treat those word forms as normal vocabulary (add them to the dictionary if they are still OOV).
3. **G2P for OOVs** — For each OOV word, generate a candidate pronunciation (IPA) using a **grapheme-to-phoneme (G2P)** model. Options we use:
   - **Sequitur G2P** (trained on Luxembourgish dictionary data), or
   - **MFA’s Luxembourgish G2P model** (`lb_g2p`), if available.
4. **Manual review** — Inspect the OOV list with suggested IPA. Correct mispronunciations, dialectal variants, and proper nouns. Save the result as word–IPA pairs (or word–replacement–IPA if you normalize spelling).
5. **Dictionary expansion** — Merge the base dictionary with the new OOV entries (word + IPA). Optionally run **`mfa train_dictionary`** on a corpus to estimate **pronunciation probabilities** for multiple variants (e.g. *a* → [aː] vs [ɑ]).
6. **Segmentation (optional)** — For long files, run **`mfa segment`** first to split the audio into segments and create segment-tier TextGrids. Then align using a corpus that contains only **wav + segment TextGrid** (no `.txt`), so that MFA respects segment boundaries.
7. **Alignment** — Run **`mfa align`** with the expanded (and optionally trained) dictionary and the Luxembourgish acoustic model. Output: TextGrids with word and phone tiers.
8. **Merge (optional)** — If you segmented first, merge the segment-tier TextGrids with the align-tier TextGrids so that you have one file with segments, words, and phones.

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
