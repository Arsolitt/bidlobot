# Devlog 16 - 2026-10-17: deploy pipeline hardened (gated chain, digest pinning, SHA-pinned actions)

## Trigger

The owner approved a set of deploy-pipeline risk fixes after two
observations from the live system: ci.yml and publish.yml are
independent workflows on a master push, so a red ci run deployed
anyway; and the gitops manifest pinned only a mutable tag, so
overwriting a tag in GHCR would silently change what the cluster runs.

## What changed

- `publish.yml` restructured into a linear chain where every stage
  gates the next: `test` (vet + `go test ./...`, go 1.26, setup-go
  cache - ci.yml's test job minus the coverage artifact) -> `image`
  (unchanged linux/amd64 GHCR contract, plus a new `DIGEST` job output
  from build-push-action's `digest` with a `docker buildx imagetools
  inspect` fallback and a sha256-format guard) -> `smoke` + `sign` in
  parallel -> `update-manifest` (needs all three).
- `smoke` pulls the pushed image **by digest** with a GITHUB_TOKEN
  docker login and, with the fake env
  `TG_BOT_TOKEN=123456789:ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghi` /
  `BOT_OWNER_ID=1`, asserts `--check-config` exit 0 (empty token must
  exit non-zero, mirroring ci.yml's docker job) and then boots the
  image detached and asserts the documented fail-fast startup: exit 1
  with the `get bot info` error in the logs. `/livez` is NOT probed:
  `cmd/bidlobot/main.go` runs GetMe and exits (lines 97-101) BEFORE
  the health listener exists (AttachHealth at line 201, listener in
  Run at line 272), and api.telegram.org answers the fake token with
  `HTTP 401 {"ok":false,...}` in ~1s - nothing ever listens on 8080,
  so a curl-until-200 loop would hold the pipeline permanently red.
  Reaching the auth gate still proves the runtime: config validation,
  data dir + bbolt open as UID 65532, a real TLS round-trip, and no
  earlier failure; liveness stays proven by the kubelet probes.
- `sign` cosigns the digest keyless (OIDC): job-scoped permissions
  `{contents: read, packages: write, id-token: write}` - id-token is
  granted to that job only - plus a docker login because the `*.sig`
  signature tag is pushed to GHCR.
- `update-manifest` now rewrites BOTH `newTag: .*` and `digest: .*` in
  `deploy/kustomization.yaml` (two seds); the commit message stays
  `chore(deploy): bump image tag to <tag>`.
- Every action in BOTH workflows is pinned to a full commit SHA with
  the major tag as a trailing comment, resolved via the GitHub tag-ref
  API (annotated tags dereferenced): checkout `11d5960a`, setup-go
  `40f1582b`, upload-artifact `ea165f8d`, setup-buildx `8d2750c6`,
  login `c94ce9fb`, build-push `10e90e36`, gitleaks-action `ff98106e`,
  cosign-installer `398d4b0e` (full SHAs in the workflows). ci.yml's
  "pin later" header TODO is replaced with the fact.
- Gitops-branch seed prepared at
  `/tmp/bidlobot-gitops-hardening/deploy/kustomization.yaml`: the
  current branch file plus `digest:
  sha256:80694b276a54116906b6bbfe7a4f0a7ca568b26593f4a4cbf625e1978de5cea9`
  (live digest of `master-ed97cc3-f9e471d3`, fetched from the
  anonymous GHCR token + manifest API) and a one-line comment that CI
  rewrites newTag+digest and the digest pins what runs.
- Docs: README Image provenance deploy paragraph and 70_deployment's
  kustomization bullet + deploy-path paragraph describe the chain,
  digest pinning and both closed holes.

## Why

- A red ci.yml can no longer deploy: the publish chain carries its own
  test gate.
- An image that cannot boot never reaches the cluster: smoke runs the
  pushed digest before the manifest job exists.
- Tag-overwrite in GHCR can no longer reach the cluster: the kustomize
  `images:` transformer with `digest:` renders
  `name:newTag@digest`, and the digest is content-addressed - Flux
  pulls exactly what CI pushed and cosign signed.
- Floating action tags in a public repo that deploys to prod are a
  takeover vector; commit SHAs freeze them at review time.

## Verification plan

- Structural review / `actionlint` of the restructured `publish.yml`
  and the pinned `ci.yml` (parent gates; this slice made no git or
  build runs).
- `kustomize build` of the seeded `deploy/` renders the image as
  `ghcr.io/arsolitt/bidlobot:master-ed97cc3-f9e471d3@sha256:8069...`
  (parent, against the gitops branch after the seed lands).
- Live: one master push must run test -> image -> smoke -> sign ->
  update-manifest in order, push a gitops commit carrying both the new
  `newTag` and `digest` lines, and Flux must roll to the digest-pinned
  image; `cosign verify` against the repo's GitHub identity on the
  pushed digest.

## Negatives

- `/livez` is unprobed in CI (GetMe gate; probing it would need either
  a real token - a second poller - or a Telegram API mock, or moving
  the listener earlier in `main.go`, which changes crash-loop
  semantics and was left to the owner).
- Until the seed lands on the `gitops` branch, the first CI run's
  `digest` sed is a no-op (no `digest:` line yet) and the deploy stays
  tag-pinned for exactly one bump.
- The cosign signature is created but nothing in the deploy path
  verifies it yet; the digest pin is the load-bearing guarantee.
- SHA pinning freezes the tag-resolution point in time; updates are
  now a manual re-resolution chore.
- Rollout latency is unchanged (~6 min worst case: 5m GitRepository +
  1m Kustomization interval).
