# Self signed SSL certificate for use with FileMaker 16 Server and other services such as MTA, web; such as SOAP, REST, GraphQL WIP

Work in progress based on
1) https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html
1) https://blog.beezwax.net/2017/12/03/creating-your-own-ssl-certificates-for-filemaker/ 
1) http://kb.mit.edu/confluence/display/istcontrib/FileMaker+Server+SSL+Certificates

The reason for signing your own SSL cert
1) Security

The reason for having your SSL cert publicly signed
1) Authenticate who you are

### If you are one of the people thinking it's cheaper to buy a publicly signed certificate
Chances are you do not understand security; Please read below and find out more.

# Precaution

## If you need a publicly signed SSL certificate
In order to prove that you are you, there are several options, my preferred option is to run this shell based Let's encrypt approach: https://github.com/lukas2511/dehydrated works fine from LaunchDaemon, Manifold, systemd, or crontab

## If you aim for security of your data follow this procedure
1) Disconnect computer from internet
1) Insert SD card to computer
1) Follow procedure below in full
1) Copy only the required files from SD card to your computer( you do not want private keys copied from the SD card unless you are required to use them )
1) Sober house rules below
1) Eject SD card and put in safe
1) Reconnect your computer to the internet

### Add a host cert
1) Disconnect computer from internet
1) Insert SD card to computer
1) Follow procedure below only the last part: Sign server and client certificates
1) Copy only the required files from SD card to your computer
1) Sober house rules below
1) Eject SD card and put in safe
1) Reconnect your computer to the internet

## Sober house rules
Sober house rules dictates copying the SD card to at least 1 more SD Card and make sure you make a new backup every 3 years after your last write; better off every 12 months. But your PMS( Planned Maintenance System ) is your PMS.

### For FileMaker Pro users

```
exa -T /Applications/FileMaker\ Pro\ 17\ Advanced/FileMaker\ Pro\ Advanced.app/Contents/Frameworks/Support.framework/Versions/A/Resources/OpenSSL/RootCA/
/Applications/FileMaker Pro 17 Advanced/FileMaker Pro Advanced.app/Contents/Frameworks/Support.framework/Versions/A/Resources/OpenSSL/RootCA
└── sn07samba01.domain.tld.CA.pem
```
I used sn07samba01, you can put ca or nothing in front of the domain, whatever you find less confusing to come back to.

### This guide is written in a way intended to be string replacable

`:%s/mymedia/nameOfYourSDCard`

# Extracting the CSR from FileMaker 16 Server

```
$ fmsadmin certificate create "/CN=fm16s00.domain.tld/O=Organization/C=US/ST=State" --keyfilepass <secret>
$ fmsadmin certificate create "/CN=fm16s01.domain.tld/O=Organization/C=US/ST=State" --keyfilepass <secret>
$ fmsadmin certificate create "/CN=fm16s02.domain.tld/O=Organization/C=US/ST=State" --keyfilepass <secret>
$ fmsadmin certificate create "/CN=fm16s03.domain.tld/O=Organization/C=US/ST=State" --keyfilepass <secret>

$ exa -T /Library/FileMaker\ Server/CStore/
/Library/FileMaker Server/CStore/
├── serverKey.pem 
├── serverRequest.pem
└── ...
```

# Create CA
## Make sure you edit openssl.conf to match your `/Volumes/mymedia/ca`
```
cd /Volumes/mymedia

mkdir ca
cd ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial

curl -kLo openssl.conf https://jamielinux.com/docs/openssl-certificate-authority/_downloads/root-config.txt
vim openssl.conf
```

You want to edit the openssl.conf file to match your setup, CommonName is the host name in your domain, regardless of wether that host exists or not. By example in table below:

| Common Name | Use case     |
|-------------|--------------|
|sn07samba01  | CA           |
|ca           | CA           |
|sn04mta05    | Intermediate |
|mail2        | server       |
|www09        | server       |
|fm16s00      | server       |
|fm16s01      | server       |
|fm16s02      | server       |
|fm16s03      | server       |

## Create the root key
```
openssl genrsa -aes256 -out private/ca.key.pem 4096
chmod 400 private/ca.key.pem
```

## Create the root certificate
```
openssl req -config openssl.conf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem
chmod 444 certs/ca.cert.pem
```

### Verify the root certificate
```
openssl x509 -noout -text -in certs/ca.cert.pem
```


# Create Intermediate
## Make sure you edit openssl.conf to match your `/Volumes/mymedia/ca/intermediate`
```
cd /Volumes/mymedia/ca
mkdir intermediate
cd intermediate
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
echo 1000 > crlnumber

curl -kLo openssl.conf https://jamielinux.com/docs/openssl-certificate-authority/_downloads/intermediate-config.txt
vim openssl.conf

openssl genrsa -aes256 -out private/intermediate.key.pem 4096
chmod 400 private/intermediate.key.pem
```

# Create the intermediate certificate
```
openssl req -config openssl.conf -new -sha256 -key private/intermediate.key.pem -out csr/intermediate.csr.pem

cd ..

openssl ca -config openssl.conf -extensions v3_intermediate_ca -days 7300 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem
chmod 444 intermediate/certs/intermediate.cert.pem
```

Note I had to edit the openssl.conf for the root as follows: `commonName optional` to make this work.

