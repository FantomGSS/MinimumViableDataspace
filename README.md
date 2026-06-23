# Minimum Viable Dataspace – Local Setup Guide

## 1. Prerequisites

Install the following tools:

* Git
* Docker Desktop with WSL2 enabled
* Java 17
* kubectl
* kind
* Helm

Verify the installation:

```bash
docker --version
docker compose version
java -version
kubectl version --client
kind version
helm version
```

## 2. Clone the Repository

```bash
git clone https://github.com/FantomGSS/MinimumViableDataspace.git
cd MinimumViableDataspace
git checkout university-project
```

## 2.5. (Optional) Build and Load Local Docker Images
If you are developing locally and the Docker images are not available in the public registry, you must build them from source and load them into the Kind cluster.

**1. Clean and build the Java project:**
Stop any background Gradle processes and compile the project (skip tests to save time):
```powershell
.\gradlew --stop
.\gradlew clean build -x test


$components = @(
    @{ Name = "controlplane";  Path = "launchers\controlplane\src\main\docker\Dockerfile" },
    @{ Name = "dataplane";     Path = "launchers\dataplane\src\main\docker\Dockerfile" },
    @{ Name = "identityhub";   Path = "launchers\identity-hub\src\main\docker\Dockerfile" },
    @{ Name = "issuerservice"; Path = "launchers\issuerservice\src\main\docker\Dockerfile" }
)

foreach ($component in $components) {
    $tag = "ghcr.io/eclipse-dataspace-hub/minimumviabledataspace/$($component.Name):latest"
    docker build -t $tag -f $component.Path .
    kind load docker-image $tag --name mvd
}

## 3. Create a Kind Cluster

```bash
kind create cluster --name mvd
kubectl get nodes
```

Expected result:

```text
mvd-control-plane   Ready
```

## 4. Install Gateway API CRDs

Install Gateway API v1.5.1 CRDs:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/experimental-install.yaml
```

Verify:

```bash
kubectl get crd | findstr gateway
kubectl get crd | findstr httproutes
kubectl get crd | findstr tlsroutes
```

## 5. Install Traefik Gateway Controller

```bash
kubectl create namespace traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik -n traefik
```

Enable Gateway API support:

```bash
kubectl delete gatewayclass traefik
helm upgrade traefik traefik/traefik -n traefik --set providers.kubernetesGateway.enabled=true --set ports.web.exposedPort=80 --set ports.web.expose.default=true
kubectl rollout restart deployment/traefik -n traefik
```

Verify:

```bash
kubectl get pods -n traefik
kubectl get gatewayclass
```

Expected result:

```text
traefik   traefik.io/gateway-controller   True
```

## 6. Deploy the MVD Components

From the project root:

```bash
kubectl apply -k k8s
```

Verify that all pods are running:

```bash
kubectl get pods -A
```

Expected components:

* consumer Control Plane
* consumer Data Plane
* consumer Identity Hub
* provider Control Plane
* provider Data Plane
* provider Identity Hub
* issuer service
* Keycloak
* Vault
* PostgreSQL
* Traefik

## 7. Verify Gateway Configuration

```bash
kubectl get gateways -A
```

Expected result:

```text
consumer-gateway   traefik   True
provider-gateway   traefik   True
issuer-gateway     traefik   True
common-gateway     traefik   True
```

If the Gateways are not programmed, check:

```bash
kubectl describe gateway consumer-gateway -n consumer
kubectl logs -n traefik deployment/traefik
```

In this project, the MVD Gateway listener ports were updated from `80` to `8000` in the following files to match the Traefik Gateway entryPoint:

```text
k8s/common/gateway.yaml
k8s/consumer/base/gateway.yaml
k8s/issuer/base/gateway.yaml
k8s/provider/base/gateway.yaml
```

## 8. Port-forward Traefik

Open a separate terminal and run:

```bash
kubectl port-forward svc/traefik 8080:80 -n traefik
```

Keep this terminal open.

## 9. Validate Provider Assets

In a second terminal:

```bash
curl -X POST http://cp.provider.localhost:8080/api/mgmt/v4/assets/request ^
  -H "Content-Type: application/json" ^
  -d "{\"@type\":\"QuerySpec\"}"
```

Expected result: the provider returns `asset-1` and `asset-2`.

## 10. Request Catalog

