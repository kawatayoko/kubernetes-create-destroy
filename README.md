# kubenetes
- Control Panel
    - kube-apiserver
        - kubectlコマンド
- Worker Node
    Worker Nodeのことを略してNodeという

# さまざまなKubernetesクラスタ構築環境
- ローカルクラスタ
    - minikube
    - kind
    - k3s
- ベンダー
    - Google Kubernetes Engine（GKE）
    - Amazon Elastic Kubernetes (EKS) 
    - Azure Kubernetes Service (AKS)

## kubernetesクラスタ環境構築
kubectlを使うためにクラスタ環境を構築しないといけない
### kindを利用してクラスタ環境を構築する
事前にdocker desctopを立ち上げておく必要がある。
```
# クラスタ構築
kind create cluster --image=kindest/node:v1.29.0
# kubectlを利用してクラスタと接続できることを確認
kubectl cluster-info --context kind-kind
```
- --contextオプションで利用するクラスタのコンテキストを指定する。
- contextが一つの場合は指定しなくてOK
- contextを切り替えることでクラスタ毎に仕様するconfigの内容を使い分ける

### kindを利用してクラスタ環境を削除
```
kind delete cluster --name kind
```

# pod
kubenetesの最小構成単位
podは複数コンテナをまとめて起動できる

# マニフェストをKubernetesクラスタに適応する
kubectl apply --filename chapter04/myapp.yaml --namespace default
# リソース情報の取得
`kubectl get <リソース名>`
kubectl get pod --namespace default
## outputオプション
kubectl get pod --output wide --namespace default
kubectl get pod --output yaml --namespace default
### outputオプションにjsonpath
kubectl get pod myapp --output jsonpath='{.spec.containers[].image}' --namespace default
kubectl get pod myapp --output json --namespace default | jq '.spec.containers[].image'
## ログレベルを指定
kubectl get pod myapp --v=7 --namespace default 
## リソースの詳細を取得
kubectl describe pod myapp --namespace default
## コンテナログを取得する
kubectl logs myapp --namespace default
## ラベルを指定
kubectl get pod --selector app=myapp
kubectl logs --selector app=myapp
## debug用コンテナを立ち上げる
デバッグ用コンテナを起動することで、多様なデバッグツールを利用できるようになる。
デバッグ用コンテナのイメージは任意のイメージを指定できるので、自分用カスタムコンテナイメージを作っても良い
```
# kubectl debug --stdin --tty <デバッグ対象Pod名> --image=<デバッグ用コンテナのimage> --target=<デバッグ対象のコンテナ名>
kubectl debug --stdin --tty myapp --image=curlimages/curl:8.4.0 --target=hello-serer --namespace default -- sh
```
## コンテナを即座に起動する（kubectl run）
kubectl debug以前は、kubectl runを使ってPodを起動する必要があった
```
# kubectl run <Pod名> --image=<イメージ名>
kubectl --namespace default run busybox --image=busybox:1.36.1 --rm --stdin --tty --restart=Never --command -- nslookup google.com
```
--rm : 実行が完了したらPodを削除
--stdin(i) : オプションで標準入力に渡す
--tty(t): オプションで擬似端末を割り当てる
--restart=Never デフォルトでは常に再起動するポリシーのため、一回きりのコマンドを実行する場合、指定する必要がある
--command --: "--" の後に渡される文字列をコマンドとして扱う

## コンテナにログイン
```
# kubectl exec --stdin --tty <Pod名> -- <コマンド名>
kubectl --namespace default run curlpod --image=curlimages/curl:8.4.0 --command -- /bin/sh -c "while true; do sleep infinity; done;"
kubectl get pod --namespace default
# IPアドレスを確認
kubectl get pod myapp --output wide --namespace default
# curlpodに入ってcurlを実行
kubectl --namespace default exec --stdin --tty curlpod -- /bin
/sh
~ $ curl 10.244.0.5:8080

```

