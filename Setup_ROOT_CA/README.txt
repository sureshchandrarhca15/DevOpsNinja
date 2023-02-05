## Generate root CA key and certificate ##


1. Create the root CA directory:

$ mkdir -p Internal_Root_CA
$ cd Internal_Root_CA

2. Generate the private key of the root CA:

## Create a file to store root CA key password ##

$ echo "My_Secret_Password" > my_root_keypass.enc

## Generate Private root CA Key using the above created password file ##

$ openssl genrsa -des3 -passout file:my_root_keypass.enc -out ca.key 4096

## Verify the key has created ## 
	
$ ls -l 
total 8
-rw-r--r-- 1 root root   19 Jan 29 15:05 my_root_keypass.enc
-rw------- 1 root root 1704 Jan 29 15:03 rootCAKey.pem

## Create Certificate Authority Certificate with 10 Years Validity ##

$ openssl req -new -x509 -days 3650 -key rootCAKey.pem -out rootCA.pem -passin file:my_root_keypass.enc
	
Country Name (2 letter code) [AU]:IN
State or Province Name (full name) [Some-State]:Delhi
Locality Name (eg, city) []:New Delhi
Organization Name (eg, company) [Internet Widgits Pty Ltd]:DevOps Ninja Inc.
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:DevOps Ninja Internal Root CA
Email Address []: 


## Verify CA certificate ## 

$ ls -l
total 12
-rw-r--r-- 1 root root   19 Jan 29 15:05 my_root_keypass.enc
-rw-r--r-- 1 root root 1399 Jan 29 15:16 rootCA.pem
-rw------- 1 root root 1704 Jan 29 15:03 rootCAKey.pem

$ openssl x509 -noout -text -in rootCA.pem

