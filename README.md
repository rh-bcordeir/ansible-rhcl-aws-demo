# ansible-rhcl-aws-demo

<p align="center">
  <img src="rhcl_architecture.png" alt="RHCL demo request flow: App Web / App Mobile → Gateway → AuthPolicy → RateLimitPolicy → HTTPRoute → Movie Quarks" width="520">
</p>

The diagram above shows the request flow this demo builds with RHCL: the **App Web**
and **App Mobile** clients (each with its own API key) hit the **Gateway**, where the
**AuthPolicy** authenticates/identifies the caller and the **RateLimitPolicy** enforces
per-identity limits; the **HTTPRoute** then routes `/api/v1/movies` to the **Movie
Quarks** service. Teal blocks are Gateway API (routing); purple blocks are RHCL policies.

Ansible automation that bootstraps a **Red Hat Connectivity Link (RHCL / Kuadrant)**
API-gateway demo on OpenShift, on top of **OpenShift GitOps (ArgoCD)** and
**OpenShift Service Mesh 3 (Sail/Istio)**.

It installs the operators (skipping any that are already present), wires up the
Service Mesh control plane, deploys the `movies-quarkus` demo app through ArgoCD,
and creates the Gateway, HTTPRoute, AuthPolicies (deny-by-default + API-key) and
RateLimitPolicy described in [`instructions.md`](instructions.md).

---

## ⚠️ Prerequisite: you must be logged into the cluster

Everything runs against **your current `oc` / kubeconfig context**. Before running
anything, log into the target cluster:

```bash
oc login --token=<token> --server=https://api.<cluster>:6443
# verify:
oc whoami
oc whoami --show-server
```

You need **cluster-admin** (operator installation creates cluster-scoped
Subscriptions, OperatorGroups and CRDs). The playbook fails fast in the
`preflight` role if you are not logged in.

---

## What gets installed

| Component | Operator (package) | Namespace | Channel / Catalog |
|-----------|--------------------|-----------|-------------------|
| OpenShift GitOps | `openshift-gitops-operator` | `openshift-operators` | `latest` / redhat-operators |
| Service Mesh 3 | `servicemeshoperator3` | `openshift-operators` | `stable` / redhat-operators |
| Red Hat Connectivity Link | `rhcl-operator` | **`kuadrant-system`** | `stable` / redhat-operators |

> RHCL is installed into its own **`kuadrant-system`** namespace (its own
> OperatorGroup), as requested. RHCL depends on **cert-manager**, which is
> expected to already be present on the cluster.
>
> The RHCL **web-console plugin** (`kuadrant-console-plugin`) is enabled
> automatically by adding it to the cluster Console operator config (it is not
> enabled by the operator install on its own). Existing console plugins are
> preserved.

Custom resources created:

- `Kuadrant/kuadrant` in `kuadrant-system`
- `Istio/default` (→ `istio-system`) and `IstioCNI/default` (→ `istio-cni`)
- ArgoCD `Application/movies-quarkus` (deploys the Helm chart into `cinema`)
- `Gateway/ingress-gateway` in `api-gateway`
- Three `HTTPRoute`s in `cinema` (named `<app>-<endpoint>`):
  - `movies-quarkus-movies` — public movies endpoints (`/api/v1/movies`, `/drama`, `/comedia`)
  - `movies-quarkus-directors` — restricted `/api/v1/directors` endpoint
  - `movies-quarkus-swagger` — public OpenAPI endpoint (`/q/openapi`)
- `AuthPolicy` deny-all on the Gateway, plus per-route policies (`<route>-auth`):
  - `movies-quarkus-movies-auth` — any valid API key
  - `movies-quarkus-directors-auth` — **web-app only** (other identities get 403)
  - `movies-quarkus-swagger-auth` — **public** (anonymous; overrides the deny-all on `/q/openapi`)
- Per-app API-key `Secret`s in `kuadrant-system` (`web-app`, `mobile-app`)
- `RateLimitPolicy/movies-quarkus-movies-rlp` on the movies route

> **Why separate HTTPRoutes instead of one with named rules + `sectionName`?**
> Targeting an individual HTTPRoute *rule* by name (`sectionName`) is an
> **experimental** Gateway API feature. OpenShift ships the **standard** Gateway
> API channel (managed by the cluster ingress-operator), where route rules cannot
> be named, so the apiserver prunes rule names and `sectionName` targeting fails
> with `TargetNotFound`. Splitting the endpoints into separate routes — each
> targeted as a whole by its AuthPolicy — gives the same behaviour on a standard
> cluster.
>
> **Note:** only `/q/openapi` (the OpenAPI spec) is exposed publicly. To also
> expose the Swagger UI, add a `{ type: PathPrefix, value: /q/swagger-ui }` match
> to `swagger_route_matches` in `group_vars/all.yml`.

