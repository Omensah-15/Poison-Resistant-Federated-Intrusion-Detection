# Poison-Resistant Federated Intrusion Detection

![Cover: poison-resistant aggregation versus naive FedAvg and coordinate median under attack](figures/cover_image.png)

CAIRLab hackathon, Day 2, **Advanced track** (detect or resist a malicious bank). Five simulated banks train a shared NSL-KDD intrusion detector without pooling data; one bank flips its labels and inflates its update 15x.

## Results (F1 on the organizers' 15,000-row held-out set, mean of 3 seeds)

| Method | Held-out F1 |
|---|---|
| Naive FedAvg under attack (before) | 0.0039 |
| Coordinate median under attack (provided baseline) | 0.9841 |
| Our defense under attack (after) | 0.9871 |
| Our defense, no attack (clean reference) | 0.9876 |

Exported model (trained through the attack, seed 0): held-out F1 0.9895. The attacker was excluded in 75 of 75 rounds. Full analysis, stress tests and limitations are in [`WRITEUP.md`](WRITEUP.md).

### F1 over communication rounds under attack

Mean of 3 seeds, band = 1 standard deviation. Left: full range. Right: zoom on the defended runs.

![Advanced track: held-out F1 over rounds for naive FedAvg, coordinate median, our defense under attack, and our defense with no attack](figures/advanced_f1_over_rounds.png)

### Which clients the defense excludes

Dark cells mark clients excluded in that round (seed 0). Client 1 is the attacker.

![Detection map: clients excluded by the defense per round](figures/detection_map.png)

### Confusion matrix of the exported model (held-out set)

<img src="figures/confusion_matrix.png" alt="Held-out confusion matrix of the exported model" width="380">

## Contents

| File | Purpose |
|---|---|
| `federated_ids_advanced.ipynb` | Final notebook with all outputs visible (runs top to bottom, no errors) |
| `model_scripted.pt`, `submission.json` | Exported TorchScript model and metrics file for organizer verification |
| `predictions.csv` | Predictions in `sample_submission.csv` format (`Id`, `Expected`), same row order as `test_public.csv` |
| `results.json` | Every number reported in the write-up |
| `figures/` | Cover image, F1-over-rounds chart, detection map, confusion matrix |
| `WRITEUP.md`, `VIDEO_SCRIPT.md` | Kaggle writeup text and a 3-minute video script |

## Reproduce

```
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace federated_ids_advanced.ipynb
```

Needs internet access to download NSL-KDD from GitHub. About 10 minutes on one CPU thread; no GPU. The data split uses the starter's fixed seed (42) and federated runs use seeds 0, 1 and 2. Results can differ in the last digit across hardware and library versions.

## Use the exported model

```python
import torch
model = torch.jit.load("model_scripted.pt").eval()
# x: float32 tensor of shape (n, 41), standardized exactly as in Section 1 of the notebook
prediction = (torch.sigmoid(model(x)) > 0.5).int()   # 1 = attack, 0 = normal
```

## Method

`RobustSkewAwareAgg`: (1) exclude clients whose update norm exceeds 3x the median, (2) exclude clients whose direction disagrees with the median direction of the rest, (3) clip survivors to 1.5x the median survivor norm; then square-root-weighted averaging with server momentum. Each round's exclusions are logged, so the defense also detects the attacker.
