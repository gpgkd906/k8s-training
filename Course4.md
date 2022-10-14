k8s：Hardening Project

# Podの管理と通信の管理
k8sでは、Podを通してコンテナーを動かしているので、通常のVMと比べてはセキュリティーに強いが、不十分なところもある。
また、システム全体的に、複数サービスを構成すると考えるとき、セキュリティーを含めて、可用性などを考慮した堅牢化する
ことは求められる。
今回では、これらのテーマについて勉強していきましょう。
* PodのResources
* PodのSecurity Context
* クラスターの通信

## Pod: リソース管理
metrics-serverをインストールして、クラスターのリソース使用量を見えるようにする

```
k create ns handson
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
k top node
```
stressアプリをデプロイして、使っているリソースを見てみる
```
k create -f course4/pod/stress.deploy.yaml
k -n handson get pod --watch
k -n handson top pod
```
podのリソースを変更してみる
```yaml
/# vim course4/pod/stress.deploy.yaml
...
      containers:
      - image: vish/stress
        name: stress
-        resources: {}
+        resources: 
+          limits:
+            cpu: "1"
+            memory: "1Gi"
+          requests:
+            cpu: "0.5"
+            memory: "200Mi"
+       args:
+       - -cpus
+       - "2"
+       - -mem-total
+       - "1950Mi"
+       - -mem-alloc-size
+       - "100Mi"
+       - -mem-alloc-sleep
+       - "1s"
```
変更を適用して、リソースの消費量の推移をみていこう
```
k replace -f course4/pod/stress.deploy.yaml
k top node
k -n handson top pod
```
## Pod: Security Context
ハンズオン2回目では、クラスタレベルで認証や権限設定などを勉強しましたが、
Security ContextはPodやコンテナーレベルの特権設定や権限昇格阻止ができます。

```
k create -f course4/pod/security-context.pod.yaml
k -n handson get pod
k -n handson exec -it security-context -- sh
/# ps
```
Podにはまた他に設定することができるが、この二つはセキュリティに関して重要なのでぜひ覚えましょう。

# クラスター: 通信の管理
クラスターに高度なネットワーク管理を行うために、ネットワークプラグインをインストールする必要がある。

```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-operator.yaml
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-crs.yaml
```

course4/network から複数名前空間のオブジェクトを立ち上げる。
```
k create ns handson2
k create -f course4/network/
```

## CoreDNS
クラスタ内部の通信では、CoreDNSを利用することで、ホスト名を利用してアクセスすることができる。

立ち上げているオブジェクトのIPアドレスを確認する。
```
k -n handson get all -o wide
k -n handson2 get all -o wide
```

もう一つのターミナルを立ち上げて、nettoolのpodから、targetのpodの通信を行って見る

``` 
k run nettool --image=ubuntu -- sleep infinity
k exec -it nettool -- bash
/#nettool: apt-get update ; apt install curl dnsutils -y
/#nettool: curl {service ip}
/#nettool: curl {pod ip}
/#nettool: cat /etc/resolv.conf
/#nettool: dig -x 10.100.0.10
/#nettool: dig -x {service ip}
```

digで見つけたホスト名をいろいろ試してみる
```
/#nettool: curl nginx.handson.svc.cluster.local.
/#nettool: curl nginx.handson2.svc.cluster.local.
/#nettool: curl nginx.handson
/#nettool: curl nginx.handson2
/#nettool: curl nginx
```

最初のbashに戻って、dnsの設定を見ていく
```
k -n kube-system get all
k -n kube-system get deployment.apps/coredns -o yaml
k -n kube-system get configmap coredns -o yaml
```

configmapにrewriteルールを追加してみる
```yaml
...
/# k -n kube-system edit configmap coredns
  Corefile: |
    .:53 {
        rewrite name regex target nginx.handson.svc.cluster.local
        rewrite name regex target2 nginx.handson2.svc.cluster.local
        errors
```
configmapを更新したらcorednsのpodを削除して設定を反映させる
```
k -n kube-system get all
k -n kube-system delete {coredns pod1} {coredns pod2}
```

nettoolのbashに戻って、再度curlしてみる
```
/#nettool: curl target
/#nettool: curl target2
```

最初のbashに戻って、configmapにrewriteルールを拡張してみる
```yaml
...
/# k -n kube-system edit configmap coredns
  Corefile: |
    .:53 {
        rewrite name regex target(\d*) nginx.handson{1}.svc.cluster.local
        errors
```
configmapを更新したらcorednsのpodを削除して設定を反映させる
```
k -n kube-system get all
k -n kube-system delete {coredns pod1} {coredns pod2}
```
nettoolのbashに戻って、再度curlしてみる
```
/#nettool: curl target
/#nettool: curl target2
```

## Network Security Policy
Network Security Policyは、kubernetesにおいて、
IPアドレスまたはポートレベルでトラフィックフォローを制御できます。

まずは、現在のnsp設定を見てみる。
```
vim -R course4/network/handson.nsp.yaml
```

handsonに入るトラフィックを全て遮断してみる
```yaml
/# vim course4/network/handson.nsp.yaml
...
  policyTypes:
  - Ingress
#   - Egress
#   ingress:
#   - {}
#   egress:
#   - {}
```
適用してみる
```
k delete -f course4/network/handson.nsp.yaml
k create -f course4/network/handson.nsp.yaml
```
nettoolから handsonのnginx をアクセスできないことを確認する
```
/#nettool: curl target
/#nettool: curl target2
```

もうちょっと細かいルールを設定してみよう、
nettoolからのみアクセスできるようにする。
まず、もう一つのnettool2を立ち上げます。
``` 
k run nettool2 --image=ubuntu -- sleep infinity
k exec -it nettool2 -- bash
/#nettool2: apt-get update ; apt install curl -y
/#nettool2: curl target
/#nettool2: curl target2
```

最初のbashに戻って、handsonのnspを変更してみる
```yaml
/# vim course4/network/handson.nsp.yaml
...
  policyTypes:
  - Ingress
  - Egress
  ingress:
+  - from:
+      - ipBlock:
+          cidr: {nettoolのip}/32
  egress:
  - {}
```
変更後のnspを適用してみる
```
k delete -f course4/network/handson.nsp.yaml
k create -f course4/network/handson.nsp.yaml
```

nettoolのbashに戻って、nettoolから handsonのnginx をアクセスできることを確認する
```
/#nettool: curl target
/#nettool: curl target2
```

nettool2のbashに戻って、nettool2から handsonのnginx をアクセスできないことを確認する
```
/#nettool: curl target
/#nettool: curl target2
```

# 最終回：　クラスターを診断しよう、そしてアプリを公開しよう
いよいよ次回が最終回となります。
最終回では、クラスタを構築したらどのように診断し、診断結果をどのように対応すべきかを見ていきましょう。
また、複数のアプリを簡単にデプロイするためのツールhelmについても見ていきましょう
最後に、複数のアプリをデプロイして、公開してみましょう