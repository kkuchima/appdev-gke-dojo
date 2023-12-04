# **GKE 道場 -応用編-**

本ハンズオンでは、前半に Google Cloud のフルマネージドなサービスメッシュである [Anthos Service Mesh (ASM)](https://cloud.google.com/service-mesh?hl=ja) の基本的な機能を体験し、後半は [Multi-cluster Gateway](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways?hl=ja) を使って複数リージョンにデプロイした GKE クラスタ間のトラフィックルーティングを行う方法について学びます。  

## **ハンズオン実施内容**
Anthos Service Mesh のハンズオンでは、サービスメッシュを使った以下機能を体験します。
- 重みづけルーティングを利用したカナリアリリース
- mTLS によるサービス間通信の暗号化・相互認証

Multi-cluster Gateway のハンズオンでは、Multi-cluster Gateway を使って複数リージョンにデプロイした GKE クラスタ間の高度なトラフィックルーティングを行う方法を体験します。
- クラスタ間のパスベース・レイテンシーベース ルーティング
- クラスタ間のヘッダベース ルーティング
- 重みづけトラフィック ルーティングによるクラスタ間のカナリアリリース

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

### **1. API の有効化**
Google Cloud では利用したい機能（API）ごとに、有効化を行う必要があります。

ここでは、以降のハンズオンで利用する機能を事前に有効化しておきます。

```bash
gcloud services enable mesh.googleapis.com \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  gkehub.googleapis.com \
  multiclusterservicediscovery.googleapis.com \
  multiclusteringress.googleapis.com \
  trafficdirector.googleapis.com
```

**GUI**: [API ライブラリ](https://console.cloud.google.com/apis/library)

## **GKE Autopilot クラスタのデプロイ**
まず最初に本ハンズオンで利用する GKE Autopilot クラスタを構築します。

### **1. 環境変数の設定**

クラスタ構築に必要となる環境変数を設定します。  

以下コマンドをコピーし、ターミナルに貼り付けて実行してください。
```
export PROJECT_NUM=`gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)"`
```

以下コマンドを実行してください。
```bash
export REGION1=asia-northeast1
export CLUSTER_NAME1=gke-tokyo
```

### **2. GKE Autopilot クラスタのデプロイ**

GKE Autoulot クラスタ `gke-tokyo` をデプロイします。デプロイ完了まで十数分かかります。

```bash
gcloud container clusters create-auto ${CLUSTER_NAME1} \
    --location=${REGION1} \
    --release-channel=stable \
    --enable-private-nodes
```

## **参考: Cloud Shell の接続が途切れてしまったときは?**

一定時間非アクティブ状態になる、またはブラウザが固まってしまったなどで `Cloud Shell` が切れてしまいブラウザのリロードが必要になる場合があります。その場合は再接続、リロードなどを実施後、以下の対応を行い、チュートリアルを再開してください。

### **1. チュートリアル資材があるディレクトリに移動する**

```bash
cd ~/appdev-gke-dojo/advanced/
```

### **2. チュートリアルを開く**

```bash
teachme tutorial.md
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

### **5. 環境変数の設定**

環境構築に必要となる環境変数を設定します。 

以下コマンドをコピーし、ターミナルに貼り付けて実行してください。  

```
export PROJECT_NUM=`gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)"`
```

以下コマンドを実行してください。  

```bash
export REGION1=asia-northeast1
export CLUSTER_NAME1=gke-tokyo
export REGION2=asia-northeast2
export CLUSTER_NAME2=gke-osaka
```

途中まで進めていたチュートリアルのページまで `Next` ボタンを押し、進めてください。


## **マネージド Anthos Service Mesh のプロビジョニング**
Fleet API を利用してマネージド Anthos Service Mesh (ASM) をプロビジョニングします。  
ASM のプロビジョニングは他にも [asmcli](https://cloud.google.com/service-mesh/docs/managed/provision-managed-anthos-service-mesh-asmcli?hl=ja) を利用した方法もありますが、Fleet API を利用して Managed ASM をプロビジョニングすることをお勧めします。  
Fleet API 経由でマネージド ASM をプロビジョニングすると Managed Data Plane (MDP) も有効化されます。

<walkthrough-info-message>[Fleet](https://cloud.google.com/kubernetes-engine/docs/fleets-overview?hl=ja) は複数の GKE クラスタのグルーピングや複数クラスタに対する各種機能の有効・無効を制御する機能です。</walkthrough-info-message>

### **1. GKE クラスタのクレデンシャルを取得する**

以下のコマンドを実行し、GKE クラスタへアクセスするためのクレデンシャルを取得します。

```bash
gcloud container clusters get-credentials ${CLUSTER_NAME1} \
     --region ${REGION1} 
