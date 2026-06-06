# GitHoney Setup

The **setup** protocol deploys and manages a [GitHoney](https://githoney.io/) *environment* on Cardano. An environment is a single **settings UTxO** that holds the protocol's fee schedule and GitHoney fee-collection address, and carries the GitHoney spend validator as a CIP-33 **reference script**.

This is the one-time admin/operator side of GitHoney. The day-to-day protocols — [`githoney-bounties`](../githoney-bounties/) and [`githoney-badges`](../githoney-badges/) — *operate against* an environment created here, selecting it through their own `settings_ref`. Deploy an environment with this protocol first; then point the operating protocols at its settings UTxO.

## Why this is separate

A trix **profile** describes one *deployed environment*. The deploy/update/close transactions here don't fit that model — they *create and mutate* an environment, so their inputs (the fee schedule, the GitHoney address, the seed UTxO) are operator choices, and the resulting `settings_ref` is an output, not a pre-existing constant. Keeping them in their own protocol means:

- this protocol's profiles describe the **compiled contract suite** for a network (validator/policy bytes + ids);
- the operating protocols' profiles describe a **deployed environment** (`settings_ref` + operating constants), without dragging in deploy-only values.

## Overview

The settings UTxO lives at the GitHoney validator (`Vault`) and is identified by a single **settings control token** (`settings_policy_id` / `settings_token_name`). It carries a `SettingsDatum` — the GitHoney address plus the `bounty_creation_fee` and `bounty_reward_fee`. The `deploy` transaction both publishes the GitHoney spend validator as a reference script (via `cardano::publish`) and mints the control token, so the same UTxO serves as the protocol's settings record *and* the reference script that bounty spends validate against.

## Transactions

| Transaction | Description |
|---|---|
| `deploy` | Mint the settings control token and publish the settings UTxO with the GitHoney address, fee schedule, and validator reference script |
| `update` | Rewrite the settings datum — new GitHoney address and/or fee schedule (`UpdateSettings` redeemer) |
| `close` | Burn the control token and tear down the settings UTxO, recovering its ADA (`CloseSettings` redeemer) |

## Important considerations

- **Deploy before operating.** `githoney-bounties` and `githoney-badges` reference the settings UTxO this protocol publishes (and bounty spends rely on its reference script). Run `deploy` first and feed the resulting UTxO ref to those protocols as `settings_ref`.
- **`deploy` is operator-driven.** The fee schedule (`creation_fee`, `reward_fee`) and GitHoney address credentials are parameters chosen at deploy time, not `env` values — they define the environment being created. `update` changes them later.
- **One control token per environment.** `deploy` mints a single NFT under `settings_policy_id` / `settings_token_name`; `update`/`close` locate the settings UTxO by that token. Multiple environments on one network use distinct token names.
- **GitHoney signs `update`/`close`.** Both spend the settings UTxO and require the `GitHoney` signature.

## Environment & profiles

The network-level contract suite lives in the `env { ... }` block of `main.tx3`,
populated per network from the active profile's `.env.<profile>` file:

| `env` value | Meaning |
|---|---|
| `githoney_script` / `githoney_script_version` | GitHoney spend validator (settings + bounty logic), published as the settings UTxO's reference script. |
| `settings_validator_script` / `settings_validator_version` | Settings spend validator that witnesses `update`/`close`. |
| `settings_minting_policy` / `settings_minting_version` | Settings minting policy that mints/burns the control token. |
| `settings_policy_id` / `settings_token_name` | Policy id and asset name of the settings control token. |

Only the `preprod` profile is configured (from the live GitHoney deployment); its
`settings_validator_script` is still a placeholder (those bytes aren't published).
Add a profile per network as new contract suites are deployed.

## Caller preparation

| Transaction | Parameter | Source |
|---|---|---|
| `deploy` | `creation_fee`, `reward_fee: Int` | The fee schedule to write into the settings datum. |
| `deploy` | `githoney_payment_credential`, `githoney_staking_credential: Bytes` | GitHoney fee-collection address credentials. |
| `deploy` | `utxo_ref: UtxoRef` | One-shot wallet UTxO consumed to make the settings mint unique. |
| `update` | `new_creation_fee`, `new_reward_fee: Int` | The new fee schedule. |
| `update` | `githoney_payment_key`, `githoney_staking_key: Bytes` | The new GitHoney address credentials. |
| `close` | `githoney_payment_credential`, `githoney_staking_credential: Bytes`, `remaining_ada: Int` | The recovering credentials and the lovelace held in the settings UTxO. |

## References

- **Homepage:** [githoney.io](https://githoney.io/)
- **Source:** [githoney-io/worker](https://github.com/githoney-io/worker/tree/main/protocol)
