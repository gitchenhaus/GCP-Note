# Cloud CDN 設定

利用 Cloud Storage 建立 Cloud CDN

## Cloud Storage 建立

> 以下的語法全部都是在`cloud shell`執行

1. 建立Cloud Storage

    gsutil mb -p **PROJECT_ID** -c standard -l **ZONE** -b on gs://**BUCKET_NAME**

    **Prod Example**

    ```bash
    gsutil mb -p vqzopdxbwntwtjv -c standard -l asia-east2 -b on gs://rexbet-cdn
    ```

2. 公開 Cloud Storage 的存儲分區

    gsutil iam ch allUsers:objectViewer gs://**BUCKET_NAME**

    **Prod Example**

    ```bash
    gsutil iam ch allUsers:objectViewer gs://rexbet-cdn
    ```

3. 將文件複製到 Cloud Storage 存儲分區

    **Cloud Storage Sync**

    gsutil cp gs://**SOURCE** gs://**BUCKET_NAME**/**TARGET**

## 建立外部 IP 地址

1. 建立外部 IP for CDN

    gcloud compute addresses create **IP_NAME** \
        --network-tier=PREMIUM \
        --ip-version=IPV4 \
        --global

    **Prod Example**

    ```bash
    ## 建立外部IP
    gcloud compute addresses create lb-public-cdn \
        --network-tier=PREMIUM \
        --ip-version=IPV4 \
        --global

    ## 查詢IP
    gcloud compute addresses describe lb-public-cdn \
        --format="get(address)" \
        --global
    ```

## 建立外部 HTTP(S) 負載平衡器

### Http 版本

1. 配置後端

    gcloud compute backend-buckets create **BACKEND_BUCKET_NAME** \
        --gcs-bucket-name=**BUCKET_NAME** \
        --enable-cdn

    **Prod Example**

    ```bash
    gcloud compute backend-buckets create public-cdn \
        --gcs-bucket-name=rexbet-public-cdn \
        --enable-cdn
    ```

2. 配置 URL Map

    gcloud compute url-maps create **URL_MAPS_NAME** \
        --default-backend-bucket=**BACKEND_BUCKET_NAME**

    **Prod Example**

    ```bash
    gcloud compute url-maps create public-cdn \
        --default-backend-bucket=lb-public-cdn
    ```

3. 配置目標代理 (public-cdn-target-prxoy)

    gcloud compute target-http-proxies create **TARGET_HTTP_PROXIES_NAME** \
    --url-map=**URL_MAP_NAME**

    **Prod Example**

    ```bash
    gcloud compute target-http-proxies create public-cdn-http-target-prxoy \
        --url-map=public-cdn
    ```

4. 配置轉發規則

    **Prod Example**

    ```bash
    gcloud compute forwarding-rules create public-cdn-forwarding-rule \
        --address=lb-public-cdn \
        --global \
        --target-http-proxy=public-cdn-http-target-prxoy \
        --ports=80
    ```


### Https 版本

1. 配置後端

    gcloud compute backend-buckets create **BACKEND_BUCKET_NAME** \
        --gcs-bucket-name=**BUCKET_NAME** \
        --enable-cdn

    **Prod Example**

    ```bash
    gcloud compute backend-buckets create public-cdn \
        --gcs-bucket-name=rexbet-public-cdn \
        --enable-cdn
    ```

2. 配置 URL Map

    gcloud compute url-maps create **URL_MAPS_NAME** \
        --default-backend-bucket=**BACKEND_BUCKET_NAME**

    **Prod Example**

    ```bash
    gcloud compute url-maps create public-cdn \
        --default-backend-bucket=lb-public-cdn
    ```

3. 配置目標代理

    gcloud compute target-https-proxies create **TARGET_HTTPS_PROXIES_NAME** \
        --url-map **URL_MAP_NAME** \
        --ssl-certificates **CERT_NAME** \
        --global-ssl-certificates \
        --global-url-map 

    **Prod Example**

    ```bash
    gcloud compute target-https-proxies create public-cdn-target-prxoy \
        --url-map public-cdn \
        --ssl-certificates rexbet-public-cdn \
        --global-ssl-certificates \
        --global-url-map 
    ```

4. 配置轉發規則

    **Prod Example**

    ```bash
    gcloud compute forwarding-rules create public-cdn-forwarding-rule \
        --address=lb-public-cdn \
        --global \
        --target-https-proxy=public-cdn-target-prxoy \
        --ports=443
    ```

## 測試

1. 將文件擺放到 Cloud Storage 的空間內後就可以執行
   curl http://**IP**/**File_PATH** or https://**IP**/**File_PATH**

2. 檢查 Logs
   Access -> Logging -> Logs Explorer -> Resouce Type(Cloud HTTP Load Balancer)
   檢查 **jsonPayload** 有沒有出現 `send_by_cached`