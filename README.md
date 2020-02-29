Create the root pair
---------------------
- Prepare the directory

```
mkdir /root/ca/{certs,crl,newcerts,private} -p
cd /root/ca 
chmod 700 private
touch index.txt
echo 1000 > serial
```

- Copy the intermediate CA configuration file from files/openssl_root_CA.cnf

```
cat << EOF > /root/ca/openssl.cnf
```

- Create the root key
```
cd /root/ca 
openssl genrsa  -out private/ca.key.pem 4096
chmod 400 private/ca.key.pem
```

- Create the root certificate. Common Name: "example.com Root CA"

```
openssl req -config openssl.cnf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -sha256 -extensions v3_ca \
      -out certs/ca.cert.pem
```

- Verify the root certificate
```
openssl x509 -noout -text -in certs/ca.cert.pem
```

Create the intermediate pair
----------------------------

- Prepare the directory
```
mkdir /root/ca/intermediate/{certs,crl,csr,newcerts,private} -p 
cd /root/ca/intermediate
chmod 700 private
touch index.txt
echo 1000 > serial
```

- Add a crlnumber file to the intermediate CA directory tree. crlnumber is used to keep track of certificate revocation lists.
```
echo 1000 > /root/ca/intermediate/crlnumber
```

- Copy the intermediate CA configuration file from openssl_intermediate_CA.cnf

- Create the intermediate key
```
cd /root/ca
openssl genrsa -out intermediate/private/intermediate.key.pem 4096
chmod 400 intermediate/private/intermediate.key.pem
```


- Create the intermediate certificate - "example.com Intermediate CA"
```
cd /root/ca
openssl req -config intermediate/openssl.cnf -new -sha256 \
      -key intermediate/private/intermediate.key.pem \
      -out intermediate/csr/intermediate.csr.pem
```

- To create an intermediate certificate, use the root CA with the v3_intermediate_ca extension to sign the intermediate CSR. The intermediate certificate should be valid for a shorter period than the root certificate. Ten years would be reasonable

```
cd /root/ca
openssl ca -config openssl.cnf -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 \
      -in intermediate/csr/intermediate.csr.pem \
      -out intermediate/certs/intermediate.cert.pem
```

- Verify the intermediate certificate
```
openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
openssl verify -CAfile certs/ca.cert.pem intermediate/certs/intermediate.cert.pem
```


- Create the certificate chain file
```
cat intermediate/certs/intermediate.cert.pem  certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

Sign server and client certificates 
------------------------------------
- (wildcard -> apps.lab.example.com && devs.lab.example.com ) + (master external -> ocp-external.lab.example.com)
- If alt_names are requiered, config file openssl_intermediate_server_alt_CA.cnf should be used.

ocp-external.lab.example.com
-----------------------------

```
cd /root/ca
openssl genrsa -out intermediate/private/ocp-external.lab.example.com.key.pem 2048
chmod 400 intermediate/private/ocp-external.lab.example.com.key.pem
```

- Create a certificate
```
cd /root/ca
openssl req -config intermediate/openssl.cnf \
      -key intermediate/private/ocp-external.lab.example.com.key.pem \
      -new -sha256 -out intermediate/csr/ocp-external.lab.example.com.csr.pem

cd /root/ca
openssl ca -config intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in intermediate/csr/ocp-external.lab.example.com.csr.pem \
      -out intermediate/certs/ocp-external.lab.example.com.cert.pem

chmod 444 intermediate/certs/ocp-external.lab.example.com.cert.pem
openssl x509 -noout -text \
      -in intermediate/certs/ocp-external.lab.example.com.cert.pem
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem \
      intermediate/certs/ocp-external.lab.example.com.cert.pem

```

*.apps.lab.example.com
-----------------------------
```
cd /root/ca
openssl genrsa -out intermediate/private/wildcard.apps.lab.example.com.key.pem 2048
chmod 400 intermediate/private/wildcard.apps.lab.example.com.key.pem

- Create a certificate
cd /root/ca
openssl req -config intermediate/openssl.cnf \
      -key intermediate/private/wildcard.apps.lab.example.com.key.pem \
      -new -sha256 -out intermediate/csr/wildcard.apps.lab.example.com.csr.pem

cd /root/ca
openssl ca -config intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in intermediate/csr/wildcard.apps.lab.example.com.csr.pem \
      -out intermediate/certs/wildcard.apps.lab.example.com.cert.pem
