---
title: "Generate PostgreSQL credentials with Hashicorp Vault and Go"
date: "2022-04-16"
tags: [
    "go", "golang", "vault", "postgres", "security", "vault"
]
---


### TLDR

Handling databases credentials manually can be a tedious work and it can also lead to security issues. Let's check how we can generate dynamic username and password pairs for PostgreSQL using the secret manager Vault on a Go REST API, I made a [Github repo](https://github.com/yanpozka/exp/tree/main/pg-vault) if you want to jump straight into code.

<!--more-->

When we write boilerplate code to handle database connections we always have the same question or concern where to store the username/password. Most common options are just having a configuration file or passing creds as environment variables through a CI platform or jut deploy scripts, these solutions are just fine for many cases, but there's other option to not even bothering to deal with usernames and passwords: Vault.

What's Hashicorp Vault? according to the official [documentation](https://learn.hashicorp.com/tutorials/vault/getting-started-intro?in=vault/getting-started):
> Vault comes with various pluggable components called secrets engines and authentication methods allowing you to integrate with external systems. The purpose of those components is to manage and protect your secrets in dynamic infrastructure (e.g. database credentials, passwords, API keys).

In other words Vault is a beast in terms of security, so many cool possibilities and opportunities to improve the security of almost any kind of distributed system. Another benefits of using Vault for dynamically generate creds is that we can create users on demand with different access roles like a `readyonly` role for connecting to read/replication Postgres servers and roles with write access to certain tables only for master/leader Postgres nodes, which at the same time will reduce the surface attack in case of SQL injections.

So let's explore how we can write a simple Go API server that connects to Vault to retrieve temporary Postgres credentials each time we start or restart the server, process, pod or container.

First we have to have/setup a PostgreSQL database as well; also running and configure a Vault server that connects to Postgres, these preparatorial steps should be typically done by a devOps person.

For Postgres we'll use the official Docker image so simplicity, you could use a managed DB solution on the cloud or anything else of your preference.

This command will do the job:

```bash
# export your own $PG_DBNAME, $PG_USER and $PG_PASSWD environment variables

docker run --rm -d -p 5432:5432 \
  -e POSTGRES_DB=$PG_DBNAME \
  -e POSTGRES_USER=$PG_USER -e POSTGRES_PASSWORD=$PG_PASSWD \
  --name pg12 postgres:12-alpine
```

Setting up Vault is a little more tricky, first step will be running an actual vault server, again for the sake of simplicity we will run vault in development mode. Follow [these steps](https://learn.hashicorp.com/tutorials/vault/getting-started-install?in=vault/getting-started) to install it on your local machine.

Run the vault server in a terminal session:

```bash
vault server -dev

...
WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is already
authenticated to the CLI, so you can immediately begin using Vault.

You may need to set the following environment variable:

    $ export VAULT_ADDR='http://127.0.0.1:8200'

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: wZUWAuw9IXL0lAOOso2GepTE4bNIqrJFBfJ32Jp8GAk=
Root Token: hvs.oiHdlJQzmWISLEUQhiUdQYW9

Development mode should NOT be used in production installations!
...
```

as you can read development mode should not be used by any means on production, then export the vault address and root token environment variables (search for Root Token as shown above) and run the next commands on new terminal session (panel or tab) and enable database [dynamic secrets](https://learn.hashicorp.com/tutorials/vault/getting-started-dynamic-secrets?in=vault/getting-started):

```bash
# terminal 2
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN="..."

vault secrets enable database
```

now we have to configure the `postgresql-database-plugin` with the master credentials for your Postgres database (again this should be done by an operational person with admin access):

```bash
# export your own $PG_DBNAME, $ROLE_NAME, $PG_USER and $PG_PASSWD environment variables

vault write database/config/$PG_DBNAME \
  plugin_name=postgresql-database-plugin \
  allowed_roles="$ROLE_NAME" \
  connection_url="postgresql://{{username}}:{{password}}@0.0.0.0:5432/storedb?sslmode=disable" \
  username="$PG_USER" password="$PG_PASSWD"
```

this is the moment to write a little of SQL, in short we have to create a query to create the role in the database and grant the permissions that this role will be basically the future generated users, I used the same role names in the query and in Vault, we can also add time to life parameters for the roles/users so they are recycled after some time, remember that each time a server instance starts it grabs a new set of username/password:

```bash
sql_role="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
  GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"

# create role for db users
vault write database/roles/$ROLE_NAME \
  db_name=$PG_DBNAME \
  creation_statements="$sql_role" \
  default_ttl="6h" max_ttl="24h"
```

if everything has worked fine you'll be able to generate your first username/password from the console line, this step is optional:

```bash
vault read database/creds/$ROLE_NAME
```

At this point we already have a vault server connected to a Postgres database, so it's time to wear the developer hat and add some Go code.
Let's create then a client to vault, thankfully Vault provides a native implementation on the same github repository, that makes thing very straight forward, a [client](https://pkg.go.dev/github.com/hashicorp/vault/api#NewClient) can be easily created as shown bellow (remember to set the environment variables `VAULT_ADDR` and `VAULT_TOKEN`):

```go
import (
    "os"

    "github.com/hashicorp/vault/api"
)

config := &api.Config{
    Address: os.Getenv("VAULT_ADDR"),
}

var (
    client *api.Client
    err error
)
client, err = api.NewClient(config)
// check err

client.SetToken(os.Getenv("VAULT_TOKEN"))
```

once we have a client all we have to do is read the role credentials as a resource and this is all we got a brand new set of username/password ready to use in your favorite postgres client, the response from vault will come in a [api.Secret](https://pkg.go.dev/github.com/hashicorp/vault/api#Secret) instance (keep the same value for `ROLE_NAME` environment variable):

```go
var secret *api.Secret
secret, err = client.Logical().Read("database/creds/" + os.Getenv("ROLE_NAME"))
if err != nil {
    return
}

username = secret.Data["username"].(string)
passwd = secret.Data["password"].(string)
```

If you reached this point Congratulations! you know how to plumb Vault and get dynamic credentials for Postgres, also more great news Vault supports and impresive list of databases so with little changes you will able to use this same solution for many datastores, take a look at [all of them](https://www.vaultproject.io/docs/secrets/databases).

I plugged everything in a repo on Github if you want to see everything put together: <https://github.com/yanpozka/exp/tree/main/pg-vault>
