Setup HashiCorp Vault with Integrated Storage (Raft) for POC / Development Purpose Only 
#######################################################################################

Prerequisite:
  1. Set server hostname
  2. Update OS packages
  3. Synchronize server time with NTP servers
  4. docker
  5. docker-compose


1. Create a Key and Certificate for vault:

## Create a directory ## 
$ mkdir -p certs 
$ cd certs 

## Generate vault key and  request for signing (CSR) ## 
$ openssl genrsa -out vault.key 4096

$ ls -l
total 4
-rw------- 1 root root 3272 Jan 29 16:00 vault.key

$ cat vault_cert.cnf

[req]
distinguished_name = req_distinguished_name
req_extensions = req_ext
prompt = no
  
[req_distinguished_name]
C   = IN
ST  = Delhi
L   = New Delhi
O   = DevOps Ninja Inc. 
OU  = devops
CN  = vault.devopsninja.lan
  
[req_ext]
subjectAltName = @alt_names
  
[alt_names]
IP.1 = 127.0.0.1
DNS.1 = vault.devopsninja.lan

$ openssl req -new -key vault.key -out vault.csr -config vault_cert.cnf

$ ls -l
total 12
-rw-r--r-- 1 root            root    1805 Jan 29 16:58 vault.csr
-rw------- 1 systemd-network student 3272 Jan 29 16:22 vault.key
-rw-r--r-- 1 root            root     313 Jan 29 16:56 vault_cert.cnf

## Verify Certificate Signing Request (CSR) ##

$ openssl req -noout -text -in vault.csr

## Sign vault.csr certificate with CA ##

$ openssl x509 -req -days 365 -in vault.csr -CA ../../../Setup_ROOT_CA/Internal_Root_CA/rootCA.pem -CAkey ../../../Setup_ROOT_CA/Internal_Root_CA/rootCAKey.pem -CAcreateserial -out vault.crt -passin file:../../../Setup_ROOT_CA/Internal_Root_CA/my_root_keypass.enc -extensions req_ext -extfile vault_cert.cnf

$ ls -l
total 16
-rw-r--r-- 1 root            root    1623 Jan 29 17:02 vault.crt
-rw-r--r-- 1 root            root    1805 Jan 29 16:58 vault.csr
-rw------- 1 systemd-network student 3272 Jan 29 16:22 vault.key
-rw-r--r-- 1 root            root     313 Jan 29 16:56 vault_cert.cnf

## Verify Server Certificate ##

$ openssl x509 -noout -text -in vault.crt 


2. Deploy vault

## Create the following directories: ## 

$ mkdir -p /opt/services/vault/{audit,config,data,file,logs,plugins,userconfig}  

$ ls -l /opt/services/vault/
total 28
drwxr-xr-x 2 root root 4096 Jan 29 16:19 audit
drwxr-xr-x 2 root root 4096 Jan 29 16:19 config
drwxr-xr-x 2 root root 4096 Jan 29 16:19 data
drwxr-xr-x 2 root root 4096 Jan 29 16:19 file
drwxr-xr-x 2 root root 4096 Jan 29 16:19 logs
drwxr-xr-x 2 root root 4096 Jan 29 16:19 plugins
drwxr-xr-x 2 root root 4096 Jan 29 16:19 userconfig

## Copy Vault server key and certificate to below directory: ##

$ mkdir -p /opt/services/vault/userconfig/tls
$ cp vault.crt /opt/services/vault/userconfig/tls/
$ cp vault.key /opt/services/vault/userconfig/tls/
$ cat ../../../Setup_ROOT_CA/Internal_Root_CA/rootCA.pem >> /opt/services/vault/userconfig/tls/vault.crt
$ cp ../../../Setup_ROOT_CA/Internal_Root_CA/rootCA.pem /opt/services/vault/userconfig/tls/ca.crt

## Configure CodeOpsPro Root CA as trusted CA on VM: ##

$ cp ../../../Setup_ROOT_CA/Internal_Root_CA/rootCA.pem /usr/local/share/ca-certificates/codeopspro.crt
$ update-ca-certificates


