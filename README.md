

**1.What is mTLS?**

**2.The certificates for mTLS?**

**3.Create OpenSSL Root CA**

**4.Generate the root CA Certificate**

**5.Generate the intermediate CA key pair and certificate**

**6.Generate and sign server certificate using Intermediate CA**

**7.Signing and Revoking a certificate using Intermediate CA**

**8.Conclusion**



**1.What is mTLS?**
============

mTLS is called mutual TLS or TLS mutual authentication.

TLS is a protocol for encryption used when communicating over a network. 

TLS consists of PKI (Public Key Infrastructure) and X.509 certificates. X.509 is a standard certificate format and is used in TLS/SSL, which is the backbone of https. It is used not only online but also offline for electronic signatures and other purposes. X.509 consists of a public key and several identities (host name, organizational information, etc.), and is signed by itself or by a certificate authority.

By default, TLS is only used by the client to verify the identity of the server, so the mechanism for the server to verify the client had to be implemented on the application side

Therefore, mTLS has come to be used as a mechanism for server and client to authenticate each other in business applications that have higher security requirements than consumer web services.

**2.The certificates for mTLS?**

Azure Gateway require chain certificate Root CA and Intermediate certificate.

Create root and intermediate certificates and then use these certificates to create certificate CA bundle in Linux.

Root certificate and Intermediate certificate

  * A certificate chain or certificate CA bundle is a sequence of certificates, where each certificate in the chain is signed by the subsequent certificate.
  * The Root CA is the top level of certificate chain while intermediate CAs or Sub CAs are Certificate Authorities that issue off an intermediate root.
  * Typically, the root CA does not sign server or client certificates directly.
  * The root CA is only ever used to create one or more intermediate CAs, which are trusted by the root CA to sign certificates on their behalf
  * It allows the root key to be kept offline and unused as much as possible, as any compromise of the root key is disastrous.
  * An intermediate certificate authority (CA) is an entity that can sign certificates on behalf of the root CA.
  * The root CA signs the intermediate certificate, forming a chain of trust.
  * The purpose of using an intermediate CA is primarily for security.
  * If the intermediate key is compromised, the root CA can revoke the intermediate certificate and create a new intermediate cryptographic pair.
  
**3.Create OpenSSL Root CA**

We will create a new directory structure /root/myCA/ to store our certificates.
~~~
mkdir -p ~/myCA/rootCA/{certs,crl,newcerts,private,csr}
mkdir -p ~/myCA/intermediateCA/{certs,crl,newcerts,private,csr}
~~~

A serial file is used to keep track of the last serial number that was used to issue a certificate. 

~~~
echo 1000 > ~/myCA/rootCA/serial
echo 1000 > ~/myCA/intermediateCA/serial
~~~

The CRL number is a unique integer that is incremented each time a new Certificate Revocation List (CRL) is generated. 

~~~
touch ~/myCA/rootCA/index.txt
touch ~/myCA/intermediateCA/index.txt
~~~

Create  openssl_root.cnf /etc/ssl folder

~~~
[ ca ]                                                   # The default CA section
default_ca = CA_default                                  # The default CA name

[ CA_default ]                                           # Default settings for the CA
dir               = /root/myCA/rootCA                    # CA directory
certs             = $dir/certs                           # Certificates directory
crl_dir           = $dir/crl                             # CRL directory
new_certs_dir     = $dir/newcerts                        # New certificates directory
database          = $dir/index.txt                       # Certificate index file
serial            = $dir/serial                          # Serial number file
RANDFILE          = $dir/private/.rand                   # Random number file
private_key       = $dir/private/ca.key.pem              # Root CA private key
certificate       = $dir/certs/ca.cert.pem               # Root CA certificate
crl               = $dir/crl/ca.crl.pem                  # Root CA CRL
crlnumber         = $dir/crlnumber                       # Root CA CRL number
crl_extensions    = crl_ext                              # CRL extensions
default_crl_days  = 30                                   # Default CRL validity days
default_md        = sha256                               # Default message digest
preserve          = no                                   # Preserve existing extensions
email_in_dn       = no                                   # Exclude email from the DN
name_opt          = ca_default                           # Formatting options for names
cert_opt          = ca_default                           # Certificate output options
policy            = policy_strict                        # Certificate policy
unique_subject    = no                                   # Allow multiple certs with the same DN

