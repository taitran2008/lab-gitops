# The GitOps Ladder — Phases 0–6

A step-by-step walkthrough taking one bare Linux VPS to a public HTTPS URL served from
Kubernetes and managed entirely from Git, with zero inbound firewall ports open.

Companion to `ArgoCD Learning Path.md`, which covers the full 13-phase ladder. This document
expands phases 0–6 into individual steps, explains *why* each one exists, and documents the
silent failures that a first-time run actually hits.

| | |
| --- | --- |
| Time | One focused weekend |
| Host needed | 2 cores, 4 GB RAM, 30 GB disk |
| Cost | A domain, ~$10/year |
| Ends with | `https://hello.example.com` |

---

## How to read this

This is a ladder, not a menu. Each phase introduces exactly one new idea and ends with a
**Gate** — a check you run yourself. Do not climb to the next phase until the gate passes.
Skipping a gate is how people end up debugging four things at once.

Two kinds of interruption appear throughout:

- **Gate** — what "done" looks like.
- **Trap** — a mistake that produces no error at the moment you make it, and surfaces hours
  later wearing a disguise. Every trap here was hit and diagnosed on a real build of this lab.

## What you need before step one

- A VPS running Ubuntu (or similar) with root or sudo, reachable over SSH.
- A domain name with nameservers pointed at Cloudflare, free plan. Adding the domain to
  Cloudflare is a prerequisite, not part of this guide.
- A GitHub account. You will create one public repository called `lab-gitops`.

**Placeholders.** Substitute throughout: `example.com` is your domain, `<you>` is your GitHub
username. Every command runs on the VPS over SSH unless stated otherwise.

## Where you are going

```
browser ──▶ Cloudflare edge ──▶ [outbound tunnel] ──▶ cloudflared pod
              DNS + TLS                                     │
                                                            ▼
                                                  traefik  (ClusterIP)
                                                            │
                                                  HTTPRoute match on Host
                                                            ▼
                                                  Service ──▶ Pod
```

Nothing in that chain dials *into* your VPS. The tunnel connector dials out and holds the
connection open, which is why the firewall can deny all inbound traffic except SSH.

## The seven rungs

| Phase | Topic | Est. |
| --- | --- | --- |
| 0 | Host preparation | 1 h |
| 1 | Helm on its own, no ArgoCD | 2 h |
| 2 | ArgoCD, one Application | 2 h |
| 3 | ArgoCD manages itself | 1 h |
| 4 | App-of-apps | 2 h |
| 5 | Traefik and the Gateway API | 2 h |
| 6 | Cloudflare Tunnel and a public URL | 1 h |

---

# Phase 0 — Host preparation

**Goal.** A working single-node Kubernetes cluster with the bundled ingress removed, plus the
tools every later phase depends on.

### Why k3d rather than plain k3s

k3d runs k3s inside Docker containers. Slightly more overhead, but you can create and destroy
whole clusters in seconds and run more than one on a host. Being able to throw a cluster away
without reinstalling the VPS is worth a lot while learning.

You will also switch off two things k3s ships with — its bundled Traefik v2 and its load
balancer, ServiceLB. Both fight with the Traefik you install through ArgoCD in Phase 5.

### Step 1 — Install Docker

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"
```

Log out and back in. Group membership is read at login, so your current shell keeps getting
`permission denied` on the Docker socket until you do. Then prove it:

```bash
docker run --rm hello-world
```

### Step 2 — Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl
```

### Step 3 — Install Helm, k3d and k9s

k9s is a terminal dashboard. Not required, but you will use it constantly.

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
curl -sL https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_amd64.tar.gz \
  | tar xz k9s && sudo install -m 0755 k9s /usr/local/bin/ && rm k9s
```

Check your Helm version now and write it down — Phase 4 needs **Helm 3.17 or newer** for a
flag that does not exist in older releases.

```bash
helm version --short
```

### Step 4 — Create the cluster

```bash
k3d cluster create hub \
  --api-port 6550 \
  --servers 1 \
  --k3s-arg "--disable=traefik@server:*" \
  --k3s-arg "--disable=servicelb@server:*"
