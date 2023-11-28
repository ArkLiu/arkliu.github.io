---
title: Streamline User and Group Management with Databricks Notebook
date: 2023-11-27
tags:
  - databricks
  - identity management
---

# Streamline User and Group Management with Databricks Notebook

## Intro

`What is the best practice for manage identities in Databricks?` is one of the
most common questions a new Databricsk customer will ask the Databricks
implementation team. The answer is always `It depends` as it depends on the
size of your organization, the number of users, the number of groups ...etc.
If you keep pushing for a more specific answer, most likely the answer is
`Using Databricks UI` for small organizations and
`Using Terraform or SCIM provisioning` for large organizations.

Unfortunately, none of the above answers is satisfying. The first one is not
scalable and the second one is not flexible due to the nature of our workflow.
I want to write this article to share my experience on this topic.

## The Problem

In order to understand the solution, we need to understand the problem first.

Why not `Using Databricks UI` should be very self-explanatory. It is not
scalable and have a lot of room for human error. It makes sense for using UI
for eloxporation and testing but not for production.

Why not `Using Terraform or SCIM provisioning`? In my opinion, Terraform is for
 infrastructure as code not for identity management. It is possible to use it
 for the this purpose but it will end up with many data resource objects which
 make the infrastructure code hard to understand and maintain.

`SCIM provisioning` is a better option than Terraform as it is designed for
identity management. This is also the solution We plan to do in the very first
place. However, We did not go with this option in the very beginning because of
the `Communication Overhead`. In our workflow, the group creation and its
permission is done by Terraform. Using SCIM provisioning mean let identity
provider to take the driver seat which might result in the group configuration
from Terraform got overwritten. In addition, our team does not have the
visibility on the identity provider configuration which make it hard to debug
when something goes wrong.

I think SCIM provisioning can be a good solution when everything is stable and a
 dedicated team is responsible for the identity management. But we are not
 there yet so we need to find a better solution.

## The Solution

After a few weeks of research, I found a better way to manage identities in
Databricks. In a nutshell, the solution is to use Databricks Notebook to manage
 the user and group. The notebook will read the YAML file which contains the
 user and group information and calling Databricks API to add or remove the
 user from the group. The benefit of this solution is that

- ensure the uptime due to no dependency on other tool
- high flexibility and scalability
- reset the permission to the desired state

### Requirements

Nothing is required per se. However, I do have some pro tips for you if you
consider adopting this solution.

- a repository to store the notebook and YAML file.
- a secret management tool to store the senstive token.
- a CI/CD tool to trigger the notebook when the YAML file is updated.
- a service principal to run the notebook.

### YAML

The YAML file contains the user and group information.

```yaml
team_1:
  members:
    - user1@email.com
    - user2@email.com
team_2:
  members:
    - user1@email.com
    - user2@email.com
```

### Databricks Notebooks

This is the `pseudocode` of what the notebook will do and I will go over each
step in detail.

```text
import all the required libraries
get the token from somewhere safe
get Databricks API token
call users API to get all the users information
call groups API to get all the group information
read the YAML file
loop through group information to add/remove users
```

#### Import the required libraries

This part should be very self-explanatory.

```python
import json

import requests
import yaml
```

#### get the token from somewhere safe

I am using Databricks secret management to store the token. The reason is to
reduce the dependency on other tools. However, you can use any secret
management tool you like.

```python
git_token = dbutils.secrets.get(  # noqa: F821
    scope="github",
    key="github_token",
)

databricks_sp_client_id = dbutils.secrets.get(  # noqa: F821
    scope="databricks_sp",
    key="client_id",
)
databricks_sp_client_secret = dbutils.secrets.get(  # noqa: F821
    scope="databricks_sp",
    key="client_secret",
)
```

#### get Databricks API token

I am using service principal token to handles all API calls. This can prevent
the workflow not working due to the end user is no longer in the organization.

For other authentication methods, you can find more information on
[Authentication for Databricks automation - overview][].

```python
response = session.post(
    url=f"{DATABRICKS_OAUTH_BASE_URL}/{DATABRICKS_ACCOUNT_ID}/v1/token",
    auth=(databricks_sp_client_id, databricks_sp_client_secret),
    data={
        "grant_type": "client_credentials",
        "scope": "all-apis",
    },
    timeout=10,
)

if response.status_code == 200:
    # Success!
    access_token = response.json()["access_token"]
else:
    # Something went wrong
    print(response.status_code)
    print(response.content)
```

