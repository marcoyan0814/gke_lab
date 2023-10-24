# gke_lab
存放 gke 部署相關資料  
GKE ver: 1.27.3

# 目錄說明
```
├── 01-dashboard # 官方 k8s dashboard yaml
├── 02-pd-nfs-pvc-nginx-rwx-only-standard # 建立PD及NFS Server可多點存取RWX
├── 03-dep-hpa-ingress # nginx+autoscale+mount nfs+ingress
├── 04-dep-hpa-svc # nginx+autoscale+mount nfs+service
├── 05-laravel-nginx-php81-docker # php8.1-fpm + nginx + laravel dockerfile
├── 06-nginx-php81-docker # alpin + nginx + php-fpm 8.1 + mysql client dockerfile
```

# 常用指令  
## 安裝 gcloud cli
https://cloud.google.com/sdk/docs/install
```
# 檢查帳號
gcloud auth list
# 顯示目前配置
gcloud config list
# 如需检查您是否安装了 gcloud beta 组件，请运行以下命令：
gcloud components list
# 進入 gcloud 交互模式
gcloud beta interactive
```

## 建立 docker image 及上傳
```
# 建立 image
docker build . -t marcoyan0814/nginx-php81 --no-cache

# 測試掛載 rm 是停止後刪除，此將 host 82 port → container 80 port
docker run --name test -d marcoyan0814/nginx-php81 -p 82:80 --rm

# 設定tag
docker tag nginx-php81(原本名稱) marcoyan0814/nginx-php81
docker images marcoyan0814/nginx-php81
docker login
docker push marcoyan0814/nginx-php81

# 刪除沒用到的images
docker rmi $(docker images -q)
```

## Rocky Linux 安裝 K9s
```
# 安裝更新套件
sudo yum install epel-release
# snap安裝及設定
sudo yum install snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
# 安裝k9s
sudo snap install k9s
# 建立捷徑
sudo ln -s /snap/k9s/current/bin/k9s /snap/bin/
```

## GKE Nginx Ingress + Cert 憑證
必需要先綁定ingress申請憑證才會通過
```
# 使用Google建立憑證(最慢約需60分鐘)
# 查詢憑證指令 kubectl describe managedcertificate managed-cert
# 列出憑證 gcloud beta compute ssl-certificates list
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: managed-cert
spec:
  domains:
    - frontend.abc.com
    - backend.abc.com

# INGRESS 必需要先綁定ingress申請憑證才會通過
annotations:
    # 使用 GCE Ingress
    kubernetes.io/ingress.class: gce
    # global 的 GCE IP
    kubernetes.io/ingress.global-static-ip-name: gke-web-ip
    # 使用 Google 建立的憑證
    networking.gke.io/managed-certificates: managed-cert
    service.beta.kubernetes.io/external-traffic: OnlyLocal

# 查詢憑證發放狀態
kubectl describe ManagedCertificate managed-cert
# 列出憑證
gcloud beta compute ssl-certificates list
```

## 建立 GKE Cluster
```
# 授權
gcloud services enable container.googleapis.com --project yourprojectname
# 建立cluster standard
gcloud container clusters create "test-gke" \
    --region "asia-east1" \
    --machine-type "e2-micro" \
    --disk-type "pd-balanced" \
    --disk-size "50" \
    --num-nodes "1" \
    --node-locations "asia-east1-b" \
    --project "yourprojectname"

# 建立 cluster autopilot
gcloud container clusters create-auto test-gke \
    --location=asia-east1 \
    --project=yourprojectname

# 授權連線
gcloud container clusters get-credentials test-gke --region asia-east1 --project yourprojectname

# 讓private cluster 內部node使用同一個IP連線
# 建立私人叢集
# 將外部要連線到API的IP加到gke裡
gcloud container clusters create-auto "test-gke" \
    --project "yourprojectname" \
    --region "asia-east1" \
    --release-channel "regular" \
    --enable-private-nodes \
    --enable-master-authorized-networks \
    --master-authorized-networks 106.104.126.60/32 \
    --network "projects/yourprojectname/global/networks/default" \
    --subnetwork "projects/yourprojectname/regions/asia-east1/subnetworks/default" \
    --cluster-ipv4-cidr "/17"

# 建立 cloud nat 並綁定外部 IP
gcloud compute routers create nat-router --region asia-east1 --network default --project=yourprojectname
gcloud compute routers nats create nat-config --router=nat-router --region=asia-east1 --nat-all-subnet-ip-ranges --project=yourprojectname --nat-external-ip-pool=test-gke-ext-ip
```

## 常用指令
```
# 連線授權使用GKE主機
gcloud container clusters get-credentials test-gke --region asia-east1 --project yourprojectname

# 套用 yaml
kubectl apply -f xxxx.yaml

# 修改已佈署資源
kubectl edit -f xxxx.yaml

# 刪除 yaml 內的資源
kubectl delete -f xxxx.yaml

# 查詢
watch -d kubectl get ns,deployment,pod,pd,pv,pvc,hpa,svc,ingress -o wide

# 對 pod 執行
kubectl exec -it nginx-web-58df6cc8d5-skc5g -- bash

# 查詢資源內容
kubectl describe pod nginx-web-58df6cc8d5-skc5g

# 顯示資訊
kubectl top pod web-nginx-77847b8499-4lctf

# 建立pod
kubectl run nginx --image=nginx

# 建立service
kubectl expose deployment laravel-kubernetes-demo --type=NodePort --port=80 --target-port=80

# 對服務掛載loadbalancer
kubectl expose deployment nginx-app --type="LoadBalancer"
```