## port-forwardでアプリケーションにアクセス
```
# kubectl port-forward <Pod名> <転送先ポート番号>:<転送元ポート番号>
kubectl port-forward myapp 5555:8080 --namespace default
# 確認(別ターミナルで)
curl localhost:5555
```

# 障害対応のためのkubectl コマンド
## マニフェストをその場で編集
```
# kubectl edit <リソース名>
kubectl edit pod myapp --namespace default
```
## リソースを削除する
```
# kubectl delete <リソース名>
# kubectlにはPodを再起動するというコマンドがないため代替される

# pod確認
kubectl get pod --namespace default
# pod削除
kubectl delete pod myapp --namespace default
```
## kubectrlのチートシート
https://kubernetes.io/docs/reference/kubectl/quick-reference/

## kubectrl役立つツール
- k9s  
    https://k9scli.io/topics/commands/
- starship  
    https://starship.rs/
- kubectx/kubens

# Chapter　6　Kubernetesリソースを作って壊そう
## 6.2 Podを冗長化するためのReplicaSetとDeployment
- 実際の運用現場ではPodを直接作ることは推奨されていない
    - Pod単体ではコンテナの冗長化ができないので
- DeploymentとReplicaSet
    - ReplicaSetは複数Podをまとめたもの
    - Deploymentは複数ReplicaSetをまとめたもの
    ```mermaid
    graph LR

    RS1[ReplicaSet] --> P1[Pod]
    RS1 --> P2[Pod]

    D[Deployment]

    D -->|v2| RS2[ReplicaSet]
    D -->|v1| RS3[ReplicaSet]

    RS2 --> P3[Pod]
    RS2 --> P4[Pod]

    RS3 --> P5[Pod]
    RS3 --> P6[Pod]
    RS3 --> P7[Pod]
    ```
### ReplicaSet
- 指定した数のPodを複製するリソース
- 複製するPodの数をreplicasで指定する
- ReplicaSet作成のコマンド
```sh
# Replica作成
kubectl apply --filename chapter-06/replicaset.yaml --namespace default

# Pod確認
kubectl get pod --namespace default

# ReplicaSetのリソース確認
kubectl get replicaset --namespace default

# ReplicaSetのリソース削除
kubectl delete replicaset --namespace default
```
### Deployment
- 本番の運用環境ではPod更新時に「無停止で更新する」ことが必要
- Deployementは、ReplicaSetを複数管理し、ローリングアップデートを実現できる

```sh
# deploymentを作成
$ kubectl apply --filename chapter-06/deployment.yaml --namespace default
deployment.apps/nginx-deployment created

# deploymentの作成を確認
$ kubectl get deployment --namespace default
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           84s

# Podが作成されていることを確認
$ kubectl get pod --namespace
 default

# ReplicaSetが作成されていることを確認
$ kubectl get replicaset --namespace default

# 作成したマニフェストを参照
# StrategyType, RollingUpdateStrategy を参照できる
kubectl describe deployment nginx-deployment
```
- StrategyType
    - どのような先約でPodを更新するか？を定義
    - Recreate
        - 全部のPodを同時に更新
    - RollingUpdate
        - Podを順番に更新
        - RollingUpdateStrategyを記載できる

### 6.2.3 作って直す　Deploymentをつくって壊そう
- 親子関係の整理
    - Deployment > ReplicaSet > Pod > Container
- RollingUpdate時の設定項目
    - maxUnavailable: 更新中に利用不能になってよいPodの最大数。小数点以下は切り下げ
    - maxSurge: 更新中に一時的に増やしてよいPodの最大数。小数点以下は切り上げ
    - Replicas: Pod数

