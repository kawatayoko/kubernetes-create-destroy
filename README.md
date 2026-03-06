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


#　chapter5以降は以下コマンドで二つpodを立ち上げる

## 事前準備
### 準備１） 以下コマンドを実行するにはDocker Desktopを起動しておくこと。
### クラスターの状態確認
kind get clusters
## クラスタ起動
kubectl apply --filename chapter-04/myapp.yaml
kubectl run myapp2 --image=blux2/hello-server:1.0 --namespace default
kubectl apply --filename chapter-05/myapp-label.yaml