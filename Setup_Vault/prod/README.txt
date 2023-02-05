Setup HashiCorp Vault HA Cluster with Integrated Storage (Raft)
###############################################################

Prerequisite:
  1. Set server hostname
  2. Update OS packages
  3. Synchronize server time with NTP servers
  4. docker
  5. docker-compose


1. Create three (3) VMs for Auto-Unseal (transit) vault cluster:

VM_NAME         VM_IP               DNS_NAME
unseal-1        10.X.X.207        unseal-1.devopsninja.lan
unseal-2        10.X.X.208        unseal-2.devopsninja.lan
unseal-3        10.X.X.209        unseal-3.devopsninja.lan

Keepalived IP Address = 10.X.X.206 with DNS_NAME "unseal.devopsninja.lan"


2. Create three (3) VMs for Production vault cluster:

VM_NAME         VM_IP               DNS_NAME
vault-1         10.X.X.211        vault-1.devopsninja.lan
vault-2         10.X.X.212        vault-2.devopsninja.lan
vault-3         10.X.X.213        vault-3.devopsninja.lan

Keepalived IP Address = 10.X.X.210 with DNS_NAME "vault.devopsninja.lan"


3. Create a Key and Certificate for Auto-Unseal (transit) vault cluster:

CN: unseal.devopsninja.lan
SAN: unseal.devopsninja.lan, unseal-1.devopsninja.lan, unseal-2.devopsninja.lan, unseal-3.devopsninja.lan, localhost, 127.0.0.1

## Create a directory ## 

# mkdir -p certs/unseal
# cd certs/unseal/

## Generate unseal key and  request for signing (CSR) ## 

# openssl genrsa -out unseal.key 4096

# ls -l
total 4
-rw------- 1 root root 3272 Jan 29 16:00 unseal.key

# cat unseal_cert.cnf

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
CN  = unseal.devopsninja.lan
  
[req_ext]
subjectAltName = @alt_names
  
[alt_names]
IP.1 = 127.0.0.1
DNS.1 = unseal.devopsninja.lan
DNS.2 = unseal-1.devopsninja.lan
DNS.3 = unseal-2.devopsninja.lan
DNS.4 = unseal-3.devopsninja.lan
DNS.5 = localhost


# openssl req -new -key unseal.key -out unseal.csr -config unseal_cert.cnf

# ls -l
total 12
-rw-r--r-- 1 root            root    1805 Jan 29 16:58 unseal.csr
-rw------- 1 systemd-network student 3272 Jan 29 16:22 unseal.key
-rw-r--r-- 1 root            root     313 Jan 29 16:56 unseal_cert.cnf

## Verify Certificate Signing Request (CSR) ##

# openssl req -noout -text -in unseal.csr


## Sign unseal.csr certificate from CA ##

# openssl x509 -req -days 365 -in unseal.csr -CA ../../../../Setup_ROOT_CA/Internal_Root_CA/rootCA.pem -CAkey ../../../../Setup_ROOT_CA/Internal_Root_CA/rootCAKey.pem -CAcreateserial -out unseal.crt -passin file:../../../../Setup_ROOT_CA/Internal_Root_CA/my_root_keypass.enc -extensions req_ext -extfile unseal_cert.cnf

# ls -l
total 16
-rw-r--r-- 1 root            root    1623 Jan 29 17:02 unseal.crt
-rw-r--r-- 1 root            root    1805 Jan 29 16:58 unseal.csr
-rw------- 1 systemd-network student 3272 Jan 29 16:22 unseal.key
-rw-r--r-- 1 root            root     313 Jan 29 16:56 unseal_cert.cnf

## Verify Server Certificate ##

# openssl x509 -noout -text -in unseal.crt 


4. Create a Key and Certificate for vault cluster:

CN: vault.devopsninja.lan
SAN: vault.devopsninja.lan, vault-1.devopsninja.lan, vault-2.devopsninja.lan, vault-3.devopsninja.lan, vault.vault.svc, localhost, 127.0.0.1

## Generate vault key and  request for signing (CSR) ## 

# mkdir -p ../vault/
# cd ../vault/

# openssl genrsa -out vault.key 4096

# ls -l
total 4
-rw------- 1 root root 3272 Jan 29 16:00 vault.key

# cat vault_cert.cnf

[req]
distinguished_name = req_distinguished_name
req_extensions = req_ext
prompt = no
  
