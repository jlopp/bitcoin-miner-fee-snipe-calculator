# Bitcoin Miner Undercut / Fee Snipe Calculator

Calculates if a rational economic miner will attempt to reorganize the blockchain for a given bounty, generally to snipe a high-fee transaction.

A clean way to model this is as a **fee-sniping reorg option**. The miner is comparing:

- **Mine honestly on the public tip**
- **Mine privately on the parent of the last block and try to make that private fork canonical**

In Bitcoin, a transaction fee is claimable by the miner whose block includes the transaction, via the block’s coinbase. If that block later becomes stale, its coinbase reward disappears. Competing blocks at the same height do not automatically replace each other; nodes usually mine on the first valid block they saw until one branch becomes the most-work chain.

References:

- [Bitcoin Developer Guide: Block Chain](https://developer.bitcoin.org/devguide/block_chain.html)
- [Bitcoin Whitepaper](https://bitsblocks.github.io/bitcoin-whitepaper)

---

## 1. Define the Variables

Let:

```math
\begin{aligned}
q &= \text{attacker's share of total hash rate} \\
p &= 1-q = \text{honest/public-chain hash share} \\
F &= \text{extraordinary transaction fee the miner wants to capture} \\
R &= \text{ordinary reward per private block, excluding } F \\
B &= \text{number of already-mined private blocks whose rewards are recoverable only if the fork wins} \\
\Phi &= \text{expected penalty/risk cost if the attack is detected or succeeds} \\
c &= \text{opportunity cost per network block event of mining privately}
\end{aligned}
```

A good first approximation is:

```math
c \approx qR_{\text{honest}} + C_{\text{extra}}
```

where:

- $qR_{\text{honest}}$ is the expected reward the miner gives up by not mining honestly on the public tip.
- $C_{\text{extra}}$ captures extra financing, reputational, pool-hash defection, price-impact, or operational risk costs.

Let:

```math
d = \text{public chain lead over the private fork}
```

For a last-block reorg:

```math
d = 1
```

at the moment the miner starts, because the public chain already has the high-fee block and the attacker’s private fork has not yet replaced it.

If the attacker mines a private replacement block at the same height:

```math
d = 0
```

If the attacker’s private fork becomes one block ahead:

```math
d = -1
```

That is the conservative “guaranteed reorg” success condition.

At each new block discovery:

```math
\begin{aligned}
d &\to d-1 \quad \text{with probability } q \\
d &\to d+1 \quad \text{with probability } p
\end{aligned}
```

So the miner is playing a biased random walk. If $q < p$, the public chain has positive drift and the miner’s chance decays exponentially as they fall further behind.

---

## 2. Simple Attack/No-Attack Rule

Suppose the miner commits to giving up if the public chain lead reaches $G$ blocks.

They continue while:

```math
d < G
```

and abandon the attack once:

```math
d = G
```

For the initial last-block attack, $d = 1$, so the smallest meaningful give-up boundary is:

```math
G = 2
```

This means:

> Try the attack, but give up if the public chain gets two blocks ahead.

Define:

```math
\rho = \frac{q}{p}
```

For $q \neq p$, the probability that the private fork reaches $d = -1$ before the public lead reaches $G$ is:

```math
P_{\text{win}}(d,G)
=
\frac{\rho^{d+1}-\rho^{G+1}}{1-\rho^{G+1}}
```

For $q = p = 1/2$, use:

```math
P_{\text{win}}(d,G)
=
\frac{G-d}{G+1}
```

The expected number of network block events before either success or abandonment is, for $q \neq p$:

```math
T(d,G)
=
\frac{
(G+1)\left(\frac{1-\rho^{d+1}}{1-\rho^{G+1}}\right)-(d+1)
}{p-q}
```

For $q = p = 1/2$:

```math
T(d,G) = (d+1)(G-d)
```

Now define the net prize if the attack succeeds:

```math
W = F + BR - \Phi
```

The simplified expected value of trying with give-up boundary $G$ is:

```math
EV(d,G)
=
P_{\text{win}}(d,G) \cdot W
-
c \cdot T(d,G)
```

So the initial decision rule is:

```math
\boxed{
\text{Attack iff }
\max_{G>1}
\left[
P_{\text{win}}(1,G)(F+BR-\Phi)-cT(1,G)
\right]
>0
}
```

At the very start, usually $B = 0$, so:

```math
\boxed{
\text{Attack iff }
\max_{G>1}
\left[
P_{\text{win}}(1,G)(F-\Phi)-cT(1,G)
\right]
>0
}
```

Equivalently, the break-even high fee is:

```math
\boxed{
F_{\min}
=
\min_{G>1}
\left[
\frac{cT(1,G)}{P_{\text{win}}(1,G)}
\right]
+\Phi
}
```

If:

```math
F > F_{\min}
```

then a risk-neutral miner would regard the attack as positive expected value under this simplified model.

---

## 3. How Many Blocks Should They Keep Mining Before Giving Up?

The miner should not precommit blindly. After every block event, they should recompute the value from the new state.

At any current public lead $d$, define:

```math
V(d)
=
\max_{G>d}
\left[
P_{\text{win}}(d,G)(F+BR-\Phi)-cT(d,G)
\right]
```

Then the stopping rule is:

```math
\boxed{
\text{Continue mining the private fork iff } V(d)>0
}
```

```math
\boxed{
\text{Give up once } V(d)\le 0
}
```

The “number of blocks” is therefore not a fixed wall-clock number. It is a **deficit threshold**.

If the largest deficit at which $V(d)>0$ is:

```math
d_{\max}
```

then the miner continues while:

```math
d \le d_{\max}
```

and gives up when the public chain lead becomes:

```math
d_{\max}+1
```

So from the current state $d_0$, the number of additional net public-chain blocks they are willing to tolerate is:

```math
\boxed{
(d_{\max}+1)-d_0
}
```

where “net public-chain blocks” means:

```math
\text{honest/public blocks} - \text{attacker/private blocks}
```

---

## 4. More Exact Bellman Formulation

The previous formula is the useful closed-form version. A more exact rational-miner formula is a dynamic program:

```math
V(d,B)
=
\max
\left\{
0,\;
-c + qV(d-1,B+1) + pV(d+1,B)
\right\}
```

with terminal success condition:

```math
V(-1,B)=F+BR-\Phi
```

Interpretation:

- The miner may stop now and get incremental value $0$.
- If they continue, they pay opportunity cost $c$.
- With probability $q$, they mine the next private block, so $d$ falls by 1 and $B$ rises by 1.
- With probability $p$, the public chain advances, so $d$ rises by 1.
- If $d=-1$, the private fork is ahead and can be published to claim the fee and private block rewards.

The stopping rule becomes:

```math
\boxed{
\text{Continue iff } V(d,B)>0
}
```

This version properly captures the fact that, once the miner has already mined private blocks, those private block rewards become contingent assets recoverable only if the reorg succeeds.

---

## 5. Tie-Publication Variant

The conservative model above assumes the attacker waits until the private fork is **one block ahead**.

A miner might instead publish once they reach a tie:

```math
d = 0
```

That is riskier because same-height blocks do not automatically displace the first-seen block.

Let:

```math
\gamma = \text{fraction of honest hash power that will mine on the attacker’s block in a tie}
```

Then the probability the attacker’s branch wins the next-block race after tie publication is approximately:

```math
\beta = q + \gamma p
```

At $d = 0$, the miner should compare:

```math
V_{\text{publish tie}}(0,B)
=
\beta(F+BR-\Phi)-c_{\text{tie}}
```

against continuing privately:

```math
V_{\text{continue private}}(0,B)
=
-c+qV(-1,B+1)+pV(1,B)
```

So at a tie:

```math
\boxed{
V(0,B)
=
\max
\left\{
0,\;
V_{\text{publish tie}}(0,B),\;
V_{\text{continue private}}(0,B)
\right\}
}
```

If network propagation favors the attacker, $\gamma$ is high and tie publication may dominate. If most honest miners already saw the original block first, $\gamma$ is low and the one-block-ahead model is safer.

---

## 6. Toy Example

Assume:

```math
\begin{aligned}
q &= 0.20 \\
p &= 0.80 \\
\rho &= 0.25 \\
F-\Phi &= 100 \text{ BTC} \\
c &= 0.8 \text{ BTC per network block event}
\end{aligned}
```

Starting from $d = 1$, suppose the miner gives up if the public lead reaches:

```math
G = 2
```

Then:

```math
P_{\text{win}}(1,2)
=
\frac{0.25^2-0.25^3}{1-0.25^3}
\approx 0.0476
```

```math
T(1,2) \approx 1.4286
```

Therefore:

```math
EV(1,2)
=
0.0476(100)-0.8(1.4286)
\approx 3.62 \text{ BTC}
```

So under those toy assumptions, the attack is positive EV, but only barely and only with a shallow give-up threshold. If the miner falls another public block behind, the expected value may turn negative quickly.

---

## Core Result

A rational miner attacks only when:

```math
\boxed{
P_{\text{win}} \cdot \text{net prize}
>
\text{opportunity cost + risk cost}
}
```

The private-fork continuation rule is:

```math
\boxed{
\text{Keep mining only while the recomputed continuation value } V(d,B) \text{ remains positive.}
}
```