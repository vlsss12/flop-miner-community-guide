# FLOP GPU miner: an independent operator field guide

> **Community guide · Devnet · reviewed 5 November 2026.** This is an independent, vendor-neutral guide, not a FLOP Labs release or a promise of earnings. FLOP changes quickly: use the [official miner instructions](https://docs.flop.network/docs/run-miner) and the release pack they link as the final authority before running a command. The [public Testnet page](https://flop.finance/testnet/) still labels Testnet **Draft**; Devnet participation has no rewards or airdrop weight.

This guide is for someone who will **operate** an NVIDIA GPU host as a FLOP inference miner. It does not recommend where to obtain a machine. The path below ends with a registered, calibrated miner serving a measured model and a reachable session endpoint. Actual paid sessions need separate network admission and a positive settled-work receipt.

## The five proof gates

| Gate | What to observe | What it does **not** establish |
| --- | --- | --- |
| 1. Host | `nvidia-smi` and a Docker GPU probe both see the GPU | FLOP registration |
| 2. Model | Backend health reports `model_identity: measured` and the expected model/policy hashes | A live worker |
| 3. Chain | Miner status reports `registered: true` and active stake | Calibration or earnings |
| 4. Calibration | On-chain calibration is accepted and the worker reports ready | Public gateway admission |
| 5. Work | A real session is accepted, verified and finalized with a receipt | Future Testnet allocation by itself |

Keep a dated, **redacted** record of each gate. A container saying “Up” proves only that its process is running.

### Route at a glance

`host + Docker GPU` → `official pack + checksum` → `private miner key + backup` → `sponsorship/registration` → `measured checkpoint + policy` → `backend health` → `worker + HTTPS` → `on-chain calibration` → `real settled session when admission opens`.

If any gate fails, stop at that gate. Do not conceal an unverified model, run a second identity, or mistake Devnet registration for a paid-session receipt.

## 0. Decide whether your host is suitable

| Resource | Practical planning target | Why |
| --- | --- | --- |
| GPU | NVIDIA GPU with **16 GB+ VRAM per GPU** for the draft Testnet recommendation | Each model also has its own fit and throughput requirements. Other cards require their own verification. |
| Host | Owned Linux machine or a **full VM** that permits Docker and NVIDIA GPU passthrough | The current inference miner has a CPU worker and a GPU backend on one host. A restricted container alone may not run Docker correctly. |
| OS | Recent Ubuntu Linux; follow the selected official pack's supported version | The public installer documentation targets Ubuntu 24.04. Confirm a manual pack's requirements before using another version. |
| Disk | For the documented Qwen2.5-0.5B path, **60 GB available** for the checkpoint and headroom. A roughly **100–130 GB total volume** is a comfortable planning choice, not a FLOP minimum. | OS, image layers, model files, logs and rollback copies consume extra space. Larger models need much more; a 27B BF16 checkpoint alone is about 55.6 GB. |
| Network | Reliable outbound HTTPS and WebSocket; a stable public HTTPS name/endpoint for sessions | RPC, model download, mailbox, certificate and endpoint advertisement must work. Keep the backend private. |
| Operation | Time to monitor the worker and respond to upgrades or failures | Calibration can expire, and unattended work has deadlines. |

**Size by the exact model and release.** The official 16 GB+ figure is a provisional Testnet recommendation, not proof that any 16 GB card can serve every model. Check GPU memory, model files, Docker storage and host-wide capacity-stake rules before committing resources. The host operator, not a GPU listing alone, owns the miner account and registration.

### Read-only preflight on the Linux host

```bash
uname -a
nvidia-smi --query-gpu=name,driver_version,memory.total --format=csv
docker --version
docker compose version
df -h /
docker info --format '{{json .Runtimes}}'
```

After installing the **official NVIDIA Container Toolkit**, confirm GPU access from inside Docker using the probe in [NVIDIA's installation guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). If the Docker GPU probe fails, fix it before downloading the miner images. Record only hardware type and capacity in public troubleshooting posts; omit hostnames, IPs, account identifiers and credentials.

## 1. Establish trusted inputs

1. Open [Official links & verification](https://docs.flop.network/docs/official-links). Follow its links to the [miner documentation](https://docs.flop.network/docs/run-miner), public release pack and [release manifest](https://flop.network/releases.json). Treat social replies, DMs and community repositories as untrusted suggestions.
2. Confirm the pack is **actually published**, download it from the official URL and verify its **published checksum** before extraction. Read its own `README.md` completely. An unreleased or future-dated pack is not an install instruction.
3. Pin the pack's CLI, worker and backend image **versions and digests**. Keep one compatible release set; do not use `latest` tags or mix random image versions.
4. Check the active chain and faucet genesis match through the pack's verification steps. Use the official Devnet RPC, `wss://rpc.devnet.flop.network`; never infer the chain from a website's appearance alone.

Do **not** copy someone else's filled `.env`, signer, session key or secret backup. The pack's examples are templates; the values for your host must be generated and verified locally.

## 2. Create and protect the miner identity

Follow the pack's **local account/keystore** instructions. The current official procedure creates an sr25519 miner identity and stores its encrypted keystore on the host with restrictive permissions. Make an encrypted offline backup before relying on the host. Keep the keystore password, recovery phrase, session/Ghost keys and tokens outside GitHub, chats, issue comments, screenshots and shell history.

The miner **address is public**; the secret that controls it is not. Never print the signer or mnemonic merely to “check” it. To confirm the intended address, use the pack's address-only command and compare it with the address returned by `miner status`. An SSH key only opens the server; it does **not** control the miner account.

When moving hosts, preserve the **same** miner identity and persistent worker state. Do not create a fresh account just because the VM changes. Restore from your encrypted backup only on a trusted machine.

## 3. Sponsor and register on Devnet

Use the official pack's sponsorship procedure and the [official Devnet sponsor](https://faucet.flop.network/sponsor/miner). Sponsorship signs a request with the miner identity, registers the miner and delegates Devnet stake when accepted. At the time of review the documented sponsored path uses a 10% commission. Verify live terms before submitting.

The currently documented manual pack calls `sudo env SPONSOR_URL=https://faucet.flop.network FLOP_RPC=wss://rpc.devnet.flop.network ./sponsor.sh` **after** creating the private signer. Run that only from a checksum-verified official pack and only after you have confirmed the account address and Devnet genesis.

**A 25 FLOP quick-fund grant is different from miner sponsorship.** The former funds a test account; it does not automatically create the delegated capacity stake. Likewise, a `registered: true` status does not mean the worker is calibrated or serving jobs.

The sponsor can return `429` when its automatic uncalibrated-miner allowance or daily limit is full. Inspect the [Devnet limits](https://faucet.flop.network/limits), wait as instructed, and retry deliberately; do not run a tight loop. A `503` means a service-side condition may need time or operator attention. If the sponsor has already registered the account, **skip** a second `miner register` transaction and verify the on-chain status instead.

Never pay someone for a Devnet stake grant, expose a seed to “support”, or assume an unsolicited direct message is an operator.

## 4. Fetch and verify the exact model

Choose a model already registered on chain and supported by the current pack. Use the pack's `model fetch` flow, which downloads a checkpoint from registered mirrors and checks its measured root. A mismatch is a **stop condition**: do not substitute an arbitrary model download and relabel it with the expected hash.

The decode-policy JSON must match the hash allowlisted for that model. Use the pack's `decode-policy.sh` resolver and [release manifest](https://flop.network/releases.json); it verifies the policy against finalized chain state. Fill the backend and worker configuration from the same resolved model, revision, precision and policy. Put private API and session tokens in root-owned local files as the official pack directs.

For the documented Qwen2.5-0.5B BF16 path, the backend should report a **measured** model identity and matching model/decode-policy hashes. If it merely starts without these checks, Gate 2 is not passed.

## 5. Expose only the session endpoint

Start the backend and worker with the **matching Compose files** in the official pack. The backend port is private; the worker session listener stays on loopback and is published through the pack's HTTPS sidecar or a controlled HTTPS tunnel. Configure the public DNS host exactly as the pack specifies. Verify that its signed endpoint advertisement reaches the Devnet mailbox and that the advertised URL resolves externally.

An ephemeral tunnel hostname may change after restart. If it changes, update `SESSION_PUBLIC_HOST`, restart the appropriate service and verify a **new** endpoint advertisement. For a long-running miner, prefer stable DNS and the official TLS setup. Do not publish port 8000, put bearer tokens into URLs, or paste tokens into tickets.

For local health checks, follow the pack's authenticated `/health` and `/session/info` probes. A public no-token probe returning **401** can show the route is reachable and access-controlled; it does not demonstrate readiness. The authenticated response must report the expected miner, model and readiness values.

## 6. Allow calibration to finish

Keep the worker running while its Ghost enrollment and calibration jobs proceed. Check worker logs for the published Ghost key, model gate, chain connection and session advertisement. Then use the pack's CLI `miner status --json` to confirm **on-chain** registration, active stake and an accepted calibration record. The worker's health and on-chain cap should agree.

Calibration is measured across the whole host. The official source describes 64 verified jobs over at least 600 finalized blocks, while a deployed Devnet can require different live constants; do not hardcode a deadline or force a result. If jobs are absent, inspect finalized chain progress, worker connectivity, sponsor/bond status and issuer enrollment before changing the account. Calibration can expire and may need renewal after changes to GPU inventory.

At this point the machine is a **calibrated Devnet miner**. The official docs still separate this from public gateway admission and paid sessions. Do not claim revenue or Testnet credit merely from calibration.

## 7. Prove work only when the network opens it

Follow the official [specific-miner request procedure](https://docs.flop.network/docs/run-miner) when its session test is published and the operator has explicitly opened that acceptance path. Check the same model identity, accepted session, proof/DA evidence and a **finalized settlement receipt**. A log line saying “serving” or “endpoint ad published” proves the service is available, not that a buyer used it.

For Testnet eligibility, use the [official Testnet page](https://flop.finance/testnet/) and future rules when published. The draft says verified compute served counts; idle uptime and self-generated circular activity do not. Devnet work currently has **no reward or airdrop weight**.

## Daily operation and safe upgrades

Use the exact Compose command from your pack to check process state and recent logs. The command pattern below follows the **current official manual pack documentation**; confirm these files exist in your checksum-verified pack before running it. With your own tunnel, omit the `docker-compose.tls.yml` file and do not start `tls`:

```bash
compose() {
  docker compose --env-file miner-01.env \
    -f docker-compose.miner.yml \
    -f docker-compose.backend.yml \
    -f docker-compose.tls.yml "$@"
}
compose config --quiet
compose ps
compose logs --tail=80 worker backend tls
```

For a **first start**, use the pack's instructions to finish identity and environment setup, run its decode-policy resolver, then bring up `backend` first. Confirm measured model health before starting `worker` and `tls`. The current official manual order is `./decode-policy.sh` → Compose config/pull → backend → authenticated backend health → worker/TLS → authenticated session health → on-chain status. Do not copy this sequence into a different pack without checking its README.

Never paste unredacted logs publicly: they can include host data, signatures, session identifiers or keys. Check disk free space, GPU memory, model-health hashes, finalized RPC progress, worker restart count, calibration validity and the published HTTPS endpoint. Keep the host's security updates and container runtime supported.

Before an upgrade, read the [official upgrade manual](https://docs.flop.network/docs/run-miner), record current pins and healthy gates, drain active work when required, back up the encrypted identity and persistent state, and then update as one compatible release set. Preserve the worker data volume. **Do not run `docker compose down -v`** as a routine restart: it deletes volumes.

### Troubleshooting by failed gate

| Symptom | First checks |
| --- | --- |
| Host GPU visible, Docker GPU absent | NVIDIA Container Toolkit, Docker runtime registration, VM GPU passthrough |
| Backend loops or health has wrong hash | Checkpoint root, model revision, precision, decode-policy JSON/hash, image digest, VRAM |
| Sponsor says `429` | Official limit and retry guidance; avoid rapid retries |
| Registered, no calibration | Ghost enrollment, worker logs, finalized blocks, stake/bond, model gate, live calibration constants |
| Endpoint unreachable | HTTPS certificate/tunnel, current public hostname, signed mailbox ad, worker loopback listener |
| Session says not ready | Accepted on-chain calibration plus matching measured model and worker state |
| Worker WebSocket disconnects | RPC reachability, DNS/TLS, host clock and upstream status; reconnect before diagnosing job flow |
| Containers “Up”, no paid work | Public gateway/paid-session admission may still be closed; ask for a genuine finalized receipt, not only container uptime |

## Publish evidence without publishing access

Safe public evidence: date, **redacted** GPU model/VRAM, pack version and digest, expected-versus-measured model hash, on-chain `registered`/calibration fields, and a public finalized receipt if one exists. Do not publish the host IP, SSH endpoint, cloud account, private tunnel token, keystore, signer file, `.env`, passwords, session bearer, recovery words, full private logs or personal identity. Redact screenshots before uploading them.

## Sources and maintenance

- [Official miner instructions](https://docs.flop.network/docs/run-miner) — current commands, pack and operational gates.
- [Official links & verification](https://docs.flop.network/docs/official-links) — canonical domains and current token/claim status.
- [Official Testnet page](https://flop.finance/testnet/) — draft timeline, provisional hardware and what counts.
- [Official release manifest](https://flop.network/releases.json) — pinned images, models and policies; verify against the chain.
- [NVIDIA Container Toolkit installation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) — Docker GPU setup.

**Maintenance rule:** if an official release changes a command, hash, network rule or readiness gate, update this guide with a dated note and a source link. The official pack always wins when this guide is stale.

## Reuse

This independent guide is shared under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/). You may adapt it with attribution and a link to the license. Keep the review date and source links visible so readers can judge whether the instructions are still current.

