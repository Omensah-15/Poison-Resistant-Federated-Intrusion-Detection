# Video script, Advanced track (target 2:50)

**0:00 - 0:20 Problem.** Five banks train together without sharing data, but what if one bank is malicious? Show the cover image.

**0:20 - 1:00 The attack.** One bank flips its labels and inflates its update 15x. Show the red curve: naive FedAvg oscillates between useless states and ends at F1 0.004. The provided coordinate median recovers most of it, at 0.984.

**1:00 - 1:55 The defense.** Three stages before averaging. One: exclude any update whose size is more than 3x the median; this exclusion is final. Two: among the rest, compare each update's direction with the median direction and exclude outliers, always keeping at least three clients. Three: clip survivors to the honest scale. Show the detection map: the attacker is excluded in every round.

**1:55 - 2:30 Results.** Held-out F1 0.9871 under attack versus 0.9876 with no attack. Detection: 75 of 75 rounds caught, 5% false alarms on honest clients. Stress tests: two attackers, label flipping without amplification, sign flipping; all stay near 0.99.

**2:30 - 2:50 Honest limits and verification.** The margin over the median is small, stress tests are single seed, and adaptive attackers were not tested. The exported model was reloaded from disk and re-scored at 0.9895 F1. Repo link, thanks.
