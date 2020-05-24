VAULT
=====

SetUP
-----

```
vault operator init
```

```
vault operator unseal
```

Unseal Vault with 3 unseal Key



Login
-----

```
vault login
```


Policies
-------

manager.hcl

```
path "/secret/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
}
```

```
vault policy write manager manager.hcl
```

```
vault policy list
```


Authentication
--------------

### Keycloak

```
vault auth enable \
        -path=keycloak \
        -listing-visibility="unauth" \
        oidc
```

```
vault write auth/keycloak/config \
        oidc_discovery_url="https://keycloak.example.com/auth/realms/example" \
        oidc_client_id="vault" \
        oidc_client_secret="XXXXXXXXXXXXXXXX" \
        default_role="manager" \
        type="oidc"
```

```
vault write auth/keycloak/role/manager \
        bound_audiences="vault" \
        allowed_redirect_uris="https://vault.example.com/ui/vault/auth/keycloak/oidc/callback,https://vault.example.com/keycloak/callback" \
        user_claim="sub" \
        policies="manager" \
        ttl=1h \
        role_type="oidc" \
        oidc_scopes="openid"
```

Secret Engines
--------------

### Database

#### MySQL

```
CREATE DATABASE vault_test;
CREATE USER 'vault-user'@'%' IDENTIFIED BY 'vault-password';
GRANT ALL PRIVILEGES ON * . * TO 'vault-user'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

```
CREATE USER 'vault_static'@'%' IDENTIFIED BY 'vault-password';
GRANT ALL PRIVILEGES ON * . * TO 'vault_static'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

```
vault secrets enable \
        -path=mysql \
        database
```

```
vault write mysql/config/mysql-production \
        plugin_name=mysql-database-plugin \
        connection_url="{{username}}:{{password}}@tcp(mysql:3306)/" \
        allowed_roles="*" \
        username="root" \
        password="vault-root"
```

##### Rotate Root
https://learn.hashicorp.com/vault/secrets-management/db-root-rotation

```
vault write -f mysql/rotate-root/mysql-production
```

##### Dynamic Roles

```
vault write mysql/roles/mysql-dynamic \
        db_name="mysql-production" \
        creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON vault_test.* TO '{{name}}'@'%';" \
        default_ttl="1h" \
        max_ttl="24h"
```

```
vault read mysql/creds/mysql-dynamic
```

Renew Leases
```
vault lease renew mysql/creds/vault_test/C46YFIbsp0yjUPgzl3SnoWdb
```

##### Static roles
https://learn.hashicorp.com/vault/secrets-management/db-creds-rotation

```
vault write mysql/static-roles/mysql-static \
        db_name="mysql-production" \
        username="vault_static" \
        rotation_statements="ALTER USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';" \
        rotation_period="1h"
```

```
vault write -f mysql/rotate-role/mysql-static
```

```
vault read mysql/static-creds/mysql-static
```

### SSH
https://www.vaultproject.io/docs/secrets/ssh/signed-ssh-certificates

```
vault secrets enable ssh
```

```
vault write ssh/config/ca generate_signing_key=true
```

```
curl -o /etc/ssh/trusted-user-ca-keys.pem http://vault.example.com/v1/ssh/public_key
```

```
vault read -field=public_key ssh/config/ca > /etc/ssh/trusted-user-ca-keys.pem
```

/etc/ssh/sshd_config
```
TrustedUserCAKeys /etc/ssh/trusted-user-ca-keys.pem
```

```
vault write ssh/roles/my-role -<<"EOH"
{
  "allow_user_certificates": true,
  "allowed_users": "*",
  "default_extensions": [
    {
      "permit-pty": ""
    }
  ],
  "key_type": "ca",
  "default_user": "saregune",
  "ttl": "30m0s"
}
EOH
```

```
vault write ssh/sign/my-role \
    public_key=@id_rsa.pub
```

```
vault write ssh/sign/my-role -<<"EOH"
{
  "public_key": "ssh-rsa AAA...",
  "valid_principals": "my-user",
  "key_id": "custom-prefix",
  "extension": {
    "permit-pty": ""
  }
}
EOH
```