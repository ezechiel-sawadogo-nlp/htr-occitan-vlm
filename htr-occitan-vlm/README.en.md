# HTR for medieval Occitan manuscripts: Kraken vs VLM

🇫🇷 [Version française](README.md)

Handwritten text recognition (HTR) on **medieval Occitan** manuscripts: a controlled comparison between a lightweight specialised model (**Kraken**, 4 M parameters) and a general-purpose vision-language model (**Qwen2.5-VL-3B**, 3.8 B parameters), with and without fine-tuning, followed by an export of the transcriptions to **TEI XML**.

## Results

CER (character error rate) on 574 test lines from **3 manuscripts never seen during training**.

| Model | Parameters | Overall CER | additional-21218 | arsenal-6355 | naf-6195 |
|---|---|---|---|---|---|
| Kraken CATMuS Medieval, off the shelf | 4 M | 16.7 % | 10.5 % | 7.7 % | 24.5 % |
| **Kraken CATMuS Medieval, fine-tuned** | 4 M | **7.2 %** | **6.5 %** | **5.0 %** | **8.5 %** |
| Qwen2.5-VL-3B-Instruct, zero-shot | 3.8 B | 38.9 % | 33.8 % | 32.8 % | 44.9 % |
| Qwen2.5-VL-3B + LoRA | 3.8 B | 22.1 % | 22.2 % | 17.8 % | 23.4 % |
| Qwen2.5-VL-3B + LoRA, images ×2 | 3.8 B | *running* | | | |

**Main finding**: with 3,400 training lines, the fine-tuned specialised model is **three times more accurate** than the fine-tuned VLM, with about 900 times fewer parameters, and it runs on CPU.

## Error analysis

### Error types (as a share of reference characters)

| Model | CER | CER without spaces | Substitutions | Deletions | Insertions |
|---|---|---|---|---|---|
| Kraken off the shelf | 16.7 % | 17.2 % | 7.2 % | 7.1 % | 2.4 % |
| Kraken fine-tuned | 7.2 % | 7.1 % | 3.7 % | 2.1 % | 1.4 % |
| Qwen zero-shot | 38.9 % | 42.1 % | 22.6 % | 7.8 % | 8.5 % |
| Qwen + LoRA | 22.1 % | 24.1 % | 13.5 % | 5.3 % | 3.4 % |

### Share of characters read correctly, by category

| Category | Count | Kraken fine-tuned | Qwen + LoRA |
|---|---|---|---|
| Letters a-z | 12,693 | 94.4 % | 79.1 % |
| Spaces | 2,615 | 96.9 % | 94.0 % |
| Punctuation / digits | 200 | 88.5 % | 88.5 % |
| Abbreviations and special signs | 204 | 52.9 % | 45.6 % |

### What the numbers show

1. **Line segmentation matters as much as the model.** Cropping each line by its *bounding box* pulled in neighbouring lines on skewed photographs. Switching to *polygon* cropping brought the off-the-shelf Kraken CER from **40.1 % down to 16.7 %**, without touching the model.
2. **Off-the-shelf Kraken was dropping characters.** Its deletions (7.1 %) matched its substitutions: 35 characters of the Occitan corpus (digits, `k`, abbreviations such as `ł` or `ħ`, superscript letters) were missing from its alphabet. After extending the alphabet and fine-tuning, deletions fall to 2.1 %.
3. **Fine-tuning cuts VLM hallucinations by a factor of 2.5.** Zero-shot, Qwen "modernises" the text: apostrophes, accents, even modern French words that are not on the image ("Dauãt que 9dampnatz" → "D'aumône que la plume de la vaine"). Insertions drop from 8.5 % to 3.4 %.
4. **Spaces are not the main source of error**: removing them does not lower the CER.
5. **The gap between Kraken and Qwen comes from ordinary letters**, not abbreviations: 94.4 % vs 79.1 % of letters a-z read correctly. Abbreviations are hard for both models but account for only 1.3 % of characters.
6. **Some confusions are shared by both models** and stem from the script itself: `l` ↔ `s` (the *long s* looks like an `l`), `u` ↔ `n` ↔ `m` (letters built from the same minims).
7. **One confusion is specific to Qwen: many letters become `i`** (`a → i`, `u → i`, `r → i`, `y → i`). Hypothesis: at native resolution a letter covers roughly one image token (28 × 28 px), so the model sees vertical strokes without telling letter shapes apart. The "images ×2" experiment tests this hypothesis.

## Data

