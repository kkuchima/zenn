---
title: "外部 Multi-cluster Gateway で複数クラスタ間のトラフィックを制御する"
emoji: "🐢"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [GCP, GoogleCloud, GKE]
published: false
---

Multi-cluster Gateway でどういうことができるか色々試してみたので備忘も兼ねて記事にします。
Gateway とは？や Multi-cluster Gateway とは？という話はしないので、以下の解説記事もしくは公式ドキュメントをご参照ください。

# tl;dr
* 

Single IP で hostname を追加していく形にする
→ hostanme or 

1. 通常の動作（地理ベース）
2. Header Based Routing
3. Traffic Mirroring
4. Weighted Routing

## 0. 事前準備
### 環境変数の設定
今回は `gke01` というクラスタ上のワークロードを `gke02` というまっさらなクラスタ上に移行するというのを試してみたいと思います。どちらも東京(`asia-northeast1`) 上に構築しています。  
まず環境構築に必要となる環境変数を設定します。
```bash
export PROJECT_ID=<PROJECT ID>
export REGION1=asia-northeast1
export REGION2=asia-northeast2
export CLUSTER_NAME1=gke-tokyo
export CLUSTER_NAME2=gke-osaka
```

Multi-cluster Gateway に必要な API の有効化
```bash
gcloud services enable \
    container.googleapis.com \
    gkehub.googleapis.com \
    multiclusterservicediscovery.googleapis.com \
    multiclusteringress.googleapis.com \
    trafficdirector.googleapis.com \
    --project=${PROJECT_ID}
```

### GKE クラスタの作成
次に移行元と移行先の GKE クラスタを作成します。Backup for GKE は、Workload Identity を有効にした上で `--addons=BackupRestore` を指定することで利用することができます。移行先のクラスタでも Backup for GKE を有効にする必要があるので同じ構成で移行先のクラスタも作成します。
（Todo）クラスタはマイナー指定可能？
```bash
gcloud config set project ${PROJECT_ID}
gcloud services enable gkebackup.googleapis.com

gcloud beta container clusters create ${CLUSTER_NAME1} \
    --project=${PROJECT_ID}  \
    --region=${REGION} \
    --cluster-version=1.22.12-gke.300 \
    --num-nodes 1 \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --addons=BackupRestore

gcloud beta container clusters create ${CLUSTER_NAME2} \
    --project=${PROJECT_ID}  \
    --region=${REGION} \
    --cluster-version=1.22.12-gke.300 \
    --num-nodes 1 \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --addons=BackupRestore
```

### サンプルアプリケーションのデプロイ
挙動を確認するためのサンプルアプリケーションをデプロイします。  
永続データを PV 上に保管するよう修正した Guestbook アプリケーションを移行元のクラスタ `gke01` 上にデプロイします。
```bash
$ kubectx gke01
Switched to context "gke01".

$ kubectl get pods
NAME                                             READY   STATUS    RESTARTS   AGE
guestbook-frontend-deployment-85595f5bf9-ck7q5   1/1     Running   0          8h
redis-follower-deployment-84cd76b975-t66tt       1/1     Running   0          8h
redis-leader-deployment-86c48bb945-t5hxf         1/1     Running   0          8h

$ kubectl get pv  
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS   REASON   AGE
pvc-3944cbd4-6441-401f-bd12-9fa8400ddb80   10Gi       RWO            Delete           Bound    gkebackup/gkebackup-agent-vol-gkebackup-agent-0   standard                11h
pvc-679f60fc-fd95-4a94-9cba-aeeaf5485090   5Gi        RWO            Delete           Bound    default/redis-follower-pvc                        standard                8h
pvc-6f8702b0-c164-48d8-8de6-b23c8ba1c111   5Gi        RWO            Delete           Bound    default/redis-leader-pvc                          standard                8h

$ kubectl get svc
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
guestbook-frontend   LoadBalancer   10.64.5.175    34.85.xx.xx   80:30005/TCP   8h
kubernetes           ClusterIP      10.64.0.1      <none>         443/TCP        11h
redis-follower       ClusterIP      10.64.9.21     <none>         6379/TCP       8h
redis-leader         ClusterIP      10.64.12.147   <none>         6379/TCP       7h41m
```

リストアしたときに分かりやすくするため `BfG Test` という文字列を残しておきます。   
![Guestbook (gke01)](/images/bfg-cluster-migration/gke01-guestbook.png)  
ここまでで事前準備は完了です。