```

The `@server:*` suffix is a k3d node filter meaning "apply this argument to every server node".
It is not decoration — without it k3d does not know which nodes the flag is for, silently
ignores it, and Traefik installs anyway.

> **Trap — bundled Traefik comes back.** If `kubectl get pods -A` shows a `traefik` pod or any
> `svclb-*` pods, the node filter was dropped. Do not delete the pods; they are managed by a
> k3s bootstrap manifest and will return. Run `k3d cluster delete hub` and create it again
> with the flags correct.

### Gate 0

- `kubectl get nodes` shows one node, status `Ready`.
- `kubectl get pods -A` shows **no** Traefik pod and no `svclb-*` pods.
- `helm version` and `k9s` both run.

---

# Phase 1 — Helm on its own, no ArgoCD

**Goal.** Understand chart, values, template and release before any GitOps machinery hides them.

This is the phase people skip and later regret. Almost every "ArgoCD is broken" report turns
out to be a Helm rendering error. If you cannot predict what a values file produces, you
cannot debug ArgoCD — you can only guess at it.

### Step 1 — Create a throwaway chart

```bash
mkdir -p ~/lab && cd ~/lab
helm create base-chart
cd base-chart
rm -rf templates/tests templates/ingress.yaml templates/NOTES.txt
```

Deleting the generated extras leaves a chart small enough to read end to end in one sitting.

### Step 2 — Render without installing

The single most useful Helm command, and the one you will run hundreds of times:

```bash
helm template demo . | less
```

Nothing touches the cluster. Helm reads the values, runs the templates, prints the YAML that
*would* be applied.

### Step 3 — Prove a value flows through

```bash
helm template demo . --set replicaCount=3 > /tmp/a.yaml
helm template demo . --set replicaCount=1 > /tmp/b.yaml
diff /tmp/a.yaml /tmp/b.yaml
```

One line differs. Change `service.port` instead and notice it lands in two places at once —
the container port and the Service port — because one value feeds both templates.

### Step 4 — Install, inspect, upgrade, roll back

```bash
kubectl create namespace lab
helm install demo . -n lab
helm get values demo -n lab       # what you supplied
helm get manifest demo -n lab     # what was actually applied

helm upgrade demo . -n lab --set replicaCount=3
helm history demo -n lab
helm rollback demo 1 -n lab
```

A *release* is Helm's record of an install in a namespace. Revisions are kept, which is what
makes rollback possible. Note this well: from Phase 3 onward ArgoCD renders charts and applies
the output directly, so these Helm release records stop being the source of truth. Git becomes
the source of truth instead.

### Step 5 — Clean up

```bash
helm uninstall demo -n lab
```

> **Worth doing before you move on.** Override a **map** value from a second `-f` file, then
> override a **list** value the same way. The results differ: maps merge key by key, lists are
> replaced whole. That asymmetry silently destroys configuration in larger setups. Just observe
> it now so you recognise it later.

### Gate 1

- You can read a values file and predict the rendered YAML *before* running `helm template` —
  and be right.

---

# Phase 2 — ArgoCD, one Application

**Goal.** See sync, self-heal and prune with your own eyes, using plain manifests so only one
new concept is in play.

### Step 1 — Pick the chart version deliberately, and record it

Read this step twice. The version you choose here has to match the version you write into Git
in Phase 3, and a mismatch is the nastiest failure in the whole ladder.

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-cd --versions | head -5
```

The top line is the newest chart. Write that number down. This guide uses `10.3.3` as its
example — **use whatever your own command printed.**

### Step 2 — Install ArgoCD by hand, this once

Pin the version explicitly rather than letting Helm pick.

```bash
helm install argocd argo/argo-cd \
  --version 10.3.3 \
  -n argocd --create-namespace \
  --set configs.params."server\.insecure"=true
```

`server.insecure` makes the ArgoCD server speak plain HTTP. Correct here because TLS is
terminated in front of it later, and it avoids certificate warnings while port-forwarding.

### Step 3 — Open the UI

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