---

## Requirements (control machine)

- `oc` CLI, logged in (see above)
- `ansible-core` (tested with 2.18)
- Python `kubernetes` client library
- Ansible collections from `requirements.yml`

```bash
ansible-galaxy collection install -r requirements.yml
# if the kubernetes python lib is missing:
pip install kubernetes
```

---

## Usage

Run the whole thing:

```bash
ansible-playbook main.yml
```

Run only parts of it with tags:

```bash
ansible-playbook main.yml --tags preflight    # read-only connectivity checks
ansible-playbook main.yml --tags operators    # install/verify the 3 operators
ansible-playbook main.yml --tags gitops        # just OpenShift GitOps
ansible-playbook main.yml --tags mesh          # Service Mesh operator + Istio CRs
ansible-playbook main.yml --tags rhcl,kuadrant # RHCL operator + Kuadrant CR + secrets
ansible-playbook main.yml --tags console       # enable the RHCL web-console plugin
ansible-playbook main.yml --tags demo-app      # ArgoCD Application only
ansible-playbook main.yml --tags gateway,auth,ratelimit  # routing + policies
```

The `preflight` and `info` roles always run.

---

## Idempotency / "is it already installed?"

Before creating a Subscription, the `olm_operator` role queries the existing
`ClusterServiceVersion`s in the operator namespace. If one matching the operator
is already present, it **skips** the Subscription/OperatorGroup creation and just
waits for it to be `Succeeded`. Re-running the playbook is safe — all custom
resources are applied declaratively.

---

## Things you will likely need to confirm / tweak

Edit [`group_vars/all.yml`](group_vars/all.yml):

- **`app_service_name`** (default `movies-quarkus`) — the HTTPRoute backend points
  here. After ArgoCD syncs the Helm chart, confirm the real Service name:
  ```bash
  oc get svc -n cinema
  ```
- **`demo_app.path`** (default `chart`) and **`demo_app.value_files`**
  (default `values/values-dev.yaml`, relative to the chart path) — the Helm
  chart location inside
  [movies-quarkus-devops](https://github.com/rh-bcordeir/movies-quarkus-devops).
  If the values file lives at the repo root, use `../values/values-dev.yaml`.
- **`gateway_class`** (default `istio`) — uses Service Mesh. Set to
  `openshift-default` to instead use the built-in OpenShift Gateway controller
  (the playbook then creates the `openshift-default` GatewayClass and skips the
  Service Mesh control-plane CRs).
- **`api_keys`** — the demo keys `CHAVE_DO_APP_WEB` / `CHAVE_DO_APP_MOBILE`.
  Change these for anything beyond a throwaway demo.

---

## Testing the demo

After a successful run, the `info` role prints the AWS LoadBalancer hostname and
ready-to-paste `curl` commands. The LB may take a minute to provision:

```bash
oc get svc -n api-gateway          # wait for an EXTERNAL-IP / hostname

GW=http://<lb-hostname>

# Denied by default (no key):
curl -i $GW/api/v1/movies

# Allowed with an API key:
curl -i -H 'Authorization: APIKEY CHAVE_DO_APP_WEB' $GW/api/v1/movies

# Rate limit — mobile is 2 req / 10s, should start returning 429:
for i in $(seq 1 6); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H 'Authorization: APIKEY CHAVE_DO_APP_MOBILE' $GW/api/v1/movies
done
```

> If the deny-all AuthPolicy does not take effect, roll out the controller as the
> instructions note:
> `oc rollout restart deploy/kuadrant-operator-controller-manager -n kuadrant-system`

---

## Layout

```
main.yml                 # orchestrator playbook
group_vars/all.yml       # all tunables
inventory.yml            # localhost (runs against your kubeconfig)
requirements.yml         # Ansible collections
roles/
  preflight/             # verify oc login, python k8s lib, Gateway API CRDs
  olm_operator/          # generic, idempotent OLM operator installer
  console_plugins/       # enable the RHCL web-console plugin
  kuadrant/              # Kuadrant CR + API-key secrets
  mesh/                  # Istio + IstioCNI control plane (Service Mesh 3)
  demo_app/              # ArgoCD Application (movies-quarkus)
  gateway/               # Gateway, HTTPRoute, AuthPolicies, RateLimitPolicy
  info/                  # prints endpoint + test commands
```
