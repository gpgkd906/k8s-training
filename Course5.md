Kubernetesでサービスを作成しよう
===================
# Kubernetesを仕上げよう
[CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
```
k apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
k get pods --watch
k logs {pod名}
```

# Helmを使ってみよう
HelmはKubernetesのパッケージマネージャーです。
Helmを使うと、Kubernetesのアプリケーションを簡単にインストールできます。

[Helm](https://helm.sh/)

# APIをクラスタにデプロイしてみよう
```
k apply -f https://raw.githubusercontent.com/openfaas/faas-netes/master/namespaces.yml
helm repo add openfaas https://openfaas.github.io/faas-netes/
helm repo update
helm upgrade openfaas --install openfaas/openfaas --namespace openfaas --set functionNamespace=openfaas-fn --set generateBasicAuth=true
k apply -f course5/api/ingress.alb.yaml
```
openfaasの管理画面URLを取得する
```
k -n openfaas get ingress
```
```
export __API_HOST__={ingressのdomain}
```

openfaasのBasic認証パスワードを取得する
```
echo $(kubectl -n openfaas get secret basic-auth -o jsonpath="{.data.basic-auth-password}" | base64 --decode)
```
openfaasのパスワード: {Basic認証パスワード}

APIの管理画面に入って、APIをデプロイしてみよう。
* サイドバーの「Deploy New Function」をクリック
* [Have I Been Pwned]のFunctionを選択し、右下の[Deploy]をクリック
* [Alpine]のFunctionを選択し、右下の[Deploy]をクリック
* デプロイされたFunctionのstatusが「Ready」になるまで待つ

curlでデプロイしたAPIを叩いてみる
```
curl http://$__API_HOST__/function/haveibeenpwned \
    --data-raw 'chen@kdl.co.jp'
```

# APIに認証をかけよう
## 認証サーバーをデプロイする
認証サーバーの管理画面にアクセスするために、デフォルトの管理者情報をk8sのシークレットに保存する
```
k create ns keycloak-system
k -n keycloak-system create secret generic api-auth --from-env-file=./course5/auth/env
```
認証サーバーをデプロイし、サービスを公開する
```
k create -f course5/auth/keycloak.yaml
k -n keycloak-system rollout status deploy/keycloak
k -n keycloak-system get svc
```
```
export __KEYCLOAK_HOST__={keycloak svcのexternalIP}
```


認証サーバーにて、下記手順で認証用tokenを取得する
* 左上のプルダウンメニューから、[Create realm]を選択、handsonという名前でRealmを作成する。
* Realmを作成次第、左のサイドバーから[Clients]を選択し、[Create client]を選択する。
* client idには、handson-clientと入力し、[Next]を選択する。
* [Client authentication]と[Authorization]をonにチェックする。
* Authentication flowの[Standard flow]と[Direct Access Grants]のチェックを外す。
* [Save]を選択する。
* タブの[Credentials]を選択し、[Client secret]を見つけて、コピーする。

```
export __CLIENT_ID__={client id}
export __CLIENT_SECRET__={client secret}
```

アクセスtokenを取得してみる。
```
curl --request POST \
    --url http://$__KEYCLOAK_HOST__/realms/handson/protocol/openid-connect/token \
    --data client_id=$__CLIENT_ID__ \
    --data client_secret=$__CLIENT_SECRET__ \
    --data grant_type=client_credentials
```
tokenをとれたら、認証サーバーの正常に設定できたことが確認できる。

そして、下記手順で認証サーバーの公開鍵をダウンロードして、API側に設定する。
* サイドバーの[Realm Settings]を選択する。
* タブの[Keys]を選択する。
* [RSA256]の[Public key]を選択し、コピーする。

コピーした公開鍵を、下記ファイルに貼り付ける
```
vim course5/auth/keycloak.pub
```

## APIと認証サーバーをつなげる

haproxyを使って、APIと認証サーバーをつなげる
```
helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update
helm install kubernetes-ingress haproxytech/kubernetes-ingress \
    --create-namespace \
    --namespace haproxy-controller \
    --set controller.service.type=LoadBalancer
```
apiのingressを更新部分は下記となりますが、今回は変更後のファイルをそのまま適用するようにします。
```yaml
  annotations:
-    alb.ingress.kubernetes.io/scheme: internet-facing
-    alb.ingress.kubernetes.io/target-type: ip
+    ingress.class: haproxy
  labels:
    app: openfaas
spec:
-  ingressClassName: alb
+  ingressClassName: haproxy
  rules:
```
修正後のingressを適用し、エンドポイントが変わるのでもう一回取得する
```
k delete -f course5/api/ingress.alb.yaml 
k create -f course5/api/ingress.haproxy.yaml
k -n openfaas get ingress
```
新しいエンドポイントを使って、APIを叩いてみる
```
export __API_HOST__={新しいdomain}
```

```
curl http://$__API_HOST__/function/haveibeenpwned \
  --data-raw 'chen@kdl.co.jp'
```

新しいAPIのエンドポイントを生成すれば、それを認証サーバーに登録する、認証サーバー管理画面に戻る。
* 左のサイドバーから[Clients]を選択し、先ほど作成したClientを選択する。
* タブの[Client scopes]を選択し、[*******-dedicated]を選択する。
* タブの[Mappers]を選抲し、[Add mapper]をクリックし、[By configuration]クリックして[Audience]を選択。
* [name]には適当に設定し、[Included Custom Audience]には、APIのURLを設定する。
  > http://__API_HOST__/function
* [Save]を選択する。

認証サーバーに認証をかけるようにhaproxyに設定を追加する
まずは、ダウンロードした認証サーバーの公開鍵をhaproxyに登録する。
configmapを作成し、ファイルとしてhaproxyにマウントさせる。
```
k -n haproxy-controller create configmap keycloak-pub --from-file=keycloak=course5/auth/keycloak.pub
```

実行中のhaproxyのdeploymentの定義ファイルを書き出して、下記２点修正する
* haproxyのoauthモジュールを有効化したイメージに変更
* configmapで作成した公開鍵をマウントさせる

トレニーングの時間上の都合で、修正後のファイルを用意したので、それを使ってhaproxyのdeploymentを更新する。
```yaml
    spec:
+      volumes:
+      - name: keycloak-pubkey-file
+        configMap:
+          name: keycloak-pub
      containers:
        ...
+        volumeMounts:
+        - name: keycloak-pubkey-file
+          mountPath: /etc/haproxy/pem/
        ...
-        image: haproxytech/kubernetes-ingress:1.8.6
+        image: gpgkd906/kubernetes-ingress-oauth:latest
```
```
k delete -f course5/haproxy/kubernetes-ingress.yaml
k create -f course5/haproxy/kubernetes-ingress.yaml
```

そして、haproxyのoauthの設定を追加する
haproxy-kubernetes-ingressの設定はconfigmap/kubernetes-ingressで管理されているので、
それを書き出して、変更し反映させる。
こちらもトレニーングのため、修正後のファイルを用意しておくので、それを更新して、haproxyのconfigmapを更新する。
__KEYCLOAK_HOST__と__API_HOST__と更新してください。

```yaml
...
  namespace: haproxy-controller
+data:
+  global-config-snippet: |
+    lua-load /usr/local/share/lua/5.3/jwtverify.lua
+    setenv OAUTH_ISSUER http://__KEYCLOAK_HOST__/realms/handson
+    setenv OAUTH_AUDIENCE http://__API_HOST__/function
+    setenv OAUTH_PUBKEY_PATH /etc/haproxy/pem/keycloak
+  frontend-config-snippet: |
+    http-request allow if { path_beg /ui/ }
+    http-request allow if { path_beg /system/ }
+    http-request deny deny_status 401 unless { req.hdr(authorization) -m found }
+    http-request lua.jwtverify
+    http-request deny deny_status 403 unless { var(txn.authorized) -m bool }
```
```
k apply -f course5/haproxy/configmap.yaml
k -n haproxy-controller rollout restart deploy/kubernetes-ingress
k -n haproxy-controller rollout status deploy/kubernetes-ingress
```

APIを叩いて、認証がかかっていることを確認する
```
curl http://$__API_HOST__/function/haveibeenpwned \
  --data-raw 'chen@kdl.co.jp'
```
もう一度、アクセスtokenを取得して、ヘッダーにつけてAPIを叩く。
```
curl --request POST \
    --url http://$__KEYCLOAK_HOST__/realms/handson/protocol/openid-connect/token \
    --data client_id=$__CLIENT_ID__ \
    --data client_secret=$__CLIENT_SECRET__ \
    --data grant_type=client_credentials

curl http://$__API_HOST__/function/haveibeenpwned \
  -H 'authorization: Bearer __TOKEN__' \
  --data-raw 'chen@kdl.co.jp'
```

# APIをマネタイズしてみよう
認証サーバーの管理画面に戻って、課金Tierを作成する。
* 左上のプルダウンメニューから、[Handson]を選択
* サイドメニューから、[Client scopes]を選択
* [Create client scope]をクリックして、Nameに[free]を入力して、[Save]をクリック
* サイドメニューから、[Clients]を選択
* [Create client]をクリックして、CLient IDに[free-client]を入力して、[Next]を選択する。
* [Client authentication]と[Authorization]をonにチェックする。
* Authentication flowの[Standard flow]と[Direct Access Grants]のチェックを外して、[Save]を選択する。
* タブの[Cleint Scopes]を選択して、[Assigned client scope]をクリックして、すべてのscopeを選択して、[free-client-dedicated]をクリックして除外する。
* [Change type to]の横のメニューから[Remove]を選択し、[free-client-dedicated]以外のscopeを全部削除する。
* [Add client scope]をクリックして、先ほど作成された[free]を選択して、[Add]>[Default]を選択する。
* [free-client-dedicated]をクリックして、* タブの[Mappers]を選抲し、[Add mapper]をクリックし、[By configuration]クリックして[Audience]を選択。
* [name]には適当に設定し、[Included Custom Audience]には、APIのURLを設定する。
* [Save]を選択する。
* 同じように、client scopeに[pro]およびclientに[pro-client]を作成する。

```
export __FREE_CLIENT_ID__={free client id}
export __FREE_CLIENT_SECRET__={free client secret}
export __PRO_CLIENT_ID__={pro client id}
export __PRO_CLIENT_SECRET__={pro client secret}
```

そして、API側でfree-clientおよびpro-clientのrate limitを設定する。
haproxyのconfigmapを変更し、設定を追加する
こちらもトレニーングのため、修正後のファイルを用意しておくので、それを更新して、haproxyのconfigmapを更新する。
__KEYCLOAK_HOST__と__API_HOST__と更新してください。

vim course5/haproxy/configmap.plan.yaml
```yaml
...
  namespace: haproxy-controller
data:
    ...
  frontend-config-snippet: |
    ...
+    stick-table  type ipv6  size 100k  expire 24h  store http_req_cnt
+    http-request track-sc0 src

+    http-request deny deny_status 429 if { var(txn.oauth.scope) -m sub free } { sc_http_req_cnt(0) gt 3 }
+    http-request deny deny_status 429 if { var(txn.oauth.scope) -m sub pro } { sc_http_req_cnt(0) gt 5 }
```
設定を変更次第、反映させる。
```
k apply -f course5/haproxy/configmap.plan.yaml
k -n haproxy-controller rollout restart deploy/kubernetes-ingress
k -n haproxy-controller get pod --watch
```

freeのアクセスtokenを取得して、ヘッダーにつけてAPIを叩く。
```
curl --request POST \
    --url http://$__KEYCLOAK_HOST__/realms/handson/protocol/openid-connect/token \
    --data client_id=$__FREE_CLIENT_ID__ \
    --data client_secret=$__FREE_CLIENT_SECRET__ \
    --data grant_type=client_credentials

curl http://$__API_HOST__/function/haveibeenpwned \
  -H 'authorization: Bearer __TOKEN__' \
  --data-raw 'chen@kdl.co.jp'
```

proのアクセスtokenを取得して、ヘッダーにつけてAPIを叩く。
```
curl --request POST \
    --url http://$__KEYCLOAK_HOST__/realms/handson/protocol/openid-connect/token \
    --data client_id=$__PRO_CLIENT_ID__ \
    --data client_secret=$__PRO_CLIENT_SECRET__ \
    --data grant_type=client_credentials

curl http://$__API_HOST__/function/haveibeenpwned \
  -H 'authorization: Bearer __TOKEN__' \
  --data-raw 'chen@kdl.co.jp'
```
