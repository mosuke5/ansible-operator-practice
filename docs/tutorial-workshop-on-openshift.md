## チュートリアル実施
まずはOperator SDKのチュートリアルを行い、基本的な操作感を体験します。
チュートリアルは下記URLから実施できますが、OpenShift環境にあわせたチュートリアルへ変更するため（OpenShiftのレジストリを利用するなど）、本ドキュメントを用意しました。  
https://v0-19-x.sdk.operatorframework.io/docs/ansible/quickstart/

### バージョン確認
```
$ ansible --version
ansible 2.10.3
  config file = None
  configured module search path = ['/home/user10/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.6/site-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 3.6.8 (default, Dec  5 2019, 15:45:45) [GCC 8.3.1 20191121 (Red Hat 8.3.1-5)]
  
$ operator-sdk version
operator-sdk version: "v0.19.4", commit: "125d0dfcc71fef4f9d7e2a42b1354cb79ffdee03", kubernetes version: "v1.18.2", go version: "go1.13.15 linux/amd64"
```

### プロジェクトの作成
実行するOpenShiftクラスタは参加者で共有します。
API定義名が重複しないように、`--api-version`にはユーザ名を必ずつけてください。

```
$ oc login xxxxxxx
$ oc new-project userXX-operator
$ operator-sdk new memcached-operator --api-version=user10.example.com/v1alpha1 --kind=Memcached --type=ansible
$ cd memcached-operator
```

### ディレクトリ構成
ディレクトリ構成を確認していきます。

```
$ tree
.
|-- build
|   `-- Dockerfile
|-- deploy
|   |-- crds
|   |   |-- cache.example.com_memcacheds_crd.yaml
|   |   `-- cache.example.com_v1alpha1_memcached_cr.yaml
|   |-- operator.yaml
|   |-- role.yaml
|   |-- role_binding.yaml
|   `-- service_account.yaml
|-- molecule
|   |-- cluster
|   |   |-- converge.yml
|   |   |-- create.yml
|   |   |-- destroy.yml
|   |   |-- molecule.yml
|   |   |-- prepare.yml
|   |   `-- verify.yml
|   |-- default
|   |   |-- converge.yml
|   |   |-- molecule.yml
|   |   |-- prepare.yml
|   |   `-- verify.yml
|   |-- templates
|   |   `-- operator.yaml.j2
|   `-- test-local
|       |-- converge.yml
|       |-- molecule.yml
|       |-- prepare.yml
|       `-- verify.yml
|-- requirements.yml
|-- roles
|   `-- memcached
|       |-- README.md
|       |-- defaults
|       |   `-- main.yml
|       |-- files
|       |-- handlers
|       |   `-- main.yml
|       |-- meta
|       |   `-- main.yml
|       |-- tasks
|       |   `-- main.yml
|       |-- templates
|       `-- vars
|           `-- main.yml
`-- watches.yaml
```

### ロジックの追記
以下のロジックを追記します。
Operatorのロジックを基本的に、playbook, roleに記述します。
APIと実行するロジックの紐付けは `watches.yaml` で管理されます。

```
$ cat watches.yaml
---
- version: v1alpha1
  group: user10.example.com
  kind: Memcached
  role: memcached
  
$ vim roles/memcached/tasks/main.yml
---
- name: start memcached
  community.kubernetes.k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ meta.name }}-memcached'
        namespace: '{{ meta.namespace }}'
      spec:
        replicas: "{{size}}"
        selector:
          matchLabels:
            app: memcached
        template:
          metadata:
            labels:
              app: memcached
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -m=64
              - -o
              - modern
              - -v
              image: "docker.io/memcached:1.4.36-alpine"
              ports:
                - containerPort: 11211
```

### Image build
Operatorのイメージをビルドします。
ドキュメントではquay.ioなどの外部のレジストリを利用していますが、OpenShift内部のレジストリを利用することとしてみます。
OpenShiftのレジストリもRouteを使って公開しています。

まずはレジストリにログインします。

```
$ export REGISTRY_URL=default-route-openshift-image-registry.apps.cluster-610d.610d.sandbox580.opentlc.com
$ docker login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false $REGISTRY_URL
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
WARNING! Using --password via the cli is insecure. Please consider using --password-stdin
Login Succeeded!
```

Operatorのイメージをビルドします。`operator-sdk`コマンドでビルドができます、v1.0以降はmakeを使って操作します。
Operator自身のコンテナイメージは`build/Dockerfile`を用いてビルドされます。

