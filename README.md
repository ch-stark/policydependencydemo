# ACM Policy Dependency Demo

PolicyGenerator demo for Red Hat Advanced Cluster Management (RHACM) policy dependencies. Ten policies are generated into two hub namespaces (`policiesa` and `policiesb`). Some wait for **Compliant**, some wait for **NonCompliant**, and several use **extraDependencies** on individual policy templates.

This follows the patterns in:

- [gatekeeper-examples policyGenerator.yaml](https://github.com/ch-stark/gatekeeper-examples/blob/main/policyGenerator.yaml)
- [Using Policy Dependencies to Apply Resources in a Specific Order](https://www.redhat.com/en/blog/using-policy-dependencies-to-apply-resources-in-a-specific-order)
- [RHACM 2.12 policy deployment](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.12/html/governance/policy-deployment)

Policy Generator cannot override `policyDefaults.namespace` per policy, so this repo uses **two PolicyGenerator files**.

## What you should see

On each managed cluster the policies create objects in namespace `dep-demo`. Unsatisfied dependencies show as **Pending** in Governance until the target compliance state is reached.

| Hub namespace | Policy | Activates when |
|---------------|--------|----------------|
| `policiesa` | `policy-demo-namespace` | Immediately |
| `policiesa` | `policy-demo-baseline` | `policy-demo-namespace` is **Compliant** |
| `policiesa` | `policy-demo-rbac` | `policy-demo-namespace` is **Compliant** |
| `policiesa` | `policy-demo-stack` | Baseline and RBAC are **Compliant**, then templates run in order via extraDependencies |
| `policiesa` | `policy-demo-health` | `policy-demo-stack` is **Compliant** |
| `policiesb` | `policy-required-annotation` | `policiesa/policy-demo-namespace` is **Compliant** |
| `policiesb` | `policy-remediate-annotation` | ConfigurationPolicy `demo-required-annotation` is **NonCompliant** |
| `policiesb` | `policy-followup-compliant` | `policiesa/policy-demo-health` is **Compliant** |
| `policiesb` | `policy-alert-on-violation` | `policy-required-annotation` is **NonCompliant** |
| `policiesb` | `policy-gated-release` | extraDependencies: local precheck **Compliant**, `policiesa/policy-demo-stack` **Compliant**, and `policy-followup-compliant` **Compliant** |

```mermaid
flowchart TD
  subgraph policiesa["policiesa"]
    ns[policy-demo-namespace]
    base[policy-demo-baseline]
    rbac[policy-demo-rbac]
    stack[policy-demo-stack]
    health[policy-demo-health]
    ns -->|Compliant| base
    ns -->|Compliant| rbac
    base -->|Compliant| stack
    rbac -->|Compliant| stack
    cfg[demo-app-config] -->|extraDep Compliant| secret[demo-app-secret]
    secret -->|extraDep Compliant| ready[demo-app-ready]
    stack --> cfg
    stack -->|Compliant| health
  end

  subgraph policiesb["policiesb"]
    check[policy-required-annotation]
    rem[policy-remediate-annotation]
    follow[policy-followup-compliant]
    alert[policy-alert-on-violation]
    gated[policy-gated-release]
    ns -->|Compliant| check
    check -->|NonCompliant extraDep| rem
    check -->|NonCompliant| alert
    health -->|Compliant| follow
    pre[demo-release-precheck] -->|extraDep Compliant| rel[demo-release]
    stack -->|extraDep Compliant| rel
    follow -->|extraDep Compliant| rel
    gated --> pre
  end
```

## Dependency kinds used

**`spec.dependencies`** gates the whole Policy. **`extraDependencies`** gates one ConfigurationPolicy template.

| Pattern | Where |
|---------|--------|
| Policy `dependencies` + `Compliant` | baseline, rbac, stack, health, required-annotation, followup |
| Policy `dependencies` + `NonCompliant` | `policy-alert-on-violation` |
| Template `extraDependencies` + `Compliant` | `policy-demo-stack`, `policy-gated-release` |
| Template `extraDependencies` + `NonCompliant` | `policy-remediate-annotation` |
| Cross-namespace Policy dependency | policiesb → policiesa (`namespace: policiesa`) |
| Multiple extraDependencies on one template | `demo-release` waits on three objects |
| `ignorePending: true` | remediator and alert, so Pending is treated as Compliant when there is nothing to do |

`policy-remediate-annotation` matches the certificate-refresh example in the RHACM blog: the enforce template stays Pending while the check is Compliant, and `ignorePending: true` keeps the Policy Compliant when there is nothing to remediate. `pruneObjectBehavior` is `None` on that policy so it does not delete `dep-demo`. The alert notice uses `DeleteAll` so it is removed when the violation is gone.

Leave `namespace` empty on ConfigurationPolicy extraDependencies. Those objects live in the managed cluster namespace. Set `namespace` on Policy dependencies (`policiesa` or `policiesb`).

## Apply with OpenShift GitOps

You do **not** need two Argo CD instances. `policiesa` and `policiesb` are hub namespaces for the generated Policies. One cluster-scoped OpenShift GitOps instance (`openshift-gitops` in `openshift-gitops`) runs the Policy Generator plugin. Two **Applications** (not two Argo CD CRs) each read one path from this git repo.

That matches [policy-openshift-gitops-policygenerator.yaml](https://github.com/open-cluster-management-io/policy-collection/blob/main/community/CM-Configuration-Management/policy-openshift-gitops-policygenerator.yaml): patch the default `openshift-gitops` Argo CD CR, grant it Policy RBAC, then let Applications generate Policies.

```mermaid
flowchart LR
  git["github.com/ch-stark/policydependencydemo"]
  argocd["Argo CD openshift-gitops<br/>Policy Generator plugin"]
  appA["Application<br/>path: policiesa"]
  appB["Application<br/>path: policiesb"]
  nsA["Policies in policiesa"]
  nsB["Policies in policiesb"]
  git --> appA --> argocd --> nsA
  git --> appB --> argocd --> nsB
```

```bash
# 1. Hub namespaces, ManagedClusterSetBindings, and GitOps bootstrap Policies
#    (operator Subscription + Policy Generator on the default Argo CD instance).
#    Those Policies are bound only to local-cluster.
kubectl apply -k setup/

# 2. Wait until both GitOps Policies are Compliant on local-cluster
kubectl get policy -n open-cluster-management-global-set
oc -n openshift-gitops get pods -l app.kubernetes.io/name=openshift-gitops-repo-server

# 3. Applications that generate Policies from this git repo
kubectl apply -k setup/applications/
```

`setup/gitops/policy-openshift-gitops-policygenerator.yaml` waits for `openshift-gitops-operator` to be **Compliant**, then enforces:

- Policy Generator init container on `ArgoCD/openshift-gitops` (binary from the hub `acm-cli-downloads` image)
- `kustomizeBuildOptions: --enable-alpha-plugins`
- ClusterRole/Binding so the `openshift-gitops-argocd-application-controller` service account can create Policies, PolicySets, Placements, and PlacementBindings

After sync, Governance should show 10 Policies (5 in `policiesa`, 5 in `policiesb`).

To preview generated Policies without GitOps:

```bash
kustomize build --enable-alpha-plugins --enable-exec policiesa/
kustomize build --enable-alpha-plugins --enable-exec policiesb/
```

Install the [Policy Generator plugin](https://github.com/open-cluster-management-io/policy-generator-plugin) under `~/.config/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator/PolicyGenerator` (or set `KUSTOMIZE_PLUGIN_HOME`) if you apply with `kubectl -k` instead of Argo CD.

## Placement

Each generator creates a Placement that matches all clusters in the bound cluster set (`global`). Restrict it by editing `policyDefaults.placement` / `policySets[].placement` in the PolicyGenerator files, for example:

```yaml
placement:
  name: placement-policiesa
  labelSelector:
    matchExpressions:
      - key: local-cluster
        operator: In
        values:
          - "true"
```

## Layout

```
setup/
  01_namespaces.yaml              # policiesa, policiesb
  02_managedclustersetbinding.yaml
  gitops/                         # ACM Policies that configure OpenShift GitOps on local-cluster
  applications/                   # Argo CD AppProject + 2 Applications
policiesa/                        # PolicyGenerator → 5 Policies in namespace policiesa
policiesb/                        # PolicyGenerator → 5 Policies in namespace policiesb
```
