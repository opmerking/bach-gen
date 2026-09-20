# bach-gen

This project aims to investigate the validity of evaluation metrics in generative music literature. To do so, a small decoder-only transformer (~1.85M parameters) is trained on Johann Sebastian Bach chorales and evaluated on commonly used MusPy-based distributional metrics, as well as rule-based metrics predicated on conventions in historical counterpoint, which are introduced for this study. These metrics are tested on real Bach chorales, a set of shuffled Bach chorales that is created to test the metrics on their ability to distinguish sequential continuity, and a set of chorales generated with a small transformer trained on real Bach. Finally, an optimized version of the model was created.

*This work was completed as Assignment 3 for the course Generative Artificial Intelligence at the Open University of the Netherlands, and received a grade of **9.0/10**.*

## Setup

**Dataset:** Download `jsb-chorales-16th.pkl` from [czhuang/JSB-Chorales-dataset](https://github.com/czhuang/JSB-Chorales-dataset) and place it in the repo root.

**Dependencies:**
```
pip install torch numpy pandas matplotlib pretty_midi muspy music21 scipy
```

**Run order:** `1_training.ipynb` → `2_evaluation.ipynb`

Training takes ~5 minutes on a 4GB GPU. The evaluation notebook requires `gen_chorales.json` and `Bach_model.pt`, which are included so the evaluation can be run without training first.

Training the optimized model can be done with `3_optimization.ipynb`, and takes about an hour and a half on a 4GB GPU. The optimized model can be evaluated with `2_evaluation.ipynb`, with the hyperparameters changed to match the new training configuration.

## Metric Validity Study

The chosen distributional metrics are widely used in symbolic music generation: pitch-class entropy, scale consistency, polyphony, pitch range.

The rule-based metrics introduced for this study are:
- **Voice crossing**: In chorales, the pitch of each voice should ideally be hierarchical at each timestep. The soprano should always be higher than the alto, etc. in accordance with SATB (soprano > alto > tenor > bass). This metric measures how often the voice registers are crossed.
- **Parallel fifths**: When two voices remain a perfect fifth (7 semitones) or an octave/unison apart during consecutive distinct notes, they lose their independent melodies. Bach avoided this meticulously. This metric measures how often this occurs.
- **Proper cadence**: In Bach chorales, the final chord is often a root-position major or minor triad, with the bass on the chord root, which gives a sense of resolution to the piece. This metric counts in how many chorales of a set this is the case. 
- **Segment-boundary interval analysis**: Because in the set of shuffled chorales the connections between segments are broken, the structural cohesiveness should no longer be intact. To test for musical continuity, an analysis is performed that quantifies the jumps in pitch at segment boundaries for the soprano voice.

The model is also tested on model likelihood.

Three test sets are compared:

| Set | Description |
|---|---|
| **Real Bach** | 77 test chorales withheld from training |
| **Shuffled** | The same 77 chorales with bars randomly reordered — locally identical to Bach, structurally destroyed |
| **Generated** | 77 chorales sampled from the trained model |

### Results

Distributional metrics were found to be unable to distinguish real Bach from the shuffled set (no statistically significant differences for any of the four), and to have a relatively small effect size when comparing real Bach to the generated set. Rule-based metrics, by contrast, show strong discriminative power with effect sizes up to *r* = 0.85. 

| Metric | Real Bach | Shuffled | Generated |
|---|---|---|---|
| Pitch class entropy | 3.026 | 3.021 | 2.985 |
| Scale consistency | 0.930 | 0.930 | 0.903 |
| Pitch range | 34.351 | 34.351 | 32.610 |
| Polyphony | 3.923 | 3.923 | 3.817 |
| Voice crossing rate | 0.021 | 0.021 | 0.142 |
| Parallel fifths rate | 0.0004 | 0.0026 | 0.0059 |
| Proper cadence | 97.4% | 80.5% | 22.1% |

![Box plots of metric distributions across the three systems](figures/metric_comparison_boxplots.png)

### Statistical tests

Effect sizes reported as rank-biserial correlation |*r*|. Bold = significant after Holm–Bonferroni correction.

| Metric | Real vs. Shuffled | | Real vs. Generated | |
|---|---|---|---|---|
| | *p* | \|*r*\| | *p* | \|*r*\| |
| Pitch class entropy | 0.817 | 0.02 | 0.249 | 0.11 |
| Scale consistency | 0.868 | 0.02 | **0.002** | **0.29** |
| Polyphony | 1.000 | 0.00 | **< 0.001** | **0.40** |
| Pitch range | 1.000 | 0.00 | **0.004** | **0.27** |
| Voice crossing | 1.000 | 0.00 | **< 0.001** | **0.49** |
| Parallel fifths | **< 0.001** | **0.85** | **< 0.001** | **0.85** |
| Boundary interval | **< 0.001** | **0.84** | 0.355 | 0.09 |
| Cadence (Fisher's) | **0.001** | – | **< 0.001** | – |

Full results, interpretation, and discussion are in the [report](Report.pdf).

## Model optimization

After completing the assignment, the model was optimized with the following changes:
- Reworked end-of-sequence (EOS) and padding handling.
- Increased the training duration, adjusted regularisation (dropout and weight decay), and added a learning-rate scheduler option.
- Context window increased from 4 to 16 bars. Stride changed from 50% to 25%.
- Random transpositions for extra training data.
- Added optional auxiliary penalties for voice crossings and fifth/octave interval violations.
- Increased the evaluation sample to 200 generated chorales per model.

These changes were implemented in the third notebook `3_optimization.ipynb`. Two models were trained with this notebook: a baseline and a version with auxiliary penalties for voice crossings and fifth/octave interval violations. Both configurations were run for 400 epochs.

| Metric | Real Bach | Baseline | Auxiliary loss |
|---|---:|---:|---:|
| Voice-crossing rate ↓ | 0.0211 | 0.0533 | 0.0337 |
| Parallel fifths/octaves rate ↓ | 0.0004 | 0.0061 | 0.0059 |
| Cadence quality ↑ | 97.4% | 50.0% | 39.5% |
| Best validation cross-entropy ↓ | — | 0.4371 | 0.4383 |

Both updated models achieved substantially lower voice-crossing rates and higher cadence scores than the original model. Under the same 400-epoch training budget, auxiliary loss reduced voice crossings by 36.8% compared with the baseline, while fifths/octaves and validation cross-entropy changed little. Cadence quality was lower, indicating that the targeted improvement did not extend to cadence quality or validation cross-entropy. Distributional metrics were also very similar between the baseline and auxiliary-loss models. These results reflect one training run per configuration.