```
$ operator-sdk build $REGISTRY_URL/your-project/memcached-operator-tutorial:v0.0.1
INFO[0000] Building OCI image default-route-openshift-image-registry.apps.cluster-610d.610d.sandbox580.opentlc.com/user10-operator/memcached-operator-tutorial:v0.0.1
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
STEP 1: FROM quay.io/operator-framework/ansible-operator:v0.19.4
STEP 2: COPY requirements.yml ${HOME}/requirements.yml
--> Using cache c7fbd91d945dda3798ef20aa6fa61d0f5c8e9d85f19bfe3ba3c5e0433ccb3cb6
STEP 3: RUN ansible-galaxy collection install -r ${HOME}/requirements.yml  && chmod -R ug+rwx ${HOME}/.ansible
--> Using cache 12c258c7bdbf0fe201820f468079824c5e49283220d83c1c1bc409d87774d32b
STEP 4: COPY watches.yaml ${HOME}/watches.yaml
--> Using cache d33311fb8b75dffa8d66cd49032a133b2bf17e0a67de74f7fed828547ece76e1
STEP 5: COPY roles/ ${HOME}/roles/
--> Using cache 808fa50d5b59ef29eafacaabb24d0caf3f2ab684dc33b08dc1ef61b8bc084e66
STEP 6: COMMIT default-route-openshift-image-registry.apps.cluster-610d.610d.sandbox580.opentlc.com/user10-operator/memcached-operator-tutorial:v0.0.1
--> 808fa50d5b5
808fa50d5b59ef29eafacaabb24d0caf3f2ab684dc33b08dc1ef61b8bc084e66
INFO[0001] Operator build complete.

$ docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                                      TAG       IMAGE ID       CREATED         SIZE
default-route-openshift-image-registry.apps.cluster-610d.610d.sandbox580.opentlc.com/user10-operator/memcached-operator-tutorial   v0.0.1    808fa50d5b59   23 minutes ago   454 MB
quay.io/operator-framework/ansible-operator     v0.19.4   96c2b4ba1762   6 weeks ago     453 MB

$ docker push --tls-verify=false default-route-openshift-image-registry.apps.cluster-610d.610d.sandbox580.opentlc.com/user10-operator/memcached-operator-tutorial:v0.0.1
...
Writing manifest to image destination
Storing signatures

$ oc get is
NAME                          IMAGE REPOSITORY                                                                                                                    TAGS     UPDATED
memcached-operator-tutorial   default-route-openshift-image-registry.apps.cluster-610d.610d.sandbox580.opentlc.com/user10-operator/memcached-operator-tutorial   v0.0.1   2 minutes ago
```

完了後にOpenShiftのImageStreamを確認してみましょう。

### Operatorのデプロイ
Operatorが利用するイメージのパスは、内部パスを指定する。

```
// Operatorを起動するPodのイメージ名を変更
$ vim deploy/operator.yaml
...
          # Replace this with the built image name
          image: image-registry.openshift-image-registry.svc:5000/user10-operator/memcached-operator-tutorial:v0.0.1
...

// CRDのデプロイ
$ oc create -f deploy/crds/user10.example.com_memcacheds_crd.yaml
customresourcedefinition.apiextensions.k8s.io/memcacheds.user10.example.com created

// CRDはクラスタ共通。各参加者のCRDがユニークに作成されていることを確認
$ oc get crd | grep memcached
memcacheds.user10.example.com                               2020-11-03T08:18:55Z
memcacheds.user11.example.com                               2020-11-03T08:18:55Z
memcacheds.user12.example.com                               2020-11-03T08:18:55Z

// Operator自身をデプロイ
$ oc create -f deploy/service_account.yaml
$ oc create -f deploy/role.yaml
$ oc create -f deploy/role_binding.yaml
$ oc create -f deploy/operator.yaml

$ oc get deploy,pod
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/memcached-operator   1/1     1            1           2m44s

NAME                                      READY   STATUS    RESTARTS   AGE
pod/memcached-operator-679f4c7849-wqlhb   1/1     Running   1          18s
```

### 独自のCRのデプロイ
独自の設定のアプリケーションをデプロイします。

