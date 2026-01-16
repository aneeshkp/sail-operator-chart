# Sail Operator Helm Chart - Architecture

Deploy Red Hat Sail Operator (OSSM 3.x) on vanilla Kubernetes (AKS, EKS, GKE) without OLM.

## Source Repositories

| Repo | Purpose |
|------|---------|
| [istio-ecosystem/sail-operator](https://github.com/istio-ecosystem/sail-operator) | Operator code (upstream) |
| [openshift-service-mesh/sail-operator](https://github.com/openshift-service-mesh/sail-operator) | Red Hat fork |
| [lburgazzoli/olm-extractor](https://github.com/lburgazzoli/olm-extractor) | Extract OLM bundles for non-OLM clusters |

**OLM Bundle:** `registry.redhat.io/openshift-service-mesh/istio-sail-operator-bundle`
- [Red Hat Catalog](https://catalog.redhat.com/software/containers/openshift-service-mesh/istio-operator-bundle)

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Helm Chart                                │
├─────────────────────────────────────────────────────────────────┤
│  Presync (helmfile)                                             │
│  ├── Gateway API CRDs (v1.4.0)                                  │
│  ├── Gateway API Inference Extension CRDs (v1.2.0)              │
│  ├── Sail Operator CRDs (19 Istio CRDs, server-side apply)      │
│  └── istio-system namespace                                     │
├─────────────────────────────────────────────────────────────────┤
│  Helm Install                                                   │
│  ├── Pull secret (redhat-pull-secret)                           │
│  ├── istiod ServiceAccount with imagePullSecrets                │
│  ├── Sail Operator deployment + RBAC                            │
│  └── Istio CR (with Gateway API enabled)                        │
├─────────────────────────────────────────────────────────────────┤
│  Operator (post-install)                                        │
│  ├── Reconciles istiod SA (preserves imagePullSecrets)          │
│  └── Deploys Istio components                                   │
│      ├── istiod (control plane)                                 │
│      ├── istio-cni (optional)                                   │
│      └── Gateway pods (per-namespace, on demand)                │
├─────────────────────────────────────────────────────────────────┤
│  Manual Steps                                                   │
│  └── Copy pull secret to app namespaces (for Gateway pods)      │
└─────────────────────────────────────────────────────────────────┘
```

## Components

### Namespaces
- `istio-system` - Operator and control plane namespace
- Application namespaces - Where Gateway pods run (need pull secret copied)

### Images (registry.redhat.io)

| Component | Image |
|-----------|-------|
| Sail Operator | `registry.redhat.io/openshift-service-mesh/istio-sail-operator-rhel9` |
| Istiod | `registry.redhat.io/openshift-service-mesh/istio-pilot-rhel9` |
| Proxy (sidecar/gateway) | `registry.redhat.io/openshift-service-mesh/istio-proxyv2-rhel9` |
| CNI | `registry.redhat.io/openshift-service-mesh/istio-cni-rhel9` |

### CRDs

**Sail Operator CRDs:**
- `istios.sailoperator.io` - Main Istio CR
- `istiorevisions.sailoperator.io` - Revision management
- `istiorevisiontags.sailoperator.io` - Revision tags
- `istiocnis.sailoperator.io` - CNI configuration
- `ztunnels.sailoperator.io` - Ambient mesh ztunnel

**Istio Networking CRDs:**
- `virtualservices.networking.istio.io`
- `destinationrules.networking.istio.io`
- `gateways.networking.istio.io`
- `serviceentries.networking.istio.io`
- `sidecars.networking.istio.io`
- `workloadentries.networking.istio.io`
- `workloadgroups.networking.istio.io`
- `envoyfilters.networking.istio.io`
- `proxyconfigs.networking.istio.io`

**Istio Security CRDs:**
- `authorizationpolicies.security.istio.io`
- `peerauthentications.security.istio.io`
- `requestauthentications.security.istio.io`

**Istio Extensions CRDs:**
- `wasmplugins.extensions.istio.io`
- `telemetries.telemetry.istio.io`

**Gateway API CRDs (from Kubernetes SIG):**
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `grpcroutes.gateway.networking.k8s.io`
- `tcproutes.gateway.networking.k8s.io`
- `tlsroutes.gateway.networking.k8s.io`
- `referencegrants.gateway.networking.k8s.io`
- `gatewayclasses.gateway.networking.k8s.io`

**Gateway API Inference Extension CRDs (for LLM serving):**
- `inferencepools.inference.networking.x-k8s.io`
- `inferencemodels.inference.networking.x-k8s.io`

## Non-OpenShift Adaptations

| OpenShift Feature | Problem | Solution |
|-------------------|---------|----------|
| OLM (Subscription, OperatorGroup) | Not available on vanilla K8s | Use Helm + helmfile |
| Global pull secret | Node-level registry auth | Manual SA patching + copy to namespaces |
| Console Plugin | OpenShift console integration | Excluded via olm-extractor |
| Routes | OpenShift routing | Use Gateway API instead |

## olm-extractor Integration

The `scripts/update-bundle.sh` uses [olm-extractor](https://github.com/lburgazzoli/olm-extractor) to extract manifests from Red Hat's OLM bundle:

```bash
podman run --rm $AUTH_ARG \
  quay.io/lburgazzoli/olm-extractor:main \
  run "$BUNDLE_IMAGE" \
  -n istio-system \
  --exclude '.kind == "ConsoleCLIDownload"'
```

## Pull Secret Distribution

### istiod ServiceAccount (automated)

The `istiod` SA is pre-created by Helm with `imagePullSecrets`:

```yaml
# templates/serviceaccount-istiod.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: istiod
  namespace: istio-system
imagePullSecrets:
  - name: redhat-pull-secret
```

The operator uses **strategic merge patch** when reconciling, preserving the `imagePullSecrets`.

### Gateway SAs (manual)

Gateway SAs are created dynamically per-namespace when Gateway resources are deployed:

```bash
# Copy secret and patch SA in app namespace
./scripts/copy-pull-secret.sh <namespace> <gateway-sa-name>
```

**Why not pre-create Gateway SAs?**
- Gateway SAs are dynamically named based on Gateway resource names
- SAs are created in user namespaces, not operator namespace
- Would require knowing all namespaces and Gateway names in advance

## Bundle Sources

| Source | Registry | Auth Required | Use Case |
|--------|----------|---------------|----------|
| `redhat` | registry.redhat.io | Yes (pull secret) | Production, supported |
| `konflux` | quay.io/redhat-user-workloads | No | Development, testing |

## File Structure

```
sail-operator-chart/
├── Chart.yaml                             # Helm chart metadata
├── values.yaml                            # Default values
├── helmfile.yaml.gotmpl                   # Helmfile for deployment
├── .helmignore                            # Excludes large CRDs from Helm
├── environments/
│   └── default.yaml                       # Environment config
├── manifests-crds/                        # CRDs (applied with --server-side)
│   ├── customresourcedefinition-istios-*.yaml
│   └── ... (19 Istio CRDs)
├── templates/
│   ├── deployment-servicemesh-operator3.yaml
│   ├── serviceaccount-*.yaml              # Operator SA
│   ├── serviceaccount-istiod.yaml         # istiod SA with imagePullSecrets
│   ├── istio-cr.yaml                      # Istio CR with Gateway API
│   ├── pull-secret.yaml                   # Registry pull secret
│   └── *role*.yaml                        # RBAC
└── scripts/
    ├── update-bundle.sh                   # Extract new bundle version
    ├── update-pull-secret.sh              # Update expired pull secret
    ├── cleanup.sh                         # Uninstall
    ├── cleanup-images.sh                  # Remove cached images (uses Eraser)
    ├── copy-pull-secret.sh                # Copy secret to app namespaces
    └── post-install-message.sh            # Post-install instructions
```

## Upgrade Process

```bash
# Update to new bundle version
./scripts/update-bundle.sh 3.3.0 redhat

# Review changes
git diff

# Deploy
helmfile apply
```

The `istiod` SA with `imagePullSecrets` is preserved across upgrades.

## Gateway API Integration

The chart enables Gateway API by default in the Istio CR:

```yaml
spec:
  values:
    pilot:
      env:
        PILOT_ENABLE_GATEWAY_API: "true"
        PILOT_ENABLE_GATEWAY_API_STATUS: "true"
        PILOT_ENABLE_GATEWAY_API_DEPLOYMENT_CONTROLLER: "true"
```

When users create Gateway resources, Istio automatically:
1. Creates a Deployment for the Gateway
2. Creates a ServiceAccount for the Gateway
3. Runs Envoy proxy pods (istio-proxyv2)

These pods need pull secrets, hence the manual `copy-pull-secret.sh` step.

## Comparison with cert-manager-operator-chart

| Aspect | cert-manager | sail-operator |
|--------|--------------|---------------|
| Operand SAs | Fixed (3 known SAs) | istiod (fixed) + Gateway (dynamic) |
| SA namespaces | Single (`cert-manager`) | `istio-system` + user namespaces |
| Pre-create SAs? | Yes (all 3) | Yes (istiod only) |
| Pull secret approach | Fully automated | istiod: automated, Gateway: manual |
| OpenShift API stubs | Infrastructure CRD | None needed |
