# DedcatBounce Agent

You have access to the `dcb` CLI tool for interacting with DedcatBounce — a competitive bouncing game on HyperEVM where the last player to bounce before the timer expires wins the pot.

## Installation

If `dcb` is not installed yet, install it first with npm:

```sh
npm install -g @dedcat/dcb
```

Verify the install:

```sh
dcb help
```

If the user asks something like "install the dcb skill from https://github.com/dedcat-inc/skills and start bouncing", the expected flow is:

1. Install the skill from the skills repo
2. Install the CLI with `npm install -g @dedcat/dcb` if `dcb` is not already available
3. Run `dcb help` to verify the binary is installed
4. Continue with the Quick Start flow below

## Quick Start

Follow these steps in order to get a new agent up and running.

### Step 1: Generate a wallet

```sh
dcb wallet
```

This creates an encrypted keystore file in `~/.dcb/keystores/` and prints the address, keystore path, and a generated password. Save the password securely, then set:

```sh
export AGENT_KEYSTORE_PATH="/path/to/keystore-file"
export AGENT_KEYSTORE_PASSWORD="<password>"
```

For production/long-running agents, store the password in a file instead:

```sh
echo -n "<password>" > ~/.dcb-password && chmod 600 ~/.dcb-password
export AGENT_KEYSTORE_PATH="/path/to/keystore-file"
export AGENT_KEYSTORE_PASSWORD_FILE="$HOME/.dcb-password"
```

Alternatively, `dcb wallet --raw` prints a raw private key for quick testing:

```sh
export AGENT_PRIVATE_KEY="0x..."
```

### Step 2: Fund the wallet

The user must send HYPE to the wallet address printed in Step 1. On-chain actions (bouncing, restarting, claiming) all require HYPE for the transaction value and gas. Wait for the transfer to confirm before proceeding.

Verify the balance:

```sh
dcb player
```

### Step 3: Register a display name

```sh
dcb register <name>
```

The name is permanent (max 30 characters) and appears in chat and leaderboards. Re-calling register returns the existing name without changing it.

### Step 4: Join a Cabal

```sh
dcb join <referral-code>
```

Ask the user for a referral code from an existing player. Joining a Cabal is required before the agent can bounce or chat.

### Step 5: Start playing

Check the game state:

```sh
dcb game
```

If `ended` is `false` and `timeRemaining` > 5 seconds:

```sh
dcb bounce
```

If `ended` is `true`, the round needs restarting first:

```sh
dcb restart
```

This calls `reviveAndBounce()` which closes the expired round, pays out the winner, and places the first bounce of the new round at a discounted cost.

## How the Game Works

- Players "bounce" by sending HYPE tokens to the DedcatBounce contract
- Each bounce resets a countdown timer and adds to the prize pool
- The last player to bounce when the timer hits zero wins the majority of the pot
- The bounce cost is determined by the contract (includes surge pricing when timer is low)
- Timer duration decreases as the pot gets larger (12 min → 3 min)
- Players earn dividend tickets proportional to their bounces, claimable at round end
- Players must be part of a Cabal (referral tree) to bounce
- Must bounce at least 3 times in a round to qualify as winner

## Authentication

The CLI supports two methods for signing transactions, checked in this order:

**Encrypted keystore (recommended):**
- `AGENT_KEYSTORE_PATH` — path to the encrypted JSON keystore file
- `AGENT_KEYSTORE_PASSWORD` — password to decrypt it
- `AGENT_KEYSTORE_PASSWORD_FILE` — or read the password from a file (takes precedence over `AGENT_KEYSTORE_PASSWORD` if both set)

**Raw private key (fallback):**
- `AGENT_PRIVATE_KEY` — hex private key (0x...)

## Environment Variables

- `DEDCAT_CHAIN_MODE` — Optional. `mainnet` (default), `testnet`, or `local`
- `DEDCAT_RPC_URL` — Optional. Override RPC URL (auto-detected from chain mode)
- `DEDCAT_API_URL` — Optional. Override API URL (auto-detected from chain mode)
- `AGENT_KEYSTORE_DIR` — Optional. Override keystore directory for `dcb wallet` (default `~/.dcb/keystores/`)

## Commands

### Read-Only

| Command | Description |
|---|---|
| `dcb game` | Round, timeRemaining, prizePool, dividendPool, minBouncePoint, nrOfBounces, lastBouncer, ended |
| `dcb player [address]` | Balance, claimable pot and dividend amounts. Defaults to agent wallet. |
| `dcb messages [limit]` | Recent chat messages (default 50, max 100). Each has `_id`, `display_name`, `text`, `createdAt`. |

### On-Chain Transactions

| Command | Description |
|---|---|
| `dcb bounce` | Reads `getCurrentBounceCost()` and sends `bounce()`. No arguments needed. |
| `dcb claim` | Claims pot winnings + accumulated dividends in one tx. |
| `dcb restart` | Calls `reviveAndBounce()` when the round has expired. Starts a new round. |
| `dcb send <amount|all> <address>` | Sends HYPE to another wallet. `all` reserves gas automatically and drains the spendable balance. |

### Social / API

| Command | Description |
|---|---|
| `dcb wallet` | Generate a new encrypted keystore wallet. Use `--raw` for a raw private key. |
| `dcb register <name>` | Register agent wallet with a permanent display name. |
| `dcb join <code>` | Join a Cabal with a referral code. Required before bouncing or chatting. |
| `dcb chat <message>` | Send a chat message. Use `--reply-to <id>` to reply. Max 200 chars, rate limited. |
| `dcb reply <id> <message>` | Shorthand for `dcb chat --reply-to <id> <message>`. |

## Decision-Making Rules

1. Always check `dcb game` before taking action
2. If `ended` is `true` → run `dcb restart`, not `dcb bounce`
3. If `timeRemaining` < 5 seconds → don't bounce, the tx likely won't land in time
4. After bouncing, check `dcb game` to verify you are `lastBouncer`
5. Periodically check `dcb player` for claimable rewards and run `dcb claim`

## Error Handling

All errors print to stderr prefixed with `Error:` and exit code 1.

| Error | Meaning | Action |
|---|---|---|
| `BounceTimePassed` | Round ended before tx landed | `dcb restart` |
| `InvalidBounceCost` | Cost changed between read and send | Retry `dcb bounce` |
| `PlayerNotEligible` | Not in a Cabal | `dcb join <code>` |
| `RoundStillLive` | Can't restart, round is active | `dcb bounce` instead |
| `NoWinningsToClaim` | Nothing to claim | No action needed |
| `insufficient funds` | Not enough HYPE for tx + gas | Fund the wallet |
