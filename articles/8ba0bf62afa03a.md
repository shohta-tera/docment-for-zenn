---
title: "k6をKubernetes内で実行しダッシュボードで結果を確認する"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes","k6","Grafana","負荷テスト"]
published: true
---

## はじめに

この記事は前回紹介の"k6で始める負荷テスト"の続きです。
https://zenn.dev/shorter/articles/e52c0047c4f0c5

k6はCLIツールで、負荷スクリプトをJavascript ES2015/ES6で記載できることやコンテナ化することが可能でありかつ、Extension経由でPrometheusに結果メトリクスを送信することができ、Grafanaのダッシュボードで確認が可能という特徴を有しています。そのためKubernetesとの親和性も高く、なおかつCICDパイプラインに組み込んで継続的に性能を確認可能であることから他のツールと比較して優位性があると感じています。  
そこで今回はk6をKubernetesにJobとしてデプロイして測定した結果をGrafanaのダッシュボードで確認するというところを紹介しようと思います。

全体のソースコードはGitHubに置いています。
https://github.com/shohta-tera/loadtesting-with-k6-prometheus

## 環境

* WSL2: Ubuntu 20
* Docker Desktop: v4.17.1
* Kubernetes: Docker Desktop組み込み

## k6をKubernetesにデプロイして結果をダッシュボードで確認する

### k6のコンテナの準備

k6はコンテナをビルドして適宜ECRにデプロイする想定です。今回はDocker DesktopのKubernetesを使用するため、コンテナイメージはローカルにて管理します。  
k6コンテナのビルドには軽量化のためマルチステージビルドを、またAttackSurfaceの最小化と性能のためAlpineではなくDistrolessイメージを利用します。  
また、Prometheusへの結果メトリクスのアップロードのために、xk6を利用しExtensionをインストールします。  
https://github.com/grafana/xk6-output-prometheus-remote  

今回は前回紹介のダッシュボードとPrometheusをビルドしていますが、Extensionは非常に多く存在しこれらを組み合わせることで効率的にテストを実行できます。  
https://k6.io/docs/extensions/get-started/bundle/

このとき同時に負荷スクリプトもコンテナイメージにコピーしています。  

```Dockerfile
ARG K6_VERSION
ARG XK6_VERSION
ARG K6_PROMTHEUS_VERSION

FROM golang:1.20 as builder
ARG K6_VERSION=v0.42.0
ARG XK6_VERSION=v0.9.0
ARG K6_PROMETHEUS_VERSION=v0.2.0

WORKDIR $GOPATH/src/go.k6.io/k6
ADD . .
RUN go install -trimpath go.k6.io/xk6/cmd/xk6@${XK6_VERSION}
RUN xk6 build \
  --with github.com/grafana/xk6-output-prometheus-remote@${K6_PROMETHEUS_VERSION} \
  --with github.com/szkiba/xk6-dashboard@latest
RUN cp -r k6 $GOPATH/bin/k6
WORKDIR /go/src/app
COPY ./job ./

USER k6:k6
FROM gcr.io/distroless/static-debian11
WORKDIR /app
COPY --from=builder --chown=k6:k6 /go/bin/k6 ./
COPY --from=builder --chown=k6:k6 /go/src/app ./

ENTRYPOINT [ "k6" ]
```

また、負荷スクリプトのなかで、HTTPリクエストやレスポンスのチェック等にタグを付与することで、Grafanaの結果確認時にわかりやすくすることが可能です。以下は非常にシンプルな例でk6の試す用のエンドポイントにリクエストを投げて、レスポンスが200であることを確認するものです。

