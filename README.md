# Explore/Demo Concourse + Hashicorp Vault

This demo was developed with these versions of VirtualBox, docker-compose, and docker-machine on OsX 10.11:

```
$ docker-compose version
docker-compose version 1.14.0, build c7bdf9e
docker-py version: 2.3.0
CPython version: 2.7.12
OpenSSL version: OpenSSL 1.0.2j  26 Sep 2016

$ docker-machine version
docker-machine version 0.12.0, build 45c69ad

$ VBoxManage --version
5.1.12r112440
```

## 1. Set up vault and a secret

Set up keys and start vault:

```
make vault
docker-compose up vault
```

In another window set up Vault:

```
DOCKER_IP=$(echo $DOCKER_HOST | perl -ne 'm=tcp://([\d\.]+):= && print "$1"')
export VAULT_ADDR=https://$DOCKER_IP:8200
export VAULT_SKIP_VERIFY=true
vault init -check  # should return 'Vault is not initialized'
vault init -key-shares=1 -key-threshold=1 | tee -a keys/init_output |
  awk 'BEGIN{OFS=""} /Unseal/ {print "export VAULT_UNSEAL_KEY=",$4};/Root/ {print "export VAULT_ROOT_TOKEN=",$4}' > keys/init_vars
```

Then unseal the vault and authorize as 'root', and write a secret:

```
source keys/init_vars
vault unseal $VAULT_UNSEAL_KEY
vault auth $VAULT_ROOT_TOKEN
vault write secret/concourse/main/helloworld/from-vault value="a value from vault" 
```

### Set up AppRole

see https://www.vaultproject.io/docs/auth/approle.html. These TTLs below are all way too high:

```
vault auth-enable approle
vault write auth/approle/role/concourse_role secret_id_ttl=60m token_num_uses=60 token_ttl=120m token_max_ttl=300m secret_id_num_uses=400

# Update policy to allow role to read concourse secrets:

vault policy-write concourse concourse_policy.hcl

# Associate that policy with app_roll

curl -X POST -H "X-Vault-Token:$VAULT_ROOT_TOKEN" -d '{"policies":"default,concourse"}' ${VAULT_ADDR}/v1/auth/approle/role/concourse_role -k
```

Get the roleid and set a secret id

```
export CONCOURSE_ROLE_ID=$(
    vault read -format=json auth/approle/role/concourse_role/role-id | jq -r '.data.role_id')

export CONCOURSE_SECRET_ID=$(
    vault write -format=json -force auth/approle/role/concourse_role/secret-id | jq -r '.data.secret_id')
echo $CONCOURSE_ROLE_ID
echo $CONCOURSE_SECRET_ID
```


### Demonstrate getting using short-lived tokens to fetch desired key/value

```
export VAULT_TOKEN=$(
  vault write -format=json auth/approle/login role_id=$CONCOURSE_ROLE_ID secret_id=$CONCOURSE_SECRET_ID| jq -r '.auth.client_token')
curl -k -X GET -H "X-Vault-Token:$VAULT_TOKEN" $VAULT_ADDR/v1/secret/concourse/main/helloworld/from-vault -vs
```

### Kill `docker-compose` in other window

## Run concourse and vault:

Make sure the env vars are set for `CONCOURSE_ROLE_ID` and `CONCOURSE_SECRET_ID`

Generate the keys for concourse web and workers, then bring up docker-compose

```
make concourse
docker-compose up
```

While that's coming up, unseal the vault:

```
vault unseal $VAULT_UNSEAL_KEY
```

## Set up pipeline

```
fly -t docker login # concourse:changeme
fly -t docker set-pipeline -c ci/pipeline.yml -l ci/credentials.yml -p helloworld
fly -t docker unpause-pipeline -p helloworld
```
Now watch:

```
fly -t docker watch -j helloworld/job-hello-world
```


To Fix:
- vault.path_prefix not vault_prefix
- writing secrets to '/concourse' fails, using 'secret/concourse'