kubectl -n argocd port-forward svc/argocd-server 8080:80
```

Browse `http://localhost:8080`, username `admin`. If the VPS is remote, tunnel from your
laptop: `ssh -L 8080:localhost:8080 user@vps`. Port-forwarding is temporary scaffolding —
Phase 6 replaces it with a real URL.

### Step 4 — Create the Git repository

On GitHub create a **public** repository named `lab-gitops`. Public keeps this phase simple; a
private repo needs credentials registered with `argocd repo add`, which is one more thing to
debug.

`manifests/nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: lab
spec:
  replicas: 1
  selector:
    matchLabels: { app: nginx }
  template:
    metadata:
      labels: { app: nginx }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
```

`manifests/nginx-service.yaml`, which Phase 5 will need:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: lab
spec:
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
```

### Step 5 — Point an Application at the repository

Save on the VPS as `application.yaml`. It does not need to be in Git yet.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/lab-gitops.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: lab
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f application.yaml
kubectl -n argocd get app nginx
```

### Step 6 — Run the three experiments that matter

**Self-heal.** Change the cluster by hand and watch Git win:

```bash
kubectl -n lab scale deploy nginx --replicas=5
watch kubectl -n lab get deploy nginx   # drops back to 1
```

**Prune.** Delete `manifests/nginx.yaml` from Git, push, and watch the Deployment vanish from
the cluster. Then put it back.

**Drift.** Edit an annotation by hand and read the diff in the UI before self-heal fires.

The distinction to keep: *selfHeal* fixes objects that exist but were modified. *prune* deletes
objects that exist in the cluster but no longer exist in Git. Different failures, different
switches.

### Gate 2

- `kubectl scale` is reverted automatically, with no action from you.
- Deleting a file from Git deletes the object from the cluster.
- You can explain `selfHeal` versus `prune` without looking it up.

---

# Phase 3 — ArgoCD manages itself

**Goal.** The bootstrap trick — ArgoCD's own installation becomes an Application tracked in
Git, so upgrading it is a commit rather than a command.

This is the one phase where the tool can break itself. Read the trap *before* you write the
file, not after.

> **Trap — the version downgrade that fakes being healthy.**
>
> The Application you are about to write pins a chart version. If that pin is **older** than
> the chart you installed in Phase 2, ArgoCD takes ownership and immediately downgrades itself.
> The new, older repo-server pod fails in its init container with a message like:
>
> ```
> /bin/cp: target '/bin/cp --update=none /usr/local/bin/argocd ...': No such file or directory
> ```
>
> Now the cruel part. Two ReplicaSets end up holding one replica each: the old healthy pod
> keeps serving traffic while the new one crash-loops forever. So
> `kubectl get deploy argocd-repo-server` proudly reports **READY 1/1** and nothing alerts. The
> only outward sign is the Application sitting at `Degraded`.
>
> **Avoid it:** put the exact version you installed in Phase 2 into `targetRevision`.
> **Recover from it:** correct the version in Git, push, and sync — the rollout unsticks itself.
>
> The general lesson outlives this lab: *Deployment readiness does not detect a failed
> rollout.* To catch one you must read the rollout condition —
> `kubectl rollout status deploy/<name> --timeout=30s` — not the ready count.

### Step 1 — Add the self-management Application to Git

