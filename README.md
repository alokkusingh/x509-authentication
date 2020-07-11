# x509-authentication
Spring Security X.509 Based Authentication

Root Certificate:
	openssl req -x509 -sha256 -days 3650 -newkey rsa:4096 -keyout rootCA_Alok.key -out rootCA_Alok.crt
		Pwd: changeit
	generates: 
		rootCA_Alok.key
		rootCA_Alok.crt

Server Side Certificate: - this will be used by Spring Boot Server
	openssl req -new -newkey rsa:4096 -keyout localhost.key -out localhost.csr                                                            SIGINT(2) ↵  869  14:38:00
		Generating a 4096 bit RSA private key
		............................................................................................................................................................................................................................................++
		.................................................................................................++
		writing new private key to 'localhost.key'
		Enter PEM pass phrase: <password and remeber for furture use>
		Verifying - Enter PEM pass phrase: <password and remeber for furture use>
		-----
		You are about to be asked to enter information that will be incorporated
		into your certificate request.
		What you are about to enter is what is called a Distinguished Name or a DN.
		There are quite a few fields but you can leave some blank
		For some fields there will be a default value,
		If you enter '.', the field will be left blank.
		-----
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

	Sign Cert with Alok Root:
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


		Import to Keystore:

		1st) add the loaclhost.key and loaclhost.crt in single PKCS 12 bundle:
			openssl pkcs12 -export -out localhost.p12 -name "localhost" -inkey localhost.key -in localhost.crt
			Pwd: <password and remeber for furture use>
			generates:
				localhost.p12

		2nd) Import PKCS bundle to JKS
			keytool -importkeystore -srckeystore localhost.p12 -srcstoretype PKCS12 -destkeystore keystore.jks -deststoretype JKS
			Pwd: <password and remeber for furture use>
			generates:
				keystore.jks


Import Root CA cert to browser as Authority certificate- so that browser trust cert which is signed using this Root Cert (no risk warning will be shown in the browser)

	An exemplary installation of our certificate authority for Mozilla Firefox would look like follows:

	Type about:preferences in the address bar
	Open Advanced -> Certificates -> View Certificates -> Authorities
	Click on Import
	Locate the Baeldung tutorials folder and its subfolder spring-security-x509/keystore
	Select the rootCA.crt file and click OK
	Choose “Trust this CA to identify websites” and click OK


Create Trust Store and Import Root CA cert which is used to sign Client Certificate - so that server trust certificate signed using the same Root CA certificate:
	keytool -import -trustcacerts -noprompt -alias ca -ext san=dns:localhost,ip:127.0.0.1 -file rootCA_Alok.crt -keystore truststore.jks
	Pwd: <password and remeber for furture use>
	generates: truststore.jks


Cretate Client Certificate for Alok and Sign using RootCA_Alok so that server trusts:
	openssl req -new -newkey rsa:4096 -nodes -keyout clientAlok.key -out clientAlok.csr
	Pwd: no password
	generates:
		clientAlok.key
		clientAlok.csr

	Sign cert for Alok with RootCA_Alok:
		openssl x509 -req -CA rootCA_Alok.crt -CAkey rootCA_Alok.key -in clientAlok.csr -out clientAlok.crt -days 365 -CAcreateserial
		generates: 
			clientAlok.crt

	Import cert to PKCS Bundle:
		openssl pkcs12 -export -out clientAlok.p12 -name "clientAlok" -inkey clientAlok.key -in clientAlok.crt
		Pwd: No password
		generates:
			clientAlok.p12

Import Alok Client Cert (clientAlok.p12) to browser so that when communicating to localhost this certificate will be sent for Authentication:
	Again, we'll use Firefox:

	Type about:preferences in the address bar
	Open Advanced -> View Certificates -> Your Certificates
	Click on Import
	Locate the Baeldung tutorials folder and its subfolder spring-security-x509/store
	Select the clientBob.p12 file and click OK
	Input the password for your certificate and click OK