```bash
curl -X POST http://cp.consumer.localhost:8080/api/mgmt/v4/catalog/request ^
  -H "Content-Type: application/json" ^
  -d "{\"@context\":[\"https://w3id.org/edc/connector/management/v2\"],\"@type\":\"CatalogRequest\",\"counterPartyAddress\":\"http://controlplane.provider.svc.cluster.local:8082/api/dsp/2025-1\",\"counterPartyId\":\"did:web:identityhub.provider.svc.cluster.local%3A7083:provider\",\"protocol\":\"dataspace-protocol-http:2025-1\",\"querySpec\":{\"offset\":0,\"limit\":50}}"
```

Expected result: catalog response containing `asset-1` and `asset-2`.

## 11. Initiate Contract Negotiation

Use the policy ID returned for `asset-1` from the catalog response.

```bash
curl -X POST http://cp.consumer.localhost:8080/api/mgmt/v4/contractnegotiations ^
  -H "Content-Type: application/json" ^
  -d "{\"@context\":[\"https://w3id.org/edc/connector/management/v2\"],\"@type\":\"ContractRequest\",\"counterPartyAddress\":\"http://controlplane.provider.svc.cluster.local:8082/api/dsp/2025-1\",\"counterPartyId\":\"did:web:identityhub.provider.svc.cluster.local%3A7083:provider\",\"protocol\":\"dataspace-protocol-http:2025-1\",\"policy\":{\"@type\":\"Offer\",\"@id\":\"bWVtYmVyLWFuZC1tYW51ZmFjdHVyZXItZGVm:YXNzZXQtMQ==:ODhiODRkZjAtNDgxMS00MzdiLTk2ZDUtNTAwNTBmOTUwMWYw\",\"assigner\":\"did:web:identityhub.provider.svc.cluster.local%3A7083:provider\",\"permission\":[],\"prohibition\":[],\"obligation\":{\"action\":\"use\",\"constraint\":{\"leftOperand\":\"ManufacturerCredential.part_types\",\"operator\":\"eq\",\"rightOperand\":\"non_critical\"}},\"target\":\"asset-1\"},\"callbackAddresses\":[]}"
```

## 12. Check Contract Negotiation Status

```bash
curl -X POST http://cp.consumer.localhost:8080/api/mgmt/v4/contractnegotiations/request ^
  -H "Content-Type: application/json" ^
  -d "{\"@type\":\"QuerySpec\"}"
```

Expected result:

```text
state = FINALIZED
```

Save the returned `contractAgreementId`.

## 13. Initiate Transfer

Replace `contractId` with the returned `contractAgreementId`.

```bash
curl -X POST http://cp.consumer.localhost:8080/api/mgmt/v4/transferprocesses ^
  -H "Content-Type: application/json" ^
  -d "{\"@context\":[\"https://w3id.org/edc/connector/management/v2\"],\"assetId\":\"asset-1\",\"@type\":\"TransferRequest\",\"counterPartyAddress\":\"http://controlplane.provider.svc.cluster.local:8082/api/dsp/2025-1\",\"connectorId\":\"did:web:identityhub.provider.svc.cluster.local%3A7083:provider\",\"contractId\":\"58250037-589b-420d-a3b5-e93a35640e60\",\"dataDestination\":{\"@type\":\"DataAddress\",\"type\":\"HttpProxy\"},\"protocol\":\"dataspace-protocol-http:2025-1\",\"transferType\":\"HttpData-PULL\"}"
```

## 14. Check Transfer Status

```bash
curl -X POST http://cp.consumer.localhost:8080/api/mgmt/v4/transferprocesses/request ^
  -H "Content-Type: application/json" ^
  -d "{\"@type\":\"QuerySpec\"}"
```

Expected result:

```text
state = STARTED
```

## 15. Useful Troubleshooting Commands

```bash
kubectl get pods -A
kubectl get gateways -A
kubectl get httproutes -A
kubectl get gatewayclass
kubectl describe gateway consumer-gateway -n consumer
kubectl logs -n traefik deployment/traefik
```

## 16. Environment Used

* MVD release: `0.17.0-dps`
* Kubernetes: `v1.36.1`
* Gateway API: `v1.5.1`
* Traefik: `v3.7.5`
* Java: `OpenJDK 17.0.19`
* Docker: `29.5.3`





