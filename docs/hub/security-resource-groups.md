# Advanced Access Control in Organizations with Resource Groups

> [!WARNING]
> This feature is part of the <a href="https://huggingface.co/enterprise">Team, Enterprise, & Academia Hub</a> plans.

In your Hugging Face organization, you can use Resource Groups to control which members have access to specific repositories.

## How does it work?

Resource Groups allow organization administrators to group related repositories together, allowing different teams in your organization to work on independent sets of repositories.

A repository can belong to only one Resource Group.

Organizations members need to be added to the Resource Group to access its repositories. An Organization Member can belong to several Resource Groups.

Members are assigned a role in each Resource Group that determines their permissions for the group's repositories. Four distinct roles exist for Resource Groups:

- `read`: Grants read access to repositories within the Resource Group.
- `contributor`: Provides extra write rights to the subset of the Organization's repositories created by the user (i.e., users can create repos and then modify only those repos). Similar to the 'Write' role, but limited to repos created by the user.
- `write`: Offers write access to all repositories in the Resource Group. Users can create, delete, or rename any repository in the Resource Group.
- `admin`: In addition to write permissions on repositories, admin members can administer the Resource Group — add, remove, and alter the roles of other members. They can also manage already existing repositories in a Resource Group.

In addition, Organization admins can manage all resource groups inside the organization. This includes moving repositories in and out of any Resource Group.

Resource Groups also affect the visibility of private repositories inside the organization. A private repository that is part of a Resource Group will only be visible to members of that Resource Group. Public repositories, on the other hand, are visible to anyone, inside and outside the organization.

## Getting started

Head to your Organization's settings, then navigate to the "Resource Group" tab in the left menu.

<div class="flex justify-center">
    <img class="block dark:hidden" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-resource-groups-page.png"/>
    <img class="hidden dark:block" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-resource-groups-page-dark.png"/>
</div>

If you are an admin of the organization, you can create and manage Resource Groups from that page.

After creating a resource group and giving it a meaningful name, you can start adding repositories and users to it.

<div class="flex justify-center">
    <img class="block dark:hidden" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-resource-groups-manage-empty-page.png"/>
    <img class="hidden dark:block" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-resource-groups-manage-empty-page-dark.png"/>
</div>

Remember that a repository can be part of only one Resource Group. You'll be warned when trying to add a repository that already belongs to another Resource Group.

<div class="flex justify-center">
    <img class="block dark:hidden" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-resource-groups-manage-move-repo.png"/>
    <img class="hidden dark:block" src="https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/hub/org-resource-groups-manage-move-repo-dark.png"/>
</div>

## Resource Groups API

The following endpoints let you **list** resource groups and **add** users to them. To **change** an existing member's organization-level role along with their resource group assignments, use the [Change member role via API](./organizations-security#change-member-role-via-api) endpoint in [Access control in organizations](./organizations-security).