# 6.3 Podへのアクセスを助けるService
- Service
    - Deploymentで作成した複数Podへのアクセスを適切にルーティングするために利用するリソース
    - Type
        - ClusterIP: 
            クラスタ内部のIPアドレスでServiceを公開    
            ClusterIPはクラスタ内部からしか疎通できない。
            外部公開するにはIngressというリソースを利用する
            -> クラスタ内部とは？
        - NodePort:
            全ておNodeのIPアドレスで指定したポート番号を公開する
            クラスタ外からもアクセス可能。port-fowardする必要がなくなる。
            Nodeが故障すると使えなくなるので、本番環境ではClusterIPやLoadBalancerを利用するほうがよい
        - LoadBalancer:
            外部ロードバランサを用いて外部IPアドレスを公開
            LBは別途よういする必要あり
        - ExternaName:
            ServiceをexternalNameフィールドの内容にマッピングする
            （たとえば、ホスト名api.example.com）
    - Serviceを利用したDNS
        - KubernetesではService用のDNSレコードを自動で作成してくれる
            <Service名>.<Namespace名>.svc.cluster.local
            で接続可能
            ```
            kubectl apply --filename chapter-06/service.y
aml --namespace default
            kubectl apply --filename chapter-06/deploymen
t-hello-server.yaml --namespace default
            kubectl --namespace default run curl --image curlimages/curl --rm --stdin --tty --restart=Never --command -- curl hello-server-service.default.svc.cluster.local:8080
            ```
# 6.4 Podの外部から情報を読み込むConfigMap
- ConfigMap
    - コンテナの外部から値を設定したいときに利用するリソース
- 利用方法(2と３がよく使われる)
    1. コンテナ内のコマンドの引数として読み込む
    2. コンテナの環境変数として読み込む
        - 変更後アプリケーションの再作成が必要
        ```
        kubectl rollout restart deployment/hello-server --namespace d
efault
        ```        
    3. ボリュームを利用してアプリケーションのファイルとして読み込む
        - アプリケーションの再作成不要
        - ボリューム
            Pod間でファイルを共有できるファイルシステム
        
- コンテナの環境変数として読み込む

# 6.5 機密データを扱うためのSecret
- Secretリソース
    - Base64でエンコードして登録

# 6.6 1回限りのタスクを実行するためのJob
- kind: Job
# 6.7 Jobをテウイキ的に実行するためのCronJob
- CronJobリソース
    - 定期的にJobを生成するリソース
    - CronJobはJobを作成、JobはPodを作成する

# 7.1 アプリケーションのヘルスチェックを行う
- Readiness probe
    - コンテナがReadyになるまでの時間やエンドポイントを制御する
    - 失敗したらServiceから接続をはずす
- Liveness probe
    - Readminess probeとの違いは失敗時の挙動
    - 失敗時はPodを再起動する
        - 再起動を無限に繰り返してしじゃうリスクがあるため安易に導入しない
        - 再起動でなおるケースが想定される場合に有効
    - Readiness probeとの同時設定は可能
        - Readiness probeが先に実行されることが推奨される
            - initialDelaySeconds で調整する
- Startup probe
    - Pod初回起動時のみに使用するProbe

# 7.2 アプリケーションに適切なリソースを指定しよう
CPU・メモリ・EphemeralStrage　が指定できる
- Resource requests
    - コンテナのリソース使用量（下限）を要求する
    - kubernetesのスケジューラはこの値を参照してスケジュールするNodeを決定する
        - Nodeは物理サーバー、EC2、仮想マシンなどのPodをうごかすマシンのこと
        - KubernetesはPodをどのNodeで動かすかを自動で決める
            - その決定をするのがscheduler
    - コンテナごとにCPUとメモリを指定できる
- Resource limits
    - コンテナのリソース使用量（上限）を指定する
    - CPUが上限を超えた場合、スロットリングが発生する
    - メモリが上限を超えた場合、Out Of Memory(OOM) でPodはkillされる
- メモリ
    - 単位を指定しない場合、1 = 1byte
    - K, M, Ki, Mi などを指定可
