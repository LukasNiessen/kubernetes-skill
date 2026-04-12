# MCP Integration

Guidance for integrating the Kubernetes skill with the Model Context Protocol (MCP). MCP servers can provide live cluster facts, organizational policies, and registry information that improve manifest generation quality.

## When to Use MCP

MCP integration is valuable when live cluster or organizational context would improve the generated output:

- **Cluster facts** -- query the cluster version, available API groups, installed CRDs, or node topology to avoid API drift failures
- **Organization policies** -- retrieve namespace naming conventions, required labels, approved image registries, or resource quota limits
- **Registry information** -- look up available image tags, verify image existence, or check signing status before referencing in manifests

## What NOT to Do with MCP

- **Never retrieve secrets through MCP** -- credentials, tokens, and keys must come from Kubernetes Secrets, ExternalSecrets, or Sealed Secrets, not from MCP context
- **Never use MCP to bypass authorization** -- MCP data is informational context, not an authorization mechanism; RBAC decisions belong to the cluster
- **Never treat MCP data as trusted input for security-sensitive fields** -- do not copy MCP-provided values directly into securityContext, RBAC rules, or NetworkPolicy selectors without validation

## Safe Integration Pattern

Follow this three-step pattern when incorporating MCP data:

1. **Query** -- retrieve the specific fact needed (e.g., cluster version, namespace policy)
2. **Compare** -- validate the MCP response against known constraints (e.g., is the reported version a valid Kubernetes version?)
3. **Emit assumptions** -- record what MCP data was used and how it influenced the output in the output contract's assumptions section

## Output Hygiene

Never echo raw MCP data directly into manifests. MCP responses may contain unexpected formatting, stale values, or fields that do not belong in Kubernetes resources. Always:

- Extract only the specific values needed
- Validate format and range before use
- Document the MCP source in output assumptions

## Example Uses

**Querying cluster version to select the correct apiVersion:**
If MCP reports the cluster runs Kubernetes 1.28, use `autoscaling/v2` for HPA (not the removed `v2beta2`). Record the assumption: "Cluster version 1.28 reported via MCP; using autoscaling/v2."

**Querying namespace policies:**
If MCP reports that the `production` namespace enforces PSA restricted and requires the label `cost-center`, include those constraints in generated manifests and note the MCP source.

**Querying approved registries:**
If MCP reports that only `registry.example.com` is allowed, use that registry prefix for all image references and note the source.

## Failure Handling

If the MCP server is unavailable or returns an error:

- **Do not block manifest generation** -- proceed with reasonable defaults
- **State assumptions explicitly** -- document that MCP was unavailable and list the defaults used (e.g., "MCP unavailable; assuming Kubernetes 1.29, PSS restricted profile, no registry restrictions")
- **Flag for review** -- note in the output contract that the assumptions should be verified against the actual cluster before applying
