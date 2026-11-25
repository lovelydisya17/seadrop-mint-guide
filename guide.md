# SeaDrop mintSigned, how it works

SeaDrop is a shared drop contract. Collections that use it do not write their own mint function, they configure stages on SeaDrop and let it mint for them.

## The moving parts

- **SeaDrop contract** — one shared contract per chain. On Robinhood chain the well-known deployment is `0x00005EA00Ac477B1030CE78506496e8C2dE24bf5`.
- **Collection contract** — the NFT contract itself, created through SeaDrop. It holds the stages.
- **Stage** — a time window with its own rules: start time, end time, mint price, max supply for the stage, per-wallet limit, and an allowlist root for GTD/FCFS stages.
- **MintParams** — the struct the minter sends: `mintRecipient`, `minter`, `quantity`, `feeRecipient`, `feeBps`, `signature`, and more.
- **Signature** — the collection owner signs the stage parameters. The signature is only valid for the stage it was made for.

## The flow

1. The minter reads the open stage from the collection contract: stage index, start and end time, price, and the stage uuid.
2. The minter fetches a signature for that stage from the project's drop API. The API only returns a signature for the stage that is currently open.
3. The minter builds the `mintSigned` call with the MintParams and the signature, then simulates it.
4. If the simulation passes, the transaction is sent. If it reverts, nothing is sent.

## Things that go wrong

- **Wrong stage** — fetching a signature before the target stage opens returns data for the wrong stage. Always fetch after the stage opens and check the uuid and times match.
- **Per-wallet limit is total, not per phase** — some collections set the limit across all stages. A wallet that minted 2 in the GTD stage may already be at its limit for FCFS.
- **Site says FREE, contract charges** — always read the price from the stage on-chain. The website text is not the price.
- **Fee** — SeaDrop takes a fee through `feeRecipient` and `feeBps` (for example 1000 bps = 10% of the mint price). On a free mint the fee is 0.

## Rules

- Simulate before sending. Never hand-type calldata or amounts, generate them from the ABI.
- Check and revoke approvals after the mint.
- A free mint still costs gas. Gas is the only cost on a price-0 mint.
