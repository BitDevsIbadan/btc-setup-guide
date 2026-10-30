# BitDevs Ibadan: Regtest Workshop

Hands-on session from **BitDevs Ibadan, Edition 3** (October 31, 2026).

In about 45 minutes you'll run a private Bitcoin network on your laptop, mine blocks, send coins between two wallets, decode the transaction you made, and write a message on-chain with `OP_RETURN`.

## What is regtest?

Regtest (regression test mode) is built into Bitcoin Core. It's a blockchain that lives only on your machine:

- Coins are free. You mine them yourself.
- A new block appears whenever you type a command, not every 10 minutes.
- Nothing touches real bitcoin, and there's nothing to download or sync. It works offline.

## Contents

1. [Install Bitcoin Core](#1-install-bitcoin-core)
2. [Start your network](#2-start-your-network)
3. [Make wallets and mine coins](#3-make-wallets-and-mine-coins)
4. [Send coins and watch them confirm](#4-send-coins-and-watch-them-confirm)
5. [Decode the transaction](#5-decode-the-transaction)
6. [Write a message on-chain with OP_RETURN](#6-write-a-message-on-chain-with-op_return)
7. [No terminal? Use the app console](#7-no-terminal-use-the-bitcoin-core-app-console)
8. [Troubleshooting](#8-troubleshooting)

## Requirements

- A laptop running Windows (with WSL), macOS or Linux
- A terminal
- About 100 MB of free space

---

## 1. Install Bitcoin Core

We only need the command-line tools: `bitcoind` (the node) and `bitcoin-cli` (how you talk to it).

> Installing does **not** download the blockchain. Just never start Bitcoin Core without regtest mode, and it never will.

Check [bitcoincore.org/en/download](https://bitcoincore.org/en/download/) for the latest version and set `VERSION` to match. Any version from 28 up works for this workshop.

### Windows: set up WSL first

WSL gives you a Linux terminal inside Windows, so you can follow the same commands as everyone else.

1. Open PowerShell as Administrator and run `wsl --install`
2. Restart your laptop.
3. Open **Ubuntu** from the Start menu and create a username and password.
4. Follow the Linux steps below inside Ubuntu.

### Linux and WSL

```bash
VERSION=32.1
ARCH=$(uname -m)   # x86_64 on most laptops, aarch64 on ARM
cd ~
wget https://bitcoincore.org/bin/bitcoin-core-$VERSION/bitcoin-$VERSION-$ARCH-linux-gnu.tar.gz
tar -xzf bitcoin-$VERSION-$ARCH-linux-gnu.tar.gz
sudo install -m 0755 -t /usr/local/bin bitcoin-$VERSION/bin/*
bitcoind --version
```

<details>
<summary>Prefer downloading from the website?</summary>

Download **Linux (tgz)** from [bitcoincore.org/en/download](https://bitcoincore.org/en/download/). On WSL it lands in your Windows Downloads folder, which Ubuntu can reach:

```bash
cd /mnt/c/Users/YOUR-WINDOWS-NAME/Downloads   # on plain Ubuntu: cd ~/Downloads
tar -xzf bitcoin-*-linux-gnu.tar.gz
sudo install -m 0755 -t /usr/local/bin bitcoin-*/bin/*
bitcoind --version
```

</details>

### macOS

With [Homebrew](https://brew.sh):

```bash
brew install bitcoin
bitcoind --version
```

Without Homebrew:

```bash
VERSION=32.1
ARCH=$(uname -m)   # arm64 on Apple Silicon, x86_64 on Intel
cd ~
curl -LO https://bitcoincore.org/bin/bitcoin-core-$VERSION/bitcoin-$VERSION-$ARCH-apple-darwin.tar.gz
tar -xzf bitcoin-$VERSION-$ARCH-apple-darwin.tar.gz
xattr -dr com.apple.quarantine bitcoin-$VERSION   # stops macOS blocking the files
sudo mkdir -p /usr/local/bin
sudo cp bitcoin-$VERSION/bin/* /usr/local/bin/
bitcoind --version
```

If `bitcoind --version` prints a version number, you're ready.

---

## 2. Start your network

Everything lives in its own folder, `~/bitdevs`, so it never mixes with a real node. Create the folder and a config file inside it:

```bash
mkdir -p ~/bitdevs
cat > ~/bitdevs/bitcoin.conf << 'EOF'
regtest=1

[regtest]
fallbackfee=0.0001
txindex=1
daemon=1
listenonion=0
EOF
```

| Setting | What it does |
| --- | --- |
| `regtest=1` | Run the private test network, not real Bitcoin |
| `[regtest]` | Settings below this line only apply to regtest |
| `fallbackfee=0.0001` | Fee to use when there's no fee history to estimate from (a new chain has none) |
| `txindex=1` | Index every transaction so you can look any of them up by ID |
| `daemon=1` | Run in the background and give you your terminal back |
| `listenonion=0` | Don't look for Tor, which isn't needed here |

Start the node and create a shortcut for talking to it:

```bash
bitcoind -datadir=$HOME/bitdevs
alias bcli="bitcoin-cli -datadir=$HOME/bitdevs"
```

`-datadir` points both programs at `~/bitdevs`, where they find the config. The `alias` means you can type `bcli` instead of the full command. It only lasts for the current terminal window, so run it again if you open a new one.

Check it's alive:

```bash
bcli getblockchaininfo
```

You should see `"chain": "regtest"` and `"blocks": 0`. Height 0 is the **genesis block**, the first block, which is built into Bitcoin Core rather than mined. If you get a connection error, wait a few seconds and try again.

---

## 3. Make wallets and mine coins

Create two wallets. Alice will be our miner.

```bash
bcli createwallet alice
bcli createwallet bob
ALICE=$(bcli -rpcwallet=alice getnewaddress)
echo $ALICE
```

`-rpcwallet=alice` tells the node which wallet you mean, since two are open.

Mine 100 blocks to Alice:

```bash
bcli generatetoaddress 100 $ALICE
bcli -rpcwallet=alice getbalances
```

`trusted` is 0, but `immature` is 5000. Alice mined 100 blocks at 50 BTC each, yet she can't spend any of it. Bitcoin locks new mining rewards until 100 more blocks are built on top, so coins from a block that later gets replaced can never already have been spent.

Mine one more:

```bash
bcli generatetoaddress 1 $ALICE
bcli -rpcwallet=alice getbalance
```

Now she has 50. The reward from block 1 has 100 blocks on top, so it's spendable. That's why regtest tutorials always say "mine 101 blocks".

---

## 4. Send coins and watch them confirm

Alice sends 10 BTC to Bob:

```bash
BOB=$(bcli -rpcwallet=bob getnewaddress)
TXID=$(bcli -rpcwallet=alice sendtoaddress $BOB 10)
echo $TXID
```

That's the transaction ID (txid). The transaction is broadcast but not in a block yet. It's waiting in the **mempool**:

```bash
bcli getrawmempool
bcli -rpcwallet=bob getbalances
```

Your txid is in the mempool, and Bob's 10 BTC shows as `untrusted_pending`: on its way, but not final.

Confirm it by mining a block:

```bash
bcli generatetoaddress 1 $ALICE
bcli getrawmempool
bcli -rpcwallet=bob getbalance
bcli -rpcwallet=alice gettransaction $TXID
```

The mempool is empty, Bob has 10, and the transaction shows `"confirmations": 1`. On the real network, miners do this about every 10 minutes.

---

## 5. Decode the transaction

This is your transaction as raw bytes, exactly what nodes send each other:

```bash
RAW=$(bcli getrawtransaction $TXID)
echo $RAW
```

Turn it into something readable:

```bash
bcli decoderawtransaction $RAW
```

| Field | What it means |
| --- | --- |
| `txid` | The transaction's ID, a hash of its contents |
| `vin` | Inputs: which earlier coins are spent. Each points to a previous `txid` and a `vout` (which output of that transaction) |
| `txinwitness` | The signature proving Alice may spend the input |
| `vout` | Outputs: the new coins created, each with a `value` and a locking script (`scriptPubKey`) |
| `scriptPubKey.asm` | The locking script in readable form. `hex` is the same thing as raw bytes |
| `locktime` | The earliest block this transaction can be mined in (0 = any) |

You'll see **two** outputs. Bitcoin spends whole coins: Alice spent her 50 BTC coin, so one output pays Bob 10 and the other sends the rest back to Alice as **change**. Whatever's left over is the fee. See it with:

```bash
bcli -rpcwallet=alice gettransaction $TXID
```

---

## 6. Write a message on-chain with OP_RETURN

An `OP_RETURN` output carries data instead of money. It can never be spent, but its data stays in the blockchain permanently. It's used for timestamps and proofs, and how much data Bitcoin should allow is a hot debate right now.

Turn your message into hex (keep it under 80 characters):

```bash
MSG=$(printf 'gm from BitDevs Ibadan' | od -An -tx1 | tr -d ' \n')
echo $MSG
```

Send a transaction with that data as an output:

```bash
bcli -rpcwallet=alice send "[{\"data\": \"$MSG\"}]"
```

The wallet adds a coin to pay the fee, adds change, and broadcasts. Copy the `txid` it prints, then mine it and look inside:

```bash
OPTXID=paste-the-txid-here
bcli generatetoaddress 1 $ALICE
bcli getrawtransaction $OPTXID 1
```

Find the output with `"type": "nulldata"`. Its `asm` reads `OP_RETURN` followed by your hex. Decode it back to text:

```bash
printf "$(echo $MSG | sed 's/../\\x&/g')"; echo
```

Your message is now part of the chain. On mainnet, it would stay there for as long as Bitcoin exists.

---

## 7. No terminal? Use the Bitcoin Core app console

If you installed the Bitcoin Core app (`.exe` or `.dmg`), you can do everything in its built-in console. Start it in regtest mode.

> **Don't open it by double-clicking.** That starts downloading the real blockchain (700 GB+).

**Windows** (Command Prompt):

```
"C:\Program Files\Bitcoin\bitcoin-qt.exe" -regtest -fallbackfee=0.0001 -txindex=1
```

**macOS** (Terminal; check the app's name in Applications if this fails):

```bash
open -a Bitcoin-Qt --args -regtest -fallbackfee=0.0001 -txindex=1
```

The window title should say **[regtest]**. Open **Window → Console** and follow sections 3–6 with these changes:

- Leave out `bcli`. Type the command itself, e.g. `getblockchaininfo`.
- Leave out `-rpcwallet=...`. Pick the wallet from the dropdown at the top of the console.
- There are no `$VARIABLES`. Copy values from the output and paste them in. Shortcut: `generatetoaddress(101, getnewaddress())` mines 101 blocks to the selected wallet.
- For OP_RETURN, use this ready-made hex for "gm from BitDevs Ibadan":
  `send [{"data":"676d2066726f6d20426974446576732049626164616e"}]`

---

## 8. Troubleshooting

| You see | Fix |
| --- | --- |
| `bitcoind: command not found` | The install didn't finish. Rerun the `install` or `cp` line from section 1 |
| `Could not connect to the server` | The node isn't running or is still starting. Wait a few seconds, or rerun `bitcoind -datadir=$HOME/bitdevs` |
| `bcli: command not found` | New terminal window. Rerun the `alias` line from section 2 |
| `Wallet file not specified` | Add `-rpcwallet=alice` (or `bob`) after `bcli` |
| `Insufficient funds` | Alice's coins are still locked. Mine more blocks: `bcli generatetoaddress 101 $ALICE` |
| `Fee estimation failed` | Check `fallbackfee=0.0001` is in `~/bitdevs/bitcoin.conf` under `[regtest]`, then `bcli stop` and start `bitcoind` again |
| `Wallet ... is already loaded` | Fine, the wallet exists. Carry on |
| `tor: Error connecting to address 127.0.0.1:9051` in the log | Harmless. Add `listenonion=0` under `[regtest]` to silence it |
| A `%` after your decoded message (macOS) | That's zsh marking a missing newline, not part of the message |
| macOS: "developer cannot be verified" | Rerun the `xattr` line from section 1, or allow it in System Settings → Privacy & Security |
| The app started syncing the real blockchain | Close it and restart with `-regtest` as in section 7 |

### Watch the node's log

```bash
tail -f ~/bitdevs/regtest/debug.log
```

Press Ctrl+C to stop watching. The node keeps running.

### Stop or start over

```bash
bcli stop            # stop the node
rm -rf ~/bitdevs     # delete everything and start fresh (recreate bitcoin.conf after)
```

---

## Keep learning

- [Bitcoin Core RPC reference](https://developer.bitcoin.org/reference/rpc/): every command you used today, and more
- [Learn Me a Bitcoin](https://learnmeabitcoin.com/): clear visual explanations of transactions, scripts and blocks
- [Mastering Bitcoin](https://github.com/bitcoinbook/bitcoinbook): the free, open-source book on how Bitcoin works

---

**BitDevs Ibadan** is a monthly meetup for developers exploring Bitcoin protocol development and open source.
Follow us on X: [@BitDevsIbadan](https://x.com/BitDevsIbadan) · Edition 3 sponsored by Btrust