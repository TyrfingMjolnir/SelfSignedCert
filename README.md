# Self signed cert WIP

Work in progress based on: https://jamielinux.com/docs/openssl-certificate-authority/sign-server-and-client-certificates.html and https://blog.beezwax.net/2017/12/03/creating-your-own-ssl-certificates-for-filemaker/ as the latter is the only blog I have seen focusing on FileMaker security.

The reason for signing your own SSL cert
1) Security

The reason for having your SSL cert publicly signed
1) Authenticate who you are



# Precaution

## If you need a publicly signed SSL certificate
In order to prove that you are you, there are many option, my preferred option is: https://github.com/lukas2511/dehydrated



## If you aim for security of your data follow this procedure
1) Disconnect computer from internet
1) Insert SD card to computer
1) Follow procedure below
1) Copy only the required files from SD card to your computer
1) Eject SD card and put in safe
1) Reconnect your computer to the internet

This guide is written in a way intended to be string replacable

`:%s/mymedia/nameOfYourSDCard`



# Create CA
```
cd /Volumes/mymedia

mkdir ca
cd ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial

curl -kLo openssl.conf https://jamielinux.com/docs/openssl-certificate-authority/_downloads/root-config.txt
```

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
```
cd /Volumes/mymedia/ca
mkdir intermediate
cd intermediate
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
echo 1000 > crlnumber

cp ../openssl.conf ./

openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
chmod 400 intermediate/private/intermediate.key.pem
```

# Create the intermediate certificate
```
openssl req -config openssl.cnf -new -sha256 -key private/intermediate.key.pem -out csr/intermediate.csr.pem

cd ..

openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem
chmod 444 intermediate/certs/intermediate.cert.pem
```

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

openssl req -config intermediate/openssl.cnf -key intermediate/private/www.example.com.key.pem -new -sha256 -out intermediate/csr/www.example.com.csr.pem
openssl ca -config intermediate/openssl.cnf -extensions server_cert -days 375 -notext -md sha256 -in intermediate/csr/www.example.com.csr.pem -out intermediate/certs/www.example.com.cert.pem
chmod 444 intermediate/certs/www.example.com.cert.pem
```

### Verify the certificate
```
openssl x509 -noout -text -in intermediate/certs/www.example.com.cert.pem
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/www.example.com.cert.pem
```
