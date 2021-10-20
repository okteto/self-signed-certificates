# Generate self-signed certificates for Okteto Enterprise

This guide will show you how to use self-signed certificates with your Okteto Enterprise instance. This assumes that you are already familiar with [Okteto's deployment guide](https://okteto.com/docs/enterprise/install/deployment/). 

## Generate the Certificates 

### Install CFSSL 

```
brew install cfssl
```

You can also get the binaries directly [from their github repo](https://github.com/cloudflare/cfssl).


### The Certificate Authority

The certificate authority is created using the information on `ca.json`. Modify as needed, and then run the following command to create the CA's private and public keys:

```
cfssl gencert -initca ca.json | cfssljson -bare ca
```

### The Intermediate CA

The intermediate certificate authority is created using the information on `intermediate-ca.json`. Modify as needed, and then run the following command to create the Intermediate CA's private and public keys:

```
cfssl gencert -initca intermediate-ca.json | cfssljson -bare intermediate_ca
cfssl sign -ca ca.pem -ca-key ca-key.pem -config profile.json -profile intermediate_ca intermediate_ca.csr | cfssljson -bare intermediate_ca
```

### The Certificate

The certificate is created using the information on `certificate.json`. Modify as needed, and then run the following command to create the private and public keys:


```
cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config profile.json -profile=server certificate.json | cfssljson -bare self-signed-certificate
```

## Install the Certificates

Once the certificates are created, you'll need to upload them to your Kubernetes cluster for Okteto to be able to use them. 

```
kubectl create secret tls self-signed-certificate --key self-signed-certificate-key.pem --cert self-signed-certificate.pem --namespace okteto
```

You also need to upload the CA we just created

```
kubectl create secret generic self-signed-ca --from-file=ca.crt=intermediate_ca.pem --namespace okteto
```

## Configure Okteto to use your self-signed certificates

Update your helm configuration file to use the certificates we created by adding the values below. You can see the extended version [of this instructions here](https://okteto.com/docs/enterprise/administration/certificates/).

```
wildcardCertificate:
  create: false
  name: self-signed-certificate
  privateCA:
    enabled: true
    secret:
      name: self-signed-ca 
      key: ca.crt
  
ingress-nginx:
  controller:
    extraArgs:
      default-ssl-certificate: $(POD_NAMESPACE)/self-signed-certificate
```

Install (or upgrade) your instance by following [these instructions](https://okteto.com/docs/enterprise/install/deployment/#deploy-the-okteto-enterprise-chart). 

Once the install/upgrade command finishes, your Okteto Enterprise instance will be configured to the self-signed certificates you provided. 

## Install the self-signed CA

We recommend that you install the self-signed CA we created on all the devices that you'll be using to access your Okteto Enterprise instance, to avoid having any SSL trust issues. 

For MacOS:

1. Open the KeyChain Access
1. Import `ca.pem` to the `Login` keychain.
1. Import `ca-intermediate.pem` to the `Login` keychain.