## 1. 新規作成した同一リージョンの GKE クラスタにワークロードを移行する
まずは同一リージョン内での移行を試してみます。
### バックアップの取得
最初に `Backup Plan` というリソースでバックアップの対象クラスタや取得範囲（特定 Namespace or クラスタ全体、Secret や PV を含むかなど）、バックアップファイルを保管するリージョン、バックアップの保管期間等を設定します。`Backup Plan` はこれから作成する `Backup` リソースの親リソースです。
```bash
$ gcloud beta container backup-restore backup-plans create ${BACKUP_PLAN} \
    --location=${REGION} \
    --cluster=projects/${PROJECT_ID}/locations/${REGION}/clusters/${CLUSTER_NAME1} \
    --all-namespaces \
    --include-secrets \
    --include-volume-data \
    --backup-retain-days=3
Create request issued for: [bp-gke01]
Waiting for operation [projects/kkuchima-sandbox/locations/asia-northeast1/operations/operation-1662395287905-5e7f0909ca40f-ba588535-e441d240]
 to complete...done.
```
* `--location` ではバックアップしたアーティファクト（構成アーカイブやボリュームスナップショット）を保管するリージョンを指定します。（サポートされているリージョンは `gcloud beta container backup-restore locations list` で確認可能です）  
* `--cluster` でバックアップ取得対象のクラスタを指定します。対象クラスタをパス形式で指定する必要があるという点にご注意ください。  
* `--all-namespaces` というのは全ての Namespace を対象にバックアップをするという設定です。特定 Namespace のみバックアップを取りたい場合は `--selected-applications` で対象を指定することもできます。また、`--selected-applications` で特定アプリケーションのみに絞ることも可能です。
* `--include-secrets` や `--include-volume-data` はバックアップ対象対象リソースの定義です。Secret や PV をバックアップ対象に含めるかどうかを定義しています。今回はどちらも含めるのでオプションで設定しています。
* `--backup-retain-days` で取得したバックアップを何日間保管するかを指定します。ここで定義した期間を過ぎるとバックアップは自動的に削除されます。その他 `--backup-delete-lock-days` を指定することで特定の期間はバックアップが削除されない（手動・自動どちらの場合でも）ように設定することができます。

定期的にバックアップジョブを実行する場合は `--cron-schedule` で実行スケジュールを設定することもできます。その他、利用可能なオプションについては以下をご参照ください：  
https://cloud.google.com/sdk/gcloud/reference/beta/container/backup-restore/backup-plans/create

続いて、作成した `Backup Plan` を指定してバックアップを取得します。バックアップを取得する際の設定は `Backup` リソースで定義します。  
`Backup Plan` と比べて設定項目が少なく、必要なのはどのリージョンにどの `Backup Plan` を使ってバックアップを取るかという設定くらいです。  
ちなみに `ProtectedApplication` というリソースを定義することにより、バックアップを取得する際の PreHook や PostHook をアプリケーション毎に定義することもでき、アプリケーションレベルで静止点をとって安全にバックアップすることも可能です。  
https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/how-to/protected-application

```bash
$ gcloud beta container backup-restore backups create backup-${CLUSTER_NAME1} \
    --location=${REGION} \
    --backup-plan=${BACKUP_PLAN}
Create in progress for backup backup-gke01 [projects/kkuchima-sandbox/locations/asia-northeast1/operations/operation-1662395553893-5e7f0a0774d84-c06193f0-184f17c4].
Creating backup backup-gke01...done.
```
`gcloud beta container backup-restore backups list` を実行するとバックアップの状態を確認することができます。今回はちゃんとバックアップが取得できました。
```bash
$ gcloud beta container backup-restore backups list \
    --location=${REGION} \
    --backup-plan=${BACKUP_PLAN}
NAME: backup-gke01
LOCATION: asia-northeast1
BACKUP_PLAN: bp-gke01
CREATE_TIME: 2022-09-05T16:32:33 UTC
COMPLETE_TIME: 2022-09-05T16:34:13 UTC
STATE: SUCCEEDED
```

Google Cloud Console からだとこのような感じで確認できます。  
![Backup](/images/bfg-cluster-migration/bk-gke01.png)

