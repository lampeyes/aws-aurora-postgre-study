# Amazon Redshift ハンズオン - Day1

# 1. 環境構築準備
## 1-0. ハンズオンで利用するアカウント
* TBD
```
TBD
```

## 1-1. ハンズオン向けコンテンツの取得
```
利用するデータやスクリプトのダウンロード
```

## 1-2. CloudFormationによるハンズオン環境の構築
* 「サービス」 → 「CloudFormation」 → 「スタックの作成」→
* 「新しいスタックの作成」ボタン →　
* 「Amazon S3 テンプレート URL の指定」チェックボックス →
*  https://<your_workshop_bucket>.s3.amazonaws.com/master.yaml を入力　→
*  スタックの名前 : aws-XXXX-study　(XXXXはユーザー名)　を入力　→
*  MasterUserPass : xxxxxx (8文字以上、半角英数字大文字小文字いずれも１文字以上含む)を入力　→
* 「次へ」ボタン →　オブション入力画面はデフォルトのまま「次へ」ボタン　→
*  確認画面の最下部の「AWS CloudFormation によってカスタム名のついた IAM リソースが作成される場合があることを承認します。」チェックボックスをチェック　→　「作成」ボタン
* ステータスが "CREATE_COMPLETE"　になるまで待ちます

**以下のスタックが構築されます:**
```
aws-XXXX-study-SecurityGroupStack-XXXXXXXXXXXXX [NESTED]
aws-XXXX-study-IAMStack-XXXXXXXXXXXX [NESTED]
aws-XXXX-study-VPCStack-XXXXXXXXXXXX [NESTED]
aws-XXXX-study
```

# 2. 接続確認

## 2-1. Management Console（MC）からデータベースの状態を確認
* 「サービス」→「Redshift」→「クラスター」タブに移動します。
* クラスターが作成されていることを確認します。
* 「クラスター」→当該クラスターをクリックすると、そのクラスターに移動して詳細情報を閲覧できます。「設定」タブ、「ステータス」タブをクリックして、それぞれどのような情報が表示されるか確認してみて下さい。

## 2-2. データベースへの接続確認
### 2-2-1 接続設定の作成
* PC上にインストールされている、Aginity Workbench for Redshiftを起動します。
* 接続情報を以下の通り入力します。
```
TBD
```
* Redshiftクラスターのエンドポイント名は、**2-1.**に記載のクラスター詳細情報から探してみて下さい。

### 2-2-2 接続とテスト
* 入力した接続情報に基づいて、実際にRedshiftクラスターに接続します。
```
TBD
```

# 3. ハンズオン
ここでは、Redshiftの基本的な使い方についてハンズオンを行っていきます。

## 3-1. COPYの実行
* TBD

```
入力するコマンドなど(コピペで可能な様に）
```

## 3-2. メンテナンストラック、パラメーターグループ設定、WLM設定の閲覧
* TBD

```
入力するコマンドなど(コピペで可能な様に）
```

## 3-3. クエリーの実行
* TBD

```
入力するコマンドなど(コピペで可能な様に）
```
## 3-4. COPYおよびクエリーの実行状況の確認

### 3-4-1 COPY（ロード）の実行確認
* TBD

### 3-4-2 クエリーの実行確認
* TBD

### 3-4-3 クラスターの性能情報確認
* TBD

## 3-5. バックアップの取得
* TBD

## 3.6. イベントの閲覧
* TBD

## 3.7. リストア
### 3.7.1 リストアの実行とクエリーの実行
* TBD

### 3.7.2. テーブルレベルの復元
* TBD


## 4. ハンズオン環境の削除
* TBD







# AWS XXXX ハンズオン (テンプレ)

# 1. 環境構築準備
## 1-0. ハンズオンで利用するアカウント
* TBD
```
TBD
```

## 1-1. ハンズオン向けコンテンツの取得
```
利用するデータやスクリプトのダウンロード
```

## 1-2. CloudFormationによるハンズオン環境の構築
* 「サービス」 → 「CloudFormation」 → 「スタックの作成」→
* 「新しいスタックの作成」ボタン →　
* 「Amazon S3 テンプレート URL の指定」チェックボックス →
*  https://<your_workshop_bucket>.s3.amazonaws.com/master.yaml を入力　→
*  スタックの名前 : aws-XXXX-study　(XXXXはユーザー名)　を入力　→
*  MasterUserPass : xxxxxx (8文字以上、半角英数字大文字小文字いずれも１文字以上含む)を入力　→
* 「次へ」ボタン →　オブション入力画面はデフォルトのまま「次へ」ボタン　→
*  確認画面の最下部の「AWS CloudFormation によってカスタム名のついた IAM リソースが作成される場合があることを承認します。」チェックボックスをチェック　→　「作成」ボタン
* ステータスが "CREATE_COMPLETE"　になるまで待ちます

**以下のスタックが構築されます:**
```
aws-XXXX-study-SecurityGroupStack-XXXXXXXXXXXXX [NESTED]
aws-XXXX-study-IAMStack-XXXXXXXXXXXX [NESTED]
aws-XXXX-study-VPCStack-XXXXXXXXXXXX [NESTED]
aws-XXXX-study
```

# 2. 接続確認

## 2-1. Management Consoleからデータベースの状態を確認
* TBD

## 2-2. EC2からデータベースへの接続確認
* TBD

利用するPCのOSが全て同一で特定のものだけであれば、そのOS固有の手順のみにした方がいいかも
#### 2-2-1 Windows利用時

#### 2-2-2 macOS利用時

# 3. ハンズオン
TBD

## 3-1. ハンズオン１
* TBD

```
入力するコマンドなど(コピペで可能な様に）
```

## 3-2. ハンズオン2
* TBD

```
入力するコマンドなど(コピペで可能な様に）
```

## 3-3. ハンズオン3
* TBD

```
入力するコマンドなど(コピペで可能な様に）
```


## 4. ハンズオン環境の削除
* TBD
