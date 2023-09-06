# **Multi-cluster Gateway ハンズオン**

本ハンズオンでは、[Multi-cluster Gateway](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways?hl=ja) を使って複数リージョンにデプロイした GKE クラスタ間のトラフィックルーティングを行う方法について学びます。

- 
- 
![Multi Cluster Gateway](https://cloud.google.com/static/kubernetes-engine/images/multi-cluster-gateway-resource.svg)

## Google Cloud プロジェクトの設定、確認

### **1. 対象の Google Cloud プロジェクトを設定**

ハンズオンを行う Google Cloud プロジェクトのプロジェクト ID を環境変数に設定し、以降の手順で利用できるようにします。 (右辺の `testproject` を手動で置き換えてコマンドを実行します)

```bash
export PROJECT_ID=testproject
```

`プロジェクト ID` は [ダッシュボード](https://console.cloud.google.com/home/dashboard) に進み、左上の **プロジェクト情報** から確認します。

### **2. プロジェクトの課金が有効化されていることを確認する**

```bash
gcloud beta billing projects describe ${PROJECT_ID} \
  | grep billingEnabled
```

**Cloud Shell の承認** という確認メッセージが出た場合は **承認** をクリックします。

出力結果の `billingEnabled` が **true** になっていることを確認してください。

**false** の場合は、こちらのプロジェクトではハンズオンが**進められません**。別途、課金を有効化したプロジェクトを用意し、本ページの #1 の手順からやり直してください。

## **環境準備**

最初に、ハンズオンを進めるための環境準備を行います。

下記の設定を進めていきます。

- gcloud コマンドラインツール設定
- Google Cloud 機能（API）有効化設定

## **gcloud コマンドラインツール**

Google Cloud は、コマンドライン（CLI）、GUI から操作が可能です。ハンズオンでは作業の自動化を目的に主に CLI を使い作業を行います。

### **1. gcloud コマンドラインツールとは?**

gcloud コマンドライン インターフェースは、Google Cloud でメインとなる CLI ツールです。これを使用すると、多くの一般的なプラットフォーム タスクを様々なツールと組み合わせて実行することができます。

たとえば、gcloud CLI を使用して、以下のようなものを作成、管理できます。

- Google Compute Engine 仮想マシン
- Google Kubernetes Engine クラスタ

**ヒント**: gcloud コマンドラインツールについての詳細は[こちら](https://cloud.google.com/sdk/gcloud?hl=ja)をご参照ください。

### **2. gcloud から利用する Google Cloud のデフォルトプロジェクトを設定**

gcloud コマンドでは操作の対象とするプロジェクトの設定が必要です。操作対象のプロジェクトを設定します。

```bash
gcloud config set project ${PROJECT_ID}
```

承認するかどうかを聞かれるメッセージがでた場合は、`承認` ボタンをクリックします。

## **Google Cloud 環境設定**

Google Cloud では利用したい機能（API）ごとに、有効化を行う必要があります。
ここでは、以降のハンズオンで利用する機能を事前に有効化しておきます。

```bash
gcloud services enable \
    container.googleapis.com \
    gkehub.googleapis.com \
    multiclusterservicediscovery.googleapis.com \
    multiclusteringress.googleapis.com \
    trafficdirector.googleapis.com
```

**GUI**: [API ライブラリ](https://console.cloud.google.com/apis/library)
<walkthrough-footnote>必要な機能が使えるようになりました。次に Multi-cluster Gateway を使って 2 つの GKE クラスタ間のトラフィックを制御する方法を学びます。</walkthrough-footnote>

## **参考: Cloud Shell の接続が途切れてしまったときは?**

一定時間非アクティブ状態になる、またはブラウザが固まってしまったなどで `Cloud Shell` が切れてしまいブラウザのリロードが必要になる場合があります。その場合は再接続、リロードなどを実施後、以下の対応を行い、チュートリアルを再開してください。

### **1. チュートリアル資材があるディレクトリに移動する**

```bash
cd ~/gcp-getting-started-lab-jp/workstations/
```

### **2. チュートリアルを開く**

```bash
teachme tutorial_ws.md
```

### **3. プロジェクト ID を設定する**

ハンズオンを行う Google Cloud プロジェクトのプロジェクト ID を環境変数に設定し、以降の手順で利用できるようにします。 (右辺の `testproject` を手動で置き換えてコマンドを実行します)

```bash
export PROJECT_ID=testproject
```

### **4. gcloud のデフォルト設定**

```bash
gcloud config set project ${PROJECT_ID}
```

途中まで進めていたチュートリアルのページまで `Next` ボタンを押し、進めてください。




## **GKE Autopilot クラスタのデプロイ**
### **環境変数の設定**
本ハンズオンでは Multi-cluster Gateway を使って `gke-tokyo` と `gke-osaka` という 2 つのクラスタ間のトラフィックを制御します。 `gke-tokyo` は東京(`asia-northeast1`) 上に構築し、`gke-osaka` は大阪(`asia-northeast2`) 上に構築します。  
まずクラスタ構築に必要となる環境変数を設定します。
```bash
export PROJECT_NUM=`gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)"`
export REGION1=asia-northeast1
export REGION2=asia-northeast2
export CLUSTER_NAME1=gke-tokyo
export CLUSTER_NAME2=gke-osaka
export CLUSTER_VERSION=1.26 #1.26.5
```

### **GKE Autopilot クラスタのデプロイ**
まず `gke-tokyo` をデプロイします。デプロイ完了まで数分かかります。
```bash
gcloud container clusters create-auto ${CLUSTER_NAME1} \
    --location=${REGION1} \
    --release-channel=stable \
    --cluster-version=${CLUSTER_VERSION}
```

次に `gke-osaka` をデプロイします。デプロイ完了まで数分かかります。
```bash
gcloud container clusters create-auto ${CLUSTER_NAME2} \
    --location=${REGION2} \
    --release-channel=stable \
    --cluster-version=${CLUSTER_VERSION}
```

### **コンテキストの設定**
あとから操作しやすいように、各クラスタのコンテキストを変更しておきます。
```bash
kubectx ${CLUSTER_NAME1}=gke_${PROJECT_ID}_${REGION1}_${CLUSTER_NAME1}
kubectx ${CLUSTER_NAME2}=gke_${PROJECT_ID}_${REGION2}_${CLUSTER_NAME2}
```

## Fleet への登録
Multi-cluster Service を有効にするため、`gke-osaka` クラスタを Fleet へ登録します。
```bash
gcloud container fleet memberships register ${CLUSTER_NAME2} \
  --gke-cluster ${REGION2}/${CLUSTER_NAME2} \
  --enable-workload-identity \
  --project ${PROJECT_ID}
```

Fleet にクラスタが登録されていることを確認します。
```bash
gcloud container fleet memberships list --project ${PROJECT_ID}
```

以下のように表示されれば OK です。
```text
gcloud container fleet memberships list --project ${PROJECT_ID}
NAME             EXTERNAL_ID                           LOCATION
gke-tokyo        3111cde5-8ede-43fe-9b9b-d81b8c11e65b  asia-northeast1
gke-osaka        341bbc08-c5ba-4181-befc-466bc807532c  asia-northeast2
```

## Multi-cluster Services (MCS) の有効化
複数クラスタ間で Kubernetes Service を共有できるように、Multi-cluster Services (MCS) を有効化します。
```bash
gcloud container hub multi-cluster-services enable
```

MCS のサービスアカウントに必要な IAM 権限を付与します。
```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member "serviceAccount:${PROJECT_ID}.svc.id.goog[gke-mcs/gke-mcs-importer]" \
    --role "roles/compute.networkViewer"
```

MCS の設定が各クラスタで有効になっているか確認します。
```bash
gcloud container hub multi-cluster-services describe
```

以下のように `gke-tokyo` と `gke-osaka` の status が OK となっていれば大丈夫です。
```text
gcloud container hub multi-cluster-services describe
createTime: '2023-09-06T02:10:54.891172224Z'
membershipStates:
  projects/1072516076243/locations/asia-northeast1/memberships/gke-tokyo:
    state:
      code: OK
      description: Firewall successfully updated
      updateTime: '2023-09-06T02:11:39.489022075Z'
  projects/1072516076243/locations/asia-northeast2/memberships/gke-osaka:
    state:
      code: OK
      description: Firewall successfully updated
      updateTime: '2023-09-06T02:11:35.309992770Z'
name: projects/kkuchima-sandbox/locations/global/features/multiclusterservicediscovery
resourceState:
  state: ACTIVE
```

## **Multi-cluster Gateway Controller の有効化**
複数クラスタ間での負荷分散設定を行うコントローラとして、Multi-cluster Gateway Controller をデプロイします。  
今回は Controller のデプロイ先である構成クラスタとして `gke-tokyo` を選択します。
```bash
gcloud container fleet ingress enable \
    --config-membership=projects/${PROJECT_ID}/locations/${REGION1}/memberships/${CLUSTER_NAME1} \
    --project=${PROJECT_ID}
```

Multi-cluster Gateway が各クラスタで有効になっているか確認します。
```bash
gcloud container fleet ingress describe --project=${PROJECT_ID}
```

以下のように `gke-tokyo` と `gke-osaka` の status が OK となっていれば大丈夫です。
```text
gcloud container fleet ingress describe --project=${PROJECT_ID}
createTime: '2023-09-06T02:31:27.865780905Z'
membershipStates:
  projects/1072516076243/locations/asia-northeast1/memberships/gke-tokyo:
    state:
      code: OK
      updateTime: '2023-09-06T02:31:43.310060758Z'
  projects/1072516076243/locations/asia-northeast2/memberships/gke-osaka:
    state:
      code: OK
      updateTime: '2023-09-06T02:31:43.310062278Z'
name: projects/kkuchima-sandbox/locations/global/features/multiclusteringress
resourceState:
  state: ACTIVE
spec:
  multiclusteringress:
    configMembership: projects/kkuchima-sandbox/locations/asia-northeast1/memberships/gke-tokyo
state:
  state:
    code: OK
    description: Ready to use
```

Multi-cluster Gateway 構成に必要な IAM 権限を付与します。
```bash
 gcloud projects add-iam-policy-binding ${PROJECT_ID} \
     --member "serviceAccount:service-${PROJECT_NUM}@gcp-sa-multiclusteringress.iam.gserviceaccount.com" \
     --role "roles/container.admin" \
     --project=${PROJECT_ID}
```

以下のコマンドを実行し、Multi-cluster Gateway Controller が構成クラスタ (`gke-tokyo`) にデプロイされていることを確認します。
```bash
kubectl get gatewayclasses --context ${CLUSTER_NAME1}
```

以下のように `gke-l7-global-external-managed-mc` という Gateway Class が作成されていれば OK です。mc は multi-cluster の略です。
```text
kubectl get gatewayclasses --context ${CLUSTER_NAME1}
NAME                                  CONTROLLER                    ACCEPTED   AGE
gke-l7-global-external-managed        networking.gke.io/gateway     True       23h
gke-l7-global-external-managed-mc     networking.gke.io/gateway     True       10m
gke-l7-gxlb                           networking.gke.io/gateway     True       23h
gke-l7-gxlb-mc                        networking.gke.io/gateway     True       10m
gke-l7-regional-external-managed      networking.gke.io/gateway     True       23h
gke-l7-regional-external-managed-mc   networking.gke.io/gateway     True       10m
gke-l7-rilb                           networking.gke.io/gateway     True       23h
gke-l7-rilb-mc                        networking.gke.io/gateway     True       10m
istio                                 istio.io/gateway-controller   True       22h
```

## **サンプルアプリケーションのデプロイ**
2 つのクラスタにサンプルアプリケーションをデプロイします。
```bash
kubectl apply -f manifests/store.yaml --context ${CLUSTER_NAME1}
kubectl apply -f manifests/store.yaml --context ${CLUSTER_NAME2}
```

以下コマンドを実行し、各クラスタ上に正常に Pod がデプロイできた確認します。
```bash
kubectl get pods -n store --context ${CLUSTER_NAME1}
kubectl get pods -n store --context ${CLUSTER_NAME2}
```

以下のように `store` という名前のついた Pod が Running ステータスになっていたら OK です。
```text
NAME                     READY   STATUS    RESTARTS   AGE
store-5c65bdf74f-lghk6   1/1     Running   0          9m
store-5c65bdf74f-mnk8f   1/1     Running   0          8m59s
NAME                     READY   STATUS    RESTARTS   AGE
store-5dbdf67f49-5jl8g   1/1     Running   0          8m58s
store-5dbdf67f49-th9lq   1/1     Running   0          8m58s
```

## **Multi-cluster Service 関連リソースのデプロイ**
Multi-cluster Gateway では宛先のエンドポイントとして Multi-cluster Service で公開したサービスを利用しています。  
そのため、Multi-cluster Gateway で公開したいサービスは予め MCS で共有サービスとしてエンドポイントを用意してあげる必要があります。  
MCS で利用されるリソースは以下の通りです：
- **ServiceExport** ... クラスタ内のサービスを他クラスタに公開する際に利用するリソースです。
- **ServiceImport** ... Multi-cluster Service Controller によって、自動作成されるリソースです。`ServiceExport` によって公開されたサービスを自クラスタ内に `ServiceImport` としてリソースを作成しクラスタ間でのサービスディスカバリーを実現します。

![MCS Service Discovery](https://cloud.google.com/static/kubernetes-engine/images/multi-cluster-service-example1.svg)

### **ServiceExport のデプロイ**
では、各クラスタにデプロイした `store` サービスを他のクラスタにも公開してみましょう。他クラスタからもサービスディスカバリーできるように `ServiceExport` をデプロイします。
`gke-tokyo` クラスタでは以下のように、`store` サービスと `store-tokyo` サービス 2 種類を export します。
```yaml:store-export-tokyo.yaml
apiVersion: v1
kind: Service
metadata:
  name: store
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store
  namespace: store
---
apiVersion: v1
kind: Service
metadata:
  name: store-tokyo
  namespace: store
spec:
  selector:
    app: store
  ports:
  - port: 8080
    targetPort: 8080
---
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: store-tokyo
  namespace: store
```

同様に `gke-osaka` では `store` サービスと `store-osaka` サービス 2 種類を export します。  
`store` サービスは両方のクラスタから export をしておいて、`store-**` は片方のクラスタのみ公開します。  

以下のコマンドを実行し、実際にデプロイしてみましょう。
```bash
kubectl apply -f manifests/store-export-tokyo.yaml --context ${CLUSTER_NAME1}
kubectl apply -f manifests/store-export-osaka.yaml --context ${CLUSTER_NAME2}
```

正常にデプロイできたか、以下のコマンドを実行して確かめます。
```bash
kubectl get serviceexports -n store --context ${CLUSTER_NAME1}
kubectl get serviceexports -n store --context ${CLUSTER_NAME2}
```

以下のように各クラスタに `store` と `store-**` という ServiceExport が作成されていれば OK です。
```text
NAME          AGE
store         5m25s
store-tokyo   5m25s
NAME          AGE
store         5m23s
store-osaka   5m23s
```

数分以内に MCS Controller によって ServiceImport が自動的に作成されます。  
時間に余裕があれば少し時間を空けて以下のコマンドを実行してみてください（このステップはスキップしてもらっても良いです）。
```bash
kubectl get serviceimports -n store --context ${CLUSTER_NAME1}
kubectl get serviceimports -n store --context ${CLUSTER_NAME2}
```

以下のように各クラスタに、双方の `store-**` サービスが import されていると思います。
```text
NAME          TYPE           IP                   AGE
store         ClusterSetIP   ["34.118.226.131"]   5m
store-osaka   ClusterSetIP   ["34.118.238.136"]   3m14s
store-tokyo   ClusterSetIP   ["34.118.229.217"]   4m59s
NAME          TYPE           IP               AGE
store         ClusterSetIP   ["10.5.3.50"]    4m59s
store-osaka   ClusterSetIP   ["10.5.2.88"]    3m18s
store-tokyo   ClusterSetIP   ["10.5.3.205"]   4m58s
```

### **Gatewy リソースを作成する**

Gateway リソースをデプロイする前に、外部 IP アドレスを事前に予約しておきます。  
`nip.io` を使った名前解決ができるように httoroute リソース内の hostname 定義も変更します。
```bash
gcloud compute addresses create mc-gatewayip --global --ip-version IPV4
export MC_IP_ADDR=$(gcloud compute addresses list --format='value(ADDRESS)' --filter="NAME:mc-gatewayip")
export MC_DOMAIN="${MC_IP_ADDR//./-}.nip.io"

sed -i "s/x-x-x-x.nip.io/$MC_DOMAIN/g" manifests/public-store-route.yaml
sed -i "s/x-x-x-x.nip.io/$MC_DOMAIN/g" manifests/public-store-route-header.yaml
```

以下のコマンドを実行し、Gateway と HTTPRoute リソースをデプロイします。
```bash
kubectl apply -f manifests/external-http-gateway.yaml --context ${CLUSTER_NAME1}
kubectl apply -f manifests/public-store-route.yaml    --context ${CLUSTER_NAME1}
```

正常にリソースがデプロイできているか確認します。
```bash
kubectl get -f manifests/external-http-gateway.yaml --context ${CLUSTER_NAME1}
kubectl get -f manifests/public-store-route.yaml    --context ${CLUSTER_NAME1}
```

以下のように Gateway の IP アドレスがアサインされ、また HTTPRoute で Hostname が正しく設定されていることを確認します。
```text
NAME            CLASS                               ADDRESS         PROGRAMMED   AGE
external-http   gke-l7-global-external-managed-mc   34.160.250.20   True         14m
NAME                 HOSTNAMES                  AGE
public-store-route   ["34-160-250-20.nip.io"]   14m
```

### **Multi-cluster Gateway の動作を確認する**
Gateway リソースが正常に構成されていることを確認できたら、以下コマンドを実行し　Multi-cluster Gateway で公開されたサンプルアプリケーションにアクセスしてみましょう。
```bash
curl http://${MC_DOMAIN}/tokyo

curl http://${MC_DOMAIN}/osaka

curl http://${MC_DOMAIN}
```

以下のように `/tokyo` にアクセスすると `gke-tokyo` にルーティングされ、`/osaka` にアクセスすると `gke-osaka` にルーティングされることが確認できます。  
また、パスを指定しない場合は Multi-cluster Gateway のデフォルトの動きとして、クライアントから地理的に近い（レイテンシーが低い）クラスタにルーティングされます。クライアントが東京にある場合は `gke-tokyo` が優先され、大阪にあるクライアントからは `gke-osaka` が優先されるはずです。
```text
curl http://${MC_DOMAIN}/tokyo
{"cluster_name":"gke-tokyo","gce_instance_id":"4721284863869288114","gce_service_account":"kkuchima-sandbox.svc.id.goog","host_header":"34-160-250-20.nip.io","pod_name":"store-5c65bdf74f-mnk8f","pod_name_emoji":"\ud83c\udff5","project_id":"kkuchima-sandbox","timestamp":"2023-09-06T04:33:55","zone":"asia-northeast1-a"}

curl http://${MC_DOMAIN}/osaka
{"cluster_name":"gke-osaka","gce_instance_id":"702247582886946019","gce_service_account":"kkuchima-sandbox.svc.id.goog","host_header":"34-160-250-20.nip.io","pod_name":"store-5dbdf67f49-5jl8g","pod_name_emoji":"\ud83c\udde6\ud83c\uddf8","project_id":"kkuchima-sandbox","timestamp":"2023-09-06T04:34:10","zone":"asia-northeast2-b"}

curl http://${MC_DOMAIN}
{"cluster_name":"gke-tokyo","gce_instance_id":"4721284863869288114","gce_service_account":"kkuchima-sandbox.svc.id.goog","host_header":"34-160-250-20.nip.io","pod_name":"store-5c65bdf74f-lghk6","pod_name_emoji":"\u270c\ufe0f","project_id":"kkuchima-sandbox","timestamp":"2023-09-06T04:34:15","zone":"asia-northeast1-a"}
```

## **ヘッダーベースのルーティング**
続いて HTTP ヘッダーの内容を基にルーティングするよう構成してみます。  
ユースケースとしては、片方のクラスタに新バージョンのサービスをデプロイし特定の HTTP ヘッダー付きでアクセスし動作を確認したり、クラスタの Blue/Green アップグレードをした際の動作確認などで活用できます。  
今回は `env: canary` という HTTP ヘッダが付与されているものは `gke-osaka` にルーティングされるように HTTPRoute リソースを構成します。

### **HTTPRoute の構成**
以下のように `env: canary` という HTTP ヘッダが付与されているものは `gke-osaka` にルーティングする HTTPRoute リソースを適用して挙動を確認します。
```yaml
  - matches:
    - headers:
      - name: env
        value: canary
    backendRefs:
      - group: net.gke.io
        kind: ServiceImport
        name: store-osaka
        port: 8080
```

上記構成が追加された HTTPRoute リソースを適用します。
```bash
kubectl apply -f manifests/public-store-route-header.yaml --context ${CLUSTER_NAME1}
```

`env: canary` という HTTP ヘッダを付与して何度かアクセスしてみましょう。
```bash
curl -H "env: canary" http://${MC_DOMAIN}
```

以下のように `gke-osaka` にルーティングされれば OK です。反映まで少し時間がかかる可能性があるので、想定通りの挙動とならない場合は少し待って再度ためしてみてください。
```text
curl -H "env: canary" http://${MC_DOMAIN}
{"cluster_name":"gke-osaka","gce_instance_id":"702247582886946019","gce_service_account":"kkuchima-sandbox.svc.id.goog","host_header":"34-160-250-20.nip.io","pod_name":"store-5dbdf67f49-th9lq","pod_name_emoji":"\u267b\ufe0f","project_id":"kkuchima-sandbox","timestamp":"2023-09-06T04:55:25","zone":"asia-northeast2-b"}

❯ curl -H "env: canary" http://${MC_DOMAIN}
{"cluster_name":"gke-osaka","gce_instance_id":"702247582886946019","gce_service_account":"kkuchima-sandbox.svc.id.goog","host_header":"34-160-250-20.nip.io","pod_name":"store-5dbdf67f49-5jl8g","pod_name_emoji":"\ud83c\udde6\ud83c\uddf8","project_id":"kkuchima-sandbox","timestamp":"2023-09-06T04:55:26","zone":"asia-northeast2-b"}
```

## **重みづけトラフィック ルーティング**
最後にクラスタ間のルーティングに重みづけをしてクラスタ間の Canary Release を構成してみます。  
ユースケースとしては、ヘッダーベースルーティングと同様にクラスタの Blue/Green アップグレードをした際などに、全体トラフィックの数%のみを新バージョンのクラスタに流すことでアップグレードによるリスクを低減すること等が考えられます。   
今回は `env: canary` という HTTP ヘッダが付与されているものは `gke-osaka` にルーティングされるように HTTPRoute リソースを構成します。

### **HTTPRoute の構成**
以下のように `env: canary` という HTTP ヘッダが付与されているものは `gke-osaka` にルーティングする HTTPRoute リソースを適用して挙動を確認します。
```yaml
  - backendRefs:
    - name: store-tokyo
      group: net.gke.io
      kind: ServiceImport
      port: 8080
      weight: 70
    - name: store-osaka
      group: net.gke.io
      kind: ServiceImport
      port: 8080
      weight: 30
```

上記構成が追加された HTTPRoute リソースを適用します。
```bash
kubectl apply -f manifests/public-store-route-canary.yaml --context ${CLUSTER_NAME1}
```

以下のコマンドを実行し 10 回程度 URL にアクセスし挙動を確認します。(Contorl + C でループを止めることができます)  
全体の約 90% が`gke-tokyo` に残りの 10% 程度が `gke-osaka` にルーティングされるはずです。反映まで少し時間がかかる可能性があるので、想定通りの挙動とならない場合は少し待って再度ためしてみてください。
```bash
while true; do curl http://${MC_DOMAIN} | grep "cluster_name"; sleep 1; done
```

## **Congraturations!**

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

これにて Multi-cluster Gateway を利用したアプリケーションのデプロイ、継続的デプロイ設定を使ったサービスの作成、セキュリティ向上策の導入、パフォーマンス・チューニング、そしてロードバランサを使ったグローバル展開が完了しました。

デモで使った資材が不要な方は、次の手順でクリーンアップを行って下さい。

## **クリーンアップ（プロジェクトを削除）**

ハンズオン用に利用したプロジェクトを削除し、コストがかからないようにします。

### **1. Google Cloud のデフォルトプロジェクト設定の削除**

```bash
gcloud config unset project
```

### **2. プロジェクトの削除**

```bash
gcloud projects delete ${PROJECT_ID}
```

### **3. ハンズオン資材の削除**

```bash
cd $HOME && rm -rf gke-advanced
```
