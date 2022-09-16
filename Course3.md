Storage

永続的なストレージは、重要なアプリケーションにとって必要不可欠なものです。
kubernetesにおいても重要、そして難しい概念となっています。

2回目のコースでは、secretやconfigmapをpodにvolumeとして、ファイルとしてマウントするハンズオンをやってましたが、
今日は、ストレージにフォーカスして、より汎用的なストレージの使い方を勉強します。

kubernetesのような分散化システムでは、ストレージをどのように存在しているか、
そしてどのようにpodに与えたり、複数のpodでストレージを共有したりする方法論から、
ストレージの用途に合わせて、書き込み専有だったり種類別の設定方法をハンズオンして行きます。

# kubernetesにおけるストレージの考え方

dockerやkubernetesで動いているコンテナでは、現状コンテナ側でコンテナよりも長生きするストレージが提供されていないんです。
つまり、コンテナーが実行している間にファイルシステムにファイルを書き込んでも、コンテナが終了すると共に、ファイルが消滅してしまいます。
永続的にファイルやデータを保存するために、コンテナの外にあるストレージをコンテナにマウントする必要があります。
dockerなら、もっとも簡単な方法では、ローカルファイルシステムをコンテナにマウントすることです。
しかし、kubernetesではそう簡単な話しではありません。
複数のノードを持つクラスターにおいて、コンテナが落ちて再生する場合、同じノードで再生する保証はないんです。
むしろノードのリソース不足によるコンテナの死亡や再生は別ノードにて再生されることを意味する。
その場合、もちろんローカルファイルシステムは別物となり、データがなくなったり、壊れたりもする。
さらに、kubernetesでは、ノード間のファイルシステムが違うこともありえるので、同じパス指定してマウントしても
実際に存在しないパスだったりすることも考えられる。
まとめると、コンテナにおけるストレージの問題は、コンテナがクラッシュしたときにファイルが失われること、そしてPodで一緒に実行されているコンテナ間でファイルを共有すること。 Kubernetesボリュームの抽象化は、これらの問題の両方を解決します。

・Volume
Podと同じライフタイムを共有するストレージの抽象化となり、同じpod上にある複数コンテナがそれを共有できる。

・Persistent Volume
PersistentVolume (PV)はストレージクラスを使って管理者もしくは動的にプロビジョニングされるクラスターのストレージの一部です。これはNodeと同じようにクラスターリソースの一部です。PVはVolumeのようなボリュームプラグインですが、PVを使う個別のPodとは独立したライフサイクルを持っています。
基本的に、nodeと同じライフタイムを共有するストレージの抽象化となり、podが終了後にもデータが保存される。
また、クラウドサービスがContainer Storage Interface (CSI)に準拠し提供したものはnodeのライフタイムを超えてデータを保存する。

・Persistent Volume Claim
PersistentVolumeClaim (PVC)はユーザーによって要求されるストレージです。
PVCはPVリソースを消費し、特定のサイズやアクセスモード（例えば、ReadWriteOnce、ReadOnlyMany、ReadWriteMany)を要求することができます。
「要求する」というのは、下記なことを意味する。
あるクラスタにて、ユーザーから35GBiのPVCを要求する場合、その要求を満たせる最も適切なストレージを与えられる。
例えばクラスタ上、10GBi, 20GBi, 50GBiのPVが存在するなら、この場合、50GBiのストレージが与えられる。

・StorageClass
StorageClass は、管理者が提供するストレージの「クラス」を記述する方法を提供します。さまざまなクラスが、サービス品質レベル、バックアップ ポリシー、またはクラスター管理者によって決定された任意のポリシーにマップされる場合があります。Kubernetes 自体は、クラスが何を表すかについて意見を持っていません。この概念は、他のストレージ システムでは「プロファイル」と呼ばれることがあります。

今回は、ストレージを中心に勉強しながらハンズオンをして行きます。
最後に、databaseをクラスターにデプロイしましょう。

```
k create ns handson

```

# 2回目の復習
configmapを作成して、コンテナにファイルとしてマウントする

```
k -n handson create configmap handson-config --from-literal=name=handson --from-literal=env=develop
```
```yaml
      containers:
      - image: nginx
      ...
        volumeMounts:
+        - name: nginx-index-file
+          mountPath: /usr/share/nginx/html/
      volumes:
+      - name: nginx-index-file
+        configMap:
+          name: index-html-configmap
```

# 本編開始

```
# nodeのローカルストレージを使って 永続的なストレージ を作成する
```
k apply -f course3/local_sample/
kubectl get pods --watch
k get pv,pvc,storageclass
kubectl describe pv {pv name}
k exec -it app -- cat /data/out.txt
```

作成したリソースを削除する

```
k delete -f course3/local_sample/
```

# EKSにEBS（Elastic Block Store）のPVを作成する

クラスタ名およびリージョンを取得
```
k config view
> - cluster:
>     ...
>   name: __ClusterName__.__Region__.eksctl.io
```

IAM OIDC Providerを取得
```
aws eks describe-cluster --name __ClusterName__ --query "cluster.identity.oidc.issuer" --output text
> https://oidc.eks.ap-northeast-1.amazonaws.com/id/__OidcProvider__
```

AWS AccountIdを取得
```
aws iam get-user
> {
>     "User": {
>         ...
>         "Arn": "arn:aws:iam::__AccountId__:user/xxxx",
>         ...
>     }
> }
```

AmazonEKS_EBS_CSI_Driver_Policy という IAM ポリシーを作成
```
aws iam create-policy --policy-name __PolicyName__ --policy-document file://course3/example-iam-policy.json
> {
>     "Policy": {
>         ...
>         "Arn": "__PolicyArn__",
>         ...
>     }
> }
```
AmazonEKS_EBS_CSI_DriverRole を作成する

```
aws iam create-role \
  --role-name __RoleName__ \
  --assume-role-policy-document file://course3/trust-policy.json
> {
>     "Role": {
>         ...
>         "Arn": "__RoleArn__",
>         ...
>     }
> }
```

新しい IAM ポリシーをロールにアタッチします

```
aws iam attach-role-policy \
--policy-arn __PolicyArn__ \
--role-name __RoleName__
```

Amazon EBS CSI ドライバーをデプロイする

```
k apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
```

EBS CSI Controllerのサービスアカウントに、ロールのリソースネームをアノテーションつける

```
k annotate serviceaccount ebs-csi-controller-sa \
  -n kube-system \
  eks.amazonaws.com/role-arn=__RoleArn__
```

EBS CSI Controllerのpodを削除して、再生させる

```
k delete pods -n kube-system -l=app=ebs-csi-controller
```

ebsのサンプルを適用してみる

```
k apply -f course3/ebs_sample/
kubectl get pods --watch
k get pv,pvc,storageclass
kubectl describe pv {pv name}
k exec -it app -- cat /data/out.txt
```

# クラスターにmariadbをデプロイしてみる

```
k apply -f course3/mariadb/
kubectl get pods --watch
k get pv,pvc,storageclass
kubectl describe pv {pv name}
```

3回目お疲れ様です！

# podとコンテナ
今回ストレージを勉強しました。
kubernetesのストレージでは、
・コンテナーのライフタイムを共有するもの
・podのライムタイムを共有するもの
・nodeのライムタイムを共有するもの
など存在していることがわかる。

では、podとは何か？コンテナーと違うけど、どう違うか、とみんなは思うと思います。
次回、4回目はpodを詳しく見て行きたいのでまたよろしくお願いします。