[ policy_strict ]                                        # Policy for stricter validation
countryName             = match                          # Must match the issuer's country
stateOrProvinceName     = match                          # Must match the issuer's state
organizationName        = match                          # Must match the issuer's organization
organizationalUnitName  = optional                       # Organizational unit is optional
commonName              = supplied                       # Must provide a common name
emailAddress            = optional                       # Email address is optional

[ req ]                                                  # Request settings
default_bits        = 2048                               # Default key size
distinguished_name  = req_distinguished_name             # Default DN template
string_mask         = utf8only                           # UTF-8 encoding
default_md          = sha256                             # Default message digest
prompt              = no                                 # Non-interactive mode

[ req_distinguished_name ]                               # Template for the DN in the CSR
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name (full name)
localityName                    = Locality Name (city)
0.organizationName              = Organization Name (company)
organizationalUnitName          = Organizational Unit Name (section)
commonName                      = Common Name (your domain)
emailAddress                    = Email Address

[ v3_ca ]                                           # Root CA certificate extensions
subjectKeyIdentifier = hash                         # Subject key identifier
authorityKeyIdentifier = keyid:always,issuer        # Authority key identifier
basicConstraints = critical, CA:true                # Basic constraints for a CA
keyUsage = critical, keyCertSign, cRLSign           # Key usage for a CA

[ crl_ext ]                                         # CRL extensions
authorityKeyIdentifier = keyid:always,issuer        # Authority key identifier