## Change Onwership / Groupownership of /opt/services/vault Directory ## 

$ chown -R 100:1000 /opt/services/vault/ 
 

## Create Vault Server config file ## 

$ cat /opt/services/vault/config/vault-config.hcl

disable_cache           = true
disable_mlock           = true
ui                      = true
  
listener "tcp" {
   address              = "0.0.0.0:8200"
   tls_disable          = false
   tls_client_ca_file   = "/vault/userconfig/tls/ca.crt"
   tls_cert_file        = "/vault/userconfig/tls/vault.crt"
   tls_key_file         = "/vault/userconfig/tls/vault.key"
}
  
storage "raft" {
  
   node_id              = "vault-1"
   path                 = "/vault/data"
  
   retry_join {
  
      leader_api_addr   = "https://vault.devopsninja.lan:8200"
   }
  
}
  
cluster_addr            = "https://vault.devopsninja.lan:8201"
api_addr                = "https://vault.devopsninja.lan:8200"
max_lease_ttl           = "2h"
default_lease_ttl       = "20m"
cluster_name            = "vault"
raw_storage_endpoint    = true
disable_printable_check = true  


## Create a docker-compose.yaml file ## 

$ cat /opt/services/docker-compose.yaml
version: '3.6'
services:
  vault:
    image: "hashicorp/vault:1.9.6"
    container_name: vault
    ports:
    - "8201:8201"
    - "8200:8200"
    environment:
      VAULT_ADDR: "https://vault.devopsninja.lan:8200"
      VAULT_CACERT: /vault/userconfig/tls/ca.crt
    cap_add:
    - IPC_LOCK
    volumes:
    - /opt/services/vault/audit:/vault/audit
    - /opt/services/vault/config:/vault/config
    - /opt/services/vault/data:/vault/data
    - /opt/services/vault/file:/vault/file
    - /opt/services/vault/logs:/vault/logs
    - /opt/services/vault/userconfig:/vault/userconfig
    - /opt/services/vault/plugins:/vault/plugins
    - /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.crt
    - /etc/hosts:/etc/hosts
    command: vault server -config=/vault/config/vault-config.hcl
    networks:
    - services_net

networks:
  services_net:
    driver: bridge  


## Start the Vault container ## 

$ cd /opt/service/
$ docker-compose up -d 
$ docker ps
CONTAINER ID   IMAGE                   COMMAND                  CREATED         STATUS         PORTS                                                           NAMES
dd8dce40c254   hashicorp/vault:1.9.6   "docker-entrypoint.s…"   4 seconds ago   Up 2 seconds   0.0.0.0:8200-8201->8200-8201/tcp, :::8200-8201->8200-8201/tcp   vault 

## Initialize vault with 5 key shares and a key threshold of 3 ## 

$ docker exec -it vault sh
$ vault operator init -key-shares=5 -key-threshold=3 | tee /tmp/vault-creds.txt 
$ docker cp vault:/tmp/vault-creds.txt /opt/services/vault-creds.txt


## Verify the status and unseal the vault ## 

$ docker exec -it vault sh

/ # vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    0/3
Unseal Nonce       n/a
Version            1.9.6
Storage Type       raft
HA Enabled         true

$ / # vault operator unseal  <Unseal Key 1>
$ / # vault operator unseal  <Unseal Key 2>
$ / # vault operator unseal  <Unseal Key 3>

$ / # vault status
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            5
Threshold               3
Version                 1.9.6
Storage Type            raft
Cluster Name            vault
Cluster ID              a4e4ba22-fa88-82ac-6704-c4a801ef27b0
HA Enabled              true
HA Cluster              https://vault.devopsninja.lan:8201
HA Mode                 active
Active Since            2023-01-29T13:19:13.905106781Z
Raft Committed Index    30
Raft Applied Index      30


## Open Web Browser and point to following URL to access Vault UI. ##

https://vault.devopsninja.lan:8200
