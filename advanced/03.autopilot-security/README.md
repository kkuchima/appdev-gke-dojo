
export PROJECT_ID=kkuchima-sandbox
export CLUSTER_NAME=test-ap01
export CLUSTER_REGION=asia-northeast1

gcloud container clusters create-auto ${CLUSTER_NAME} --region ${CLUSTER_REGION} --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE

# 失敗するパターン
kubectl apply -f https://k8s.io/examples/application/deployment.yaml

nginx の
Cloud Logging 上で確認
```bash
gcloud logging read --order="desc" --freshness=7d ""

resource.type="k8s_cluster"
jsonPayload.message=~"Denied by always_deny admission rule"
```
gcloud container clusters update test-api01 --region asia-northeast1 --binauthz-evaluation-mode=DISABLED

Sample は nginx で順番は
1. runAsNonroot 構成の Online Boutique をデプロイ
2. GKE Security Posture で検出
3. マニフェスト修正してデプロイ
4. Binary Authorization のポリシー変更
5. nginx をデプロイ→
6. 

# Service Mesh のハンズオン