[req_distinguished_name]
C   = IN
ST  = Delhi
L   = New Delhi
O   = CodeOpsPro Academy
OU  = devops
CN  = vault.devopsninja.lan
  
[req_ext]
subjectAltName = @alt_names
  
[alt_names]
IP.1 = 127.0.0.1
DNS.1 = vault.devopsninja.lan
DNS.2 = vault-1.devopsninja.lan
DNS.3 = vault-2.devopsninja.lan
DNS.4 = vault-3.devopsninja.lan
DNS.5 = vault.vault.svc
DNS.6 = localhost

# openssl req -new -key vault.key -out vault.csr -config vault_cert.cnf

# ls -l
total 12
-rw-r--r-- 1 root            root    1805 Jan 29 16:58 vault.csr
-rw------- 1 systemd-network student 3272 Jan 29 16:22 vault.key
-rw-r--r-- 1 root            root     313 Jan 29 16:56 vault_cert.cnf

## Verify Certificate Signing Request (CSR) ##

# openssl req -noout -text -in vault.csr


## Sign vault.csr certificate from CA ##

# openssl x509 -req -days 365 -in vault.csr -CA ../../../../Setup_ROOT_CA/Internal_Root_CA/rootCA.pem -CAkey ../../../../Setup_ROOT_CA/Internal_Root_CA/rootCAKey.pem -CAcreateserial -out vault.crt -passin file:../../../../Setup_ROOT_CA/Internal_Root_CA/my_root_keypass.enc -extensions req_ext -extfile vault_cert.cnf

# ls -l
total 16
-rw-r--r-- 1 root            root    1623 Jan 29 17:02 vault.crt
-rw-r--r-- 1 root            root    1805 Jan 29 16:58 vault.csr
-rw------- 1 systemd-network student 3272 Jan 29 16:22 vault.key
-rw-r--r-- 1 root            root     313 Jan 29 16:56 vault_cert.cnf

## Verify Server Certificate ##

# openssl x509 -noout -text -in vault.crt 


5. Setup Auto-Unseal (transit) vault cluster

## Create Directory structure 

# mkdir -p /opt/services/unseal/{audit,config,data,file,logs,plugins,userconfig}

# ls -l /opt/services/unseal/
total 28
drwxr-xr-x 2 root root 4096 May 11 11:38 audit
drwxr-xr-x 2 root root 4096 May 11 11:38 config
drwxr-xr-x 2 root root 4096 May 11 11:38 data
drwxr-xr-x 2 root root 4096 May 11 11:38 file
drwxr-xr-x 2 root root 4096 May 11 11:38 logs
drwxr-xr-x 2 root root 4096 May 11 11:38 plugins
drwxr-xr-x 2 root root 4096 May 11 11:38 userconfig


## Copy Auto-Unseal (transit) Key, Certificate and CA Certificate 

# mkdir -p /opt/services/unseal/userconfig/tls

# cp ca.crt /opt/services/unseal/userconfig/tls/ca.crt
# cp unseal.crt /opt/services/unseal/userconfig/tls/unseal.crt
# cp unseal.key /opt/services/unseal/userconfig/tls/unseal.key

# chown -R 100:1000 /opt/services/unseal


## Create the below docker-compose.yaml file 

# cat /opt/services/docker-compose.yaml 

version: '3.6'
services:
  vault:
    image: "hashicorp/vault:1.9.6"
    container_name: unseal
    ports:
    - "8201:8201"
    - "8200:8200"
    environment:
      VAULT_ADDR: "https://unseal-1.devopsninja.lan:8200"           ## Change it as per the node ##
      VAULT_CACERT: /vault/userconfig/tls/ca.crt
    cap_add:
    - IPC_LOCK
    volumes:
    - /opt/services/unseal/audit:/vault/audit
    - /opt/services/unseal/config:/vault/config
    - /opt/services/unseal/data:/vault/data
    - /opt/services/unseal/file:/vault/file
    - /opt/services/unseal/logs:/vault/logs
    - /opt/services/unseal/userconfig:/vault/userconfig
    - /opt/services/unseal/plugins:/vault/plugins
    - /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.crt
    command: vault server -config=/vault/config/vault-config.hcl
    networks:
    - services_net

networks:
  services_net:
    driver: bridge


## Create the below config file  

# cat /opt/services/unseal/config/vault-config.hcl 

disable_cache           = true
disable_mlock           = true
ui                      = false

