# Amazon Redshift ハンズオン - Day2

# 1. 環境構築準備
## 1-1. ハンズオンで利用するアカウント
* AWSプロフェッショナルサービスにて用意した以下のアカウントをご利用下さい。

|no.|アカウントID|管理者|
|---:|:---|:---|
|1||AWS仲谷|
|2||AWS仲谷|
|3||AWS仲谷|
|4||AWS関口|
|5||AWS関口|
|6||AWS関口|

* IAMユーザー名、パスワード、アカウントIDは当日お知らせします。
* ハンズオンでは **東京リージョン** をご利用ください。

## 1-2. CloudFormationによるハンズオン環境の構築
* 今回は事前にVPCとRedshift、Aurora PostgreSQL環境を構築済みです。
* 今回使用するSQL等は、本マークダウン中から直接コピーして下さい。

## 1−3. スナップショットの復元
* 事前にクラスター作成、テーブル作成、データロードを完了済みのクラスターから取得したスナップショットを使って、各自のAWSアカウントにRedshiftクラスターを復元します。
  * スナップショットを選択
  * 以下のスナップショットを選択
```
スナップショット名　：　TBD
```
* 完了までしばらく待ちます。

# 2. 接続確認

## 2-1. データベースへの接続確認
### 2-2-1 接続設定の作成
* PC上にインストールされている、Aginity Workbench for Redshift（以下Aginity）を起動します。
* 接続情報を以下の通り入力します。
* Redshiftクラスターエンドポイントはコンソールから確認し、コピペして下さい。

```
Name      : （任意）
Server    : （クラスターエンドポイントのURL。ポートを除く）
DB Port   : 5439
SSL Mode  : Prefer
User ID   : awsuser
Password  : （上記で入力したパスワード）
```

### 2-2-2 接続テスト
* 入力した接続情報に基づいて、Redshiftクラスターに接続します。

# 3. ハンズオン
* ここでは、Redshiftチューニングやトラブルシューティングについてハンズオンを行っていきます。
  * **sh1**という1,500万件ほどの小型のスキーマをサンプルに使っていきます。

## 3-1. WLM実習
* 「ワークロード管理」→「WLM」から、当該クラスター用のWLM設定を作成します。

### 3-1.1. WLM関連ビューの作成
* WLM_QUEUE_STATE_VW（キュー状態を確認するためのビュー）を作成します。
```
create view WLM_QUEUE_STATE_VW as
select (config.service_class-5) as queue
, trim (class.condition) as description
, config.num_query_tasks as slots
, config.query_working_mem as mem
, config.max_execution_time as max_time
, config.user_group_wild_card as "user_*"
, config.query_group_wild_card as "query_*"
, state.num_queued_queries queued
, state.num_executing_queries executing
, state.num_executed_queries executed
from
STV_WLM_CLASSIFICATION_CONFIG class,
STV_WLM_SERVICE_CLASS_CONFIG config,
STV_WLM_SERVICE_CLASS_STATE state
where
class.action_service_class = config.service_class 
and class.action_service_class = state.service_class 
and config.service_class > 4
order by config.service_class;
```
* WLM_QUERY_STATE_VW（クエリー状態を確認するためのビュー）を作成します。
```
create view WLM_QUERY_STATE_VW as
select query, (service_class-5) as queue, slot_count, trim(wlm_start_time) as start_time, trim(state) as state, trim(queue_time) as queue_time, trim(exec_time) as exec_time
from stv_wlm_query_state;
```
* 以下を実行してそれぞれのビューの内容を確認します。
```
select * from wlm_queue_state_vw;
select * from wlm_query_state_vw;
```

### 3-1.2. WLMキューの作成と設定
* WLMキューを作成します。
　　* パラメーターグループ→クラスターパラメーターグループの作成
```
パラメーターグループファミリー ... デフォルト
パラメーターグループ名 ... 任意
説明 ... 任意
```
　　* ワークロード管理→パラメーターグループ→先程作成したパラメーターグループを選択→キューの追加
```
クエリーキュー ... メモリ割り当て70%、同時実行数（スロット）12、ユーザーグループ=queryusers
バッチキュー ... メモリ割り当て20%、同時実行数（スロット）2、ユーザーグループ=batchusers
デフォルトキュー ... メモリ割り当て残り全て、同時実行数（スロット）1
```
  * クラスター→当該クラスターのチェックボックスをチェック→クラスターボタン→変更→先程作成したパラメーターグループ
  * ***再起動がかかりますので、クラスター画面がどう変わるかを確認しつつ、暫くお待ち下さい。***
  