### 別クラスタ上でのリストア
`gke01` で取得したバックアップを `gke02` 上でリストアしてワークロードを移行します。  
バックアップと同じく、まずは `Restore Plan` を作成します。`Restore Plan` は `Restore` というリソースの親リソースで、リストア先のクラスタや利用するバックアップ、リストア方法（リソース競合時の挙動）等を設定します。  
```bash
$ gcloud beta container backup-restore restore-plans create ${RESTORE_PLAN} \
    --location=${REGION} \
    --backup-plan=projects/${PROJECT_ID}/locations/${REGION}/backupPlans/${BACKUP_PLAN} \
    --cluster=projects/${PROJECT_ID}/locations/${REGION}/clusters/${CLUSTER_NAME2} \
    --namespaced-resource-restore-mode=delete-and-restore \
    --all-namespaces \
    --cluster-resource-restore-scope=storage.k8s.io/StorageClass,apiextensions.k8s.io/CustomResourceDefinition \
    --cluster-resource-conflict-policy=use-backup-version \
    --volume-data-restore-policy=restore-volume-data-from-backup
Create request issued for: [rp-gke02]
Waiting for operation [projects/kkuchima-sandbox/locations/asia-northeast1/operations/operation-1662396041266-5e7f0bd8405f3-081ed352-0c8dc287]
 to complete...done.                                                                                                                          
Created restore plan [rp-gke02].
```
`--backup-plan` は先ほど `gke01` 用に作成した Backup Plan を指定し、ターゲットクラスタ (`--cluster`) を `gke02` として設定します。  

ここでポイントとなるのは、`--cluster-resource-restore-scope` というオプションを指定することです。2022.09 現在、gcloud コマンドではデフォルトだとこのオプションが指定されていないため、CRD や StorageClass といったクラスタスコープリソースがリストアされないようになっています（Console 上からの操作の場合は当該オプションがデフォルトで設定されています）。  
また、`--volume-data-restore-policy=restore-volume-data-from-backup` というオプションを指定することでバックアップとして取得した Volume 上のデータを使って PV をリストアすることができるようになります。この指定が無いとまっさらな PV が新規作成/マウントされてしまいます。

その他オプションについてはこちらをご参照ください：  
https://cloud.google.com/sdk/gcloud/reference/beta/container/backup-restore/restore-plans/create   

Restore Plan が作成できたら実際にリストアを実行します。リストア処理自体はすぐ完了しますが、リストア処理が完了するのと実際に全てのワークロードがデプロイされるまではタイムラグがありますので少々待ちます。
```bash
$ gcloud beta container backup-restore restores create restore-${CLUSTER_NAME2} \
    --location=${REGION} \
    --restore-plan=${RESTORE_PLAN} \
    --backup=projects/${PROJECT_ID}/locations/${REGION}/backupPlans/${BACKUP_PLAN}/backups/backup-${CLUSTER_NAME1}
Create in progress for restore restore-gke02 [projects/kkuchima-sandbox/locations/asia-northeast1/operations/operation-1662396089305-5e7f0c061085e-fc9b676c-8d64babe].
Creating restore restore-gke02...done.

$ gcloud beta container backup-restore restores list \
    --location=${REGION} \
    --restore-plan=${RESTORE_PLAN}
NAME: restore-gke02
LOCATION: asia-northeast1
RESTORE_PLAN: rp-gke02
BACKUP: backup-gke01
CREATE_TIME: 2022-09-05T16:41:29 UTC
COMPLETE_TIME: 2022-09-05T16:41:36 UTC
STATE: SUCCEEDED
```

`gke02` 上に Guestbook アプリケーションがデプロイされていることを確認します。PV もプロビジョニングされています。
```bash
$ kubectx gke02
Switched to context "gke02".

$ kubectl get pods
NAME                                             READY   STATUS    RESTARTS   AGE
guestbook-frontend-deployment-85595f5bf9-spj4g   1/1     Running   0          14m
redis-follower-deployment-84cd76b975-z7np8       1/1     Running   0          14m
redis-leader-deployment-86c48bb945-5scrn         1/1     Running   0          14m

$ kubectl get pv  
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS   REASON   AGE
pvc-7ed39318-fff0-4f5b-9b78-b6934170b771   10Gi       RWO            Delete           Bound    gkebackup/gkebackup-agent-vol-gkebackup-agent-0   standard                10h
pvc-c480af9581fdff76                       5Gi        RWO            Delete           Bound    default/redis-leader-pvc                          standard                13m
pvc-ff7f0f2e35329447                       5Gi        RWO            Delete           Bound    default/redis-follower-pvc                        standard                13m

$ kubectl get svc
NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
guestbook-frontend   LoadBalancer   10.92.14.54   34.84.xx.xx   80:30902/TCP   14m
kubernetes           ClusterIP      10.92.0.1     <none>          443/TCP        10h
redis-follower       ClusterIP      10.92.4.112   <none>          6379/TCP       14m
redis-leader         ClusterIP      10.92.7.39    <none>          6379/TCP       14m
```