```

### **2. クラスタを Fleet に登録する**

Fleet で Anthos Service Mesh を有効化します。

```bash
gcloud container fleet mesh enable
```

Fleet で GKE クラスタを管理するために、Fleet へ登録します。

```bash
gcloud container fleet memberships register ${CLUSTER_NAME1} \
  --gke-cluster ${REGION1}/${CLUSTER_NAME1} \
  --enable-workload-identity \
  --project ${PROJECT_ID}
```

Fleet にクラスタが登録されていることを確認します。

```bash
gcloud container fleet memberships list --project ${PROJECT_ID}
```

GKE クラスタに mesh_id ラベルとして Fleet プロジェクト番号を付与します。

```bash
gcloud container clusters update ${CLUSTER_NAME1} \
  --region ${REGION1} --update-labels mesh_id=proj-${PROJECT_NUM}
```

### **3. マネージド ASM のプロビジョニング**

Fleet 経由でマネージド ASM をプロビジョニングします。  

```bash
gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME1} \
    --project ${PROJECT_ID} \
    --location ${REGION1}
```
  
約10分後、コントロール プレーンのステータスが ACTIVE になっていることを確認します。

```bash
gcloud container fleet mesh describe --project ${PROJECT_ID}
```
  
以下のように `controlPlaneManagement` が `ACTIVE` になることを確認します。  
プロビジョニングが完了するまで数分かかります。  

```text
membershipSpecs:
  projects/1072516076243/locations/asia-northeast1/memberships/test-ap01:
    mesh:
      management: MANAGEMENT_AUTOMATIC
membershipStates:
  projects/1072516076243/locations/asia-northeast1/memberships/test-ap01:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_READY
          details: 'Ready: asm-managed'
        state: ACTIVE
      dataPlaneManagement:
        details:
        - code: OK
          details: Service is running.
        state: ACTIVE
    state:
      code: OK
      description: 'Revision(s) ready for use: asm-managed.'
      updateTime: '2023-09-05T03:49:39.353047207Z'