`bootstrap/templates/argocd.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    chart: argo-cd
    repoURL: https://argoproj.github.io/argo-helm
    targetRevision: 10.3.3        # MUST match what you installed in Phase 2
    helm:
      releaseName: argocd         # never omit this — see below
      values: |
        global:
          revisionHistoryLimit: 1
        configs:
          params:
            server.insecure: true
            log.level: debug
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

`releaseName: argocd` is load-bearing. Helm bakes the release name into every rendered object
name. If it differs from the release you installed, ArgoCD renders a set of differently-named
objects, sees the originals as unmanaged leftovers, and `prune: true` deletes ArgoCD out from
under itself.

### Step 2 — Apply it and let ArgoCD adopt the release

```bash
kubectl apply -f bootstrap/templates/argocd.yaml
kubectl -n argocd get app argocd -w
```

It shows `OutOfSync` at first: the running install came from your Helm CLI, not from ArgoCD.
One sync and it takes ownership. Expect the UI to drop for ~30 seconds while the server pod
restarts — normal, not a failure.

### Step 3 — Prove self-management works

In Git, change `log.level` from `debug` to `info`. Commit, push, wait for the sync. ArgoCD
upgrades **itself** with no `helm upgrade` typed by you.

> **If you break it beyond repair.** Run `helm install` by hand again exactly as in Phase 2.
> The Applications live in Git and the cluster, so they are unaffected and will be re-adopted.
> This is a safe phase to fail in.

### Gate 3

- `kubectl -n argocd get app argocd` reads `Synced` and `Healthy`.
- A committed change to the values block reaches the running ArgoCD with no command from you.
- `kubectl -n argocd get rs | grep repo-server` shows exactly one ReplicaSet with a non-zero
  replica count.

---

# Phase 4 — App-of-apps

**Goal.** Install one thing by hand; everything else pulls itself in. From here on, adding a
component to the cluster means adding a file to a directory.

The idea in one sentence: a directory called `apps/` holds **Application manifests, not
workload manifests**, and a single Application watches that directory. Its whole job is to
create other Applications.

```
lab-gitops/
├── bootstrap/
│   ├── Chart.yaml            # required — see Trap A
│   └── templates/
│       ├── argocd.yaml       # from Phase 3
│       └── apps.yaml         # points at apps/
├── apps/                     # Applications live here
│   └── nginx.yaml
└── manifests/                # workloads live here
    ├── nginx.yaml
    └── nginx-service.yaml
```

> **Trap A — a chart directory with no Chart.yaml installs nothing.**
>
> Helm identifies a chart by the `Chart.yaml` at its root. A directory holding only
> `templates/` is not a chart. If you run `helm install bootstrap .` from the repository root
> and a stray `Chart.yaml` happens to be sitting there — the leftover from Phase 1's
> `helm create` is a common culprit — Helm installs *that* chart instead. It has no
> `templates/` directory, renders zero resources, reports `STATUS: deployed`, and you spend an
> hour wondering why nothing appeared.
>
> **Verify rather than trust:** `helm get manifest bootstrap -n argocd` must print your two
> Applications. Empty output means you installed the wrong directory.

> **Trap B — the directory name is a contract.**
>
> `apps.yaml` declares `path: apps`. If your directory is named `app`, ArgoCD points at a path
> that does not exist and creates nothing — with no error that mentions the word "app".
> Singular and plural are not interchangeable here.

### Step 1 — Give the bootstrap directory its own Chart.yaml

`bootstrap/Chart.yaml` — three lines is genuinely all it needs:

```yaml
apiVersion: v2
name: lab-bootstrap
version: 1.0.0
```

If a `Chart.yaml` is sitting in your repository root from Phase 1, delete it now.

### Step 2 — Add the app-of-apps Application

`bootstrap/templates/apps.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/lab-gitops.git
    targetRevision: HEAD
    path: apps
    directory:
      recurse: false
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
```

`recurse: false` means top-level files in `apps/` only, no subdirectories. That is what you
want: it lets you keep values or notes in a subfolder without ArgoCD trying to apply them.

### Step 3 — Move the nginx Application into Git

Phase 2's Application was applied by hand and is not tracked. Copy it to `apps/nginx.yaml` —
identical content — so the app-of-apps takes it over. Commit and push everything.

### Step 4 — Install the bootstrap chart, once, by hand

The last manual `helm install` in the entire lab.

```bash
cd ~/lab-gitops   # your clone on the VPS
git pull
helm install bootstrap ./bootstrap -n argocd --take-ownership
```

`--take-ownership` (Helm 3.17+) lets the release adopt the `argocd` Application you applied by
hand in Phase 3. Without it Helm refuses with `invalid ownership metadata`, because that object
exists and no release claims it.

### Step 5 — Confirm before moving on

```bash
helm get manifest bootstrap -n argocd | grep -E "^kind:|^  name:"
kubectl -n argocd get app
```

The first must list two Applications. The second must show `apps` alongside `argocd` and
`nginx`.

> **Deleting the argocd Application is not safe.** It carries
> `resources-finalizer.argocd.argoproj.io`. Deleting the Application cascades to every resource
> it manages — which is all of ArgoCD. If you ever need to remove one, strip the finalizer
> first, or use `--cascade=orphan`. This is why the step above uses `--take-ownership` instead
> of "delete it and reinstall".

### Gate 4

- Adding a file to `apps/` and pushing creates an Application, with no command run.
- Deleting that file removes the Application again.
- You can say why `recurse: false` is the right default here.

---

# Phase 5 — Traefik and the Gateway API

**Goal.** Route traffic by hostname inside the cluster, using the Gateway API that replaces the
older Ingress resource.

### Three resources, two owners

| Resource | What it is | Who owns it |
| --- | --- | --- |
| `GatewayClass` | Which controller implements gateways | Installed by Traefik |
| `Gateway` | The listener: ports, protocols, hostnames | Platform — one, in `traefik` |
| `HTTPRoute` | A routing rule: host and path to a Service | App teams — many, in app namespaces |

That split is the point of the Gateway API. The platform owns the entry point; each
application declares its own route next to its own workload. Because those live in different
namespaces, cross-namespace attachment has to be explicitly allowed — and is denied by default.

### Step 1 — Install the Gateway API CRDs first

Order matters here more than anywhere else in the lab.

```bash
curl -s https://api.github.com/repos/kubernetes-sigs/gateway-api/releases/latest | grep tag_name

kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

kubectl get crd | grep gateway
```

Substitute the tag the first command printed. You should see at least `gatewayclasses`,
`gateways` and `httproutes`.

### Step 2 — Add Traefik as an Application

`apps/traefik.yaml`. Check the current chart version with
`helm search repo traefik/traefik --versions` after adding the repo.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: traefik
  namespace: argocd
spec:
  project: default
  source:
    chart: traefik
    repoURL: https://traefik.github.io/charts
    targetRevision: "39.0.6"
    helm:
      releaseName: traefik
      values: |
        providers:
          kubernetesGateway:
            enabled: true
        gateway:
          listeners:
            web:
              namespacePolicy:
                from: All
        service:
          type: ClusterIP
        logs:
          access:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: traefik
  syncPolicy:
    automated: { selfHeal: true, prune: true }
    syncOptions: [ CreateNamespace=true ]
```

Push it. The app-of-apps creates the Application for you. Three settings deserve attention:

- `providers.kubernetesGateway.enabled` — without it Traefik ignores Gateway API resources
  entirely.
- `namespacePolicy.from: All` — permits an `HTTPRoute` in another namespace to attach to this
  Gateway. Omit it and your routes silently do nothing.
- `service.type: ClusterIP` — deliberate. Nothing is exposed to the internet yet. Phase 6
  supplies the entry point.

Access logs are switched on now because they cost nothing and become your independent witness
when debugging Phase 6.

### Step 3 — Find the Gateway that the chart created

```bash
kubectl get gatewayclass
kubectl get gateway -A
```

Note the Gateway's name — commonly `traefik-gateway` — and use it in the next step. Do not
assume; chart defaults change between versions.

### Step 4 — Route the nginx Service

`manifests/nginx-httproute.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx
  namespace: lab
spec:
  parentRefs:
    - name: traefik-gateway
      namespace: traefik
  hostnames:
    - demo.local
  rules:
    - matches:
        - path: { type: PathPrefix, value: / }
      backendRefs:
        - name: nginx
          port: 80
```

### Step 5 — Test through Traefik without any DNS

Routing is by `Host` header, so you can fake the hostname with curl.

```bash
kubectl -n traefik port-forward svc/traefik 8000:80 &
curl -H "Host: demo.local" http://localhost:8000
```

The nginx welcome page means the request went *through Traefik* and matched your route — not
straight to the pod. Also try an unknown host; a `404` confirms routing is host-scoped rather
than a catch-all.

> **Trap — CRDs installed after Traefik.** If you applied the Gateway API CRDs after the
> Traefik pod started, Traefik never noticed them and no Gateway is created. Restart it:
> `kubectl -n traefik rollout restart deploy/traefik`. A separate symptom,
> `no matches for kind "HTTPRoute"`, means the CRDs are not installed at all.

