# bach-gen

Evaluating and generating Bach chorales with a small transformer.

## Setup

**Dataset:** Download `jsb-chorales-16th.pkl` from [czhuang/JSB-Chorales-dataset](https://github.com/czhuang/JSB-Chorales-dataset) and place it in `metric-study/`.

**Dependencies:**
```
pip install torch numpy pandas matplotlib pretty_midi muspy music21 scipy
```

**Run order:** `JSB-training.ipynb` → `JSB-evaluation.ipynb`

Training takes ~15 minutes on a 4GB GPU. The evaluation notebook runs on CPU.

## Metric Validity Study

Standard metrics for symbolic music generation — pitch-class entropy, scale consistency, polyphony, pitch range — are widely used to evaluate generative models. But do they actually measure musical quality?

To find out, a small decoder-only transformer (~1.85M parameters) is trained on the [JSB Chorales dataset](https://github.com/czhuang/JSB-Chorales-dataset) and three test sets are compared:

| Set | Description |
|---|---|
| **Real Bach** | 77 test chorales withheld from training |
| **Shuffled** | The same 77 chorales with bars randomly reordered — locally identical to Bach, structurally destroyed |
| **Generated** | 77 chorales sampled from the trained model |

### Key finding

Distributional metrics cannot distinguish shuffled Bach from real Bach (*p* ≈ 1 for all four). Counterpoint-based metrics introduced in this study — voice crossing, parallel fifths, cadence quality, boundary-interval analysis — separate the sets with effect sizes up to *r* = 0.85. Each metric family is blind to failures outside its own domain, making single-metric evaluation unreliable.

### Results summary

| Metric | Real Bach | Shuffled | Generated |
|---|---|---|---|
| Pitch class entropy | 3.026 | 3.021 | 2.985 |
| Scale consistency | 0.930 | 0.930 | 0.903 |
| Voice crossing rate | 0.021 | 0.021 | 0.142 |
| Parallel fifths rate | 0.0004 | 0.0026 | 0.0059 |
| Proper cadence | 97.4% | 80.5% | 22.1% |

Full results, statistical tests (Mann–Whitney U with Holm–Bonferroni correction, Fisher's exact, rank-biserial effect sizes), and interpretation are in the report.

This work was completed as Assignment 3 for the course Generative Artificial Intelligence (IM1412) at the Open Universiteit and received a grade of **9.0/10**.
