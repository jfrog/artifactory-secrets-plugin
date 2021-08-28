----------------------------------------------------------------
This plugin is now being actively maintained by JFrog Inc. Please refer to [CONTRIBUTING.md](CONTRIBUTING.md) for contributions.
-----------------------------------------------------------------


This is not yet ready for production use. Please file issues though as you find them.

![Build](https://github.com/idcmp/artifactory-secrets-plugin/workflows/Build/badge.svg)

# Vault Artifactory Secrets Plugin

This is a [HashiCorp Vault](https://www.vaultproject.io/) plugin which talks to JFrog Artifactory server (5.0.0 or later) and will
dynamically provision access tokens with specified scopes. This backend can be mounted multiple times
to provide access to multiple Artifactory servers.

Using this plugin, you can limit the accidental exposure window of Artifactory tokens; useful for continuous integration servers.

## Access Token Creation and Revoking

This backend creates access tokens in Artifactory using the admin credentials provided. Note that if you
provide non-admin credentials, then the "username" must match the username of the credential owner.

Ideally this plugin would create "expiring access tokens" in Artifactory, however expiring access tokens
cannot be revoked early (see RTFACT-15293).

When the lease on the Artifactory access token ends, this plugin will ask Artifactory to revoke the token. Note
that as reported in RTFACT-22519, Artifactory will continue to honour the revoked access token for a few minutes.

I've also filed RTFACT-22477, proposing CIDR restrictions on the created access tokens.

## What's Missing

* I'm still spelunking through the Vault code base to determine who is responsible for enforcing TTLs. I _think_ I have
it correct, but I can't stay 100% yet.

* I'd like to rotate the admin/config access_token when it's configured (if it's refreshable).

## Testing Locally

If you're compiling this yourself and want to do a local sanity test, you
can do something like:

```bash
terminal-1$ make
...

terminal-2$ export VAULT_ADDR=http://127.0.0.1:8200
terminal-2$ export VAULT_TOKEN=root
terminal-2$ make setup
...

terminal-2$ make artifactory &  # Runs netcat returning a static JSON response
terminal-2$ vault read artifactory/token/test
```

## Usage

### Artifactory

This has been tested with Artifactory 6.18.1. If you have a newer/older version, please report (positive/negative) outcomes in Issues.

You will need the "admin" user's password (not an admin, but admin specifically).

1. Log into the Artifactory UI as "admin".
1. Under "Welcome, admin" (top right) go to "Edit Profile".
1. Unlock your user profile and get your API Key. Save your API Key as an environment variable `KEY`.

You will now create the Access Token that Vault will use to interact with Artifactory. In Artifactory 7.4+ this can be done in the UI (Service: Artifactory, Expiry: Never Expires), otherwise use the REST API:

```
curl -XPOST -u admin:$KEY "https://artifactory.example.org/artifactory/api/security/token" \
    -dusername=admin \
    -dexpires_in=0 \
    "-dscope=member-of-groups:*"
```

Note that "username" must be "admin" otherwise you will not be able to specify different usernames for roles. Save the "access_token" from the JSON response as the environment variable `TOKEN`.

### Vault

To actually integrate it into Vault:

```bash
$ vault secrets enable artifactory

# Should be able to use "tune" ( https://www.vaultproject.io/docs/commands/secrets/tune )
$ vault write artifactory/config/admin \
               url=https://artifactory.example.org/artifactory \
               access_token=$TOKEN

# Also supports grant_type=, and audience= (see JFrog documentation)
$ vault write artifactory/roles/jenkins \
               username="example-service-jenkins" \
               scope="api:* member-of-groups:ci-server" \
               default_ttl=1h max_ttl=3h 

$ vault list artifactory/roles
Keys
----
jenkins

$ vault read artifactory/token/jenkins 
Key                Value
---                -----
lease_id           artifactory/token/jenkins/25jYH8DjUU548323zPWiSakh
lease_duration     1h
lease_renewable    true
access_token       adsdgbtybbeeyh...
role               jenkins
scope              api:* member-of-groups:ci-server
```

