<!--- Deploy -->

# Partition Service - Helm Deployment

> ⚠️ **This chart deploys TWO components:**
> - **Partition Service** (main Java API) - always running
> - **Bootstrap** (one-time initialization on startup) - runs once then idles
> 
> See [main README](../README.md) for component details.

---

## Introduction

This chart bootstraps a deployment on a [Kubernetes](https://kubernetes.io) cluster using [Helm](https://helm.sh) package manager.

## Prerequisites

- **Kubernetes** v1.21.11+ with **Istio** v1.12.6+ (or Anthos Service Mesh)
- **Helm** v3.7.1+ ([install](https://helm.sh/docs/intro/install/))
- **kubectl** v1.21.0+ ([install](https://kubernetes.io/docs/tasks/tools/#kubectl))
- Namespace not labeled with Istio (Development mode)

> **OS Support:** Debian-based Linux (Debian 10, Ubuntu 20.04), Windows WSL 2

## Installation

First you need to set variables in **values.yaml** file using any code editor. Some of the values are prefilled, but you need to specify some values as well. You can find more information about them below.

### What Gets Deployed

#### Partition Service (Non-Bootstrap)
- Deployment `partition` with Java application
- ConfigMap `partition-config` (LOG_LEVEL)
- Service, VirtualService, AuthorizationPolicy
- Uses: `data.image`, `data.logLevel`, `conf.*`

#### Bootstrap
- Deployment `partition-bootstrap` with bash scripts
- ConfigMap `partition-config-bootstrap` (extensive config)
- Uses: `data.bootstrapImage`, `data.bucket*`, `data.*Enabled` flags

---

### Global variables (Both Components)

| Name | Description | Type | Default |Required |
|------|-------------|------|---------|---------|
**global.domain** | your domain (used for the Keycloak issuer). The route host comes from `global.hostname` | string | - | yes
**global.hostname** | host the `HTTPRoute` attaches to when `httpRoute.hostnames` is empty, ex `osdu.example.com` | string | `""` | only if `httpRoute.enabled` is true and `httpRoute.hostnames` is empty
**global.useHttps** | defines whether to use HTTPS instead of HTTP for external minio s3 endpoint connection | boolean | true | yes
**global.limitsEnabled** | whether CPU and memory limits are enabled | boolean | true | yes
**global.dataPartitionId** | data partition id | string | - | yes

---

## Common Configuration Variables

These variables are shared by both partition service and bootstrap components.

### Application Configuration

| Name | Description | Type | Default |Required |
|------|-------------|------|---------|---------|
**auth.localUrl** | authentication local URL | string | keycloak | yes
**auth.realm** | realm in keycloak | string | osdu | yes
**conf.appName** | name of the app | string | partition | yes
**conf.configmap** | configmap to be used | string | partition-config | yes
**conf.replicas** | number of pod replicas | integer | 1 | yes
**conf.secret** | secret for postgres | string | partition-postgres-secret | yes

### Istio Configuration

| Name | Description | Type | Default |Required |
|------|-------------|------|---------|---------|
**istio.bootstrapProxyCPU** | CPU request for bootstrap Envoy sidecars | string | 5m | yes
**istio.bootstrapProxyCPULimit** | CPU limit for bootstrap Envoy sidecars | string | 100m | yes
**istio.proxyCPU** | CPU request for partition service Envoy sidecars | string | 5m | yes
**istio.proxyCPULimit** | CPU limit for partition service Envoy sidecars | string | 500m | yes
**istio.proxyMemory** | memory request for partition service Envoy sidecars | string | 64Mi | yes
**istio.proxyMemoryLimit** | memory limit for partition service Envoy sidecars | string | 512Mi | yes
**istio.sidecarInject** | whether Istio sidecar will be injected. Setting to "false" reduces security, because disables authorization policy. | boolean | true | yes

---

## Bootstrap Configuration

> **⚠️ DEPRECATION NOTICE:** MinIO configuration (`data.minio*`, `data.bucketPrefix`) will be deprecated in the next release and replaced with SeaweedFS. Please plan your migration accordingly.

### Bootstrap Variables

| Name | Description | Type | Default |Required |
|------|-------------|------|---------|---------|
**data.asIngestedCoordinatesEnabled** | enable as-ingested coordinates feature | string | true | no
**data.autocompleteEnabled** | enable autocomplete search feature | string | false | no
**data.bagOfWordsEnabled** | enable bag-of-words search feature | string | true | no
**data.bootstrapImage** | path to the bootstrap image in a registry | string | - | yes
**data.bucketPrefix** | minio bucket name prefix | string | refi | only in case of Reference installation when _onPremEnabled_ is set to "_true_"
**data.collaborationsEnabled** | enable collaborations feature | string | true | no
**data.datafierSa** | datafier service account | string | datafier | yes
**data.edsEnabled** | enable EDS (External Data Sources) feature | string | false | no
**data.elasticHttps** | use https(true) or http(false) in interservice communication (search/indexer <-> elasticsearch)| bool | "" | yes
**data.indexerAugmenterEnabled** | enable indexer Augmenter | string | true | no
**data.keywordLowerEnabled** | enable lower-case keyword search | string | true | no
**data.minioExternalEndpoint** | api url for external minio, if external minio is configured - this value will be set for MINIO_ENDPOINT and MINIO_EXTERNAL_ENDPOINT in bootstrap configmap | string | - | no
**data.minioIgnoreCertCheck** | whether minio should ignore TLS certs validity check, set to true if external minio is protected by self-signed certificates | string | false | no
**data.minioUIEndpoint** | UI endpoint for gathering minio versions | string | `http://minio:9001` | yes
**data.partitionSuffix** | suffix for partition secret values | string | _SYSTEM | yes
**data.policyServiceEnabled** | enable policy service integration | string | false | no
**data.secretAdminNamespace** | namespace suffix for the secret admin service, combined with release namespace as `<namespace>-<value>` | string | secret-admin | yes

---

## Partition Service Variables (Non-Bootstrap)

### Service ConfigMap Variables

| Name | Description | Type | Default |Required |
|------|-------------|------|---------|---------|
**data.logLevel** | logging level for partition service | string | ERROR | yes

### Service Deployment Variables

| Name | Description | Type | Default |Required |
|------|-------------|------|---------|---------|
**data.requestsCpu** | amount of requests CPU for partition service | string | 5m | yes
**data.requestsMemory** | amount of requests memory for partition service | string | 350Mi | yes
**data.limitsCpu** | CPU limit for partition service | string | 500m | only if `global.limitsEnabled` is true
**data.limitsMemory** | memory limit for partition service | string | 1G | only if `global.limitsEnabled` is true
**data.serviceAccountName** | name of your service account | string | partition | yes
**data.imagePullPolicy** | when to pull the image | string | IfNotPresent | yes
**data.image** | path to the partition service image in a registry | string | - | yes

---

### Routing (HTTPRoute)

The chart owns the routing for the service. It renders a Gateway API `HTTPRoute`, disabled by default. The chart no longer ships an Istio `VirtualService`.

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
**httpRoute.enabled** | render the `HTTPRoute` | boolean | `false` | yes
**httpRoute.name** | route name. When empty the name is `<release>-route`, which differs from any route of the same name defined outside this chart | string | `""` | no
**httpRoute.parentRefs** | gateways the route attaches to | list | `cimpl-gateway` in `istio-system` | yes
**httpRoute.hostnames** | hosts the route serves. When empty, `global.hostname` is used | list | `[]` | one of `httpRoute.hostnames` or `global.hostname`
**httpRoute.routes** | one rule per entry, see below | list | `/api/partition/` | yes

The render fails when `httpRoute.enabled` is true and both `httpRoute.hostnames` and `global.hostname` are empty, so the route is never exposed on all hosts.

Each entry of `httpRoute.routes` accepts:

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
**path** | path to match | string | - | yes
**pathType** | match type, `PathPrefix` or `Exact` | string | `PathPrefix` | no
**serviceName** | backend Service name | string | `conf.appName` | no
**port** | backend Service port | integer | `80` | no
**filters** | Gateway API filters, for example `URLRewrite` | list | - | no

Example, a route with a rewrite and another port:

```yaml
httpRoute:
  enabled: true
  hostnames:
    - osdu.example.com
  routes:
    - path: /api/partition/
    - path: /other/
      port: 9002
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /
```

### Install the helm chart

Run this command from within this directory:

```console
helm install core-partition-deploy .
```

To expose the service through the Gateway API:

```console
helm install core-partition-deploy . --set httpRoute.enabled=true --set global.hostname=osdu.example.com
```

To check the rendered route without installing:

```console
helm template core-partition-deploy . --set httpRoute.enabled=true --set global.hostname=osdu.example.com -s templates/httproute.yaml
```

## Uninstalling the Chart

To uninstall the helm deployment:

```console
helm uninstall core-partition-deploy
```

To delete secrets and PVCs:

```console
kubectl delete secret --all; kubectl delete pvc --all
```

[Move-to-Top](#deploy-helm-chart)
