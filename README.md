# Plexa — Decentralized ROSCA Protocol on Stellar / Soroban

Plexa is a decentralized **Rotating Savings and Credit Association (ROSCA)** protocol
built on Stellar's Soroban smart-contract platform. A group of members each contribute a
fixed amount per period into a shared pot; every period exactly one member receives the
pot — chosen by an open **discount auction** falling back to join-order rotation. This repeats until
every member has won exactly once, after which locked collateral is returned.

Think of it as a trustless, on-chain version of the informal savings circles
(known as *susu*, *tanda*, *chit fund*, *hui*, *chama*…) used by billions of people —
but with programmable collateral, automatic default coverage, and a transparent, publicly
verifiable ledger of every action.

---

## Demo video link : https://youtu.be/pvfV9YEylpg
## Live demo link : https://plexa-v1.vercel.app/
## Feedback Form: [Google Form](https://docs.google.com/forms/d/e/1FAIpQLScuDHDzIp4WTlVfaYT1PZZXE5snbTBucxjzs3YbXMmQffshLg/viewform?usp=dialog)
## Feedback sheet: [Google Sheet](https://docs.google.com/spreadsheets/d/1YvV6IvuoG-wsqKcACewvXJMPD38hP5kKXVMfI43n2vE/edit?usp=sharing)

---

## Table of Contents

