<!--
SPDX-FileCopyrightText: Red Hat

SPDX-License-Identifier: Apache-2.0
-->

# Preparing GitOps repository

## Requirements

Get HUB or install https://github.com/openshift-kni/telco-reference/tree/main/telco-hub#telco-hub-cluster-setup

## Repository layout

This guide describes the Git repository layout that ArgoCD reconciles to
create CRs on the hub cluster. The O-Cloud Manager processes ClusterTemplates,
HardwareTemplates, and HardwareProfiles. ACM processes the PolicyGenerator
CRs. The BMH inventory is managed by the O-Cloud Manager metal3 hardware
plugin. For concrete examples, see the sample content under
[git-setup](../samples/git-setup/).

Recommended layout:

```text
git-root/
  clustertemplates/
    kustomization.yaml                   # Each dir has a kustomization.yaml for ArgoCD
    hardwareprofiles/                    # HardwareProfile CRs          (O-Cloud Manager)
    hardwaretemplates/                   # HardwareTemplate CRs         (O-Cloud Manager)
    inventory/                           # BareMetalHost inventory       (O-Cloud Manager metal3 plugin)
    version_4.Y.Z/                       # Version matches the OCP version to be installed
      sno-ran-du/
        sno-ran-du-v4-Y-Z-*.yaml         # ClusterTemplate CRs          (O-Cloud Manager)
        clusterinstance-defaults-*.yaml  # ConfigMap, ref by ClusterTemplate
        policytemplates-defaults-*.yaml  # ConfigMap, ref by ClusterTemplate
        extra-manifest/                  # Day-0 manifests applied during cluster installation
        pull-secret.yaml                 # Secret for spoke install      (SiteConfig operator)
        ns.yaml                          # Namespace
    version_4.Y.Z+1/                     # Optional upgrade content
  policytemplates/
    kustomization.yaml
    version_4.Y.Z/                       # Version matches the OCP version to be installed
      sno-ran-du/
        ns.yaml                          # Namespace
        msc-binding.yaml                 # ManagedClusterSetBinding      (ACM)
        sno-ran-du-pg-v4-Y-Z-*.yaml      # PolicyGenerator CRs           (ACM)
      source-crs/                        # Source CRs are reusable YAML templates used by the ACM PolicyGenerator (PG)
    version_4.Y.Z+1/                     # Optional upgrade content
```

Notes