### Gate 5

- `curl` with the `Host` header returns the nginx page through Traefik.
- `kubectl -n lab describe httproute nginx` shows the parent Gateway with `Accepted: True`.
- An unrecognised `Host` header returns 404.

---

# Phase 6 — Cloudflare Tunnel and a public URL

**Goal.** A real, public, HTTPS address for a Hello World page — with no inbound ports open on
the VPS at all.

The `cloudflared` connector dials **out** to Cloudflare and holds the connection open. Nothing
dials in. Cloudflare terminates TLS at its edge and hands the request back down the tunnel, so
you get a valid certificate without running one.

### Step 1 — Build something worth serving

Create `hello/` in the repository with four files.

`hello/configmap.yaml` — the page itself, so you can tell at a glance which app answered:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-html
  namespace: lab
data:
  index.html: |
    <!doctype html>
    <html lang="en">
      <head><meta charset="utf-8"><title>Hello World</title></head>
      <body>
        <h1>Hello World</h1>
        <p>k3d &rarr; Traefik Gateway API &rarr; Cloudflare Tunnel</p>
      </body>
    </html>
```

`hello/deployment.yaml`, mounting that ConfigMap over the nginx document root:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: lab
spec:
  replicas: 1
  selector:
    matchLabels: { app: hello }
  template:
    metadata:
      labels: { app: hello }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
          readinessProbe:
            httpGet: { path: /, port: 80 }
            initialDelaySeconds: 2
      volumes:
        - name: html
          configMap:
            name: hello-html
```

`hello/service.yaml` — same shape as the nginx Service, with `hello` throughout.

`hello/httproute.yaml`, using your real hostname:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: hello
  namespace: lab
spec:
  parentRefs:
    - name: traefik-gateway
      namespace: traefik
  hostnames:
    - hello.example.com
  rules:
    - matches:
        - path: { type: PathPrefix, value: / }
      backendRefs:
        - name: hello
          port: 80
```

Finally `apps/hello.yaml` — an Application with `path: hello` and destination namespace `lab`,
copying the shape of `apps/nginx.yaml`. Push it all.

### Step 2 — Test the route before involving Cloudflare

Do not skip this. Proving the inside works first means any later failure has one possible cause
instead of four.

```bash
kubectl -n traefik port-forward svc/traefik 8000:80 &
curl -H "Host: hello.example.com" http://localhost:8000
```

You want the Hello World HTML. Only then continue.

### Step 3 — Create the tunnel in Cloudflare

Zero Trust dashboard: *Networks → Tunnels → Create a tunnel*, connector type **Cloudflared**,
name it `lab`. Cloudflare then shows an install command containing a long token.

**Copy the token only. Do not run the command it is embedded in.** Step 6's trap explains why
that command is a loaded gun in this setup.

### Step 4 — Store the token as a Secret, without leaking it

Run on the VPS. This keeps the token out of your shell history, which `--from-literal` typed
directly would not.

```bash
kubectl create namespace cloudflared --dry-run=client -o yaml | kubectl apply -f -
read -rs TOKEN && kubectl -n cloudflared create secret generic tunnel-token \
  --from-literal=tunnelToken="$TOKEN" && unset TOKEN
```

Paste at the blank prompt and press Enter. `read` is a shell builtin, so nothing reaches the
history file; `-s` stops the token echoing to the screen.

The Secret is deliberately not in Git. Everything else in this lab is Git-native; credentials
are the exception until you add an external secrets operator later.

### Step 5 — Deploy the connector into the cluster

`cloudflared/deployment.yaml`. Check the current image tag first — the newest is visible on
Docker Hub, and pinning a tag from an old tutorial gets you `ImagePullBackOff`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflared
spec:
  replicas: 2
  selector:
    matchLabels: { app: cloudflared }
  template:
    metadata:
      labels: { app: cloudflared }
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:2026.8.2
          args: [ tunnel, --no-autoupdate, --metrics, "0.0.0.0:2000", run ]
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: tunnel-token
                  key: tunnelToken
          ports:
            - name: metrics
              containerPort: 2000
          readinessProbe:
            httpGet: { path: /ready, port: metrics }
            initialDelaySeconds: 10
```

