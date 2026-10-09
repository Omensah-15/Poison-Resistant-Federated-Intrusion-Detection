# Poison-Resistant Federated Intrusion Detection

**Subtitle:** Excluding a malicious bank in every round: F1 0.004 to 0.987 under poisoning

**Track submitted:** Advanced

![Cover: poison-resistant aggregation versus naive FedAvg and coordinate median under attack](figures/cover_image.png)a# Poison-Resistant Federated Intrusion Detection

**Subtitle:** One cheating bank can ruin a shared security model. We made the model immune and caught the cheater (F1 0.004 to 0.987).

**Track submitted:** Advanced

![Cover: poison-resistant aggregation versus naive FedAvg and coordinate median under attack](figures/cover_image.png)

## The problem

Five banks want a shared model that tells normal network traffic from attacks. They cannot share their data, so each one trains on its own data and sends only its update to a central server, which averages the updates. This is federated learning.

The server has to trust every update, and that is the weak spot. In the starter's attack, one bank:

- **flips its labels**, so it teaches the model that attacks are normal and normal traffic is an attack
- **inflates its update 15x**, so its bad update drowns out the four honest ones

With plain averaging, the model bounces between useless states and ends near zero F1. One bad bank out of five is enough to break the whole system.

## What we did

We replaced plain averaging with a defense that does not blindly trust everyone. Each round, before averaging, the server checks every update in three steps:

1. **Size check.** If an update is more than 3x bigger than the typical (median) one, exclude it. This decision is final. An earlier version let a fallback rule overrule it, kept two 50x attackers, and dropped to 0.15 F1 in one round, which is why the order matters.
2. **Direction check.** Among the rest, find the typical direction of the updates. Exclude any bank pulling against it or far out of line with the others. This catches a cheater who keeps their update the same size as an honest one. At least three banks always survive.
3. **Turn down the volume.** Cap the survivors so none is more than 1.5x the typical survivor size.

The server then averages what is left, using square-root size weights, server momentum and logit-adjusted local training (details in the Intermediate write-up).

Every round's exclusions are logged, so the defense also tells you who the attacker is.

## Results

F1 on the organizers' 15,000-row held-out set, mean of 3 training seeds. F1 is a score from 0 to 1, where 1 is perfect. Scenario: client 1 flips its labels and inflates its update 15x. Every row uses the same model (a 128-64 MLP), 25 rounds, 2 local epochs, learning rate 0.1 and the same split of data across banks.

| Method | Held-out F1 | KDDTest+ F1 |
|---|---|---|
| Plain averaging (FedAvg) under attack | 0.0039 +/- 0.0054 | 0.0139 |
| Coordinate median under attack (provided baseline) | 0.9841 +/- 0.0007 | 0.7488 |
| **Our defense under attack** | **0.9871** +/- 0.0020 | 0.7868 |
| Our defense, no attack (clean reference) | 0.9876 +/- 0.0030 | 0.7837 |

**What this means:** with a cheating bank in the room, our model scores 0.9871. With no cheater at all, it scores 0.9876. The two are statistically the same, so the attack has essentially no effect on us.

The exported model (`model_scripted.pt` and `submission.json`, trained through the attack, seed 0) scores 0.9895 F1 (precision 0.9883, recall 0.9907) on the held-out set. Seed-0 numbers come from a single run, so the 3-seed means in the table are the fairer estimate.

![Held-out F1 over rounds for naive FedAvg, coordinate median, our defense under attack, and our defense with no attack](figures/advanced_f1_over_rounds.png)

*Held-out F1 over communication rounds under the attack, mean of 3 seeds (band = 1 standard deviation). Left: full range. Right: zoom on the defended runs.*

## Did we catch the cheater?

Yes. Over 3 seeds and 25 rounds:

| Setting | Cheater excluded | Cheater missed | Honest banks wrongly excluded |
|---|---|---|---|
| Attack (1 of 5 banks cheating) | 75 of 75 rounds | 0 | 15 of 300 bank-rounds (5%) |
| No attack | not applicable | not applicable | 33 of 375 bank-rounds (8.8%) |

Sometimes an honest bank is excluded by mistake, but this costs almost nothing because at least three banks always survive. That is why the clean score (0.9876) and the attacked score (0.9871) are so close.

![Detection map: clients excluded by the defense per round](figures/detection_map.png)

*Dark cells mark banks excluded in that round (seed 0). Client 1 is the cheater.*

<img src="figures/confusion_matrix.png" alt="Held-out confusion matrix of the exported model" width="380">

*Confusion matrix of the exported model (trained through the attack) on the 15,000-row held-out set.*

## Harder tests

We also tried other attacks. Each is a single run, shown as held-out F1.

| Scenario | Plain averaging | Coord. median | Ours | Cheaters caught |
|---|---|---|---|---|
| 2 cheaters, label flip + 50x size | 0.6356 | 0.9873 | 0.9883 | 50 of 50, 0 false alarms |
| 2 cheaters, label flip only | 0.7309 | 0.9868 | 0.9881 | 45 of 50, 0 false alarms |
| 1 cheater, sign flip x10 | 0.0000 | 0.9803 | 0.9895 | 25 of 25, 6 false alarms |