```
$ oc apply -f deploy/crds/user10.example.com_v1alpha1_memcached_cr.yaml
memcached.user10.example.com/example-memcached created

$ oc get memcached
NAME                AGE
example-memcached   101s

$ oc get memcached -o yaml
...定義を確認

$ oc get pod
NAME                                          READY   STATUS    RESTARTS   AGE
example-memcached-memcached-b885dcc75-b9zlc   1/1     Running   0          44s
example-memcached-memcached-b885dcc75-p6c42   1/1     Running   0          44s
example-memcached-memcached-b885dcc75-x75sj   1/1     Running   0          44s
memcached-operator-679f4c7849-wqlhb           1/1     Running   1          2m45s
```

### Operatorのログ
Operator Podのログを確認できます。
内部でAnsible Playbookが実行されている様子を確認しましょう。

```
$ oc logs -f memcached-operator-679f4c7849-wqlhb 
--------------------------- Ansible Task StdOut -------------------------------

TASK [Gathering Facts] *********************************************************

-------------------------------------------------------------------------------
{"level":"info","ts":1604393417.5752716,"logger":"logging_event_handler","msg":"[playbook task]","name":"example-memcached","namespace":"user10-operator","gvk":"user10.example.com/v1alpha1, Kind=Memcached","event_type":"playbook_on_task_start","job":"6334824724549167320","EventData.Name":"start memcached"}

--------------------------- Ansible Task StdOut -------------------------------

TASK [start memcached] *********************************************************
task path: /home/ec2-user/work/memcached-operator/roles/memcached/tasks/main.yml:2

-------------------------------------------------------------------------------

{"level":"info","ts":1604393421.08038,"logger":"proxy","msg":"Cache miss: apps/v1, Kind=Deployment, user10-operator/example-memcached-memcached"
....

--------------------------- Ansible Task Status Event StdOut  -----------------

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### CRの定義変更・削除
レプリカ数を変更してみる。

```
$ vim deploy/crds/user10.example.com_v1alpha1_memcached_cr.yaml
apiVersion: user10.example.com/v1alpha1
kind: Memcached
metadata:
  name: example-memcached
spec:
  # Add fields here
  size: 5

$ oc apply -f deploy/crds/user10.example.com_v1alpha1_memcached_cr.yaml

$ oc get pod
NAME                                          READY   STATUS    RESTARTS   AGE
example-memcached-memcached-b885dcc75-2ntxw   1/1     Running   0          23s
example-memcached-memcached-b885dcc75-9gchm   1/1     Running   0          23s
example-memcached-memcached-b885dcc75-g66n6   1/1     Running   0          75s
example-memcached-memcached-b885dcc75-k5fks   1/1     Running   0          75s
example-memcached-memcached-b885dcc75-vdx8m   1/1     Running   0          75s
memcached-operator-admin-7d8bc8889d-jkdls     1/1     Running   0          6m35s
```

CRを削除してみる。
```
$ oc delete -f deploy/crds/user10.example.com_v1alpha1_memcached_cr.yaml
memcached.admin.example.com "example-memcached" deleted

$ oc get pod
NAME                                          READY   STATUS        RESTARTS   AGE
example-memcached-memcached-b885dcc75-6wg6s   0/1     Terminating   0          3m58s
memcached-operator-admin-7d8bc8889d-jkdls     1/1     Running       0          4m35s
```

## ローカル実行
開発時にはOperatorを必ずしもPodとして起動しておく必要はありません。
ローカルの開発環境にOperatorを起動し、テストすることが可能です。
ターミナルを２つ起動し、片方で `operator-sdk run local` を動かしておき、もう片方でCRの操作をするとわかりやすいです。

```
//ローカル実行前にクラスタ上のOperator Podを削除しておきます
$ oc delete -f deploy/operator.yaml

