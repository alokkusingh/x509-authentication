[![Build Status](https://travis-ci.org/alokkusingh/x509-authentication.svg?branch=master)](https://travis-ci.org/github/alokkusingh/x509-authentication)
[![GitHub issues](https://img.shields.io/github/issues/alokkusingh/x509-authentication.svg)](https://github.com/alokkusingh/x509-authentication/issues)
[![GitHub issues closed](https://img.shields.io/github/issues-closed-raw/alokkusingh/x509-authentication.svg?maxAge=2592000)](https://github.com/alokkusingh/x509-authentication/issues?q=is%3Aissue+is%3Aclosed)

# x509-authentication
Spring Security X.509 Certificate Based Authentication

Instead of Password based challenge, the server identifies client using their certificate.

Table of Contents
=================

   * [Certificate Generation and Usage](#certificate-generation-and-usage)
   * [Contents in Key Store and Truststore](#contents-in-key-store-and-truststore)
   * [TCP Dump and Analysis](#tcp-dump-and-analysis)

Created by [Alok Singh](https://github.com/alokkusingh)

## Certificate Generation and Usage

1) Root Certificate:

		openssl req -x509 -sha256 -days 3650 -newkey rsa:4096 -keyout rootCA_Alok.key -out rootCA_Alok.crt
		
		Pwd: changeit
		generates: 
			rootCA_Alok.key
			rootCA_Alok.crt

2) Server Side Certificate: - this will be used by Spring Boot Server
	
	2.1) Generate Server Side Certificate
	
		openssl req -new -newkey rsa:4096 -keyout localhost.key -out localhost.csr

		Country Name (2 letter code) []:IN
		State or Province Name (full name) []:KA
		Locality Name (eg, city) []:BLR
		Organization Name (eg, company) []:Home
		Organizational Unit Name (eg, section) []:Abc
		Common Name (eg, fully qualified host name) []:localhost
		Email Address []:alok.ku.singh@gmail.com

		Please enter the following 'extra' attributes
		to be sent with your certificate request
		A challenge password []:

		generates:
			localhost.key
			localhost.csr

	2.2) Sign Cert with Alok Root:
		
		vim localhost.ext
		authorityKeyIdentifier=keyid,issuer
		basicConstraints=CA:FALSE
		subjectAltName = @alt_names
		[alt_names]
		DNS.1 = localhost

		openssl x509 -req -CA rootCA_Alok.crt -CAkey rootCA_Alok.key -in localhost.csr -out localhost.crt -days 365 -CAcreateserial -extfile localhost.ext
		generates:
			rootCA_Alok.srl
			localhost.crt


	2.3) Import to Keystore:

		1st) Add the loaclhost.key and loaclhost.crt in single PKCS 12 bundle:
			
			openssl pkcs12 -export -out localhost.p12 -name "localhost" -inkey localhost.key -in localhost.crt
			Pwd: <password and remeber for furture use>
			generates:
				localhost.p12

		2nd) Import PKCS bundle to JKS
			
			keytool -importkeystore -srckeystore localhost.p12 -srcstoretype PKCS12 -destkeystore keystore.jks -deststoretype JKS
			Pwd: <password and remeber for furture use>
			generates:
				keystore.jks


3) Import Root CA cert to browser as Authority certificate - so that browser trust cert which is signed using this Root Cert (no risk warning will be shown in the browser)

		An exemplary installation of our certificate authority for Mozilla Firefox would look like follows:

		Type about:preferences in the address bar
		Open Advanced -> Certificates -> View Certificates -> Authorities
		Click on Import
		Select rootCA_Alok.crt file and click OK
		Choose “Trust this CA to identify websites” and click OK


4) Create Trust Store and Import Root CA cert which is used to sign Client Certificate - so that server trusts certificate signed using the same Root CA certificate:

		keytool -import -trustcacerts -noprompt -alias ca -ext san=dns:localhost,ip:127.0.0.1 -file rootCA_Alok.crt -keystore truststore.jks
		Pwd: <password and remeber for furture use>
		generates: truststore.jks


5) Cretate Client Certificate for Alok and Sign using RootCA_Alok so that server trusts:

	Server Says: I dont trust "Client Alok" (since Alok certificate is not addded in JKS) but I do trust "Root CA Alok" and he trusts you so do I.
	
	5.1) Generate Client Key and CSR
	
		openssl req -new -newkey rsa:4096 -nodes -keyout clientAlok.key -out clientAlok.csr
		Pwd: no password
		generates:
			clientAlok.key
			clientAlok.csr

	5.2) Sign cert for Alok with RootCA_Alok:
		
		openssl x509 -req -CA rootCA_Alok.crt -CAkey rootCA_Alok.key -in clientAlok.csr -out clientAlok.crt -days 365 -CAcreateserial
		generates: 
			clientAlok.crt

	5.3) Import cert to PKCS Bundle:
		
		openssl pkcs12 -export -out clientAlok.p12 -name "clientAlok" -inkey clientAlok.key -in clientAlok.crt
		Pwd: No password
		generates:
			clientAlok.p12