- [Key Features](#key-features)
- [Screenshots](#screenshots)
- [How a ROSCA Round Works](#how-a-rosca-round-works)
- [Architecture](#architecture)
- [Smart Contracts](#smart-contracts)
- [Group Lifecycle & Period Mechanics](#group-lifecycle--period-mechanics)
- [Tech Stack](#tech-stack)
- [Deployed Contracts (Testnet)](#deployed-contracts-testnet)
- [Verified On-Chain Activity](#verified-on-chain-activity)
- [Repository Layout](#repository-layout)
- [Getting Started](#getting-started)
- [Building, Testing & Deploying Contracts](#building-testing--deploying-contracts)
- [Design Decisions](#design-decisions)
- [Status & Roadmap](#status--roadmap)

---

## Key Features

- **Rotating payouts** — one member wins the pot each period until everyone has won once.
- **Open discount auction** — members bid the discount they'll give up to win early; the
  discount is redistributed equally among all members. Join-order rotation breaks the
  no-bid case.
- **Per-group currency** — each group runs entirely in **USDC** *or* **XLM**
  (contributions, pot, payouts, and claims all flow in the chosen currency).
- **Multi-asset collateral** — members lock collateral to join:
  - USDC groups: USDC at **100%** of pot, or XLM at **150%** (oracle-sized, health-factor
    monitored).
  - XLM groups: same-asset XLM at **100%** of pot (no oracle / no swaps needed).
- **Automatic default coverage** — a dedicated **settlement window** liquidates a missed
  contribution from the member's collateral. For USDC groups this swaps only the needed
  XLM → USDC through the **real Soroswap testnet router**; uncoverable shortfalls become
  tracked on-chain debt instead of bricking the group.
- **Health factor + top-up grace** — XLM collateral is continuously valued; if it drops
  below 1.0 the member has one cycle to top up (XLM or USDC) or be liquidated.
- **On-chain governance** — majority-vote join approval, per-group reputation gating.
- **Full transparency** — every state change emits an event **and** is written to an
  on-chain history log the frontend reads directly.
- **Demo mode** — the whole UI runs offline against an in-memory store (no wallet, no
  chain) for local development and demos.

---

## Screenshots

| Landing page | Multi-account wallets | Phone view | Transaction through wallet |
|:---:|:---:|:---:|:---:|
| ![Landing page](screeenshot/landing_pg.png) | ![Two wallets](screeenshot/two_wallet.png) | ![Phone view](screeenshot/ph_view.png) | ![Transaction complete](screeenshot/paument_frieghter.png) |

> 📸 For more screenshots (group creation, collateral lock, auction round, Freighter/Albedo
> connect, claiming payouts, …) see the [`screeenshot/`](screeenshot) folder.

---

## How a ROSCA Round Works

```
Members join → lock collateral → auto-start when full & funded
        │
        ▼
  ┌─────────────────── each period ───────────────────┐
  │  Contribution → Settlement → Auction → Payout      │
  │  everyone pays   cover misses  bid to win  winner   │
  │                                 the pot    is paid   │
  └────────────────────────────────────────────────────┘
        │  (repeat: each member wins exactly once)
        ▼
   Cycle complete → grace window → withdraw remaining collateral
```

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          Frontend (React + Vite)                        │
│   Landing · Groups · CreateGroup wizard · GroupDetail · Dashboard       │
│   Freighter wallet · @stellar/stellar-sdk · notifications · demo mode   │
└───────────────┬────────────────────────────────────────────────────────┘
                │ simulate (reads) / prepare · sign · submit · poll (writes)
                ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     Soroban Contracts (Rust workspace)                  │
│                                                                          │
│   ┌────────────┐  create_group()   ┌──────────────────────────────┐    │
│   │  Factory   │ ─────────────────▶ │  Group  (one per ROSCA)       │    │
│   │ discovery  │  reputation sync   │  members · collateral · pot   │    │
│   │ reputation │ ◀───────────────── │  auction · periods · history  │    │
│   └────────────┘                    └───────┬───────────┬──────────┘    │
│                                              │ price     │ swap          │
│                                              ▼           ▼               │
│                                       ┌──────────┐  ┌──────────────┐     │
│                                       │  Oracle  │  │  Soroswap     │     │
│                                       │ XLM/USDC │  │  Router       │     │
│                                       │  price   │  │ (XLM→USDC)    │     │
│                                       └──────────┘  └──────────────┘     │
└──────────────────────────────────────────────────────────────────────┘
                       │ token transfers (USDC / XLM SACs)
                       ▼
                Stellar Testnet (Protocol 23)
```

- **Reads** go through `simulateTransaction` (no signature, no fee).
- **Writes** are built, `prepareTransaction`-simulated, signed by the user's Freighter
  wallet, submitted, then polled for `SUCCESS`.
- The **Group contract** calls the **Oracle** for collateral sizing/health factors and the
  **Soroswap router** for liquidation swaps — the frontend never talks to Soroswap
  directly.

---

## Smart Contracts

A 4-crate Rust workspace (`contracts/`), each compiled to a Soroban Wasm contract.

### `plexa-factory` — deployment, discovery & reputation
- `create_group(CreateParams)` — deploys a new Group instance (`deploy_v2` + constructor),
  registers it, and lists it in the public discovery feed when `Public`.
- `sync_reputation(group)` — permissionless, idempotent; pulls reputation from a completed
  group into the cross-group registry.
- Views: `rep_of`, `get_public_groups`, `get_all_groups`, `admin`.

### `plexa-group` — one instance per ROSCA
Holds all group state: config, members, collateral (per-asset buckets), contributions,
period/phase timing, auction state, governance votes, and win history.

| Function | Purpose |
|---|---|
| `__constructor(GroupParams)` | Locks immutable params (incl. `currency`) at deploy. |
| `request_join` / `vote_on_join` | Majority-vote join approval. |
| `lock_collateral(member, asset)` | Lock collateral to become a member. |
| `top_up(member, asset, amount)` | Add collateral to restore a low health factor. |
| `contribute(member)` | Pay a period's contribution (funds period 1 while Forming). |
| `settle` | Verify contributions, liquidate misses, finalize pot, update health factors. |
| `place_bid(member, discount)` | Open auction bid — the discount given up to win. |
| `resolve_period` | Pick winner (top bid / rotation), split discount, advance period. |
| `claim_payout(member)` | Claim payouts + redistributed discount to wallet. |
| `withdraw_collateral(member)` | Withdraw remaining collateral after cycle + grace. |
| **Views** | `get_config/state/members/phase/claimable/current_bid/join_request/pending_joins/history`, `has_won`, `is_completed`, `get_settled`, `get_pot`, `health_factor`, `required_collateral`, `collateral_unlock_at` |

### Upgrades & the keeper

Two operational pieces added in v6:

- **`Group::upgrade(new_wasm_hash)`** — replaces a group's code in place, keeping
  its state and address. Authorized by the **factory's admin**, not the group
  owner: a group custodies member collateral, so letting its creator swap the code
  in would let them drain it. The admin is read live from `config.factory`, so
  `Factory::set_admin` rotation takes effect immediately. This is a trusted role —
  put it behind a multisig or timelock before mainnet.
- **`Factory::set_group_wasm(hash)`** — points future `create_group` calls at a new
  build without deploying a new factory. Existing groups keep the wasm they were
  created with; move those with `Group::upgrade`.
- **Keeper** (`keeper/keeper.mjs`, scheduled by `.github/workflows/keeper.yml`) —
  calls `settle` and `resolve_period`, which are permissionless, so **members never
  sign to advance a period**. They sign only for their own money: contribute, bid,
  claim, withdraw. Needs a funded account in `KEEPER_SECRET` and `FACTORY_ID`.

> **Why the keeper widens the footprint.** Older builds drew the no-bid winner with
> `env.prng()`, which is seeded per-transaction, so preflight and execution could
> pick different members. Simulation declared only `Claimable(winner_it_drew)`, and
> execution writing a different key trapped with `scecExceededLimit` — wedging the
> group permanently. The current wasm uses fixed rotation, so this cannot happen;
> the keeper still declares every eligible member's key client-side, which is what
> lets it drive groups from the superseded factory that remain on `d58bb092…` and
> have no `upgrade` entrypoint to rescue them.

### `plexa-oracle` — XLM/USDC price feed
A **Reflector adapter** (7-decimal fixed point), used for XLM-collateral sizing and
health factors. Reads real Reflector CEX/DEX feeds for XLM and USDC and derives a
cross rate; refuses stale prices past `max_age`. No admin key can author a price —
this replaced the earlier admin-set oracle.

### `plexa-swap` — Soroswap-compatible venue *(testnet fallback)*
A mock XLM→USDC router matching the Soroswap router ABI, kept as an emergency fallback.
The live deployment liquidates through the **real Soroswap testnet router** instead.

---

## Group Lifecycle & Period Mechanics

- **Auto-start** — a group starts automatically when member count == target **and** all
  collateral is locked **and** all first contributions are paid. There is no "start" button.
- **Four windows per period** — `contribution → settlement → auction → payout`, derived
  from `period_length` (`payout = period − contribution − settlement − auction`). The
  settlement window is creator-configurable for dev/testing.
- **Settlement** (`settle`, permissionless — auto-run by `place_bid`/`resolve_period` as a
  safety net) verifies contributions, liquidates unpaid members (same-currency bucket
  first, then a minimal XLM→USDC swap via Soroswap for USDC groups), finalizes the pot, and
  recomputes health factors. The auction always starts from a full pool.
- **Open auction** — highest discount leads; bids are public. The winner receives
  `pot − discount`, and the discount is split equally among **all** members incl. the
  winner (integer-division dust goes to the winner). A member who has already won cannot
  bid again.
- **Default coverage** — a missed contribution is covered from collateral and logged. If
  collateral can't fully cover it, the shortfall is recorded as **debt** (netted from
  future claims/collateral) and the group keeps running — payouts reflect what was actually
  collected (`pot_collected`).
- **No-bid fallback** — plain rotation: the eligible member who joined earliest wins.
  It only chooses *order*, never *whether* someone wins (everyone wins exactly once),
  so randomness bought nothing while creating two problems — a per-transaction PRNG
  seed made preflight and execution disagree and trap the host, and simulation let
  whoever submitted preview the draw and broadcast only when it favoured them.
- **Keeper-independent** — `contribute`, `place_bid`, `claim_payout` and
  `withdraw_collateral` close out any overdue periods first (bounded to 2 per call).
  The keeper keeps periods punctual, but if it stops, the group still advances as
  soon as any member acts, so funds are never frozen behind a stalled bot.


---

## Deployed Contracts (Testnet)

Current live deployment (**v8** — Reflector oracle, upgrade timelocks, registry
verification), from `frontend/.env`:

| Contract | Address |
|---|---|
| **Factory** | `CDOYIGNCIR4QTUTAUYEFSW7IJVS6ZMOFV6CW574VFGHQ5ZDCQCJZ4GDZ` |
| **USDC** (Circle testnet SAC) | `CBIELTK6YBZJU5UP2WWQEUCYKLPU6AUNZ2BQ4WWFEIE3USCIHMXQDAMA` |
| **XLM** (native SAC) | `CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC` |
| **Oracle** (Reflector adapter) | `CADEJYCKYSWL7V5HKMPV5ZYGLU7G6PCZ4WPASQT26FIGVSRIP3ZZ7DE5` |
| **Soroswap Router** | `CCJUD55AG6W5HAI5LRVNKAE5WDP5XGZBUDS5WNTIVDU7O264UZZE7BRD` |

- **Network:** `Test SDF Network ; September 2015` · RPC `https://soroban-testnet.stellar.org`
- **Group Wasm hash:** `0c17b4600b662d389e07a953ef7f1f4367d8bbdf28fa5887329b7b180b458939`
  (v8 — Reflector-fed oracle instead of an admin-set price, 48h-timelocked
  upgrades on both Group and Factory, and `Factory::is_group` so the frontend
  can reject a look-alike group deployed against a spoofed factory).
- **Factory admin:** `GBQLFIT7UWCURWEDBR5JPCQPFIFYJ23AKIX24P6MOAAZ3AIDTSNE24FY`
- **Superseded:** factory `CD6OKM7JO3BFZ644VM6J7NP4BVXEDUQYDR4SFJJJNGDK2KWP3DFIVAMY`
  (group wasm `d58bb092…`). Its groups are **not upgradeable** — that build predates
  the `upgrade` entrypoint — and its `resolve_period` can trap (see below). Those
  groups still resolve via the keeper, which widens the footprint itself.
- **Mainnet Soroswap router (for later, re-verify before use):**
  `CAG5LRYQ5JVEUI5TEID72EYOVX44TTUJT5BQR2J6J77FH65PCCFAJDDH`

> Amounts use `i128` at **7 decimals** (Stellar convention): `1 USDC = 10_000_000`.
> Older factory/USDC deployments are retained (commented) in `frontend/.env`.

---

## Verified On-Chain Activity

Every wallet below is a real Stellar testnet keypair, freshly generated and
funded from the public Friendbot faucet, that submitted a **real, signed
transaction** to the live factory/group contracts above. No hash here is
invented — each row is read directly off the ledger (via Soroban RPC
`getTransaction`) after the transaction reached `SUCCESS`, and every hash
resolves independently on
[Stellar Expert](https://stellar.expert/explorer/testnet). Follow any link
below to verify it yourself.

**65 distinct wallets · 34 groups · 242 recorded on-chain actions** as of the
last export (one representative action per wallet is shown; most wallets have
several — joining, locking collateral, contributing, claiming).

> **How this traffic was produced (transparency note):** most of these
> wallets were generated by [`scripts/seed-activity.mjs`](scripts/seed-activity.mjs),
> which pairs up fresh keypairs into small 2-member XLM groups and drives them
> through create → join → lock collateral → contribute — real transactions
> exercising the real contract paths, not synthetic or fabricated records.
> They are test traffic used to demonstrate the protocol working end-to-end
> on testnet, not organic distinct users. A handful of the earlier wallets
> (the ones with 7-8 actions each) ran a full ROSCA cycle by hand, including
> auction bidding, payout claims, and collateral withdrawal — see
> [`e2e/e2e.mjs`](e2e/e2e.mjs) for that flow.
>
> Regenerate this table anytime with:
> ```bash
> node scripts/seed-activity.mjs --pairs=25   # optional: add more wallets
> node scripts/export-activity.mjs            # re-reads the ledger, writes docs/activity/*
> ```
> `export-activity.mjs` writes its output (full CSV + markdown log) to
> `docs/activity/`, which is not checked into this repo — regenerate it
> locally any time you want the complete action-by-action detail behind this
> summary. Soroban RPC only serves contract events for a rolling ~week
> window, so re-run the export periodically to keep the archive extending
> itself.

| # | Wallet Address | Transaction Hash | View on Stellar Expert |
|---|---|---|---|
| 1 | `GDW6YTO7NCMPQ52NSM7IO3LD4R4V6WGHLVXHUO5DJOHTGKTE4KAOT6WH` | `d96f5f9bf5356bae9b49f966ae6d2f6b65febfb3c8e72c027aab17c388fdf895` | [View ↗](https://stellar.expert/explorer/testnet/tx/d96f5f9bf5356bae9b49f966ae6d2f6b65febfb3c8e72c027aab17c388fdf895) |
| 2 | `GA33CN4OP6DNUX2XJG2DCE5Y3MLW5QDI5VCN4H6AONY6UOZY746XGD65` | `c5329097719214d17bbfc0702852ef8c76c091338209ce4f75e2236ed0e997d1` | [View ↗](https://stellar.expert/explorer/testnet/tx/c5329097719214d17bbfc0702852ef8c76c091338209ce4f75e2236ed0e997d1) |
| 3 | `GB3NSAI6DU43ACN627X5RR4UO5SGP3I7DGGGKDEDBQZ52VYM4INW7N27` | `f35c0112021c22c306e284228df39a9beb7ee716083d3ceb698be0bfedc75788` | [View ↗](https://stellar.expert/explorer/testnet/tx/f35c0112021c22c306e284228df39a9beb7ee716083d3ceb698be0bfedc75788) |
| 4 | `GBEL4AIXHX3HVTAIMIIF4NL7EEWWQ26LTI4SJXUKKRAFYU77LTZGNWCG` | `a58026da7390d46289dec2fbc1ef410d131dde935371e5731d04a12896052e92` | [View ↗](https://stellar.expert/explorer/testnet/tx/a58026da7390d46289dec2fbc1ef410d131dde935371e5731d04a12896052e92) |
| 5 | `GDSQBZEQMSKP25VKN4Q7G74346MB24BDJEO3UEPWJOOJIWNS3HEEJ6BZ` | `f95b694d9c7b96ea48dc557d2af4c918066f2b711b63094d695269505091ea34` | [View ↗](https://stellar.expert/explorer/testnet/tx/f95b694d9c7b96ea48dc557d2af4c918066f2b711b63094d695269505091ea34) |
| 6 | `GBOD3TRHDXKBFC2V3JBJ2ANMAGWUZCWJ2Y72NGL2UCFT5NDQCERRP7HL` | `6e74eca72480904d8e05396ea3532d283e96bb59119a617f38733690de92c767` | [View ↗](https://stellar.expert/explorer/testnet/tx/6e74eca72480904d8e05396ea3532d283e96bb59119a617f38733690de92c767) |
| 7 | `GDKFMZAH2VIRNLW5WUYLBTOJBQGO7ZOHDWMSHUEDY27QEOYXAJZJIFYP` | `dc5968d07c8a8931be3709eed459395c09cf6750f361460edc261c19ab26a23e` | [View ↗](https://stellar.expert/explorer/testnet/tx/dc5968d07c8a8931be3709eed459395c09cf6750f361460edc261c19ab26a23e) |
| 8 | `GCOG233UUT4DXCPQWKHPOWZH5XFW5ZVXM34TWJQ6DXYQJTPEQSRKZK5T` | `4c84e45f65226489722c18b348d0a43b1c0689f6eb8f8e77c63ced130d93c871` | [View ↗](https://stellar.expert/explorer/testnet/tx/4c84e45f65226489722c18b348d0a43b1c0689f6eb8f8e77c63ced130d93c871) |
| 9 | `GAD75JYQ7BNPCAQ75E3TAMVZQJJHXTUC3CDCAXV6KTEZA4RLDIZCLNIJ` | `bfbb409e0c5dda0e6453eea1bf29ac8a0eaaf3b360771704c1261714c9a4a6c9` | [View ↗](https://stellar.expert/explorer/testnet/tx/bfbb409e0c5dda0e6453eea1bf29ac8a0eaaf3b360771704c1261714c9a4a6c9) |
| 10 | `GB6RUFVZAKZ7LUEMLZD43YAK3B6MNPJBF7SP52NYDSXDDWZO56HKJOQJ` | `ec32be4d189e81e1cf70aac6e6e898cb74d23a7cd8ed28d9dda2692023b09d93` | [View ↗](https://stellar.expert/explorer/testnet/tx/ec32be4d189e81e1cf70aac6e6e898cb74d23a7cd8ed28d9dda2692023b09d93) |
| 11 | `GCYWKXKHC333DJAPUKATYRCB7LHPQLQLWCK3YGM77CDMUUYMQANM3DDI` | `cae250901f3bc955b55b8ffa588bc7e21cb3b05afdfef93c50eee1a8eae7497c` | [View ↗](https://stellar.expert/explorer/testnet/tx/cae250901f3bc955b55b8ffa588bc7e21cb3b05afdfef93c50eee1a8eae7497c) |
| 12 | `GAWCWJ3R37A56FSQ5HDG7WGOG7VZY3SRYPHCEJA2GVMN2R6HOQFSMMDR` | `d676387a2d31329c9232f256c37e9296f34f2b254277e660804929410c996b52` | [View ↗](https://stellar.expert/explorer/testnet/tx/d676387a2d31329c9232f256c37e9296f34f2b254277e660804929410c996b52) |
| 13 | `GDCBAPUJ4235VJZ3LGR7HJVB64ELWPOFBT5FGMILAV5TI27UCP3UYFT4` | `2ba4b7387cb5a1dac33d165ce4389d842cbeb718623050530e3365fd3ceed892` | [View ↗](https://stellar.expert/explorer/testnet/tx/2ba4b7387cb5a1dac33d165ce4389d842cbeb718623050530e3365fd3ceed892) |
| 14 | `GBRLGPDZV4KPRZI6NOTQ7USZLUXM3NFJBMENPTSQKZ2UHX6G3XEN5ARI` | `78c28b7b6ea7e795576c486c25196ac2ad37cb03ed3be073217c4f599f0c66d5` | [View ↗](https://stellar.expert/explorer/testnet/tx/78c28b7b6ea7e795576c486c25196ac2ad37cb03ed3be073217c4f599f0c66d5) |
| 15 | `GDD4OB7JJSB4JGTXPKV7KQIOMAPTKHSN5AX52UURYC476MCEVGLVBINZ` | `580db328bc5a1e0fc1d3ed48697e3ed82f540af70fef4f0f73ba28b38228ef87` | [View ↗](https://stellar.expert/explorer/testnet/tx/580db328bc5a1e0fc1d3ed48697e3ed82f540af70fef4f0f73ba28b38228ef87) |
| 16 | `GAM7DCW4DPK3QMCWRXX3SWAQVSVSB6BJHMIF6NXDVHZPN6TJQDL7YEAI` | `fc7a27ba32d3d7b3477c363c7588c0d2b43e80ef9aa782ab9ad9122c7f8d0573` | [View ↗](https://stellar.expert/explorer/testnet/tx/fc7a27ba32d3d7b3477c363c7588c0d2b43e80ef9aa782ab9ad9122c7f8d0573) |
| 17 | `GA6NBQG6JPVQUXP4JFPAVMMUY3CWODMVPXUI7TYJZS3TVK6EJK3754CI` | `d187387807e94a4583045c3f618522b9547a7ca1776e3f33e0d76fad8b273602` | [View ↗](https://stellar.expert/explorer/testnet/tx/d187387807e94a4583045c3f618522b9547a7ca1776e3f33e0d76fad8b273602) |
| 18 | `GC4732K3PQILHH6ASA25J6QIVYR2OGIPDNIZYFULZLADMU7BWS6T33Q2` | `811b4f369d869c4129f18160433bef03860a0a81b436a4cbc50aa9e1bda7aca9` | [View ↗](https://stellar.expert/explorer/testnet/tx/811b4f369d869c4129f18160433bef03860a0a81b436a4cbc50aa9e1bda7aca9) |
| 19 | `GDE6FO6TYTVS6PDK2GLCRNQ7BOBW2TUMTM7XPKX2HX2HJ5X2UQOS6UIO` | `d7548973d65909e5c056702e3042b9f2a691e1785446350c84efb2abc7a11719` | [View ↗](https://stellar.expert/explorer/testnet/tx/d7548973d65909e5c056702e3042b9f2a691e1785446350c84efb2abc7a11719) |
| 20 | `GBT7MVT44S2HYU3JVL6M2QC4I22W7S4W5SIMSNCM3OSQRACGX5F2N2SH` | `f19fd255b31881aacfd5e9790fee15deec6e8eef6931ca603b52fb60c8fff304` | [View ↗](https://stellar.expert/explorer/testnet/tx/f19fd255b31881aacfd5e9790fee15deec6e8eef6931ca603b52fb60c8fff304) |
| 21 | `GDFER75TQXGHDMMSZQR54PUKRPQI45KE4VIZW3BWF5WNIWUNFXSADGLO` | `69361cc83f99a8fdb56303796b8d628e02c62b4ac2d6b95a566fd90a68b7b6ef` | [View ↗](https://stellar.expert/explorer/testnet/tx/69361cc83f99a8fdb56303796b8d628e02c62b4ac2d6b95a566fd90a68b7b6ef) |
| 22 | `GAPZNHIJXY2DWVNIMEW7SKIZOU5CNQ23BFF2P5NBSQUSAL6HKZT6WZQL` | `1b5c7e08a704a0036ef478921dd27a3d50b717035a33f4a132c9c09ee72318d1` | [View ↗](https://stellar.expert/explorer/testnet/tx/1b5c7e08a704a0036ef478921dd27a3d50b717035a33f4a132c9c09ee72318d1) |
| 23 | `GA5QMWO5LI7INIY3QRW27QDN7OPAUMTNVNDIRCRWF5BRV76AAWWDUCDF` | `13fecf81860de4f3d9d74518c2b07033ec264c354f1beef4036df97e52642a6c` | [View ↗](https://stellar.expert/explorer/testnet/tx/13fecf81860de4f3d9d74518c2b07033ec264c354f1beef4036df97e52642a6c) |
| 24 | `GCH74JQZBSLWZSV2FWNADMR2AX4G5LWA7OJABML6YYVZ3CITNUO7DMA5` | `45ab2a89b46d670b89bf8c002a381c3424588d41e21113af5b7729345a147ed6` | [View ↗](https://stellar.expert/explorer/testnet/tx/45ab2a89b46d670b89bf8c002a381c3424588d41e21113af5b7729345a147ed6) |
| 25 | `GA3ZXDZXZUO2HM4OYIWX2T5YKR3W6E4BJL6JQFPABAD76AVKOOUGJCG5` | `60c09f2fccca230b36735d8859f5ff9bccb30620c474fcd4e29fd1f27221c3ce` | [View ↗](https://stellar.expert/explorer/testnet/tx/60c09f2fccca230b36735d8859f5ff9bccb30620c474fcd4e29fd1f27221c3ce) |
| 26 | `GBENYYG5O5LXYKPT44QVUAV2LAEIVLWKS35IWWHKA6K3KBI4JO5U5KLZ` | `ccbdae679e4d44421fc510ff7c9f9fd877651d3551cb02ff208f81b11d3f0811` | [View ↗](https://stellar.expert/explorer/testnet/tx/ccbdae679e4d44421fc510ff7c9f9fd877651d3551cb02ff208f81b11d3f0811) |
| 27 | `GAKBHWVUIACXOKBFXZ56ZUMWDGQBYQDMYMKD5O2ZQ55JFZPOUFUSBEUR` | `dd91ed082c3e802ae7bc18edd1e5baaa5684324e9f1c1ee68b46fdcc50b761aa` | [View ↗](https://stellar.expert/explorer/testnet/tx/dd91ed082c3e802ae7bc18edd1e5baaa5684324e9f1c1ee68b46fdcc50b761aa) |
| 28 | `GDY5IHUTDBX6BSOKRG22DENO7UVQQTAMO3MHM4VI4UT5SDN6VAUE7ISU` | `16c7ddff37024fe6de84de2875a848371b972a057679e4cb5b5c750a9ed31189` | [View ↗](https://stellar.expert/explorer/testnet/tx/16c7ddff37024fe6de84de2875a848371b972a057679e4cb5b5c750a9ed31189) |
| 29 | `GBVT7745QXEI2XEE2DCFNCRDZF57FCBGKUSKB7BS2AAJQ62R7DEBK742` | `6b5bfc5edb6f3ce641af050fb6eade44496d421548e6e64858154e953babcf4e` | [View ↗](https://stellar.expert/explorer/testnet/tx/6b5bfc5edb6f3ce641af050fb6eade44496d421548e6e64858154e953babcf4e) |
| 30 | `GCLPCWTZMRFMQ6KM7FAMK3C3KAHU5XKBSZP5ZBN5D2LIMGMVCZY3ILVP` | `6d89c4e6cdcc07b4a69e8c0634199588744bd08a3e7bfaf250c9fb399446f59e` | [View ↗](https://stellar.expert/explorer/testnet/tx/6d89c4e6cdcc07b4a69e8c0634199588744bd08a3e7bfaf250c9fb399446f59e) |
| 31 | `GCWGEZUDO5JIKMXI2LHZMMYZFQXGUT4ALDL2TFSH34KKMSYHOTUZVLIK` | `a1d0352039559133c6b0877233740b436f3fdb3f2b25b77a5fbf840991d981e5` | [View ↗](https://stellar.expert/explorer/testnet/tx/a1d0352039559133c6b0877233740b436f3fdb3f2b25b77a5fbf840991d981e5) |
| 32 | `GDZGARJZY4VCPA2BUJ34IG2V2ZZGV2WC262GO2IWCHL34RFNV3DPVFTR` | `89512a785e26a0f918c95d7ff1f49a0619af6b182b14f80a7e94655f965a14d7` | [View ↗](https://stellar.expert/explorer/testnet/tx/89512a785e26a0f918c95d7ff1f49a0619af6b182b14f80a7e94655f965a14d7) |
| 33 | `GDADX5QHRERGQ7PX6TP62SJZZNS2BPQQYWYB33INSM26WNFB6Y2OXUAD` | `7a141ccee4b648bbf2ed76bf29931609b5b0cf58bc96222900d2bbf14e8ba573` | [View ↗](https://stellar.expert/explorer/testnet/tx/7a141ccee4b648bbf2ed76bf29931609b5b0cf58bc96222900d2bbf14e8ba573) |
| 34 | `GDX4FTDE5Q4HCYSGH7B43OH7PVQXN423GOACH26PT7USIVJN75T7EJLL` | `73393d0eb9c5c79c61abb1eb8663438d9cbf76a792e742666ae96e872d87ecb1` | [View ↗](https://stellar.expert/explorer/testnet/tx/73393d0eb9c5c79c61abb1eb8663438d9cbf76a792e742666ae96e872d87ecb1) |
| 35 | `GAUBHC6CXXTAFT3WYBZBCEHUAHQAV6BFSF5AFCN24LNIMBQ4PFF4NJWD` | `a733fa75bd416e12ef21f91d9a50fb9ff958eef7c028c43b0a72be38e8c740d6` | [View ↗](https://stellar.expert/explorer/testnet/tx/a733fa75bd416e12ef21f91d9a50fb9ff958eef7c028c43b0a72be38e8c740d6) |
| 36 | `GAGQ5DEFKUCYSDYSAY56MPTDRN65TU6CIY2UGZFWOLKVDT7NE2IKJQQK` | `ed6d80f0d2f5cf462395ce04757f96ac82f1de87905eb0db0a489c93336effcf` | [View ↗](https://stellar.expert/explorer/testnet/tx/ed6d80f0d2f5cf462395ce04757f96ac82f1de87905eb0db0a489c93336effcf) |
| 37 | `GB5VNIQDEAS6PPEBUFXURXXXQMFSXIZGZLC7O4KGLGFWUZKSXNXWLWNO` | `445401c9a9e1c4dd0b3ef6d8802c68f227944b1e6734986a85d8eff892482608` | [View ↗](https://stellar.expert/explorer/testnet/tx/445401c9a9e1c4dd0b3ef6d8802c68f227944b1e6734986a85d8eff892482608) |
| 38 | `GDHV6GJIEOC4Z5A2PEMVQCDSS6HLYYWWZZRZSWVYCAMGXVDMLZYBQDSY` | `7a8dfbe407694ebbb4d37a6589979c93b7f37de8aeda1f87a895e70ea677d782` | [View ↗](https://stellar.expert/explorer/testnet/tx/7a8dfbe407694ebbb4d37a6589979c93b7f37de8aeda1f87a895e70ea677d782) |
| 39 | `GCNTOT56BOUEAZGPSUV565VVTMUYSG4EW7YS7QKMGTWRQB5S4NP75G3X` | `f8d4a0c9253772b57f86fdb4e9b9ad37245e3e503a87b5f11e731c79fa4fd0f4` | [View ↗](https://stellar.expert/explorer/testnet/tx/f8d4a0c9253772b57f86fdb4e9b9ad37245e3e503a87b5f11e731c79fa4fd0f4) |
| 40 | `GB35GHLXBEM32FYKXYLC62RZNA6LPZZPSMSIEIUSBLVMBY4A27ZECLMJ` | `325acb872512050563aac594307e003733a0c4d63d6d469a23a776a2ba0c963c` | [View ↗](https://stellar.expert/explorer/testnet/tx/325acb872512050563aac594307e003733a0c4d63d6d469a23a776a2ba0c963c) |
| 41 | `GD5LTKERJSP5QD7KMUL3HT2MVOEBYB4Q2TIDHDBCSIU7BO6TJ6RKYKLP` | `ef974786c9ad3b41aebbf5b4bfb2bb3981a90838b314e6e14a15119175dfac12` | [View ↗](https://stellar.expert/explorer/testnet/tx/ef974786c9ad3b41aebbf5b4bfb2bb3981a90838b314e6e14a15119175dfac12) |
| 42 | `GCNJSGSGXRNTRBW52OLW7IBRICDWR6KH5FNHUKXM5WNOPJ7NCPDUTI2A` | `2a6318a7cdbca3bceea3ebf2fcf5136c0b7e8e03c9908732e8c52330bd9ea261` | [View ↗](https://stellar.expert/explorer/testnet/tx/2a6318a7cdbca3bceea3ebf2fcf5136c0b7e8e03c9908732e8c52330bd9ea261) |
| 43 | `GCA34ZEU352N7ZDBN3ITZBZA6PDLZW7GRX4KUJLL3JY73IE3S4M7ZMI4` | `8d859b4a6cb22e68f6beb461f5bd5cf2a3b650df284407bccbc1225b821c1c58` | [View ↗](https://stellar.expert/explorer/testnet/tx/8d859b4a6cb22e68f6beb461f5bd5cf2a3b650df284407bccbc1225b821c1c58) |
| 44 | `GDNQULHW3AIE5IZWACEGOFN4K5AD3TIXN5WSXU7W2HKSMKCAQ6AYR7DS` | `4909f4928f15335154931fe0607d2527124b0202b95be9c1b845271390babcb0` | [View ↗](https://stellar.expert/explorer/testnet/tx/4909f4928f15335154931fe0607d2527124b0202b95be9c1b845271390babcb0) |
| 45 | `GCRCJOFJ35NNLQBAZ2I4THDRPTAQAVOAUPOVKGRBGKAJW36BQKADQ2MX` | `12a8093d2e85b77fa9270ee5a9c7b2e0cf08fb8e1e4f0b22f3000d46763489c0` | [View ↗](https://stellar.expert/explorer/testnet/tx/12a8093d2e85b77fa9270ee5a9c7b2e0cf08fb8e1e4f0b22f3000d46763489c0) |
| 46 | `GCONSSVBDEIUEWHS6ZXKUKNNVHZT4AMSFMGUZSZXQGOYVHJF3ZNUMXIH` | `229ac30ce038fa99015dd2a36cc2066ff474fb799a11f81faad84c93380f6eb2` | [View ↗](https://stellar.expert/explorer/testnet/tx/229ac30ce038fa99015dd2a36cc2066ff474fb799a11f81faad84c93380f6eb2) |
| 47 | `GBO4F4KMZUSAG4SXXB4FQWRATU5WVGMIK7VSDSGQCJMWZFWEYTO2YTTB` | `49e10de795dc5eb56599c01158ae84498e3d744df34069c0c4484b1ddba049f1` | [View ↗](https://stellar.expert/explorer/testnet/tx/49e10de795dc5eb56599c01158ae84498e3d744df34069c0c4484b1ddba049f1) |
| 48 | `GB2XSYTSBVB3UUBTIIT4FY4VB7TPSOLYEUJ3ZPAFEPFCSEARHPGMO6XX` | `95a3252c082cba98d7e9750965fa299416e37cd008f7b7d1ebdf8841b3306a0c` | [View ↗](https://stellar.expert/explorer/testnet/tx/95a3252c082cba98d7e9750965fa299416e37cd008f7b7d1ebdf8841b3306a0c) |
| 49 | `GDOBVQCYQZLG7W7MQGDIGFWKIOGOHBA6TWDXOZN36M62XZRPTXAL7P3L` | `b47eb83e9040fc2770b0f41c21f47eb2048498434df31fb69843363def19b77f` | [View ↗](https://stellar.expert/explorer/testnet/tx/b47eb83e9040fc2770b0f41c21f47eb2048498434df31fb69843363def19b77f) |
| 50 | `GD5JBJMCSLBGO2BVLI5AJFI4NCBADZLZUO5JZ3LP4JZEGXQWWFDLPBG2` | `9d42877a9fdd35bf441f24389de09bb0b542d0010ed209858074c53c7b63bcab` | [View ↗](https://stellar.expert/explorer/testnet/tx/9d42877a9fdd35bf441f24389de09bb0b542d0010ed209858074c53c7b63bcab) |
| 51 | `GDRXF3OEX5GOXSVNXQMHLCIJ7VVJENPPJ4VKEUVY3XWCP5S664MXBME3` | `99668f367d7ea6a0da7db2a266ff669607182c4837aa8e8e808552bab1ef06e7` | [View ↗](https://stellar.expert/explorer/testnet/tx/99668f367d7ea6a0da7db2a266ff669607182c4837aa8e8e808552bab1ef06e7) |
| 52 | `GC4IT6SBLODQKH334XQBDWXIPZKQOOUK7PYTAKL4HSAVCAZO77C23NVZ` | `701357f367dce1b7ec6d99044d76509d7493f853996d39c2fb6b8dbfaedfe2dd` | [View ↗](https://stellar.expert/explorer/testnet/tx/701357f367dce1b7ec6d99044d76509d7493f853996d39c2fb6b8dbfaedfe2dd) |
| 53 | `GAJZ636O7LRCDSZDKVVVBYOGPUO6X2F6GJFWVQS2UJUJYJZ4BNHFIXKQ` | `eb014ab25f908aeb83e62530c7aa0156147fb28cd02684afaa735c7f7f082e5c` | [View ↗](https://stellar.expert/explorer/testnet/tx/eb014ab25f908aeb83e62530c7aa0156147fb28cd02684afaa735c7f7f082e5c) |
| 54 | `GBYAEUCHZGBQTNNNIOJC3IHYVP4YFYVFU34UMS5RHGQXI5Q2TZG2DQTY` | `ae8a2cf47b7f5434803a32d60b7f1e116f3bbc7c706fbb2ad3e54fa56318de6b` | [View ↗](https://stellar.expert/explorer/testnet/tx/ae8a2cf47b7f5434803a32d60b7f1e116f3bbc7c706fbb2ad3e54fa56318de6b) |
| 55 | `GBORSNQSFNIXRM4UB5XY5QXLFQDQSRLO7QJD43FWSEENH7HBZKTAYDDO` | `63f0c291bcf37de19122b0f8c136cd191d98e73552260a87d200d843fef61dd5` | [View ↗](https://stellar.expert/explorer/testnet/tx/63f0c291bcf37de19122b0f8c136cd191d98e73552260a87d200d843fef61dd5) |
| 56 | `GBAZ7GPUILAUJVOFWEDAUPRU5V2SGJCDCYIVKTZKP5KY2WESXPGXQ543` | `07e6ffec13a4dc0f80067fcdab5c1188c5ccec0ae80628b395e215491dc99d8c` | [View ↗](https://stellar.expert/explorer/testnet/tx/07e6ffec13a4dc0f80067fcdab5c1188c5ccec0ae80628b395e215491dc99d8c) |
| 57 | `GBV4VA3APVU2RJSRDU7HCV5RFIPNHD4CCBKOKCGL2R5GXGEWLQW4KZ6K` | `79d3af6baea9d1119267010d7aad226570af8ac8c99d948e2dffc3f7c61f4270` | [View ↗](https://stellar.expert/explorer/testnet/tx/79d3af6baea9d1119267010d7aad226570af8ac8c99d948e2dffc3f7c61f4270) |
| 58 | `GCSF4IWL57GC7GLS2QTMKMJQRMCLQYNJIC6DEZBMK2QX7TUH4LE52MP7` | `fc938d8b9685fb782599ddc57afc96062fa8be6d5fc831bfc1feba3a24355f25` | [View ↗](https://stellar.expert/explorer/testnet/tx/fc938d8b9685fb782599ddc57afc96062fa8be6d5fc831bfc1feba3a24355f25) |
| 59 | `GC3BERRMQXXGP6CPE2PFW5RDMXRQFIJFISY2YVPKXWGUSJOZKCDXANLT` | `732546a895dffd2a55b8f937156f029368e004ff638698b9a2a23dd7ad449ea3` | [View ↗](https://stellar.expert/explorer/testnet/tx/732546a895dffd2a55b8f937156f029368e004ff638698b9a2a23dd7ad449ea3) |
| 60 | `GAIAONI6C7OCD7C5DSA5GZ52EEVAYTVZLGTZPMFZ5HOMOSQXTD5CEG4M` | `39fb9dcfac328330a2e49f4a1be27ec7a2157f2205ea898ac2d449d2652a0a72` | [View ↗](https://stellar.expert/explorer/testnet/tx/39fb9dcfac328330a2e49f4a1be27ec7a2157f2205ea898ac2d449d2652a0a72) |
| 61 | `GAN3SJEIA5CELFYKSFMFTDJWGVJNROPWNMJAZNACPSWNXXCOC3QYQABY` | `67de8d35979bea90127a54886af97d566f360b3ae1c4f9fd1375fbfb7148c74b` | [View ↗](https://stellar.expert/explorer/testnet/tx/67de8d35979bea90127a54886af97d566f360b3ae1c4f9fd1375fbfb7148c74b) |
| 62 | `GDA7EOV5MR3SKCZM5MSL5ESEHWEM3MA2QCKTQPAMRXBTVQ2M3CYPLJA6` | `6e581cbbd05e65c9468c7a47f3cd5a6f1e6c303efedc135d98452d5764865594` | [View ↗](https://stellar.expert/explorer/testnet/tx/6e581cbbd05e65c9468c7a47f3cd5a6f1e6c303efedc135d98452d5764865594) |
| 63 | `GA5VVXJOZHR7YZXVVALDYBEGYBUNBSBOBBXZZQN5Q2WQLCBS7UN3GOEG` | `19bedb6656f258cf13a856e8a06c92d49f0002430a1fe5480bebc9d2e8a5cf14` | [View ↗](https://stellar.expert/explorer/testnet/tx/19bedb6656f258cf13a856e8a06c92d49f0002430a1fe5480bebc9d2e8a5cf14) |
| 64 | `GCIKBUUSS6YF4ZVWQD7WBJ6EZ4UYQRSMOCI6PWLGTAPBFIT5P5HAVEFV` | `238bb6b51fc2582ff9d036559b6b8200a8216e03ac7994cc75813c9bd0330eec` | [View ↗](https://stellar.expert/explorer/testnet/tx/238bb6b51fc2582ff9d036559b6b8200a8216e03ac7994cc75813c9bd0330eec) |
| 65 | `GADP2X4LCF7ZS4RHBDAIBSNTJRRO4JVL3IAZY7RIMIWE2MXHFHPS6CVO` | `d53c1858dff94d5e4c3977136325ce6e0b89b72b8c55c964bafef166c9ef8200` | [View ↗](https://stellar.expert/explorer/testnet/tx/d53c1858dff94d5e4c3977136325ce6e0b89b72b8c55c964bafef166c9ef8200) |
---

## Repository Layout

```
Plexa(v1)/
├── contracts/                Soroban smart contracts (Rust workspace)
│   ├── group/                Group contract — one instance per ROSCA
│   ├── factory/              Factory — deploys groups + discovery + reputation
│   ├── oracle/               XLM/USDC price feed (admin-set for dev)
│   ├── swap/                 Soroswap-compatible mock router (testnet fallback)
│   └── Cargo.toml            Workspace manifest
├── frontend/                 React + TypeScript + Vite app
│   └── src/
│       ├── pages/            Landing · Groups · CreateGroup · GroupDetail · Dashboard · Profile
│       ├── components/       Header · GroupCard · WalletModal · PriceChart · …
│       ├── lib/              contracts.ts · wallet.ts · config.ts · demo.ts · notifications.ts
│       └── types.ts          Shared contract/UI types
├── scripts/                  build.sh · deploy.sh · test.sh
└── README.md
```

---

*** CI/CD pipeline supported ***


### Run the frontend

```bash
cd frontend
npm install
npm run dev            # → http://localhost:5173
```

Modes (set in `frontend/.env`):
- `VITE_DEMO=true` — **offline demo**: connects instantly as a simulated account, uses an
  in-memory + localStorage store. No wallet, no chain.
- `VITE_DEMO=false` — **real testnet**: signs with Freighter and submits to the deployed
  factory above. Fund your account and grab testnet USDC from Circle's faucet.

Build / typecheck:

```bash
npm run build          # tsc + vite build
npm run lint           # tsc --noEmit
```

---

## Building, Testing & Deploying Contracts

Toolchain: `rustc`/`cargo` 1.96+, `stellar-cli` 26+.

```bash
cd contracts

# Build deployable wasm (use the wasm32v1-none target — testnet rejects the
# reference-types/multivalue output of wasm32-unknown-unknown).
cargo build --target wasm32v1-none --release --offline

# Run unit tests
./scripts/test.sh      # or: cargo test   (see Windows note)
```

Deploy (see `scripts/deploy.sh`): upload the group wasm → deploy oracle, swap, and the
factory (with `--admin`, group wasm hash, USDC/XLM SACs, oracle, router) → call
`factory.create_group(...)`.

### ⚠️ Windows (GNU toolchain) test note
On `x86_64-pc-windows-gnu`, building the native `cdylib` for `cargo test` hits the PE
"export ordinal too large" linker limit (unrelated to contract code; the wasm build is
unaffected). Temporarily set `crate-type = ["rlib"]` in each crate's `Cargo.toml`, run
tests, then restore `["cdylib", "rlib"]`. `scripts/test.sh` automates this swap.

---

## Continuous Integration / Deployment

GitHub Actions workflows live in [`.github/workflows/`](.github/workflows):

| Workflow | Trigger | What it does |
|---|---|---|
| **`ci.yml`** | push / PR to `main` | **Contracts:** `cargo test` + Wasm build on Linux (fmt/clippy advisory). **Frontend:** `npm ci` → `npm run lint` (tsc) → `npm run build`. Uploads Wasm + `dist` artifacts. |
| **`deploy-contracts.yml`** | manual (`workflow_dispatch`) | Builds the `wasm32v1-none` artifacts and runs `scripts/deploy.sh` to deploy oracle + swap + factory. |

Notes:
- CI runs on **Linux**, which sidesteps the Windows-GNU `cdylib` linker limit — no
  crate-type swap needed, so `cargo test --workspace` runs directly.
- The deploy workflow is **manual-only** and needs a repository secret
  **`STELLAR_SECRET_KEY`** (the funded deployer's `S…` seed). It's gated behind a GitHub
  **Environment** matching the chosen network for approval control.

## Design Decisions

Choices explicitly surfaced rather than silently defaulted:

1. **Platform fee** — *removed for v1.* No fee is taken from any pot; re-introducing it is
   an isolated change in `resolve_period`.
2. **Collateral depletion** — *deduct + flag, group continues.* Uncovered defaults become
   on-chain debt (netted from future claims); the defaulter is flagged and payouts reflect
   what was actually collected.
3. **No-bid winner** — *fixed rotation (earliest eligible joiner).* Deterministic, so preflight and execution always agree and nobody can preview or reroll the outcome. Everyone wins exactly once regardless, so the fallback only decides order.
   not manipulation-proof against validators — acceptable for the low-value fallback
   (order only). Swap in commit-reveal / VRF later if needed.
4. **Reputation** — *count of cleanly-completed cycles*, held in the Factory registry,
   read at join time. A group's `min_reputation` gates joining; `0` disables the gate.

**Other flags:** the on-chain `history` Vec grows unbounded (fine for typical groups);
`resolve_period` is permissionless (a keeper bot can advance periods).

---

## Status & Roadmap

- [x] Group, Factory, Oracle, Swap contracts — built + unit tested
- [x] Per-group currency (USDC / XLM), multi-asset collateral, settlement window
- [x] Real Soroswap liquidation integration, verified end-to-end on testnet
- [x] Frontend (create wizard, dashboard, group view, governance, notifications) —
      typechecks + production build pass
- [x] Offline demo mode
- [x] Reflector oracle live on testnet (replaces the admin-set price feed)
- [x] End-to-end JS integration tests against deployed wasm (`e2e/e2e.mjs`)
- [ ] **Mainnet blockers still open:** no external security audit; admin key is a
      single signer (no multisig — a Stellar account-level change, not a contract
      one); mainnet Soroswap router address is unverified; history storage is a
      capped 200-entry buffer, not true paged/event-sourced storage (older entries
      are dropped, not archived); keeper bot is coded but not yet hosted anywhere

---
### Successfully ci/cd run for frontend and contract
<sub>Built on [Stellar](https://stellar.org) · [Soroban](https://soroban.stellar.org). Testnet only — not audited, not for real funds.</sub>
