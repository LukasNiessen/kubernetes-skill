# OpenShift Patterns

**Load this reference when detected:** OpenShift, OKD, ROSA, ARO, Route, SecurityContextConstraints, SCC, restricted-v2, OpenShift Pipelines, OpenShift GitOps, OperatorHub, OLM, ImageStream, or `oc`.

## Why this matters

OpenShift is Kubernetes with important platform APIs and security defaults. Generic upstream manifests often fail because of SecurityContextConstraints, arbitrary UID requirements, Routes, and operator-managed platform components. Do not load this file for vanilla clusters unless OpenShift APIs are present.

## Security Context Constraints

SCCs are admission controls for pod privileges.

- Do not modify default SCCs.
- Prefer workloads that run under `restricted-v2` or the platform default restricted SCC.
- Do not hardcode `runAsUser: 1000` or another fixed UID unless the namespace/SCC permits it.
- Build images to run with an arbitrary UID and writable group-owned paths.
- If a workload needs `anyuid`, host networking, host mounts, or privileged mode, require a justification and bind the narrowest SCC only to the dedicated ServiceAccount.
- Use RBAC to grant SCC use to ServiceAccounts, not broad users or groups.

## Routes and Ingress

OpenShift Routes are first-class edge routing resources.

- Use `Route` when the user asks for OpenShift-native exposure.
- Choose TLS termination deliberately: edge, passthrough, or re-encrypt.
- Do not assume Ingress annotations from nginx, AWS, GCE, or AGIC apply to Routes.
- For portable upstream manifests, use Ingress only when the target OpenShift cluster supports the intended IngressController behavior.

## Images and Runtime Assumptions

OpenShift security often exposes image problems.

- Avoid images that require root by default.
- Ensure writable directories can be written by an arbitrary UID, commonly through group permissions.
- Do not rely on Docker socket mounts or hostPath except for platform-level agents with explicit SCC approval.
- For internal registry or ImageStream workflows, keep image references and pull policies aligned with the platform's promotion model.

## Operators and OLM

When OperatorHub, OLM, or custom operators are in scope:

- Prefer Subscription/OperatorGroup/InstallPlan patterns only when the user is actually managing operators.
- Do not hand-roll CRDs owned by an installed operator unless the operator documentation requires it.
- Validate custom resources against installed CRDs, not only generic Kubernetes schemas.

## Validation

- `oc apply --dry-run=server -f <manifest>`
- `oc auth can-i use scc/restricted-v2 --as=system:serviceaccount:<namespace>:<serviceaccount>`
- `oc describe pod <name>` for SCC admission failures
- `oc get route -n <namespace>` for Route readiness
- `oc get csv,subscription,operatorgroup -A` when OLM-managed resources are involved

## LLM Mistake Checklist

- Hardcoding a UID that violates OpenShift namespace UID ranges.
- Asking users to edit default SCCs.
- Granting `anyuid` or `privileged` SCC to broad groups.
- Generating Ingress-controller annotations for OpenShift Routes.
- Assuming root-capable images will run under restricted SCCs.
- Forgetting `oc` validation and SCC checks.
- Creating operator-owned resources without verifying the CRD exists.

## Grounding Sources

- OpenShift SecurityContextConstraints: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/authentication_and_authorization/managing-pod-security-policies
- OpenShift Routes: https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/ingress_and_load_balancing/routes