- CPU
    - 単位を指定しない場合、1 = 1コア
        - 1m = 0.001 コア
        - 通常、整数 or ミリコアで指定する
- PodのQuality of Service (QoS) Classes
    - Nodeのメモリが完全に枯渇すると、Nodeに載っている全てのコンテナが動作できなくなる
        - それを防止するためOOM Killerというプログラムがある
        - OOMKillerは、OOMkillするPodの優先順位を決定し、優先度の低いPodからOOMkillする
    - 優先順は以下
        1. Guaranteed
            Pod内のコンテナすべてにリソースのrequestsとlimitsが指定されている
            メモリとCPUの値が、requests = limitsとなっている 
        2. Burstable
            Pod内のコンテナのうち、すくなくとも一つはメモリorCPUのrequests/limitsが指定あれている
        3. BestEffort
            Guaranteed, Burstableでないもの
    - 以下のコマンドでQoSクラスを確認できる
        `kubectl get pod hello-server --output jsonpath='{.status.qosClass}' --namespace default`

# 7.3 Podのスケジュールに便利な機能
NodeとPodの関係性の制御
- Node selector
    - 特定のNodeにのみスケジュールする
        SSDをつかっているNodeにのみ`disktype: ssd`というラベルが付与されている場合
        `nodeSelector: ssd`とマニフェストで指定することで、SSDを使っているNodeにのみPodをスケジュールできる
- Affinity/Anti-affinity
    - Affinity：類似性、密接な関係
    - NodeとPod、あるいはPod同士が「近くなるように」「近づかないように」スケジュールする
    - 3種類ある
        - Node affinity
            - Node selectorと近い。Node selectorより柔軟に設定できる。
            - 「可能ならスケジュールする」という選択ができる
            - `requiredDuringSchedulingIgnoredDuringExecution`
                - 対応するNodeが見つからない場合、Podをスケジュールしない
                - Node selectorと同じ考え方だが、Nodeの指定方法より柔軟
            - `preferredDuringSchedulingIgnoredDuringExecution`
                - 対応するNodeが見つからない場合、適当なNodeにスケジュールする
            - `matchExpressions`を利用してNodeを指定する
        - Pod affinity / Pod anti-affinity
            - spec.affinity以下にpodAddinity/podAntiAffinityを指定する
            - Pod間のAffinity
            - すでにNodeにスケジュールされている既存Podのラベルに基づいて新規Podのスケジューリングする
            - ユースケース
                - Node障害に備えて「同じアプリケーションを動かしているPodは同じNodeに乗せない」
            - Node affinityと同様に以下のルールを設定可能
                - `requiredDuringSchedulingIgnoredDuringExecution`
                - `preferredDuringSchedulingIgnoredDuringExecution`
        - Pod Topology Spread Constraints
            - Podを分散するための設定
            - topologyKeyを使うことでどのようにPodを分散させるかを表現できる
            - hostnameラベルを指定するとホスト間でPodを分散してスケジュールできる
            - Pod anti-affinitiyでもおなじような設定ができるが、より柔軟に設定可能
            - Pod Topology Spread Constraints ではPod数がNode数を超えてもなるべく分散させる設定ができる
        - Taint / Toleration
            - Taint: Nodeに付与する設定
                - 「汚れ」の意味
            - Toleration: Podに付与する設定
                - 「寛容」の意味
            - Traint（汚れ）をPodが許容できるか？という考えかたの設定
            - あるNodeが特定のPodしかスケジュールしたくない、という指定方法
            `kubectl taint nodes <対象ノード名> <label名>=<labelの値>:<Taintの効果>`
        - Pod PriorityとPreemption
            - Priority
                - PodにPriorityを設定できる
                - PriorityClassというリソースを使って設定する
                    1. PriorityClassを作成する
                    2. 1で設定したPriorityClassをPodのマニフェストに指定する
            - Preemption
                - priorityが低いPodをEvicctさせること
