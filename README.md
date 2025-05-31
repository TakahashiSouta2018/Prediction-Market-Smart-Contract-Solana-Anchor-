## Prediction Market Smart Contract (Solana, Anchor)

An on-chain prediction market built with Anchor on Solana. It allows:

- Create a market with YES/NO positions
- Mint the NO token and initialize the market’s YES token and accounts
- Trade (buy/sell) YES or NO with platform and LP fees
- Add and withdraw SOL liquidity
- Resolve a market and settle

Program ID (devnet): `5q1C8N47AYvLu7w6LKngwXhLjrZCZ5izMB8nbziZhYEV`


### Tech stack

- Anchor framework `0.30.x`
- Solana `1.18.x`
- TypeScript CLI using `@coral-xyz/anchor` and `@solana/web3.js`


## Repo structure

- `programs/prediction-market/` – Anchor program (Rust)
  - `src/lib.rs` – instruction entrypoints
  - `src/state/` – accounts: `Config`, `Market`, `UserInfo`, `Global`
  - `src/instructions/` – admin and market instructions
  - `src/events.rs` – program events
- `cli/` – TypeScript commands for local usage
- `lib/` – Transaction builders and utilities consumed by the CLI (referenced by `cli/scripts.ts`)
- `Anchor.toml` – Anchor configuration
- `package.json` – scripts for CLI and tests


## Features overview

- Admin/config
  - `configure` – set global `Config` (fees, team wallet, supply/decimals, min SOL liquidity)
  - `nominate_authority` / `accept_authority` – 2-step admin handover
- Market lifecycle
  - `mint_no_token` – mint NO token with metadata (symbol/URI)
  - `create_market` – create a market, initialize YES/NO mints and reserves
  - `swap` – buy/sell YES/NO with fees and slippage check (min receive)
  - `add_liquidity` – provide SOL liquidity
  - `withdraw_liquidity` – withdraw SOL liquidity
  - `resolution` – resolve (complete) market and settle


## Prerequisites

- Node.js 16+ and Yarn
- Rust and Solana toolchain
- Anchor CLI `0.30.1`
- A funded Solana keypair (devnet by default)


## Setup

1) Install dependencies

```bash
yarn install
```

2) Verify Anchor and Solana versions

```bash
anchor --version
solana --version
```

3) Configure devnet and fund wallet

```bash
solana config set --url https://api.devnet.solana.com
solana airdrop 2
```


## Build and deploy (program)

Anchor config is in `Anchor.toml` and pins the program ID on devnet. Typical flow:

```bash
# Clean and build
anchor build

# (optional) localnet
anchor test --skip-build

# Deploy to devnet if needed
anchor deploy
```

Note: This repo includes a convenience script in `Anchor.toml[scripts.build]` to prep `target/deploy` and copy a program keypair. Adjust if your keys differ.


## CLI usage (TypeScript)

All commands support cluster, rpc, and keypair flags:

- `-e, --env` – `mainnet-beta | testnet | devnet` (default `devnet`)
- `-r, --rpc` – custom RPC URL (default Anchor/cluster default)
- `-k, --keypair` – path to keypair JSON (default `./keys/EgBc...SNLv.json` in this repo; replace with yours)

Run commands via yarn:

```bash
# Configure global settings (admin-only)
yarn script config -e devnet -k ./keys/admin.json -r https://api.devnet.solana.com

# Create market (mints NO + initializes YES/NO market)
yarn script market -e devnet -k ./keys/admin.json -r https://api.devnet.solana.com

# Add LP SOL liquidity
yarn script addlp \
  -y <YES_TOKEN_MINT> -n <NO_TOKEN_MINT> -a <LAMPORTS> \
  -e devnet -k ./keys/admin.json -r https://api.devnet.solana.com

# Withdraw LP SOL liquidity
yarn script withdraw \
  -y <YES_TOKEN_MINT> -n <NO_TOKEN_MINT> -a <LAMPORTS> \
  -e devnet -k ./keys/admin.json -r https://api.devnet.solana.com

# Swap (trade) – style: 0 buy, 1 sell | tokenType: 0 NO, 1 YES
yarn script swap \
  -y <YES_TOKEN_MINT> -n <NO_TOKEN_MINT> \
  -a <AMOUNT> -s <0|1> -t <0|1> \
  -e devnet -k ./keys/admin.json -r https://api.devnet.solana.com

# Resolve market
yarn script resolution \
  -y <YES_TOKEN_MINT> -n <NO_TOKEN_MINT> \
  -e devnet -k ./keys/admin.json -r https://api.devnet.solana.com
```

Examples are also embedded at the bottom of `cli/command.ts`.


## Accounts and data structures

- `Config` (`programs/prediction-market/src/state/config.rs`)
  - Admin authority and pending authority
  - Team wallet
  - Platform fees: `platform_buy_fee`, `platform_sell_fee`
  - LP fees: `lp_buy_fee`, `lp_sell_fee`
  - Supply/decimals and initial reserve configs
  - `min_sol_liquidity`, `initialized`

- `Market` (`programs/prediction-market/src/state/market.rs`)
  - YES/NO token mints
  - Creator
  - Reserve tracking (YES/NO real reserves and totals)
  - Completion flags and optional start/end slots
  - LPs list and total LP amount

- `UserInfo` (`programs/prediction-market/src/state/market.rs`)
  - User’s YES/NO balances and LP status

- `Global` (`programs/prediction-market/src/state/global.rs`)
  - System-wide defaults, authority, team wallet, fees, and mint settings
  - Last updated slot for config freshness


## Instructions (entrypoints)

Defined in `src/lib.rs`:

- `configure(Config)` – admin-only, writes global `Config`
- `nominate_authority(Pubkey)` – admin nominates new authority
- `accept_authority()` – pending authority accepts
- `mint_no_token(no_symbol, no_uri)` – mints NO token and metadata
- `create_market(CreateMarketParams)` – creates market, configures YES/NO
- `swap(amount, direction, token_type, minimum_receive_amount)` – trade
- `resolution(yes_amount, no_amount, token_type, is_completed)` – resolve/settle
- `add_liquidity(amount)` – deposit SOL as LP
- `withdraw_liquidity(amount)` – withdraw SOL as LP


## Events

Emitted events include:

- `GlobalUpdateEvent` – global settings changed
- `CreateEvent` – market created and mints set
- `TradeEvent` – swap executed (buy/sell)
- `WithdrawEvent` – fees/withdraws
- `CompleteEvent` – resolution complete

See `programs/prediction-market/src/events.rs` for details.


## Testing

Mocha/ts-mocha are configured. Run:

```bash
yarn test
```

Anchor tests can also be run with:

```bash
anchor test
```


## Configuration

- `Anchor.toml` sets the devnet program ID and default provider settings.
- Override the CLI’s `--env/--rpc/--keypair` to target different clusters or wallets.


## Notes and caveats

- Ensure your wallet has enough SOL for fees on devnet.
- The CLI defaults to sample constants from `lib/constant` for symbols, URIs, and supplies where appropriate.
- Some on-chain math/paths are still in progress; treat this repo as a reference implementation and adapt for production.


## License

ISC