### 3-1-3. クエリーのテスト
* TBD

### 3-1-4. QMRルールの作成と設定
* QMRルールを作成します。
  * ワークロード管理→パラメーターグループ→先程作成したパラメーターグループを選択→編集
  * クエリーキューから「テンプレートからルールを追加」
  * 「クエリーが多くの行を返す」をチェックし、追加ボタンをクリック
  * ルールに名前を付け、アクションを「ホップ」に変更
  * 保存をクリック
* これで、クエリーキューで長大なクエリーが流れた際、自動的にバッチキューに流れるようになります。

## 3-2. その他のチューニング

### 3-2-1. テーブル情報の取得（Admin Scripts）
* Admin Scripts（table-info.sql）を使用して、クラスター内のテーブル情報を取得します。
```
SELECT TRIM(pgn.nspname) AS SCHEMA,
       TRIM(a.name) AS TABLE,
       id AS TableId,
       decode(pgc.reldiststyle,
             0, 'EVEN',
             1,det.distkey ,
             8,'ALL'
       ) AS DistKey,
       decode(pgc.reldiststyle,
             8,NULL,
             dist_ratio.ratio::DECIMAL(20,4)
       ) AS Skew,
       det.head_sort AS "SortKey",
       det.n_sortkeys AS "#SKs",
       CASE WHEN pgc.reldiststyle = 8 THEN a.rows_all_dist ELSE a.rows END AS rows,
       b.mbytes,
       decode(det.max_enc,
             0,'N',
             'Y'
       ) AS Enc,
       det.pct_enc,
       decode(b.mbytes,
             0,0,
             ((b.mbytes/part.total::DECIMAL)*100)::DECIMAL(20,2)
       ) AS pct_of_total,
       (CASE WHEN a.rows = 0 THEN NULL ELSE 
          CASE WHEN pgc.reldiststyle = 8 THEN ((a.rows_all_dist - pgc.reltuples)::DECIMAL(20,3) / a.rows_all_dist::DECIMAL(20,3)*100)::DECIMAL(20,2)
                ELSE ((a.rows - pgc.reltuples)::DECIMAL(20,3) / a.rows::DECIMAL(20,3)*100)::DECIMAL(20,2) END END
       ) AS pct_stats_off,
       CASE WHEN pgc.reldiststyle = 8 
          THEN decode( det.n_sortkeys,0, NULL,DECODE( a.rows_all_dist,0,0, (a.unsorted_rows_all_dist::DECIMAL(32)/a.rows_all_dist)*100))::DECIMAL(20,2)
          ELSE decode( det.n_sortkeys,0, NULL,DECODE( a.rows,0,0, (a.unsorted_rows::DECIMAL(32)/a.rows)*100))::DECIMAL(20,2) END
        AS pct_unsorted
FROM (SELECT db_id,
             id,
             name,
             SUM(ROWS) AS ROWS,
             MAX(ROWS) AS rows_all_dist,
             SUM(ROWS) - SUM(sorted_rows) AS unsorted_rows,
             MAX(ROWS) - MAX(sorted_rows) AS unsorted_rows_all_dist
      FROM stv_tbl_perm a
      GROUP BY db_id,
               id,
               name) AS a
  JOIN pg_class AS pgc ON pgc.oid = a.id
  JOIN pg_namespace AS pgn ON pgn.oid = pgc.relnamespace
  LEFT OUTER JOIN (SELECT tbl, COUNT(*) AS mbytes FROM stv_blocklist GROUP BY tbl) b ON a.id = b.tbl
  INNER JOIN (SELECT attrelid,
                     MIN(CASE attisdistkey WHEN 't' THEN attname ELSE NULL END) AS "distkey",
                     MIN(CASE attsortkeyord WHEN 1 THEN attname ELSE NULL END) AS head_sort,
                     MAX(attsortkeyord) AS n_sortkeys,
                     MAX(attencodingtype) AS max_enc,
                     SUM(case when attencodingtype <> 0 then 1 else 0 end)::DECIMAL(20,3)/COUNT(attencodingtype)::DECIMAL(20,3)  *100.00 as pct_enc
              FROM pg_attribute
              GROUP BY 1) AS det ON det.attrelid = a.id
  INNER JOIN (SELECT tbl,
                     MAX(Mbytes)::DECIMAL(32) /MIN(Mbytes) AS ratio
              FROM (SELECT tbl,
                           TRIM(name) AS name,
                           slice,
                           COUNT(*) AS Mbytes
                    FROM svv_diskusage
                    GROUP BY tbl,
                             name,
                             slice)
              GROUP BY tbl,
                       name) AS dist_ratio ON a.id = dist_ratio.tbl
  JOIN (SELECT SUM(capacity) AS total
        FROM stv_partitions
        WHERE part_begin = 0) AS part ON 1 = 1
WHERE mbytes IS NOT NULL
AND   pgc.relowner > 1
-- and pgn.nspname = 'schema' -- schemaname
-- and a.name like 'table%' -- tablename
-- and det.max_enc = 0 -- non-compressed tables
ORDER BY mbytes DESC;

```

