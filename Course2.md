dive to kubernetes

前回はハンズオンを通して、下記k8sの基本操作と基本概念を理解しました:

* kubectl、Namespace、Pod、ReplicaSet、DaemonSet、Deployment、Service

今回は、kubernetesのより深い部分にダイブして見っていきます。

* Secret  
    パスワードやトークン、キーなどの少量の機密データを含むオブジェクトのことです。Secretを使用すれば、アプリケーションコードに機密データを含める必要がなくなります。
* ConfigMap  
    機密性のないデータをキーと値のペアで保存するために使用されるAPIオブジェクトです。Podは、環境変数、コマンドライン引数、またはボリューム内の設定ファイルとしてConfigMapを使用できます。ConfigMapを使用すると、環境固有の設定をコンテナイメージから分離できるため、アプリケーションを簡単に移植できるようになります。
* Kubernetes API  
    Kubernetesの中核であるControl PlaneはAPI serverです。APIサーバーは、エンドユーザー、クラスターのさまざまな部分、および外部コンポーネントが相互に通信できるようにするHTTP APIを公開します。
* ServiceAccount  
    Podに付与し、アプリケーションが利用するアカウント
* RBAC  
    kubernetesクラスタのリソースへのアクセスを制御する方法です。
    ユーザー（およびサービスアカウント）の権限管理です。

前回より、少し難しくなったと思いますが、セキュリティに関する重要なこともあるので、
頑張っていきましょう。

# 1回目の宿題

EKSでServiceでアプリケーションを公開すると、ALBオブジェクトが自動的に作成されます。
少し時間を経てば、ブラウザなどで見れるようになります。

```
k create ns handson
k create -f course2/nginx.yaml
k create -f course2/nginx.svc.yaml
```

## Secret
Secretを作成する
```bash
k -n handson get secret
k -n handson create secret generic handson-secret --from-literal=name=handson --from-literal=password=handson-password
k -n handson get secret
k -n handson get secret handson-secret -o yaml
echo "xxxxxxxxx" | base64 --decode
```
ここでは、Secretの中身を見えていますが、実体は暗号化されています。  
k8s上全てのオブジェクト情報がetcdに保存されているが、Secretはetcdに保存時および読み取り時に  
暗号化・復号化が行われることができる。  
デフォルトでは暗号化・復号化されてないが、下記のプロバイダーから適切に設定することができる。  
https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#providers  
今回はEKSを使ってますので、既にセキュアな設定になっています。  
・デフォルトetcdが保護されている  
・クラスター作成時に、AWS KMSがエンベロープ暗号化に設定済み  

作成したSecretをpodから利用できるようにする

・環境変数として設定する。
```bash
code course2/nginx.yaml
```
```yaml
      containers:
      - image: nginx
      ...
+        env:
+        - name: HANDSON_PASSWORD
+          valueFrom:
+            secretKeyRef:
+              name: handson-secret
+              key: password
```

・ファイルとしてマウントさせる。
```bash
code course2/nginx.yaml
```
```yaml
      containers:
      - image: nginx
      ...
+        volumeMounts:
+        - mountPath: /handson_password
+          name: handson-secret
+      volumes:
+      - name: handson-secret
+        secret:
+            secretName: handson-secret
```

変更を適用して、コンテナに入って確認する
```
k apply -f course2/nginx.yaml
k -n handson get pod
k -n handson exec -it {pod-name} -- bash
# 環境変数を見る
env
# マウントファイルを見る
ls /handson_password
cat /handson_password/name
cat /handson_password/password
```

## ConfigMap

```bash
k -n handson get configmap
k -n handson create configmap handson-config --from-literal=name=handson --from-literal=env=develop
k -n handson get configmap 
k -n handson get configmap handson-config -o yaml
```

