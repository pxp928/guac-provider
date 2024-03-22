# guac Data Provider

A repository for using guac as a data provider for Gatekeeper.

## Prerequisites

- [`docker`](https://docs.docker.com/get-docker/)
- [`kind`](https://kind.sigs.k8s.io/)
- [`helm`](https://helm.sh/)
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl)

## Quick Start

1. Create a [kind cluster](https://kind.sigs.k8s.io/docs/user/quick-start/).

1.  Setup GUAC

```bash
git clone git@github.com:pxp928/kusari-helm-charts.git

cd kusari-helm-charts

git checkout update-guacrest
```

```bash 
helm dependency update ./charts/guac
```
```bash
helm install guac ./charts/guac
```

Might have to wait for the graphQL server to be running.

```bash
kubectl port-forward svc/graphql-server 8080:8080
```

```bash
git clone git@github.com:pxp928/artifact-ff.git

cd artifact-ff

# This is an important step otherwise the hash will not be captured, which is needed for OPA
git checkout issue-1734

go run ./cmd/guacone collect files cdx_vuln.json
go run ./cmd/guacone collect files guac-slsa.json

go run ./cmd/guacone collect files ../guac-data/docs
```

```bash
go run ./cmd/guacone certifier osv
```

```bash
# Make sure to run after the OSV has completed!
go run ./cmd/guacone collect files cdx_guac.json
```

```bash
go run ./cmd/guacone certify package "critical vulnerability reported by maintainer" "pkg:alpine/alpine-baselayout@3.2.0-r18?arch=x86_64&upstream=alpine-baselayout&distro=alpine-3.15.6"
```

1. Install the latest version of Gatekeeper and enable the external data feature.

```bash
# Install the latest version of Gatekeeper with the external data feature enabled.
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper/gatekeeper  \
    --name-template=gatekeeper \
    --namespace gatekeeper-system --create-namespace \
    --set enableExternalData=true \
    --set controllerManager.dnsPolicy=ClusterFirst,audit.dnsPolicy=ClusterFirst
```

1. Build and deploy the guac data provider.

```bash
git clone https://github.com:dejanb/guac-provider.git
cd guac-provider
```

```bash
# generate a self-signed certificate for the guac data provider
./scripts/generate-tls-certificate.sh
```

```bash
# build the image via docker 
docker build . -t ghcr.io/dejanb/guac-provider:latest
```

```bash
# load the image into kind
kind load docker-image ghcr.io/dejanb/guac-provider:latest --name kind
```

```bash
# Install guac data provider into gatekeeper-system to use mTLS
helm install guac-provider charts/guac-provider \
    --set provider.tls.caBundle="$(cat certs/ca.crt | base64 | tr -d '\n\r')" \
    --namespace gatekeeper-system
```

1. Install constraint template and constraint.

```bash
kubectl apply -f policy/template.yaml
kubectl apply -f policy/constraint.yaml
```

1. Check the logs for the guac-provider
```bash
# Do this from another terminal. Not the one that you used with guac-provider as you will need it for the later steps
kubectl logs -n gatekeeper-system deployments/guac-provider -f
```

1. Examples
```bash
kubectl create ns test
```
```bash
kubectl apply -f policy/examples/vulnerable.yaml -n test
```
```bash
kubectl apply -f policy/examples/bad.yaml -n test
```
```bash
kubectl apply -f policy/examples/sbom.yaml -n test
```
```bash
kubectl apply -f policy/examples/slsa.yaml -n test
```
```bash
kubectl apply -f policy/examples/good.yaml -n test
```

The output will look similar to this (as you run through the various examples above):
```bash
2024-05-03T19:21:21.933Z	INFO	guac-provider/provider.go:104	Image received: ghcr.io/guacsec/vul-image:latest@sha256:b6f1a6e034d40c240f1d8b0a3f5481aa0a315009f5ac72f736502939419c1855
2024-05-03T19:21:21.933Z	INFO	guac-provider/provider.go:113	Image Digest: sha256:b6f1a6e034d40c240f1d8b0a3f5481aa0a315009f5ac72f736502939419c1855
2024-05-03T19:21:22.207Z	INFO	guac-provider/provider.go:132	found Vulnerabilities: dsa-5343-1,dsa-5650-1,ghsa-8489-44mv-ggj8,ghsa-fxph-q3j8-mv87,ghsa-p6xc-xr62-6r2g,dsa-5417-1,ghsa-7rjr-3q55-vv33,ghsa-jfh8-c2jp-5v3q,ghsa-vwqq-5vrc-xw9h,ghsa-599f-7c49-w659
2024-05-03T19:21:28.708Z	INFO	guac-provider/provider.go:104	Image received: bash@sha256:020031cbba4cccf13061c0c089b52eb8ff067a15033f2b9f839ea503d60ec037
2024-05-03T19:21:28.709Z	INFO	guac-provider/provider.go:113	Image Digest: sha256:020031cbba4cccf13061c0c089b52eb8ff067a15033f2b9f839ea503d60ec037
2024-05-03T19:21:28.733Z	INFO	guac-provider/provider.go:138	found CertifyBad: critical vulnerability reported by maintainer
2024-05-03T19:21:56.241Z	INFO	guac-provider/provider.go:104	Image received: ghcr.io/guacsec/guac@sha256:0ea6c5ec80900ad1b96c604f9311b2335292a35d6ff1b9d955354d66ce0216c5
2024-05-03T19:21:56.241Z	INFO	guac-provider/provider.go:113	Image Digest: sha256:0ea6c5ec80900ad1b96c604f9311b2335292a35d6ff1b9d955354d66ce0216c5
2024-05-03T19:21:56.250Z	INFO	guac-provider/provider.go:144	SBOM not found for image: ghcr.io/guacsec/guac@sha256:0ea6c5ec80900ad1b96c604f9311b2335292a35d6ff1b9d955354d66ce0216c5
2024-05-03T19:22:01.771Z	INFO	guac-provider/provider.go:104	Image received: alpine@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
2024-05-03T19:22:01.771Z	INFO	guac-provider/provider.go:113	Image Digest: sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
2024-05-03T19:22:01.791Z	INFO	guac-provider/provider.go:150	SLSA not found for image: alpine@sha256:1304f174557314a7ed9eddb4eab12fed12cb0cd9809e4c28f29af86979a3c870
2024-05-03T19:22:07.538Z	INFO	guac-provider/provider.go:104	Image received: ghcr.io/guacsec/guac@sha256:af080e45d452929203c3a57219d2af6293eacf9bf905624ff555034cb0e7027c
2024-05-03T19:22:07.538Z	INFO	guac-provider/provider.go:113	Image Digest: sha256:af080e45d452929203c3a57219d2af6293eacf9bf905624ff555034cb0e7027c
2024-05-03T19:22:08.289Z	INFO	guac-provider/provider.go:156	guac verified image: ghcr.io/guacsec/guac@sha256:af080e45d452929203c3a57219d2af6293eacf9bf905624ff555034cb0e7027c
```



1. Delete

```bash
kubectl delete -f policy/

helm uninstall guac
helm uninstall guac-provider --namespace gatekeeper-system
helm uninstall gatekeeper --namespace gatekeeper-system
```