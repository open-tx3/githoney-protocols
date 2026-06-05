# GitHoney Bounties

[GitHoney](https://githoney.io/) is a bounty platform for open-source work on Cardano. Maintainers fund a bounty against a GitHub issue, contributors get assigned to it, and once the work is merged the contributor claims the locked reward. GitHoney mediates the lifecycle as an on-chain admin and collects a configurable fee.

This tx3 covers the full protocol: the global **settings** registry, the contributor **badge** tokens, and the **bounty** lifecycle from creation through claim or close.

## Overview

State lives in UTxOs at the GitHoney Plutus validators:

- **Settings** — a single UTxO holding the protocol's `SettingsDatum`: the GitHoney fee-collection address, the flat `bounty_creation_fee` (lovelace paid on every new bounty), and the `bounty_reward_fee` (the share of the reward GitHoney takes on merge). Bounty transactions read it as a `reference` input; only the GitHoney operator can `deploy`, `update`, or `close` it. A one-shot mint backs the settings UTxO with an NFT so it can be located unambiguously.
- **Bounty** — one UTxO per bounty, holding the locked reward (a native token or ADA), an identifying bounty NFT, and a `GithoneyDatum` recording the admin payment credential, the maintainer address, the (optional) assigned contributor address, the reward fee, the deadline, and a `merged` flag. The bounty NFT is minted on create and burned on claim/close.
- **Badges** — `BadgesDatum` reference UTxOs (CIP-68 style: a reference NFT carries the metadata, fungible badge tokens circulate) used to recognize contributors and maintainers. Minted, re-pointed, transferred, and reclaimed independently of the bounty flow.

Three roles drive the contract:

- **Maintainer** — owns the repository/issue, creates and funds the bounty, and recovers funds if it is closed unassigned.
- **Contributor** — assigns themselves to a bounty and claims the reward once the work is merged.
- **Admin / GitHoney** — the protocol operator. The `Admin` key signs `merge` and the `close*` transactions; the `GitHoneyAddr` collects creation and reward fees.

A bounty moves `create → (add) → assign → merge → claim`, or is unwound at any earlier point with one of the `close*` transactions.

## Transactions

### Bounties

| Transaction | Description |
|---|---|
| `createWithLovelace` | Maintainer creates a bounty whose reward is ADA, mints the bounty NFT, and pays the creation fee to GitHoney |
| `createWithToken` | Same as above but the reward is a native token (`reward_policy_id` / `reward_asset_name`) plus the min-ADA floor |
| `add` | Sponsor adds more reward tokens to an existing bounty (`AddRewards` redeemer) |
| `assign` | Contributor assigns themselves to a bounty, topping up min-ADA and writing their address into the datum (`Assign` redeemer) |
| `assignBugBounty` | Maintainer assigns a reporter to a bug bounty — the maintainer pays the min-ADA, the reporter's credentials go into the datum |
| `merge` | Admin marks the bounty `merged`, takes the GitHoney reward fee, and returns the maintainer's min-ADA (`Merge` redeemer) |
| `claim` | Contributor claims the full reward and burns the bounty NFT (`Claim` redeemer) |
| `closeUnassigned` | Admin closes a never-assigned bounty, refunds the maintainer, burns the NFT (`Close` redeemer) |
| `closeUnassignedSponsored` | As above, also refunding a sponsor's added tokens |
| `closeAssigned` | Admin closes an assigned-but-unmerged bounty, refunding both maintainer and contributor min-ADA |
| `closeAssignedSponsored` | As above, also refunding a sponsor's added tokens |

### Settings

| Transaction | Description |
|---|---|
| `deploy` | Mint the settings NFT and publish the settings UTxO with the GitHoney address and fee schedule |
| `update` | Rewrite the settings datum — new address and/or fees (`UpdateSettings` redeemer) |
| `close` | Burn the settings NFT and tear down the settings UTxO (`CloseSettings` redeemer) |

### Badges

| Transaction | Description |
|---|---|
| `mintBadge` | Mint a CIP-68 badge pair — a reference NFT (metadata) to the badge script and the fungible badge tokens to the FT address |
| `updateBadge` | Rewrite a badge's on-chain metadata datum |
| `payBadgesTo` | Transfer a fungible badge token from the FT holding address to a recipient |
| `collectUtxos` | Admin reclaims a badge-script UTxO back to the GitHoney wallet |

## Important considerations

- **Settings is read, never spent, by bounty transactions.** Every bounty tx takes the settings UTxO as a `reference` input (`contract`). Because tx3 cannot read a referenced datum, the fee values (`bounty_creation_fee`, `bounty_rewards_fee`) are passed as explicit parameters and must match the on-chain settings datum, or the validator will reject the transaction.
- **Reward shape is fixed per transaction.** tx3 cannot iterate over a value map, so the reward asset is passed as a single explicit `(reward_policy_id, reward_asset_name, reward_amount)` triple. ADA-denominated bounties use `createWithLovelace`; native-token bounties use `createWithToken`. Multi-asset bounties are not modeled.
- **`initial_value` mirrors the locked value.** The `GithoneyDatum.initial_value` map records what the bounty UTxO holds so on-chain logic can verify it across the lifecycle. Callers must compute it to match the actual output value (reward + min-ADA), including the `"": { "": ... }` lovelace entry.
- **The admin signs the back half of the lifecycle.** `merge` and every `close*` transaction require the `Admin` signature and the `Close`/`Merge` redeemers — these are GitHoney-operated steps, not maintainer/contributor self-service.
- **Min-ADA accounting is explicit.** `assign`/`assignBugBounty` add a second min-ADA to the bounty UTxO (one for the maintainer, one for the contributor); `merge` and the `close*` flows return those min-ADAs to the right parties. The paying party differs: the contributor funds `assign`, the maintainer funds `assignBugBounty`.
- **Validity windows are required on bounty actions.** Create, add, assign, merge, claim, and close all take `since` / `until` slots; the `deadline` in the datum is enforced against this window by the validator.
- **Scripts and versions are passed in.** Validator and minting-policy scripts (`*_script` / `*_policy`) and their Plutus `*_version` are transaction parameters supplied via `cardano::plutus_witness`, so the same tx3 works across script revisions without being recompiled.

## Caller preparation

Several values must be prepared off-chain before invoking transactions.

### `createWithLovelace` / `createWithToken`

| Parameter | Source |
|---|---|
| `bounty_id: Bytes` | Unique token name for the bounty NFT, minted under `minting_policy_id`. |
| `admin_payment_key`, `maintainer_payment_key`, `maintainer_stake_key: Bytes` | Credentials written into the datum; the admin key gates later admin actions. |
| `bounty_creation_fee`, `bounty_rewards_fee: Int` | Read from the on-chain settings datum (passed in because the reference datum is not readable). |
| `reward_amount` (+ `reward_policy_id`, `reward_asset_name` for token) | The reward to lock. |
| `min_ada: Int` | Min-UTxO floor for the bounty output. |
| `settings_ref: UtxoRef` | The settings UTxO to reference. |
| `since`, `until`, `time_limit: Int` | Validity window and the bounty `deadline`. |

### `assign` / `assignBugBounty`

| Parameter | Source |
|---|---|
| `bounty_ref: UtxoRef` | The bounty UTxO being assigned. |
| `contributor_payment_credential`, `contributor_stake_credential: Bytes` | Credentials of the contributor/reporter, written into the datum. |
| `min_ada`, `initial_funds: Int` | The min-ADA top-up and the updated `initial_value` lovelace entry. |
| `settings_ref: UtxoRef`, `since`, `until: Int` | Settings reference and validity window. |

### `merge` / `claim`

| Parameter | Source |
|---|---|
| `bounty_ref: UtxoRef` | The bounty UTxO. |
| `githoney_fee`, `script_fee`, `min_ada`, `initial_funds: Int` (merge) | The GitHoney reward-fee cut, returned min-ADA, and updated `initial_value`. |
| `bounty_id`, `minting_policy_id: Bytes` (claim) | Name and policy of the bounty NFT to burn. |
| `reward_policy_id`, `reward_asset_name: Bytes` (merge) | The reward asset the GitHoney fee is taken from. |

### `close*`

| Parameter | Source |
|---|---|
| `bounty_id`, `minting_policy_id: Bytes` | The bounty NFT to burn. |
| `reward_amount`, `reward_policy_id`, `reward_asset_name` | Reward refunded to the maintainer. |
| `refundings_*` (sponsored variants) | Sponsor-added tokens refunded to the `Sponsor`. |
| `min_ada`, `settings_ref`, `since`, `until`, `time_limit` | Min-ADA, settings reference, and validity window. |

## References

- **Homepage:** [githoney.io](https://githoney.io/)
- **Source:** [githoney-io/worker](https://github.com/githoney-io/worker/tree/main/protocol)
