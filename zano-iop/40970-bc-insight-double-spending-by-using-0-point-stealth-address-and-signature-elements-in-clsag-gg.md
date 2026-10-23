# #40970 \[BC-Insight] Double spending by using 0-point stealth address and signature elements in CLSAG

**Submitted on Mar 8th 2025 at 12:46:32 UTC by @Blockian for** [**IOP | Zano**](https://immunefi.com/audit-competition/iop-zano)

* **Report ID:** #40970
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/zano-iop/tree/main/src/crypto>
* **Impacts:**
  * Direct loss of funds

## Description

After hallucinating some complex CLSAG-GGX verification bypasses including 8-torsion subgroup elements, I think I've found an actual bypass that works and is actually way simpler.

In this analysis I'll assume the ring is of size 1. This bypass also works for other ring sizes and is easily expandable, but I wanted to keep it simple.

As Immunefi's MD rederer doesn't render links nor Math, I suggest copying this report to your favorite MD renderer. I'll also attach a rendered version.

## Overview

The original challenge-response relation that is required when validating a CLSAG-GGX signature is:

$$
c = H\left( r\_gG + c(\alpha\_0 S + \alpha\_1 (A - \tilde{C})), ; r\_gH(S) + c (\alpha\_0 I + \alpha\_1 K\_1), r\_xX + c\alpha\_2 (Q - \tilde{Q}), r\_xH(S) + c\alpha\_2K\_2 \right)
$$

Where:

* $S$ is the stealth address (pulled from the transaction output of a previous transaction)
* $A$ is the amount commitment (multiplied by 8 in the code) (pulled from the transaction output of a previous transaction)
* $\tilde{C}$ is the pseudo-out amount commitment (provided by the attacker, used to mask A)
* $Q$ is the blinded asset ID (multiplied by 8) (pulled from the transaction output of a previous transaction)
* $\tilde{Q}$ is the pseudo-out blinded asset ID (provided by the attacker, used to mask Q)
* $I$ is the key image (provided by the attacker) - this is the component that is marked as spent to prevent double spending
* , $K\_1$, and $K\_2$ are public keys used in the proof (provided by the attacker)
* $\alpha\_0$, $\alpha\_1$, $\alpha\_2$ are aggregation coefficients calculated by hashing the proof elements
* $r\_g$ and $r\_x$ are response scalars provided by the attacker
* $G$ and $X$ are base points
* $H$ is a hash function from s point on the curve to a point on the curve

## Step 1 - outputs creation (previous transaction)

$S$ is attacker controlled when creating an output.\
If $S$ is chosen so that it is a zero point on the curve (there are a large number different representations of the zero point in the representation used by Zano as can be seen in `point_t::is_zero`):

$$
S = 0
$$

This simplifies the verification equation when trying to spend that output to:

$$
c = H\left( r\_gG + c(\alpha\_1 (A - \tilde{C})), ; r\_gH(S) + c (\alpha\_0 I + \alpha\_1 K\_1), r\_xX + c\alpha\_2 (Q - \tilde{Q}), r\_xH(S) + c\alpha\_2K\_2 \right)
$$

## Step 2 - signature verification

### First term

$\tilde{C}$ is attacker controlled when verifying a proof (used to verify that the output is equal to the input, and to mask $A$.\
Setting $\tilde{C} = A$ causes the term $A - \tilde{C}$ to vanish.\
This simplifies the verification equation to:

$$
c = H\left( r\_gG , ; r\_gH(S) + c (\alpha\_0 I + \alpha\_1 K\_1), r\_xX + c\alpha\_2 (Q - \tilde{Q}), r\_xH(S) + c\alpha\_2K\_2 \right)
$$

### Second term

$I$ is attacker controlled (the key image) and is used to verify the double spend and ownership of the output.\
$K\_1$ is attacker controlled (the first public key) and is used to verify ownership and mask.\
Setting both to a 0 point simplifies the equation even further:

$$
c = H\left( r\_gG , ; r\_gH(S), r\_xX + c\alpha\_2 (Q - \tilde{Q}), r\_xH(S) + c\alpha\_2K\_2 \right)
$$

### Third term

$\tilde{Q}$ is attacker controlled and is used to verify the output asset is the same as the input asset, and for masking.\
Setting is to equal to $Q$ simplified the equation further:

$$
c = H\left( r\_gG , ; r\_gH(S), r\_xX, r\_xH(S) + c\alpha\_2K\_2 \right)
$$

### Fourth term

$K\_2$ is attacker controlled and is the second public key, aslo used to verify ownership and mask.\
Setting is to 0 simplifies the equation further to:

$$
c = H\left( r\_gG , ; r\_gH(S), r\_xX, r\_xH(S)\right)
$$

### Verification conclusion

We reached a verification formula that can be proven with any $r$.\
So - by setting `sig.c` to $c = H\left( r\_gG , ; r\_gH(S), r\_xX, r\_xH(S)\right)$ and by setting $\tilde{Q}=Q$, $\tilde{C}=A$, $K\_1=0$, $K\_2=0$, $I=0$, and chosen some $r$, ,we can pass verification.

## Attack conclusion

As can be seen - $I$ isn't unique! The only requirement is that it would be a 0 point.\
As there are many valid zero points in the Zano representation, this transaction output can be double, triple, and many more spent.

## Suggested fix

Call `is_zero` to verify that the key\_image is never zero, as there is no valid key\_image that is equal to zero anyway.

## Note

I hope I didn't miss anything, sorry for having a few false reports so far, the math was complex to verify and I wanted to leave to stone unturned.\
Nice project and nicely done.

## Proof of Concept

2 different valid key\_images might be: (0,2,2,0), (0,3,3,0).
