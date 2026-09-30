# Heavenz: Poison-Resistant Federated Intrusion Detection

**Subtitle:** Excluding a malicious bank in every round: F1 0.004 to 0.987 under poisoning

**Track submitted:** Advanced

## Headline result (Advanced track)

F1 on the organizers' 15,000-row held-out set, mean of 3 training seeds. Scenario: client 1 flips its labels and inflates its update 15x (the starter's attack). Same 128-64 MLP, 25 rounds, 2 local epochs, learning rate 0.1 and client split for every row.

| Method | Held-out F1 | KDDTest+ F1 |
|---|---|---|
| Naive FedAvg under attack (before) | 0.0039 +/- 0.0054 | 0.0139 |
| Coordinate median under attack (provided baseline) | 0.9841 +/- 0.0007 | 0.7488 |
| **Our defense under attack (after)** | **0.9871** +/- 0.0020 | 0.7868 |
| Our defense, no attack (clean reference) | 0.9876 +/- 0.0030 | 0.7837 |

The defense recovers essentially all of the clean performance. The exported model (`model_scripted.pt`, `submission.json` in the repo root, trained through the attack, seed 0) scores 0.9895 F1 (precision 0.9883, recall 0.9907) on the held-out set. Seed-0 numbers are single runs; the 3-seed means above are the fair estimate.

## Threat model

Fewer than half of the banks are malicious and may send arbitrary updates. The starter's poisoning attack makes naive FedAvg oscillate between useless states (see the round chart) and end near zero F1.

## Defense

Each round, before the (skew-aware) averaging, updates pass three stages:

1. **Magnitude screen.** Exclude any client whose update norm exceeds 3x the median norm. This exclusion is final. An earlier version let a fallback rule override it, kept two 50x attackers, and dropped to 0.15 F1 in one round, which is why the order matters.
2. **Direction screen.** Among the rest, take the coordinate-wise median of unit-length updates as a reference. Exclude clients pointing against it or far below the robust cosine distribution (median minus 3 scaled MADs and a 0.25 margin). At least three clients always survive. This catches attackers who match the honest magnitude.
3. **Clipping.** Survivors are clipped to 1.5x the median survivor norm.

The aggregation that follows uses square-root size weights, server momentum and logit-adjusted local training (see the Intermediate write-up). Every round's exclusion vector is logged, so the defense also acts as a detector.

## Detection results (3 seeds x 25 rounds)

| Setting | Malicious excluded | Missed | Honest wrongly excluded |
|---|---|---|---|
| Headline attack (1 of 5 clients) | 75 of 75 rounds | 0 | 15 of 300 client-rounds (5%) |
| No attack | not applicable | not applicable | 33 of 375 client-rounds (8.8%) |

False alarms are cheap because at least three clients always survive: clean F1 (0.9876) and attacked F1 (0.9871) are statistically indistinguishable.

## Stress tests (single seed, held-out F1)

| Scenario | Naive FedAvg | Coord. median | Ours | Detection |
|---|---|---|---|---|
| 2 attackers, label flip + 50x scale | 0.6356 | 0.9873 | 0.9883 | 50 of 50 caught, 0 false alarms |
| 2 attackers, label flip only | 0.7309 | 0.9868 | 0.9881 | 45 of 50 caught, 0 false alarms |
| 1 attacker, sign flip x10 | 0.0000 | 0.9803 | 0.9895 | 25 of 25 caught, 6 false alarms |

## Evaluation

The reserved 15,000 rows are identical to `test_public.csv`, so we recovered their labels and score models the way the organizers will. KDDTest+ (the file the starter notebook scores on) is reported as a secondary check; F1 there is lower for every method because of NSL-KDD's known train/test shift.

## Limitations

The margin over the coordinate median is small on this dataset (about +0.003 for the headline attack; larger only for sign flipping). Stress tests are single-seed. Two colluding label-flippers were missed in 5 of 50 rounds. We did not test adaptive attackers that match honest norm and direction, or attackers above 40% of clients; the defense assumes an honest majority. Experiments use 5 simulated clients on CPU.

## Reproduce

Run `federated_ids_advanced.ipynb` top to bottom (about 12 minutes on CPU). It writes `model_scripted.pt` and `submission.json` along with all figures. See `README.md`.
