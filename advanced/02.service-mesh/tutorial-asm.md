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

<walkthrough-info-message>GKE クラスタをすでにデプロイしている場合、`container.googleapis.com` の有効化は不要です。</walkthrough-info-message>

**GUI**: [API ライブラリ](https://console.cloud.google.com/apis/library)

## **マネージド Anthos Service Mesh のプロビジョニング**

Fleet API を利用してマネージド Anthos Service Mesh (ASM) をプロビジョニングします。
ASM のプロビジョニングは他にも [asmcli](https://cloud.google.com/service-mesh/docs/managed/provision-managed-anthos-service-mesh-asmcli?hl=ja) を利用した方法もありますが、Fleet API を利用して Managed ASM をプロビジョニングすることをお勧めします。Fleet API 経由でマネージド ASM をプロビジョニングすると Managed Data Plane (MDP) も有効化されます。

<walkthrough-info-message>[Fleet](https://cloud.google.com/kubernetes-engine/docs/fleets-overview?hl=ja) は複数の GKE クラスタのグルーピングや複数クラスタに対する各種機能の有効・無効を制御する機能です。</walkthrough-info-message>

### **1. 環境変数・gcloud コマンドの設定**

```bash
export CLUSTER_NAME1=gke-tokyo
export CLUSTER_LOCATION1=asia-northeast1

gcloud container clusters get-credentials ${CLUSTER_NAME1} \
     --region ${CLUSTER_LOCATION1} 
```

### **2. クラスタを Fleet に登録する**
Fleet で GKE クラスタを管理するために、Fleet へ登録します。
```bash
gcloud container fleet memberships register ${CLUSTER_NAME1} \
  --gke-uri=https://container.googleapis.com/v1/projects/${PROJECT_ID}/locations/${CLUSTER_LOCATION1}/clusters/${CLUSTER_NAME1} \
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
  --region ${CLUSTER_LOCATION1} --update-labels mesh_id=proj-${PROJECT_NUM}
```

### **3. マネージド ASM のプロビジョニング**
Fleet による自動管理を有効にし、マネージド ASM をプロビジョニングします。（プロビジョニング完了まで数分かかります）
```bash
gcloud container fleet mesh update \
    --management automatic \
    --memberships ${CLUSTER_NAME1} \
    --project ${PROJECT_ID} \
    --location ${CLUSTER_LOCATION1}
```

数分後、コントロール プレーンのステータスが ACTIVE になっていることを確認します。
```bash
gcloud container fleet mesh describe --project ${PROJECT_ID}
```

以下のように `controlPlaneManagement` が `ACTIVE` になっていることを確認します。
```
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
IP_ADDR=$(gcloud compute addresses list --format='value(ADDRESS)' --filter="NAME:gatewayip")
DOMAIN="${IP_ADDR//./-}.nip.io"

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

## **チャレンジ問題**

### **Python 用拡張機能をプリインストールする**

Python が社内の公式開発言語の１つになっているとします。

ワークステーションの利用ユーザーから Python 用の拡張機能 (ms-python) をプリインストールしておいてほしいと要望があがりました。

今まで学んできたカスタマイズの手順を参考に、作成済みのワークステーション構成 (codeoss-customized) に Python 用拡張機能をプリインストールしてみましょう。

## **本番で Cloud Workstations を利用する**

ここまで Cloud Workstations を開発者、管理者目線から利用する方法を学んできました。

しかし、本番で Cloud Workstations を利用する際に検討、意識すべき事項があります。ここからは以下のようなポイントをそれぞれ説明します。

- 開発者個別にワークステーションを払い出す
- カスタムコンテナイメージをセキュアに保つ
- 無駄な費用がかからないようにする

## **開発者個別にワークステーションを払い出す**

<walkthrough-info-message>本手順は作業している Google アカウントとは**別の** Google アカウントを擬似的に開発者のアカウントとして利用します。アカウントがない場合、こちらのステップはスキップしてください</walkthrough-info-message>

### **1. 開発者にどのような権限を与えるか検討する**

権限を制御することで、ワークステーションを開発者にどのように利用させるかを制御することが可能です。

一般的に以下 2 つの提供形態のいずれかが選ばれます。

1. 管理者がワークステーションを払い出し、開発者はそれを利用する
1. 開発者自身がワークステーションを作成し、利用する

本ステップでは `1. 管理者がワークステーションを払い出し、開発者はそれを利用する` の手順を説明します。`2. 開発者自身がワークステーションを作成し、利用する` を試す場合は次のステップに進んでください。

### **2. 専用のカスタムロールを作成する**

開発者がワークステーションを削除できないようにするためには、専用のカスタムロールが必要です。

```shell
cat << EOF > workstationDeveloper.yaml
title: "Workstations Developer"
description: "Developer who only uses workstations"
stage: "GA"
includedPermissions:
- workstations.operations.get
- workstations.workstations.get
- workstations.workstations.start
- workstations.workstations.stop
- workstations.workstations.update
- workstations.workstations.use
EOF
gcloud iam roles create workstationDeveloper \
  --project ${PROJECT_ID} \
  --file workstationDeveloper.yaml
```

### **2. 開発者に作成したカスタムロールを付与する**

こちらは GUI から設定します。

1. Cloud Workstations の UI にアクセスします。
1. `ワークステーション` をクリックします。
1. 一覧から `ws-customized` をクリックします。
1. 上のメニューから `ADD USERS` をクリックします。
1. 新しいプリンシパルに開発者のメールアドレスを入力します。
1. ロールにカスタム -> `Workstations Developer` を選択します。
1. `保存` ボタンをクリックします。

### **3. 開発者に Cloud Workstations Operation 閲覧者 の権限を付与する**

`test-developer@gmail.com` を開発者アカウントのメールアドレスに置き換えて実行してください。

```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member "user:test-developer@gmail.com" \
  --role "roles/workstations.operationViewer"
```

### **4. (開発者) ワークステーションの管理コンソールにアクセスする**

下記コマンドで出力された URL に **開発者アカウント** でアクセスします。

```bash
echo "https://console.cloud.google.com/workstations/list?project=$PROJECT_ID"
```

先程権限を付与したワークステーションのみが見え、以下の操作ができれば問題なく設定ができています。

- 起動
- 停止

管理者は本手順を繰り返すことで、開発者に安全にワークステーションを利用させることができます。

## **開発者自身がワークステーションを作成し、利用する**

### **1. 開発者に Cloud Workstations 作成者 の権限を付与する**

こちらは GUI から設定します。

1. Cloud Workstations の UI にアクセスします。
1. `ワークステーションの構成` をクリックします。
1. 一覧から `codeoss-customized` をクリックします。
1. 上のメニューから `ADD USERS` をクリックします。
1. 新しいプリンシパルに開発者のメールアドレスを入力します。
1. ロールが `Cloud Workstations Creator` になっていることを確認します。
1. `保存` ボタンをクリックします。

### **2. 開発者に Cloud Workstations Operation 閲覧者 の権限を付与する**

`test-developer@gmail.com` を開発者アカウントのメールアドレスに置き換えて実行してください。

```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member "user:test-developer@gmail.com" \
  --role "roles/workstations.operationViewer"
```

### **3. (開発者) ワークステーションの管理コンソールにアクセスする**

下記コマンドで出力された URL に **開発者アカウント** でアクセスします。

```bash
echo "https://console.cloud.google.com/workstations/list?project=$PROJECT_ID"
```

### **4. (開発者) ワークステーションを作成、アクセスする**

1. `+ 作成` をクリックします。
1. `名前` に `ws-developer` と入力します。

   ```shell
   ws-developer
   ```

1. `Configuration` から `codeoss-customized` を選択します。
1. `作成` をクリックします。
1. 作成が完了したら、`START`, `LAUNCH` からアクセスできることを確認します。

管理者は本手順を繰り返すことで、開発者に安全にワークステーションを利用させることができます。

## **カスタムコンテナイメージをセキュアに保つ**

カスタムコンテナイメージをセキュアに保つには、定期的にビルドをし直す必要があります。

Google が提供している [事前構成されたベースイメージ](https://cloud.google.com/workstations/docs/preconfigured-base-images?hl=ja) は、定期的にセキュリティ更新を含むアップデートが行われています。

そのため、これらのイメージ (の latest) をそのまま使い続ける場合、ワークステーションが停止、起動するときに自動的に最新のイメージが使われます。

しかし、今回行った手順のように、コンテナをカスタマイズした場合、ビルドした段階でのベースイメージで固定されてしまうため、カスタムイメージも定期的にビルド (ベースイメージの更新を取り込む) 必要があります。

詳細は [コンテナ イメージの再ビルドを自動化してベースイメージの更新を同期する](https://cloud.google.com/workstations/docs/tutorial-automate-container-image-rebuild?hl=ja) を参照してください。

## **無駄な費用がかからないようにする**

ワークステーション構成を見直すことで、トータルのコストをおさえる事が可能です。

ここではいくつか有用な設定を示します。

**参考**: [開発環境をカスタマイズする](https://cloud.google.com/workstations/docs/customize-workstation-configurations?hl=ja)

### **1. 仮想マシンのタイプ**

ワークステーション構成作成後に**更新可能**です。

利用可能な仮想マシンタイプは [使用可能なマシンタイプ](https://cloud.google.com/workstations/docs/available-machine-types?hl=ja) を参照してください。

### **2. ブートディスクのサイズ**

**注**: ワークステーション構成作成後は**更新できません**。

ブートディスクのサイズです。デフォルトは 50GB で、最小で 30GB の指定が可能です。

### **3. 永続ディスクのタイプ、サイズ**

**注**: ワークステーション構成作成後は**更新できません**。

永続ディスク (ホームディレクトリ) のディスクタイプ、サイズです。デフォルトは `pd-standard` タイプ、`200GB` が指定されています。

200GB 未満に指定するときは、ディスクタイプに `pd-balanced`, `pd-ssd` のいずれかを指定しなければなりません。

### **4. アイドル時の自動停止時間**

ワークステーション構成作成後に**更新可能**です。

ワークステーションが一定時間アイドル状態になったときに、自動的にストップする機能があります。デフォルトで `20分 (1200s)` が設定されています。

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