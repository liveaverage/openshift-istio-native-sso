# openshift-istio-native-sso
Solution to configure authn/authz on OpenShift 4 using OpenShift Service Mesh v2, oauth2-proxy, and an IDP supporting OIDC (Gitlab in this case)

| **NOTE**: still working on updating hard-coded manifest values to variables - YMMV if using this example immediately

## Configuration

1. Configure variables
```

OIDC_DISCOVERY_URL=https://gitlab.int.shifti.us/.well-known/openid-configuration
OIDC_DISCOVERY_URL_RESPONSE=$(curl $OIDC_DISCOVERY_URL)
OIDC_ISSUER_URL=$(echo $OIDC_DISCOVERY_URL_RESPONSE | jq -r .issuer)
OIDC_JWKS_URI=$(echo $OIDC_DISCOVERY_URL_RESPONSE | jq -r .jwks_uri)

OAUTH2_PROXY_CLIENT_ID=""
OAUTH2_PROXY_CLIENT_SECRET=""
OAUTH2_PROXY_COOKIE_SECRET=$(openssl rand -hex 16)

APP_HOSTNAME=httpbin.apps.vmx.int.shifti.us

PROJECT_APPS=alpha-httpbin
PROJECT_SMCP=alpha-smcp

```

2. Many of the manifest templates have references to variables, so let's swap these out and generate standard yaml manifests:
```
### FIXME - envsubst
```

## Notes & To-Do

- Add `envsubst` commands to populate manifest templates with current environment variables


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