* Keep content versioned by OCP release using the `version_4.Y.Z/` folders so templates and policies align with the target OCP.
* The `extra-manifest` and `source-crs` directories should be extracted from the `ztp-site-generate` container image. This image ships the ZTP reference
  CRs and extra manifests for each OCP release.  Make sure to bring over the
  `extra-manifest` and `source-crs` corresponding to the OCP release provided
  in the ClusterTemplate CR by using the right tag, for example: `registry.redhat.io/openshift4/ztp-site-generate-rhel8:v4.20`
  Available tags can be found in the
  [Red Hat Ecosystem Catalog](https://catalog.redhat.com/software/containers/openshift4/ztp-site-generate-rhel8/6154c29fd2c7f84a4d2edca1).
* ACM policies must be created under the namespace `ztp-<cluster-template-namespace>`. See the [example](../samples/git-setup/policytemplates/version_4.Y.Z/sno-ran-du/ns.yaml).
* In the ACM PGs, set `policyAnnotations` to include the annotation `clustertemplates.clcm.openshift.io/templates` with a comma-separated
  list of ClusterTemplates that PG is associated with. Use the ClusterTemplate metadata.name for each entry. This annotation is propagated to
  each generated root Policy. It enables the O‑Cloud Manager to identify which root policies are associated with the
  ClusterTemplate used by a ProvisioningRequest, determine the expected child policies, and accurately detect when configuration
  is complete - ensuring correct provisioning status reporting during Day-2 policy configuration changes. See the [example](../samples/git-setup/policytemplates/version_4.Y.Z/sno-ran-du/sno-ran-du-pg-v4-Y-Z-v1.yaml).

## SNO Full DU (Distributed Unit) profile

For configuring an SNO with a full DU profile according to the [RAN RDS](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/scalability_and_performance/telco-ran-du-ref-design-specs#telco-ran-du-reference-configuration-crs),
the following main samples can be used as a starting example:

* [ClusterInstance defaults ConfigMap](../samples/git-setup/clustertemplates/version_4.Y.Z/sno-ran-full-du/clusterinstance-defaults-full-du-v1.yaml)
* [PolicyTemplate defaults ConfigMap](../samples/git-setup/clustertemplates/version_4.Y.Z/sno-ran-full-du/policytemplates-defaults-full-du-v1.yaml)
* [ClusterTemplate](../samples/git-setup/clustertemplates/version_4.Y.Z/sno-ran-full-du/sno-ran-full-du-v4-Y-Z-1.yaml)
* [Full DU profile ACM Policy Generator](../samples/git-setup/policytemplates/version_4.Y.Z/sno-ran-full-du/sno-ran-full-du-pg-v4-Y-Z-v1.yaml)
* Observability configuration. This requires the creation of an ACM policy in the
`open-cluster-management-observability` namespace, as seen in [copy-acm-route-observability-v1.yaml](../samples/git-setup/policytemplates/common/copy-acm-route-observability-v1.yaml).
This policy will create a ConfigMap containing the acm-route in the same namespace as the ACM policies such that it can be used in the hub templates.
  * A custom source-cr ([source-cr-observability.yaml](../samples/git-setup/policytemplates/common/source-cr-observability.yaml)) is used. Currently **the namespaces where ACM policies are created need to be manually added to the namespace list.**
  * For allowing the creation of this policy in the `open-cluster-management-observability` namespace,
  the `AppProject` associated to the desired ACM policies needs to also contain the following in
  its `spec.destinations`:

  ```yaml
    - namespace: open-cluster-management-observability
      server: '*'
  ```

## Preparation of ArgoCD applications

The ArgoCD deployment manifests are shipped inside the `ztp-site-generate`
container. Extract them and patch for this flow:

```console
# List available tags
skopeo list-tags docker://quay.io/openshift-kni/ztp-site-generator

# Extract the ArgoCD deployment manifests for your OCP version
TAG=v4.21
mkdir -p ./ztp-out-${TAG}
podman run --rm --log-driver=none \
  quay.io/openshift-kni/ztp-site-generator:${TAG} \
  extract /home/ztp --tar | tar x -C ./ztp-out-${TAG}
```

The extracted `argocd/deployment/` directory contains the base
`AppProject`, `clusters` Application, and `policies` Application.
These need to be patched.

Save the following as `ztp-out-patch-${TAG}` and apply it after
extracting the manifests:

```console
patch -p0 < ztp-out-patch-${TAG}
```

```diff
diff -ruN ztp-out/argocd/deployment/app-project.yaml ztp-out-patched/argocd/deployment/app-project.yaml
--- ztp-out/argocd/deployment/app-project.yaml
+++ ztp-out-patched/argocd/deployment/app-project.yaml
@@ -29,5 +29,13 @@
     kind: DataImage
   - group: 'siteconfig.open-cluster-management.io'
     kind: ClusterInstance
+  - group: 'metal3.io'                    # BMH inventory managed via GitOps
+    kind: BareMetalHost
+  - group: clcm.openshift.io              # O-Cloud Manager CRDs
+    kind: ClusterTemplate
+  - group: clcm.openshift.io
+    kind: HardwareProfile
+  - group: clcm.openshift.io
+    kind: HardwareTemplate
   sourceRepos:
   - '*'
diff -ruN ztp-out/argocd/deployment/clusters-app.yaml ztp-out-patched/argocd/deployment/clusters-app.yaml
--- ztp-out/argocd/deployment/clusters-app.yaml
+++ ztp-out-patched/argocd/deployment/clusters-app.yaml
@@ -18,14 +18,19 @@
     namespace: clusters-sub
   project: ztp-app-project
   source:
-    path: ztp/gitops-subscriptions/argocd/example/clusterinstance
-    repoURL: https://github.com/openshift-kni/cnf-features-deploy
-    targetRevision: master
+    path: clustertemplates                 # point to your GitOps repo
+    repoURL: <YOUR_GITOPS_REPO_URL>
+    targetRevision: <YOUR_BRANCH>
   ignoreDifferences:
+    - group: metal3.io                     # operator updates these at runtime
+      jsonPointers:
+        - /spec/preprovisioningNetworkDataName
+        - /spec/online
+      kind: BareMetalHost
     - group: cluster.open-cluster-management.io
       kind: ManagedCluster
       managedFieldsManagers:
diff -ruN ztp-out/argocd/deployment/policies-app.yaml ztp-out-patched/argocd/deployment/policies-app.yaml
--- ztp-out/argocd/deployment/policies-app.yaml
+++ ztp-out-patched/argocd/deployment/policies-app.yaml
@@ -18,9 +18,9 @@
     namespace: policies-sub
   project: policy-app-project
   source:
-    path: ztp/gitops-subscriptions/argocd/example/policygentemplates
-    repoURL: https://github.com/openshift-kni/cnf-features-deploy
-    targetRevision: master
+    path: policytemplates                  # was policygentemplates in older ZTP
+    repoURL: <YOUR_GITOPS_REPO_URL>
+    targetRevision: <YOUR_BRANCH>
```

Replace `<YOUR_GITOPS_REPO_URL>` and `<YOUR_BRANCH>` with your GitOps
repository URL and branch before applying.

```
 oc apply -k ztp-out/argocd/deployment/
 ```

In the standard ZTP flow (OCP 4.21+), ArgoCD manages `ClusterInstance`
CRs and the SiteConfig operator creates the downstream `BareMetalHost`
resources. This flow differs because it manages a pre-provisioning BMH
inventory directly through ArgoCD (under `clustertemplates/inventory/`).
The operator updates fields on these BMHs at runtime (e.g., `spec.online`,
`spec.preprovisioningNetworkDataName`), so ArgoCD must be configured to
ignore those fields to prevent drift reconciliation.



## Next steps

Once ArgoCD has synced all the resources and the ClusterTemplates are in
`Validated` state, a cluster can be deployed by creating a
`ProvisioningRequest` CR. The `ProvisioningRequest` references a
`ClusterTemplate` and supplies per-cluster parameters. It is not part of the
GitOps repo — it is created at runtime by the SMO or a user. See the
[cluster provisioning guide](./cluster-provisioning.md) for details.
