# GitOps Case Study
This repository contains a minimal and reproducible local Kubernetes GitOps demo using kind, Flux, Vector, Loki and Grafana.

The setup is intentionally small and is not intended for production use. Its purpose is to demonstrate GitOps principles.

The setup was created and tested on an ARM-based Mac using Podman. Other operating systems, container runtimes or CPU architectures may require small adjustments.

| Component | App Version | Helm Chart |
| --- | --- | --- |
| Kubernetes | v1.35.1 | |
| Flux | v2.8.6 | |
| Vector | 0.55.0 | 0.52.0 |
| Loki | 3.7.1 | 13.5.0 |
| Grafana | 13.0.1 | 12.3.0 |

## Bootstrap Kubernetes and Flux

### Prerequisites
Make sure recent versions of the following tools are installed.

- git (https://git-scm.com/install/)
- podman (https://podman.io/docs/installation)
- kind (https://kind.sigs.k8s.io/docs/user/quick-start/#installing-with-a-package-manager)
- kubectl (https://kubernetes.io/docs/tasks/tools/#kubectl)


### Kubernetes cluster

1. Bootstrap the Kubernetes cluster with kind
    ```
    kind create cluster --image kindest/node:v1.35.1
    ```
1. Set kubectl cluster context
    ```
    kubectl cluster-info --context kind-kind
    ```

### Flux preparation

1. Install Flux CLI (https://fluxcd.io/flux/installation/#install-the-flux-cli)
1. Test prerequisites
    ```
    flux check --pre
    ```

### Option A: Flux initial bootstrap
This section describes how the repository and Flux was initially bootstrapped.

The setup uses a monorepo structure for simplicity (https://fluxcd.io/flux/guides/repository-structure/#monorepo)


1. Create an empty GitHub repository
1. Create a fine-grained GitHub PAT (https://fluxcd.io/flux/installation/bootstrap/github/#github-pat)
1. Export GitHub token
    ```bash
    export GITHUB_TOKEN=<gh-token>
    ```
1. Bootstrap Flux
    ```bash
    flux bootstrap github \
        --version=v2.8.6 \
        --token-auth \
        --owner=pascalstindt \
        --repository=gitops-case-study \
        --branch=main \
        --path=clusters/local \
        --personal
    ```
1. Quick health check
    ```bash
    flux check
    ```

### Option B: Reproduce this setup

To reproduce this setup, the repository can be forked and bootstrap ran against it.

A GitHub PAT is required because ```flux bootstrap github``` writes the Flux manifests to the repository and configures Flux to authenticate against GitHub.

1. Fork this repository
1. Create a fine-grained GitHub PAT for the fork
1. Export the token
    ```bash
    export GITHUB_TOKEN=<gh-token>
    ```
1. Bootstrap Flux with your GitHub owner and repository name

    Replace ```<your-github-user-or-org>``` and ```<your-forked-repository>``` with the values of your fork.

    ```bash
    flux bootstrap github \
        --version=v2.8.6 \
        --token-auth \
        --owner=<your-github-user-or-org> \
        --repository=<your-forked-repository> \
        --branch=main \
        --path=clusters/local \
        --personal
    ```
1. Verify Flux
    ```bash
    flux check
    ```


## Deploy components

After Flux is installed, the cluster monitors the repository path ```clusters/local```. The infrastructure components are deployed by adding Flux ```Kustomization``` resources at this path.

Each component is defined in ```infrastructure/observability/<component>``` and referenced from ```clusters/local```.

File and folder structure:

```text
.
├── clusters
│   └── local
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── infra-grafana.yaml
│       ├── infra-loki.yaml
│       └── infra-vector.yaml
└── infrastructure
    └── observability
        ├── grafana
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   ├── release.yaml
        │   └── repository.yaml
        ├── loki
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   ├── release.yaml
        │   └── repository.yaml
        └── vector
            ├── kustomization.yaml
            ├── namespace.yaml
            ├── release.yaml
            └── repository.yaml
```

The component definitions are already committed in this repository. After Flux bootstrap, Flux reconciles the resources from ```clusters/local``` automatically.

If changes are made locally, commit and push them to the Git repository. Flux will pick up the changes automatically.
```bash
git pull
git add .
git commit -m "Add observability components"
git push
```

### Limitations

- No persistent volumes
- No TLS certificates for Grafana
- No authentication or reverse proxy for Loki
- No Prometheus
- No labels on components
- Chart versions are pinned by version but are not pinned by digest
- Image versions are not explicitly pinned by version or digest


## Verification

### Flux

Make sure all Flux resources are in Ready state.

```bash
flux get kustomizations
flux get sources oci -A
flux get helmreleases -A
```
Output
```text
NAME         	REVISION          	SUSPENDED	READY	MESSAGE
flux-system  	main@sha1:8cbe0296	False    	True 	Applied revision: main@sha1:8cbe0296	
infra-grafana	main@sha1:8cbe0296	False    	True 	Applied revision: main@sha1:8cbe0296	
infra-loki   	main@sha1:8cbe0296	False    	True 	Applied revision: main@sha1:8cbe0296	
infra-vector 	main@sha1:8cbe0296	False    	True 	Applied revision: main@sha1:8cbe0296	

NAMESPACE	NAME   	REVISION              	SUSPENDED	READY	MESSAGE
grafana  	grafana	12.3.0@sha256:5b1660f4	False    	True 	stored artifact for digest '12.3.0@sha256:5b1660f4'	
loki     	loki   	13.5.0@sha256:946c4e71	False    	True 	stored artifact for digest '13.5.0@sha256:946c4e71'	
vector   	vector 	0.52.0@sha256:b43d5f99	False    	True 	stored artifact for digest '0.52.0@sha256:b43d5f99'	

NAMESPACE	NAME   	REVISION           	SUSPENDED	READY	MESSAGE
grafana  	grafana	12.3.0+5b1660f4732b	False    	True 	Helm install succeeded for release grafana/grafana.v1 with chart grafana@12.3.0+5b1660f4732b	
loki     	loki   	13.5.0+946c4e711815	False    	True 	Helm install succeeded for release loki/loki.v1 with chart loki@13.5.0+946c4e711815         	
vector   	vector 	0.52.0+b43d5f998bf9	False    	True 	Helm install succeeded for release vector/vector.v1 with chart vector@0.52.0+b43d5f998bf9
```

### Vector

Check for Kubernetes logs in console sink.
```bash
kubectl -n vector logs ds/vector -f
```
Vector should print Kubernetes log events to the screen through its console sink.

### Loki

Check that Loki has data from Vector.

```bash
kubectl -n loki port-forward svc/loki 3100:3100
curl -s http://localhost:3100/loki/api/v1/label/source/values -H 'X-Scope-OrgID: local'
```
Output
```json
{"status":"success","data":["vector"]}
```

### Grafana
Port forward Grafana UI and log in with user admin and password from secret.
```bash
kubectl -n grafana get secret grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl -n grafana port-forward svc/grafana 3000:80
```
Grafana should be pre-configured with a Loki data source. You can use the explorer to see the log entries from Loki.