ConfigMapを超活用する面白い使い方を一つ紹介する
```bash
k apply -f course2/configmap.yaml
```
nginx.yamlファイルを変更
```bash
code course2/nginx.yaml
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

## APIによる操作

今までは、全てkubectlでkubernetesを操作したが、kubernetesは本来全ての操作をRestfulApiを通して実行できる。  
kubectlはあくまでApiを呼び出すだけのコマンドツールです。  
今回以降の内容では、一部APIで操作する内容になりますので、まずApiを呼び出す方法を見てみよう。  
そして、APIによるクラスタの操作において、セキュリティ上すごく重要なポイントも押さえておきましょう。  

### Api呼び出すに必要な認証情報を取得(RBAC)

```bash
k -n handson get pod
k -n handson get pod {pod/name} -o yaml
k -n handson exec -it {pod/name} -- bash
cd /var/run/secrets/kubernetes.io/serviceaccount
export TOKEN=$(cat token)
```

kubernetesのApi-ServerのURLを確認する

```bash
kubectl config view | grep server
```

取得した認証情報を使ってAPIを呼び出す

```bash
curl {server}/apis --header "Authorization: Bearer $TOKEN" -k
```

ApiでPodを作成してみよ

```bash
curl {server}/api/v1/namespaces/handson/pods --header "Authorization: Bearer $TOKEN" -k -X POST -H 'Content-Type: application/json' -d "{\"kind\": \"Pod\",\"apiVersion\": \"v1\",\"metadata\": {\"name\": \"curlpod\",\"namespace\": \"handson\",\"labels\": {\"name\": \"examplepod\"}},\"spec\": {\"containers\": [{\"name\": \"nginx\",\"image\": \"nginx\",\"ports\": [{\"containerPort\": 80}]}]}}"
```

RoleおよびRoleBindingを作成する
```bash
k -n handson create role r-pod-admin --verb=get,list,create,update --resource=pod
k -n handson create rolebinding rb-pod-admin --role=r-pod-admin --serviceaccount=handson:default
```

もう一回ApiでPodを作成してみよ

```bash
curl {server}/api/v1/namespaces/handson/pods --header "Authorization: Bearer $TOKEN" -k -X POST -H 'Content-Type: application/json' -d "{\"kind\": \"Pod\",\"apiVersion\": \"v1\",\"metadata\": {\"name\": \"curlpod\",\"namespace\": \"handson\",\"labels\": {\"name\": \"examplepod\"}},\"spec\": {\"containers\": [{\"name\": \"nginx\",\"image\": \"nginx\",\"ports\": [{\"containerPort\": 80}]}]}}"
```

kubectlでpod作成されたか確認してみる
```bash
k -n handson get pod
```

podにサービスアカウントをデフォルトで付与しないようにする

```bash
code course2/nginx.yaml
```
```yaml
    spec:
+      automountServiceAccountToken: false
      containers:
```

変更を適用し確認をする
```bash
k apply -f course2/nginx.yaml
k -n handson get pod
k -n handson get pod {pod/name} -o yaml
k -n handson exec -it {pod/name} -- bash
cd /var/run/secrets/kubernetes.io/serviceaccount
```

# ServiceAccountとRBAC
実際のプロジェクトにおいて、Podからk8sのオブジェクトを操作したい時がある。  
defaultのサービスアカウントを設定しない代わりに、専用のサービスアカウントをちゃんと設定してあげましょう。  
また、RBACも理解し、権限設定をしっかり押さえましょう。  

## ServiceAccountを作成
```bash
k create -f course2/serviceaccount.yaml
```

## Roleを作成
```bash
k create -f course2/role.yaml
```

## ServiceAccountとRoleを紐付けます。
```bash
k create -f course2/rolebinding.yaml
```

## SerciceAccountをPodに設定する
```bash
code course2/nginx.yaml
```
```yaml
    spec:
-      automountServiceAccountToken: false
+      serviceAccountName: handson-account
      containers:
```
```bash
k apply -f course2/nginx.yaml
```

