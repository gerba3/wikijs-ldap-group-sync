# WikiJS LDAP Group Sync

WikiJS currently lacks the ability to sync groups from authentication providers to WikiJS.
They plan to provide a feature like that [in its 3.0 release](https://js.wiki/feedback/p/group-mapping), until then this Python script can help.

WikiJS version tested: 2.5.284

## Architecture

This script relies on the WikiJS GraphQL server to get group membership information which unfortunately has no way of getting a list of users
together with their internal ids and group memberships.
Due to that this script has to send as many requests as there are users times the amount of groups so depending on your network the runtime may vary.

Roughly speaking the script does the following:

1. Connect to LDAP
2. Retrieve user information, group names and group membership from LDAP
3. Retrieve user information (id, email) and group information (id, name) from WikiJS
4. Add group to WikiJS if group doesn't already exist
5. Compare data and map group ownership to users
6. Tell WikiJS to assign groups to users

Because we have no way of knowing which user is assigned to which group without making individual requests to the WikiJS GraphQL API (or batching them) the majority of runtime is going to be spent on the last step.

The script itself is stateless, all logging goes to stdout.

## Installing

### Docker

Pull from GitHub Container registry:
```bash
docker pull ghcr.io/macbrayne/wikijs-ldap-group-sync
```
Pull from Docker Hub:
```bash
docker pull macbrayne/wikijs-ldap-group-sync
 ```

Alternatively, you can build it yourself:
```bash
docker build -t macbrayne/wikijs-ldap-group-sync 'https://github.com/macbrayne/wikijs-ldap-group-sync.git#main'
```

### Poetry

Using this script requires the use of the [poetry Python
package](https://python-poetry.org/) packaging utility. On Debian like systems
it can be installed with the following command:

```bash
apt-get install --no-install-recommends python3-poetry build-essential libsasl2-dev libldap2-dev
```

Before running the sync, you need to install the dependencies using poetry,
which will also automatically create a virtual environment:

```bash
poetry install
```

Then, after installing poetry simply execute `poetry run sync` to run the script.

## Configuration

By default, WikiJS LDAP Group Sync uses environment variables however you can also provide a `.env` file in the main execution directory.

| Environment Variable             | Example                                                                                       | Meaning                                                                  |
|----------------------------------|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| WIKIJS_URL                       | "https://wiki.example.com/graphql"                                                            | URL of the WikiJS GraphQL endpoint                                       |
| WIKIJS_TOKEN                     | "Bearer rVcMvDUsAAv4abPxcIEg"                                                                 | Used for authenticating to GraphQL [^1]                                  |
| LDAP_URL                         | "ldaps://ldap.example.com:636"                                                                | URL of the LDAP Server [^2]                                              |
| (optional) ADMIN_BIND_DN         | "UID=admin,CN=Users,DC=example,DC=com"                                                        | DN used for authenticating to LDAP, leave empty for anonymous bind       |
| (optional) ADMIN_BIND_CRED       | "supersecretpassword"                                                                         | Password used for authenticating to LDAP, leave empty for anonymous bind |
| GROUPS_SEARCH_BASE               | "CN=Groups,DC=example,DC=com"                                                                 | LDAP base groups will be searched for under                              |
| GROUPS_SEARCH_FILTER             | "(objectClass=posixGroup)"                                                                    | LDAP group search filter                                                 |
| USER_SEARCH_BASE                 | "CN=Users,DC=example,DC=com"                                                                  | LDAP base users will be searched for under                               |
| USER_SEARCH_FILTER               | "(&(objectClass=organizationalPerson)(memberof=CN=Domain Users,CN=Groups,DC=example,DC=com))" | LDAP user search filter                                                  |
| (optional) LOG_LEVEL             | (default) "INFO"                                                                              | Log level to be used [^3]                                                |
| (optional) LDAP_TLS_VERIFICATION | any value                                                                                     | If this value is set the certificate will be verified [^4]               |
| (untested) LDAP_TLS_CERT_FILE    | "/etc/ldap/cacert/cacert.pem"                                                                 | Path to a certificate [^4]                                               |


[^1]: Authentication tokens can be generated in the WikiJS Admin Panel under "API Access"

[^2]: Note that you have to also set "LDAP_TLS_VERIFICATION" if you want the certificate to be validated

[^3]: Server down at "ERROR", group creation at "WARNING", general program flow at "INFO", the results of group assignments at "DEBUG"

[^4]: Note that a failure when verifying the certificate won't output any more information than "Can't contact LDAP server"
