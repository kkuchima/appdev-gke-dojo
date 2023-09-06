# **Anthos Service Mesh ハンズオン**

本ハンズオンでは、Google Cloud のフルマネージドなサービスメッシュである [Anthos Service Mesh (ASM)](https://cloud.google.com/service-mesh?hl=ja) の基本的な機能を体験します。

- サービス間の重みづけルーティングを利用した Canary Release
- mTLS によるサービス間通信の暗号化・相互認証
- (Optional) Authorization Policy を活用したサービス間通信の認可設定

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
gcloud services enable mesh.googleapis.com \
  container.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

**GUI**: [API ライブラリ](https://console.cloud.google.com/apis/library)

## **マネージド Anthos Service Mesh のプロビジョニング**

Fleet API を利用してマネージド Anthos Service Mesh (ASM) をプロビジョニングします。
ASM のプロビジョニングは他にも [asmcli](https://cloud.google.com/service-mesh/docs/managed/provision-managed-anthos-service-mesh-asmcli?hl=ja) を利用した方法もありますが、Fleet API を利用して Managed ASM をプロビジョニングすることをお勧めします。Fleet API 経由でマネージド ASM をプロビジョニングすると Managed Data Plane (MDP) も有効化されます。

<walkthrough-info-message>[Fleet](https://cloud.google.com/kubernetes-engine/docs/fleets-overview?hl=ja) は複数の GKE クラスタのグルーピングや複数クラスタに対する各種機能の有効・無効を制御する機能です。</walkthrough-info-message>

### **1. 環境変数・gcloud コマンドの設定**

```bash
export REGION1=asia-northeast1
export CLUSTER_NAME1=gke-tokyo

gcloud container clusters get-credentials ${CLUSTER_NAME1} \
     --region ${REGION1} 
```

### **2. クラスタを Fleet に登録する**
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
export PROJECT_NUM=`gcloud projects describe ${PROJECT_ID} --format="value(projectNumber)"`

gcloud container clusters update ${CLUSTER_NAME1} \
  --region ${REGION1} --update-labels mesh_id=proj-${PROJECT_NUM}
```

### **3. マネージド ASM のプロビジョニング**
Fleet による自動管理を有効にし、マネージド ASM をプロビジョニングします。（プロビジョニング完了まで数分かかります）
```bash
gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME1} \
    --project ${PROJECT_ID} \
    --location ${REGION1}
```

数分後、コントロール プレーンのステータスが ACTIVE になっていることを確認します。
```bash
gcloud container fleet mesh describe --project ${PROJECT_ID}
```

以下のように `controlPlaneManagement` が `ACTIVE` になっていることを確認します。
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


### **4. Ingress Gateway のデプロイ**

メッシュ外部からのアクセスを受け付けるためのコンポーネントとして Ingress Gateway を GKE クラスタ内にデプロイします。
まず、Ingress Gateway 用の Namespace `gateway` を作成し、`gateway` に Sidecar Injection 用のラベルを付与します。
Istio / ASM ではこのラベルを用いて Envoy Proxy をアプリケーションに inject したりバージョン管理を行っています。
```bash
kubectl create namespace gateway

kubectl label namespace gateway istio.io/rev=asm-managed --overwrite

kubectl apply -n gateway -f istio-ingress-gateway/
```
<walkthrough-info-message>従来は Istio Operator 等を利用し Ingress Gateway のライフサイクル管理を行っていましたが、最近の Istio や ASM では通常のワークロードと同様に Sidecar Injection の機能を用いて Ingress Gateway を管理する方法が推奨されています。これにより、コントロールプレーンと異なるタイミングで Ingress Gateway のアップデートがし易くなります。</walkthrough-info-message>

デプロイが完了するまで数分待機します。
Todo: Serviceは NEG にしておく
```bash
kubectl get pod,service -n gateway
```

### **5. サンプルアプリケーションにサイドカーを injection する**
Lab01 でデプロイした Online Boutique アプリケーションにサイドカーを inject し、サービスメッシュの機能を利用できるようにします。  
ASM では Namespace や Deployment 等に付与されたラベルを基に Mutating Webhook を使って自動的にアプリケーション Pod にサイドカーを injection しているため、ASM の設定導入前にデプロイされていたアプリケーションに injection するために一度 Pod を再起動 (restart) する必要があります。
![mutatingwebhook](https://d33wubrfki0l68.cloudfront.net/af21ecd38ec67b3d81c1b762221b4ac777fcf02d/7c60e/images/blog/2019-03-21-a-guide-to-kubernetes-admission-controllers/admission-controller-phases.png)

```bash
kubectl label namespace default istio.io/rev=asm-managed --overwrite

kubectl rollout restart deployment

kubectl get pods
```
<walkthrough-info-message>サンプルアプリケーションがデプロイされていない場合は `kubectl apply -f sample-online-boutique` コマンドを実行し、</walkthrough-info-message>

以下のように Pod の数 (READY の下) が `2/2` と表示されていれば、アプリケーション コンテナとサイドカー合わせて2つのコンテナが立ち上がっていることが確認できます。
```
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

```bash
gcloud compute addresses create gatewayip --global --ip-version IPV4
export IP_ADDR=$(gcloud compute addresses list --format='value(ADDRESS)' --filter="NAME:gatewayip")
export DOMAIN="${IP_ADDR//./-}.nip.io"

sed -i "s/x-x-x-x.nip.io/$DOMAIN/g" gateway/httproute.yaml
```

## **Ingress Gateway 経由でアプリケーションを公開する**

### **1. Istio Gateway リソースの適用**
Istio の Gateway リソースを適用し、ingress gateway 経由でサンプルアプリケーションを公開できるようにします。
`istio-ingress-gateway` 配下のマニフェストを適用し、ingress gateway をデプロイします。
```bash
kubectl apply -f istio-ingress-gateway/
```

### **2. Kubernetes Gateway リソースの適用**
次に、Google Cloud の Applicaiton Load Balancer (ALB) 経由で istio ingress gateway にアクセスできるように Kubernetes の Gateway リソースを構成します。
Todo：画像を追加
```bash
kubectl apply -f k8s-gateway/
```
<walkthrough-info-message>同じような名前が出てきて混乱するかもしれませんが、ALB 等クラスタ外のロードバランサーを構成するのが Kubernetes の Gateway リソースで、Istio の　Ingress / Egress Gateway というコンポーネントを構成するのが Istio の Gateway とここでは理解してください。 (細かい話をすると K8s Gateway でも Istio Ingress/Egress Gateway を構成できますが、ここでは割愛します。)</walkthrough-info-message>

### **3. サンプルアプリケーションへのアクセス**
以下コマンドを実行し、表示された URL にアクセスしてみましょう。Istio Ingress Gateway 経由でサンプルアプリケーションにアクセスできるようになっています。
```bash
echo http://${DOMAIN}
```

何回かサンプルアプリケーションにアクセスした後、[ASM トポロジービュー](https://console.cloud.google.com/anthos/services) でサービスメッシュのトポロジーを確認することができます。
Todo: トポロジーの図を追加

## **カナリアリリースを試す**
ここから ASM のトラフィック管理機能を用いて、カナリアリリースを試します。カナリアリリースはリリース手法の1つで、新しいバージョンのリリースをする際にトラフィックの全量ではなく一部だけを新バージョンにルーティングすることにより、新バージョンで問題が発生した際の影響を抑え、リリースのリスクを低減させることができます。
ASM では `VirtualService` と `DestinationRule` というリソースを用いて、カナリアリリースを実現することができます。
今回は ProductCatalog というサービスを２バージョンデプロイし、新しいバージョン (v2) には全体の 25 % のみトラフィックを流し、残りの 75 % のトラフィックを安定したバージョン (v1) に流し安全にリリースできるようにします。

### **新バージョンの ProductCatalog サービスをデプロイ**
実は ProductCatalog サービスには `EXTRA_LATENCY` という環境変数で遅延を発生させる機能があります。今回はアプリケーション自体には手を加えずに、この環境変数を使って遅延を発生させた状態を v2 としてリリースします。
```bash
kubectl apply -f productcatalog-v2.yaml

# cat productcatalog-v2.yaml
# ...省略...
#        env:
#        - name: PORT
#          value: "3550"
#        - name: DISABLE_PROFILER
#          value: "1"
#        - name: EXTRA_LATENCY
#          value: 3s
```

### **DestinationRule と VirtualService のデプロイ**
ASM のリソースである DestinationRule と VirtualService をデプロイし、ProductCatalog v2 には全体の 25 % のみトラフィックを流すように設定します。  

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

VirtualService では v1 Subset に 全体の75% のトラフィックを流し、v2 Subset に全体の 25% のトラフィックを流すようルーティングを定義しています。
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
      weight: 75
    - destination:
        host: productcatalogservice
        subset: v2
      weight: 25
```

上記マニフェストを適用します。
```bash
kubectl apply -f istio-manifests/
```

### **サンプルアプリケーションへのアクセス**
以下コマンドを実行し、表示された URL に再度アクセスしてみましょう。少し分かりにくいですが、商品画像をクリックするとおよそ4分の1の確率で遅延が発生するようになっています。
```bash
echo http://${DOMAIN}
```

変化が確認できたら VirtualService と DestinationRule を削除しましょう。
```bash
kubectl delete -f vs-split-traffic.yaml
kubectl delete -f dr-catalog.yaml
```

## **mTLS を試す**
ASM の Mutual TLS (mTLS) 機能を有効化するとサーバ - クライアント間の相互認証により、接続先だけでなく接続元の信頼も保証することができます。  
また通信を TLS で暗号化することができ、ワークロード間の通信をよりセキュアにすることが可能です。 mTLS で利用する証明書はサイドカープロキシで保持・管理されるため、ワークロードからは透過的に mTLS が利用されます。  
今回はサービスメッシュ全体で mTLS を有効化し、メッシュ外のワークロードからのアクセスを禁止するように構成します。

### **現在の mTLS モードを確認する**
mTLS のモードは 3 種類あります。デフォルトは `Permissive` というモードで、暗号化された通信と平文どちらも許容します。
- Permissive: 暗号化された通信と平文どちらも許容する (デフォルト)
- Strict: 暗号化された通信のみ許容する
- Disable: mTLS を無効化する

適用されている mTLS モードは [Anthos Security ダッシュボード](https://console.cloud.google.com/anthos/security/policy-audit/asia-northeast1/gke-tokyo)から確認することができます。

### **Sleep Pod のデプロイ**
sleep というサービスメッシュ外の Namespace 上に Pod をデプロイし、メッシュ外からのアクセスの挙動を確認します。
```bash
kubectl create ns sleep
kubectl apply -f mtls/sleep.yaml
```

Sleep pod から Frontend サービスへ正常にアクセスできることを確認します。(ステータスコード 200 が返ってくる)
```bash
export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS frontend.default -o /dev/null -s -w '%{http_code}\n'
# 想定される出力：
# 200
```

### **mTLS を適用する**
`PeerAuthentication` リソースで mTLS を STRICT mode で適用します。  
デフォルトの構成では対象 Namespace を `istio-system` にすることで、サービスメッシュ全体に対して mTLS を STRICT mode で設定することができます。
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
kubectl apply -f mtls/mtls-meshwide.yaml
```

### **STRICT mode の確認**
[Anthos Security ダッシュボード](https://console.cloud.google.com/anthos/security/policy-audit/asia-northeast1/gke-tokyo)上でサンプル アプリケーションの mTLS mode が STRICT になっていることを確認します。  

Sleep pod から Frontend サービスへアクセスすると STRICT mode により平文でのアクセスがエラーとなることを確認します。
```bash
export SLEEP_POD=$(kubectl get pod -n sleep -l app=sleep -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n sleep -c sleep $SLEEP_POD -- curl -sS frontend.default -o /dev/null -s -w '%{http_code}\n'
# 想定される出力：
# curl: (56) Recv failure: Connection reset by peer
```

## **Congraturations!**

<walkthrough-conclusion-trophy></walkthrough-conclusion-trophy>

これにて Cloud Workstations を利用した開発者目線でのアプリケーション実行、デプロイ、管理者目線でのカスタマイズ、本番提供にむけたプラクティスの学習が完了しました。

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
cd $HOME && rm -rf gcp-getting-started-lab-jp gopath
```