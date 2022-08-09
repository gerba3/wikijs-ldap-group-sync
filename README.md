# WikiJS LDAP Group Sync

WikiJS currently lacks the ability to sync groups from authentication providers to WikiJS.
They plan to provide a feature like that [in its 3.0 release](https://js.wiki/feedback/p/group-mapping), until then this Python script can help.

## Architecture

This script relies on the WikiJS GraphQL server to get group membership information which unfortunately doesn't provide ways to list users
together with their ids and group memberships.
Due to that this script has to send as many requests as there are users times the amount of groups.

Roughly speaking the script does the following:

1. Connect to LDAP
2. Retrieve user information, group names and group membership from LDAP
3. Retrieve user information (id, email) and group information (id, name) from WikiJS
4. Add group to WikiJS if group doesn't already exist
5. Compare data and map group ownership to users
6. Tell WikiJS to assign groups to users

Because we have no way of knowing which user is assigned to which group without making individual requests to the WikiJS GraphQL API (or batching them) the majority of runtime is going to be spent on the last step.

## Installing

### Docker

```bash
docker build -t wikijs-ldap-group-sync .
docker run wikijs-ldap-group-sync
```

### Poetry

After cloning the project and installing poetry simply execute `poetry run sync` to run the script.

## Configuration

By default, WikiJS LDAP Group Sync uses environment variables however you can also provide a `.env` file in the main execution directory.

| Environment Variable | Example                                                                                       | Meaning                                     |
|----------------------|-----------------------------------------------------------------------------------------------|---------------------------------------------|
| WIKIJS_URL           | "https://wiki.example.com/graphql"                                                            | URL of the WikiJS GraphQL endpoint          |
| WIKIJS_TOKEN         | "Bearer rVcMvDUsAAv4abPxcIEg"                                                                 | Used for authenticating to GraphQL [^1]     |
| LDAP_URL             | "ldaps://ldap.example.com:636"                                                                | URL of the LDAP Server [^2]                 |
| ADMIN_BIND_DN        | "UID=admin,CN=Users,DC=example,DC=com"                                                        | DN used for authenticating to LDAP          |
| ADMIN_BIND_CRED      | "supersecretpassword"                                                                         | Password used for authenticating to LDAP    |
| GROUPS_SEARCH_BASE   | "CN=Groups,DC=example,DC=com"                                                                 | LDAP base groups will be searched for under |
| GROUPS_SEARCH_FILTER | "(objectClass=posixGroup)"                                                                    | LDAP group search filter                    |
| USER_SEARCH_BASE     | "CN=Users,DC=example,DC=com"                                                                  | LDAP base users will be searched for under  |
| USER_SEARCH_FILTER   | "(&(objectClass=organizationalPerson)(memberof=CN=Domain Users,CN=Groups,DC=example,DC=com))" | LDAP user search filter                     |

[^1]: Authentication tokens can be generated in the WikiJS Admin Panel under "API Access" 

[^2]: Note that there is currently no way of providing a certificate