chmod 444 intermediate/certs/wildcard.apps.lab.example.com.cert.pem
openssl x509 -noout -text \
      -in intermediate/certs/wildcard.apps.lab.example.com.cert.pem
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem \
      intermediate/certs/wildcard.apps.lab.example.com.cert.pem

```
*.devs.lab.example.com
-----------------------------
```
cd /root/ca
openssl genrsa -out intermediate/private/wildcard.devs.lab.example.com.key.pem 2048
chmod 400 intermediate/private/wildcard.devs.lab.example.com.key.pem

- Create a certificate
cd /root/ca
openssl req -config intermediate/openssl.cnf \
      -key intermediate/private/wildcard.devs.lab.example.com.key.pem \
      -new -sha256 -out intermediate/csr/wildcard.devs.lab.example.com.csr.pem

cd /root/ca
openssl ca -config intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in intermediate/csr/wildcard.devs.lab.example.com.csr.pem \
      -out intermediate/certs/wildcard.devs.lab.example.com.cert.pem
chmod 444 intermediate/certs/wildcard.devs.lab.example.com.cert.pem
openssl x509 -noout -text \
      -in intermediate/certs/wildcard.devs.lab.example.com.cert.pem
openssl verify -cafile intermediate/certs/ca-chain.cert.pem \
      intermediate/certs/wildcard.devs.lab.example.com.cert.pem
```

Inventory file variables
--------------------------------

```
openshift_master_named_certificates=[{"certfile": "/home/sperezto/ocp-preq/certs/ocp-external.lab.example.com.cert.pem", "keyfile": "/home/sperezto/ocp-preq/certs/ocp-external.lab.example.com.key.pem", "names": ["ocp-external.lab.example.com"], "cafile": "/home/sperezto/ocp-preq/certs/ca-chain.cert.pem"}] 

openshift_hosted_routers=[{"name": "router-apps", "certificate": {"certfile": "/home/sperezto/ocp-preq/certs/wildcard.apps.lab.example.com.cert.pem", "keyfile": "/home/sperezto/ocp-preq/certs/wildcard.apps.lab.example.com.key.pem", "cafile": "/home/sperezto/ocp-preq/certs/ca-chain.cert.pem"}, "replicas": 1, "serviceaccount": "router", "namespace": "default", "stats_port": 1936, "edits": [{"action": "append", "key": "spec.template.spec.containers[0].env", "value": {"name": "NAMESPACE_LABELS", "value": "router=apps"}}], "images": "openshift3/ose-${component}:${version}", "selector": "node-role.kubernetes.io/infra=true,env=apps", "ports": ["80:80", "443:443"]},{"name": "router-devs", "certificate": {"certfile": "/home/sperezto/ocp-preq/certs/wildcard.devs.lab.example.com.cert.pem", "keyfile": "/home/sperezto/ocp-preq/certs/wildcard.devs.lab.example.com.key.pem", "cafile": "/home/sperezto/ocp-preq/certs/ca-chain.cert.pem"}, "replicas": 1, "serviceaccount": "router", "namespace": "default", "stats_port": 1936, "edits": [{"action": "append", "key": "spec.template.spec.containers[0].env", "value": {"name": "NAMESPACE_LABELS", "value": "router=devs"}}], "images": "openshift3/ose-${component}:${version}", "selector": "node-role.kubernetes.io/infra=true,env=devs", "ports": ["80:80", "443:443"]}]

```

Openshift node groups
--------------------------------
```
openshift_node_groups=[{'name': 'node-config-master', 'labels': ['node-role.kubernetes.io/master=true','logging=true']}, {'name': 'node-config-infra-apps', 'labels': ['node-role.kubernetes.io/infra=true', 'logging=true', 'env=apps']}, {'name': 'node-config-infra-devs', 'labels': ['node-role.kubernetes.io/infra=true', 'logging=true','env=devs']} ,{'name': 'node-config-compute-apps', 'labels': ['node-role.kubernetes.io/compute=true','logging=true', 'env=apps']}, {'name': 'node-config-compute-devs', 'labels': ['node-role.kubernetes.io/compute=true','logging=true','env=devs']}, {'name': 'node-config-compute-gluster', 'labels': ['node-role.kubernetes.io/compute=true','logging=true','glusterfs=storage-host']}]
```

--------------------------------
Source
--------------------------------
https://jamielinux.com/docs/openssl-certificate-authority/create-the-root-pair.html