**OpenAPI reference:** [Resource groups](https://huggingface.co/spaces/huggingface/openapi#tag/resource-groups)

### Base URL and authentication

- **Base URL:** `https://huggingface.co` (or your Hub host if self-hosted).
- **Authentication:** Use one of:
  - **Access token (recommended for scripts):** Create a token at [https://huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) with at least **Write** access. Send it in the request header:
    ```http
    Authorization: Bearer <your_access_token>
    ```
  - **Session cookie:** If calling from a browser or tool that shares the same session as the Hub UI, the cookie is sent automatically.

### List resource groups

Get all resource groups you can manage for the organization. Use this to obtain each group's `id` for the add-users calls.

**Request**

```http
GET /api/organizations/{org_name}/resource-groups
Authorization: Bearer <your_access_token>
```

**Example (curl)**

```bash
curl -s -H "Authorization: Bearer $HF_TOKEN" \
  "https://huggingface.co/api/organizations/my-org/resource-groups"
```

**Example response (trimmed)**

```json
[
  {
    "id": "507f1f77bcf86cd799439011",
    "name": "Cohort 2024",
    "description": "Members in this group",
    "users": [...],
    "repos": [...]
  }
]
```

Use the `id` of each resource group when adding users.

### Add users to a resource group

Add one or more users to a single resource group in one request. You can send multiple users in the same request.

**Request**

```http
POST /api/organizations/{org_name}/resource-groups/{resource_group_id}/users
Authorization: Bearer <your_access_token>
Content-Type: application/json

{
  "users": [
    { "user": "member1", "role": "read" },
    { "user": "member2", "role": "read" },
    { "user": "member3", "role": "write" }
  ]
}
```

- **Path parameters**
  - `org_name`: Organization slug (e.g. `my-org`).
  - `resource_group_id`: The resource group's `id` (24-character hex string from the list endpoint).
- **Body**
  - `users`: Array of objects. Each object must have:
    - `user`: Hugging Face **username** (required).
    - `role`: One of `"read"`, `"contributor"`, `"write"`, `"admin"`.

**Example (curl)**

```bash
curl -s -X POST \
  -H "Authorization: Bearer $HF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"users":[{"user":"member1","role":"read"},{"user":"member2","role":"read"}]}' \
  "https://huggingface.co/api/organizations/my-org/resource-groups/507f1f77bcf86cd799439011/users"
```

**Success:** Status `200 OK`; body is the updated resource group object (includes the new users in `users`).

**Typical errors:**

- `400` — e.g. user not found, duplicate usernames, or invalid body.
- `403` — Not allowed (e.g. not in org, or already in the resource group). The message will indicate whether users are not in the organization or already in the group.

### Adding members via email (workaround)

The add-users endpoint only accepts **Hugging Face usernames**, not emails. If you have a list of **emails** (e.g. member emails), you can resolve email → username first, then call the add-users API.

The email filter works when the email's domain matches one of the organization's allowed domains: the **Organization email domain** (Settings → Account → Organization email domain) and/or the org's **SSO allowed domains** (if SSO is configured).

**Step 1 – Resolve email to username**

```http
GET /api/organizations/{org_name}/members?email={email}&limit=1
Authorization: Bearer <your_access_token>
```

The `email` query parameter only filters when the email's domain matches the organization's Organization email domain or one of its SSO allowed domains. Response is an array of members; each member has `user` (username). Use `user` for the add-users call.

**Step 2 – Add to resource group**

Use the username from step 1 in a normal add-users request:

```http
POST /api/organizations/{org_name}/resource-groups/{resource_group_id}/users
Content-Type: application/json
Body: { "users": [{ "user": "<username from step 1>", "role": "read" }] }
```

**Example: one email (bash)**

```bash
ORG_NAME="my-org"
RG_ID="507f1f77bcf86cd799439011"
EMAIL="member@org.com"

# Step 1: look up member by email (domain must match org's Organization email domain or SSO allowed domains)
MEMBERS=$(curl -s -H "Authorization: Bearer $HF_TOKEN" \
  "https://huggingface.co/api/organizations/$ORG_NAME/members?email=$EMAIL&limit=1")
USERNAME=$(echo "$MEMBERS" | jq -r '(.[0] // {} | .user // "")')
if [ -z "$USERNAME" ]; then
  echo "No member found for $EMAIL"
  exit 1
fi
# Step 2: add to resource group
curl -s -X POST -H "Authorization: Bearer $HF_TOKEN" -H "Content-Type: application/json" \
  -d "{\"users\":[{\"user\":\"$USERNAME\",\"role\":\"read\"}]}" \
  "https://huggingface.co/api/organizations/$ORG_NAME/resource-groups/$RG_ID/users"
```

**Example: multiple emails in a loop (Python)**

```python
import os
import requests

BASE = "https://huggingface.co"
ORG = "my-org"
RG_ID = "507f1f77bcf86cd799439011"
ROLE = "read"
headers = {"Authorization": f"Bearer {os.environ['HF_TOKEN']}", "Content-Type": "application/json"}

emails = ["member1@org.com", "member2@org.com"]
for email in emails:
    # Step 1: resolve email → username (email domain must match org's Organization email domain or SSO allowed domains)
    r = requests.get(f"{BASE}/api/organizations/{ORG}/members", params={"email": email, "limit": 1}, headers=headers)
    r.raise_for_status()
    members = r.json()
    if not members:
        print(f"No member found for {email}")
        continue
    username = members[0]["user"]
    # Step 2: add that user to the resource group
    add_r = requests.post(
        f"{BASE}/api/organizations/{ORG}/resource-groups/{RG_ID}/users",
        headers=headers,
        json={"users": [{"user": username, "role": ROLE}]},
    )
    if add_r.status_code == 200:
        print(f"Added {username} ({email})")
    else:
        print(f"Failed {email}: {add_r.status_code} {add_r.text}")
```

If a user is already in the resource group, the add call returns `403`; the script reports it as a failure and you can skip or ignore that case if you prefer.

**Limitation:** The email filter only applies when the org has an **Organization email domain** and/or **SSO allowed domains** set, and the email's domain matches one of them. Otherwise you cannot look up by email via the members API; you'd need another source for email → username (e.g. your own directory).

### Batch-add by looping over the API

You can add many users to **one** resource group in one or a few requests (e.g. chunk your list of usernames), or add users to **several** resource groups by looping over groups and calling the add-users endpoint for each.

**Example: Bash – one group, multiple users in one request**

```bash
#!/bin/bash
# Add a list of users to a single resource group.
# Usage: ./add-users-to-rg.sh <org_name> <resource_group_id> <role>

ORG_NAME="${1:-my-org}"
RG_ID="${2:-507f1f77bcf86cd799439011}"
ROLE="${3:-read}"

USERS="member1 member2 member3 member4"
USERS_JSON=$(echo "$USERS" | tr ' ' '\n' | while read u; do
  [ -n "$u" ] && echo "{\"user\":\"$u\",\"role\":\"$ROLE\"}"
done | paste -sd ',' -)

curl -s -w "\nHTTP_STATUS:%{http_code}" -X POST \
  -H "Authorization: Bearer $HF_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"users\":[$USERS_JSON]}" \
  "https://huggingface.co/api/organizations/$ORG_NAME/resource-groups/$RG_ID/users"
```

**Example: Bash – loop over multiple groups**

```bash
# Get group IDs and add users to each
curl -s -H "Authorization: Bearer $HF_TOKEN" \
  "https://huggingface.co/api/organizations/my-org/resource-groups" \
  | jq -r '.[].id' \
  | while read -r RG_ID; do
      [ -z "$RG_ID" ] && continue
      echo "Adding users to resource group $RG_ID ..."
      curl -s -X POST -H "Authorization: Bearer $HF_TOKEN" -H "Content-Type: application/json" \
        -d "{\"users\":[$USERS_JSON]}" \
        "https://huggingface.co/api/organizations/my-org/resource-groups/$RG_ID/users"
    done
```

**Example: Python – batch-add to one or many groups**

```python
import os
import requests

BASE_URL = "https://huggingface.co"
HF_TOKEN = os.environ.get("HF_TOKEN", "")

def list_resource_groups(org_name: str):
    r = requests.get(
        f"{BASE_URL}/api/organizations/{org_name}/resource-groups",
        headers={"Authorization": f"Bearer {HF_TOKEN}"},
    )
    r.raise_for_status()
    return r.json()

def add_users_to_resource_group(org_name: str, resource_group_id: str, users_with_roles: list):
    """users_with_roles: list of {"user": "username", "role": "read"|"write"|"contributor"|"admin"}"""
    r = requests.post(
        f"{BASE_URL}/api/organizations/{org_name}/resource-groups/{resource_group_id}/users",
        headers={"Authorization": f"Bearer {HF_TOKEN}", "Content-Type": "application/json"},
        json={"users": users_with_roles},
    )
    if r.status_code != 200:
        raise RuntimeError(f"Add users failed {r.status_code}: {r.text}")
    return r.json()

# Example: same users added to every resource group
org_name = "my-org"
role = "read"
usernames = ["member1", "member2", "member3"]
users_with_roles = [{"user": u, "role": role} for u in usernames]

for rg in list_resource_groups(org_name):
    add_users_to_resource_group(org_name, rg["id"], users_with_roles)
```

For a long list of usernames, chunk them (e.g. 50 per request) and call the API once per chunk to avoid large request bodies or timeouts.

### Important notes

1. **Usernames only** — The API accepts Hugging Face **usernames**, not emails. You need a mapping from email → username (e.g. from your directory or the org members list) before calling the API.
2. **Users must be in the organization** — Every user in the request must already be a member of the organization. Otherwise the request returns `403` with a message that some users are not in the org.
3. **Idempotency** — If a user is already in the resource group, the backend may return `403` for that request. Your script can catch errors and continue, or skip users already in the group if you first fetch the group's `users` list.
4. **Rate limits** — For large batches, consider adding a short delay between requests (e.g. 0.5–1 second) to avoid hitting rate limits.
5. **Token scope** — The access token must have sufficient permissions for the organization (typically at least Write). Create and store the token securely; do not commit it to version control.

### Summary

| Goal                                               | Approach                                                                                                                                      |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Add many users to **one** resource group           | One or more `POST .../users` requests with a `users` array (optionally chunk the list).                                                       |
| Add the **same** users to **many** resource groups | Loop over resource group IDs (from `GET .../resource-groups`), and for each ID call `POST .../resource-groups/{id}/users` with the same body. |
| Add **different** users per group                  | Loop over your mapping (e.g. group_id → list of usernames) and call `POST .../users` for each group with the corresponding list.              |
