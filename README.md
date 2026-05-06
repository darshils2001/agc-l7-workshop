# AGC + AKS multi-site workshop — Cloud Shell walkthrough

A self-paced, hands-on walkthrough for **Application Gateway for Containers (AGC)** and **Advanced Container Networking Services (ACNS) L7** on **Azure Kubernetes Service**. Every command runs in **Azure Cloud Shell** (<https://shell.azure.com> or the `>_` icon in the portal — `az`, `kubectl`, and `curl` are pre-installed). No local tooling, no `git`, every manifest inline.

> **The framing in one line:** **AGC brings traffic *into* the cluster. ACNS L7 controls how traffic flows *within* the cluster.** Two features, two directions, one zero-trust story.

## What you'll build

By the end of this walkthrough you'll have:

- An AKS cluster with **Azure CNI Overlay + Cilium dataplane + ACNS L7** enabled.
- A managed **Application Gateway for Containers** frontend with a public FQDN, provisioned automatically from a single Kubernetes CR.
- A `Gateway` plus three `HTTPRoute`s that put **three sample tenant sites behind one AGC frontend** (host-based routing).
- Four `CiliumNetworkPolicy` objects implementing **default-deny + HTTP-method-and-path L7 enforcement** at every pod.
- Optionally, an **Azure WAF policy bound to AGC** that blocks OWASP-class attacks at the edge before they reach your pods.

## What you'll demonstrate

| Test | Layer | Direction |
|---|---|---|
| 4a Multi-site routing | AGC | Internet → cluster |
| 4a-bonus Edge WAF blocks SQLi / path-traversal | AGC + Azure WAF (DRS 2.1) | Internet → AGC (request never reaches pod) |
| 4b GET vs POST/PUT/DELETE, `/products` vs `/admin` | ACNS L7 at the pod | AGC → pod (north-south behind AGC) |
| 4c Pod → Pod with method enforcement | ACNS L7 east-west | Pod ↔ pod inside cluster |
| 4d Pod → public internet blocked | ACNS default-deny egress | Pod → internet |
| 4e DNS still resolves under default-deny | ACNS DNS carve-out | Pod → kube-dns |
| 5 Live drop monitor | ACNS observability | Whatever you generate |

---

## 0. Set variables and pick your subscription

Set these to whatever subscription / region / resource group / cluster name you want to use. Pick a region where AGC is generally available and your subscription has capacity for a small AKS cluster. The rest of this guide references these variables, so you only edit them once.

```bash
export SUBSCRIPTION_ID="<your-subscription-id>"
export LOCATION="<your-region>"               # e.g. eastus, westus3, westeurope
export RESOURCE_GROUP="<your-resource-group>"
export AKS_NAME="<your-aks-name>"
export ALB_NAMESPACE="alb-demo"
export ALB_NAME="alb-demo"
export APP_NAMESPACE="agc-sites"

az account set --subscription "$SUBSCRIPTION_ID"
```

> **Region tip:** AGC is multi-region, but during this walkthrough's build `westus3` was reliable while `eastus2` returned transient AGC subnet-association errors (`Microsoft.ServiceNetworking`). If `kubectl wait` on the `ApplicationLoadBalancer` in step 3b sits on `Updating` for >10 min, that's typically a regional backend issue — switch regions and retry from step 0.

---

## 1. Register providers, install CLI extensions, create RG

One-time per subscription. Registers the four resource providers AGC + ACNS need, the `AdvancedNetworkingPreview` feature flag (gates ACNS L7), the `aks-preview` and `alb` CLI extensions, and the parent resource group.

```bash
for rp in Microsoft.ContainerService Microsoft.Network Microsoft.ServiceNetworking Microsoft.OperationsManagement; do
  az provider register --namespace "$rp" --wait
done

az feature register --namespace Microsoft.ContainerService --name AdvancedNetworkingPreview
az provider register --namespace Microsoft.ContainerService

az extension add --name aks-preview --upgrade --yes
az extension add --name alb         --upgrade --yes

az group create -n "$RESOURCE_GROUP" -l "$LOCATION"
```

---

## 2. Create the AKS cluster (~7 min)

Single `az aks create` that turns on every relevant feature: Cilium dataplane, ACNS L7, AGC add-on, Gateway API CRDs, OIDC issuer + workload identity.

| Flag | What it does |
|---|---|
| `--network-plugin azure --network-plugin-mode overlay` | Pods get IPs from a non-routable overlay; keeps your VNet plan small. |
| `--network-dataplane cilium` | Replaces kube-proxy/iptables with Cilium's eBPF dataplane. Required for L7. |
| `--enable-acns --acns-advanced-networkpolicies L7` | Turns on ACNS L7 — Cilium now understands HTTP method/path/header rules. |
| `--enable-application-load-balancer` | Installs the AGC add-on (`alb-controller` pods in `kube-system`). AKS owns the controller image, the workload-identity federation, the delegated subnet (`aks-appgateway` in the `MC_*` RG), and the AGC resource. There's no Helm chart and no manual identity glue — but you also can't customize the controller image. |
| `--enable-gateway-api` | Installs upstream Gateway API CRDs and registers `azure-alb-external` GatewayClass. |
| `--enable-oidc-issuer --enable-workload-identity` | Required so AGC's controller authenticates to Azure as a workload identity. |

```bash
az aks create \
  -g "$RESOURCE_GROUP" -n "$AKS_NAME" -l "$LOCATION" \
  --kubernetes-version 1.34.4 \
  --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium \
  --enable-acns --acns-advanced-networkpolicies L7 \
  --enable-gateway-api --enable-application-load-balancer \
  --enable-oidc-issuer --enable-workload-identity \
  --node-vm-size Standard_D4s_v5 --node-count 2 \
  --ssh-access disabled --generate-ssh-keys

az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_NAME" --overwrite-existing
kubectl get nodes
kubectl -n kube-system get pods -l app=alb-controller
kubectl get gatewayclass azure-alb-external
```

Verify three things before continuing: 2 Ready nodes, 2 Running `alb-controller` pods, GatewayClass `azure-alb-external` with `Accepted=True`.

---

## 3. Deploy everything (manifests inline)

Five sub-steps. Each one introduces a new layer:

- **3a Namespaces** — ownership boundary between AGC config (`alb-demo`) and workloads + policies (`agc-sites`).
- **3b ApplicationLoadBalancer CR** — applying this CR is what makes Azure provision the AGC frontend.
- **3c Sample apps + client pod** — three nginx tenants plus a curl pod for east-west tests.
- **3d Gateway + HTTPRoutes** — Gateway API objects that AGC translates into routing config.
- **3e Cilium policies** — four `CiliumNetworkPolicy` objects: default-deny + DNS carve-out + L7 ingress allow-list + east-west allow-list.

### 3a. Namespaces

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata: { name: alb-demo }
---
apiVersion: v1
kind: Namespace
metadata: { name: agc-sites }
EOF
```

### 3b. Tell AGC to provision a managed frontend (~5 min)

`spec.associations: []` (an *empty list*) is the magic for **managed-by-ALB** mode. Empty list = "AKS, please create the subnet, the AGC, the workload identity federation, everything, on my behalf." The instant this CR is applied, AKS:

1. carves a `/24` out of the cluster VNet called `aks-appgateway`,
2. delegates that subnet to `Microsoft.ServiceNetworking/TrafficController`,
3. provisions the AGC Azure resource (`alb-<hash>`) in the auto-created `MC_` resource group,
4. associates the new subnet to the new AGC,
5. federates the AGC's workload identity with the cluster's OIDC issuer.

```bash
kubectl apply -f - <<EOF
apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: $ALB_NAME
  namespace: $ALB_NAMESPACE
spec:
  associations: []
EOF

kubectl wait --for=condition=Deployment=True \
  applicationloadbalancer/$ALB_NAME -n $ALB_NAMESPACE --timeout=10m
```

If `kubectl wait` sits on `Updating` for >10 min, you've likely hit a transient regional backend issue. Pick a different region and retry from step 0.

### 3c. Three sample sites + a client pod

Three nginx pods each fronted by a ClusterIP Service on port 8080, each serving a unique HTML body so you can prove which backend served a given response. Plus a `curlimages/curl` pod (`client`) that just sleeps — used for east-west tests in step 4c.

Each backend has labels `app: <name>` and `site: <name>`. The L7 policy in 3e selects on `site IN [contoso, fabrikam, adventure]`, so adding a fourth tenant is one label away.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata: { name: contoso-html, namespace: agc-sites }
data:
  index.html: |
    <html><body><h1>Hello from Contoso</h1><p>Served from contoso.example backend</p></body></html>
  default.conf: |
    server { listen 8080; root /usr/share/nginx/html; index index.html; }
---
apiVersion: v1
kind: ConfigMap
metadata: { name: fabrikam-html, namespace: agc-sites }
data:
  index.html: |
    <html><body><h1>Hello from Fabrikam</h1><p>Served from fabrikam.example backend</p></body></html>
  default.conf: |
    server { listen 8080; root /usr/share/nginx/html; index index.html; }
---
apiVersion: v1
kind: ConfigMap
metadata: { name: adventure-html, namespace: agc-sites }
data:
  index.html: |
    <html><body><h1>Hello from Adventure Works</h1><p>Served from adventure.example backend</p></body></html>
  default.conf: |
    server { listen 8080; root /usr/share/nginx/html; index index.html; }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: contoso, namespace: agc-sites, labels: { app: contoso, site: contoso } }
spec:
  replicas: 1
  selector: { matchLabels: { app: contoso } }
  template:
    metadata: { labels: { app: contoso, site: contoso } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports: [{ containerPort: 8080 }]
          volumeMounts:
            - { name: html, mountPath: /usr/share/nginx/html }
            - { name: conf, mountPath: /etc/nginx/conf.d }
      volumes:
        - name: html
          configMap: { name: contoso-html, items: [{ key: index.html, path: index.html }] }
        - name: conf
          configMap: { name: contoso-html, items: [{ key: default.conf, path: default.conf }] }
---
apiVersion: v1
kind: Service
metadata: { name: contoso, namespace: agc-sites }
spec:
  selector: { app: contoso }
  ports: [{ port: 8080, targetPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: fabrikam, namespace: agc-sites, labels: { app: fabrikam, site: fabrikam } }
spec:
  replicas: 1
  selector: { matchLabels: { app: fabrikam } }
  template:
    metadata: { labels: { app: fabrikam, site: fabrikam } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports: [{ containerPort: 8080 }]
          volumeMounts:
            - { name: html, mountPath: /usr/share/nginx/html }
            - { name: conf, mountPath: /etc/nginx/conf.d }
      volumes:
        - name: html
          configMap: { name: fabrikam-html, items: [{ key: index.html, path: index.html }] }
        - name: conf
          configMap: { name: fabrikam-html, items: [{ key: default.conf, path: default.conf }] }
---
apiVersion: v1
kind: Service
metadata: { name: fabrikam, namespace: agc-sites }
spec:
  selector: { app: fabrikam }
  ports: [{ port: 8080, targetPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: adventure, namespace: agc-sites, labels: { app: adventure, site: adventure } }
spec:
  replicas: 1
  selector: { matchLabels: { app: adventure } }
  template:
    metadata: { labels: { app: adventure, site: adventure } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports: [{ containerPort: 8080 }]
          volumeMounts:
            - { name: html, mountPath: /usr/share/nginx/html }
            - { name: conf, mountPath: /etc/nginx/conf.d }
      volumes:
        - name: html
          configMap: { name: adventure-html, items: [{ key: index.html, path: index.html }] }
        - name: conf
          configMap: { name: adventure-html, items: [{ key: default.conf, path: default.conf }] }
---
apiVersion: v1
kind: Service
metadata: { name: adventure, namespace: agc-sites }
spec:
  selector: { app: adventure }
  ports: [{ port: 8080, targetPort: 8080 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: client, namespace: agc-sites, labels: { app: client, role: client } }
spec:
  replicas: 1
  selector: { matchLabels: { app: client } }
  template:
    metadata: { labels: { app: client, role: client } }
    spec:
      containers:
        - name: curl
          image: curlimages/curl:8.10.1
          command: ["sleep", "infinity"]
EOF

for d in contoso fabrikam adventure client; do
  kubectl -n agc-sites rollout status deployment/$d
done
```

### 3d. Gateway + 3 HTTPRoutes (multi-site)

One `Gateway` (`gateway-01`) with a single HTTP listener on port 80, plus three `HTTPRoute`s — one per hostname, each pointing at a different backend Service. **One Gateway, one public IP, three sites.** Adding a fourth tenant is just one more `HTTPRoute`.

The annotations link the Gateway to the `ApplicationLoadBalancer` from 3b — that's how the controller knows which AGC resource to program.

You don't need to own these hostnames; we use `curl --resolve` later to forge the `Host:` header. In a real deployment you'd point DNS A/AAAA records at the AGC FQDN.

> **If your `HTTPRoute` and backend `Service` live in different namespaces**, add a `ReferenceGrant` in the Service's namespace so the Route is allowed to refer across the boundary. This walkthrough keeps both in `agc-sites` so a grant isn't needed, but production multi-tenant setups usually need one.

```bash
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-01
  namespace: $APP_NAMESPACE
  annotations:
    alb.networking.azure.io/alb-namespace: $ALB_NAMESPACE
    alb.networking.azure.io/alb-name: $ALB_NAME
spec:
  gatewayClassName: azure-alb-external
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes: { namespaces: { from: Same } }
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: contoso-route, namespace: $APP_NAMESPACE }
spec:
  parentRefs: [{ name: gateway-01 }]
  hostnames: ["contoso.example.com"]
  rules:
    - backendRefs: [{ name: contoso, port: 8080 }]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: fabrikam-route, namespace: $APP_NAMESPACE }
spec:
  parentRefs: [{ name: gateway-01 }]
  hostnames: ["fabrikam.example.com"]
  rules:
    - backendRefs: [{ name: fabrikam, port: 8080 }]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: adventure-route, namespace: $APP_NAMESPACE }
spec:
  parentRefs: [{ name: gateway-01 }]
  hostnames: ["adventure.example.com"]
  rules:
    - backendRefs: [{ name: adventure, port: 8080 }]
EOF
```

### 3e. Cilium L7 policies

Four `CiliumNetworkPolicy` (CNP) objects. CNP is Cilium's superset of Kubernetes `NetworkPolicy` — it speaks not just "allow port X from pod Y" but also "allow HTTP method M on path P." Cilium policies are **additive whitelists**, so we layer them:

| # | Policy | What it does |
|---|---|---|
| 1 | `default-deny-all` | Empty selector + `ingress: [{}]` and `egress: [{}]` flips every pod in the namespace into default-deny. **Use `[{}]`, not `[]` — `[]` shows `VALID=False`.** |
| 2 | `allow-dns-egress` | Carve-out so pods can resolve service names via kube-dns. The `dns: matchPattern: "*"` makes Cilium parse and inspect actual DNS queries. |
| 3 | `allow-agc-l7-get-only` | For pods labelled `site IN [contoso, fabrikam, adventure]`: allow ingress from `world` AND `cluster` (covers AGC and east-west) but **only `GET /` and `GET /products` on port 8080**. Anything else → Cilium returns 403 *before nginx ever sees it*. |
| 4 | `client-may-call-contoso-get-only` | Pod with `app: client` may egress to pod with `app: contoso` on `GET /` only. Both this policy AND policy 3 must allow the call (additive). |

Why include `cluster` in `fromEntities`: AGC routes traffic through a node-local hop that Cilium identifies as `cluster`, not `world`. If you only allow `world`, the GET sometimes returns 403 even though the L7 rule matches. List both.

```bash
kubectl apply -f - <<'EOF'
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: default-deny-all, namespace: agc-sites }
spec:
  endpointSelector: {}
  ingress:
    - {}
  egress:
    - {}
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: allow-dns-egress, namespace: agc-sites }
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - matchLabels:
            "k8s:io.kubernetes.pod.namespace": kube-system
            "k8s:k8s-app": kube-dns
      toPorts:
        - ports: [{ port: "53", protocol: ANY }]
          rules:
            dns: [{ matchPattern: "*" }]
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: allow-agc-l7-get-only, namespace: agc-sites }
spec:
  endpointSelector:
    matchExpressions:
      - { key: site, operator: In, values: [contoso, fabrikam, adventure] }
  ingress:
    - fromEntities: [world, cluster]
      toPorts:
        - ports: [{ port: "8080", protocol: TCP }]
          rules:
            http:
              - { method: "GET", path: "/" }
              - { method: "GET", path: "/products" }
---
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata: { name: client-may-call-contoso-get-only, namespace: agc-sites }
spec:
  endpointSelector: { matchLabels: { app: client } }
  egress:
    - toEndpoints: [{ matchLabels: { app: contoso } }]
      toPorts:
        - ports: [{ port: "8080", protocol: TCP }]
          rules:
            http:
              - { method: "GET", path: "/" }
EOF

kubectl get cnp -n $APP_NAMESPACE
```

All 4 should show `VALID=True`.

---

## 4. Run the tests

### Set up the test variables

Grab the AGC FQDN once. Every test below pins to this IP via `curl --resolve` since you don't own `*.example.com`:

```bash
FQDN=$(kubectl get gateway gateway-01 -n $APP_NAMESPACE -o jsonpath='{.status.addresses[0].value}')
IP=$(getent hosts "$FQDN" | awk '{print $1}' | head -1)
echo "$FQDN -> $IP"
```

Both values should be populated. The IP is in a Microsoft-owned range; in production always use the FQDN, not the IP.

### 4a. Multi-site routing (AGC)

One AGC public FQDN, three different hostnames, three different backend pods. AGC routes each request based purely on the `Host:` header.

```bash
for h in contoso fabrikam adventure; do
  echo "[$h.example.com]"
  curl -s --resolve $h.example.com:80:$IP http://$h.example.com/
  echo
done
```

**Expected output:**

```text
[contoso.example.com]
<html><body><h1>Hello from Contoso</h1><p>Served from contoso.example backend</p></body></html>

[fabrikam.example.com]
<html><body><h1>Hello from Fabrikam</h1><p>Served from fabrikam.example backend</p></body></html>

[adventure.example.com]
<html><body><h1>Hello from Adventure Works</h1><p>Served from adventure.example backend</p></body></html>
```

**What this proves:** same public IP for all three; only `Host:` differs. AGC is doing L7 hostname-based routing on a single managed frontend.

### 4a-bonus. Add Azure WAF to AGC

Azure WAF on AGC is an AGC-only capability — there's no DIY ingress controller path to it. Once enabled, AGC inspects each request against the Azure-managed **Default Rule Set (DRS) 2.1** (the only ruleset AGC WAF supports — no OWASP, no Bot Manager) and rejects malicious requests at the edge. The pod never sees them.

The wiring is two pieces:

1. An Azure-side `Microsoft.Network/ApplicationGatewayWebApplicationFirewallPolicies` resource that holds the rules.
2. A Kubernetes-side `WebApplicationFirewallPolicy` CRD that points the ALB Controller at it (scoped to the entire `Gateway`, a specific listener, or a specific `HTTPRoute`).

#### Why a single atomic ruleset swap

`az ... waf-policy create` requires `--type/--version` and only accepts OWASP. AGC WAF only supports DRS 2.1. You can't fix this in two steps because:

- `remove OWASP` fails with `NoValidPrimaryRuleSetsAttached` (a policy must always have one primary).
- `add DRS` fails with `HasMultiplePrimaryRuleSets` (can't add a second one).

So you create with the forced OWASP, then **swap the entire `managedRuleSets` array atomically** in one `update`.

#### Setup

```bash
# 1a. Create the policy if it doesn't already exist (idempotent — re-running create
#     against an already-attached policy fails with ApplicationGatewayFirewallAttachAGCUnsupportedRuleSetVersion).
if ! az network application-gateway waf-policy show \
      --name agc-waf-policy --resource-group "$RESOURCE_GROUP" >/dev/null 2>&1; then
  az network application-gateway waf-policy create \
    --name agc-waf-policy \
    --resource-group "$RESOURCE_GROUP" \
    --location "$LOCATION" \
    --type OWASP --version 3.2
else
  echo "WAF policy agc-waf-policy already exists — skipping create."
fi

# 1b. Atomically replace OWASP 3.2 with DRS 2.1.
az network application-gateway waf-policy update \
  --name agc-waf-policy --resource-group "$RESOURCE_GROUP" \
  --set "managedRules.managedRuleSets=[{\"ruleSetType\":\"Microsoft_DefaultRuleSet\",\"ruleSetVersion\":\"2.1\",\"ruleGroupOverrides\":[]}]"

# 1c. Set Prevention/Enabled.
az network application-gateway waf-policy update \
  --name agc-waf-policy --resource-group "$RESOURCE_GROUP" \
  --set policySettings.mode=Prevention policySettings.state=Enabled

# 1d. Sanity check — should show only Microsoft_DefaultRuleSet 2.1.
az network application-gateway waf-policy show \
  --name agc-waf-policy --resource-group "$RESOURCE_GROUP" \
  --query 'managedRules.managedRuleSets'

WAF_ID=$(az network application-gateway waf-policy show \
  --name agc-waf-policy --resource-group "$RESOURCE_GROUP" --query id -o tsv)
echo "$WAF_ID"

# 1e. Grant the ALB Controller's managed identity permission to "join" the WAF policy.
#     Without this, the CRD will sit in DeploymentFailed with LinkedAuthorizationFailed.
NODE_RG=$(az aks show -g "$RESOURCE_GROUP" -n "$AKS_NAME" --query nodeResourceGroup -o tsv)
echo "Node RG: $NODE_RG"
echo "Identities in node RG:"
az identity list -g "$NODE_RG" --query "[].{name:name,principalId:principalId}" -o table

# Pick the ALB controller identity. Naming varies across add-on versions:
#   - `applicationloadbalancer-<aks>` (current GA naming)
#   - `azurealb-<aks>` (older preview naming)
ALB_PRINCIPAL_ID=$(az identity list -g "$NODE_RG" \
  --query "[?starts_with(name, 'applicationloadbalancer') || starts_with(name, 'azurealb')].principalId | [0]" -o tsv)

# If empty, set it manually from the table above:
# ALB_PRINCIPAL_ID=<paste-objectid-here>
echo "ALB Controller identity: $ALB_PRINCIPAL_ID"

az role assignment create \
  --assignee-object-id "$ALB_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Network Contributor" \
  --scope "$WAF_ID"

# 2. Bind it to the Gateway via the WebApplicationFirewallPolicy CRD.
kubectl apply -f - <<EOF
apiVersion: alb.networking.azure.io/v1
kind: WebApplicationFirewallPolicy
metadata:
  name: agc-gateway-waf
  namespace: $APP_NAMESPACE
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: gateway-01
    namespace: $APP_NAMESPACE
  webApplicationFirewall:
    id: $WAF_ID
EOF

# 3. Wait for the controller to reconcile, then confirm Programmed=True.
sleep 30
kubectl get webapplicationfirewallpolicy -n $APP_NAMESPACE agc-gateway-waf \
  -o jsonpath='{.status.conditions[?(@.type=="Programmed")]}{"\n"}'
```

> **If you applied the CRD before granting the role assignment** (or had any other permission issue at first reconcile), the controller caches the failure on the CRD. Force a re-reconcile by deleting and re-applying:
>
> ```bash
> kubectl delete webapplicationfirewallpolicy -n $APP_NAMESPACE agc-gateway-waf
> # then re-run the kubectl apply block above
> ```

> **If `kubectl` returns `the server has asked for the client to provide credentials`**, your Cloud Shell session has lost the AKS credentials. Re-run:
>
> ```bash
> az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_NAME" --overwrite-existing
> ```

#### Run the tests

> **If you're in a fresh Cloud Shell tab**, re-export `$APP_NAMESPACE` and re-derive `$IP` first, otherwise curl returns `000`:
>
> ```bash
> export APP_NAMESPACE="agc-sites"
> FQDN=$(kubectl get gateway gateway-01 -n $APP_NAMESPACE -o jsonpath='{.status.addresses[0].value}')
> IP=$(getent hosts "$FQDN" | awk '{print $1}' | head -1)
> [ -z "$IP" ] && IP=$(dig +short "$FQDN" | head -1)
> echo "FQDN=$FQDN  IP=$IP"
> ```

```bash
# Benign — should still return 200.
curl -s -o /dev/null -w "benign      GET /                       -> %{http_code}\n" \
  --resolve contoso.example.com:80:$IP http://contoso.example.com/

# Malicious — path-traversal payload.
curl -s -o /dev/null -w "malicious   GET /?text=/etc/passwd      -> %{http_code}\n" \
  --resolve contoso.example.com:80:$IP "http://contoso.example.com/?text=/etc/passwd"

# Malicious — classic SQLi tautology.
curl -s -o /dev/null -w "malicious   GET /?id=1%20OR%201=1       -> %{http_code}\n" \
  --resolve contoso.example.com:80:$IP "http://contoso.example.com/?id=1%20OR%201=1"
```

**Expected output:**

```text
benign      GET /                       -> 200
malicious   GET /?text=/etc/passwd      -> 403
malicious   GET /?id=1 OR 1=1           -> 403
```

**What this proves:**

- The 200 confirms WAF doesn't break legitimate traffic.
- The two 403s came **from AGC**, not from Cilium. ACNS L7 wouldn't have caught either — both are GETs to `/`, which the L7 allow-list permits. Without AGC WAF, both malicious requests would have reached nginx.
- AGC WAF (signature-based, edge) and ACNS L7 (behavioral, pod-side) are complementary layers.

### 4b. ACNS L7 — north-south behind AGC

AGC forwards every request below to the contoso pod regardless of method or path. ACNS L7 at the pod is what decides which actually reach nginx.

```bash
for m in GET POST PUT DELETE; do
  curl -s -o /dev/null -w "$m / -> %{http_code}\n" \
    --max-time 10 -X $m --resolve contoso.example.com:80:$IP http://contoso.example.com/
done
for p in / /products /admin; do
  curl -s -o /dev/null -w "GET $p -> %{http_code}\n" \
    --max-time 10 --resolve contoso.example.com:80:$IP http://contoso.example.com$p
done
```

**Expected output:**

```text
GET / -> 200
POST / -> 403
PUT / -> 403
DELETE / -> 403
GET / -> 200
GET /products -> 404
GET /admin -> 403
```

**What this proves:**

| Line | Who decided | Why it matters |
|---|---|---|
| `GET / -> 200` | nginx | Happy path. ACNS allowed; nginx served. |
| `POST/PUT/DELETE / -> 403` | **ACNS** | Same port, wrong method. Vanilla L4 NetworkPolicy could not block this. |
| `GET /products -> 404` | **nginx** | ACNS *allowed* the path (it's whitelisted); nginx returned 404 because no such file. **Proves ACNS does real L7 inspection, not blanket-blocking.** |
| `GET /admin -> 403` | **ACNS** | Wrong path; rejected at the pod boundary before nginx saw it. |

The 403-vs-404 distinction is the headline: 403 is a Cilium-synthesized response (request never reached nginx); 404 is a real nginx response (ACNS forwarded; nginx had nothing to serve).

### 4c. ACNS L7 east-west — pod ↔ pod, no AGC involved

The same Cilium L7 rules also enforce on pod-to-pod traffic — even though AGC isn't in the data path.

```bash
CLIENT=$(kubectl get pod -n $APP_NAMESPACE -l app=client -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n $APP_NAMESPACE $CLIENT -- curl -s -o /dev/null -w "client->contoso GET  -> %{http_code}\n" --max-time 5 http://contoso:8080/
kubectl exec -n $APP_NAMESPACE $CLIENT -- curl -s -o /dev/null -w "client->contoso POST -> %{http_code}\n" --max-time 5 -X POST http://contoso:8080/
kubectl exec -n $APP_NAMESPACE $CLIENT -- curl -s --ipv4 -o /dev/null -w "client->fabrikam     -> %{http_code}\n" --max-time 5 http://fabrikam:8080/
```

**Expected output:**

```text
client->contoso GET  -> 200
client->contoso POST -> 403
client->fabrikam     -> 000
```

**What this proves:**

| Line | Who decided | Why it matters |
|---|---|---|
| `client->contoso GET -> 200` | nginx | Both `client-may-call-contoso-get-only` AND `allow-agc-l7-get-only` permit it. Both ends must agree (additive whitelists). |
| `client->contoso POST -> 403` | **ACNS L7** | Right pod, right port, wrong method — Cilium synthesized 403. |
| `client->fabrikam -> 000` | **ACNS L4** | No policy whitelists `client → fabrikam`. Default-deny dropped the SYN; TCP handshake never completed. |

**403 vs 000:** 403 means Cilium let the connection complete and rejected the HTTP request at L7. 000 means Cilium dropped the SYN at L4 — the destination might as well not exist. Different layer, same result: denied.

### 4d. ACNS default-deny egress — pod cannot call out

```bash
CONTOSO=$(kubectl get pod -n $APP_NAMESPACE -l app=contoso -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n $APP_NAMESPACE $CONTOSO -- wget -q -T 5 -O /dev/null https://www.bing.com
echo "rc=$?  (non-zero = blocked)"
```

**Expected output:**

```text
wget: download timed out
command terminated with exit code 1
rc=1  (non-zero = blocked)
```

**What this proves:** DNS resolution succeeded (`allow-dns-egress` carve-out), but the TCP connection to Bing's IP is silently dropped by Cilium because no rule whitelists egress to the public internet. AGC is irrelevant here — AGC handles ingress, not pod egress. ACNS owns this dimension entirely.

### 4e. ACNS DNS carve-out — lockdown is *precise*, not blunt

```bash
kubectl exec -n $APP_NAMESPACE $CLIENT -- nslookup contoso.agc-sites.svc.cluster.local
```

**Expected output (IPs will differ):**

```text
Server:         10.0.0.10
Address:        10.0.0.10:53


Name:   contoso.agc-sites.svc.cluster.local
Address: 10.0.37.14
```

**What this proves:** the client pod reaches kube-dns on port 53 because `allow-dns-egress` permits *exactly* that. Compare with 4d: same pod, same default-deny, but DNS works (allowed) while a TCP connection to Bing fails (not allowed). That's surgical enforcement, not a blunt firewall.

---

## 5. Live drop monitor

Watch ACNS enforcement happen in real time. `cilium monitor` reads the eBPF event ring buffer on the agent that hosts the target pod, surfacing every L7 verdict and L4 drop with full identity and HTTP context.

In one Cloud Shell tab:

```bash
kubectl -n kube-system exec -it ds/cilium -- cilium monitor --type drop
```

In another tab, send a denied request. **Cloud Shell tabs don't share environment variables** — re-export the basics first:

```bash
SUBSCRIPTION_ID="<your-subscription-id>"
RESOURCE_GROUP="<your-resource-group>"
AKS_NAME="<your-aks-name>"
APP_NAMESPACE="agc-sites"

az account set --subscription "$SUBSCRIPTION_ID"
az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_NAME" --overwrite-existing

FQDN=$(kubectl get gateway gateway-01 -n $APP_NAMESPACE -o jsonpath='{.status.addresses[0].value}')
IP=$(getent hosts "$FQDN" | awk '{print $1}' | head -1)

curl -X POST --resolve contoso.example.com:80:$IP http://contoso.example.com/
```

**Expected output in the monitor tab** (exact identity numbers and IPs differ per cluster):

```text
-> Request http from 0 ([reserved:world]) to 4521 ([k8s:app=contoso k8s:io.kubernetes.pod.namespace=agc-sites k8s:site=contoso]), identity 2->4521, verdict Denied POST http://contoso.example.com/ => 403
```

…and possibly an L4 drop event:

```text
xx drop (Policy denied) flow 0x4f3a2b1c to endpoint 4521, ifindex 12, file bpf_lxc.c:1843, , identity 2->4521: 10.224.0.5:42118 -> 10.244.1.7:8080 tcp ACK
```

**What this proves:**

- The decision is rendered **in the kernel** by Cilium's eBPF programs (note `file bpf_lxc.c:1843`).
- Cilium **synthesized the 403** — nginx never saw the request.
- Enforcement is **identity-based**, not IP-based (`identity 2->4521`). Pod restarts and reschedules don't break the rule.
- Every drop is observable through the same stream that feeds **Hubble** (and Hubble UI), **Container Insights**, and **Azure Monitor for AKS** — so you don't lose visibility when you turn on policy, you gain a dimension of it.

---

## 6. Tear it down

One command. Deletes the parent RG, which cascades to the AKS cluster, the auto-created `MC_` group, the AGC resource, the subnet, and all networking. `--no-wait` returns immediately; the actual delete takes a few minutes.

```bash
az group delete -n "$RESOURCE_GROUP" --yes --no-wait
```

---

## Notes for Cloud Shell

- **Idle timeout:** Cloud Shell disconnects after about 20 minutes of inactivity. On reconnect, re-run the variable block from step 0 plus `az aks get-credentials`.
- **Variables don't persist across sessions or tabs.** Each new tab needs the variables re-exported.
- **Persistent storage:** Cloud Shell mounts `~/clouddrive` if you want to save snippets to a file.

---

## Resources

- AKS + Cilium L7 policies — <https://learn.microsoft.com/azure/aks/how-to-apply-l7-policies> (short link: <https://aka.ms/aks/l7-policies>)
- AGC ALB Controller add-on quickstart — <https://learn.microsoft.com/azure/application-gateway/for-containers/quickstart-deploy-application-gateway-for-containers-alb-controller-addon>
- AGC multi-site hosting via Gateway API — <https://learn.microsoft.com/azure/application-gateway/for-containers/how-to-multiple-site-hosting-gateway-api>
- AGC components and connectivity / egress — <https://learn.microsoft.com/azure/application-gateway/for-containers/application-gateway-for-containers-components#connectivity>
- Azure WAF on Application Gateway for Containers — <https://learn.microsoft.com/azure/application-gateway/for-containers/web-application-firewall-overview>
- Cilium HTTP-aware policy concepts — <https://docs.cilium.io/en/stable/security/policy/language/#http>

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `(FeatureNotFound) The feature 'AzureServiceMeshPreview' could not be found.` | That feature isn't needed. Step 1 only requires `AdvancedNetworkingPreview`. |
| Step 5 in a second tab: `curl: (49) Couldn't parse CURLOPT_RESOLVE entry 'contoso.example.com:80:'` | Cloud Shell tabs are independent shells. Re-export `$APP_NAMESPACE` and re-derive `$FQDN` / `$IP` in every new tab. |
| Step 4a-bonus: `(ApplicationGatewayFirewallManagedRuleSetsHasMultiplePrimaryRuleSets)` after `rule-set add Microsoft_DefaultRuleSet`. | `az ... waf-policy create` forces OWASP, but AGC WAF only supports DRS 2.1. Use the atomic `update --set "managedRules.managedRuleSets=[{...DRS 2.1...}]"` shown in step 4a-bonus 1b. |
| Step 4a-bonus: CRD stuck in `DeploymentFailed` with `LinkedAuthorizationFailed` / `does not have permission to perform 'microsoft.network/applicationgatewaywebapplicationfirewallpolicies/join/action'`. | The ALB Controller's managed identity needs the `join` permission on the WAF policy. Grant `Network Contributor` scoped to the WAF policy resource (step 1e). After granting, delete + re-apply the CRD to clear the cached failure. |
| Step 4a-bonus 1e prints `ALB Controller identity:` empty, then `az role assignment create` fails with `usage error: --assignee STRING \| --assignee-object-id GUID`. | The ALB Controller identity name varies (`applicationloadbalancer-<aks>` in GA, `azurealb-<aks>` in preview). The query in 1e matches both. If neither matches, look at the `az identity list` table printed above and set `ALB_PRINCIPAL_ID` manually. |
| Re-running step 4a-bonus on a cluster where the WAF policy already exists fails at step 1a with `(ApplicationGatewayFirewallAttachAGCUnsupportedRuleSetVersion) RuleSet Version is not supported on Application Gateway for Containers resources`. | `az ... waf-policy create` is an upsert — on a re-run it tries to set OWASP again on the already-attached policy, which AGC rejects. Step 1a is wrapped in `if ! az ... waf-policy show ...; then create; fi` so it's a no-op on re-runs. |
| `kubectl` returns `the server has asked for the client to provide credentials`. | Cloud Shell session lost AKS credentials. Re-run `az aks get-credentials -g "$RESOURCE_GROUP" -n "$AKS_NAME" --overwrite-existing`. |
