# GCP Add HTTPS Cloud LoadBalancer

1. 該方案採用HTTPS Load Balancer 因此為L7模式,無法使用443以外Port
2. 使用該方案必須將後端服務都以Instance Group模式建立(GKE預設也是採用該方式).
3. 會需要額外設定信任Root憑證
4. 憑證有效期為397(13個月)

## 操作步驟(腳本)

1. Create GKE

    > 如果已經有GKE,可以跳過該步驟

    ```sh
    gcloud beta container clusters create cluster-1 \
        --addons=Istio --istio-config=auth=MTLS_PERMISSIVE \
        --machine-type=n1-standard-2 \
        --release-channel rapid \
        --cluster-version 1.18.12-gke.1200 \
        --zone asia-east2-a \
        --network=default \
        --num-nodes=3 \
        --node-labels=application=enable
    ```

2. 新增外部IP,給HTTPS Load Balancer使用

    ```sh
    gcloud compute addresses create <ip-name> \
    --global
    ```

    Example:

    ```sh
    gcloud compute addresses create lb-https \
    --global
    ```

3. 創建Https Cloud LoadBalancer健康檢查

    ```sh
    gcloud compute health-checks create http <health-check-name> --port 10256 --request-path="/healthz" --global
    ```

    Example:

    ```sh
    gcloud compute health-checks create http http-basic-check --port 10256 --request-path="/healthz" --global
    ```

4. 創建後端服務

    ```sh
    gcloud compute backend-services create <backend-services-name> \
    --protocol=HTTP \
    --port-name=http \
    --health-checks=<health-check-name(步驟3)> \
    --global
    ```

    Example:

    ```sh
    gcloud compute backend-services create gke-cluster \
    --protocol=HTTP \
    --port-name=http \
    --health-checks=http-basic-check \
    --global
    ```

5. 確認GKE-Cluster instance group

    ```sh
    gcloud compute instance-groups list
    ```

6. 確認目前GKE-Istio-Ingressgateway 使用哪種Port

    ```sh
    kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}' 
    ```

7. 添加後端服務Port 到instance group

    ```sh
    gcloud compute instance-groups set-named-ports {{可以參照步驟5執行結果-NAME}} --named-ports=http:{{請使用步驟6的結果}} --zone={{可以參照步驟5執行結果-LOCATION}}
    ```

    Example:

    ```sh
    gcloud compute instance-groups set-named-ports gke-cluster-1-default-pool-fca4a109-grp --named-ports=http:31919 --zone=asia-east2-a
    ```

8. 將GKE Instance Group 綁到後端服務

    ```sh
    gcloud compute backend-services add-backend <backend-service-name(步驟4)> \
    --instance-group={{可以參照步驟5執行結果-Name}} \
    --instance-group-zone={{可以參照步驟執行結果-LOCATION}}
    --global
    ```

    Example:

    ```sh
    gcloud compute backend-services add-backend gke-cluster \
    --instance-group=gke-cluster-1-default-pool-fca4a109-grp  \
    --instance-group-zone=asia-east2-a \
    --global
    ```

9.  創建url-maps

    ```sh
    gcloud compute url-maps create <url-maps-name> \
        --default-service <backend-service-name(步驟4)> \
        --global
    ```

    Example:

    ```sh
    gcloud compute url-maps create https-lb \
        --default-service gke-cluster \
        --global
    ```

