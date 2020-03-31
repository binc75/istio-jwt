# ISTIO tutorial -- JWT authentication
The purpose of the demo is to demonstare the ability of ISTIO to do an end-user authentication and authorization. 

We are going to setup a *minikube* cluster, install *istio*, install a demo application and install *Keycloak* (https://www.keycloak.org/) to demonstrate how ***JWT authn/authz*** works. 

The workflow will be:
 - the user will get a JWT from keycloak with correct attributes ("aud": "account")
   that will be matched againts "audiences" attribute into the Policy object configuration
 - the user will call the app inside k8s/istio presenting the JTW 
   - if the token is valid the user will get an HTTP 200
   - if the token is not give, is invalid, is expired it will get an HTTP401 (Unauthorized)

![Setup schema](img/istio-JWT-schema.png?raw=true "Schema")


## Environment setup
Setup minikube with the appropriate resources:
```bash
minikube start -p istio-mk --memory=8192 --cpus=3 \
  --kubernetes-version=v1.18.0 \
  --driver=virtualbox --disk-size=20g
```

## ISTIO setup
```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.*
export PATH=$PWD/bin:$PATH
cd ..

## The demo configuration profile is not suitable for performance evaluation. 
## It is designed to showcase Istio functionality with high levels of tracing and access logging
## For a production setup consider: $ istioctl manifest apply

istioctl manifest apply --set profile=demo
```

## Configure sidecar injector 
Setup the automatic sidecar injection feature of istio for the *default* namespace:  
```bash
kubectl label namespace default istio-injection=enabled
```
Check
```bash
kubectl describe ns default
```

## Deploy example app
```bash
kubectl apply -f deployment/backend.yaml
kubectl apply -f deployment/istio-backend.yaml
kubectl apply -f deployment/istio-gw.yaml
```
Quick explanation:  
* backend.yaml
  * custom python flask API application (2 versions: backend-v1 & backend-v2)  
    This application read the env variable *VERSION* and return it when called.
  * service (ClusterIP) to expose the application (label: app: backend)
* istio-gw.yaml
  * define an istio gateway named *demo-gateway* that route traffic from the external world to the inside of k8s
* istio-backend.yaml
  * defines an istio VirtualService that route traffic to the instances of the python app
  * defines an istio DestinationRule to distinguish between the 2 versions of the python app  
    (labels: version: v1 & v2)

### Checking out basic setup
Get and set env variables
```bash
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
export INGRESS_HOST=$(minikube -p istio-mk ip)
```
Check if the app respond on base url (should balance 80% to v1 and 20% to v2):
```bash
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT; sleep .2;done
```

Check if you call /v1 you only see response from v1 app:
```bash
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT/v1; sleep .2;done
```

Check if you call /v2 you only see response from v2 app:
```bash
while true; do curl http://$INGRESS_HOST:$INGRESS_PORT/v2; sleep .2;done
```

## Deploy Keycloak 
**Open Source Identity and Access Management**
Keycloak will act as a JWT provider trusted by istio. 

```bash
# Create dedicated namespace for keycloak
kubectl create ns keycloak-ns

# Deploy keycloak and a service (NodePort)
kubectl apply -f deployment/keycloak.yaml
```
Get the service external address
```bash
minikube -p istio-mk -n keycloak-ns service keycloak --url
```
NOTE: it may take a while for keycloak to be up&running. 
Wait few minutes before try to reach the service. 

Check keycloak logs
```bash
stern -n keycloak-ns .
```

### Configure Keycloak
#### Manual configuration
Now configure Keycloak as following:
 - login to the administration console via browser
   - address: minikube -p istio-mk -n keycloak-ns service keycloak --url
   - user: admin, pass: password
 - create a new *realm* called: **istio**
 - inside the **istio realm** create  a client configuration with:
   - Client ID: *istio*
   - Access Type: *confidential*
   - Valid Redirect URIs: output of command **echo "http://$INGRESS_HOST:$INGRESS_PORT"** 

Take note of the **Secret** from the Credentials tab in the Client configuration to be used further ahead. 

#### Automated config (copy & paste)
```bash
kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh config credentials --server http://localhost:8080/auth --realm master --user admin --password password

kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh create realms -s realm=istio -s enabled=true

kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh create clients -r istio -s clientId=istio -s enabled=true -s clientAuthenticatorType=client-secret -s serviceAccountsEnabled=true -s directAccessGrantsEnabled=true
```
Get the client secret
```bash
# First get the client id of the istio client
CLIENT_ID=$(kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh get clients -r istio --fields id,clientId | jq -j '.[] | select(.clientId | contains("istio")).id')

# Get the istio client secret to be used lately
CLIENT_SECRET=$(kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh get -r istio clients/$CLIENT_ID/client-secret | jq -j '.value')

# Export the variable for further use
export CLIENT_SECRET
```
Optionally you can create ad additional user in keycloak
```bash
# Create user
kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh create users -r istio -s username=testuser -s enabled=true

# Set user password
kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh set-password -r istio --username testuser --new-password abc123

# Set a new role (realm role)
kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh create roles -r istio -s name=backendaccess -s 'description=Access to backend app'

# Assign role to user
kubectl exec -it -n keycloak-ns $(kubectl get pod -n keycloak-ns -o jsonpath='{.items[0].metadata.name}') -- /opt/jboss/keycloak/bin/kcadm.sh add-roles --uusername testuser --rolename backendaccess -r istio
```

## Istio Policy
Activate the istio policy
```bash
export KEYCLOAK_URL=$(minikube -p istio-mk -n keycloak-ns service keycloak --url)
cat deployment/istio-policy.yaml.template | envsubst | kubectl apply -f -
```

## Checkout the feature
Configure the client secret to get the access token from Keycloak. 
```bash
# Only needed if you configured Keycloak manualy
export CLIENT_SECRET="XXXXX XXXXX XXXXXX XXXXX"
```
Get the token from Keycloak (emulating an application)
```bash
TOKEN=$(curl -s -d "audience=istio" -d "client_id=istio" -d "client_secret=$CLIENT_SECRET" -d "grant_type=client_credentials" "$(minikube -p istio-mk -n keycloak-ns service keycloak --url)/auth/realms/istio/protocol/openid-connect/token" | jq -r ".access_token")
```
Alternative TOKEN (emulating a simple user) (optional)
```bash
TOKEN=$(curl -s -d "client_id=istio" -d "client_secret=$CLIENT_SECRET" -d 'username=testuser' -d 'password=abc123' -d 'grant_type=password' "$(minikube -p istio-mk -n keycloak-ns service keycloak --url)/auth/realms/istio/protocol/openid-connect/token" | jq -r ".access_token")
```
Call the app sending the correct token, you should get a HTTP 200:
```bash
curl -svI http://$INGRESS_HOST:$INGRESS_PORT --header "Authorization: Bearer ${TOKEN}"
```
Try now without token or with an invalid or expired one!


### Tricks
DEBUG for ingress activation
```bash
$ kubectl exec $(kubectl get pods -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -n istio-system -- curl -X POST "localhost:15000/logging?level=debug" -s
```
Tail logs of ingress-controller
```bash
stern istio-ingress -n istio-system
```

Put all the sidecar proxies of the backend pods in DEBUG mode:
```bash
for x in $(kubectl get pods -l app=backend -o jsonpath='{.items[*].metadata.name}')
  do
  kubectl exec -c istio-proxy $x -- curl -X POST "localhost:15000/logging?level=debug" -s
done
```
Tail log of istio sidecars
```bash
stern backend
```

## Inspect JTW token 
https://jwt.io/

Example:
```json
# Header
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "rtJ6g22byb1WSQ2u5MEzPkgm4fH9BjSQAOCJz7oY-Bk"
}
# Payload
{
  "jti": "37b0906e-6042-4308-9423-aac76d63a3b7",
  "exp": 1585572946,
  "nbf": 0,
  "iat": 1585572646,
  "iss": "http://192.168.99.100:30080/auth/realms/istio",
  "aud": "account",
  "sub": "2126d82d-cb36-4e8d-a999-71cdfcd3da5a",
  "typ": "Bearer",
  "azp": "istio",
  "auth_time": 0,
  "session_state": "ab5fbb83-f5b8-4a69-986b-2a4fb10104d6",
  "acr": "1",
  "realm_access": {
    "roles": [
      "offline_access",
      "uma_authorization"
    ]
  },
  "resource_access": {
    "account": {
      "roles": [
        "manage-account",
        "manage-account-links",
        "view-profile"
      ]
    }
  },
  "scope": "email profile",
  "clientHost": "172.17.0.1",
  "email_verified": false,
  "clientId": "istio",
  "preferred_username": "service-account-istio",
  "clientAddress": "172.17.0.1",
  "email": "service-account-istio@placeholder.org"
}
```


## Kiali
To have a realtime graphical representation of the situation you can look at *kiali*
```bash
istioctl dashboard kiali &
```

## Cleanup 
```bash
minikube -p istio-mk stop
minikube -p istio-mk delete
```
