# AGENTS.md — horcrux (BryanLabs fork)

## What it is

Horcrux is a multi-party-computation (MPC) remote signing service for CometBFT
(Tendermint) validators. It splits the validator private key into shares across
a cluster of cosigner nodes and uses threshold Ed25519 signatures so no single
node holds the full key. This repo is a BryanLabs fork of
[strangelove-ventures/horcrux](https://github.com/strangelove-ventures/horcrux).

## Where it's used

The cluster runs three cosigner deployments (`cosigner-1`, `cosigner-2`,
`cosigner-3`) in the `horcrux` namespace. **Those deployments use the upstream
image `ghcr.io/strangelove-ventures/horcrux:v3.3.2`, not an image built from
this fork.** This local fork is not currently deployed anywhere.

## BryanLabs customizations

None. `git diff upstream/main HEAD` produces no output. The fork is a clean
mirror of upstream at the `c42b04f` commit ("fix spelling of grpc"). No
BryanLabs-specific commits exist.

## How it works

- **Threshold signing:** The validator Ed25519 key is split into N shares (e.g.
  3 cosigners, threshold 2). Any `t-of-n` cosigners can collaboratively sign a
  block without any single node reconstructing the full key.
- **Raft consensus:** Leader election and high-watermark tracking (prevents
  double-signing) use Raft. One cosigner is the leader at a time.
- **Nonce pre-sharing:** Cosigners pre-exchange nonces so signing latency is
  minimal at block time.
- **gRPC transport:** Cosigners communicate over mutually-authenticated gRPC
  (RSA or ECIES security modes). Validators connect to cosigners via the same
  gRPC remote-signer protocol CometBFT expects.

Details: `docs/signing.md`, `docs/migrating.md`.

## Code map

| Path | Purpose |
|---|---|
| `cmd/horcrux/` | CLI entrypoint (`main.go`) and subcommands |
| `signer/` | Core signing logic: threshold signer, local/remote cosigner, Raft store, nonce cache, health, metrics |
| `signer/threshold_signer_soft.go` | Software-only Ed25519 threshold signer |
| `signer/threshold_validator.go` | Validates and coordinates threshold signing across cosigners |
| `signer/remote_cosigner.go` | gRPC client to peer cosigners |
| `signer/cosigner_security_*.go` | RSA and ECIES encryption for inter-cosigner communication |
| `signer/raft_store.go` | Raft FSM for high-watermark state |
| `docker/horcrux/Dockerfile` | Multi-stage scratch image build |
| `docs/` | Signing design, migration guide, metrics reference |
| `proto/` | gRPC protobuf definitions |
| `test/` | Integration tests (separate Go module) |

## Build and deploy

The Dockerfile at `docker/horcrux/Dockerfile` produces a minimal scratch image:

1. Builds `horcrux` binary with CGO enabled (static musl linkage) via `make install`.
2. Assembles a scratch image with minimal busybox utils and the binary at `/bin/horcrux`.
3. Runs as non-root user `horcrux` (UID 2345).

To build for AMD64 (required for the cluster):

```
docker buildx build --builder worker1 --platform linux/amd64 \
  -f docker/horcrux/Dockerfile -t ghcr.io/bryanlabs/horcrux:<tag> .
```

The upstream image tag in use is `ghcr.io/strangelove-ventures/horcrux:v3.3.2`.

## Gotchas

- **Not deployed from this fork.** Any change here requires building and pushing
  a new image and updating the `horcrux` namespace deployments in bare-metal
  manifests before it has any effect.
- **go.mod still references `strangelove-ventures`** as the module path
  (`github.com/strangelove-ventures/horcrux/v3`). If you add custom code,
  update the module path to avoid confusion.
- **Key shares must be pre-provisioned.** Each cosigner node requires its
  `*.shards` key file. Losing a cosigner's shard below the threshold breaks
  signing permanently. Back up shards securely before any cosigner maintenance.
- **Raft quorum:** With 3 cosigners and threshold 2, one cosigner can be down
  without losing signing ability. Two down = no signatures.
- **High watermark state is on-disk** (`signer/raft_store.go`). Do not wipe the
  data directory without understanding the double-sign risk.
