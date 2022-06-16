今回はハンズオンを通して、k8sの基本操作と基本概念を理解する:
* kubectl
* Namespace
* Deployment
* Pod
* Service
* ReplicaSet
* DaemonSet

など

またk8sの構築やアーキテクチャーは今回触らない。


## namespaceを見る
namespaces一覧を見る

```bash
k get ns
// kubectl get namespace
```

namespaceを作成

```bash
k create ns handson
// kubectl create namespace handson
```

## シンプルのアプリをデプロイする
名前空間handsonにアプリをデプロイする

```bash
k -n handson create deploy nginx --image=nginx
// kubectl -n handson create deployment nginx --image=nginx
```
名前空間handsonにデプロイしたアプリを見る

```bash
k -n handson get deploy
```
全ての名前空間にデプロイしたアプリを見る

```bash
k get deploy --all-namespaces
```
名前空間handsonにデプロイしたアプリの詳細をみる

```bash
k -n handson describe deploy nginx
```
名前空間handsonにデプロイしたアプリをyamlファイルに書き出して、再生できるように編集する

```bash
k -n handson get deploy nginx -o yaml > nginx.yaml
code nginx.yaml
```
* creationTimestamp 削除
* resourceVersion 削除
* uid 削除
* status以降 削除

もしくは、リソースを作成せずに、yamlを書き出す

```bash
k -n handson create deploy nginx --image=nginx --dry-run=client -o yaml > nginx.yaml
```

名前空間handsonにデプロイしたアプリを削除する
```bash
k -n handson delete deploy nginx
```
更新したyamlファイルからアプリをもう一回デプロイする

```bash
k create -f nginx.yaml
```

## アプリを外部からアクセスできるようにする
名前空間handsonにあるオブジェクトを見る（deploy, pod, serviceなど）

```bash
k -n handson get all
k -n handson get all -o wide
```

nginxのpodの中からアクセスしてみる

```bash
k -n handson exec -it nginx-xxxxxxx -- /bin/bash
curl 127.0.0.1
```

外からアクセスできるようにexposeしてみる...

```bash
k -n handson expose deploy nginx
```

k explainを覚えましょう！

```bash
k explain ...
```

podのポートを開ける

```bash
code nginx.yaml
```
* ports 追加
* containerPort: 80　追加
* protocol: TCP 追加
* 上記項目をどこに追加するかは、k explainを使って特定する

更新したyamlファイルを適用

```bash
k apply -f nginx.yaml
```
ポートを開けたデプロイをexposeする

```bash
k -n handson expose deploy nginx
k -n handson get svc
// kubectl -n handson get service
k -n handson get svc nginx -o yaml > nginx.svc.yaml
```
serviceの設定を更新する

```bash
code nginx.svc.yaml
```
* typeをLoadBalancerに変更
* nodePort: 32080　追加
* 上記項目をどこに変更あるいは追加するかは、k explainを使って特定する

serviceを更新する

```bash
k -n handson replace -f nginx.svc.yaml
```


### スケールアウトしてみる
scaleコマンドを使ってみる

```bash
k -n handson scale deploy nginx --replicas=2
```
デプロイを更新する

```bash
k -n handson edit deploy nginx
```
デプロイの更新履歴をみる

```bash
k -n handson rollout history deploy/nginx
k -n handson rollout history deploy/nginx --revision=2
```
デプロイをロルバックする

```bash
k -n handson rollout undo deploy/nginx --to-revision=2
```

### レプリカを理解する
レプリカセットを作ってみる

```bash
k create -f course1/rs.yaml
```

レプリカセットの状態を確認する
podは複数存在するが、均等にnodeに分布していないことを注目
```bash
k -n handson get rs,pod -o wide
```

### DaemonSetを理解する
DaemonSetのyamlファイルを作成

```bash
cp course1/rs.yaml course1/ds.yaml
code course1/ds.yaml
```
* replicas 削除
* ラベルのsystem: ReplicaSample を system: DaemonsetSample に変更（２箇所ある）

DaemonSetを作成

```bash
k create -f course1/ds.yaml
```

DaemonSetの状態を確認する
replicasの設定がないが、必ず各nodeに対象のpodが存在することを確認
```bash
k -n handson get ds,pod -o wide
```

## APIによる操作
今までは、全てkubectlでkubernetesを操作したが、kubernetesは本来全ての操作をRestfulApiを通して実行できる。

kubectlはあくまでApiを呼び出すだけのコマンドツールです。

次回以降の内容を進む前に、Apiを呼び出す方法を見てみよう。

### Api呼び出すに必要な認証情報を取得

```bash
less $HOME/.kube/config
export client=$(grep client-cert $HOME/.kube/config |cut -d" " -f 6)
echo $client | base64 -d - > ./client.pem
export key=$(grep client-key-data $HOME/.kube/config |cut -d " " -f 6)
echo $key | base64 -d - > ./client-key.pem
export auth=$(grep certificate-authority-data $HOME/.kube/config |cut -d " " -f 6)
echo $auth | base64 -d - > ./ca.pem
```

kubernetesのApi-ServerのURLを確認する

```bash
kubectl config view | grep server
```

取得した認証情報を使ってAPIを呼び出す

```bash
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://0.0.0.0:6443/api/v1/pods
```

ApiでPodを作成してみよ

```bash
curl --cert ./client.pem --key ./client-key.pem --cacert ./ca.pem https://0.0.0.0:6443/api/v1/namespaces/handson/pods -X POST -H 'Content-Type: application/json' -d@course1/pod.json
```

Podが作成されていることを確認する

```bash
k -n handson get pod
```

考えてみよう：kubectlを実行するとき、認証情報を使っているのでしょうか？