実際にアプリケーションにアクセスしてみると、バックアップ前に入力していた内容が反映されていることが分かります。  
これで同一リージョン内でのクラスタ移行は完了です。  
![Guestbook (gke02)](/images/bfg-cluster-migration/gke02-guestbook.png)   

ちなみに今回は同じバージョンの GKE クラスタ上にリストアしましたが、バックアップを取得したクラスタより上のバージョンのクラスタをターゲットにすることも可能です(+3 バージョンまでサポート)。クラスタレベルでの Blue/Green アップグレードの場合に活用できそうです。
```
復元には、最大 GKE バージョン差分制約が適用されます。具体的には、ターゲット クラスタのバージョンは、[バックアップ クラスタ バージョン、バックアップ クラスタ バージョン + 3 つのマイナー バージョン] の範囲内である必要があります。たとえば、GKE 1.22 を実行しているクラスタからバックアップが生成された場合、1.22 ～ 1.25 を実行しているクラスタにのみインストールできます。
```
https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/how-to/restore-plan

## 2. 新規作成した別リージョンの GKE クラスタにワークロードを移行する

続いて、取得したバックアップを使って別リージョンの GKE クラスタへ移行を試してみます。（基本的にはこれまでと同じ手順です）  
まず Backup for GKE リソースとリージョンの関係性についてですが、Backup Plan では生成されたバックアップ アーティファクト (Volume snapshot 含む) を保存するリージョンを定義します。  
またドキュメントに `BackupPlan を介してバックアップされるクラスタが BackupPlan とは異なるリージョンにある場合、これらのバックアップを実行するときにリージョン間のデータ転送料金が発生する可能性があります` と記載があるように必ずしも Backup Plan とバックアップ対象クラスタが同一リージョンにある必要は無いようです。  
Restore Plan ではバックアップターゲットが存在するリージョンを定義します。公式ドキュメントにも記載がある通り、Restore Plan で指定するリージョンとリストア先クラスタのリージョンは合わせる必要があります。  
`RestorePlan と同じリージョン内のゾーンクラスタまたはリージョン クラスタである必要があります` 

ちなみに本題の別リージョン上のクラスタでのリストアについては、可能ではあるもののリージョンを跨いだ転送が発生するので料金面での考慮が必要になりますのでご注意ください。  
`RestorePlan リージョンが BackupPlan リージョンと異なる場合、復元時にリージョン間のデータ転送料金が発生する可能性があります`
https://cloud.google.com/kubernetes-engine/docs/add-on/backup-for-gke/concepts/about-locations

### 別リージョン (asia-northeast2) に新規クラスタ (gke03) を構築
それでは実際に試していきます。まずは各種環境変数を設定し、`asia-northeast2` に新しい GKE クラスタ (`gke03`) を構築します。クラスタの構成は `gke01`, `gke02` と同じです。
```bash
export REGION2=asia-northeast2
export CLUSTER_NAME3=gke03
export RESTORE_PLAN=rp-gke03

gcloud beta container clusters create ${CLUSTER_NAME3} \
    --project=${PROJECT_ID}  \
    --region=${REGION2} \
    --cluster-version=1.22.12-gke.300 \
    --num-nodes 1 \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --addons=BackupRestore
```