### 3-2-2. オープンセッションの確認
* Admin Viewsを使用して、最近使用されたセッション（現在オープン中のものを含む）を確認します。
  * ビューを作成
```
CREATE OR REPLACE VIEW admin.v_open_session
AS
SELECT
	CASE WHEN disc.recordtime IS NULL THEN 'Y' ELSE 'N' END AS connected
	,init.recordtime AS conn_recordtime
	,disc.recordtime AS disconn_recordtime
	,init.pid AS pid
	,init.remotehost
	,init.remoteport
	,init.username AS username
	,disc.duration AS conn_duration
FROM 
	(SELECT event, recordtime, remotehost, remoteport, pid, username FROM stl_connection_log WHERE event = 'initiating session') AS init
LEFT OUTER JOIN
	(SELECT event, recordtime, remotehost, remoteport, pid, username, duration FROM stl_connection_log WHERE event = 'disconnecting session') AS disc
		ON init.pid = disc.pid
		AND init.remotehost = disc.remotehost
		AND init.remoteport = disc.remoteport
;  
```
  * ビューの内容を確認（UTC表記となる点に留意）
```
select * from v_open_session
order by conn_recordtime desc
;
```

### 3-2-3. 圧縮エンコーディング
* 圧縮エンコーディングが適切に設定されているかどうかを確認します。
  * 現在の圧縮エンコーディングを表示（DbVisualizerでスキーマを**sh1**に変更すること）
```
select "column", type, encoding, distkey, sortkey, "notnull" 
from pg_table_def
where tablename = 'sales'
```
  * 推奨される圧縮エンコーディングを表示
```
analyze compression sh1.sales;
```

### 3-2-4. Actualの確認
* クエリー分析に頻繁に利用される「Actual」画面を確認します。
  * クラスター→当該クラスターをクリック→クエリータブ
  * 任意のクエリーを選んでクエリーIDをクリック
  * クエリーの実行の詳細→Plan
  * クエリーの実行の詳細→Actual
  * Cluster Performance During Query Executionも併せて確認

# 4. ハンズオン環境の削除
* Redshiftハンズオン環境の削除は弊社にて行います。そのままにしておいていただいて結構です。

***

# Amazon Aurora PostgreSQL ハンズオン - Day2

# 4. 環境構築準備
  * **本ハンズオンでは環境は事前に作成済です。**
  * **マスターユーザー名及びパスワード等は別途おしらせします。**
  * **コンソール右上で「東京」リージョンが選択されていることだけ、再度ご確認下さい。**

  * **アクセス元の制限（既に他のアドレスに制限されている状態になっています）**

  * 「サービス」→「RDS」→「インスタンス」→「インスタンス」ペイン →「DBインスタンス」にリストされている2つのDBのいずれか一つのリンクをクリック
    * スクロールダウンして「詳細」ペイン →「セキュリティグループ」のActive状態のセキュリティグループのリンクをクリック
    * 「EC2」ダッシュボード → 「セキュリティグループ」ペイン　→ 「インバウンド」タブ → 「編集」ボタン をクリック
    * 「インバウンドのルールの編集」ダイアログ → 「タイプ」が"PostgreSQL"となっているルールの「ソース」列 → 「マイIP」→ 「保存」ボタンをクリック

  * この操作で、Aurora PostgreSQLの各インスタンスは、現在操作中のPCのIPアドレスからのアクセスのみ受け付けるようになります


## 4-1 クライアントの接続設定
  * 事前にご利用するPCへ DbVisualizer インストール願います。
  * psqlでも可能ですが利用される場合は、事前にお知らせするマスターパスタワード及び、読み込みエンドポイントとクラスターエンドポイント、各レプリカのDBインスタンスエンドポイントをメモ帳へコピーしておいてください。

  * **Database Serverに設定するクラスターエンドポイント及び、読み込みエンドポイントの確認方法は以下の通りです。**
    * 「サービス」→「RDS」→「クラスター」→「DBクラスター識別子」以下の該当クラスターのリンク→「詳細」ペインから確認できます。

  * **各レプリカのインスタンスエンドポイントの確認方法は以下の通りです。**
    * 「サービス」→「RDS」→「クラスター」→「クラスターメンバー」ペイン → 「ロール」列が **読み込み** となっている「dbインスタンス」列のリンクをクリック → 「接続」ペインから確認できます。


