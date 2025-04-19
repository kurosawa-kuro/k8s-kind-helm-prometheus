# k8s-kind-helm-prometheus チュートリアル

> **目的**: kind で構築したシングルノード Kubernetes クラスタ上に、
> - **Helm** で `kube‑prometheus‑stack` (Prometheus Operator + Grafana)
> - **Helm** で ECR から取得した **Node.js API (container-nodejs-api-8000)**
>
> をデプロイし、ServiceMonitor を介してアプリケーションメトリクスを収集するまでを 1 時間以内で完了できる最小手順を提供します。

---

## 0. 前提

| 項目 | バージョン例 |
|------|--------------|
| OS   | Ubuntu 22.04 / Amazon Linux 2023 |
| kind | v0.23.0 |
| kubectl | v1.29.x |
| Helm | v3.14.x |
| Docker | 24.0 以降 |
| AWS CLI | v2  (ECR 認証用) |

> **作業ディレクトリ:** `~/dev/k8s-kind-helm-prometheus`
>
> **GitHub Repo:** <https://github.com/kurosawa-kuro/k8s-kind-helm-prometheus>
>
> **ECR イメージ:** `986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000:latest`
>
> **Node.js API Git:** <https://github.com/kurosawa-kuro/container-nodejs-api-8000>

```bash
mkdir -p ~/dev && cd ~/dev
git clone https://github.com/kurosawa-kuro/k8s-kind-helm-prometheus.git
cd k8s-kind-helm-prometheus
```

---

## 1. kind クラスタ作成

`kind-config.yaml`（1 ノード + extraPortMappings で Grafana をホスト 3000 に公開）

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30000  # Grafana (NodePort)
        hostPort: 3000
```

```bash
kind create cluster --name monitoring --config kind-config.yaml

# kubeconfig を確認
kubectl cluster-info --context kind-monitoring
```

---

## 2. Helm リポジトリ追加

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 3. Prometheus Operator スタックのインストール

> **namespace:** `monitoring`

```bash
kubectl create namespace monitoring

helm install kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 70.7.0 \
  --set grafana.service.type=NodePort \
  --set grafana.service.nodePort=30000 \
  --set grafana.adminPassword="admin" \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

> ⏱ **所要 2–3 分**

### Grafana へのアクセス方法

Grafana にアクセスするには、以下のいずれかの方法を使用します：

#### 方法1: ポートフォワーディング（推奨）

```bash
# 別のターミナルで実行
kubectl port-forward svc/kps-grafana 3000:80 -n monitoring
```

そして、ブラウザで以下のURLにアクセスします：
```
http://localhost:3000  (user: admin / pass: admin)
```

#### 方法2: NodePort経由（Kindクラスタの設定に依存）

Kindクラスタの設定で`extraPortMappings`が正しく設定されている場合、以下のURLでアクセスできます：
```
http://localhost:3000  (user: admin / pass: admin)
```

#### 方法3: リモートサーバーからのアクセス

リモートサーバー上で実行している場合、SSHポートフォワーディングを使用します：

```bash
# ローカルマシンから実行
ssh -L 3000:localhost:3000 ユーザー名@リモートサーバーのIP
```

そして、ローカルマシンのブラウザで以下のURLにアクセスします：
```
http://localhost:3000  (user: admin / pass: admin)
```

---

## 4. Node.js API 用 Helm チャート

### 4-1. Helm チャートの作成

```bash
# Helm チャートの作成
helm create nodejs-api

# values.yaml の編集
cat > nodejs-api/values.yaml << 'EOF'
# Default values for nodejs-api.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.0.4"

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: false
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: ClusterIP
  port: 8000

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
  scaleDownDelaySeconds: 300
  scaleUpDelaySeconds: 0

nodeSelector: {}

tolerations: []

affinity: {}
EOF

# deployment.yaml の編集
cat > nodejs-api/templates/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nodejs-api.fullname" . }}
  labels: { app: nodejs-api }
spec:
  replicas: 1
  selector:
    matchLabels: { app: nodejs-api }
  template:
    metadata:
      labels: { app: nodejs-api }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
EOF

# service.yaml の編集
cat > nodejs-api/templates/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nodejs-api
  labels: { app: nodejs-api }
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
spec:
  type: ClusterIP
  selector: { app: nodejs-api }
  ports:
    - port: 8000
      targetPort: 8000
EOF

# servicemonitor.yaml の作成
cat > nodejs-api/templates/servicemonitor.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nodejs-api
  labels:
    release: kps   # kube-prometheus-stack リリース名
spec:
  selector:
    matchLabels:
      app: nodejs-api
  endpoints:
    - port: 8000
      path: /metrics
      interval: 15s
EOF

# 不要なファイルの削除
rm -f nodejs-api/templates/ingress.yaml nodejs-api/templates/hpa.yaml nodejs-api/templates/notes.txt nodejs-api/templates/test-connection.yaml
```

### 4-2. Helm チャートの構造

作成されたHelmチャートの構造は以下のようになります：

```
nodejs-api/
├─ Chart.yaml
├─ values.yaml
└─ templates/
     ├─ deployment.yaml
     ├─ service.yaml
     ├─ servicemonitor.yaml
     ├─ _helpers.tpl
     ├─ NOTES.txt
     └─ _serviceaccount.yaml
```