#### call users API to get all the users information

Generate a dictionary with email as key and user id as value so we can use it
to translate the YAML file.

```python
response = session.get(
    url=f"{DATABRICKS_ACCOUNT_LEVEL_BASE_URL}/{DATABRICKS_ACCOUNT_ID}/scim/v2/Users",
    headers={"Authorization": f"Bearer {access_token}"},
    timeout=10,
)

active_users_dict = {
    row["emails"][0]["value"]: row["id"]
    for row in response.json()["Resources"]
    if row["active"]
}
```

#### call groups API to get all the group information

Similar to the prior block. The dictionary key is the group name and the value
is the group id and the members id.

```python
response = session.get(
    url=f"{DATABRICKS_ACCOUNT_LEVEL_BASE_URL}/{DATABRICKS_ACCOUNT_ID}/scim/v2/Groups",
    headers={"Authorization": f"Bearer {access_token}"},
    timeout=10,
)

groups_dict = {}

for row in response.json()["Resources"]:
    groups_dict[row["displayName"]] = {
        "id": row["id"],
        "members": {user["value"] for user in row.get("members", [])},
    }
```

#### read the YAML file

As I mentioned before, it is highly recommended to store the YAML file in a
repository. This will make the workflow more transparent and easier to share
the provisioning responsibility with other teams as the end user does not need
Databricks access to update the YAML file.

After the YAML file is read, we need to convert the email to user id so we can
communicate with Databricks API.

```python
res = session.get(
    url=(
        f"https://api.github.com/repos/{user_name}/{repo_name}/contents/"
        f"{the_file_path}/config.yml"
    ),
    headers={
        "Authorization": f"Bearer {git_token}",
        "Accept": "application/vnd.github.v3.raw",
    },
    timeout=10,
)

if res.status_code == 200:
    config = yaml.safe_load(res.content.decode("utf-8"))

for _, value in config.items():
    value["members"] = (
        {
            active_users_dict[member]
            for member in value["members"]
            if member in active_users_dict
        }
        if value["members"]
        else set()
    )
```

#### loop through group information to add/remove users

A basic `set` operation to find the delta between the group members from
Databricks API and the YAML file. Then call the PATCH API to update it.

```python
for key, value in groups_dict.items():
    if key in config:
        group_name = key
        group_id = value["id"]
        members_from_databricks_api = value["members"]
        members_from_yaml = config[key].get("members", set())

        print(f"Check {group_name}")
        operations = []
        if len(members_from_databricks_api - members_from_yaml):
            operations.append(
                {
                    "op": "remove",
                    "path": "members",
                    "value": [
                        {"value": member}
                        for member in members_from_databricks_api - members_from_yaml
                    ],
                }
            )
        if len(members_from_yaml - members_from_databricks_api):
            operations.append(
                {
                    "op": "add",
                    "path": "members",
                    "value": [
                        {"value": member}
                        for member in members_from_yaml - members_from_databricks_api
                    ],
                }
            )

        if operations:
            print(f"[{group_name}] delta found: {operations}")
        if operations:
            url = (
                f"{DATABRICKS_ACCOUNT_LEVEL_BASE_URL}/{DATABRICKS_ACCOUNT_ID}"
                f"/scim/v2/Groups/{group_id}"
            )
            payload = {
                "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
                "Operations": operations,
            }

            response = session.patch(
                url,
                data=json.dumps(payload),
                headers={"Authorization": f"Bearer {access_token}"},
                timeout=10,
            )

            if response.status_code == 200:
                print(f"[{group_name}] provision is finished.")
            else:
                # Something went wrong
                print(response.status_code)
                print(response.content)
```

## The Conclusion

With this solution, my team is able to spend less time on the Databricks
administration work. Even with the new group been created to support the new
project, we can provision the new group to the end user who need it because we
also wire this job run to the CI/CD pipeline to fully automate the process.

As you can see, this solution is built on top of the Databricks API and python
basic operation to make it very flexible and scalable. You can easily modify
the logic to fit your need.

I hope you find this article helpful. If you have any questions or suggestions,
 please feel free to reach out to me

[Authentication for Databricks automation - overview]: https://docs.databricks.com/dev-tools/api/latest/authentication.html#authentication-for-databricks-automation-overview
