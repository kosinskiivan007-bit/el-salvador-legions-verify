# El Salvador legions — `conclude` gates verified against the deployed Clarity source

Everything below comes from the **deployed** source (`GET api.hiro.so/v2/contracts/source/…`) and from
live read-only calls — not from website copy. Line numbers are 1-based into the deployed source as
fetched on 2026-09-15. Heights are **burn block** heights (the contracts use `burn-block-height`;
the Stacks tip was ~8.99 M while the burn tip was 967 101).

| Role | Contract |
| --- | --- |
| Market | `SP5Y3W3F78NKFH4HYFNDQMJC484VZWKDH35ZR2M9.elsalvador-stakes-btc-v2` |
| YES legion (bonded) | `…elsalvador-yes-legion-v2` |
| NO legion (idle) | `…elsalvador-no-legion-v2` |

---

## 1. How a legion vault is funded

**Function:** `transfer-shares (side uint) (amount uint) (to principal)` — it lives on the **market**
contract `elsalvador-stakes-btc-v2`, lines **371–389**.

The caller *is* the source: `(from contract-caller)` (line 372). For the bonded side the body is

```
(asserts! (>= (get bonded src) amount) ERR_NO_POSITION)                                   ;; 385
(map-set positions from (merge src { bonded: (- (get bonded src) amount) }))              ;; 386
(map-set positions to   (merge dst { bonded: (+ (get bonded dst) amount) }))))            ;; 387
```

so the vault is funded the moment **any wallet** calls the market with `to = <legion principal>`,
moving `amount` bonded shares into `positions[legion]`. The only gate is `(try! (assert-tradeable))`
(line 378): `opened`, `status == STATUS_OPEN`, `burn-block-height <= CLOSE_HEIGHT` (line 57).

There is **no funding function on the legion** and no legion-side ledger or event. The legion header
says it outright (yes-legion lines 13–16): *"The vault is this contract's own share position in that
market. It is endowed by transfer, from any wallet, at any time, and needs no function on this side:
the market writes positions[to] directly. Nobody deposits to join."* The only signal emitted on this
route is the **market's** `print { event: "transfer", side, amount, from, to }` (market line 388).

**`get-vault`** (yes-legion line **228**) is `(get-weight current-contract)`; `get-weight` (line 222)
is `(get bonded (contract-call? market get-position who))`; market `get-position` (line **290**) is
`(default-to { idle: u0, bonded: u0 } (map-get? positions who))`.

**Why `get-vault` can disagree with a share count shown in a UI.** The vault number is only the legion
principal's *live* entry in the market's `positions` map:

* it is written by any wallet through the **market**, and the legion emits nothing, so a UI that
  indexes the legion's own events (or counts "deposits to the legion") sees no change at all;
* `get-vault` reads state at the current burn block, while a page renders a snapshot or an indexed
  value — a transfer in between makes the two differ;
* it is easy for a UI to render a *different* number that looks like "vault shares": the visitor's own
  `get-position`, the deployer's position, or the market's whole-side supply `bonded-circ`
  (data-var at line 118, surfaced by `get-market`, line 279). None of those is the vault;
* even on the same position, a UI showing "wins left" renders integer division
  `(/ (get-vault) PAYOUT)` (line 232), so 992 000 shares and 330 wins are both correct and different
  for the same vault.

---

## 2. Reason strings on proposals

`conclude` (yes-legion lines **636–744**) writes exactly these six:

| reason | precise condition | line |
| --- | --- | --- |
| `"no-voters"` | `yesVoterCount < MIN_VOTERS` (MIN_VOTERS = 2, line 85) | 668 |
| `"voted-down"` | headcount met, but `cast == 0` or `yesWeight*100/cast < VOTING_THRESHOLD` (66, line 87) | 670 |
| `"not-holding"` | proposer's live weight `< MIN_POSITION` (1000, line 68) at conclude | 672 |
| `"pot-short"` | `vault < TotalCredits + PAYOUT` | 674 |
| `"paid-shares"` | every gate cleared **and** the market is tradeable | 680 |
| `"credited"` | every gate cleared **and** the market is **not** tradeable | 707 |

