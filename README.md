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