# 7.4 アプリケーションをスケールさせよう
- 水平スケール（Horizontal Pod Autoscaler: HPA）
    - サーバーの台数を増やす
    - Pod数を増やす・減らす
    - HPAを利用するにはmetrics-serverのインストールが必要
    - インストール
    ```
    kubectl apply --filename https://github.com/kubernetes-sigs/metric
s-server/releases/download/v0.6.4/components.yaml
    kubectl patch --namespace kube-system deployment metrics-server --type=json --patch '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
    ```
- 垂直スケール(Vertical Pod Autoscaler: VPA)
    - サーバーの使用リソースを増やす
    - VPAを利用することで自動でResource Requests/Limits の値を変更できる
    - HPAと同じリソースに同時に適応不可
    - なのでHPAのみ利用しているケースがおおい

# 7.5 Nodeの退役に備える
- PodDisruptioBudget(PDB)
    - DeploymentでカバーできるのはあくまでPodを更新するときだけ
    - 本番の運用では、NodeをメンテナンスするためにNodeからPodを退避させるなど、Podが増えたり減ったりするケースがある
    - そういうケースでPodを安全に退避させるための機能がPodDisruptionBudget
    - 直訳すると「Podがはかいされるときの予算」
    - 予算を設定しておくことで、予算を超えないようにKubernetesが制御してくれる
    - 設定項目
        - minAvailable
            最低いくつのPodが利用可能な状態であるか
        - maxUnavailable
            最大でいくつのPodが利用可能なじょうたいであるか

# 9.1 Kubernetesのアーキテクチャ
# 9.3 Control Plane
- kube-apiserver
    RESTで通信可能なAPIサーバー
- etcd
    分散型キーバリューストア
    kubr-apiserverはkubectlからリクエストを受けつけて、etcdにデータを保存している
    kubectl get では etcdに保存してあるデータをkube-apiserverを通じて受け取っている
- kube-scheduler
    PodをNodeにスケジュールする
- kube-controller-manager
    Kubernetesを最低限動かすために必要なコントローラーを動かす
    「マニフェストに書かれている内容に応じて動作する」プログラム全般をコントローラーという
    replicas: 3 の場合に、Podを3つよういするのは、Replication Controllerの仕事
# 9.4 Worker Node
- Worker Nodeは、実際にアプリケーションコンテナの起動を行うNode
- kubelet
    クラスタ内の各Nodeでうごいている
    Podに紐づくコンテナを管理する
    kubeletが起動しているNodeにPodがスケジュールされると、コンテナランタイムに指示してコンテナを起動する
- kube-proxy
    ネットワーク設定を行うコンポーネント
    クラスタ内の各ノード上で動作する
    kube-proxyによって、クラスタ内外のネットワークセッションからPodへのネットワーク通信が可能となる
    Podの増減、Service追加、Endpoint変更を監視し、iptablesルールを同期させる役割（iptablesを書き換えている）
- コンテナランタイム
    containerd, CRI-O
- kubectl
    - kube-apiserverと通信するためのCLIツール
    - kube-apiserverはRESTfulなAPIサーバーなのでcurlでも通信可能

# 9.8 Kubernetesを拡張する方法
- Kubernetesをユーザー自身が拡張できることがKubernetesの特徴の一つ
- 独自リソース
    - Custom Resource (CR)
- 独自リソースを作るために必要な定義
    - Custom Resource Definition(CRD)