listener "tcp" {
   address              = "0.0.0.0:8200"
   tls_disable          = false
   tls_client_ca_file   = "/vault/userconfig/tls/ca.crt"
   tls_cert_file        = "/vault/userconfig/tls/unseal.crt"
   tls_key_file         = "/vault/userconfig/tls/unseal.key"
}

storage "raft" {

   node_id              = "unseal-1"                              ## Change it as per the node ##
   path                 = "/vault/data"

   retry_join {

      leader_api_addr   = "https://unseal-1.devopsninja.lan:8200"
   }

   retry_join {

      leader_api_addr   = "https://unseal-2.devopsninja.lan:8200"
   }

   retry_join {

      leader_api_addr   = "https://unseal-3.devopsninja.lan:8200"
   }

}

cluster_addr            = "https://unseal-1.devopsninja.lan:8201"         ## Change it as per the node ##
api_addr                = "https://unseal.devopsninja.lan:8200"
max_lease_ttl           = "2h"
default_lease_ttl       = "20m"
cluster_name            = "vault"
raw_storage_endpoint    = true
disable_printable_check = true


## Start Auto-Unseal (transit) vault instance 

# cd /opt/services/
# docker-compose pull

# docker-compose up -d

# docker ps 
CONTAINER ID   IMAGE                   COMMAND                  CREATED          STATUS          PORTS                              NAMES
759b7cf5e0f1   hashicorp/vault:1.9.6   "docker-entrypoint.s…"   11 seconds ago   Up 10 seconds   0.0.0.0:8200-8201->8200-8201/tcp   unseal



NOTE: Please repeat the above mentioned steps on all Auto-Unseal (transit) vault VMs  and change the configuration as per the given instructions. 


## Initialize the Auto-Unseal (transit) vault cluster with 5 key shares and a key threshold of 3 (Node 1 Only)

# docker exec -it unseal sh
# vault operator init -key-shares=5 -key-threshold=3 | tee /tmp/unseal-vault-creds.txt 
# docker cp unseal:/tmp/unseal-vault-creds.txt /opt/services/unseal-vault-creds.txt


## Verify the status and unseal the vault ## 

$ docker exec -it unseal sh

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
Total Shares            3
Threshold               2
Version                 1.9.6
Storage Type            raft
Cluster Name            vault
Cluster ID              2b6be943-4907-f056-0b8e-165d5f0c74fc
HA Enabled              true
HA Cluster              https://unseal-1.devopsninja.lan:8201
HA Mode                 standby
Active Node Address     https://unseal.devopsninja.lan:8200
Raft Committed Index    35
Raft Applied Index      35


## Verify the cluster status ## 

# vault operator raft list-peers                
Node        Address                      State       Voter
----        -------                      -----       -----
unseal-1    unseal-1.devopsninja.lan:8201    leader      true
unseal-2    unseal-2.devopsninja.lan:8201    follower    true
unseal-3    unseal-3.devopsninja.lan:8201    follower    true


# Install and Configure Keepalived on ALL Auto-Unseal (transit) vault  Nodes

# apt-get update && apt-get install keepalived -y

# cat /etc/keepalived/keepalived.conf 

global_defs {

	router_id LVS_LB
}

vrrp_script check_vault_health {

	script "/opt/services/vault-health.sh https://localhost:8200/v1/sys/health"
 	interval 3
}

vrrp_instance VI_1 {
	state MASTER
	virtual_router_id 100
	priority 100
	advert_int 1
	interface ens160      ## change it  ##

	virtual_ipaddress {
		# Floating IP
		10.X.X.206        ## change it  ##
	}

	track_interface {
		ens160              ## change it  ##
	}

	track_script {
		check_vault_health
	}
}



# cat /opt/services/vault-health.sh 