10. 創建自簽憑證

    請創建一個可執行腳本檔並執行

    檔案內容如下:

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
        O = rexbet
        OU = rexbet
        emailAddress = rexbet@gg.com.tw
        
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
        O = rexbet
        OU = rexbet
        emailAddress = rexbet@gg.com.tw
        CN = rexbet
        
    EOF

    cat <<EOF> v3.ext
        authorityKeyIdentifier=keyid,issuer
        basicConstraints=CA:FALSE
        keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
        subjectAltName = @alt_names

        [alt_names]
        DNS.1 = jpi74.com
        DNS.2 = *.jpi74.com
        DNS.3 = gdo74.com
        DNS.4 = *.gdo74.com
        DNS.5 = qca28.com
        DNS.6 = *.qca28.com
        DNS.7 = api.cda15.com 
        DNS.8 = *.cda15.com 
        DNS.9 = api.tje71.com 
        DNS.10 = *.tje71.com 
        DNS.11 = qye92.com
        DNS.12 = *.qye92.com
        DNS.15 = localhost
        DNS.16 = *.localhost
    EOF

    openssl req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key -config openssl.cnf
    
    sudo openssl x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 397 -sha256 -extfile v3.ext
    ```

11. 下載憑證並在Mac設定為Trust

    ```sh
    cloudshell download <pemfile-path>
    ```

    Example:

    ```sh
    cloudshell download ~/ssl/rootCA.pem
    ```

    > 需要回到MacOS執行下列語法

    ```sh
    sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain <從cloud-shell下載回來的檔案>
    ```

    Example:

    ```sh
    sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain rootCA.pem
    ```

12. 將自簽憑證交給GCP管理

    ```sh
    gcloud compute ssl-certificates create <CERTIFICATE_NAME> \
    --certificate=<CERTIFICATE_FILE> \
    --private-key=<PRIVATE_KEY_FILE> \
    --global
    ```

    Example:

    ```sh
    gcloud compute ssl-certificates create ssl-cert \
    --certificate=./ssl/server.crt \
    --private-key=./ssl/server.key \
    --global
    ```

13. 將自簽憑證加到Https Load Balancer

    ```sh
    gcloud compute target-https-proxies create <target-https-proxies-name> \
        --url-map <url-maps-name(步驟9)> --ssl-certificates <ssl-name(步驟11)> \
        --global-ssl-certificates \
        --global-url-map 
    ```

    Example:

    ```sh
     gcloud compute target-https-proxies create https-lb-proxy \
        --url-map https-lb --ssl-certificates ssl-cert \
        --global-ssl-certificates \
        --global-url-map 
    ```

14. 創建轉發規則將Https Load Balancer 連接到後端服務

    ```sh
    gcloud compute forwarding-rules create <forwarding-rules-name> \
    --target-https-proxy=<target-https-proxy-name(步驟12)> \
    --global-target-https-proxy \
    --ports=443 \
    --address=<address-ip(步驟2)>\
    --global-address \
    --global
    ```

    Example:

    ```sh
    gcloud compute forwarding-rules create https-content-rule \
    --target-https-proxy=https-lb-proxy \
    --global-target-https-proxy \
    --ports=443 \
    --address=lb-https\
    --global-address \
    --global
    ```

15. 確認目前防火牆

    > 需要k8s-fw 開頭的 firewall-rules, 以及 targetTags

    ```sh
    gcloud compute firewall-rules list
    gcloud compute firewall-rules describe {{firewall-rule-name}} --format="value(targetTags.scope())"
    ```

    Example:

    ```sh
    gcloud compute firewall-rules list
    gcloud compute firewall-rules describe k8s-fw-a8421eea6a3d74c5780fe485b85e5688  --format="value(targetTags.scope())"
    ```

16. 新增防火牆

    ```sh
    gcloud compute firewall-rules create <firewall-name> \
    --allow tcp:{{請使用步驟6的結果}} \
    --direction=INGRESS \
    --source-ranges=0.0.0.0/0 \
    --target-tags={{請使用步驟14的結果}}
    ```

    Example:

    ```sh
    gcloud compute firewall-rules create ssl-lb \
    --allow tcp:31919 \
    --direction=INGRESS \
    --source-ranges=0.0.0.0/0 \
    --target-tags=gke-cluster-1-91c28054-node
    ```

17. 部署測試用Nginx

    ```sh
    kubectl apply -f test-nginx.yaml
    ```

18. 測試語法

    ```sh
    curl -k -H 'Host:bo.qca28.com' https://{ssl-load balancer ip} -i -v 
    curl -H 'Host:bo.qca28.com' http://{istio balancer ip} -i -v 
    ```

    Example:

    ```sh
    curl -k -H 'Host:bo.qca28.com' https://34.120.254.125 -i -v 
    curl -H 'Host:bo.qca28.com' http://34.92.209.122 -i -v
    ```