- カスタムコントローラー（コントローラー）
    - CRを参照して動くプログラム
    - kube-controller-managerのコントローラと同じ概念
    - Deployment Controllerはkube-controller-manager に内包されている
    - ConfigMapに必要な情報を書いておき、その情報を読むようなプログラムを書けばよいのでは？
        - 判断基準がある
            https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#should-i-use-a-configmap-or-a-custom-resource
            - 公式ドキュメントでは次のうち一つでも当てはまるものがあればConfigMapを使うと良いと書かれています。
                1. 既存のよく文書化された設定ファイル形式がある
                    (ex) mysql.conf, pom.xml
                2. 設定全体をConfigMapの1つのキーに入れたい
                3. Podで動作するプログラムが自身を設定するためにそのファイルを利用する
                4. Kubrnetes APIではなく、PodのファイルやPodの環境変数を通じて利用したい
                5. ファイルが更新されたとき、Deploymentなどを使ってローリングアップデートを実施したい
            - 例えば、`kubectl get <CR名>`で情報を取得したり、宣言的なマニフェストを利用してReconcile処理を走らせたいときなど、Kubernetesの流儀に従ってKubernetesを拡張したいケースにおいてCRを利用するとよい

# 10.1 Kuvernetesにデプロイする
- kubectl apply コマンドを使って継続的にデプロイするには以下の課題がある
    - いつ誰がコマンドを実施したかわからない
    - コマンドの実施によりマニフェストの衝突が発生する
    - 手動での実施によるヒューマンエラーの発生
- Push型のデプロイ方法：CIOps
    - Push -> CI -> `kubectl apply` の実行
    - masterブランチにfeatureブランチがマージされたら本番環境にkubectl applyを実行する、というもの
- Pull型のデプロイ方法：GitOps
    - 宣言的
    - バージョン管理と不変
    - 自動的に取得
    - 継続的な調整
- CIOps vs GitOps
    - CIOps
        - シンプル、わかりやすい、構築しやすい
    - GitOps
        - セキュリティリスクに強い 
            Pull型なので読み取り権限があればよい（書き込み権限不要）
        - CIとCDが分離できる
            専用のデプロイツールを利用し、デプロイ情報を宣言的に管理することでCIと分離することが可能
- GitOpsを実現するためのソフトウェア
    - ArgoCD
        - GitOps用のOSS
        - Applicationという名前のCustom Resourceを利用する
        - どのリポジトリの、どのマニフェストの、どのバージョンのマニフェストをどの環境に適応するか、を指定する
    - Spinnaker
        - Netflix社が開発していたツール
        - Kubernetes以外にも主要なクラウドプロバイダに対応している
    - FluxCD
        - GitOpsを提唱したWeaveworks社が開発していた
        - Argo CDとかなり近い
        - マルチテナンシー
# 10.2 Kubernetesのマニフェスト管理
- Helm
    - Helmはパッケージマネージャー
    - マニフェストを作成する以上のことが可能
    - Chartというテンプレートをもとにhelm installすることでKubernetesクラスタにマニフェストをデプロイする仕組み
    - 他の開発者が開発したカスタムコントローラ用のマニフェストを利用したいケースで特に有効
- Helmをインストール
    brew install helm
- Helm Chart Repositoryを追加
    - Helm Chart Repositoryを追加
    ```
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```
    -  namespace 追加
    ```
    kubectl create namespace monitoring
    ```
- helm installを実行
    - helm install <任意のリソース名> --namespace monitoring <Chart名>
        - `helm install kube-prometheus-stack --namespace monitoring prometheus-community/kube-prometheus-stack`
        - `helm show values prometheus-community/kube-prometheus-stack`
    - `helm install` コマンドを直接実行する方法はGitOpsと相性が悪い
        - ArgoCDなど各GitOpsエージェントを使用して、Helmインストール、マニフェスト管理を行う
- Jsonnet
    - KustomizeやHelmよりもかなり柔軟性が高いツール
- 自作テンプレート
    - どのツールでもうまく当てはまらない場合テンプレートを自作する方法もある
    - 各言語にテンプレートライブラリが存在するので、自分の得意な言語でテンプレートを作ってみてもよい