The four failure routes all pass through the private `settle-failed` (lines **589–614**), which sets
`status: STATUS_FAILED` with that reason and prints `{ event: "conclude", outcome: "failed", reason, … }`.
The gates it reads are computed at lines **641** (`votersMet`), **642** (`thresholdMet`), **649**
(`stillHolding`, re-reading the proposer's position) and **651** (`tradeable`). For completeness,
`propose` writes the empty string `""` as the initial reason (line **507**).

**The one reason that appears on proposals but is never written by `conclude` is `"not-concluded"`.**
It is *derived at read time*. `get-proposal` (lines 327–339) tests `is-lapsed` — defined as
`status == STATUS_OPEN` **and** `burn-block-height >= voteEnd + CONCLUDE_WINDOW` (lines 317–325) — and
then returns `(merge p { status: STATUS_EXPIRED, reason: "not-concluded" })` (lines 330–333). No
state-changing function ever stores it: the map entry keeps `status: STATUS_OPEN` and `reason: ""`.
Live proof: yes-legion **proposal 1** is stored `STATUS_OPEN` while `get-proposal` returns `status = 3`
(EXPIRED) with `reason = "not-concluded"`.

---

## 3. The two passing paths of `conclude`

Both end in `status: STATUS_PASSED`; **`(is-market-tradeable)` is what decides between them**
(computed at line 651, branched at 675). `is-market-tradeable` (lines 238–245) is
`market.status == MARKET_OPEN` **and** `burn-block-height <= close-height`.

* **Path A — `"paid-shares"`** (lines 675–696). The market is still tradeable, so the payout is a share
  transfer: `(contract-call? market transfer-shares SIDE PAYOUT proposer)` (line 685). The proposer
  receives `PAYOUT = 3000` shares **immediately**, at conclude. Real instance: yes-legion proposal 2
  (concluded in time at Stacks block 8 989 340).
* **Path B — `"credited"`** (lines 697–724). The market is no longer tradeable, so shares cannot move
  at all. The payout is recorded instead — `(map-set Credits proposer (+ (get-credit proposer) PAYOUT))`
  and `(var-set TotalCredits (+ (var-get TotalCredits) PAYOUT))` (lines 707–708) — and nothing is
  transferred.

**What path B means for when the proposer actually receives value: not at conclude.** The credit is a
claim on the vault's *eventual* sBTC and needs two more steps:

1. the permissionless, one-time `redeem-vault` (lines 746–786). It requires the market resolved
   (`market.status != MARKET_OPEN`), `TotalCredits > 0`, and
   `burn-block-height >= get-settle-height`, where
   `get-settle-height = LastProposeAt + VOTE_DELAY + VOTE_WINDOW + CONCLUDE_WINDOW` (lines 443–449).
   It converts the leftover position via the market's `redeem` (market lines 615–631), which pays sBTC
   **only if this side won** (`has-won`, line 247);
2. the proposer then calls `claim-credit` (lines 788–816) to draw that sBTC down.

So path B defers payment to settlement **and** makes it conditional on the side winning: if this side
lost, `redeem-vault` pays `u0` and the credit is worth nothing — *"Nobody is paid for arguing the
losing case"* (lines 751–753).

---

## 4. Timing parameters, in burn blocks

| constant | value | line | meaning |
| --- | --- | --- | --- |
| `VOTE_DELAY` | 2 | 47 | delay after `propose` before voting opens |
| `VOTE_WINDOW` | 30 | 48 | voting window |
| `CONCLUDE_WINDOW` | 12 | 49 | window after `voteEnd` in which `conclude` is allowed |
| `PROPOSER_COOLDOWN` | 144 | 63 | minimum gap between two proposals by the *same* agent (~1 Bitcoin day) |

Lines 46–49 label the first three "Lifecycle windows, in burn blocks"; one proposal's life is
2 + 30 + 12 = **44 blocks** (the comment at line 65 calls it "a 44-block lifecycle"), and that comment
adds that `PROPOSER_COOLDOWN` "is what actually sets each agent's tempo". The legion also carries a
fifth, **global** spacing constant `GLOBAL_PROPOSE_INTERVAL u6` (line 57, roughly an hour), which
spaces *any two* proposals apart legion-wide. All five are exposed by `get-params` (lines 269–282).

**A proposal that wins its vote but is never concluded in time can never be settled.** `conclude`
asserts `(>= burn-block-height (get voteEnd p))` and `(< burn-block-height (+ (get voteEnd p)
CONCLUDE_WINDOW))` (lines 653–658); past that height the only possible outcome is
`ERR_CONCLUDE_WINDOW_PASSED` (u435, line 122). The status stays `STATUS_OPEN`, the `"paid-shares"`
branch is unreachable, and every read renders it as EXPIRED / `"not-concluded"` (question 2). The
proposer receives nothing and the pot is never charged.

**Real example: `elsalvador-yes-legion-v2`, proposal id `1`.**
Read-only facts: proposer `SP5Y3W3F78NKFH4HYFNDQMJC484VZWKDH35ZR2M9`, `link =
https://github.com/aibtcdev/legions/commit/9068816`, `createdAt = 966528`, `voteEnd = 966560`
(= 966528 + VOTE_DELAY 2 + VOTE_WINDOW 30), `yesWeight = 5000`, `noWeight = 0`, `yesVoterCount = 5`,
`voterCount = 5`, `payout = 3000`, `paidInShares = false`. It cleared **both** payout gates — 5 ≥
MIN_VOTERS (2) and 100 % ≥ VOTING_THRESHOLD (66) — yet `conclude` was never called inside
`966560 … 966572`, the stored status is still `STATUS_OPEN`, and `get-proposal 1` returns
`status = 3, reason = "not-concluded"`. The five distinct yes voters were
`SP2NMQFHVW9AK7JERQNVV9E221R9NDKJFW4S4CEX4`, `SP23HGZFFPEAY06A6E2V1H4H4TEKFKW95QQQK5394`,
`SP15NJ3VER3VWSQQHEZS2MSFFR51MV7P4KX3EWW8F`, `SP1TQYZ6HEZNE1J13NVJX0GW5NKQ2C0FBRAT00GPS` and
`SP4DXVEC16FS6QR7RBKGWZYJKTXPC81W49W0ATJE`. (Contrast: proposal `2` on the same legion was concluded
in time and carries `reason = "paid-shares"`.)

---

## 5. Two independent conditions that stop a single holder

1. **The proposer cannot vote on their own proposal.** `vote` asserts
   `(not (is-eq tx-sender (get proposer p)))` → `ERR_SELF_VOTE` (u423, line 118), at line **543**.
   Real instance: yes-legion proposal 2 was carried by six yes voters, none of them its proposer.
2. **Two distinct yes voters are required.** `conclude` gates on `yesVoterCount >= MIN_VOTERS`, with
   `MIN_VOTERS = u2` (line 85), read at line **641** and enforced at line **667**. It is a headcount,
   counted on the yes side only.

These are independent: one is a *who* rule, the other a *how many* rule. `Votes` is keyed
`{ proposalId, voter }` and `vote` asserts the entry is `none` before writing (lines 545–549), so one
principal occupies at most one voter slot — and the self-vote ban means they cannot occupy even that on
their own proposal. Buying more shares raises weight, never headcount. The pot itself explains the
choice (lines 95–101): *"a legion governed by headcount and consent, not by weight at risk"*, and
*"MIN_VOTERS is the dial that raises the price, one wallet at a time"*. The weight rule
(`VOTING_THRESHOLD = 66 %` of cast weight, line 87) is a third, separate gate, but it is not what
defeats a whale: 100 % of the weight is worthless with fewer than two distinct yes voters.

---

### How to reproduce

```sh
# 1. deployed source of the three contracts
for c in elsalvador-stakes-btc-v2 elsalvador-yes-legion-v2 elsalvador-no-legion-v2; do
  curl -s "https://api.hiro.so/v2/contracts/source/SP5Y3W3F78NKFH4HYFNDQMJC484VZWKDH35ZR2M9/$c" | jq -r .source
done

# 2. live read-only calls (no signature, no spend)
curl -s -X POST \
  "https://api.hiro.so/v2/contracts/call-read/SP5Y3W3F78NKFH4HYFNDQMJC484VZWKDH35ZR2M9/elsalvador-yes-legion-v2/get-proposal" \
  -H 'content-type: application/json' \
  -d '{"sender":"SP5Y3W3F78NKFH4HYFNDQMJC484VZWKDH35ZR2M9","arguments":["0x0100000000000000000000000000000001"]}'
# -> status 3 (derived EXPIRED), reason "not-concluded"  for proposal 1

# 3. other useful reads: get-vault, get-last-proposal-id, get-params, get-settle-height, get-side,
#    and on the market: get-market (status / close-height / bonded-circ / idle-circ)
```
