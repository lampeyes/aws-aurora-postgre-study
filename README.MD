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

## 2-2. EC2からのデータベースへの接続確認
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