#!/bin/bash
if [ $# -ne 1 ];then
  echo "WARNING: You must provide health check URL"
  exit 1
else
  CHECK_URL=$1
  CMD=$(/usr/bin/curl -k -I ${CHECK_URL} 2>/dev/null | grep "200" | wc -l)

  if [ ${CMD} -eq 1 ];then
  exit 0
  else
  exit 1
  fi
fi


# chmod +x /opt/services/vault-health.sh 
# systemctl enable --now keepalived.service
# systemctl status keepalived.service

Check if the floating IP is assigned to a node.

# ip addr | grep "10.X.X.206" 


## Configure Auto-Unseal Key provider on Auto-Unseal (transit) vault cluster ## 


## Set the below ENV virtual_ipaddress

# export VAULT_ADDR=https://unseal.devopsninja.lan:8200
# export VAULT_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX


## Verify vault status 

# vault status 

Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            3
Threshold               2
Version                 1.9.6
Storage Type            raft
Cluster Name            vault
Cluster ID              2b6be943-4907-f056-0b8e-165d5f0c74fc
HA Enabled              true
HA Cluster              https://unseal-1.devopsninja.lan:8201
HA Mode                 active
Active Since            2022-05-17T07:57:21.005271796Z
Raft Committed Index    50
Raft Applied Index      50



## Enable an Audit device to capture and examine audit logs 

# vault audit enable file file_path=/vault/audit/audit.log 



## Enable transit secrets engine 

# vault secrets enable transit 



## Create an encryption key named "autounseal"

# vault write -f transit/keys/autounseal



## Create a policy file named "autounseal.hcl" which permits "update" against "transit/encrypt/autounseal" and "transit/decrypt/autounseal" paths. 

# cat autounseal.hcl 


path "transit/encrypt/autounseal" {
    capabilities = [ "update" ]
}

path "transit/decrypt/autounseal" {
    capabilities = [ "update" ]
}




## Create a policy named "autounseal"

# vault policy write autounseal autounseal.hcl
# vault policy list 

autounseal
default
root


6. Install and Configure Prod vault cluster with Auto_Unseal (transit) configuration  


root@vault-1:~# mkdir -p /opt/services/vault/{audit,config,data,file,logs,plugins,userconfig}

root@vault-1:~# ls -l /opt/services/vault/
total 28
drwxr-xr-x 2 root root 4096 May 11 11:38 audit
drwxr-xr-x 2 root root 4096 May 11 11:38 config
drwxr-xr-x 2 root root 4096 May 11 11:38 data
drwxr-xr-x 2 root root 4096 May 11 11:38 file
drwxr-xr-x 2 root root 4096 May 11 11:38 logs
drwxr-xr-x 2 root root 4096 May 11 11:38 plugins
drwxr-xr-x 2 root root 4096 May 11 11:38 userconfig


root@vault-1:~# mkdir -p /opt/services/vault/userconfig/tls

root@vault-1:~# cp ca.crt /opt/services/vault/userconfig/tls/ca.crt
root@vault-1:~# cp vault.crt /opt/services/vault/userconfig/tls/vault.crt
root@vault-1:~# cp vault.key /opt/services/vault/userconfig/tls/vault.key

root@vault-1:~# chown -R 100:1000 vault/


root@vault-1:~# cat /opt/services/docker-compose.yaml 

version: '3.6'
services:
  vault:
    image: "hashicorp/vault:1.9.6"
    container_name: vault
    ports:
    - "8201:8201"
    - "443:8200"
    environment:
      VAULT_ADDR: "https://vault-1.devopsninja.lan"           ## Change it as per the node ##
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
    command: vault server -config=/vault/config/vault-config.hcl
    networks:
    - services_net

networks:
  services_net:
    driver: bridge


root@vault-1:~# cat /opt/services/vault/config/vault-config.hcl 

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

   node_id              = "vault-1"                              ## Change it as per the node ##
   path                 = "/vault/data"

   retry_join {

      leader_api_addr   = "https://vault-1.devopsninja.lan"
   }

   retry_join {

      leader_api_addr   = "https://vault-2.devopsninja.lan"
   }

   retry_join {

      leader_api_addr   = "https://vault-3.devopsninja.lan"
   }

}

seal "transit" {
  address               = "https://unseal.devopsninja.lan:8200"
  token                 = "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"          # Generate the  token from unseal cluster and paste it here # 
  disable_renewal       = "false"
  key_name              = "autounseal"
  mount_path            = "transit/"
  tls_ca_cert           = "/vault/userconfig/tls/ca.crt"
  tls_client_cert       = "/vault/userconfig/tls/vault.crt"
  tls_client_key        = "/vault/userconfig/tls/vault.key"
  tls_server_name       = "unseal-ext.devopsninja.lan"
  tls_skip_verify       = "false"


}

cluster_addr            = "https://vault-1.devopsninja.lan:8201"         ## Change it as per the node ##
api_addr                = "https://vault.devopsninja.lan"
max_lease_ttl           = "2h"
default_lease_ttl       = "20m"
cluster_name            = "vault"
raw_storage_endpoint    = true
disable_printable_check = true