```

### **4. Ingress Gateway のデプロイ**
メッシュ外部からのアクセスを受け付けるためのコンポーネントとして Ingress Gateway を GKE クラスタ内にデプロイします。  

まず、Ingress Gateway 用の Namespace `gateway` を作成し、`gateway` に Sidecar Injection 用のラベルを付与します。  
ASM ではこのラベルを用いて Envoy Proxy をアプリケーションに inject したりバージョン管理を行っています。  

```bash
kubectl create namespace gateway
kubectl label namespace gateway istio.io/rev=asm-managed-stable --overwrite
```

```bash
kubectl apply -n gateway -f asm/istio-ingress-gateway/
```
<walkthrough-info-message>従来は Istio Operator 等を利用し Ingress Gateway のライフサイクル管理を行っていましたが、最近は通常のワークロードと同様に Sidecar Injection の機能を用いて Ingress Gateway を管理する方法が推奨されています。これにより、コントロールプレーンと異なるタイミングで Ingress Gateway のアップデートがし易くなります。</walkthrough-info-message>

デプロイが完了するまで数分待機します。  

```bash
kubectl get pods -n gateway -w
```

以下のように STATUS が `Running` となっていれば OK です。  
Running となりましたら `Control + c` で watch を終了してください。  

```text
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-7d99cdb85d-56r4j   1/1     Running   0          26h
```

### **5. サンプルアプリケーションにサイドカーを injection する**
Online Boutique というサンプルアプリケーションにサイドカーを inject し、サービスメッシュの機能を利用できるようにします。  

ASM では Namespace や Deployment 等に付与されたラベルを基に Mutating Webhook を使って自動的にアプリケーション Pod にサイドカーを injection しているため、対象の Namespace に injection 用のラベルを付与します。
![mutatingwebhook](https://d33wubrfki0l68.cloudfront.net/af21ecd38ec67b3d81c1b762221b4ac777fcf02d/7c60e/images/blog/2019-03-21-a-guide-to-kubernetes-admission-controllers/admission-controller-phases.png)

```bash
kubectl label namespace default istio.io/rev=asm-managed-stable --overwrite
kubectl apply -f asm/sample-online-boutique/
```

```bash
kubectl get pods -w
```
  
以下のように Pod の STATUS が `Running` となっていること、またコンテナの数 (READY の下) が `2/2` と表示されていることを確認します。  

アプリケーション コンテナとサイドカー合わせて 2 つのコンテナが立ち上がっていることが確認できます。  

全て Running となりましたら `Control + c` で watch を終了してください。  

```text
kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-d567fdc66-kk4hx                2/2     Running   0          6m33s
cartservice-b67cf9ff9-dqxcd              2/2     Running   0          6m34s
checkoutservice-697d84c9dd-l55wk         2/2     Running   0          6m34s
currencyservice-778f6ff9dd-z89xx         2/2     Running   0          6m34s
emailservice-6475cfb896-45pvl            2/2     Running   0          6m31s
frontend-76d74657d7-qxw7x                2/2     Running   0          6m31s
paymentservice-86f78d5d44-mp5js          2/2     Running   0          6m32s
productcatalogservice-cc5bf5cf8-zdmhz    2/2     Running   0          6m31s
recommendationservice-79dc7dcb7b-d8z6d   2/2     Running   0          6m32s
redis-cart-66df4dc7f4-c9b6c              2/2     Running   0          6m33s
shippingservice-58786fbcd4-bvtkq         2/2     Running   0          6m32s
```

## **Ingress Gateway 経由でアプリケーションを公開する**

### **1. Istio Gateway リソースの適用**

Istio の Gateway リソースを適用し、ingress gateway 経由でサンプルアプリケーションを公開できるようにします。  

```bash
kubectl apply -f asm/istio-manifests/gw-frontend.yaml
```

### **2. サンプルアプリケーションへのアクセス**
  
プロビジョニングされたロードバランサーの外部IPアドレスを取得します。  
以下コマンドをコピーし、ターミナルに貼り付けて実行してください。  

```text
export IP_ADDR=`kubectl get svc istio-ingressgateway -n gateway -o jsonpath="{.status.loadBalancer.ingress[*].ip}"`
```
  
以下コマンドを実行し、表示された URL にアクセスしてみましょう。Ingress Gateway 経由でサンプルアプリケーションにアクセスできるようになっています。  
LB のプロビジョニングに時間がかかるため、アクセスできない場合は数分待ってから再度お試しください。  

```bash
echo http://${IP_ADDR}
```

何回かサンプルアプリケーションにアクセスした後、 [ASM トポロジービュー](https://console.cloud.google.com/anthos/services)でサービスメッシュのトポロジーを確認することができます。

## **カナリアリリースを試す**

カナリアリリースはリリース手法の1つで、新しいバージョンのリリースをする際にトラフィックの全量ではなく一部だけを新バージョンにルーティングすることにより、新バージョンで問題が発生した際の影響を抑え、リリースのリスクを低減させることができます。  

ASM では `VirtualService` と `DestinationRule` というリソースを用いて、カナリアリリースを実現することができます。  

今回は ProductCatalog というサービスを２バージョンデプロイし、新しいバージョン (v2) には全体の 5 % のみトラフィックを流し、残りの 95 % のトラフィックを安定したバージョン (v1) に流し安全にリリースできるようにします。  

### **1. 新バージョンの ProductCatalog サービスをデプロイ**

実は ProductCatalog サービスには `EXTRA_LATENCY` という環境変数で遅延を発生させる機能があります。  

今回はアプリケーション自体には手を加えずに、この環境変数を使って遅延を発生させた状態を v2 としてリリースします。  

```bash
kubectl apply -f asm/istio-manifests/productcatalog-v2.yaml
```

v2 には以下のような環境変数が追加されています。
```text
cat productcatalog-v2.yaml
...省略...
       env:
       - name: PORT
         value: "3550"
       - name: DISABLE_PROFILER
         value: "1"
       - name: EXTRA_LATENCY
         value: 3s
