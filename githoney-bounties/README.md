# GitHoney Bounties

[GitHoney](https://githoney.io/) is a bounty platform for open-source work on Cardano. Maintainers fund a bounty against a GitHub issue, contributors get assigned to it, and once the work is merged the contributor claims the locked reward. GitHoney mediates the lifecycle as an on-chain admin and collects a configurable fee.

This tx3 covers the global **settings** registry and the **bounty** lifecycle from creation through claim or close. The contributor/maintainer **badge** tokens live in the companion [`githoney-badges`](../githoney-badges/) protocol.

## Overview

State lives in UTxOs at the GitHoney Plutus validators:

- **Settings** — a single UTxO holding the protocol's `SettingsDatum`: the GitHoney fee-collection address, the flat `bounty_creation_fee` (lovelace paid on every new bounty), and the `bounty_reward_fee` (the share of the reward GitHoney takes on merge). Bounty transactions read it as a `reference` input; only the GitHoney operator can `deploy_settings`, `update_settings`, or `close_settings` it. A one-shot mint backs the settings UTxO with an NFT so it can be located unambiguously.
- **Bounty** — one UTxO per bounty, holding the locked reward (a native token or ADA), an identifying bounty NFT, and a `GithoneyDatum` recording the admin payment credential, the maintainer address, the (optional) assigned contributor address, the reward fee, the deadline, and a `merged` flag. The bounty NFT is minted on create and burned on claim/close.

Three roles drive the contract:

- **Maintainer** — owns the repository/issue, creates and funds the bounty, and recovers funds if it is closed unassigned.
- **Contributor** — assigns themselves to a bounty and claims the reward once the work is merged.
- **Admin / GitHoney** — the protocol operator. The `Admin` key signs `merge_bounty` and the `close_bounty_*` transactions; the `GitHoneyAddr` collects creation and reward fees.

A bounty moves `create_bounty → (add_bounty_rewards) → assign_bounty → merge_bounty → claim_bounty`, or is unwound at any earlier point with one of the `close_bounty_*` transactions.

## Transactions

### Bounties

| Transaction | Description |
|---|---|
| `create_bounty_with_lovelace` | Maintainer creates a bounty whose reward is ADA, mints the bounty NFT, and pays the creation fee to GitHoney |
| `create_bounty_with_token` | Same as above but the reward is a native token (`reward_policy_id` / `reward_asset_name`) plus the min-ADA floor |
| `add_bounty_rewards` | Sponsor adds more reward tokens to an existing bounty (`AddRewards` redeemer) |
| `assign_bounty` | Contributor assigns themselves to a bounty, topping up min-ADA and writing their address into the datum (`Assign` redeemer) |
| `assign_bug_bounty` | Maintainer assigns a reporter to a bug bounty — the maintainer pays the min-ADA, the reporter's credentials go into the datum |
| `merge_bounty` | Admin marks the bounty `merged`, takes the GitHoney reward fee, and returns the maintainer's min-ADA (`Merge` redeemer) |
| `claim_bounty` | Contributor claims the full reward and burns the bounty NFT (`Claim` redeemer) |
| `close_bounty_unassigned` | Admin closes a never-assigned bounty, refunds the maintainer, burns the NFT (`Close` redeemer) |
| `close_bounty_unassigned_sponsored` | As above, also refunding a sponsor's added tokens |
| `close_bounty_assigned` | Admin closes an assigned-but-unmerged bounty, refunding both maintainer and contributor min-ADA |
| `close_bounty_assigned_sponsored` | As above, also refunding a sponsor's added tokens |

### Settings

| Transaction | Description |
|---|---|
| `deploy_settings` | Mint the settings NFT and publish the settings UTxO with the GitHoney address and fee schedule |
| `update_settings` | Rewrite the settings datum — new address and/or fees (`UpdateSettings` redeemer) |
| `close_settings` | Burn the settings NFT and tear down the settings UTxO (`CloseSettings` redeemer) |

## Important considerations

- **Settings is read, never spent, by bounty transactions.** Every bounty tx takes the settings UTxO as a `reference` input (`contract`), located via the network-level `settings_ref` from the active profile. Because tx3 cannot read a referenced datum, the fee values (`bounty_creation_fee`, `bounty_rewards_fee`) are supplied from `env` and must match the on-chain settings datum, or the validator will reject the transaction.
- **Reward shape is fixed per transaction.** tx3 cannot iterate over a value map, so the reward asset is passed as a single explicit `(reward_policy_id, reward_asset_name, reward_amount)` triple. ADA-denominated bounties use `create_bounty_with_lovelace`; native-token bounties use `create_bounty_with_token`. Multi-asset bounties are not modeled.
- **`initial_value` mirrors the locked value.** The `GithoneyDatum.initial_value` map records what the bounty UTxO holds so on-chain logic can verify it across the lifecycle. Callers must compute it to match the actual output value (reward + min-ADA), including the `"": { "": ... }` lovelace entry.
- **The admin signs the back half of the lifecycle.** `merge_bounty` and every `close_bounty_*` transaction require the `Admin` signature and the `Close`/`Merge` redeemers — these are GitHoney-operated steps, not maintainer/contributor self-service.
- **Min-ADA accounting is explicit.** `assign_bounty`/`assign_bug_bounty` add a second min-ADA to the bounty UTxO (one for the maintainer, one for the contributor); `merge_bounty` and the `close_bounty_*` flows return those min-ADAs to the right parties. The paying party differs: the contributor funds `assign_bounty`, the maintainer funds `assign_bug_bounty`.
- **Validity windows are required on bounty actions.** Create, add, assign, merge, claim, and close all take `since` / `until` slots; the `deadline` in the datum is enforced against this window by the validator.
- **Scripts, policy ids, and the fee schedule live in `env`.** The validator and minting-policy scripts and their Plutus versions, the settings control token + `settings_ref`, the bounty policy id, the GitHoney admin credential, and the fee schedule are network-level `env` values loaded from the active profile's `.env.<profile>` file (see [Environment & profiles](#environment--profiles)) — not per-call parameters.

## Environment & profiles

The network-level deployment config lives in the `env { ... }` block of `main.tx3`,
populated per network from the active profile's `.env.<profile>` file
(`local` / `preview` / `preprod` / `mainnet`). These are constant for a deployment,
so they are not passed per call:

| `env` value | Meaning |
|---|---|
| `githoney_script` / `githoney_script_version` | GitHoney spend validator (settings + bounty logic) and its Plutus version. |
| `settings_validator_script` / `settings_validator_version` | Settings spend validator and version. |
| `settings_minting_policy` / `settings_minting_version` | Settings minting policy and version. |
| `settings_policy_id` / `settings_token_name` | Settings control token policy id and asset name. |
| `settings_ref` | Published settings UTxO referenced by every bounty tx. |
| `admin_payment_key` | GitHoney admin credential recorded in every bounty datum. |
| `bounty_creation_fee` / `bounty_rewards_fee` | Fee schedule, mirroring the settings datum. |
| `bounty_policy_id` | Bounty NFT minting policy id. |

The committed `.env.*` files hold **placeholders** — fill them from a deployed GitHoney
`config.json` before invoking against a real network.

## Caller preparation

Beyond the `env` values above, these per-call parameters must be prepared off-chain before invoking.

### `create_bounty_with_lovelace` / `create_bounty_with_token`

| Parameter | Source |
|---|---|
| `bounty_id: Bytes` | Unique token name for the bounty NFT (minted under `env.bounty_policy_id`). |
| `maintainer_payment_key`, `maintainer_stake_key: Bytes` | Maintainer credentials written into the datum. |
| `reward_amount` (+ `reward_policy_id`, `reward_asset_name` for token) | The reward to lock. |
| `min_ada: Int` | Min-UTxO floor for the bounty output. |
| `since`, `until`, `time_limit: Int` | Validity window and the bounty `deadline`. |

### `assign_bounty` / `assign_bug_bounty`

| Parameter | Source |
|---|---|
| `bounty_ref: UtxoRef` | The bounty UTxO being assigned. |
| `contributor_payment_credential`, `contributor_stake_credential: Bytes` | Credentials of the contributor/reporter, written into the datum. |
| `min_ada`, `initial_funds: Int` | The min-ADA top-up and the updated `initial_value` lovelace entry. |
| `since`, `until: Int` | Validity window. |

### `merge_bounty` / `claim_bounty`

| Parameter | Source |
|---|---|
| `bounty_ref: UtxoRef` | The bounty UTxO. |
| `githoney_fee`, `script_fee`, `min_ada`, `initial_funds: Int` (merge) | The GitHoney reward-fee cut, returned min-ADA, and updated `initial_value`. |
| `bounty_id: Bytes` (claim) | Name of the bounty NFT to burn. |
| `reward_policy_id`, `reward_asset_name: Bytes` (merge) | The reward asset the GitHoney fee is taken from. |

### `close_bounty_*`

| Parameter | Source |
|---|---|
| `bounty_id: Bytes` | The bounty NFT to burn. |
| `reward_amount`, `reward_policy_id`, `reward_asset_name` | Reward refunded to the maintainer. |
| `refundings_*` (sponsored variants) | Sponsor-added tokens refunded to the `Sponsor`. |
| `min_ada`, `since`, `until`, `time_limit` | Min-ADA and validity window. |

## References

- **Homepage:** [githoney.io](https://githoney.io/)
- **Source:** [githoney-io/worker](https://github.com/githoney-io/worker/tree/main/protocol)
