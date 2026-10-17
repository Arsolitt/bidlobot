# Devlog 15 - 2026-10-17: deploy payload moved to the `gitops` branch (Flux + ExternalSecret)

## Trigger

The owner asked to move the bot's Kubernetes payload out of the private
homelab flux repo (`apps/bidlobot/`, sops-decrypted secret) into this
public repository: deploys driven from a dedicated orphan `gitops`
branch, the secret carried by an ExternalSecret referencing the
existing Bitwarden ClusterSecretStore, amd64 image build unchanged.

## What changed

- New orphan `gitops` branch holding only `deploy/`:
  `deployment.yaml` and `pvc.yaml` are verbatim copies of the homelab
  manifests except the image line — `ghcr.io/arsolitt/bidlobot:master`
  is a placeholder rewritten by the kustomize `images:` transformer.
  `externalsecret.yaml` maps the three Bitwarden keys
  (`bidlobot-tg-bot-token`, `bidlobot-bot-owner-id`,
  `bidlobot-deepseek-api-key`) to the `bidlobot-secrets` Secret the
  Deployment `envFrom`s (`deletionPolicy: Retain`, pure reference, no
  values). `kustomization.yaml` seeds `newTag: master-ed97cc3-f0963759`
  (the running tag) — the single line CI rewrites. `README.md` keeps
  the bootstrap-restore and backup runbooks plus a new Deploy section.
- `.github/workflows/publish.yml`: the `image` job exports `TAG` as a
  job output (`steps.tag.outputs.TAG`); a new `update-manifest` job
  (needs `image`, `permissions: contents: write`) checks out `gitops`,
  runs `sed -i "s|newTag: .*|newTag: $TAG|" deploy/kustomization.yaml`,
  and pushes `chore(deploy): bump image tag to <tag>` when the line
  changed. GITHUB_TOKEN pushes never trigger workflows, so there is no
  CI loop back into publish.
- `README.md`: Image provenance gains the gitops-branch deploy flow.
- `70_deployment.md`: new "Kubernetes deploy (production)" section
  (GitRepository -> gitops branch `deploy/` -> ExternalSecret /
  Bitwarden); the compose content stays as the local flow.

## Why

The private flux-repo payload duplicated manifests this repo already
describes and tied the deploy to a sops-encrypted secret in another
tree. With the payload on the public `gitops` branch, a push to
`master` is the entire deploy path: build + push image -> bump `newTag`
on `gitops` -> Flux reconciles (GitRepository `bidlobot`, interval 5m —
public repo, no PAT, GitHub unauthenticated budget 60 req/h per IP;
owner Kustomization `path: deploy`, interval 1m, `prune: false`,
`wait: true`) -> Recreate rolls the single poller. Secret values stay
out of git entirely; the transition is tag-identical, so the first
reconcile rolls nothing.

## Verification plan

- `kustomize build --load-restrictor LoadRestrictionsNone deploy/`
  renders Deployment + PVC + ExternalSecret with the expected tag and
  store refs.
- `actionlint` or structural review of `publish.yml`.
- `kubectl --context arsolitt@homelab apply --dry-run=server -f` on the
  rendered output.
- After the branch and the homelab wiring land: ExternalSecret Ready,
  `bidlobot-secrets` has the three keys, pod Running with no restart
  storm, then one end-to-end master-push bump observed.

## Negatives

- Rollout latency: a master push lands at most ~6 min after the CI push
  (5m GitRepository interval + 1m Kustomization) — acceptable for a
  single-tenant bot.
- No PAT on the GitRepository (public repo), so the Flux interval must
  respect GitHub's unauthenticated request budget.
- Homelab-side wiring (GitRepository + owner Kustomization + deletion
  of the old `apps/bidlobot/` payload) lives in the homelab gitops repo
  and is out of this repo's tree.
