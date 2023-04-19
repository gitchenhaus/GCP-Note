# Create Cluster for Channel Downgrade

如果要完全參照本文件執行，必須確定原始 GKE 已建立完成，並且跟 GCE Server 的防火牆已打通

## Prepare before change

### Create New Cluster (On Cloud Shell)

>cli default 必須夾帶 `default-pool`,可以考慮使用 UI

1. 創建新的 Cluster with `default-pool` 

    gcloud beta container clusters create **CLUSTER_NAME** \
    --addons=Istio,HttpLoadBalancing \
    --istio-config=auth=MTLS_PERMISSIVE \
    --enable-logging-monitoring-system-only \
    --machine-type=**MACHINE_TYPE**  \
    --release-channel **CHANNEL** \
    --no-enable-autoupgrade \
    --zone **ZONE** \
    --cluster-version **VERSION** \
    --network=**NETWORK** \
    --num-nodes=**NUM_NODES** \
    --disk-size=100GB \
    --disk-type=pd-ssd

    **Example:**

    ```bash
    gcloud beta container clusters create [CLUSTER_NAME] \
    --addons=Istio,HttpLoadBalancing \
    --istio-config=auth=MTLS_PERMISSIVE \
    --enable-logging-monitoring-system-only \
    --machine-type=e2-medium \
    --release-channel None \
    --zone asia-east2-a \
    --cluster-version 1.18.16-gke.1200 \
    --no-enable-autoupgrade \
    --network=default \
    --num-nodes=1 \
    --disk-size=100GB \
    --disk-type=pd-ssd
    ```

    **參數說明:**
     - --enable-logging-monitoring-system-only: 設定Cloud Operations只紀錄[系統相關](https://cloud.google.com/stackdriver/docs/solutions/gke/installing#collecting_system-only_logs)日誌及監控.


2. 創建 `application-pool` `monitor-pool` work node

    gcloud container node-pools create **POOL_NAME** \
    --cluster CLUSTER_NAME
    --machine-type=**MACHINE_TYPE**  \
    --zone **ZONE** \
    --node-version **VERSION** \
    --network=**NETWORK** \
    --num-nodes=**NUM_NODES** \
    --disk-size=100GB \
    --disk-type=pd-ssd \
    --no-enable-autoupgrade

    **Example:**

    ```bash
    gcloud container node-pools create application-pool \ 
    --cluster [CLUSTER_NAME] \
    --machine-type=n1-highcpu-64 \
    --zone asia-east2-a \
    --node-version 1.18.16-gke.1200 \
    --num-nodes=4 \
    --disk-size=100GB \
    --disk-type=pd-ssd \
    --no-enable-autoupgrade

    gcloud container node-pools create monitor-pool \
    --cluster [CLUSTER_NAME] \
    --machine-type=c2-standard-16 \
    --zone asia-east2-a \
    --node-version 1.18.16-gke.1200 \
    --num-nodes=1 \
    --disk-size=100GB \
    --disk-type=pd-ssd \
    --no-enable-autoupgrade
    ```

3. Delete `default-pool`

    gcloud container node-pools delete **POOL_NAME** \
    --cluster **CLUSTER_NAME** \
    --zone **ZONE**

    **Example:**

    ```bash
    gcloud container node-pools delete default-pool \
    --cluster [CLUSTER_NAME] \
    --zone asia-east2-a
    ```

### Init Before Setup Sport-betting Cluster (On Cloud Shell)

1. Create nfs Storage **(之前做過的，就不需要)**

    gcloud beta filestore instances create **STORAGE_NAME** \
    --project=**PROJECT_ID** \
    --zone **ZONE** \
    --tier=STANDARD \
    --file-share=name="volumes",capacity=**SIZE** \
    --network=name=**NETWORK**

    **Example:**
    
    ```bash
    gcloud beta filestore instances create nfs-storage \
    --project=[PROJECT_ID] \
    --zone asia-east2-a \
    --tier=STANDARD \
    --file-share=name="volumes",capacity=2560G \
    --network=name="default"
    ```

2. Setup Helm repo **(之前做過的，就不需要)**

    **Example:**

    ```bash
    helm repo add stable https://charts.helm.sh/stable
    helm repo add incubator https://charts.helm.sh/incubator
    helm repo add inspur https://inspur-iop.github.io/chartsss
    helm repo update
    ```

### Setup New Cluster for Sport-betting (On Deploy-Server, CloudShell and Jenkins)

1. 將 terminal 轉換成新的 cluster 的驗證
    >on deploy-server

    gcloud container clusters get-credentials **CLUSTER_NAME** \
    --zone **ZONE** \
    --project **PROJECT_ID**

    **Example:**

    ```bash
    gcloud container clusters get-credentials [CLUSTER_NAME] \
    --zone asia-east2-a \
    --project [PROJECT_ID]
    ```

2. 將 Istio 完整更新到 1.6
    >on deploy-server

   [官方文件](https://cloud.google.com/istio/docs/istio-on-gke/upgrade-with-operator?hl=zh-tw)

    **檢查語法** 確認版本為 1.6
    ```bash
    kubectl get pods -n istio-system -l app=istio-ingressgateway -o yaml | grep image
    ```

3. Create Namespace
    >on deploy-server

    **Example:**

    ```bash
    kubectl create namespace sport-betting
    kubectl create namespace monitor
    kubectl create namespace kafka
    kubectl create namespace redis
    kubectl create namespace influxdb
    ```

4. Setting Istio
    >on deploy-server

    **Example:**

    VERSION = $(kubectl -n istio-system get pods -lapp=istiod --show-labels) and check `istio.io/rev=VERSION`

    ```bash
    kubectl label namespace sport-betting istio-injection- istio.io/rev=**VERSION** --overwrite
    kubectl patch -n=istio-system svc istio-ingressgateway --type=json -p '[{"op":"replace","path":"/spec/externalTrafficPolicy","value":"Local"}]'

    cat <<EOF | kubectl apply -f -
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      namespace: sport-betting
      name: sport-betting-gateway
    spec:
      selector:
        istio: ingressgateway
      servers:
      - port: 
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - "*"
    EOF
    ```

5. Setting goproxy via Isito ServiceEntry
    >on deploy-server

    **Example:**

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: networking.istio.io/v1alpha3
    kind: ServiceEntry
    metadata:
      namespace: sport-betting
      name: https-proxy-hosts
    spec:
      hosts:
      - "*.prdasbbwla1.com"
      - "*.xj-intg888.com"
      ports:
      - number: 443
        name: https-tunnel
        protocol: TLS
      location: MESH_INTERNAL
      resolution: STATIC
      endpoints:
      - address: 10.170.0.51 #請自行替換成 goporxy host 的 server ip
    EOF
    ```

6. Create secret for docker pulling image
    >on deploy-server

    **Example:**
    
    ```bash
    kubectl create secret docker-registry nexus-regcred \
    -n sport-betting \
    --docker-server="10.170.0.2:8082" \ #請自行替換成 nexus host 的 server ip
    --docker-username="username" \      #請自行替換成 nexus 的 username
    --docker-password='password'        #請自行替換成 nexus 的 password
    ```

7. Create nfs-client and change default storageclass to nfs
    >on deploy-server

    **Example:**

    ```bash
    helm install  nfs-cp --set nfs.server=[FIRESTRE_IP] --set nfs.path=/volumes stable/nfs-client-provisioner

    kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}' && kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

8. Create `Sport-betting` PV and PVC
    >on deploy-server

    **Example:**

    ```bash
    cat <<EOF | kubectl apply -n=sport-betting -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: app-pv-claim
    spec:
      storageClassName: nfs-client
      accessModes:
      - ReadWriteMany
      resources:
        requests:
        storage: 200Gi
    EOF

    cat <<EOF | kubectl apply -n=sport-betting -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: app-resources-claim
    spec:
    storageClassName: nfs-client
    accessModes:
    - ReadWriteMany
    resources:
        requests:
        storage: 200Gi
    EOF

    cat <<EOF | kubectl apply -n=sport-betting -f -
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: app-uploads-claim
    spec:
    storageClassName: nfs-client
    accessModes:
    - ReadWriteMany
    resources:
        requests:
        storage: 200Gi
    EOF
    ```

    完成後可以將就的 `upload` 和 `resource` 的資料直接複製到新的資料夾內


9.  Install Consul, Grafana, Promethus and Jaeger via it-automation
    >on deploy-server

    沿用 it-automation 安裝
    ```bash
    ansible-playbook --private-key ~/.ssh/gce_keys -i inventories/gcp-prod.ini playbooks/consul.yml
    ansible-playbook --private-key ~/.ssh/gce_keys -i inventories/gcp-prod.ini playbooks/jaeger.yml
    ansible-playbook --private-key ~/.ssh/gce_keys-i inventories/gcp-prod.ini playbooks/prometheus-grafana.yml
    ansible-playbook --private-key ~/.ssh/gce_keys-i inventories/gcp-prod.ini fluentd-server.yml
    ```

10. Install lagerlog, internal_service, external_service, fluentd_client and auth_proxy
    >on jenkins
    
    - edit Job `0.re-auth-if-jenkins-restart` setup cluster auth
        ```bash
        gcloud container clusters get-credentials [CLUSTER_NAME] --zone=asia-east2-a
        ```
    - use Job `tools-deploy-foundation` deploy lagerlog, internal_service, external_service, fluentd_client and auth_proxy

11. 開通防火牆讓 GKE 跟 GCE 能夠溝通
    >on cloudshell

    **查詢 gke 自動生成 xxx-all 名稱**
    ```bash
    gcloud compute firewall-rules list
    gcloud compute firewall-rules describe [GKE_FIREWALL_NAME] --format="value(sourceRanges)"
    ```

    **開通防火牆** 
    如果 `gke-to-gce` 不存在, 使用 create

    ```bash
    gcloud compute firewall-rules create gke-to-gce \
    --allow tcp \
    --direction=INGRESS \
    --source-ranges=[上面步驟查詢的值]
    ```

    如果 `gke-to-gce` 存在, 使用 update
    ```bash
    gcloud compute firewall-rules update gke-to-gce \
    --source-ranges=[上面步驟查詢的值]
    ```

## Actual For Change Cluster day

### Use Old Auth to Down all Pods (On Jenkins)

1. Set Jenkins terminal with old Cluster

    edit Job `0.re-auth-if-jenkins-restart` setup cluster auth
    
    ```bash
    gcloud container clusters get-credentials [CLUSTER_NAME] --zone=asia-east2-a
    ```

2. Use Prod Jenkins to down all pods on `sport-betting`

### Special delete for old GKE SSL setting (On CloudShell)

1. 刪除 `forwarding-rules` for release IP
    ```bash
    gcloud compute forwarding-rules list 
    gcloud compute forwarding-rules delete https-content-rule --global
    ```

2. 刪除 `target-https-proxies`, `url-maps`. **這個步驟可以在全部完成後刪除**
    ```bash
    gcloud compute target-https-proxies list
    gcloud compute target-https-proxies delete https-lb-proxy
    gcloud compute url-maps list
    gcloud compute url-maps delete https-lb
    ```

### Setup SSL for new Cluster (On CloudShell)

1. 新增外部IP,給HTTPS Load Balancer使用 **(之前做過的，就不需要)**
   或是將現在使用的 release 出來準備轉交給新的 cluster

    gcloud compute addresses create **IP_NAME** \
    --global

    **Example:**

    ```sh
    gcloud compute addresses create lb-ssl-gke \
    --global
    ```

2. 創建自簽憑證 **(之前做過的，就不需要)**

    創建一個可執行腳本檔並執行

    **Example:**

    ```sh
    #!/usr/bin/env bash
    mkdir ~/ssl/
    cd ~/ssl/

    cat <<EOF> root.cnf
        [req]
        default_bits = 2048
        prompt = no
        default_md = sha256
        distinguished_name = dn

        [dn]
        C = hk
        ST = hk
        L = hk
        O = testcert
        OU = testcert
        emailAddress = testcert@gg.com.tw
        
    EOF

    openssl req -x509 -new -nodes -keyout ~/ssl/rootCA.key -sha256 -days 1024 -out ~/ssl/rootCA.pem -config root.cnf

    cat <<EOF> openssl.cnf
        [req]
        default_bits = 2048
        prompt = no
        default_md = sha256
        distinguished_name = dn

        [dn]
        C = hk
        ST = hk
        L = hk
        O = testcert
        OU = testcert
        emailAddress = testcert@gg.com.tw
        CN = testcert
        
    EOF

    cat <<EOF> v3.ext
        authorityKeyIdentifier=keyid,issuer
        basicConstraints=CA:FALSE
        keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
        subjectAltName = @alt_names

        [alt_names]
        DNS.1 = localhost
        DNS.2 = *.localhost
    EOF

    openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key -config openssl.cnf
    
    sudo openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 397 -sha256 -extfile v3.ext
    ```

3.  將自簽憑證交給GCP管理 **(之前做過的，就不需要)**

    gcloud compute ssl-certificates create **CERTIFICATE_NAME** \
    --certificate=**CERTIFICATE_FILE** \
    --private-key=**PRIVATE_KEY_FILE** \
    --global

    **Example:**

    ```bash
    gcloud compute ssl-certificates create ssl-cert-test \
    --certificate=./ssl/server.crt \
    --private-key=./ssl/server.key \
    --global
    ```


4. Istio 新增 HttpLoadBalancing addons

    gcloud container clusters update **CLUSTER_NAME** \
    --update-addons=HttpLoadBalancing=ENABLED \
    --zone **ZONE**

    **Example:**

    ```sh
    gcloud container clusters update [CLUSTER_NAME] \
    --update-addons=HttpLoadBalancing=ENABLED \
    --zone asia-east2-a
    ```

5. 更新 Istio-ingressgateway ,添加 NEG 設定

    **Example:**

    ```bash
    cat <<EOF > ingress-service-patch.yaml
    kind: Service
    metadata:
        name: istio-ingressgateway
        namespace: istio-system
        annotations:
            cloud.google.com/backend-config: '{"default": "ingress-backendconfig"}'
            cloud.google.com/app-protocols: '{"https":"HTTPS","http2":"HTTP"}'
            cloud.google.com/neg: '{"ingress": true}'
    EOF
    kubectl patch svc istio-ingressgateway -n istio-system --patch "$(cat ingress-service-patch.yaml)"
    ```

6. 新增 GKE Health Check

    **Example:**

    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: cloud.google.com/v1
    kind: BackendConfig
    metadata:
        name: ingress-backendconfig
        namespace: istio-system
    spec:
        healthCheck:
            requestPath: /healthz/ready
            port: 15020
            type: HTTP
    EOF

    cat <<EOF | kubectl apply -f -
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      namespace: istio-system
      name: health
    spec:
      gateways:
      - health-gateway
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
          authority: istio-ingressgateway.istio-system.svc.cluster.local:15020
          uri: /healthz/ready
        route:
        - destination:
            host: istio-ingressgateway.istio-system.svc.cluster.local
            port:
              number: 15020
    EOF

    cat <<EOF | kubectl apply -f -
    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      namespace: istio-system
      name: health-gateway
    spec:
      selector:
        istio: ingressgateway
      servers:
      - port: 
          number: 80
          name: http
          protocol: HTTP
        hosts:
        - "*"
    EOF
    ```


7. 新增 Ingress 設定綁定 static ip & GCP cert

    **For GKE version before 1.18**
    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
        name: gke-ingress
        namespace: istio-system
        annotations:
            kubernetes.io/ingress.allow-http: "false"
            kubernetes.io/ingress.global-static-ip-name: "lb-ssl-gke"
            ingress.gcp.kubernetes.io/pre-shared-cert: "ssl-cert"
    spec:
        backend:
            serviceName: istio-ingressgateway
            servicePort: 80
    EOF
    ```
    **For GKE version  1.19**
    ```bash
    cat <<EOF | kubectl apply -f -
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
        name: gke-ingress
        namespace: istio-system
        annotations:
            kubernetes.io/ingress.allow-http: "false"
            kubernetes.io/ingress.global-static-ip-name: "lb-ssl-gke"
            ingress.gcp.kubernetes.io/pre-shared-cert: "ssl-cert"
    spec:
        defaultBackend:
            service:
                name: istio-ingressgateway
                port:
                    number: 80
    EOF
    ```

8. 調整防火牆，將 targetTags 為舊的 GKE Cluster Node 調整成新的  GKE Cluster Node
    ```bash
    # List 所有 firewall-rules
    gcloud compute firewall-rules list
    # 用舊的 GKE 查詢 targetTags 
    gcloud compute firewall-rules describe [GKE_FIREWALL_ALL_NAME] --format="value(targetTags)"
    # List 所有 firewall-rules 的 targetTags 是舊的 GKE
    gcloud compute firewall-rules list --filter=上面查出來的[NODE_TARGET_NAME]
    # 用新的 GKE 查詢 targetTags 
    gcloud compute firewall-rules describe [GKE_FIREWALL_ALL_NAME] --format="value(targetTags)"
    ```

    確認下列的 firewall-rules 是否都存在，且必須將這些規定的 targetTag 轉移到新的 Cluster Node
    - cleanroom-allow-gke-node-port-browsing
    - deploy-allow-gke-node-port-browsing
    - ssl-lb
    - ssl-lb-monitor-gke

    ```bash
    gcloud compute firewall-rules update cleanroom-allow-gke-node-port-browsing \
    --target-tags=[NODE_TARGET_NAME]

    gcloud compute firewall-rules update deploy-allow-gke-node-port-browsing \
    --target-tags=[NODE_TARGET_NAME]

    gcloud compute firewall-rules update ssl-lb \
    --target-tags=[NODE_TARGET_NAME]

    gcloud compute firewall-rules update ssl-lb-monitor-gke \
    --target-tags=[NODE_TARGET_NAME]
    ```

### Deploy to New Cluster (On Jenkins)

1. Jenkins edit Job `0.re-auth-if-jenkins-restart` setup new cluster auth

    ```bash
    gcloud container clusters get-credentials [CLUSTER_NAME] \
    --zone asia-east2-a \
    --project [PROJECT_ID]
    ```

2. Use Jenkins to deploy all pods on `sport-betting`


### Update Monitor Loadbalance Setting with New Cluster (On CloudShell)

1. 將 New GKE 的 Instance Group 綁定 Monitor 的 NodePort
    ```bash
    gcloud compute instance-groups list

    gcloud compute instance-groups set-named-ports **INSTANCE_NAME** \
    --named-ports="grafana:31874,jaeger:31717,consul:30729,largelog:30292" \
    --zone=asia-east2-a
    ```

2. 刪除舊的 Monitor 的 Backend-Service
   
    **Example:**

    ```bash
    gcloud compute backend-services list #查詢 backend 使用的backends

    gcloud compute backend-services remove-backend grafana \
    --instance-group=[INSTANCE_NAME] \
    --instance-group-zone=asia-east2-a \
    --global

    gcloud compute backend-services remove-backend jaeger \
    --instance-group=[INSTANCE_NAME] \
    --instance-group-zone=asia-east2-a \
    --global

    gcloud compute backend-services remove-backend consul \
    --instance-group=[INSTANCE_NAME] \
    --instance-group-zone=asia-east2-a \
    --global

    gcloud compute backend-services remove-backend largelog \
    --instance-group=[INSTANCE_NAME] \
    --instance-group-zone=asia-east2-a \
    --global
    ```

3. 將新的 Network Endpoint Group 更新到 Backend-Service

    gcloud compute backend-services update-backend **BACKEND_SERVICE** \
    [--network-endpoint-group=NETWORK_ENDPOINT_GROUP] \
    [--network-endpoint-group-zone=ZONE] \
    --global \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=5

    **Example:**

    ```bash
    gcloud compute backend-services create grafana --protocol=HTTP --port-name=grafana --health-checks=http-basic-check --global
    gcloud compute backend-services create jaeger  --protocol=HTTP --port-name=jaeger --health-checks=http-basic-check --global
    gcloud compute backend-services create consul  --protocol=HTTP --port-name=consul --health-checks=http-basic-check --global
    gcloud compute backend-services create largelog --protocol=HTTP --port-name=largelog --health-checks=http-basic-check --global
    
    gcloud compute instance-groups list #查詢新的 instance-group name

    gcloud compute backend-services update-backend grafana \
    --instance-group=[NEW_INSTANCE_GROUP_NAME] \
    --instance-group-zone=asia-east2-a \
    --global

    gcloud compute backend-services update-backend jaeger \
    --instance-group=[NEW_INSTANCE_GROUP_NAME] \
    --instance-group-zone=asia-east2-a \
    --global

    gcloud compute backend-services update-backend consul \
    --instance-group=[NEW_INSTANCE_GROUP_NAME] \
    --instance-group-zone=asia-east2-a \
    --global

    gcloud compute backend-services update-backend largelog \
    --instance-group=[NEW_INSTANCE_GROUP_NAME] \
    --instance-group-zone=asia-east2-a \
    --global
    ```