### 4-1-1 **クラスターエンドポイント向け接続設定**
  * DbVisualizerの接続設定を行います。
  * 「Databases」タブ → 「Connections」ツリーを選択し右クリック → ポップアップメニュー「Create Database Connection」 → 「User Connection Wizard?」ダイアログ  → 「Use Wizard」ボタン → 「New Connection Wizard」ダイアログ

  ```
  Connection Alias : Cluster Endpoint
  ```
  * 「Next」ボタン

  ```
  Select Database Driver : PostgreSQL
  ```
  * 「Next」ボタン

  ```
  Notes                  : -
  Setting format         : Server Info
  Database Server        : 事前作成されているap96-clusterクラスターのクラスターエンドポイント
  Database Port          : 5432
  Database               : mydb
  Database UserId        : awsuser
  Database Password      : 本日お知らせしたマスターパスワード
  Auto Commit            : チェックボックスをチェック
  Save Database Password : Save between Sessions
  Permisssion mode       : Development
  ```
  * **「Ping Server」ボタンをクリツクし接続確認**
  * 「Finish」ボタン

### 4-1-2 **読み込みエンドポイント向け接続設定**
  * 4-1-1 で作成した **クラスターエンドポイント向け接続設定** を選択し右クリック → ポップアップメニュ「Duplicate Database Connection」
    * 以下項目のみ設定しその他内容は、4-1-1 の設定と同じ

  ```
  Name    　　　　　　　　　: Reader Endpoint
  Database Server        : 事前作成されているap96-clusterクラスターの読み込みエンドポイント
  ```
  * **「Ping Server」ボタンをクリツクし接続確認**
  * 上記を設定したタブを閉じる

### 4-1-3 **レプリカ1のインスタンスエンドポイント向け接続設定**
  * **便宜上、レプリカ1、レプリカ2と記載していますが、2つあるレプリカのうちどちらを1,2にしても構いません。**

  * 4-1-1 で作成した **クラスターエンドポイント向け接続設定** を選択し右クリック → ポップアップメニュ「Duplicate Database Connection」
      * 以下項目のみ設定しその他内容は、4-1-1 の設定と同じ

  ```
  Name    　　　　　　　　　: Replica1 Endpoint
  Database Server        : 事前作成されているap96-clusterのレプリカ1のDBインスタンスエンドポイント
  ```
  * **「Ping Server」ボタンをクリツクし接続確認**
  * 上記を設定したタブを閉じる

### 4-1-3 **レプリカ2のインスタンスエンドポイント向け接続設定**

  * 4-1-1 で作成した **クラスターエンドポイント向け接続設定** を選択し右クリック → ポップアップメニュ「Duplicate Database Connection」
    * 以下項目のみ設定しその他内容は、4-1-1 の設定と同じ

  ```
  Name    　　　　　　　　　: Replica1 Endpoint
  Database Server        : 事前作成されているap96-clusterのレプリカ2のDBインスタンスエンドポイント
  ```
  * **「Ping Server」ボタンをクリツクし接続確認**
  * 上記を設定したタブを閉じる

# 5. ハンズオン
* Aurora PostgreSQLクラスターを利用した運用監視、チューニングなどのトラブルシューティングを体験していだだきます。

## 5-1. Auto Scaling （オプション）
* レプリカに一定の負荷をかけScale out / Scale inを体験していただきます。(CloudWatch Dashboardの作成及び、イベントサブスクリプションe-mailの設定含む)

### 5-1-1. Auto Scalingの設定 （オプション）
  * 「サービス」→「サイドバー」→「クラスター」→「クラスター」ペイン → 「DBクラスター識別子」列の ap96-clusterのリンク → 「Auro Scalingポリシー」ペイン →　「追加」ボタン
  * → 「Auto Scalingポリシーの追加」画面 → 「ポリシーの詳細」ペイン

  ```
  ポリシー名　　　　　 : my-auto-scaling-polycy
  ターゲットメトリクス : 「Auroraレプリカの平均CPU使用率」ラジオボタン
  ターゲット値　　　　 : 50
  ```
  * 「▶︎追加の設定」をクリックして拡張

  ```
  スケールイン                : 「有効」スライドボタン
  スケールインクールダウン期間　 : 300
  スケールアウトクールダウン期間 : 300
  ```

  * スラスターキャパシティの詳細
  ```
  最小キャパシティ : 2
  最大キャパシティ : 3
  ```

  * 「ポリシーの追加」ボタンをクリック