Our defense held up in all three. The sign-flip attack is where it beats the coordinate median by the most.

## What we assumed

Fewer than half of the banks are cheating, but a cheater can send anything at all. We assume an honest majority.

## How we scored

The reserved 15,000 rows are identical to `test_public.csv`, so we recovered their labels and scored models the same way the organizers will. We also report KDDTest+ (the file the starter notebook scores on) as a second check. F1 is lower there for every method because of a known train/test shift in NSL-KDD.

## Limitations

- **The edge over the baseline is small.** On this dataset the improvement over the coordinate median is about +0.003 F1 for the main attack. It is bigger only for sign-flipping. The large win is against plain averaging.
- **Some tests are single runs.** The stress tests used one seed each.
- **Two colluding cheaters** who only flipped labels were missed in 5 of 50 rounds.
- **Smarter cheaters were not tested.** We did not try an attacker who copies the size and direction of an honest update, which is the hardest real-world case.
- **Honest majority needed.** We did not test with more than 40% of banks cheating, and the defense is not designed for it.
- **Small scale.** The experiments used 5 simulated banks on a CPU.

## Why it matters

Any group that wants to train a shared model without sharing data faces this risk: hospitals sharing diagnostic models, telecom operators sharing fraud detection, banks sharing intrusion detection, or phones and IoT devices training together. If one participant is compromised, plain averaging fails. A defense like this keeps the shared model working and also shows who tried to break it.

## Reproduce

Run `federated_ids_advanced.ipynb` top to bottom (about 12 minutes on CPU). It writes `model_scripted.pt` and `submission.json` along with all figures. See `README.md`.

## Headline result (Advanced track)

F1 on the organizers' 15,000-row held-out set, mean of 3 training seeds. Scenario: client 1 flips its labels and inflates its update 15x (the starter's attack). Same 128-64 MLP, 25 rounds, 2 local epochs, learning rate 0.1 and client split for every row.

| Method | Held-out F1 | KDDTest+ F1 |
|---|---|---|
| Naive FedAvg under attack (before) | 0.0039 +/- 0.0054 | 0.0139 |
| Coordinate median under attack (provided baseline) | 0.9841 +/- 0.0007 | 0.7488 |
| **Our defense under attack (after)** | **0.9871** +/- 0.0020 | 0.7868 |
| Our defense, no attack (clean reference) | 0.9876 +/- 0.0030 | 0.7837 |

The defense recovers essentially all of the clean performance. The exported model (`model_scripted.pt`, `submission.json` in the repo root, trained through the attack, seed 0) scores 0.9895 F1 (precision 0.9883, recall 0.9907) on the held-out set. Seed-0 numbers are single runs; the 3-seed means above are the fair estimate.

![Advanced track: held-out F1 over rounds for naive FedAvg, coordinate median, our defense under attack, and our defense with no attack](figures/advanced_f1_over_rounds.png)

*Held-out F1 over communication rounds under the headline attack, mean of 3 seeds (band = 1 standard deviation). Left: full range. Right: zoom on the defended runs.*

## Threat model

Fewer than half of the banks are malicious and may send arbitrary updates. The starter's poisoning attack makes naive FedAvg oscillate between useless states (see the round chart) and end near zero F1.

## Defense

Each round, before the (skew-aware) averaging, updates pass three stages:

1. **Magnitude screen.** Exclude any client whose update norm exceeds 3x the median norm. This exclusion is final. An earlier version let a fallback rule override it, kept two 50x attackers, and dropped to 0.15 F1 in one round, which is why the order matters.
2. **Direction screen.** Among the rest, take the coordinate-wise median of unit-length updates as a reference. Exclude clients pointing against it or far below the robust cosine distribution (median minus 3 scaled MADs and a 0.25 margin). At least three clients always survive. This catches attackers who match the honest magnitude.
3. **Clipping.** Survivors are clipped to 1.5x the median survivor norm.

The aggregation that follows uses square-root size weights, server momentum and logit-adjusted local training (see the Intermediate write-up). Every round's exclusion vector is logged, so the defense also acts as a detector.

## Detection results (3 seeds x 25 rounds)

![Detection map: clients excluded by the defense per round](figures/detection_map.png)

*Dark cells mark clients excluded in that round (seed 0). Client 1 is the attacker.*

| Setting | Malicious excluded | Missed | Honest wrongly excluded |
|---|---|---|---|
| Headline attack (1 of 5 clients) | 75 of 75 rounds | 0 | 15 of 300 client-rounds (5%) |
| No attack | not applicable | not applicable | 33 of 375 client-rounds (8.8%) |

False alarms are cheap because at least three clients always survive: clean F1 (0.9876) and attacked F1 (0.9871) are statistically indistinguishable.

<img src="figures/confusion_matrix.png" alt="Held-out confusion matrix of the exported model" width="380">

*Confusion matrix of the exported model (trained through the attack) on the 15,000-row held-out set.*

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
