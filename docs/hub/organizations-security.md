# Access control in organizations

> [!TIP]
> You can set up [Single Sign-On (SSO)](./security-sso) to be able to map access control rules from your organization's Identity Provider.

> [!TIP]
> Advanced and more fine-grained access control can be achieved with [Resource Groups](./security-resource-groups).
>
> The Resource Group feature is part of the <a href="https://huggingface.co/enterprise">Team, Enterprise, & Academia Hub</a> plans.

Members of organizations can have four different roles: `read`, `contributor`, `write`, or `admin`:

- `read`: read-only access to the Organization's repos and metadata/settings (eg, the Organization's profile, members list, API token, etc).

- `contributor`: additional write rights to the subset of the Organization's repos that were created by the user. I.e., users can create repos and _then_ modify only those repos. This is similar to the `write` role, but scoped to repos _created_ by the user.

- `write`: write rights to all the Organization's repos. Users can create, delete, or rename any repo in the Organization namespace. A user can also edit and delete files from the browser editor and push content with `git`.

- `admin`: in addition to write rights on repos, admin members can update the Organization's profile, refresh the Organization's API token, and manage Organization members.

As an organization `admin`, go to the **Members** section of the org settings to manage roles for users. You can also change a member's organization role (and optionally their resource group roles) programmatically via the API — see [Change member role via API](#change-member-role-via-api) at the bottom of this page.

<div class="flex justify-center">
<img class="block dark:hidden" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-members-page.png"/>
<img class="hidden dark:block" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-members-page-dark.png"/>
</div>

## Viewing members' email address

> [!WARNING]
> This feature is part of the <a href="https://huggingface.co/enterprise">Team & Enterprise</a> plans.

You may be able to view the email addresses of members of your organization. The visibility of the email addresses depends on the organization's SSO configuration, or verified organization status.

- By [verifying an email domain](./organizations-managing#organization-email-domain) for your organization, you can view the email addresses of members with a matching email domain.
- If SSO is configured for your organization, you can view the email address for each of your organization members by setting `Matching email domains` in the SSO configuration

<div class="flex justify-center">
<img class="block dark:hidden" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-members-page-emails.png"/>
<img class="hidden dark:block" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-members-page-emails-dark.png"/>
</div>

## Managing Access Tokens with access to my organization

See [Tokens Management](./enterprise-tokens-management)

## Change member role via API {#change-member-role-via-api}

You can change a member's **organization role** (Read / Contributor / Write / Admin) and, optionally, their roles in **resource groups** using the Hub API. The API updates **one member per request**. To change roles for multiple members, call the API in a loop (examples below).

**OpenAPI reference:** [PUT /api/organizations/\{name\}/members/\{username\}/role](https://huggingface.co/spaces/huggingface/openapi#tag/orgs/PUT/api/organizations/%7Bname%7D/members/%7Busername%7D/role)

### Prerequisites

- Your organization must have a **subscription plan** (e.g. Team, Enterprise, or Academia Hub). The endpoint returns 402 otherwise.
- You must be authenticated as an organization member with **Write** (or Admin) permission on the organization.
- The target user must already be a **member** of the organization.

### Base URL and authentication

- **Base URL:** `https://huggingface.co` (or your Hub host if self-hosted).
- **Authentication:** Send your token in the request header:
  ```http
  Authorization: Bearer <your_access_token>
  ```
  Create a token with at least **Write** access at [https://huggingface.co/settings/tokens](https://huggingface.co/settings/tokens).

### Change member role endpoint

**Request**

```http
PUT /api/organizations/{org_name}/members/{username}/role
Authorization: Bearer <your_access_token>
Content-Type: application/json

{
  "role": "read",
  "resourceGroups": []
}
```

- **Path parameters**
  - `org_name`: Organization slug (e.g. `my-org`).
  - `username`: Hugging Face **username** of the member whose role you are changing.
- **Body**
  - `role` (required): The member's **organization-level** role. One of: `"read"`, `"contributor"`, `"write"`, or `"admin"`.
  - `resourceGroups` (optional): Array of resource group assignments for this user. Each item:
    - `id`: Resource group ID (24-character hex string; get IDs from the [resource groups list API](./security-resource-groups#list-resource-groups)).
    - `role`: Role in that resource group: `"read"`, `"contributor"`, `"write"`, or `"admin"`.
  - If you omit `resourceGroups` or pass `[]`, the user is removed from all resource groups. To only change org role and leave resource groups unchanged, pass their current resource group memberships (the body always sets both org role and resource group list).

**Example (curl) – set org role to "read", no resource groups**

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $HF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"read","resourceGroups":[]}' \
  "https://huggingface.co/api/organizations/my-org/members/member1/role"
```

**Example (curl) – set org role and resource group roles**

```bash
curl -s -X PUT \
  -H "Authorization: Bearer $HF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"role":"write","resourceGroups":[{"id":"507f1f77bcf86cd799439011","role":"read"}]}' \
  "https://huggingface.co/api/organizations/my-org/members/member2/role"
```

**Success response:** Status `200 OK`; body: `{ "success": true }`.

**Typical errors**

- `400` — Invalid body (e.g. invalid role or resource group `id`).
- `402` — Organization does not have a subscription plan.
- `403` — Not allowed (e.g. you lack Write on the org, or a resource group is not in the org).
- `404` — Organization or user not found.

### Updating multiple members

The API changes **one member per request**. There is no bulk endpoint. To update many members, call the endpoint once per username (e.g. from a list or CSV).

**Example: Bash – loop over usernames, same role for all**

```bash
ORG_NAME="my-org"
ROLE="read"
for username in member1 member2 member3 member4; do
  echo "Setting $username to $ROLE ..."
  curl -s -w "\n%{http_code}" -X PUT \
    -H "Authorization: Bearer $HF_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"role\":\"$ROLE\",\"resourceGroups\":[]}" \
    "https://huggingface.co/api/organizations/$ORG_NAME/members/$username/role"
  echo ""
done
```

**Example: Python – loop over usernames**

```python
import os
import requests

BASE_URL = "https://huggingface.co"
HF_TOKEN = os.environ.get("HF_TOKEN", "")

def change_member_role(org_name: str, username: str, role: str, resource_groups: list | None = None):
    payload = {"role": role, "resourceGroups": resource_groups or []}
    r = requests.put(
        f"{BASE_URL}/api/organizations/{org_name}/members/{username}/role",
        headers={"Authorization": f"Bearer {HF_TOKEN}", "Content-Type": "application/json"},
        json=payload,
    )
    if r.status_code != 200:
        raise RuntimeError(f"{r.status_code}: {r.text}")
    return r.json()

org_name = "my-org"
role = "read"
for username in ["member1", "member2", "member3", "member4"]:
    print(f"Setting {username} to {role} ... ", end="")
    try:
        change_member_role(org_name, username, role)
        print("OK")
    except Exception as e:
        print(f"Failed: {e}")
```

For different roles per user, loop over `(username, role)` pairs (e.g. from a CSV) and call `change_member_role` for each.