### 5-1-2. イベントサブスクリプションの設定 （オプション）
  * 「サイドバー」→ 「イベントサブスクリプション」→ 「イベントサブスクリプションの作成」ボタン →「イベントサブスクリプションの作成」画面

  * 「詳細」ペイン
  ```
  名前 : create-or-delete-instance
  ```

  * 「ターゲット」ペイン
  ```
  通知の送信先 : 「新しいEメールトピック」ラジオボタン
  トピック名　 : create-or-delete-instance
  受信者　　　 : Confirmation可能な実在するメールアドレス
  ```  

  * 「ソース」ペイン
  ```
  ソースタイプ　　　　　　 : ドロップダウンメニュー「インスタンス」
  インスタンスを含める　　 : 「全てのインスタンス」ラジオボタン
  含まれるイベントカテゴリ : 「特定のイベントカテゴリを選択します」ラジオボタン数
  特定のイベント　　　　　 : ポップアップメニューから「作成」と「削除」を選ぶ
  ```

  * 「作成」ボタンをクリック

### 5-1-3. CloudWatch Dashboardの設定 （オプション）
  * 「サービス」→「サイドバー」→「ダッシュボード」→「ダッシュボード」ペイン → 「ダッシュボードの作成」ボタン

  * 「新しいダッシュボードの作成」ダイアログ
  ```
  ダッシュボード名 : handson
  ```
   * 「ダッシュボードの作成」ボタンをクリック

  * 「ダッシュボードに追加」ダイアログ
  ```
  「数値」ボタンアイコン
  ```
    * 「設定」ボタンをクリック

  * 「メトリクスグラフの追加」ダイアログ → 「すべてのメトリクス」タブ → 「RDS」→「データベース別メトリクス」→ 「検索ボックス」
  ```
  CPUUtilization
  ```
  を入力して↩︎[CR] → リストされた **ap96*** の 3 DBインスタンスのチェックボックスをクリック → 「ウィジェットの作成」ボタンをクリック

　* 「ウィジェットの追加」ボタン → 「ダッシュボードに追加」ダイアログ
　```
　「数値」ボタンアイコン
　```
　  * 「設定」ボタンをクリック

　* 「メトリクスグラフの追加」ダイアログ → 「すべてのメトリクス」タブ → 「RDS」→「データベース別メトリクス」→ 「検索ボックス」
　```
  FreeLocalStorage
　```
　を入力して↩︎[CR] → リストされた **ap96*** の 3 DBインスタンスのチェックボックスをクリック → 「ウィジェットの作成」ボタンをクリック → 「ダッシュボードの保存」ボタンをクリック


### 5-1-4. Auto Scalingによるスケールアウトとスケールイン （オプション）
  * DbVisualizer を 3 つ起動（無料版では、複数のSQL Commanderペインをひらけないため）
    * 2のDbVislalizerで **Replica1 Endpoint** へ接続し、残りの1つは、 **Replica2 Endpoint** へ接続し、それぞれで、SQL Commander ペインを開く

    * **操作例**
      * 「Databaes」タブ → 「Connections」ツリー → 「Replia1 Endpotion」を選択し右クリック → ポップアップメニュー「Connect」
      * 「メニュー」→「SQL Commander」→ 「Database」は **mydb** 、 「Schema」は、 **hoge** 　になっていることを確認、なっていない場合には、各ドロップダウンメニューより選択
        * それぞれの **SQL Commander** ペインで以下を入力
        ```
        DO $$BEGIN
          LOOP
            --
          END LOOP;
        END$$;
        ```
        * 入力後　→ 「メニュー」→「SQL Commander」→「Execute Buffer」を選択して実行

### 5-1-5. CloudWatch Dashboardでのモニタリング （オプション）
  * レプリカ1のCPUUtilizatonが 100% 、レプリカ2のCPUUtilizatonが 50% 程度になりしばらくすると、Auto Scalingにより、レプリカが１インスタンス自動作成
    * インスタンスが作成されると、イベントサブスクリプションで登録した e-mailへDBインスタンスが作成されたことが通知される

* ** ここで座学に戻ります **
* **レプリカが追加されたことを通知するメールが届いたら状況を確認するためハンズオンを再開します**