// ローカルでOperatorの起動
$ operator-sdk run local
....
```

### ローカル実行環境で一通りの操作を確認
ローカル実行ができたら一通りの操作を行って、開発・検証の流れになれましょう。

- CRの作成
- CRの修正
- CRの削除
- Operatorに適当な処理の追加

## 独自APIの追加
下記にしたがって、独自のAPIを追加してみましょう。

### 実現したい要件
- `kind: MyDeployment` を追加
- CRDの設定
  - 上のチュートリアルではとくにCRDの定義を変更しませんでした。独自の定義ができるように修正する。
  - CRDには下記を設定できること
    - size: memcachedのレプリカ数。入力必須。
    - image: Podの起動に利用するイメージ名（デフォルト値の設定あり）
    - imageTag: Podの起動に利用するイメージのタグ（デフォルト値の設定あり）
  - CRD登録後に`oc explain` で確認すること
  - 参考URL
    - https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/
- ロジックの追加
  - CRで設定した、`size`, `image`, `imageTag` をマニフェスト似変数として反映すること
  - deployment, serviceともに作成すること

### 実現手順
一度、下記を見ずに自分で修正にトライしてみましょう。

```
// コマンドラインでapiの追加
$ operator-sdk add api --api-version=user10.example.com/v1alpha1 --kind=MyDeployment
INFO[0000] Generating api version user10.example.com/v1alpha1 for kind MyDeployment.
INFO[0000] Created deploy/crds/user10.example.com_v1alpha1_mydeployment_cr.yaml
INFO[0000] Created roles/mydeployment/README.md
INFO[0000] Created roles/mydeployment/meta/main.yml
INFO[0000] Created roles/mydeployment/files/.placeholder
INFO[0000] Created roles/mydeployment/templates/.placeholder
INFO[0000] Created roles/mydeployment/vars/main.yml
INFO[0000] Created roles/mydeployment/defaults/main.yml
INFO[0000] Created roles/mydeployment/tasks/main.yml
INFO[0000] Created roles/mydeployment/handlers/main.yml
INFO[0000] Generated CustomResourceDefinition manifests.
INFO[0000] API generation complete.

// 新しいAPIと実行するroleの紐付けを確認
$ cat watches.yaml
---
- version: v1alpha1
  group: user10.example.com
  kind: Memcached
  role: memcached

- version: v1alpha1
  group: user10.example.com
  kind: MyDeployment
  role: mydeployment

// CRDを修正
$ vim deploy/crds/user10.example.com_mydeployments_crd.yaml
...
          spec:
            description: MemcachedSpec defines the desired state of Memcached
            properties:
              size:
                description: Size is the size of the memcached deployment
                format: int32
                type: integer
              image:
                description: Image is the name of image
                type: string
                default: "docker.io/memcached"
              imagetag:
                description: Image Tag is the name of image tag
                type: string
                default: "latest"
            required:
            - size
            type: object

// ロジックの追加
$ vim roles/mydeployment/tasks/main.yml

$ vim molecule/default/verify.yml

// CRDをクラスタに追加
$ oc apply -f  deploy/crds/user10.example.com_mydeployments_crd.yaml
customresourcedefinition.apiextensions.k8s.io/mydeployments.user10.example.com created

// CRDの確認
$ oc get crd | grep example
memcacheds.user10.example.com                               2020-11-24T06:51:20Z
mydeployments.user10.example.com                            2020-11-24T07:38:13Z

// 登録したCRDの定義を explainコマンドで確認
$ oc explain mydeployments.spec
KIND:     MyDeployment
VERSION:  user10.example.com/v1alpha1

RESOURCE: spec <Object>

DESCRIPTION:
     MemcachedSpec defines the desired state of Memcached

FIELDS:
   image	<string>
     Image is the name of image

   imagetag	<string>
     Image Tag is the name of image tag

   size	<integer> -required-
     Size is the size of the memcached deployment

// サンプルのCRの定義の変更     
$ vim deploy/crds/user10.example.com_v1alpha1_mydeployment_cr.yaml
apiVersion: user10.example.com/v1alpha1
kind: MyDeployment
metadata:
  name: example-mydeployment
spec:
  size: 3
  image: memcached
  imagetag: 1.4.36-alpine

// CRのクラスタへのデプロイ
$ oc apply -f deploy/crds/user10.example.com_v1alpha1_mydeployment_cr.yaml
mydeployment.user10.example.com/example-mydeployment created

// 起動確認
$ oc get pod,deployment,service
NAME                                                  READY   STATUS    RESTARTS   AGE
pod/example-mydeployment-memcached-6b5f9796cd-rxtzv   1/1     Running   0          39s
pod/example-mydeployment-memcached-6b5f9796cd-s6qxm   1/1     Running   0          42s
pod/example-mydeployment-memcached-6b5f9796cd-wd8n7   1/1     Running   0          45s

NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example-mydeployment-memcached   3/3     3            3           2m44s

NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/example-mydeployment-memcached   ClusterIP   172.30.189.198   <none>        11211/TCP   2m39s
```

## 外部システムのステートを利用する
Kubernetes内部ではなく、外部システムのステートを用いて処理する場合、`watches.yaml`の`reconcilePeriod`のオプションを利用します。
定期的にansible playbookを実行することができるようになります。
例えば、[こちらのサンプルのJSON](https://raw.githubusercontent.com/mosuke5/ansible-operator-practice/master/config/testdata/sample.json)の結果から、ImageTagを差し替えてみます。

