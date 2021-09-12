---
title: "Airflowでpython以外で作成したジョブを実行する"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Airflow", "Kubernetes", "Container", "python", "Nodejs"]
published: true
---

# はじめに

# 環境
- WSL2: Ver20
- Helm: v3.6.1
- Airflow: ver2

# 前回の記事

[前回の記事](https://zenn.dev/articles/dcf694061bb4bf)では、KubernetesにAirflowを導入していました。今回はその続きで、AiflowのJobについてです。

## 記事の内容

Airflowでは、ジョブをpythonで記述でき、依存関係をDAGとしてかけます。今回はそのジョブをpythonではなく、独自のContainer Imageを用いて実行する方法を記載しようと思います。
今回の記事で紹介したコードは以下のレポジトリにあります。
https://github.com/shohta-tera/workflows

## TL;DR

独自のContainer Imageを使用する際には、`KubernetesPodOperator`を用いて、実行する。

# Kubernetesにおいてpython以外での、ジョブの実行方法

python以外で記述されたジョブを実行する手段は、主に3種類あります。
- BashOperator
- DockerOperator
- KubernetesPodOperator

## Bash Operator

Bash Operatorでは、Bashシェル内で、任意のコマンドを実行できます。
コード例:
```python
run = BashOperator(
    task_id="run_with_nodejs",
    bash_command="/usr/bin/node /usr/local/aiflow/dags/job.js"
)
```
この場合だと、DAGを実行するWorker内で、node.jsのパッケージの依存性やモジュールのインストールなどの管理が必要となります。単一のジョブであるならば、問題はないかと思われますが、これが同一モジュールで複数のバージョンを別フォルダで管理するとなるとそこの管理コストは大きくなるのは容易に想像できます。  
そこでおすすめしたいのが、python以外のジョブをContainer Image化して、実行する。です。

## ジョブをContainer Image化する。

ジョブをDockerfileなどを用いてContainer Image化することで、以下のような利点があります。  
ジョブの構成や依存性を含めてコード化できることです。 これにより、CICDのパイプラインを構築していれば、ビルドやテストなど自動化できます。また、ジョブ実行のたびにコンテナが生成されるので、ジョブ実行環境が綺麗なままであることもメリットとして挙げられます。

### DockerOperator

前回紹介の記事で、KubernetesにAirflowを導入しているため、本記事では、DockerOperatorについては触れません。参考記事として、Docker Composeを利用してAirflowとDockerOperatorを利用したDAGを記載しておきます。
[参考記事](https://towardsdatascience.com/using-apache-airflow-dockeroperator-with-docker-compose-57d0217c8219)

### KubernetesPodOperator

AirflowはGithubからDAGを取得する設定にしております。そのレポジトリの構造は以下のようになっています。
```
./
├── README.md
├── requirements.txt
├── setup.cfg
└── src
    └── dags
        ├── get_dags_from_directries.py
        └── kubernetes_pod_operator
            └── sample_dag.py
```
`kubernetes_pod_operator`以下に今回のKubernetesPodOperatorを使用するDAGを配置しています。
Airflowは環境変数で設定したフォルダ直下のDAGしか読み取ってくれません。  
そのため、サブディレクトリを切ったDAGも読み取ってくれるようなDAGをdagsフォルダ直下に配置します。

```python:get_dags_from_directries.py
import os

from airflow.models import DagBag

# Need to specify directry
dags_dirs = ["./kubernetes_pod_operator"]

for dags_dir in dags_dirs:
    dag_bag = DagBag(os.path.expanduser(dags_dir))

    if dag_bag:
        for dag_id, dag in dag_bag.dags.items():
            globals()[dag_id] = dag
```
この実装は参考記事の実装から引用しています。
[参考記事](https://future-architect.github.io/articles/20200131/#8-dags-%E3%83%87%E3%82%A3%E3%83%AC%E3%82%AF%E3%83%88%E3%83%AA%E4%BB%A5%E4%B8%8B%E3%81%AE%E3%83%87%E3%82%A3%E3%83%AC%E3%82%AF%E3%83%88%E3%83%AA%E3%81%AE%E5%88%87%E3%82%8A%E6%96%B9)

### KubernetesPodOperatorを使用したジョブ

一つずつ説明していきたいと思います。

```python
default_args = {"owner": "sample", "retries": 2}
volume_mount = k8s.V1VolumeMount(
    name="sample-data", mount_path="/sample-data", sub_path=None
)
volume = k8s.V1Volume(name="sample-data", empty_dir={})
env = os.getenv("ENV", "local")

if env != "local":
    account_id = os.getenv("ACCOUNT_ID")
    image_tag = os.getenv("IMAGE_TAG", "1.0.0")
    image = f"{account_id}.dkr.ecr.us-west-2.amazonaws.com/test/nodejobs:{image_tag}"
    image_pull_secrets = [k8s.V1LocalObjectReference("aws-registry")]
else:
    image = "test/nodejobs:1.0.0"
    image_pull_secrets = []
```
- default_args: ここではAirlfowのUIに表示するOwnerであったり、ジョブのリトライ回数をここで個別に設定できます。
- volume_mount, volume: Podにディレクトリをマウントすることも可能です。今回の例では、empty_dirを用意しています。
- image: ここではローカル環境とそれ以外で分けています。ローカル以外では、ECRからContainer Imageを取得して来る想定です。同時にImage取得用の認証情報をSecretから取得してきています。
そのためリモートクラスターで実行する際には、別途Image取得用のSecretの準備が必要になります。

```python
with models.DAG(
    dag_id="node_jobs",
    schedule_interval=None,
    start_date=days_ago(1),
    is_paused_upon_creation=False,
    catchup=False,
) as dag:
```
ここではDAGの定義をしています。
- dag_id: DAGの名前です。
- schedule_interval: ジョブ実行の間隔です。今回は手動実行想定なので、Noneにしています。
- start_date: 後述のcatchupと組み合わせて、開始時刻から現在まで上記の実行間隔でジョブを実行します。
- is_paused_upon_creation: DAGの作成時に、Pause状態か否かです。
- catchup: ここをTrueにすると、初回のジョブ実行時に開始時刻からさかのぼってジョブをすべて実行します。

```python
node_jobs = KubernetesPodOperator(
        task_id="test_task",
        name="test",
        cmds=["node", "services/service/nodejobs.js"],
        arguments=["{{ dag_run.conf }}"],
        namespace="jobs",
        image_pull_secrets=image_pull_secrets,
        volumes=[volume],
        volume_mounts=[volume_mount],
        env_vars={"DB_USER": os.getenv("DB_USER")},
        annotations={"sidecar.istio.io/inject": "false"},
        resources={
            "request_cpu": "200m",
            "request_memory": "256Mi",
            "limit_cpu": "1000m",
            "limit_memory": "1Gi",
        },
```
ここでタスクの定義をしていきます。
- cmd: ここで、独自イメージのRun時のコマンドを記載します。
- arguments: ここで、DAGを手動実行する際にパラメータを指定できるので、それを読み取る際はここで`dag_run.conf`で取得しておきます。
- resources: ここで、各Podのリソースのリクエストとリミットを指定します。

## おわりに

以上でKubernetesPodOperatorを使った、ジョブの定義の紹介でした。
AirflowのExecutorをKubernetesExecutorで指定した際に、一点だけ気をつけなければ行けない点があります。  
KubernetesExecutorでは、他のExecutorとは異なりRedisなどのリソースが不要でかつ、ジョブを実行するごとにWorker(Pod)が起動されます。起動時にWorkerのInitializationの処理が走り、各ジョブごとに綺麗な環境で実行できることや不必要なリソースがいらないなどのメリットがあります。  
今回のKubernetesPodOperatorでは、Workerが起動したあとに、独自のImageでpullなどの処理が走るため、ジョブ実行から実際にContainerが起動してRunの処理が走るまでに比較的大きな時間がかかります。そのため、ジョブのIntervalが比較的短いものであったり、ジョブの実行時間に制約を課しているものがあれば、KubernetesExecutorを用いるのは現実的ではないかもしれません。その際には、`CeleryKubernetesExecutor`を指定してやるなど、Workerを常に用意しておくなどの対処が必要になるかと思われます。  
今回は一般的なPythonのOperatorを使用したAirflowのジョブ実行ではなく、python以外の独自のコンテナを使用した場合のケースを紹介しました。
次回は、Airflow ver2で追加されたTask flow APIやDAGをDynamicに生成する手法の紹介などしていきたいと思います。  