```Javascript
import { check } from "k6"
import { Trend } from "k6/metrics"
import http from "k6/http"

const myTrend = new Trend("Test Trend")

export const options = {
    vus: 10,
    duration: '30s',
    thresholds: {
        http_req_failed: ["rate<0.01"],
        http_req_duration: ["p(90)<2000"]
    }
}

export default function () {
    const headers = {
        "Content-Type": "application/json"
    }

    const res = http.post(
        "https://test.k6.io",
        { headers: headers },
        {
            tags: {
                senario: "actual request"
            }
        }
    )
    check(res, {
        'is_status_200': (r) => r.status === 200
    }, { check_response: "Check response" })

    myTrend.add(res.timings.connecting, { custom_metric: "connection" })
}
```

### Prometheusにメトリクスを送信する準備

PrometheusにはPrometheusにメトリクスを送信することができます。エンドポイントは基本的には、`http://{service of Prometheus}.{Namespace of Prometheus}.svc.cluster.local:80/api/v1/write`です。この機能はデフォルトで有効化されておらず、Flagとして有効化してやる必要があります。  
PrometheusのインストールはHelmを使います。  
https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus

Remote Writeの機能の有効化のために、values.yamlを一部編集します。

```yaml
server:
  extraFlags:
    - web.enable-lifecycle
    - web.enable-remote-write-receiver
    - enable-feature=native-histograms
```

#### Grafanaをインストールする

GrafanaのインストールにはPrometheusと同様にHelmを利用します。  
https://github.com/grafana/helm-charts/tree/main/charts/grafana

ここでもPrometheusからデータを取得するため一部values.yamlを編集します。

```yaml
datasources:
  datasoucers.yaml:
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      url: http://prometheus-server:80
      access: proxy
      isDefault: true
```

### k6をKubernetesのJobとしてデプロイする

k6の実行にはHelmを用いてJobとしてデプロイします。各種環境変数などはJobの中のコマンドで指定します。また、シナリオの一部でログインなどの処理が必要でパスワード等が必要な場合は、Secretとしてパスワードなどを作成します。また、ほかの環境変数もConfigMapとして登録して自動的に使用することも可能です。  

```yaml job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: k6-load-testing
spec:
  ttlSecondsAfterFinished: 300
  template:
    spec:
      containers:
      - name: k6-prometheus
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command:
          [
            "./k6",
            "run",
            "-e",
            "BASE_URL=$(BASE_URL)",
            "-e",
            "K6_PROMETHEUS_RW_SERVER_URL=$(K6_PROMETHEUS_REMOTE_URL)",
            "-e",
            "K6_PROMETHEUS_RW_TREND_AS_NATIVE_HISTOGRAM=true",
            "-e",
            "K6_OUT=xk6-prometheus-rw",
            "./simple_http.js"
          ]
        env:
        - name: K6_PROMETHEUS_REMOTE_URL
          value: {{ .Values.conf.PROMETHEUS_URL }}
        envFrom:
        - configMapRef:
            name: k6-target
        - secretRef:
            name: k6-auth-secret
      restartPolicy: Never
  backoffLimit: 4
```


## Grafanaによる結果の確認

k6のダッシュボードはk6のチームが提供する公式のダッシュボードが存在します。  
https://grafana.com/grafana/dashboards/18030-test-result/  


![Overview](/images/8ba0bf62afa03a/overview.png)

ダッシュボードでは上記のように、トータルのリクエスト、RPS、レスポンスタイムなどが確認できます。  

![Check response](/images/8ba0bf62afa03a/check.png)

ステータスのチェックでは指定のステータスコードの割合を確認することもできます。複数のステータスチェックもここで確認できます。

![Request timing](/images/8ba0bf62afa03a/request_time.png)

HTTPリクエストについても、レスポンスタイムの最小最大平均95パーセンタイルなどが見れます。これもHTTPリクエストのターゲットごとで確認も可能です。  

## 終わりに

k6は非常にシンプルで使いやすいツールで、現状の限界値を探るだけでなく、問題点を評価することができます。CICDパイプラインやCLIでも使いやすいことから定期的な性能確認においてもってこいなツールです。  
OpenTelemetryなどを組み合わせることで、ボトルネックの改善や課題の解決がしやすくもなります。  