### 5-1-6. Auto Scalingの確認 （オプション）
  * 「サービス」→「RDS」→「クラスター」→「クラスター」ペイン → 「DBクラスター識別子」列の **ap96-cluster** のリンク → 「DBクラスターメンバー」ペイン
    * 新規インスタンスが作成されているとを確認してください。作成されたDBインスタンス名のリンクをクリック →
    * 「詳細」ペインの「フェイルオーバー優先度」が **15** （最も低い)となっで作成されていることなどを確認

#### 5-1-6-1. DbVisualizer で実行中の負荷かけスクリプトの停止 （オプション）
  * 「DbVisualizer」→ 「メニュー」→「SQL Commander」→「Stop」を洗濯してスクリプトを停止させる
  * 同様の手順で残る 2つの負荷かけスクリプトを停止

* ** ここで座学に戻ります **


## 5-2. フェイルオーバー  （オプション）
* フェイルオーバーの優先度の確認後、フェイルオーバーさせ、フェイルオーバー完了までの時間やフェイルオーバー先の制御を体験していただきます。
  * **フェイルオーバーの優先度は以下のように設定済み**
  |DBインスタンス識別子|フェイルオーバー優先度|プライマリーインスタンス?|
  |---:|:---|:---|
  |ap96|2|Primary Instance|
  |ap96-XXX-1c|2|Replica|
  |ap96-XXX-1d|1|Replica|
  |application-autoscaling-xxx|15|Replica(Auto Scalingで追加。既に削除されている可能性もあります)|

* **ハンズオンPart.1はここで終了です。早めに終了した方は休憩していただいて結構です。**


### 5-2-1. イベントサブスクリプションの設定 （オプション）
* イベントサブスクリプションを設定しフェイルオーバー発生の通知をメールで受け取れるようにします。
  * 「サービス」→「RDS」→「イベントサブスクリプション」→「イベントサブスクリプションの作成」ボタンをクリック
  * 「イベントサブスクリプションの作成」画面にて以下を設定
    * 詳細ペイン
    ```
    名前（サブスクリプションの名前） : failover
    ```

    * ターゲットペイン
    ```
    通知の送信先 : 「新しいEメールトピック」ラジオボタンを選択
    トピック名　 : failover
    受信者　　　 : Confirmation可能な実在するメールアドレス
    ```

    * ソースペイン
    ```
    ソースタイプ : ドロップダウンメニューから「DBクラスター」を選択
    DBクラスターを含める : 「個別に選択 db クラスター」ラジオボタンを選択
    指定DBクラスター　　 : ドロップダウンメニューから「 **ap96-cluster** 」を選択
    含まれるイベントカテゴリー : 「特定のイベントカテゴリを選択します」ラジオボタンを選択
    特定のイベント　　　　　　 : ドロップダウンメニューから「 **フェイルオーバー** 」を選択
    ```
    * 「作成」ボタンをクリック

    * 「ターゲットペイン」の「受信者」で設定したメールアドレスへ、Confirmationメールが届きますので、メールのリンクをクリックしてConfirmationします。
      * なお、Confirmationできない誤ったメールアドレスを設定した場合には、Pending Confirmationとなり3日後に自動削除されます。
      * メールアドレスのConcifmation状況は以下の方法で確認できます。
        * 「サービス」→「Simple Notification Service」→「サブスクリプション」→「サブスクリプション」画面
          * 「サブスクリクション」画面のフィルターボックスで **PendingConfirmation** と入力するとPending中のConfirmation一覧を確認することができます。


### 5-2-2. 手動フェイルオーバー （オプション）
* 手動フェイルオーバーを行い、フェイルオーバー優先度:1の **ap96-XXX-1d** インスタンスがプライマリーインスタンスへ昇格することを確認してください。
  * 「詳細」ペインの「フェイルオーバー優先順位」を参照している画面上部の「インスタンス操作」メニュー → 「フェイルオーバー」→ 「Failover DB Cluster」画面 → 「フェイルオーバー」ボタンをクリック
  * 「サイドバー」→「クラスター」→「DBクラスター識別子」列下のDBクラスター名のハイパーリンク → 「DB クラスターメンバー」ペイン → 「更新」ボタン
    * レプリカ　**ap96-XXX-1d** のロールが、読み込みから書き込みへ昇格したことを確認してください。

  * フェイルオーバーが発生後、イベントサブスクリクションで登録したメールアドレスへフェイルオーバーの通知が届きます。どのような内容のメールが届くのかかご確認ください。
    * 「最近のイベント」ペインに以下のようなイベントが記録されていることも確認してみてください。

   ```
    Completed failover to DB instance: ap96-XXX-1d
    Started cross AZ failover to DB instance: ap96-XXX-1d
   ```