6) Import Alok Client Cert (clientAlok.p12) to browser so that when communicating to localhost this certificate will be sent for Authentication:
	

		Type about:preferences in the address bar
		Open Advanced -> View Certificates -> Your Certificates
		Click on Import
		Select clientAlok.p12 file and click OK
		Input the password for your certificate and click OK

7) Hit the Secure URL
	
	7.1) Using Firefox
	
		https://localhost:8443/user
		
		It will promt to select one of the installed client certificate in the Browser
		
	7.2) Hit the API URL using CURL
	
		curl --cacert rootCA_Alok.crt --key clientAlok.key --cert clientAlok.crt https://localhost:8443/api/user
		
		Where:
			cacert: Root CA Cert who signed server certificate (substitute of step 3)
			key: Client Key (substitute of step 6)
			cert: Client Certificate (substitute of step 6)
			
		Note: for this step you may skip steps - 3, 5.3, and 6 (above)

## Contents in Key Store and Truststore

1) Key Store
        
	- localhost.key
	- localhost.crt

2) Trust Store
        
	- rootCA_Alok.crt

## TCP Dump and Analysis

Find the dump file under dump/ folder. You may use Wiresark to read the dump file.

1) TCP Dump Command
````
sudo tcpdump -i lo0 -n -s0 -w /Users/aloksingh/logs/x509App_04.cap port 8443
````

Assuming lo0 is loopback interface.

2) Dump Analysis

    ![alt text](https://github.com/alokkusingh/x509-authentication/blob/master/dump/dump.png?raw=true "TCP Packets")
    - [C <-> S] First 4 packets is for TCP handshake
    - [C <-> S] 5 and 6 Client Hello and ACK from server
    - [C <-- S] 7 Server Hello along with 
                    Server Certificate, 
                    Server Key Exchange, 
                    Certificate Request (Mandatory for Mutual Authentication) 
    - [C --> S] 8 ACK 
    - [C --> S] 9 Client Certificate (Mandatory for Mutual Authentication) along with 
                    Client Key Exchange (for RSA, a 48-byte pre_master_secret (also known as session key) is generated by the client and encrypted using server public key, which can be decrypted only by private key server has), 
                    Certificate Verify, 
                    Change Cipher Spec (from now onwards to use symmetric key for encryption/decryption)
    - [C <-- S] 10 ACK
    - [C <-- S] 11 Change Cipher Spec (from now onwards to use symmetric key encryption/decryption - shared recently) 
    - [C --> S] 12 ACK
    - [C <-- S] 13 Encrypted Handshake Message (Finished) - indicates TLS negotiations completed
    - [C --> S] 14 ACK
    - [C --> S] 15 GET request from Client
    - [C <-- S] 16 ACK 
    - [C <-- S] 17 GET response from Server
    - [C --> S] 18 ACK
    - [C --> S] 19 Encrypted Alert 
    - [C <-- S] 20 ACK
    - [C --> S] 21 FIN
    - [C <-- S] 22 ACK
    
