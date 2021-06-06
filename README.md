

# vault summary

![Screenshot 2021-05-22 at 9 38 02 PM](https://user-images.githubusercontent.com/5676192/119230383-851c4e00-bb4e-11eb-8a13-3ec3f9b27379.png)

# Installation

```
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

### Check version
- `vault version `

### Check vault status
- `vault status`

# Start server
```
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=vault_token
vault server -dev
```


# Authentication
## Login 
- `vault login`
- `vault login -address="http://127.0.0.1:8200" $root_token`
- `vault login -method=userpass username=admin`

## Logout
- remove `~/.vault_token`

## Revoke a token
- `vault token revoke s.iyNUhq8Ov4hIAx6snw5mB2nL`
- `vault token revoke -mode path auth/github`

## Create token
- `vault token create`

## List Auth
- `vault auth list`

## Revoke Auth method
- `vault auth disable github`

## Tune auth description
- `vault auth tune -description="Globomantics Userpass" userpass/`

## Userpass 
```
vault auth enable userpass #enable
vault write auth/userpass/users/ned password=tacos policies="test" # create user
vault write auth/userpass/users/adam role_name="webapp" secret_id_num_uses=1 secret_id_ttl=2h # create user
```

## AppRole
```
# Read-only permission on 'secret/data/mysql/*' path
path "secret/data/mysql/*" {
  capabilities = [ "read", "update" ]
}
```
```
vault auth enable approle # enable
vault policy write jenkins jenkins-pol.hcl
vault write auth/approle/role/jenkins token_policies="jenkins" token_ttl=1h token_max_ttl=4h
vault read auth/approle/role/jenkins # Read to verify
vault read auth/approle/role/jenkins/role-id # Get role ID
vault write -f auth/approle/role/jenkins/secret-id # Get secret ID
vault write auth/approle/login role_id="ROLE_ID" secret_id="SECRET_ID" # Login using AppRole

```

## Wrap a response
- `vault write -wrap-ttl=60s -f auth/approle/role/jenkins/secret-id`

## Unwrap response
- `VAULT_TOKEN=s.2kAzCgg1kN7vdpE1xxZxzpug vault unwrap`

# Vault Tokens
- Use limit tokens: Tokens that are only good to invoke a specific number of operations. `vault token create -ttl=1h -use-limit=2 -policy=default`
- Periodic service tokens: Tokens that can be renewed indefinitely. `vault token create -policy="default" -period=24h`
- Short-lived tokens: Tokens that are valid for a short time to avoid keeping unused tokens. `vault token create -ttl=60s`
- Orphan tokens: Tokens that are root of their own token tree. `vault token create -orphan`
- Batch tokens: not persisted, lightweight and scalable token. They are encrypted binary large objects (blobs) that carry enough information for them to be used for Vault actions. `vault token create -type=batch -policy=test -ttl=20m`
- refer to [Service Tokens vs. Batch Tokens](https://learn.hashicorp.com/tutorials/vault/batch-tokens?in=vault/tokens)

### Create a token role named zabbix
```
vault write auth/token/roles/zabbix \
    allowed_policies="policy1, policy2, policy3" \
    orphan=true \
    period=8h
vault token create -role=zabbix
```

### Renew token
```
vault token renew $TOKEN
vault token renew -increment=60 $TOKEN # Renew and extend the token's TTL to 60 seconds
```

### Lookup a token

```
vault token lookup $TOKEN
```

# Secret Engines
## Vault Key/Value v2 secrets engine
```
Subcommands:
    disable    Disable a secret engine
    enable     Enable a secrets engine
    list       List enabled secrets engines
    move       Move a secrets engine to a new path
    tune       Tune a secrets engine configuration
```


### Enable KV secrets engine
- By default, Vault enables Key/Value version2 secrets engine (kv-v2) at the path secret/ when running in dev mode.
- `vault secrets enable -path=kv kv`
- `vault secrets enable kv`
- `vault secrets enable -version=2 kv`

### List of secret engines
- `vault secrets list`
- `vault secrets list -detailed`

### Write / Update a secret
- `vault kv put secret/key1 some=42`
- `vault kv put secret/hello foo=world excited=yes`
- `vault kv get -format=json secret/hello | jq -r .data.data.excited`
- `vault kv get -version=1 secret/hello`

### Get a secret
- `vault kv get secret/key1`
- `vault kv get -field=excited secret/hello`

### Soft delete a secret
- `vault kv delete secret/key1`

### Undelete a secret
- `vault kv undelete -versions=2 secret/key1`

### Destroy a secret
- `vault kv destroy -versions=1,2 secret/key1`

### Remove all data about secrets
- `vault kv metadata delete secret/key1`

### Disable a Secrets Engine
- `vault secrets disable kv/`

### Move the secrets engine path
- `vault secrets move dev prod`

### Upgrade the secrets engine to v2
- `vault kv enable-versioning dev-secrets`

## Vault Database Secret Engine

### Enable the database secrets engine
- `vault secrets enable database`

### Configure MySQL roles and permissions
```
mysql -u root -p
CREATE ROLE 'dev-role';
CREATE USER 'vault'@'<YourPublicIP>' IDENTIFIED BY 'AsYcUdOP426i';
CREATE DATABASE devdb;
GRANT ALL ON *.* TO 'vault'@'<YourPublicIP>';
GRANT GRANT OPTION ON devdb.* TO 'vault'@'<YourPublicIP>';
```

### Configure the MySQL plugin
```
vault write database/config/dev-mysql-database \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(MY_SQL_IP:3306)/" \
    allowed_roles="dev-role" \
    username="vault" \
    password="password"
``` 

### Configure a role to be used
```
vault write database/roles/dev-role \
    db_name=dev-mysql-database \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT ALL ON devdb.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
```

### Generate credentials on the DB from the role
- `vault read database/creds/dev-role`

### Renew the lease
- `vault lease renew -increment=3600 database/creds/dev-role/LEASE_ID`

### Revoke the lease
- `vault lease revoke database/creds/dev-role/LEASE_ID`

## Vault Dynamic Secret Engine

### Enable AWS secrets engine
- `vault secrets enable -path=aws aws`

### Configure AWS credentials
- option 1
```
export AWS_ACCESS_KEY_ID=<aws_access_key_id>
export AWS_SECRET_ACCESS_KEY=<aws_secret_key>
```
- option 2
```
vault write aws/config/root \
    access_key=$AWS_ACCESS_KEY_ID \
    secret_key=$AWS_SECRET_ACCESS_KEY \
    region=us-east-1
```

### Create AWS IAM Role
```
vault write aws/roles/my-role \
        credential_type=iam_user \
        policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1426528957000",
      "Effect": "Allow",
      "Action": [
        "ec2:*"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
EOF
```

### Generate IAM user with credentials and role
- `vault read aws/creds/my-role`


# Vault Policies
```
# Policy example
# Manage secrets engines
path "sys/mounts/*" {
  capabilities = ["create", "read", "update", "delete", "list", "sudo"]
}

# List existing secrets engines.
path "sys/mounts" {
  capabilities = ["read"]
}

# Deny access to privileged financial data
path "financial/data/apitokens/privleged*" {
  capabilities = ["deny"]
}

path "financial/metadata/apitokens/privileged*" {
  capabilities = ["deny"]
}
```
## List policy
`vault policy list `

## Create Policy
```
vault policy write secrets-mgmt secrets-mgmt.hcl
```

## Read policy
```
vault policy read secrets-mgmt
```

## Update policy
```
vault policy write accounting accounting-revised.hcl
```

## Delete poicy
```
vault policy delete accounting
```