[ v3_intermediate_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

~~~

Create  openssl_intermediate.cnf   /etc/ssl folder

~~~
[ ca ]                           # The default CA section
default_ca = CA_default          # The default CA name

[ CA_default ]                                           # Default settings for the intermediate CA
dir               = /root/myCA/intermediateCA            # Intermediate CA directory
certs             = $dir/certs                           # Certificates directory
crl_dir           = $dir/crl                             # CRL directory
new_certs_dir     = $dir/newcerts                        # New certificates directory
database          = $dir/index.txt                       # Certificate index file
serial            = $dir/serial                          # Serial number file
RANDFILE          = $dir/private/.rand                   # Random number file
private_key       = $dir/private/intermediate.key.pem    # Intermediate CA private key
certificate       = $dir/certs/intermediate.cert.pem     # Intermediate CA certificate
crl               = $dir/crl/intermediate.crl.pem        # Intermediate CA CRL
crlnumber         = $dir/crlnumber                       # Intermediate CA CRL number
crl_extensions    = crl_ext                              # CRL extensions
default_crl_days  = 30                                   # Default CRL validity days
default_md        = sha256                               # Default message digest
preserve          = no                                   # Preserve existing extensions
email_in_dn       = no                                   # Exclude email from the DN
name_opt          = ca_default                           # Formatting options for names
cert_opt          = ca_default                           # Certificate output options
policy            = policy_loose                         # Certificate policy

[ policy_loose ]                                         # Policy for less strict validation
countryName             = optional                       # Country is optional
stateOrProvinceName     = optional                       # State or province is optional
localityName            = optional                       # Locality is optional
organizationName        = optional                       # Organization is optional
organizationalUnitName  = optional                       # Organizational unit is optional
commonName              = supplied                       # Must provide a common name
emailAddress            = optional                       # Email address is optional

[ req ]                                                  # Request settings
default_bits        = 2048                               # Default key size
distinguished_name  = req_distinguished_name             # Default DN template
string_mask         = utf8only                           # UTF-8 encoding
default_md          = sha256                             # Default message digest
x509_extensions     = v3_intermediate_ca                 # Extensions for intermediate CA certificate

[ req_distinguished_name ]                               # Template for the DN in the CSR
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

[ v3_intermediate_ca ]                                      # Intermediate CA certificate extensions
subjectKeyIdentifier = hash                                 # Subject key identifier
authorityKeyIdentifier = keyid:always,issuer                # Authority key identifier
basicConstraints = critical, CA:true, pathlen:0             # Basic constraints for a CA
keyUsage = critical, digitalSignature, cRLSign, keyCertSign # Key usage for a CA

[ crl_ext ]                                                 # CRL extensions
authorityKeyIdentifier=keyid:always                         # Authority key identifier

[ server_cert ]                                             # Server certificate extensions
basicConstraints = CA:FALSE                                 # Not a CA certificate
nsCertType = server                                         # Server certificate type
keyUsage = critical, digitalSignature, keyEncipherment      # Key usage for a server cert
extendedKeyUsage = serverAuth                               # Extended key usage for server authentication purposes (e.g., TLS/SSL servers).
authorityKeyIdentifier = keyid,issuer                       # Authority key identifier linking the certificate to the issuer's public key.
~~~

**4.Generate the root CA Certificate**

~~~
openssl genrsa -out ~/myCA/rootCA/private/ca.key.pem 4096
chmod 400 ~/myCA/rootCA/private/ca.key.pem
~~~

View the content of key

~~~
openssl rsa -noout -text -in ~/myCA/rootCA/private/ca.key.pem
~~~

Create Root Certificate Authority Certificate cacert.pem. 

~~~
openssl req -config openssl_root.cnf -key ~/myCA/rootCA/private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out ~/myCA/rootCA/certs/ca.cert.pem -subj "/C=US/ST=California/L=San Francisco/O=Example Corp/OU=IT Department/CN=Root CA"
~~~

The CA certificate can be world readable so that it can be used to sign the cert by anyone.

~~~
chmod 444 ~/myCA/rootCA/certs/ca.cert.pem
~~~

Verify root CA certificate

~~~
openssl x509 -noout -text -in ~/myCA/rootCA/certs/ca.cert.pem
~~~


**5.Generate the intermediate CA key pair and certificate**

Create an RSA key pair for the intermediate CA without a password and secure the file by removing permissions to groups and others

~~~
openssl genrsa -out ~/myCA/intermediateCA/private/intermediate.key.pem 4096
chmod 400 ~/myCA/intermediateCA/private/intermediate.key.pem
~~~

Create the intermediate CA certificate signing request CSR

~~~
openssl req -config openssl_intermediate.cnf -key ~/myCA/intermediateCA/private/intermediate.key.pem -new -sha256 -out ~/myCA/intermediateCA/certs/intermediate.csr.pem -subj "/C=US/ST=California/L=San Francisco/O=Example Corp/OU=IT Department/CN=Intermediate CA"
~~~

Sign the intermediate CSR with the root CA key

~~~
openssl ca -config openssl_root.cnf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in ~/myCA/intermediateCA/certs/intermediate.csr.pem -out ~/myCA/intermediateCA/certs/intermediate.cert.pem
~~~

Assign 444 permission to the cert to make it readable by everyone:

~~~
chmod 444 ~/myCA/intermediateCA/certs/intermediate.cert.pem
~~~

Check the index.txt file where the OpenSSL ca tool stores the certificate database

~~~
 cat ~/myCA/rootCA/index.txt
~~~

Verify the Intermediate CA Certificate content

~~~
openssl x509 -noout -text -in ~/myCA/intermediateCA/certs/intermediate.cert.pem
~~~

Verify intermediate certificate against the root certificate.

~~~
openssl verify -CAfile ~/myCA/rootCA/certs/ca.cert.pem ~/myCA/intermediateCA/certs/intermediate.cert.pem
~~~


Generate OpenSSL Create Certificate Chain (Certificate Bundle)

Create certificate chain (certificate bundle), concatenate the intermediate and root certificates together.
~~~
cat ~/myCA/intermediateCA/certs/intermediate.cert.pem ~/myCA/rootCA/certs/ca.cert.pem > ~/myCA/intermediateCA/certs/ca-chain.cert.pem
~~~

Verify certificate chain 

~~~
openssl verify -CAfile ~/myCA/intermediateCA/certs/ca-chain.cert.pem ~/myCA/intermediateCA/certs/intermediate.cert.pem
~~~

**6.Generate and sign server certificate using Intermediate CA**

~~~
openssl genpkey -algorithm RSA -out ~/myCA/intermediateCA/private/www.example.com.key.pem
chmod 400 ~/myCA/intermediateCA/private/www.example.com.key.pem
~~~

Create a certificate signing request (CSR) for the server

~~~
openssl req -config ~/myCA/openssl_intermediate.cnf -key ~/myCA/intermediateCA/private/www.example.com.key.pem -new -sha256 -out ~/myCA/intermediateCA/csr/www.example.com.csr.pem
~~~


You can automate the certificate signing request (CSR) creation by supplying default answers in the openssl.cnf (or openssl_intermediate.cnf in this case) file, under the [ req_distinguished_name ] section.

~~~
[ req_distinguished_name ]
countryName_default = US
stateOrProvinceName_default = California
localityName_default = San Francisco
0.organizationName_default = Example Corp
organizationalUnitName_default = IT Department
commonName_default = www.example.com
emailAddress_default = certs@example.com
~~~

Use the -batch option with openssl req command to automatically use these defaults without prompting for them

~~~
openssl req -config /etc/ssl/openssl_intermediate.cnf -key ~/myCA/intermediateCA/private/www.example.com.key.pem -new -sha256 -out ~/myCA/intermediateCA/csr/www.example.com.csr.pem -batch
~~~

Sign the server CSR with the intermediate CA.

~~~
openssl ca -config /etc/ssl/openssl_intermediate.cnf -extensions server_cert -days 375 -notext -md sha256 -in ~/myCA/intermediateCA/csr/www.example.com.csr.pem -out ~/myCA/intermediateCA/certs/www.example.com.cert.pem
~~~

Verify the server certificate.

~~~
openssl x509 -noout -text -in ~/myCA/intermediateCA/certs/www.example.com.cert.pem
~~~


**7.Signing and Revoking a certificate using Intermediate CA**


Revoking a certificate is the process of invalidating a previously issued SSL/TLS certificate before its expiration date. A certificate may need to be revoked for various reasons, such as:

   * The private key associated with the certificate has been compromised.
   * The certificate was issued fraudulently.
   * The information in the certificate has changed, and it no longer accurately represents the subject.


You have generated certificates under /certs folder and signed it using my intermediate CA.
~~~
cat ~/myCA/intermediateCA/index.txt
~~~

~~~
 cat ~/myCA/intermediateCA/serial
 ls -l /certs/
~~~


Revoke the certificate using the openssl ca command with the -revoke option

~~~
openssl ca -config ~/myCA/openssl_intermediate.cnf -revoke /certs/server.cert.pem
~~~

After revoking the certificate, update the Certificate Revocation List (CRL) to include the newly revoked certificate. The CRL is a list of revoked certificates that clients can check to determine if a certificate is still valid.

~~~
openssl ca -config ~/myCA/openssl_intermediate.cnf -gencrl -out ~/myCA/intermediateCA/crl/intermediate.crl.pem
~~~

The fields indicate the certificate's status (R for revoked), revocation date, expiration date, serial number, and subject information.

~~~
cat ~/myCA/intermediateCA/index.txt
~~~

The `crlnumber` file contentrepresents the current CRL number, which is incremented each time a new Certificate Revocation List is generated.

~~~
 cat intermediateCA/crlnumber
~~~

To get the list of revoked certificates from intermediate CA

~~~
openssl crl -in ~/myCA/intermediateCA/crl/intermediate.crl.pem -text -noout
~~~

**8.Conclusion**

In this guide, we walked through the process of creating a certificate chain using OpenSSL. The certificate chain is an essential component of Public Key Infrastructure (PKI), which allows secure communication between clients and servers over the internet. We covered the following topics.

    *  Setting up a directory structure for the Root and Intermediate Certificate Authorities (CAs).
    *  Creating configuration files (openssl.cnf) for the Root and Intermediate CAs.
    *  Generating a Root CA private key and self-signed certificate.
    *  Generating an Intermediate CA private key and Certificate Signing Request (CSR).
    *  Signing the Intermediate CA's CSR with the Root CA, resulting in the Intermediate CA certificate.
    *  Creating a Certificate Authority Bundle, which includes the Root and Intermediate CA certificates.
    *  Generating a server private key and CSR.
    *  Signing the server's CSR with the Intermediate CA, resulting in the server certificate.
    *  Revoking a server certificate, updating the Certificate Revocation List (CRL), and managing the CRL numbers.
    
By following these steps, you can establish a secure and trustworthy certificate chain for your web services, allowing clients to verify the authenticity of your server and communicate securely.