### Verify the intermediate certificate
```
openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
openssl verify -CAfile certs/ca.cert.pem intermediate/certs/intermediate.cert.pem
```

## Create the certificate chain file
```
cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```



# Sign server and client certificates
```
cd /Volumes/mymedia/ca
openssl genrsa -aes256 -out intermediate/private/www.example.com.key.pem 2048
chmod 400 intermediate/private/www.example.com.key.pem
```
As a general example
```
openssl req -config intermediate/openssl.conf -key intermediate/private/www.example.com.key.pem -new -sha256 -out intermediate/csr/www.example.com.csr.pem
openssl ca -config intermediate/openssl.conf -extensions server_cert -days 7300 -notext -md sha256 -in intermediate/csr/www.example.com.csr.pem -out intermediate/certs/www.example.com.cert.pem
```
What you really want to do for the case discussed above
```
openssl ca -config intermediate/openssl.conf -extensions server_cert -days 7300 -notext -md sha256 -in fm16s00/serverRequest.pem -out intermediate/certs/tld.domain.fm16s00.cert.pem
openssl ca -config intermediate/openssl.conf -extensions server_cert -days 7300 -notext -md sha256 -in fm16s00/serverRequest.pem -out intermediate/certs/tld.domain.fm16s01.cert.pem
openssl ca -config intermediate/openssl.conf -extensions server_cert -days 7300 -notext -md sha256 -in fm16s00/serverRequest.pem -out intermediate/certs/tld.domain.fm16s02.cert.pem
openssl ca -config intermediate/openssl.conf -extensions server_cert -days 7300 -notext -md sha256 -in fm16s00/serverRequest.pem -out intermediate/certs/tld.domain.fm16s03.cert.pem

chmod 444 intermediate/certs/tld.domain.fm16s00.cert.pem
chmod 444 intermediate/certs/tld.domain.fm16s01.cert.pem
chmod 444 intermediate/certs/tld.domain.fm16s02.cert.pem
chmod 444 intermediate/certs/tld.domain.fm16s03.cert.pem
```

### Verify the certificate
```
openssl x509 -noout -text -in intermediate/certs/www.example.com.cert.pem
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/www.example.com.cert.pem
```

# Deploying the newly signed certificate in FileMaker 16 Server

## Command line interface
```
cp /Volumes/mymedia/intermediate/certs/tld.domain.fm16s00.cert.pem /Library/FileMaker\ Server/CStore/
cp /Volumes/mymedia/intermediate/certs/tld.domain.fm16s01.cert.pem /Library/FileMaker\ Server/CStore/
cp /Volumes/mymedia/intermediate/certs/tld.domain.fm16s02.cert.pem /Library/FileMaker\ Server/CStore/
cp /Volumes/mymedia/intermediate/certs/tld.domain.fm16s03.cert.pem /Library/FileMaker\ Server/CStore/
cp /Volumes/mymedia/intermediate/certs/intermediate.cert.pem /Library/FileMaker\ Server/CStore/
```

##For each server you would like to run the following

### fm16s00
```
fmsadmin certificate import "/Library/FileMaker\ Server/CStore/tld.domain.fm16s00.cert.pem" \
  --intermediateCA "/Library/FileMaker\ Server/CStore/intermediate.cert.pem" \
  --keyfilepass <secret> 
```

### fm16s01
```
fmsadmin certificate import "/Library/FileMaker\ Server/CStore/tld.domain.fm16s01.cert.pem" \
  --intermediateCA "/Library/FileMaker\ Server/CStore/intermediate.cert.pem" \
  --keyfilepass <secret> 
```

### fm16s02
```
fmsadmin certificate import "/Library/FileMaker\ Server/CStore/tld.domain.fm16s02.cert.pem" \
  --intermediateCA "/Library/FileMaker\ Server/CStore/intermediate.cert.pem" \
  --keyfilepass <secret> 
```

### fm16s03
```
fmsadmin certificate import "/Library/FileMaker\ Server/CStore/tld.domain.fm16s03.cert.pem" \
  --intermediateCA "/Library/FileMaker\ Server/CStore/intermediate.cert.pem" \
  --keyfilepass <secret> 
```

## Web interface
| File | Location( if you followed the recipe above ) |
|:-|:-|
|Signed certificate file | /Library/FileMaker Server/CStore/tld.domain.fm16s00.cert.pem |
|Signed certificate file | /Library/FileMaker Server/CStore/tld.domain.fm16s01.cert.pem |
|Signed certificate file | /Library/FileMaker Server/CStore/tld.domain.fm16s02.cert.pem |
|Signed certificate file | /Library/FileMaker Server/CStore/tld.domain.fm16s03.cert.pem |
|Private key file | /Library/FileMaker Server/CStore/serverKey.pem |
|Intermediate certificate file | /Library/FileMaker Server/CStore/intermediate.cert.pem |

# Deploy your self signed root CA to the newly signed certificate for FileMaker 16 Pro \[ Advanced \]
## This is a trick to avoid having to suppress that the self signed certificate is not verified by a 3rd party.
### You may want to run this part in a custom installer pkg, salt, munki, ansible, chef, or other deployment strategies of your choice
```
cp /Volumes/mymedia/ca/certs/ca.cert.pem /Applications/FileMaker\ Pro\ 16\ Advanced/FileMaker\ Pro\ Advanced.app/Contents/Frameworks/Support.framework/Resources/OpenSSL/RootCA/
```
