

1. Create Google Managed ssl-certifacte and new IP

```
gcloud compute ssl-certificates create gke-cert \
    --description=gke-cert \
    --domains=test.web.xyz \
    --global

#create new ip for namespace cluster
gcloud compute addresses create lb-ssl-gke --global
```

2. Create WAF
```
gcloud compute security-policies create waf-policy \
    --description "policy for strict namespace gke load balancer"

gcloud compute security-policies rules update 2147483647 \
    --security-policy waf-policy \
    --action "allow"

gcloud compute security-policies rules create 100 \
        --action=deny-403 \
        --security-policy=waf-policy \
        --expression="origin.region_code.matches('TW|SG|MY')"
```

3. Create new ingress 

ps: please go to namespace istioctl command folder

```
cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  meshConfig:
    defaultConfig:
      proxyMetadata:
        # Enable basic DNS proxying
        ISTIO_META_DNS_CAPTURE: "true"
        # Enable automatic address allocation, optional
        ISTIO_META_DNS_AUTO_ALLOCATE: "true"
  values:
    gateways:
      istio-ingressgateway:
        type: ClusterIP
  components:
    ingressGateways:
      - name: strict-istio-ingressgateway
        enabled: true
        label:
          istio: strict-istio-ingressgateway
EOF
```

4. create new gateway

```
cat <<EOF | kubectl apply -n namespace -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: strict-namespace-gateway
spec:
  selector:
    istio: strict-istio-ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF
```
1. Ingress binding loadbalancer
```

cat <<EOF > strict-ingress-service-patch.yaml
kind: Service
metadata:
    name: strict-istio-ingressgateway
    namespace: istio-system
    annotations:
        cloud.google.com/backend-config: '{"default": "strict-ingress-backendconfig"}'
        cloud.google.com/app-protocols: '{"https":"HTTPS","http2":"HTTP"}'
        cloud.google.com/neg: '{"ingress": true}'
EOF

    kubectl patch svc strict-istio-ingressgateway -n istio-system --patch "$(cat strict-ingress-service-patch.yaml)"
    kubectl patch svc strict-istio-ingressgateway -n istio-system --type=json -p '[{"op":"replace","path":"/spec/type","value":"ClusterIP"}]'


cat <<EOF | kubectl apply -f -
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
    name: strict-ingress-backendconfig
    namespace: istio-system
spec:
    healthCheck:
        requestPath: /healthz/ready
        port: 15020
        type: HTTP
    securityPolicy:
      name: "waf-policy"
EOF

cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  namespace: istio-system
  name: strict-health
spec:
  gateways:
  - strict-health-gateway
  hosts:
  - "*"
  http:
  - match:
    - headers:
        user-agent:
          prefix: GoogleHC
        method:
          exact: GET
        uri:
          exact: /
    rewrite:
      authority: strict-istio-ingressgateway.istio-system.svc.cluster.local:15020
      uri: /healthz/ready
    route:
    - destination:
        host: strict-istio-ingressgateway.istio-system.svc.cluster.local
        port:
          number: 15020
EOF

cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  namespace: istio-system
  name: strict-health-gateway
spec:
  selector:
    istio: strict-istio-ingressgateway
  servers:
  - port: 
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
EOF

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: strict-gke-ingress
    namespace: istio-system
    annotations:
        kubernetes.io/ingress.allow-http: "false"
        kubernetes.io/ingress.global-static-ip-name: lb-ssl-gke
        ingress.gcp.kubernetes.io/pre-shared-cert: "gke-cert"
spec:
    defaultBackend:
        service:
            name: strict-istio-ingressgateway
            port:
                number: 80
EOF

```