# Adding a Second Provider (provider2)

The dataspace can be extended by adding additional participants. In this example, a second provider (`provider2`) was created by cloning the existing provider configuration.

## 1. Clone the Provider Configuration

Copy the existing provider directory:

```bash
cp -r k8s/provider k8s/provider2
```

## 2. Update the Root Kustomization

Add the new participant to `k8s/kustomization.yml`:

```yaml
resources:
  - common
  - ./consumer
  - ./provider
  - ./provider2
  - ./issuer
```

## 3. Rename the Namespace

Update all namespace references:

```yaml
namespace: provider
```

to

```yaml
namespace: provider2
```

including:

- namespace.yaml
- gateway.yaml
- postgres.yaml
- vault.yaml
- controlplane.yaml
- dataplane.yaml
- identityhub.yaml
- all seed jobs

## 4. Update Internal Hostnames

Replace all references from:

```text
provider.svc.cluster.local
```

to

```text
provider2.svc.cluster.local
```

Examples:

```yaml
edc.hostname: identityhub.provider2.svc.cluster.local

edc.hostname: controlplane.provider2.svc.cluster.local

jdbc:postgresql://postgres.provider2.svc.cluster.local:5432/controlplane

http://vault.provider2.svc.cluster.local:8200
```

## 5. Create a New Participant Identity

Update the participant DID from:

```text
did:web:identityhub.provider.svc.cluster.local%3A7083:provider
```

to:

```text
did:web:identityhub.provider2.svc.cluster.local%3A7083:provider2
```

### Update All Participant-Specific Identifiers

When cloning an existing participant, all participant-specific identifiers must be updated to ensure that the new participant is treated as an independent entity within the dataspace.

Examples:

```text
provider-participant
→
provider2-participant

provider-dsp
→
provider2-dsp

provider-credentialservice-1
→
provider2-credentialservice-1

provider-participant-sts-client-secret
→
provider2-participant-sts-client-secret
```

In general, any identifier that represents the participant's identity, credentials, services, or secrets should be renamed accordingly.

This includes, but is not limited to:

- Participant DIDs
- Participant IDs
- STS client identifiers
- Secret aliases
- Credential service identifiers
- DSP identifiers
- IdentityHub participant references
- Seed configuration entries

> **Note:** Infrastructure-related names (e.g. service names, database names, or shared platform components) should only be changed when they are intended to be participant-specific.

Failing to update participant-specific identifiers may cause multiple participants to share the same logical identity, credentials, or secret references, resulting in authentication and authorization issues within the dataspace.

## 6. Update External Routes

Update HTTPRoute hostnames:

```text
vault.provider.localhost
```

→

```text
vault.provider2.localhost
```

and similarly for:

```text
cp.provider.localhost
ih.provider.localhost
dp.provider.localhost
```

## 7. Deploy

Apply the updated manifests:

```bash
kubectl apply -k k8s
```

## 8. Verify Deployment

Verify all Provider2 components are running:

```bash
kubectl get pods -n provider2
```

Expected components:

- ControlPlane
- DataPlane
- IdentityHub
- PostgreSQL
- Vault

and corresponding seed/bootstrap jobs.

## 9. Verify Dataplane Registration

```bash
kubectl exec -it -n provider2 deployment/controlplane -- \
  sh -c "curl -s http://localhost:8083/api/control/v1/dataplanes"
```

Expected:

```json
{
  "state": "REGISTERED"
}
```

## 10. Verify Readiness

ControlPlane:

```bash
kubectl exec -it -n provider2 deployment/controlplane -- \
  sh -c "curl -s http://localhost:8080/api/check/readiness"
```

IdentityHub:

```bash
kubectl exec -it -n provider2 deployment/identityhub -- \
  sh -c "curl -s http://localhost:7080/api/check/readiness"
```

Expected:

```json
{
  "isSystemHealthy": true
}
```

## 11. Verify Participant Identity

```bash
kubectl get configmap controlplane-config \
  -n provider2 \
  -o jsonpath="{.data.edc\.participant\.id}"
```

Expected:

```text
did:web:identityhub.provider2.svc.cluster.local%3A7083:provider2
```

## Result

The dataspace now contains:

- 1 Consumer
- 2 Providers
- 1 Issuer

allowing data sharing scenarios involving multiple providers and a single consumer.