### 別リージョンクラスタ上でのリストア
バックアップは元々取得していた `gke01` のバックアップを使いまわすので割愛し、リストアを実行します。  
まずは `gke03` 用の Restore Plan を作成します。先述の通りターゲットクラスタと Restore Plan のリージョンは合わせる必要があるので、`--location` 設定に気をつけてください（違うリージョンを指定した場合はエラーになります）  
今回は Backup Plan は `asia-northeast1` のまま、Restore Plan とターゲットクラスタを `asia-northeast2` に設定してリストアを試しています。
```bash
$ gcloud beta container backup-restore restore-plans create ${RESTORE_PLAN} \
    --location=${REGION2} \
    --backup-plan=projects/${PROJECT_ID}/locations/${REGION}/backupPlans/${BACKUP_PLAN} \
    --cluster=projects/${PROJECT_ID}/locations/${REGION2}/clusters/${CLUSTER_NAME3} \
    --namespaced-resource-restore-mode=delete-and-restore \
    --all-namespaces \
    --cluster-resource-restore-scope=storage.k8s.io/StorageClass,apiextensions.k8s.io/CustomResourceDefinition \
    --cluster-resource-conflict-policy=use-backup-version \
    --volume-data-restore-policy=restore-volume-data-from-backup
Create request issued for: [rp-gke03]
Waiting for operation [projects/kkuchima-sandbox/locations/asia-northeast2/operations/operation-1662434576363-5e7f9b662f0af-b1613784-1b63a02d]
 to complete...done.                                                                                                                          
Created restore plan [rp-gke03].
```

Restore Plan が作成できたら Restore リソースを作成し、リージョン間のマイグレーションを実行します。
```bash
$ gcloud beta container backup-restore restores create restore-${CLUSTER_NAME3} \
    --location=${REGION2} \
    --restore-plan=${RESTORE_PLAN} \
    --backup=projects/${PROJECT_ID}/locations/${REGION}/backupPlans/${BACKUP_PLAN}/backups/backup-${CLUSTER_NAME1}
Create in progress for restore restore-gke03 [projects/kkuchima-sandbox/locations/asia-northeast2/operations/operation-1662443484528-5e7fbc95abef5-7a770035-067a9c44].
Creating restore restore-gke03...done.

$ gcloud beta container backup-restore restores list \
    --location=${REGION2} \
    --restore-plan=${RESTORE_PLAN}
NAME: restore-gke03
LOCATION: asia-northeast2
RESTORE_PLAN: rp-gke03
BACKUP: backup-gke01
CREATE_TIME: 2022-09-06T03:24:00 UTC
COMPLETE_TIME: 2022-09-06T04:31:56 UTC
```

Restore が成功したら、`gke03` のデプロイ状況を確認します。
```bash
$ kubectx gke03
Switched to context "gke03".

$ kubectl get pods 
NAME                                             READY   STATUS    RESTARTS   AGE
guestbook-frontend-deployment-85595f5bf9-d4kcr   1/1     Running   0          72m
redis-follower-deployment-84cd76b975-g8vjz       1/1     Running   0          72m
redis-leader-deployment-86c48bb945-srptd         1/1     Running   0          72m

$ kubectl get pv  
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                             STORAGECLASS   REASON   AGE
pvc-13b705493114c085                       5Gi        RWO            Delete           Bound    default/redis-follower-pvc                        standard                72m
pvc-adca4d29cc1b9294                       5Gi        RWO            Delete           Bound    default/redis-leader-pvc                          standard                72m
pvc-b02cbacd-15c4-443d-a785-e9ffeda01f08   10Gi       RWO            Delete           Bound    gkebackup/gkebackup-agent-vol-gkebackup-agent-0   standard                174m

$ kubectl get svc
NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
guestbook-frontend   LoadBalancer   10.16.2.34    34.97.xx.xx   80:31222/TCP   73m
kubernetes           ClusterIP      10.16.0.1     <none>        443/TCP        176m
redis-follower       ClusterIP      10.16.0.140   <none>        6379/TCP       73m
redis-leader         ClusterIP      10.16.3.154   <none>        6379/TCP       73m
```

実際にアプリケーションにアクセスしてみると、`gke01` で取得したバックアップの内容が反映されていることが分かります。  
これで別リージョンのクラスタへのワークロード移行をすることができました。  
![Guestbook (gke03)](/images/bfg-cluster-migration/gke03-guestbook.png)  

# 最後に
Backup for GKE を利用することで、ステートフルなワークロードもクラスタ間で移行することが可能です。これまで OSS や 3rd Party 製品で実現できていた機能がマネージドサービスとしても利用できるようになりました。  
本記事ではあまり触れませんでしたが、Backup for GKE はクラスタ移行のユースケース以外でも、アプリケーションレベルの障害からの復旧やバックアップボリュームを基に Pod を複製する等ほかにも多くの機能・ユースケースが存在します。  
まだ Preview の機能ではありますが、ステートフルなアプリケーションや GKE クラスタをより安全に運用しやすくするための手段として活用していただけると思います。気になった方はぜひ試してみてください。