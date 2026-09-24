# YellowDot changes on top of upstream signals-dpg

This fork carries the changes the YellowDot eval instance (EC2 `signal-dpg`) needs on top of
`Blue-Dots-Economy/signals-dpg`. They live on the `yellowdot/connect-flow` branch; `main` mirrors
upstream. **Deploy and pull from `yellowdot/connect-flow`**, never from `main`, or these changes
are dropped.

## What is changed

| File | Change |
|---|---|
| `examples/schemas/yellow_dot/network.json` | The `onest_yellow_dot` network: 26 YellowDot domains (`yellowdot_learner_<cat>` / `yellowdot_provider_<cat>`), plus `connect` interactions between every learner/provider pair in both directions (`created`/`accepted`/`rejected`/`cancelled`, `metric_categories`, `reveals_pii_on_status: ["accepted"]`). |
| `apps/api/scripts/seed_service_users.ts` | Adds the `yellowdot-backend` service user. |
| `apps/api/src/routes/v1/action/_resolve_acting_actor.ts` | New `resolve_caller_id` + `x-acting-as-user-id` header. |
| `apps/api/src/routes/v1/action/update_action_status.ts` | Runs as the user named in `x-acting-as-user-id` when an acting org sends it. |
| `apps/api/src/routes/v1/action/get_action_contact_details.ts` | Same, for contact reveal; the `pii_reveal_audit` row records that user. |

### `x-acting-as-user-id`
Upstream makes `update-status` and `contact-details` self-acted only, so a channel that owns the
whole registration flow cannot drive an accept or a reveal for its users. With this header, an acting
org (`x-acting-org-id`, tier `network_service` / `aggregator` / `voice`) runs the request as one of
its users, checked by the same matrix `/action/perform` applies to `acting_as_user_id`
(aggregators only for users they onboarded). Without the header behaviour is unchanged; with it but
no acting org the request is refused (`CANNOT_OVERRIDE_SELF`).

## Consent
`connect` needs `consent: { acknowledged: true, version: 1 }` on `/action/perform` (initiator) and on
the `accepted` `/action/update-status` (receiver). The statements are in the network consent config;
the channel must show them to the user first.

## Deploying on the eval box

```bash
cd ~/signals-dpg
git fetch fork && git rebase fork/yellowdot/connect-flow      # or: git pull --rebase fork yellowdot/connect-flow
cd local-setup
docker build -f /tmp/Dockerfile.api.publicnode -t signals-dpg/api:local ..   # see below
docker compose up -d --no-deps --no-build signals-api
```

The API image is built on `dhi.io` hardened Node images, which need `docker login dhi.io` with a
Docker Hub account. The box has no such login, so it builds from a temporary copy of
`apps/api/Dockerfile` with the base swapped for the public image:

```bash
sed -e 's#dhi.io/node:24-alpine-dev#node:24-alpine#; s#dhi.io/node:24-alpine#node:24-alpine#' \
    apps/api/Dockerfile > /tmp/Dockerfile.api.publicnode
```

(Run `docker build` from the repo root: `docker build -f /tmp/Dockerfile.api.publicnode -t signals-dpg/api:local .`.)
Do not use `docker compose up signals-bootstrap`/full `up` for a code-only change: it re-runs
`drizzle-kit push --force`.

## Syncing with upstream
```bash
git fetch origin && git rebase origin/main   # origin = Blue-Dots-Economy/signals-dpg
git push --force-with-lease fork yellowdot/connect-flow
```