- Kustomize(カスタマイズ)
    - マニフェストの共通部分はbaseというディレクトリで管理し、環境ごとの差分をoverlaysというディレクトリ内のマニフェストで管理する
    - ArgoCDなどのGitOpsエージェントでサポートされている
    - 最終成果物はkustomizeというツールを利用してビルドする
    - kubectlもkustomizeに対応している
    - 実践編
        ビルドする時にkustomization.yamlを参照する
        - kustomization.yaml
            どのディレクトリ（ファイル）をbaseとするか？
            どのディレクトリ（ファイル）をoverlaysとするか？
            を定義する
            - resources: 
                ベースとなるディレクトリやファイルを記載する
            - pathces:
                overlaysでbaseの設定を上書きする時に使用する
        - kustomize build <ディレクトリ名> ですべてのマニフェストをあとめて1つに出力してくれ
            - `kustomize build ./overlays/staging | kubectl --namespace default apply -f -`
            - `kustomize build ./overlays/statging | kubectl --namespace default delete -f -`

# 11 オブザーバビリティとモニタリング
## モニタリング
あるシステムやそのシステムのコンポーネントの振る舞いや出力を観察し続ける行為（Mike Julian: 入門監視）
## オブザーバービリティ
- 可観測性：外部からシステムがどれくらい観測可能か？
        問題があったときデバッグ用のプログラムを仕込まなくても観測した情報から原因を特定できること
- CNCFのホワイトペーパー
    http://github.com/cncf/tag-observability/blob/main/whitepaper.md
    - システムの出力を「シグナル」と呼ぶ
        - 好みのシグナルを一つ使うところからはじめるとよい
        - 3つのプライマリシグナル
            - Logs
                - 「いつ、なにが起こったか」
                - Kuvernetesではデフォルトでログを収集する仕組みがある
                    - `kubectl logs`
                    - コンテナの標準出力/エラー(stdout/stderr)の内容をコンテナのログとして収集する
                - KubernetesのデフォルトのログはPodがNodeから削除されると消えてしまう
                - ログを永続化する仕組みが必要
                    - クラウドベンダーのログ保存・kdんさくサービス
                    - Fluentd
                    - Fluentbit
                    - Grafana Loki
            - Metrics
                - 測定値の集合
                - 「1日に何件アクセスがあったか」を調べるためにログファイルを開いて件数を数えるのは現実的ではない
                - Kubernetesには標準でメトリクスを収集する機能はない
                    - Datadog
                    - Prometheus
                        PromQLというクエリ言語でメトリクスを参照できる
            - Trace
                - 通信を追跡する
                - 複数のSpan（操作の終業）からなるコンテナを複数稼働させている状況で環境障害が発生したとき、ユーザーからのリクエストがどのコンテナ経由したかを正確にたどる必要が出てきた時に利用できる
                - リクエストが通る経路上の全てでTraces/Spanが導入できている必要がある
                - Tracesを導入するためにはアプリケーションの実装に手をいれる必要がある
                    - どこからどこまでをTracesとして扱い、Spanとして扱うか？
                    - OpenTelemetry
                        - 実装を開花するためのOSSライブラリ
                - 実装したTacesを収集する代表的なOSSとして
                    - Jaeger
                    - Grafana Tempo
        - 2つの新規シグナル
            - Profiles
            - Dumps
        - プライマリシグナルから使うと良い
# モニタリング
- ダッシュボード
    - Grafana
- 異常を知らせる「アラート」
    - アラートにはMetricsを使うと良い
        「監視入門」
# 11.3
- Prometheus
    - /metrics というエンドポイントに対してアクセスする

#　chapter5以降は以下コマンドで二つpodを立ち上げる

## 事前準備
### 準備１） 以下コマンドを実行するにはDocker Desktopを起動しておくこと。
### クラスターの状態確認
kind get clusters
## クラスタ起動
kubectl apply --filename chapter-04/myapp.yaml
kubectl run myapp2 --image=blux2/hello-server:1.0 --namespace default
kubectl apply --filename chapter-05/myapp-label.yaml