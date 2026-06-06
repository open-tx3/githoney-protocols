# GitHoney Badges

[GitHoney](https://githoney.io/) issues **badges** — CIP-68 tokens that recognize contributors and maintainers across the platform. This tx3 covers the badge lifecycle: minting a badge, updating its on-chain metadata, transferring a badge token to a recipient, and reclaiming badge UTxOs.

It is a companion to [`githoney-bounties`](../githoney-bounties/): the two are independent tx3 protocols that share one on-chain artifact — the GitHoney **settings UTxO**. Badge spends reference that UTxO for protocol authorization, so `settings_ref` here must point at the same settings UTxO published by `githoney-setup`'s `deploy`.

## Overview

A badge follows the CIP-68 split-token convention:

- a **reference NFT** carries the metadata datum (`BadgeDatum` — a `name`/`logo`/`description` map plus a version) and lives at the **badge validator** (`Registry`);
- **fungible badge tokens** circulate from the **badge-token wallet** (`Distributor`) to recipients.

The `GitHoney` wallet operates the badges: it funds mints, signs metadata updates, and reclaims badge-script UTxOs.

## Transactions

| Transaction | Description |
|---|---|
| `mint_badge` | Mint a CIP-68 badge pair — the reference NFT (metadata) to the badge script and the fungible badge tokens to the `Distributor` wallet |
| `update_badge` | Rewrite a badge's on-chain metadata datum, keeping the reference NFT in place |
| `pay_badges_to` | Transfer a single fungible badge token from the `Distributor` wallet to a recipient |
| `collect_badge_utxos` | Reclaim a UTxO sitting at the badge script back into the GitHoney wallet |

## Important considerations

- **Badges depend on the settings deployment.** `update_badge` and `collect_badge_utxos` take the GitHoney settings UTxO as a read-only `reference` input (via `env.settings_ref`) for on-chain authorization. That UTxO is published by `githoney-setup` — deploy there first. `mint_badge` and `pay_badges_to` do not touch settings.
- **CIP-68 split tokens.** `mint_badge` produces two assets under the same `badge_policy_id`: one reference NFT (`ref_nft_asset_name`, held at the badge script with the metadata datum) and `ft_badge_amount` fungible tokens (`ft_badge_name`, sent to the `Distributor` wallet).
- **Metadata is caller-encoded.** The `name`/`logo`/`description` keys and their `*_value` byte strings are passed as parameters and written verbatim into the `BadgeDatum` metadata map.
- **GitHoney signs the operator actions.** `update_badge` and `collect_badge_utxos` require the `GitHoney` signature; `pay_badges_to` is a plain wallet-to-wallet token transfer.

## Environment & profiles

The network-level deployment config lives in the `env { ... }` block of `main.tx3`,
populated per network from the active profile's `.env.<profile>` file:

| `env` value | Meaning |
|---|---|
| `settings_ref` | Published GitHoney settings UTxO referenced by `update_badge` / `collect_badge_utxos`. Must match the `githoney-setup` deployment. |
| `badges_script` / `badges_script_version` | Badge spend validator and its Plutus version. |
| `badge_policy_script` / `badge_policy_script_version` | Badge minting policy and its Plutus version. |
| `badge_policy_id` | Minting policy id of the badge tokens. |

Only the `preprod` profile is configured (from the live GitHoney deployment). Add a
profile per network as new environments are deployed.

## Caller preparation

Per-call parameters to prepare off-chain:

### `mint_badge`

| Parameter | Source |
|---|---|
| `name`/`logo`/`description` + `*_value: Bytes` | CIP-68 metadata keys and their values. |
| `m_version: Int` | Metadata version written into the datum. |
| `ref_nft_asset_name`, `ft_badge_name: Bytes` | Asset names of the reference NFT and the fungible badge token. |
| `ft_badge_amount: Int` | Number of fungible badge tokens to mint. |
| `utxo_ref: UtxoRef` | GitHoney wallet UTxO consumed to fund and back the mint. |

### `update_badge`

| Parameter | Source |
|---|---|
| `name`/`logo`/`description` + `*_value: Bytes`, `m_version: Int` | The new metadata to write. |
| `badge_utxo_ref: UtxoRef` | The badge reference-NFT UTxO being updated. |

### `pay_badges_to`

| Parameter | Source |
|---|---|
| `badge_name`, `badge_policy: Bytes` | Asset name and policy id of the badge token to transfer. |

### `collect_badge_utxos`

| Parameter | Source |
|---|---|
| `utxo_to_collect: UtxoRef` | The badge-script UTxO to reclaim. |

## References

- **Homepage:** [githoney.io](https://githoney.io/)
- **Source:** [githoney-io/worker](https://github.com/githoney-io/worker/tree/main/protocol)