**HTRogène – Occitan** corpus (Biblissima+): 19 manuscripts, 100 pages, 4,293 transcribed lines (132 empty lines removed), annotated in ALTO following the CATMuS/Segmonto conventions.

> Wiedner, M., Couffignal, G., Chagué, A., Clérice, T. *HTRogène, Occitan corpus of ground-truth for Handwritten Text Recognition and Layout Segmentation* [Data set]. https://github.com/HTRogene/occitan — CC-BY 4.0 licence.

**Split by manuscript**, to measure generalisation to unseen hands:

| Split | Manuscripts | Lines |
|---|---|---|
| Train | 13 manuscripts | 3,392 |
| Dev | additional-10323, paris-bnf-fr-13504, london-bl-harley-3041 | 327 |
| Test | paris-bnf-naf-6195, additional-21218, paris-bnf-arsenal-6355 | 574 |

Manuscript `paris-bnf-fr-13509` is also part of CATMuS Medieval, on which the base Kraken model was trained: it is kept in the training split to avoid test contamination.

## Method

- **Line extraction**: each line is cropped from its ALTO polygon, and everything outside the polygon is masked in white.
- **Evaluation**: corpus-level CER on the test set (total errors / total characters, with `jiwer`), after **NFD** Unicode normalisation on both sides, as used to train the CATMuS model.
- **Kraken**: [CATMuS Medieval 1.5.0](https://doi.org/10.5281/zenodo.12743230) model (Kraken 7.1.1). Fine-tuned with alphabet extension (`--resize union`), learning rate 1e-4, batch size 16, image augmentation; best checkpoint selected on dev (epoch 9, dev accuracy 0.886).
- **Qwen2.5-VL-3B-Instruct**: LoRA (r = 16, α = 32) on the attention and MLP layers of the **language model only** (0.79 % of parameters trained), 2 epochs, learning rate 2e-4 (cosine), effective batch of 16 lines, fp16; only the answer is learned (prompt and image tokens masked in the labels). Deterministic decoding (`do_sample=False`).
- **Hardware**: NVIDIA T4 GPU (Google Colab, Kaggle).

## TEI export

The fine-tuned Kraken transcriptions are exported to TEI, one file per test manuscript (`tei/`), **valid against `tei_all`**:

- `<teiHeader>`: source (HTRogène corpus, licence), model and CER measured on the manuscript;
- `<facsimile>`: one `<surface>` per page, one `<zone>` per line with its polygon;
- `<body>`: diplomatic transcription, one `<lb facs="#…"/>` per manuscript line;
- Segmonto zones converted where annotated: `NumberingZone` → `fw[@type='folNum']`, `RunningTitleZone` → `fw[@type='header']`, `MarginTextZone` → `note[@place='margin']`, `HeadingLine` → `seg[@type='heading']`.

## Limitations

- **Small test set**: 574 lines and 3 manuscripts; per-manuscript CERs rest on 93 to 337 lines.
- **Kraken training stopped at epoch 9**, while dev accuracy was still improving slightly.
- **Lines not dewarped**: Kraken was fine-tuned on pre-cropped line images, whereas the base model was trained on lines straightened along their baseline.
- **Qwen's vision encoder was frozen**: only the language model was adapted.
- **Zone annotation varies across manuscripts**: paratext (ex-libris, titles) is not always distinguished from the main text in the TEI.

## Next steps

- LoRA on Qwen's vision encoder;
- fine-tuning Kraken directly on ALTO (dewarped lines);
- pre-training on CATMuS Medieval (Old French, Latin) before Occitan;
- hybrid approach: Kraken transcription corrected by a language model.

## Repository layout

```
notebooks/   data preparation, Kraken, Qwen, error analysis and TEI export
results/     test-set predictions (CSV) and error analysis (JSON)
tei/         TEI transcriptions of the test manuscripts
```

Images and model weights are not versioned: the notebooks regenerate them from the HTRogène repository and public models.

## Licence

- **Code** (notebooks): MIT licence, see [LICENSE](LICENSE).
- **Transcriptions, predictions and TEI files**: derived from the HTRogène corpus, released under **CC-BY 4.0**; any reuse must cite the HTRogène corpus (see *Data*).

## Author

Ezéchiel Sawadogo — Master's in Language and Computer Science (NLP), Sorbonne Université · [GitHub](https://github.com/ezechiel-sawadogo-nlp) · [LinkedIn](https://www.linkedin.com/in/ezechiel-sawadogo/)
