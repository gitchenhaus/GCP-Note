# Cloud CDN Purge 檔案

### 首先先查詢CDN的LoadBalancer Map Name

Ex:
```
gcloud compute url-maps list
```

### 1. 僅Purge一個文件


Ex:
```
gcloud compute url-maps invalidate-cdn-cache URL_MAP_NAME \
    --path "/images/foo.jpg"
```

### 2. Purge整個目錄


Ex:
```
gcloud compute url-maps invalidate-cdn-cache URL_MAP_NAME \
    --path "/images/*"


```

### 3. Purge整個bucket

Ex:
```
gcloud compute url-maps invalidate-cdn-cache URL_MAP_NAME \
    --path "/*"

```