Add `apps/cloudflared.yaml` pointing at `path: cloudflared` with destination namespace
`cloudflared` and `CreateNamespace=true`. Push.

Exposing the metrics port is not optional bookkeeping — it is the instrument that makes the
next trap diagnosable in one command.

### Step 6 — Map the hostname to Traefik

In the tunnel's configuration, open **Public Hostname** and add:

| Field | Value |
| --- | --- |
| Subdomain | `hello` |
| Domain | `example.com` |
| Type | `HTTP` |
| URL | `traefik.traefik.svc.cluster.local:80` |

The DNS record is created for you. Type stays `HTTP`, not HTTPS: Cloudflare already terminated
TLS at its edge, and the tunnel-to-Traefik hop happens inside the cluster.

That URL is a Kubernetes-internal DNS name. It resolves *only* from inside the cluster. Hold on
to that fact for ninety seconds.

### Step 7 — Load the page

```bash
curl -sI https://hello.example.com/ | head -1
```

If that is `HTTP/2 200`, you are done — jump to Gate 6. If it is `502`, read on.

---

## Fixing 502 Bad Gateway

> **Trap — two connectors, one tunnel.**
>
> When Cloudflare showed you the tunnel token, it showed it inside a ready-to-paste installer —
> something like `cloudflared service install <token>`. Running that on the VPS installs a
> **systemd service on the host**, which registers as a second connector on the same tunnel.
>
> Now two connectors serve one tunnel, and Cloudflare load-balances between them. The pods can
> reach `traefik.traefik.svc.cluster.local`. The host process cannot — it lives outside
> Kubernetes and has no access to cluster DNS. Its logs say so plainly:
>
> ```
> ERR Request failed dest=https://hello.example.com/
>   error="Unable to reach the origin service ... dial tcp: lookup
>   traefik.traefik.svc.cluster.local on 127.0.0.53:53: server misbehaving"
> ```
>
> What makes this hard to spot is that *every part you built is working correctly*. The tunnel
> is healthy, DNS is right, the route is right, the pods are Ready. The requests are simply
> being answered by a connector you forgot you installed.

### The diagnosis ladder

Work bottom-up. Each rung isolates one hop, so the first rung that fails names the culprit.
Worth doing even when you already suspect the answer — it is how you find out you are wrong
quickly.

**L1 — Is the pod serving?**

```bash
kubectl -n lab get pods -l app=hello
```

**L2 — Does Traefik route the hostname?**

```bash
kubectl -n traefik port-forward svc/traefik 8000:80 &
curl -H "Host: hello.example.com" localhost:8000
```

**L3 — Can a pod in the cloudflared namespace reach Traefik?**

```bash
kubectl -n cloudflared run nettest --rm -i --restart=Never \
  --image=curlimages/curl -- -sI -H "Host: hello.example.com" \
  http://traefik.traefik.svc.cluster.local:80/
```

**L4 — Are requests reaching your connector at all?**

```bash
kubectl -n cloudflared port-forward deploy/cloudflared 2000:2000 &
curl -s localhost:2000/metrics | grep cloudflared_tunnel_total_requests
```

**L5 — What does the public internet get?**

```bash
curl -sI https://hello.example.com/ | head -1
```

### L4 is the decisive test

Read the counter, make exactly one public request, read it again:

```bash
curl -s localhost:2000/metrics | grep cloudflared_tunnel_total_requests
curl -s -o /dev/null https://hello.example.com/
curl -s localhost:2000/metrics | grep cloudflared_tunnel_total_requests
```

| Counter | Meaning | Where to look |
| --- | --- | --- |
| Increments, errors stay 0 | Your connector handled it fine | Above the tunnel — Cloudflare rules, caching, the browser |
| Increments, errors increment | Your connector cannot reach the origin | The Public Hostname URL, or the Traefik Service name and port |
| **Does not move at all** | **The request never reached your pods** | **Another connector is serving this tunnel** |

A frozen counter alongside a 502 at the browser is conclusive: something else is answering.
Find it on the host:

```bash
ps aux | grep [c]loudflared
systemctl is-active cloudflared
journalctl -u cloudflared --no-pager --since "-15min" | grep -i err
```

Confirm it is the same tunnel by comparing IDs — the host's `journalctl` output and the pods'
logs will both contain a line reading `Starting tunnel tunnelID=…`. Matching IDs prove the
duplication. Then remove the host connector and let the cluster have the traffic:

```bash
sudo systemctl stop cloudflared
sudo systemctl disable cloudflared     # or it returns on reboot
curl -sI https://hello.example.com/ | head -1
```

Give the edge a few seconds to drop the stale connections. You should now get `HTTP/2 200`, and
the pod counter should climb with every request.

> **Then clean up properly.** The installer left a live tunnel token at
> `/etc/cloudflared/token`, and a disabled unit can still be started by anything that runs
> `systemctl start`. Since the connector now lives in Kubernetes, remove the host copy for good:
> `sudo cloudflared service uninstall && sudo rm -rf /etc/cloudflared`.

### Other causes of 502, ruled out by the ladder

| Symptom | Cause |
| --- | --- |
| Tunnel HEALTHY, counter increments *with* errors | Public Hostname URL points at the wrong Service or port. Check `kubectl -n traefik get svc`. |
| 404 from Traefik rather than 502 | Reached Traefik, matched nothing. The `HTTPRoute` hostname does not match the public hostname. |
| Connector never registers | Token wrong, or outbound port 7844 blocked. The pod's startup pre-check table shows which. |
| `CreateContainerConfigError` on the pods | The `tunnel-token` Secret is missing or the key is not `tunnelToken`. |

### Gate 6 — the finish line

- `https://hello.example.com` loads from your phone on mobile data, with a valid certificate.
- Exactly one connector serves the tunnel: `systemctl is-active cloudflared` on the host
  returns `inactive`.
- `kubectl -n argocd get app` shows every Application `Synced` and `Healthy`.
- Changing the text in `hello/configmap.yaml`, pushing, and deleting the pod shows your new
  text at the public URL.

---

# After — where you landed

Your repository now looks like this, and every object in the cluster traces back to a file in
it:

```
lab-gitops/
├── bootstrap/              # installed by hand, exactly once
│   ├── Chart.yaml
│   └── templates/
│       ├── argocd.yaml     # ArgoCD manages itself
│       └── apps.yaml       # app-of-apps -> apps/
├── apps/                   # Applications, not workloads
│   ├── nginx.yaml
│   ├── hello.yaml
│   ├── traefik.yaml
│   └── cloudflared.yaml
├── manifests/              # phase 2 nginx
├── hello/                  # the public app
└── cloudflared/            # the tunnel connector
```

## Close the firewall

The tunnel gives you the zero-inbound property, but it does not enforce it. Enforcing it is
what makes the claim true.

Order matters and a mistake locks you out of a remote machine, so allow SSH **first**, confirm
it is listed, and only then enable:

```bash
sudo ufw allow OpenSSH
sudo ufw status numbered    # confirm OpenSSH is there BEFORE the next line
sudo ufw enable
```

Then verify the claim honestly: `sudo ufw status` should show SSH alone, while the public URL
keeps working.

## The gap worth remembering

One mapping — public hostname to origin service — lives in the Cloudflare dashboard, not in
Git. Everything else here is Git-native. When something routes unexpectedly and `git log` shows
nothing, that is the first place to look.

## Next rungs

The ladder continues past this point: log collection with Loki, Alloy and Grafana on a second
public hostname reusing exactly the pattern you just built; then a shared base chart with an
umbrella chart over it; values layering across environments; External Secrets so the tunnel
token itself leaves the cluster; and CI pushing image digests back into Git so nobody types a
deploy command.

> **Tearing it all down.** `k3d cluster delete hub` removes the cluster and everything in it.
> Delete the tunnel in the Cloudflare dashboard to remove the DNS record. Nothing else on the
> VPS is modified.

---

*Every version number here is a starting point, not a recommendation — check the current
release before pinning, and pin explicitly regardless. `targetRevision: HEAD` on a third-party
chart repository means a silent upgrade the next time upstream publishes.*
