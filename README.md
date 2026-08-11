# CST8919 Lab 3 – Cloud Governance Gone Rogue: Azure Policy Lab

## Summary

This lab implements Azure Policy governance for "MapleTech Solutions": three custom
deny-effect policies bundled into a policy initiative and assigned to a resource
group, enforcing region lockdown, mandatory tagging, and a ban on public IP
addresses.

- Policy definitions: [`policy-lab/policy-definitions/`](policy-lab/policy-definitions/)
- Test evidence: [`policy-lab/screenshots/`](policy-lab/screenshots/) (CLI output, see note below)
- Video demo: [https://youtu.be/9SOmQOYdLfg](https://youtu.be/9SOmQOYdLfg)

## Policies

| Policy | Effect | Rule |
|---|---|---|
| `Only-CanadaCentral` | Deny | Denies any resource whose `location` is not `canadacentral`. |
| `Require-ProjectName-Tag` | Deny | Denies any resource missing the `ProjectName` tag. |
| `Deny-Public-IP` | Deny | Denies any resource of type `Microsoft.Network/publicIPAddresses`. |

These three are bundled into the **`MapleTech Secure Foundation`** initiative
(category: Security) and assigned to resource group `rg-cst8919-lab3`
(Canada Central) with enforcement mode `Default` (Enforce).

All three were created and assigned using the Azure CLI; the JSON policy rule
files used are in `policy-lab/policy-definitions/`.

## Test Results

| # | Action | Expected | Result |
|---|--------|----------|--------|
| 1 | Deploy a resource outside Canada Central | Denied | **Denied** — `Only-CanadaCentral` |
| 2 | Deploy a Storage Account without `ProjectName` tag | Denied | **Denied** — `Require-ProjectName-Tag` |
| 3 | Create a Public IP address | Denied | **Denied** — `Deny-Public-IP` |
| 4 | Deploy a compliant resource (Canada Central + tag) | Allowed | **Not directly demonstrable** — see note below |

Full CLI transcripts for each test are in `policy-lab/screenshots/`.

### Important note on Test Case 4 and region substitutions

The Azure subscription used for this lab (Azure for Students) has a **built-in,
subscription-wide region restriction** (policy assignment `sys.regionrestriction`)
that only permits resource deployment to: Mexico Central, West US 3, Norway East,
North Central US, and South Central US. This is a platform-level restriction, set
by Azure independently of anything created for this lab, and it does **not**
include Canada Central.

Consequences:
- **Test 1** used North Central US in place of East US as the "wrong region"
  example, since East US is also outside the subscription's permitted list. The
  outcome is identical either way — the point is any region other than Canada
  Central gets denied by `Only-CanadaCentral`.
- **Tests 2 and 3** were run in an isolated scratch resource group with only the
  relevant single policy assigned (rather than the full initiative). This was
  necessary because, with all three policies assigned together, `Only-CanadaCentral`
  would deny *every* request made outside Canada Central regardless of tagging or
  resource type — masking whether the tag and public-IP policies were independently
  working. Testing them in isolation gives clean, unambiguous evidence that each
  policy fires for its own specific reason. Portal screenshots of these isolated
  test results were also captured from this scratch resource group's Activity Log;
  it is a temporary testing aid only — `rg-cst8919-lab3` and the full `MapleTech
  Secure Foundation` initiative are the artifacts that satisfy the lab requirement.
- **Test 4 (the fully-compliant "Allowed" case) could not be physically
  demonstrated**, because it requires a real deployment to Canada Central, which
  this subscription cannot reach at all — every such attempt is blocked before
  Azure Policy is even evaluated. Test 2's control case (a correctly-tagged
  resource succeeding once the tag was added, in an allowed region) is the closest
  available evidence that the initiative allows compliant resources through rather
  than denying unconditionally.

## Challenges and Lessons Learned

- The most significant obstacle wasn't the policy logic itself — it was discovering
  that the target subscription has a hard, tenant-level region restriction that
  doesn't include the region the lab requires (Canada Central). This is a good
  example of how cloud environments often have *layered* restrictions (subscription
  quota/region policies stacked on top of custom governance policies), and how
  troubleshooting "why was this denied?" requires reading the actual policy
  identifiers in the error response rather than assuming it's always your own
  policy at fault.
- Testing policies bundled in one initiative can make it hard to attribute a denial
  to a specific rule when multiple rules would independently deny the same request.
  Isolating a policy under test in its own assignment/scope is a reliable way to
  get unambiguous evidence.
- New policy assignments are not instantly enforced — there's a short propagation
  delay (in this lab, under two minutes) before `deny` effects take hold across all
  resource providers.

## Reproducing

The Azure CLI commands used to create the resource group, policy definitions,
initiative, and assignment are summarized below (see `policy-lab/policy-definitions/`
for the full rule JSON):

```bash
# Resource group
az group create --name rg-cst8919-lab3 --location canadacentral

# Policy definitions (one per JSON rule file)
az policy definition create --name Only-CanadaCentral \
  --display-name Only-CanadaCentral --mode Indexed \
  --rules policy-lab/policy-definitions/only-canada-central.json

az policy definition create --name Require-ProjectName-Tag \
  --display-name Require-ProjectName-Tag --mode Indexed \
  --rules policy-lab/policy-definitions/require-projectname-tag.json

az policy definition create --name Deny-Public-IP \
  --display-name Deny-Public-IP --mode Indexed \
  --rules policy-lab/policy-definitions/deny-public-ip.json

# Initiative bundling the three policies
az policy set-definition create \
  --name MapleTech-Secure-Foundation \
  --display-name "MapleTech Secure Foundation" \
  --definitions '[
    { "policyDefinitionId": "/subscriptions/<sub-id>/providers/Microsoft.Authorization/policyDefinitions/Only-CanadaCentral" },
    { "policyDefinitionId": "/subscriptions/<sub-id>/providers/Microsoft.Authorization/policyDefinitions/Require-ProjectName-Tag" },
    { "policyDefinitionId": "/subscriptions/<sub-id>/providers/Microsoft.Authorization/policyDefinitions/Deny-Public-IP" }
  ]'

# Assign the initiative to the resource group (Enforce)
az policy assignment create \
  --name MapleTech-Secure-Foundation-Assignment \
  --display-name "MapleTech Secure Foundation" \
  --policy-set-definition MapleTech-Secure-Foundation \
  -g rg-cst8919-lab3 \
  --enforcement-mode Default
```
