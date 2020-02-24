# How to set up Spinnaker with SAML and X509 in kubernetes


# 0 - System requirements
```
brew install helmfile
helm plugin install https://github.com/databus23/helm-diff
```

# 1 - Create okta application

- Single sign on URL: https://gate.spinnaker.DOMAIN.com/saml/SSO
- Audience URI: spinnaker.prod
- Download IdP metadata XML as metadata.xml

# 2 - Create certs for SAML
```
keytool -genkey -v -keystore saml.jks -alias saml -keyalg RSA -keysize 2048 -validity 10000 # Find Password in 3.0
keytool -importkeystore -srckeystore saml.jks -destkeystore saml.p12 -srcstoretype JKS -deststoretype PKCS12 -deststorepass ${PASSWORD}
```

# 3 - Create certs for X509

## 3.0 Create shared password for all certificates

```
export PASSWORD=12345678
```

## 3.1 - Create root CA
```
# create CA key
openssl genrsa -des3 -out ca.key -passout pass:${PASSWORD} 4096
# self sign the cert
openssl req -new -x509 -days 365 -key ca.key -out ca.crt  -passin pass:${PASSWORD}
```

## 3.2 - Deck stuff
```
# server key for deck
openssl genrsa -des3 -out deck.key -passout pass:${PASSWORD} 4096
# Generate a certificate signing request for deck
openssl req -new -key deck.key -out deck.csr -passin pass:${PASSWORD}
# Use the CA to sign the server’s request and create a Deck pem
openssl x509 -req -days 365 -in deck.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out deck.crt -passin pass:${PASSWORD} -sha256 -extfile deck.v3.ext
```

## 3.3 - Gate stuff
```
# server key for gate
openssl genrsa -des3 -out gate.key -passout pass:${PASSWORD} 4096
# Generate a csr for gate
openssl req -new -key gate.key -out gate.csr -passin pass:${PASSWORD}
# Use the CA to sign the server's request and create a Gate pem
openssl x509 -req -days 365 -in gate.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out gate.crt -passin pass:${PASSWORD} -sha256 -extfile gate.v3.ext


# Convert the pem format Gate server certificate into a PKCS12
openssl pkcs12 -export -clcerts -in gate.crt -inkey gate.key -out gate.p12 -name gate -passin pass:${PASSWORD} -password pass:${PASSWORD}

# Create a new Java Keystore (JKS) containing your p12 gate csr
keytool -importkeystore -srckeystore gate.p12 -srcstoretype pkcs12 -srcalias gate -destkeystore gate.jks -destalias gate -deststoretype pkcs12 -deststorepass ${PASSWORD} -destkeypass ${PASSWORD} -srcstorepass ${PASSWORD}

# Import the CA certificate into the Java Keystore
keytool -importcert -keystore gate.jks -alias ca -file ca.crt -storepass ${PASSWORD} -noprompt
```

## 3.4 - Verification
```
# Look at gate.jks
keytool -list -keystore gate.jks -storepass ${PASSWORD}

# Confirm the same
openssl  rsa -text -in deck.key -noout -modulus | grep Modulus= | openssl md5
openssl x509 -text -in deck.crt -noout -modulus | grep Modulus= | openssl md5

# Confirm the same
openssl  rsa -text -in gate.key -noout -modulus | grep Modulus= | openssl md5
openssl x509 -text -in gate.crt -noout -modulus | grep Modulus= | openssl md5
```

# 4 - X509
```
# Create a client key
openssl genrsa -des3 -out client.enc.key 4096
# Unencrypt key
openssl rsa -in client.enc.key -out client.key -passin pass:${PASSWORD}
# Generate a certificate signing request
openssl req -new -key client.key -out client.csr
# Use the CA to sign the server’s request
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt
# Format the client certificate into browser importable form
openssl pkcs12 -export -clcerts -in client.crt -inkey client.key -out client.p12
# Create openssl.conf
cat openssl.conf
# Generate a CSR for a new x509
COUNTRY=US # fill out the rest of the variables
STATE=CA
CITY=
ORG=
EMAIL=
openssl req -nodes -newkey rsa:2048 -keyout key.out -out client.csr -subj "/C=${COUNTRY}/ST=${STATE}/L=${CITY}/O=${ORG}/CN=${EMAIL}" -config openssl.conf
# Use the CA to sign the server’s request
openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt
```

## 4.1 Verify
```
openssl  rsa -text -in key.out -noout -modulus | grep Modulus= | openssl md5
openssl x509 -text -in client.crt -noout -modulus | grep Modulus= | openssl md5
```


# 5 - Set environment

```
export SAMLP12=$(fold -w 64 <(cat saml.p12  | base64))
export PASS=$(echo -n '${PASSWORD}' | base64)
export METADATA=$(cat metadata.xml | base64)
export GATE_JKS=$(cat gate.jks | base64)
export DECK_CRT=$(cat deck.crt | base64)
export DECK_KEY=$(cat deck.key | base64)
```

# 6 - Create Namespaces

```
kubectl create ns cert-manager
kubectl create ns nginx-ingress
kubectl create ns spinnaker
```

# 7 - Spin up
```
kubectl apply -f issuer.yaml # First edit DOMAIN and EMAIL
helmfile deps
helmfile apply # First edit DOMAIN and PASSWORD
```

# 8 - configure spin CLI
~/.spin/config
```
gate:
  endpoint: https://gate.spinnaker.DOMAIN.com # fill out domain
auth:
  enabled: true
  x509:
    certPath: "~/github.com/cgetzen/spinnaker-guides/client.crt"
    keyPath: "~/github.com/cgetzen/spinnaker-guides/key.out"
```

# What's Next / Notes
I would like to move from a self-managed CA to using lets-encrypt and cert-manager.
Please try out helmfile.letsencrypt.yaml

Some issues:

- Hal isn't smart enough to use the gate-local.yml properly, so to make the spin-gate deployment come up properly
you will have to edit the readiness check to be HTTP

- nginx-ingress does not allow you to forward multiple ports to a single path. To use the apiPort (8085),
you will have to use kubectl to port-forward and modify /etc/hosts for the spin-cli.

I have found that it makes more sense to terminate SSL at the ingress, and not use X509, and instead
just port forward when using spin-cli or terraform.