# docker-compose up -d

# docker ps 
CONTAINER ID   IMAGE                                        COMMAND                  CREATED          STATUS         PORTS                                           NAMES
0c76ec797dd8   hub.devlan.net/infra/hashicorp/vault:1.9.6   "docker-entrypoint.s…"   11 minutes ago   Up 6 minutes   0.0.0.0:8201->8201/tcp, 0.0.0.0:443->8200/tcp   vault


NOTE: On vault-2 and vault-3 nodes, Repeat the above mentioned steps. 


## Open another terminal and initialize the Production Vault cluster. Before initializing the cluster, make sure to check cluster status. ##

# export VAULT_ADDR=https://vault-1.devopsninja.lan

# vault status 

Key                      Value
---                      -----
Recovery Seal Type       transit              ## Notice here, it is "transit" Not "shamir"
Initialized              false
Sealed                   true
Total Recovery Shares    0
Threshold                0
Unseal Progress          0/0
Unseal Nonce             n/a
Version                  1.9.6
Storage Type             raft
HA Enabled               true



# vault operator init -recovery-shares=5 -recovery-threshold=3 

NOTE: the initialization generates recovery keys instead of unseal keys when using auto-unseal transit method. 


## Check the vault cluster status 


# vault status                                                                                                  

Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.9.6
Storage Type             raft
Cluster Name             vault
Cluster ID               d314defd-50f6-ad76-4ae9-91c35d8c79fd
HA Enabled               true
HA Cluster               https://vault-1.devopsninja.lan:8201
HA Mode                  active
Active Since             2022-05-17T11:04:39.568214936Z
Raft Committed Index     31
Raft Applied Index       31


NOTE: Notice that, the cluster "Initialized" is "true" and "Sealed" is "false". 


## Install and Configure Keepalived on ALL vault Nodes ##


# apt-get update && apt-get install keepalived -y

# cat /etc/keepalived/keepalived.conf 

global_defs {

	router_id LVS_LB
}

vrrp_script check_vault_health {

	script "/opt/services/vault-health.sh https://localhost/v1/sys/health"
 	interval 3
}

vrrp_instance VI_1 {
	state MASTER
	virtual_router_id 100
	priority 100
	advert_int 1
	interface ens160              ## Change here ##

	virtual_ipaddress {
		# Floating IP
		10.X.X.210                ## Change here ##
	}

	track_interface { 
		ens160                      ## Change here ##
	}

	track_script {
		check_vault_health
	}
}



# cat /opt/services/vault-health.sh 

#!/bin/bash
if [ $# -ne 1 ];then
  echo "WARNING: You must provide health check URL"
  exit 1
else
  CHECK_URL=$1
  CMD=$(/usr/bin/curl -k -I ${CHECK_URL} 2>/dev/null | grep "200" | wc -l)

  if [ ${CMD} -eq 1 ];then
  exit 0
  else
  exit 1
  fi
fi


# chmod +x /opt/services/vault-health.sh 
# systemctl enable --now keepalived.service
# systemctl status keepalived.service

Check if the floating IP is assigned to a node

# ip addr | grep "10.X.X.210" 


## Verify the Production vault cluster and node status: ##

# export VAULT_ADDR=https://vault.devopsninja.lan
# export VAULT_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# vault status                                        
Key                      Value
---                      -----
Recovery Seal Type       shamir
Initialized              true
Sealed                   false
Total Recovery Shares    5
Threshold                3
Version                  1.9.6
Storage Type             raft
Cluster Name             vault
Cluster ID               d314defd-50f6-ad76-4ae9-91c35d8c79fd
HA Enabled               true
HA Cluster               https://vault-1.devopsninja.lan:8201
HA Mode                  active
Active Since             2022-05-17T11:04:39.568214936Z
Raft Committed Index     37
Raft Applied Index       37



# vault operator raft list-peers                      
Node       Address                     State       Voter
----       -------                     -----       -----
vault-1    vault-1.devopsninja.lan:8201    leader      true
vault-2    vault-2.devopsninja.lan:8201    follower    true
vault-3    vault-3.devopsninja.lan:8201    follower    true




## To verify Auto-Unseal, restart all the Production vault instances and verify that they should come up with auto_unsealed

# docker restart vault 
