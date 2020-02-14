# OpenShift 4.x Registry Mirror

1. Set the directory in which the mirror will store configuration and data:

```
export REGISTRY_DIR=/opt/registry
```

2. Generate registry configuration:

```
mkdir -p ${REGISTRY_DIR}/{auth,certs,data}

cd ${REGISTRY_DIR}/certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 365 -out domain.crt

htpasswd -bBc ${REGISTRY_DIR}/auth/htpasswd ocp 123456
```

Note: You must add the domain used in the openssl certificate in your DNS.

3. Run the registry with `podman`:

```
sudo podman run --name mirror-registry -p 5000:5000 \
     -v ${REGISTRY_DIR}/data:/var/lib/registry:z \
     -v ${REGISTRY_DIR}/auth:/auth:z \
     -e "REGISTRY_AUTH=htpasswd" \
     -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
     -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
     -v ${REGISTRY_DIR}/certs:/certs:z \
     -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
     -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
     -d docker.io/library/registry:2
```

Note: Open port `5000` in your firewall if needed.

4. Add the certificate to your ca trust chain and test the server:

```
sudo cp ${REGISTRY_DIR}/certs/domain.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust

curl -i -u ocp:123456 -k https://registry.localdomain:5000/v2/_catalog
```

5. Copy your pull secret from [cloud.redhat.com](https://cloud.redhat.com) and put into file:

```
echo '(pull-secret-json)' > ${REGISTRY_DIR}/pull-secret.json
```

6. Add your new mirror credentials to the pull secret:

```
NEW_SECRET=$(echo -n "ocp:123456" | base64 -w0)
jq ".auths |= . + {\"registry.localdomain:5000\":{\"auth\": \"$NEW_SECRET\",\"email\":\"suavo@example.com\"}}" pull-secret.json > a && mv a pull-secret.json
```

7. Mirror it

```
export OCP_RELEASE=4.3.0-x86_64
export LOCAL_REGISTRY="registry.localdomain:5000"
export LOCAL_REPOSITORY="openshift-release-dev"
export PRODUCT_REPO="openshift-release-dev"
export LOCAL_SECRET_JSON="${REGISTRY_DIR}/pull-secret.json"
export RELEASE_NAME="ocp-release"

oc adm -a ${LOCAL_SECRET_JSON} release mirror \
     --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE} \
     --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
     --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}
```

8. Use it as a installer:

```
oc adm -a ${LOCAL_SECRET_JSON} release extract --command=openshift-install "${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}"
```