## 5-3. Performance Insights
  * **(日本語訳はパフォーマンスインサイトになっていますが、最後の s は大切です)**
  * Performance Insightsを利用した分析及び、チューニングの流れを体験していただきます。


### 5-3-1. 遅延分析及びチューニング　その１　
  * 「サービス」→「RDS」→「パフォーマンスインサイト」→ DBインスタンス「ap96」ラジオボタンを選択　→　「データベースのロード」ペイン　を表示
  * DbVisualizer → 「Databases」タブ → 「Connections」ツリー → 「Cloud Endpoint」を選択、右クリック → ポップアップメニュー「Connect」
  * DbVisualizer → 「メニュー」→「SQL Commander」で、SQL Commanderペインを開く

  * **以下のSQL文の性能要件は、TAT 1秒以内です**
    * 以下の SQL文を SQL Commanderへコピーペーストし、「メニュー」→「SQL Commander」→「Execute」を選択しSQL文を実行
  ```
  SELECT COUNT(ref_no) FROM t1 WHERE ref_no BETWEEN 1 AND 250;
  ```
  * SQL文実行後、Performance Insightsのデータベースのロードペインを確認
    * Top Load Item Table上部のスライス →　「待機」、下部のスライス → 「SQL」を選択し、時間軸から 5 分を選択 →
    * 数秒程度のSQL文が見つかるはずです。グラフが小さい場合は、拡大したい範囲をマウスで選択することで拡大できます。
    * SQL文横の▶︎アイコンをクリックしてドリルダウンするとSQL文が現れるはずです。
    ![Performance Insights](https://github.com/lampeyes/aws-aurora-postgre-study/blob/master/handson_day2_aurora_postgresql_001.png "遅延分析及びチューニング　その１")


      * このSQL文の待機イベントの傾向からどのようなことが言えますか？
      * バッファキャッシュに乗っている状態であれば、2秒程度で、待機イベントはCPUバウンドな傾向を示しているはずです。
        * バッファキャッシュでヒットしなかった状況は考慮するしなくて構いません。バッファキャッシュでヒットする前提です。
      * チューニング方法を検討するため、実行計画(Actual)を確認する必要がありますが、どのような方法で確認できますか？
      * 実行計画(Actual)と、定義(DbVisualizerで列定義や索引）を確認できます。  
      * 実行計画(Actual)は以下のように取得できます。  
      ```
      explain analyze SELECT COUNT(ref_no) FROM t1 WHERE ref_no BETWEEN 1 AND 250;
      ```

      * 実行計画は以下の通り
      ```
      Aggregate  (cost=289382.49..289382.50 rows=1 width=8) (actual time=2160.475..2160.475 rows=1 loops=1)
      ->  Seq Scan on t1  (cost=0.00..283334.00 rows=2419397 width=5) (actual time=0.093..1994.440 rows=2500000 loops=1)
        Filter: ((ref_no >= '1'::numeric) AND (ref_no <= '250'::numeric))
        Rows Removed by Filter: 7500000
      Planning time: 0.074 ms
      Execution time: 2160.498 ms
      ```

      * 以下の SQL文を SQL Commanderへコピーペーストし、「メニュー」→「SQL Commander」→「Execute」を選択しSQL文を実行して、セグメントサイズを確認してみてください
      ```
        SELECT
          objectname
          ,TO_CHAR(
            pg_relation_size(
              objectname::regclass)/1024/1024/1024
              , '999,999,999,999.99'
          ) AS "GB"
        FROM
        (
          SELECT
            tablename AS objectname
          FROM
            pg_tables
          WHERE
            schemaname = 'hoge'
          UNION
          SELECT
            indexname AS objectname
          FROM
             pg_indexes
          WHERE
            schemaname = 'hoge'
        ) AS objectlist
        ORDER BY
           2 DESC;
        ```

        * 結果は以下の通り
        ```
        objectname |         GB          
        -----------+---------------------
        t1         |                1.00
        t2         |                1.00
        ```

        * これらのことから、t1表のref_no列に索引が存在しないため、1GBのデータサイズ、かつ、 10,000,000行のデータを全表走査し、7,500,000行のデータを読み込んだ上で捨てていることがわかります。
        * この例では物理読み込みはキャッシュにヒットしているため影響していないが、1GBは読み込み過ぎ、なので、適切な索引があれば、フィルターリングによるCPU処理も全表走査の読み込み量も同時に減らせる。
        * ref_no以外に、seq_noも検索されそうなので、ref_no,seq_no列に索引を作成することで大きく改善する
        ```
        create index ix_t1 on t1(ref_no, seq_no);
        ```

        ```
        explain analyze SELECT COUNT(ref_no) FROM t1 WHERE ref_no BETWEEN 1 AND 250;
        ```

        ```
        Aggregate  (cost=91736.67..91736.68 rows=1 width=8) (actual time=624.912..624.912 rows=1 loops=1)
        ->  Index Only Scan using ix_t1 on t1  (cost=0.43..85685.98 rows=2420277 width=5) (actual time=0.034..457.772 rows=2500000 loops=1)
            Index Cond: ((ref_no >= '1'::numeric) AND (ref_no <= '250'::numeric))
            Heap Fetches: 0
        Planning time: 0.110 ms
        Execution time: 624.940 ms
        ```

### 5-3-2. 遅延分析及びチューニング　その２
  * 「サービス」→「RDS」→「パフォーマンスインサイト」→ DBインスタンス「ap96」ラジオボタンを選択　→　「データベースのロード」ペイン　を表示
  * DbVisualizerを 2つ起動して、それぞれ Cloud Endpointに接続し、メニューから「SQL Commander」を開いた状態にします。
  * DbVisualizer → 「Databases」タブ → 「Connections」ツリー → 「Cloud Endpoint」を選択、右クリック → ポップアップメニュー「Connect」
  * DbVisualizer → 「メニュー」→「SQL Commander」で、SQL Commanderペインを開く
  * DbVisualizer → 「メニュー」→「SQL Commander」→ 「Transaction」→　サブメニュー「Turn off auto commit」を選択する（2つのDbVisualizerともAuto Commitはoffにする）

  * 1つめのDbVisualizerで以下のSQL文を実行
  ```
  update t1 set description = 'hoge' where ref_no = 1;
  ```

  * 2つめのDbVisualizerで以下のSQL文を実行（こちらのSQL文はトランザクションのcommit/rollbackを待機することになります）
  ```  
  update t1 set description = 'hoge' where ref_no = 1;
  ```

* Performance InsightsのTop Load Item Tableの上部スライスは「待機」を選択、下部のスライスは「SQL」を選択することで、2つめのupdate文がロックの解放待ちになっていることが確認できるはずです。
  * Performance Insightsを活用するには待機イベントの考え方を理解することが重要です。

  * 以下のSQL文を、1つめのDbVisualizerのSQL Commanderから実行してください。
  ```
  SELECT
    clock_timestamp() AS "CURRENT_TIMESTAMP"
    , pg_class.relname
    , pg_locks.locktype
    , pg_locks.database
    , pg_locks.relation
    , pg_locks.page
    , pg_locks.tuple
    , pg_locks.virtualtransaction
    , pg_locks.pid
    , pg_locks.mode
    , pg_locks.granted
  FROM
    pg_locks
    INNER JOIN pg_class
    ON
      pg_locks.relation = pg_class.oid
  WHERE
    relname !~ '^pg_'
    AND  relname <> 'active_locks'
  ;
  ```

  * 以下のサンプルのように、ロックのタイプとロックしているオブジェクトを確認できます。
  ```
    CURRENT_TIMESTAMP       | relname | locktype | database | relation | page | tuple | virtualtransaction |  pid  |       mode       | granted
  -------------------------------+---------+----------+----------+----------+------+-------+--------------------+-------+------------------+---------
  2018-11-09 06:19:50.395239+00 | ix_t1   | relation |    16394 |    24588 |      |       | 7/334              | 27126 | RowExclusiveLock | t
  2018-11-09 06:19:50.395254+00 | t1      | relation |    16394 |    16401 |      |       | 7/334              | 27126 | RowExclusiveLock | t
  2018-11-09 06:19:50.395256+00 | ix_t1   | relation |    16394 |    24588 |      |       | 6/575              | 27722 | RowExclusiveLock | t
  2018-11-09 06:19:50.395258+00 | t1      | relation |    16394 |    16401 |      |       | 6/575              | 27722 | RowExclusiveLock | t
  2018-11-09 06:19:50.39526+00  | t1      | tuple    |    16394 |    16401 |    0 |    77 | 7/334              | 27126 | ExclusiveLock    | t
  (5 rows)
  ```

### 5-3-3. 遅延分析及びチューニング　その３
  * **TBD さらにオプションのチューニング例を追記するかもしれない。**


## お疲れ様でした。これでハンズオンは終了です。
## **ハンズオン環境の削除等はこちらで行います。そのままにしておいていただいて結構です。**