```

### **2. DestinationRule と VirtualService のデプロイ**
ASM のリソースである DestinationRule と VirtualService をデプロイし、ProductCatalog v2 には全体の 5 % のみトラフィックを流すように設定します。  
DestinationRule では `version` ラベルの値を基に、各バージョン (v1, v2) の Subset を定義しています。
```yaml:dr-catalog.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productcatalogservice
spec:
  host: productcatalogservice.default.svc.cluster.local
  subsets:
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
```

VirtualService では v1 Subset に 全体の95% のトラフィックを流し、v2 Subset に全体の 5% のトラフィックを流すようルーティングを定義しています。
```yaml:vs-split-traffic.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productcatalogservice
spec:
  hosts:
  - productcatalogservice
  http:
  - route:
    - destination:
        host: productcatalogservice
        subset: v1
      weight: 95
    - destination:
        host: productcatalogservice
        subset: v2
      weight: 5
```

上記マニフェストを適用します。
```bash
kubectl apply -f asm/istio-manifests/dr-catalog.yaml
kubectl apply -f asm/istio-manifests/vs-split-traffic.yaml
```

### **3, サンプルアプリケーションへのアクセス**

以下コマンドを実行し、表示された URL に再度アクセスしてみましょう。  

少し分かりにくいですが、商品画像をクリックするとたまに遅延が発生するようになっています。  

```bash
echo http://${IP_ADDR}
```

変化が確認できたら VirtualService と DestinationRule を削除しましょう。  

```bash
kubectl delete -f asm/istio-manifests/dr-catalog.yaml
kubectl delete -f asm/istio-manifests/vs-split-traffic.yaml
```

## **mTLS を試す**

ASM の Mutual TLS (mTLS) 機能を有効化するとサーバ - クライアント間の相互認証により、接続先だけでなく接続元の信頼も保証することができます。  

また通信を TLS で暗号化することができ、ワークロード間の通信をよりセキュアにすることが可能です。 mTLS で利用する証明書はサイドカープロキシで保持・管理されるため、ワークロードからは透過的に mTLS が利用されます。  

今回はサービスメッシュ全体で mTLS を有効化し、メッシュ外のワークロードからのアクセスを禁止するように構成します。  

### **1. 現在の mTLS モードを確認する**

mTLS のモードは 3 種類あります。デフォルトは `Permissive` というモードで、暗号化された通信と平文どちらも許容します。  

- Permissive: 暗号化された通信と平文どちらも許容する (デフォルト)
- Strict: 暗号化された通信のみ許容する
- Disable: mTLS を無効化する

適用されている mTLS モードは [Anthos Security ダッシュボード](https://console.cloud.google.com/anthos/security/policy-audit/asia-northeast1/gke-tokyo)の**ポリシー監査タブ**から確認することができます。  

### **2. Sleep Pod のデプロイ**

sleep というサービスメッシュ外の Namespace 上に Pod をデプロイし、メッシュ外からのアクセスの挙動を確認します。  

```bash
kubectl create ns sleep
kubectl apply -f asm/mtls/sleep.yaml
```

Sleep pod から Frontend サービスへ正常にアクセスできることを確認します。(ステータスコード 200 が返ってきます)  

```bash
export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS frontend.default -o /dev/null -s -w '%{http_code}\n'
```

### **3. mTLS を適用する**

`PeerAuthentication` リソースを使って mTLS を STRICT mode にし、平文通信を許可しない設定を適用します。  

対象 Namespace を `istio-system` にすることで、サービスメッシュ全体に対して mTLS を STRICT mode で設定することができます。  

```yaml:mtls-meshwide.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: mesh-wide
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

