# 🐕 Hardened Dogecoin Node

> A production-style, security-hardened Dogecoin Core node — built as a learning project for anyone curious about systemd sandboxing, Linux service hardening, and what it actually takes to run a full crypto node responsibly.

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](LICENSE)
[![Dogecoin Core](https://img.shields.io/badge/Dogecoin%20Core-1.14.6-yellow.svg)](https://github.com/dogecoin/dogecoin)
[![systemd](https://img.shields.io/badge/systemd-hardened-green.svg)](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)

---

## 🤔 Why this repo exists

Most "run a node" tutorials hand you a `wget && chmod +x && sudo` one-liner and call it a day. That works, but you end up with a process running as root, with full filesystem access, no resource limits, and no idea what would happen if it got popped.

This repo takes the opposite approach. It's a **small set of well-commented config files** that show you, line by line, how to wrap `dogecoind` in a tight systemd sandbox: dedicated user, capability dropping, syscall filtering, read-only filesystem mounts, the works. Every directive has a comment explaining what it does and *why* it's there.

It started as a personal weekend project. It's published in the hope that it's useful to anyone learning about:

- 🛡️ How modern Linux services are sandboxed (`systemd.exec(5)` is wild once you read it)
- 🪙 How a full UTXO blockchain node is provisioned and operated
- 🔐 The principle of least privilege, applied to a real workload
- 🧰 Production-style ops on a single box: services, journald, resource limits

You don't need to be a sysadmin or a Doge holder. You just need a Linux box and an evening.

## 📦 What's in the box

```
.
├── dogecoind.service   # systemd unit — the heart of the hardening
├── dogecoin.conf       # daemon config installed to /etc/dogecoin/
├── README.md           # you are here
└── LICENSE             # GPLv3
```

That's it. No installer, no scripts, no magic. Three files, ~250 lines of config, all commented.

## 🗺️ How the pieces fit together

```
              ┌─────────────────────────────────────────────┐
              │                  systemd                    │
              │   (boot, restart, limits, sandboxing)       │
              └──────────────────┬──────────────────────────┘
                                 │ launches as dogeuser
                                 ▼
        ┌────────────────────────────────────────────────────┐
        │                   dogecoind                        │
        │  • reads /etc/dogecoin/dogecoin.conf               │
        │  • writes blockchain to /mnt/data/doge/            │
        │  • exposes RPC on 127.0.0.1:22555                  │
        │  • peers with the Dogecoin P2P network on :22556   │
        └────────────────────────────────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ▼                    ▼                    ▼
      blockchain data       debug log           journald logs
     /mnt/data/doge/    /mnt/data/doge/   journalctl -u dogecoind
                          debug.log
```

**Key idea:** `systemd` is doing far more than "start the binary on boot." It's actively constraining what `dogecoind` is allowed to do — which kernel calls, which network families, which directories, which capabilities. If the daemon is ever compromised, the attacker is stuck inside that very small box.

## 📋 Prerequisites

- A Linux host with **systemd ≥ 247** (Debian 11+, Ubuntu 22.04+, Fedora 35+, RHEL 9+, modern Arch, etc.)
- **Dogecoin Core 1.14.6** binaries unpacked at `/opt/dogecoin-1.14.6/` — grab them from the [official releases](https://github.com/dogecoin/dogecoin/releases) and verify the signature
- **~50 GB free disk** for the blockchain, plus headroom (we recommend mounting a dedicated disk at `/mnt/data/doge/`)
- **Root / sudo** access to install the unit and create users
- A stable internet connection — initial sync downloads the full chain history from peers

> 💡 If your distro ships with a systemd older than 247, a few hardening directives (`ProtectProc`, `ProcSubset`) won't be recognized. The unit will still run, just with slightly less isolation. `systemd-analyze --version` tells you what you have.

## 🚀 Installation

The whole flow is: **drop the unit in place → make a service user → make a data dir → enable the service.** Each step below explains *what* it's doing and *why*.

### 1. Clone this repo somewhere convenient

```bash
git clone https://github.com/just-an-oldsalt/hardend-dogecoin-node.git
cd hardend-dogecoin-node
```

### 2. Create the dedicated service user and group

The whole point of the sandbox is that `dogecoind` runs as a non-privileged user with no shell and no home directory — so even a remote-code-execution bug in the daemon can't easily pivot to anything else on the box.

```bash
# Create a system user with no login shell and no home directory.
sudo useradd --system --no-create-home --shell /usr/sbin/nologin dogeuser

# Create the group and add the user to it.
sudo groupadd --system dogegroup
sudo usermod -aG dogegroup dogeuser
```

> 🐧 `useradd` / `groupadd` are part of `shadow-utils` and present on virtually every modern distro. Older guides use Debian-specific `adduser` / `addgroup` — those work too, but `useradd` is portable.

### 3. Install the config file

```bash
sudo install -d -m 0710 -o root -g dogegroup /etc/dogecoin
sudo install -m 0640 -o root -g dogegroup dogecoin.conf /etc/dogecoin/dogecoin.conf
```

Now **edit the config** and set a strong RPC password:

```bash
sudo "${EDITOR:-nano}" /etc/dogecoin/dogecoin.conf
# Replace CHANGE_ME_BEFORE_FIRST_START with the output of:
#   openssl rand -base64 32
```

The default ships with the placeholder `CHANGE_ME_BEFORE_FIRST_START` so the daemon fails loudly if you forget — better than running with a known-bad password.

### 4. Create the blockchain data directory

```bash
sudo install -d -m 0750 -o dogeuser -g dogegroup /mnt/data/doge
```

If you want the chain on a different mount point, change the path here **and** the matching `-datadir=` in `dogecoind.service` (line 41). The systemd unit also creates and chowns this on every start, so it'll self-heal if perms drift.

### 5. Install and enable the systemd unit

```bash
sudo install -m 0644 dogecoind.service /etc/systemd/system/dogecoind.service
sudo systemctl daemon-reload
sudo systemctl enable --now dogecoind
```

`enable --now` does both `enable` (start on boot) and `start` (start now) in one command.

### 6. Watch it come up

```bash
# Should show "active (running)" within a few seconds.
sudo systemctl status dogecoind

# Tail the journal to confirm it's syncing.
sudo journalctl -u dogecoind -f
```

🎉 You should see the daemon connecting to peers and starting the initial block download. Full sync takes anywhere from a few hours to a day depending on disk and bandwidth.

## 🎮 Day-to-day operations

```bash
# Service control
sudo systemctl start dogecoind          # start
sudo systemctl stop dogecoind           # stop (gives up to 10 min to flush)
sudo systemctl restart dogecoind        # restart
sudo systemctl status dogecoind         # current state + recent log lines

# Logs
sudo journalctl -u dogecoind -f         # live tail
sudo journalctl -u dogecoind -n 200     # last 200 lines
sudo tail -f /mnt/data/doge/debug.log   # dogecoind's own debug log

# Talking to the daemon (run as root or any user that can read the conf)
dogecoin-cli -conf=/etc/dogecoin/dogecoin.conf getblockchaininfo
dogecoin-cli -conf=/etc/dogecoin/dogecoin.conf getconnectioncount
dogecoin-cli -conf=/etc/dogecoin/dogecoin.conf getmempoolinfo
dogecoin-cli -conf=/etc/dogecoin/dogecoin.conf getnetworkinfo
```

## 🛡️ The hardening, explained

This is the part most node tutorials skip. The unit file applies the following layers — each one shrinks what an attacker who somehow controls the dogecoind process could do.

| Directive | What it stops |
|---|---|
| `User=dogeuser` / `Group=dogegroup` | Anything you can't do as a non-privileged user — no root, no sudo, nothing. The biggest single win. |
| `NoNewPrivileges=true` | Setuid binaries can't elevate, even if they're on the filesystem. Closes the entire `setuid` escalation class. |
| `CapabilityBoundingSet=` (empty) | Drops *every* Linux capability — `CAP_NET_ADMIN`, `CAP_SYS_PTRACE`, `CAP_SYS_MODULE`, all of them. Dogecoind needs none. |
| `ProtectSystem=full` | `/usr`, `/boot`, `/etc` are mounted read-only for the process. Can't tamper with system binaries or other services' configs. |
| `ProtectHome=true` | `/home`, `/root`, `/run/user` are invisible. Can't read your SSH keys or browser cookies. |
| `PrivateTmp=true` | Private `/tmp` and `/var/tmp` invisible to other processes. Stops `/tmp` race attacks dead. |
| `PrivateDevices=true` | Private `/dev` with only `null`, `zero`, `random`, `urandom`, `tty`. No raw disks, no USB, no input devices. |
| `MemoryDenyWriteExecute=true` | A page can never be both writable and executable. Defeats most shellcode injection that needs `mprotect(W\|X)`. |
| `ProtectProc=invisible` + `ProcSubset=pid` | `/proc` only shows our own PIDs and only the `pid/` subset. Can't enumerate other processes or read their memory. |
| `RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX` | Sockets are limited to IPv4 / IPv6 / Unix domain. No `AF_NETLINK`, no `AF_PACKET`, no raw sockets. |
| `SystemCallFilter=@system-service` | A seccomp allowlist. Anything outside the curated "normal service" syscall set returns `EPERM`. |
| `RestrictNamespaces=true` | Can't create new namespaces — no nested containers, no `unshare()` shenanigans. |
| `ProtectKernel{Modules,Tunables,Logs}=true` | No `insmod`, no writing `/proc/sys`, no reading `dmesg`. |
| `ProtectClock=true` / `ProtectHostname=true` | Can't change the system clock or hostname. |
| `LockPersonality=true` | Can't switch personalities (e.g. fake being 32-bit). |
| `RestrictRealtime=true` | No `SCHED_FIFO` / `SCHED_RR` — can't starve the system with realtime priority. |
| `RestrictSUIDSGID=true` | Can't create new setuid/setgid files. |

Curious how much of this is actually being applied? Ask systemd:

```bash
systemd-analyze security dogecoind
```

You'll get a per-directive breakdown with a "predicted exposure" score (0 = fortress, 10 = wide open). This unit lands well into the "ok" / "good" range.

## 🩹 Troubleshooting

**Service fails to start**

```bash
sudo journalctl -u dogecoind -n 100 --no-pager
sudo systemd-analyze verify dogecoind.service
```

`verify` will flag any typos or unknown directives in the unit. The journal will show why dogecoind itself bailed out.

**`Permission denied` reading the config**

Confirm dogeuser is in dogegroup and the conf is `0640 root:dogegroup`:

```bash
id dogeuser
ls -la /etc/dogecoin/
```

**Dogecoind starts but never finds peers**

This usually means the network sandbox is too tight for your environment. The repo's unit doesn't restrict outbound network interfaces, but if you uncommented `RestrictNetworkInterfaces=lo eth0` and your real interface isn't named `eth0`, the daemon can only see loopback. Check `ip -br link` and adjust.

**`Killed` in the journal with no other context**

The kernel OOM-killer got it. Either reduce `dbcache` / `maxmempool` in `dogecoin.conf`, or raise `MemoryHigh=` / `MemoryMax=` in the unit — keep the two in sync.

**Initial sync is brutally slow**

Expected. You're downloading and validating years of blockchain history. SSDs help enormously over spinning disks. `txindex=1` adds a bit of overhead — you can disable it (`txindex=0`) if you don't need to query arbitrary historical transactions.

## 🧭 Going further

A few directions to take this from here, in roughly increasing depth:

- 📊 **Monitoring** — point Prometheus' [`dogecoind_exporter`](https://github.com/jvstein/bitcoin-prometheus-exporter) (it works for dogecoind too) at the RPC endpoint and graph block height, peer count, mempool size.
- 🧅 **Tor-only mode** — run dogecoind with `-onlynet=onion` and a local tor proxy, then uncomment `RestrictNetworkInterfaces=` in the unit to lock outbound traffic to the tor socket.
- 🔥 **Firewall** — `ufw allow 22556/tcp` (P2P) and explicitly *deny* RPC at the firewall as belt-and-braces; the daemon already binds RPC only to localhost.
- 🧪 **Read the directives** — open `dogecoind.service` and `man systemd.exec` side by side. Almost every line is searchable in that man page.
- 🐳 **Containerize it** — once you understand the unit, porting this to a rootless Podman or Docker setup is a great follow-up exercise in mapping systemd primitives to container ones.

## 🤝 Contributing

Issues and PRs welcome. If you spot a directive that's redundant, missing, or actively harmful, open an issue — getting the unit *right* matters more than getting it *long*.

## 📄 License

GPL-3.0 — see [LICENSE](LICENSE).

## ⚠️ Disclaimer

This is published as a learning resource, not a turnkey product. Test in a non-critical environment first. The author is not responsible for lost coins, corrupted chains, or melted SSDs. If you're storing real value on this node, **back up your wallet** (`/mnt/data/doge/wallet.dat`) and read the [Dogecoin Core docs](https://github.com/dogecoin/dogecoin) end-to-end before trusting it.

## 🔗 Resources

- [Dogecoin Core repo](https://github.com/dogecoin/dogecoin)
- [`systemd.exec(5)` — the hardening directive bible](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)
- [`systemd-analyze security`](https://www.freedesktop.org/software/systemd/man/systemd-analyze.html#systemd-analyze%20security%20%5BUNIT...%5D) — score your own unit
- [Dogecoin Reddit](https://www.reddit.com/r/dogecoin/) — much wow, very community
