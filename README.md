# moor — the Moor CLI

> Trade Moor's autonomous leveraged funds on Solana from your terminal: buy and sell 2× / 3× long and −1× / −2× short
> funds at Net Asset Value (NAV), with zero margin calls.

[![Latest Release](https://img.shields.io/github/v/release/moor-protocol/moor-cli-releases)](https://github.com/moor-protocol/moor-cli-releases/releases/latest)
[![Solana: Devnet](https://img.shields.io/badge/Solana-Devnet-blue)](https://explorer.solana.com/?cluster=devnet)
[![Platforms: macOS, Linux](https://img.shields.io/badge/platforms-macOS%20%7C%20Linux-lightgrey)](#install)

> **Devnet only.** moor trades on **Solana devnet** with test tokens: nothing you hold or trade here has real value.

Every trade is simulated before it is signed, and shown with its bounds and fees before you
confirm it. Every command can print one JSON envelope for scripts (`--json`).

## Install

```console
$ curl -fsSL https://github.com/moor-protocol/moor-cli-releases/releases/latest/download/install.sh | sh
```

The installer picks your platform, downloads the archive, checks it against the release's `SHA256SUMS` and puts `moor`
in `~/.local/bin` (it says so when that folder is not on your `PATH`). It needs no administrator rights and never
touches anything else. `MOOR_INSTALL_DIR=<dir>` installs elsewhere; `MOOR_VERSION=0.1.0` installs that version.

**Updating.** `moor update` replaces `moor` with the latest release, checked against its `SHA256SUMS` the same way;
`moor update --check` only says whether there is one. It never installs an older version, and a failed update leaves
the installed one untouched. Running the install line again also updates (moor 0.1.0 has no `moor update`, so update
from it that way once).

| Platform | Archive |
|---|---|
| macOS, Apple Silicon | `moor-<version>-darwin-arm64.tar.gz` |
| Linux x64 (glibc) | `moor-<version>-linux-x64.tar.gz` |
| Linux arm64 | `moor-<version>-linux-arm64.tar.gz` |
| Linux x64 (musl, e.g. Alpine) | `moor-<version>-linux-x64-musl.tar.gz` |

On Alpine and other musl systems, install the C++ runtime first: `apk add libstdc++ libgcc`. Intel Macs and Windows are not
supported yet.

**By hand.** Download your archive and `SHA256SUMS` from the [Releases](https://github.com/moor-protocol/moor-cli-releases/releases)
page, check it, unpack it and move `moor` onto your `PATH`:

```console
$ grep moor-0.1.1-darwin-arm64.tar.gz SHA256SUMS | shasum -a 256 -c -     # macOS
$ grep moor-0.1.1-linux-x64.tar.gz SHA256SUMS | sha256sum -c -            # Linux
$ tar -xzf moor-0.1.1-darwin-arm64.tar.gz && ./moor-0.1.1-darwin-arm64/moor --version
```

The binaries are not codesigned yet. On macOS, an archive downloaded with a browser is quarantined and `moor` will not
open; clear the flag once with `xattr -d com.apple.quarantine moor` (the installer above does not need this).

## Quick start

```console
$ moor login                       # first time: pick a wallet from ~/.config/solana, or create a new one
$ moor faucet                      # 1,000 test USDC (+0.03 SOL when the wallet has almost none)
$ moor funds ls                    # the funds you can buy
$ moor buy sol3x 25                # pay 25 USDC: the summary, then Confirm → Signing → Sending → Confirming
$ moor portfolio                   # your cash, positions and profit and loss
$ moor sell sol3x 50%              # sell half your moSOL3 for USDC
$ moor activity                    # what you did, newest first
```

`moor buy` and `moor sell` with nothing after them ask which fund and how much. A fund is named by its name (`sol3x`),
its ticker (`moSOL3`, any case) or its address; a typo gets "did you mean …?".

## Commands

| Command | What it does |
|---|---|
| `moor login` | sign in; the first time, choose one of your wallet files or create a new one |
| `moor logout` | sign out on this computer (`--all`: every session of the wallet) |
| `moor wallet` | the wallet in use; `ls` lists the wallet files, `use` switches, `new` creates one in `~/.config/solana/` |
| `moor buy` | buy a fund's shares, paying USDC unless `--pay` says otherwise (`--pay sol` pays the underlying) |
| `moor sell` | sell shares (a number, `50%` or `all`) for USDC unless `--receive` says otherwise; a settled fund's sell claims its payout |
| `moor portfolio` | your cash, your positions and their profit and loss; it corrects the Moor API's share records from the chain when they differ |
| `moor activity` | your trades, newest first; `moor activity <sig>` shows one, `--follow` finishes an interrupted one |
| `moor funds` | `ls` and `show`: prices, leverage and fees; `nav`, `rebalances`, `health` from the API |
| `moor faucet` | a devnet grant: 1,000 USDC, plus 0.03 SOL when the wallet has almost none |
| `moor autosign` | `on` or `off`: trades sign without asking (the summary and safety checks stay) |
| `moor update` | install the latest release of moor; `--check` only says whether there is one |

Global flags: `--json`, `-y, --yes`, `-v, --verbose`, `-q, --quiet`, `-k, --keypair <path>` (another wallet file for one
command), `--no-color`, `--version`, `-h, --help`.

## Wallets

- **Where.** Wallet files are Solana CLI keypair files (a JSON array of 64 bytes) in `~/.config/solana/`. `moor wallet ls`
  lists every one it finds there, with its address and SOL; only addresses are ever printed.
- **Choosing.** The first `moor login` asks which to use (the Solana CLI's `id.json` is offered first) and remembers the
  choice as `keypair` in `~/.config/moor/config.toml`. `moor wallet use <file or address>` switches; each wallet keeps its
  own sign-in. `-k <path>` or `MOOR_KEYPAIR` picks another for one command.
- **Creating.** `moor wallet new` (or "Create a new wallet" at login) writes `~/.config/solana/moor-<first 4 of the
  address>.json`, mode 0600. It never overwrites a file and never writes `id.json`, so a Solana CLI installed later does
  not mistake it for its own default. **Back the file up: anyone with a copy can spend from it.**
- The Solana CLI's own config is read, never written: switching in moor never switches the Solana CLI.

## Auto-sign and dry runs

- `moor autosign on` makes every trade sign without the "Sign and send?" question, as if `--yes` were passed. The summary
  still prints, and every safety check (stale prices, slippage, leverage, balances) still stops a trade. `moor autosign
  off` asks again; `moor autosign` shows the setting.
- `DRY_RUN=true moor buy sol3x 25` (or `MOOR_DRY_RUN`) makes every trade a dry run: built, checked and simulated, never
  signed or sent, with a `DRY RUN` line saying why. Any value but empty, `0`, `false`, `no` or `off` turns it on.
  `--dry-run` does the same for one command.

## Advanced

These still work and are left out of `moor --help`:

| Command | What it does |
|---|---|
| `moor mint`, `moor redeem` | buy and sell as M1 had them: the underlying route by default (`--pay usdc` for USDC); `--owner` with `--dry-run` previews for any address |
| `moor quote` | `quote mint` / `quote redeem`: the preview and balance check alone; nothing is built |
| `moor claim` | claim a settled fund's payout (`--payout underlying` or `receipt`) |
| `moor cash-receipts` | cash the Kamino receipts a claim paid into the fund's underlying |
| `moor balance` | a wallet's SOL, tokens and fund shares (`--owner` for any address, `--all` for unknown mints and empty accounts) |
| `moor tx`, `moor txs` | `activity` with and without a signature; `moor tx <sig> --follow` resumes a pending trade without signing again |
| `moor whoami` | the wallet, the endpoints and the cached session (`moor wallet` shows the same) |
| `moor reconcile` | ask the API to correct its recorded shares from the chain (`portfolio` does it for the funds that differ) |
| `moor config` | `get`, `set`, `unset` and `list` the CLI's settings, each value with its source |

Hidden flags: `-u, --url <moniker|url>`, `--api-url <url>` and `--wide` everywhere; on trades `--venue-slippage-bps`,
`--priority-fee`, `--cu-limit`, `--cu-margin-bps`, `--allow-danger`, `--allow-low-liquidity`, `--allow-partial`,
`--no-live`, `--allow-stale`, `--owner` and `--ignore-pending`.

## Output

Human output leads with a summary: a trade shows what you pay and receive, the fees, the slippage bounds, the leverage
and every check before it asks; `--verbose` (and every `--dry-run`) adds the bounds, shares, liquidity, accounts and rent
in full. Colour marks meaning only (green gains and passed checks, red losses and errors, yellow warnings) and is off
when the stream is not a terminal, with `--no-color` or `NO_COLOR`, and in `--json`; `FORCE_COLOR` forces it on. Spinners
and the sign → send → confirm line animate only on an interactive terminal; with `--quiet`, `--json`, `CI` set or
`TERM=dumb`, or when piped, the same lines print once with no animation.

## Config files

- **Keypair.** A Solana CLI keypair file (a JSON array of 64 bytes), from `-k`, `MOOR_KEYPAIR`, `keypair` in the moor
  config (what `moor login` and `moor wallet use` save) or `keypair_path` in the Solana CLI config, else
  `~/.config/solana/id.json`. Ledger and prompt signers are not
  supported yet. Only the public address is ever printed.
- **Files.** The CLI reads the Solana CLI config (`~/.config/solana/cli/config.yml`) but never writes it. Its own files live
  under `~/.config/moor/`: `config.toml` (`moor config set`), `sessions/` (the cached sign-in tokens) and `pending/` (the
  signed bytes of a trade whose outcome is not known yet). Files are mode 0600 in 0700 directories. The CLI owns only
  these paths, never the directory.
- **Precedence**, per value: **flags > environment > `~/.config/moor/config.toml` > `~/.config/solana/cli/config.yml` >
  defaults**. `moor config list` shows every value with its source.
- **Environment:** `MOOR_CLUSTER`, `MOOR_RPC_URL`, `MOOR_API_URL`, `MOOR_KEYPAIR`, `MOOR_OUTPUT` (`json`), `DRY_RUN` /
  `MOOR_DRY_RUN`,
  `MOOR_CONFIG_DIR` (another root for the CLI's files), `MOOR_SOLANA_CONFIG` (`none` ignores the Solana config), and
  `NO_COLOR` / `FORCE_COLOR`.
- **Devnet only.** `-u d` / `devnet` or an http(s) RPC URL; `-u m` (mainnet) and `-u l` (localhost) are refused. Every
  RPC is checked by its genesis hash, so a URL that is not devnet is refused before anything is read or sent.

## Scripting

With `--json`, stdout carries exactly one JSON document and a newline, for success and failure alike; progress and
warnings go to stderr, and nothing ever prompts (a trade needs `--yes`, `moor autosign on` or `--dry-run`).

```json
{ "ok": true,  "command": "funds ls", "data": { "funds": [] }, "meta": { "cluster": "devnet", "requestId": "moorcli-…" } }
{ "ok": false, "command": "mint", "error": { "code": "sdk.BalanceShortfall", "message": "…", "exit": 1, "details": { "balances": {} } } }
```

`error.details` carries typed fields where they help: the signature, the pending file and the resume command of an
unknown outcome, the stage of a refusal (`local`, `simulation`, `preflight`, `landed`), a decoded program error, the
preview, the balance check. Amounts are `{ "raw": "1000000", "ui": "1", "decimals": 6, "symbol": "USDC", "mint": "…" }`
with strings, never JSON numbers.

| Exit | Meaning |
|---|---|
| 0 | success |
| 1 | input, validation or configuration error (also an internal bug, reported as `cli.Internal`) |
| 2 | rejected or failed on chain; with `details.stage "simulation"` nothing was sent and no fee was paid |
| 3 | the Moor API or the RPC is unavailable (retry later) |
| 4 | the outcome is unknown: the transaction may still land; a pending file was kept (resume with `moor tx <sig> --follow`) |
| 5 | sign-in required or failed |
| 130 | interrupted (Ctrl-C, SIGTERM, SIGHUP) |

## Safety

- Every trade is simulated before signing and shows its slippage bounds, fees, guards and balance check before the confirm.
  `--dry-run` builds and simulates and stops there; `--yes` skips the confirm.
- Before the first send, the signed bytes are saved to `~/.config/moor/pending/<signature>.json`. If the outcome is
  unknown (exit 4), `moor tx <sig> --follow` resumes it from that file and **never signs again**. A new trade of the same
  wallet and fund is refused until the earlier one is resolved. `--ignore-pending` overrides that refusal and can
  double the trade.
- No secret is ever printed: no keypair bytes, no session token, and the RPC URL only as `scheme://host/…`, because a
  provider URL often embeds an access key.

## Uninstall

```console
$ rm "$(command -v moor)"
```

`~/.config/moor/` holds your settings, sign-ins and any pending trade; delete it too if you are done. Your wallet files
in `~/.config/solana/` are not moor's to remove: **back them up before you delete anything there**.

## Reporting problems

Open an issue in this repository with what you ran, `moor --version --json`, and the failing command's output with
`--json`. Never paste a wallet file or its contents: moor never prints a secret, so nothing it shows needs one.

## Licence

moor is not open source (`UNLICENSED`): © 2026 Moor Protocol, free to download and run; the source is not public. Each
archive's `THIRD_PARTY_NOTICES.md` lists every package the binary bundles with its licence text, and the Bun runtime it
embeds.
