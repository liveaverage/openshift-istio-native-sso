# openshift-istio-native-sso
Solution to configure authn/authz on OpenShift 4 using OpenShift Service Mesh v2, oauth2-proxy, and an IDP supporting OIDC (Gitlab in this case)

| **NOTE**: still working on updating hard-coded manifest values to variables - YMMV if using this example immediately

## Configuration

1. Configure variables
```

export OIDC_DISCOVERY_URL=https://gitlab.int.shifti.us/.well-known/openid-configuration
export OIDC_DISCOVERY_URL_RESPONSE=$(curl -k $OIDC_DISCOVERY_URL)
export OIDC_ISSUER_URL=$(echo $OIDC_DISCOVERY_URL_RESPONSE | jq -r .issuer)
export OIDC_JWKS_URI=$(echo $OIDC_DISCOVERY_URL_RESPONSE | jq -r .jwks_uri)
export OIDC_PROFILE_URL=$(echo $OIDC_DISCOVERY_URL_RESPONSE | jq -r .userinfo_endpoint)
export OIDC_TOKEN_ENDPOINT=$(echo $OIDC_DISCOVERY_URL_RESPONSE | jq -r .token_endpoint)
export OIDC_AUTH_ENDPOINT=$(echo $OIDC_DISCOVERY_URL_RESPONSE | jq -r .authorization_endpoint)

export OAUTH2_PROXY_CLIENT_ID=""
export OAUTH2_PROXY_CLIENT_SECRET=""
export AUTH2_PROXY_COOKIE_SECRET=$(openssl rand -hex 16)

export APP_HOSTNAME=httpbin.apps.vmx.int.shifti.us

export PROJECT_APPS=alpha-httpbin
export PROJECT_SMCP=alpha-smcp

```

2. Many of the manifest templates have references to variables, so let's swap these out and generate standard yaml manifests:
```bash
git clone https://github.com/liveaverage/openshift-istio-native-sso.git
cd openshift-istio-native
mkdir -p tmp/project tmp/project-smcp
envsubst < project/sm-virtualsvc.yaml.tmpl > tmp/project/sm-virtualsvc.yaml
envsubst < project/sm-gateway.yaml.tmpl > tmp/project/sm-gateway.yaml

envsubst < project-smcp/auth-requestauthentication.yaml.tmpl > tmp/project-smcp/auth-requestauthentication.yaml
envsubst < project-smcp/auth-envoyfilter.yaml.tmpl > tmp/project-smcp/auth-envoyfilter.yaml

```

```bash
### Configure oauth2-proxy client details in configmap
oc apply -f - <<EOF
apiVersion: v1
data:
  OAUTH2_PROXY_CLIENT_ID: ${OAUTH2_PROXY_CLIENT_ID}
  OAUTH2_PROXY_CLIENT_SECRET: ${OAUTH2_PROXY_CLIENT_SECRET}
  OAUTH2_PROXY_COOKIE_SECRET: ${OAUTH2_PROXY_COOKIE_SECRET}
kind: Secret
metadata:
  name: oauth2-proxy
type: Opaque
EOF

```

3. Apply the generated manifests to appropriate projects/namespaces:
```bash
oc apply -f project/httpbin/*.yaml -n ${PROJECT_APPS}
oc apply -f project/oauth2-proxy/*.yaml -n ${PROJECT_APPS}
oc apply -f project/*.yaml -n ${PROJECT_APPS}
oc apply -f tmp/project/*.yaml -n ${PROJECT_APPS}

oc apply -f project-smcp/*.yaml -n ${PROJECT_SMCP}
oc apply -f tmp/project-smcp/*.yaml -n ${PROJECT_SMCP}
```

## Notes & To-Do

- Add `envsubst` commands to populate manifest templates with current environment variables
- This example uses the generic `oidc` provider vs. `gitlab`, which requires a couple of extra args being provided to the oauth2-proxy container:

```bash
        - --profile-url=https://gitlab.int.shifti.us/oauth/userinfo
        - --oidc-groups-claim=groups
```

## Troubleshooting

- Bump debug output envoy on the ingress gateway and sidecars:
```bash
# Debug for httpbin istio-proxy
oc exec $(oc get pods -o custom-columns=POD:.metadata.name --no-headers -n $PROJECT_APPS | grep http) -c istio-proxy -n $PROJECT_APPS -- curl -X POST http://localhost:15000/logging?level=debug

# Debug for oauth2-proxy istio-proxy
oc exec $(oc get pods -o custom-columns=POD:.metadata.name --no-headers -n $PROJECT_APPS | grep oauth) -c istio-proxy -n $PROJECT_APPS -- curl -X POST http://localhost:15000/logging?level=debug

# Debug for gateway
oc exec $(oc get pods -o custom-columns=POD:.metadata.name --no-headers -n $PROJECT_SMCP | grep ingress) -n $PROJECT_SMCP -- curl -X POST http://localhost:15000/logging?level=debug

```
- You may also need to consult the istiod/pilot-discovery pod as well -- this is where I noticed x509 validation errors when fetching jwt keys
```
error	model	Failed to fetch public key from "https://idp/auth/realms/customer/protocol/openid-connect/certs": Get https://idp/auth/realms/customer/protocol/openid-connect/certs: x509: certificate signed by unknown authority
```
- Although support for `spec.istio.pilot.jwksResolverExtraRootCA` was [added to OSSMv1.1](https://github.com/maistra/istio/issues/99), specifiying this for OSSMv2 SMCP definitions made no difference. I ultimately had to create a configmap manually and add appropriate configmap volume/mount to the `istiod` deployment manifest
```
oc create configmap pilot-jwks-extra-cacerts-smcp --from-file=extra.pem=/etc/letsencrypt/live/int.shifti.us/chain.pem
```
  - Additions to the `istiod` deployment manifest :

```
    volumes:
    - configMap:
        defaultMode: 420
        name: pilot-jwks-extra-cacerts-smcp
      name: extracacerts
    ...
    ...
    volumeMounts:
      - mountPath: /cacerts
        name: extracacerts
  
```