`nodejs-api/Chart.yaml`
```yaml
apiVersion: v2
name: nodejs-api
version: 0.1.0
appVersion: "1.0.0"
dependencies: []
```

`nodejs-api/values.yaml`
```yaml
# Default values for nodejs-api.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  tag: "v1.0.4"

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: false
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name: ""

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: ClusterIP
  port: 8000

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
  scaleDownDelaySeconds: 300
  scaleUpDelaySeconds: 0

nodeSelector: {}

tolerations: []

affinity: {}
```

`nodejs-api/templates/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nodejs-api.fullname" . }}
  labels: { app: nodejs-api }
spec:
  replicas: 1
  selector:
    matchLabels: { app: nodejs-api }
  template:
    metadata:
      labels: { app: nodejs-api }}
    spec:
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8000
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
```

`nodejs-api/templates/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodejs-api
  labels: { app: nodejs-api }
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
spec:
  type: ClusterIP
  selector: { app: nodejs-api }
  ports:
    - port: 8000
      targetPort: 8000
```

`nodejs-api/templates/servicemonitor.yaml`
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nodejs-api
  labels:
    release: kps   # kube-prometheus-stack リリース名
spec:
  selector:
    matchLabels:
      app: nodejs-api
  endpoints:
    - port: 8000
      path: /metrics
      interval: 15s
```

> `/metrics` エンドポイントが Node.js アプリに実装されている前提です。

---

## 5. Node.js API デプロイ

### 5‑1. (オプション) ECR へログイン & イメージを kind に読み込む

```bash
aws ecr get-login-password --region ap-northeast-1 | \
  docker login --username AWS --password-stdin 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com

docker pull 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000:v1.0.4

# kind 内にイメージを転送
kind load docker-image 986154984217.dkr.ecr.ap-northeast-1.amazonaws.com/container-nodejs-api-8000:v1.0.4 --name monitoring
```

### 5‑2. Helm リリース

```bash
helm install api nodejs-api --namespace monitoring
```

---

## 6. 動作確認

### 6-1. Node.js API の確認

```bash
# Pod 状態
kubectl get pods -n monitoring -l app=nodejs-api

# ポートフォワーディングでAPIにアクセス
kubectl port-forward svc/nodejs-api 8000:8000 -n monitoring &
curl http://localhost:8000/healthz
```

### 6-2. Prometheus の確認

```bash
# Prometheus Targets
kubectl port-forward svc/kps-kube-prometheus-stack-prometheus 9090 -n monitoring &
open http://localhost:9090/targets   # nodejs-api が UP になっているか確認
```

### 6-3. Grafana での監視設定

1. Grafanaへのアクセス
```bash
# ポートフォワーディングを使用する場合（別のターミナルで実行）
kubectl port-forward svc/kps-grafana 3000:80 -n monitoring &
# ブラウザで http://localhost:3000 にアクセス（user: admin / pass: admin）
```

2. Node.js APIの監視メトリクス

以下のメトリクスを「Explore」画面で確認できます：

- HTTPリクエスト数の確認
```promql
rate(http_requests_total[5m])
```

- レスポンスタイムの確認
```promql
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

- メモリ使用量の確認
```promql
nodejs_memory_usage_bytes
```

- CPU使用時間の確認
```promql
rate(process_cpu_user_seconds_total[5m])
```

3. ダッシュボードの作成手順

a. 新規ダッシュボード作成
   - 左側メニュー → 「+ New Dashboard」をクリック
   - 「Add visualization」をクリック
   - データソースで「Prometheus」を選択

b. パネルの追加（以下は推奨メトリクス）
   - リクエスト数パネル
     - メトリクス: `rate(http_requests_total[5m])`
     - パネルタイプ: Time series
     - 説明: 5分間の平均リクエスト数

   - レスポンスタイムパネル
     - メトリクス: `rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])`
     - パネルタイプ: Time series
     - 説明: 5分間の平均レスポンスタイム

   - メモリ使用量パネル
     - メトリクス: `nodejs_memory_usage_bytes`
     - パネルタイプ: Time series
     - 説明: Node.jsプロセスのメモリ使用量

   - CPU使用率パネル
     - メトリクス: `rate(process_cpu_user_seconds_total[5m])`
     - パネルタイプ: Time series
     - 説明: 5分間のCPU使用率

### 6-4. Node.js API のエンドポイント

Node.js API には以下のエンドポイントが実装されています：

- `/healthz` - Kubernetes ヘルスチェック用エンドポイント
- `/metrics` - Prometheus メトリクスエンドポイント（アプリケーションの各種メトリクスを提供）
- `/api-docs` - Swagger UI ドキュメント
- `/` - ルートエンドポイント（ヘルスチェック）

---

## 7. 後片付け

```bash
helm uninstall api kps -n monitoring
kind delete cluster --name monitoring
```

---

## 参考リンク

- kube‑prometheus‑stack Helm Chart <https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack>
- Node.js API サンプル <https://github.com/kurosawa-kuro/container-nodejs-api-8000>

---

### 完了 🎉

これで **Helm と Prometheus Operator、Node.js API の ServiceMonitor 連携** までを kind 上で検証できました。継続的なチューニングや GitOps 化が必要になった際は、`values.yaml` と `charts/` ディレクトリをそのまま Git リポジトリへ push し、Argo CD にアプリケーションとして登録するだけで本番移行の土台になります。