以下コマンドを実行しサービスメッシュ全体に mTLS STRICT mode を適用します。  

```bash
kubectl apply -f asm/mtls/mtls-meshwide.yaml
```

### **STRICT mode の確認**

[Anthos Security ダッシュボード](https://console.cloud.google.com/anthos/security/policy-audit/asia-northeast1/gke-tokyo)の**ポリシー監査タブ**上でサンプル アプリケーションの mTLS mode が STRICT になっていることを確認します。  

また、Sleep pod から Frontend サービスへアクセスすると STRICT mode により平文でのアクセスがエラーとなることを確認します。  

```bash
export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS frontend.default -o /dev/null -s -w '%{http_code}\n'
```

```text
想定される出力：
curl: (56) Recv failure: Connection reset by peer
```
これで Anthos Service Mesh のハンズオンは以上です。

## **Multi-cluster Gateway ハンズオン**
本ハンズオンでは [Multi-cluster Gateway](https://cloud.google.com/kubernetes-engine/docs/how-to/deploying-multi-cluster-gateways?hl=ja) を使って複数リージョンにデプロイした GKE クラスタ間のトラフィックルーティングを行う方法について学びます。  
- クラスタ間のパスベース・レイテンシーベース ルーティング
- クラスタ間のヘッダベース ルーティング
- 重みづけトラフィック ルーティングによるクラスタ間のカナリアデプロイ

## **2つ目の GKE Autopilot クラスタのデプロイ**
### **1. 環境変数の設定**

まずクラスタ構築に必要となる環境変数を設定します。  

```bash
export REGION1=asia-northeast1
export CLUSTER_NAME1=gke-tokyo
export REGION2=asia-northeast2
export CLUSTER_NAME2=gke-osaka
```

### **2. GKE Autopilot クラスタのデプロイ**

2台目のクラスタである `gke-osaka` をデプロイします。デプロイ完了まで十数分かかります。  

```bash
gcloud container clusters create-auto ${CLUSTER_NAME2} \
    --location=${REGION2} \
    --release-channel=stable \
    --enable-private-nodes
```

### **3. コンテキストの設定**

あとから操作しやすいように、各クラスタのコンテキストを変更しておきます。  

```bash
kubectx ${CLUSTER_NAME1}=gke_${PROJECT_ID}_${REGION1}_${CLUSTER_NAME1}
kubectx ${CLUSTER_NAME2}=gke_${PROJECT_ID}_${REGION2}_${CLUSTER_NAME2}
```

## Fleet への登録

Fleet で Multi-cluster Gateway / Multi-cluster Service を有効にするため、`gke-osaka` クラスタを Fleet へ登録します。  

```bash
gcloud container fleet memberships register ${CLUSTER_NAME2} \
  --gke-cluster ${REGION2}/${CLUSTER_NAME2} \
  --enable-workload-identity \
  --project ${PROJECT_ID}
```

Fleet に`gke-tokyo` と `gke-osaka`クラスタが登録されていることを確認します。  

```bash
gcloud container fleet memberships list --project ${PROJECT_ID}
```

以下のように表示されれば OK です。  

```text
NAME: gke-tokyo
EXTERNAL_ID: 9fc00540-609f-4738-939f-84a37f89993e
LOCATION: asia-northeast1

NAME: gke-osaka
EXTERNAL_ID: 31b95fda-47f5-4236-a80a-ae08e43844f9
LOCATION: asia-northeast2
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

以下のように `gke-tokyo` と `gke-osaka` の status が OK となっていれば大丈夫です。(反映まで数分かかる場合があります)  

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

複数クラスタ間での負荷分散設定を行うために Multi-cluster Gateway Controller をデプロイします。  

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

以下のように `gke-tokyo` と `gke-osaka` の status が OK となっていれば大丈夫です。(反映まで数分かかる場合があります)  

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

今回は `store` という稼働しているクラスタ・ノード情報を返してくれるサンプルアプリケーションを利用します。  

```bash
kubectl apply -f mcgw/manifests/store.yaml --context ${CLUSTER_NAME1}
kubectl apply -f mcgw/manifests/store.yaml --context ${CLUSTER_NAME2}
```

以下コマンドを実行し、各クラスタ上に正常に Pod がデプロイできたことを確認します。  

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

Multi-cluster Gateway では宛先のエンドポイントとして Multi-cluster Service (MCS) で公開したサービスを指定しています。  

そのため、Multi-cluster Gateway で公開したいサービスは予め MCS で共有サービスとしてエンドポイントを用意してあげる必要があります。  

MCS で利用されるリソースは以下の通りです：  

- **ServiceExport** ... クラスタ内のサービスを他クラスタに公開する際に利用するリソースです。
- **ServiceImport** ... Multi-cluster Service Controller によって、自動作成されるリソースです。`ServiceExport` によって公開されたサービスを自クラスタ内に `ServiceImport` としてリソースを作成しクラスタ間でのサービスディスカバリーを実現します。

![MCS Service Discovery](https://cloud.google.com/static/kubernetes-engine/images/multi-cluster-service-example1.svg)

### **1. ServiceExport のデプロイ**

では、各クラスタにデプロイした `store` サービスを他のクラスタにも公開してみましょう。  

他クラスタからもサービスディスカバリーできるように `ServiceExport` をデプロイします。  

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
kubectl apply -f mcgw/manifests/store-export-tokyo.yaml --context ${CLUSTER_NAME1}
kubectl apply -f mcgw/manifests/store-export-osaka.yaml --context ${CLUSTER_NAME2}
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

数分以内に MCS Controller によって ServiceImport も自動的に作成されます。  

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

### **2. Gatewy リソースを作成する**

Gateway リソースをデプロイする前に、外部 IP アドレスを事前に予約しておきます。  

`nip.io` を使った名前解決ができるように httoroute リソース内の hostname 定義も変更します。

```bash
gcloud compute addresses create mc-gatewayip --global --ip-version IPV4
export MC_IP_ADDR=$(gcloud compute addresses list --format='value(ADDRESS)' --filter="NAME:mc-gatewayip")
export MC_DOMAIN="${MC_IP_ADDR//./-}.nip.io"

sed -i "s/x-x-x-x.nip.io/$MC_DOMAIN/g" mcgw/manifests/public-store-route.yaml
sed -i "s/x-x-x-x.nip.io/$MC_DOMAIN/g" mcgw/manifests/public-store-route-header.yaml
sed -i "s/x-x-x-x.nip.io/$MC_DOMAIN/g" mcgw/manifests/public-store-route-canary.yaml
```

以下のコマンドを実行し、Gateway と HTTPRoute リソースをデプロイします。  

```bash
kubectl apply -f mcgw/manifests/external-http-gateway.yaml --context ${CLUSTER_NAME1}
kubectl apply -f mcgw/manifests/public-store-route.yaml    --context ${CLUSTER_NAME1}
```

適用した Multi-cluster Gateway のリソースは以下です(一部省略)。  
`/tokyo` の場合は `gke-tokyo` へ、`/osaka` の場合は `gke-osaka` へトラフィックが流れるように設定しています。  

```yaml
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: public-store-route
~~~
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /tokyo
    backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: store-tokyo
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /osaka
    backendRefs:
      - group: net.gke.io
        kind: ServiceImport
        name: store-osaka
        port: 8080
  - matches:
    - headers:
      - name: env
        value: canary
    backendRefs:
      - group: net.gke.io
        kind: ServiceImport
        name: store-osaka
        port: 8080
  - backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: store
      port: 8080
```

正常にリソースがデプロイできているか確認します。  

```bash
kubectl get -f mcgw/manifests/external-http-gateway.yaml --context ${CLUSTER_NAME1}
kubectl get -f mcgw/manifests/public-store-route.yaml    --context ${CLUSTER_NAME1}
```

以下のように Gateway の IP アドレスがアサインされ、また HTTPRoute で Hostname が正しく設定されていることを確認します。  

```text
NAME            CLASS                               ADDRESS         PROGRAMMED   AGE
external-http   gke-l7-global-external-managed-mc   34.160.250.20   True         14m
NAME                 HOSTNAMES                  AGE
public-store-route   ["34-160-250-20.nip.io"]   14m
```

### **3. Multi-cluster Gateway の動作を確認する**

Gateway リソースが正常に構成されていることを確認できたら、以下コマンドを実行し　Multi-cluster Gateway で公開されたサンプルアプリケーションにアクセスしてみましょう。  

`curl: (52) Empty reply from server` 等のエラーが返ってくる場合は Load Balancer への反映ができていない可能性があるため、数分待機した後に再度お試しください。  

```bash
curl http://${MC_DOMAIN}/tokyo
```

```bash
curl http://${MC_DOMAIN}/osaka
```

```bash
curl http://${MC_DOMAIN}
```

以下のように `/tokyo` にアクセスすると `gke-tokyo` にルーティングされ、`/osaka` にアクセスすると `gke-osaka` にルーティングされることが確認できます。  

また、パスを指定しない場合は Multi-cluster Gateway のデフォルトの動きとして、クライアントから地理的に近い（レイテンシーが低い）クラスタにルーティングされます。  
クライアントが東京にある場合は `gke-tokyo` が優先され、大阪にあるクライアントからは `gke-osaka` が優先されるはずです。  

Cloud Shell はインターネット上の1つの VM であるため、東京からアクセスしているつもりでも `gke-osaka` にアクセスする場合があります。  
その場合は以下のコマンドで表示される URL にアクセスし、ブラウザを開いている物理的な端末 (PC) からアクセスしたときの挙動も確認してみましょう。  
東京からアクセスしている場合は `gke-tokyo` にアクセスされると思います。  

```bash
echo http://${MC_DOMAIN}
```

## **ヘッダーベースのルーティング**

続いて HTTP ヘッダーの内容を基にルーティングするよう構成してみます。  

ユースケースとしては、片方のクラスタに新バージョンのサービスをデプロイし特定の HTTP ヘッダー付きでアクセスし動作を確認したり、クラスタの Blue/Green アップグレードをした際の動作確認などで活用できます。  

今回は `env: canary` という HTTP ヘッダが付与されているものは `gke-osaka` にルーティングされるように HTTPRoute リソースを構成します。  

### **1. HTTPRoute の構成**

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
kubectl apply -f mcgw/manifests/public-store-route-header.yaml --context ${CLUSTER_NAME1}
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

### **1. HTTPRoute の構成**

以下のように全体トラフィックの 90% を`gke-tokyo` に、残りの 10% を `gke-osaka` にルーティングする HTTPRoute リソースを適用して挙動を確認します。  

```yaml
  - backendRefs:
    - name: store-tokyo
      group: net.gke.io
      kind: ServiceImport
      port: 8080
      weight: 90
    - name: store-osaka
      group: net.gke.io
      kind: ServiceImport
      port: 8080
      weight: 10
```

上記構成が追加された HTTPRoute リソースを適用します。  

```bash
kubectl apply -f mcgw/manifests/public-store-route-canary.yaml --context ${CLUSTER_NAME1}
```

以下のコマンドを実行し 10 回程度 URL にアクセスし挙動を確認します。(Contorl + C でループを止めることができます)  

全体の約 90% が`gke-tokyo` に残りの 10% 程度が `gke-osaka` にルーティングされるはずです。  
反映まで少し時間がかかる可能性があるので、想定通りの挙動とならない場合は少し待って再度ためしてみてください。  

```bash
while true; do curl -s http://${MC_DOMAIN} | grep "cluster_name"; sleep 1; done
```

## **Congraturations!**

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

これにてフルマネージドなサービス メッシュである Anthos Service Mesh (ASM) と Multi-cluster Gateway の機能を活用し GKE をより高度に運用する方法を学習しました。

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
cd $HOME && rm -rf appdev-gke-dojo/
```