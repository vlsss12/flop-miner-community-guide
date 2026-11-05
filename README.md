# Run a FLOP GPU miner — step by step

> **Independent community guide · Devnet · checked 5 November 2026.** The [official miner guide](https://docs.flop.network/docs/run-miner) and its release pack are the final authority when commands change. This guide does not promise earnings. The [Testnet page](https://flop.finance/testnet/) still says **Draft**. Devnet activity does not currently count toward an airdrop.

This guide is for someone operating their **own GPU host or full Linux VM**. It does not recommend where to get a machine. The official same-host setup has two services: a **GPU backend** runs the model; a **worker** connects to FLOP, handles calibration and serves sessions.

## The route at a glance

1. Check the host and Docker GPU access.
2. Download and verify the official pack.
3. Create a miner account and back it up privately.
4. Configure the model, network and HTTPS endpoint.
5. Get Devnet sponsorship and registration.
6. Download and verify the exact model and decode policy.
7. Start the backend, then the worker.
8. Confirm calibration and readiness. Confirm **real settled work** separately when public paid sessions open.

**Important:** `Up` containers ≠ registered miner ≠ calibrated miner ≠ paid work. Stop at the first failed step.

## Before you start

| You need | Practical target |
| --- | --- |
| GPU | NVIDIA GPU that fits the chosen model. The draft Testnet recommendation is **16 GB+ VRAM per GPU**; this does not mean every model fits. |
| Host | Linux full VM or owned host with Docker, Compose, NVIDIA driver and Container Toolkit. A restricted container may not support this setup. |
| Disk | For the documented small Qwen2.5-0.5B model, plan **60 GB free** for checkpoint and headroom. **100–130 GB total disk** is a comfortable example for OS, images and logs, not a FLOP minimum. Larger models need more. |
| Network | Reliable outbound HTTPS/WebSocket and a public **HTTPS** hostname or tunnel for sessions. |
| Time | Keep the host online for calibration and monitor it. Your provider may bill even without jobs. |

Run these **read-only** commands on the *Linux GPU host*, not your laptop:

```bash
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv
docker --version
docker compose version
df -h /
docker info --format '{{json .Runtimes}}'
```

Then run the GPU-in-container test in [NVIDIA's Container Toolkit guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). **Continue only when both the host and Docker see the GPU.**

## Step 1 — Get the official pack

Open [FLOP's miner guide](https://docs.flop.network/docs/run-miner) and follow **Set up with your agent → Public pack**. Download the archive and its README, verify the **published checksum**, then extract it. Read the pack README: filenames, image digests and commands may change.

The current [public manual pack](https://flop.network/releases/cli/miner-pack-devnet-ts-20261106/) is published. The separate one-command `install.sh` mentioned in the docs is **not published yet**. Do not run a guessed installer or a social-post script.

**Done when:** the checksum passes and you have the matching pack README, env template and Compose files.

## Step 2 — Create and protect one miner account

Follow the pack's instructions to create a **dedicated sr25519 miner account**. Store its signer in a root-owned file with mode `0600`. Make an **encrypted backup off the host** before proceeding. Preserve the same identity and persistent worker state during host moves and upgrades.

- The **miner address** is public and safe to use for status checks.
- The **seed, mnemonic, keystore password and session keys** control access. Never paste them into GitHub, chats, screenshots or support tickets.
- Your **SSH key** opens the server; it is not the miner wallet key.
- Never print the signer merely to “check” it. Use the pack's **address-only** command.

**Done when:** you know the public address and have verified your private backup.

## Step 3 — Configure the services privately

Copy the pack's env template to a **private, uncommitted** file. Follow its README to set:

- Release-pinned CLI, worker and backend image **versions and digests**.
- Devnet RPC (`wss://rpc.devnet.flop.network`) and the **same** registered model, precision and decode policy for backend and worker.
- Signer path, fresh private inference/session tokens and persistent session key.
- `SESSION_PUBLIC_HOST`: your HTTPS hostname **without** `https://`.

The backend port stays private; the worker's raw session port stays on loopback. Use the pack's TLS sidecar with working DNS and inbound 80/443, or its own-tunnel path if behind NAT. If a temporary tunnel hostname changes, update it and verify a new endpoint advertisement. Confirm the **Devnet genesis identity** with the pack before signing anything.

**Done when:** Compose configuration validates and the HTTPS name points to this host. Do not expose backend port 8000 or the raw worker port.

## Step 4 — Sponsor and register on Devnet

Use the verified pack's `sponsor.sh` as its README directs. The [official sponsor](https://faucet.flop.network/) signs a request from the miner account and, when accepted, **registers it and delegates stake**. The documented default commission is 10%; check live terms first. **Do not run `miner register` again** after successful sponsorship.

The faucet's **25 FLOP quick-fund** is different. It gives test tokens to an account but does not create the miner's delegated capacity stake.

If sponsorship returns **429**, read [live limits](https://faucet.flop.network/limits) and retry only when the reason or retry time permits. **503** can mean a service-side problem; check later or request operator review. Do not retry in a tight loop.

**Done when:** the official CLI's `miner status --json` shows your expected public address, `registered: true` and active stake.

## Step 5 — Fetch the exact model and policy

Use the pack's `model fetch` command for a **registered model**. The official small-model example is `Qwen/Qwen2.5-0.5B-Instruct` with BF16. Fetch must reproduce its **registered model root**. A mismatch is a stop condition; do not relabel another download.

Leave the decode-policy fields empty initially, then run the pack's `decode-policy.sh` on your private env file. It gets the policy from the [official release manifest](https://flop.network/releases.json) and checks its hash against the chain.

**Done when:** fetch reports the expected verified root and the resolver fills an allowlisted policy/hash.

## Step 6 — Start the backend, then the worker

Use the **exact Compose commands from your pack's README**:

1. Validate Compose and pull the pinned images.
2. Start **backend**. Its authenticated `/health` must report `model_identity: measured`, the expected model hash, BF16 precision and decode-policy hash.
3. Start **worker** and the TLS sidecar, or worker alone with your tunnel.
4. Check worker logs for chain connection, model gate, Ghost-key publication and an HTTPS endpoint advertisement.
5. Test the endpoint externally. A no-token `401` proves the route is reachable and protected; it does **not** prove readiness.

**Done when:** the backend measures the correct model and the worker publishes the current HTTPS endpoint. “Containers Up” is insufficient.

## Step 7 — Wait for calibration

Keep the worker online while the Devnet issuer sends calibration jobs. Check **on-chain** `miner status --json` for registration, active stake and accepted calibration. Compare it with the worker's authenticated `/session/info` result for the same address and model. The live network's job count and time window may differ from examples in docs.

**Done when:** an accepted on-chain calibration exists and the worker reports ready. Calibration can expire or need renewal after a GPU change.

## Step 8 — Separate readiness from earnings

The [official miner guide](https://docs.flop.network/docs/run-miner) currently says **public paid sessions are not open yet**. A registered, calibrated worker with a reachable endpoint is ready for the next gate; it has **not** proved that a buyer used it. When official admission opens, require an accepted, verified, **finalized settlement receipt** before calling it paid work.

The [draft Testnet rules](https://flop.finance/testnet/) describe credit for verified compute served, not idle uptime or merely holding test tokens. Do not assume Devnet work counts.

## Fast troubleshooting

| Symptom | Check first |
| --- | --- |
| GPU works on host, not in Docker | NVIDIA Container Toolkit, Docker runtime, VM passthrough. |
| Sponsor says 429 | [Live limits](https://faucet.flop.network/limits), reason and retry time. |
| Registered, no calibration | Worker logs, Ghost enrollment, finalized blocks, stake and model gate. |
| Backend hash/policy wrong | Checkpoint root, revision, precision, policy JSON/hash and image digests. |
| WebSocket disconnected | RPC, DNS/TLS, host clock and upstream status. |
| HTTPS endpoint unreachable | Current hostname, certificate/tunnel, loopback listener and signed endpoint ad. |
| All services Up, no paid work | Public admission may still be closed. Look for a real finalized receipt. |

For daily checks, use the pack's Compose `ps` and recent logs, backend health, on-chain status, calibration validity and the HTTPS endpoint. **Never use `docker compose down -v` as a routine restart**: it removes persistent volumes. Read the [official upgrade instructions](https://docs.flop.network/docs/run-miner) before changing image pins.

## Share results without sharing access

Useful public details: date, GPU model/VRAM, pack/image versions, **public** model hash, redacted health fields, registration/calibration status and a public finalized receipt if one exists. Hide personal identity, host IP/SSH access, full logs, env files, seeds, keystores, passwords, bearer tokens and backups.

## Sources and reuse

- [Official FLOP miner guide](https://docs.flop.network/docs/run-miner)
- [Official links and verification](https://docs.flop.network/docs/official-links)
- [Official Devnet release pack](https://flop.network/releases/cli/miner-pack-devnet-ts-20261106/)
- [Official Testnet status and draft rules](https://flop.finance/testnet/)
- [NVIDIA Container Toolkit guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

This independent guide is shared under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You may adapt it with attribution. Keep the review date and source links visible because FLOP's commands and network gates can change.
