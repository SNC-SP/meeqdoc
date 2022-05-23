# MEEQデータプラットフォーム IoTストレージ

REST for Jsonを利用したデータ収集・保存サービスです。  
ユーザーは、サービス開始時に指定されたURLに対してHTTPSリクエストを送信するだけで、専用のDynamoテーブル、S3バケットにデータが保存されます。

MEEQ閉域SIMを利用したサービスのため、同サービスではインターネットを介しません。  
セキュアにデータ送信が可能なため、端末側のセキュリティを考慮する必要なくご利用頂けます。

# API

サービス開始時に指定されたURLに対してHTTPSリクエストを送信してください。

## setDeviceData

HTTPS POST  
```text
https://awsapi.gateway.com/api/setdevicedata
```
定期/計測データ保存用API。  
Content-Typeには、application/jsonを指定し、ボディにはJson形式のデータを設定してください。

本APIでは、受信データに対し、データ送信端末の電話番号とデータ受信時刻のTimestamp情報を自動的に付与します。  
テーブル上のプライマリパーティションキーは、「tel」  
プライマリソートキーは、「createtime」となります。  
受信したデータはdeviceDataパラメータにラッピングされてDynamoテーブルに保存されます。  

このため、端末が送信するデータに固有情報を付与しなくても端末に割り当てたSIMの電話番号を利用したデータ解析も可能です。  

例)
* 端末送信データ
```json
{
  "data1":123456,
  "data2":"hoge",
  "data3":9.8765
}
```

* Dynamoテーブル書き込み情報
```json
{
  "tel": "08012345678",
  "createtime": 123456789,
  "deviceData": {
    "data1": 123456,
    "data2": "hoge",
    "data3": 9.8765
  }
}
```

### 送信データのAttribute名について

送信データのAttribute名は、DynamoDBの予約語を使用しないように注意してください。  
予約語の種類については、下記を参照ください。  
https://docs.aws.amazon.com/ja_jp/amazondynamodb/latest/developerguide/ReservedWords.html

## putImageFile

HTTPS POST/PUT

イメージファイル保存用API。  
サービスアカウント固有に生成したS3バケットにデータを保存します。  
データ送信端末の電話番号を利用してディレクトリを作成し、その配下にファイルを保存します。
定期的にアップロードする場合、ユーザー側でタイムスタンプ情報等固有情報をファイル名に付与してください。

ファイル名はURLのラストパスに指定します。  
例)  
```text
https://awsapi.gateway.com/api/putimagefile/abc.jpeg
```

対応するContent-Typeは下記の通り。  
```
application/octet-stream
image/jpeg
image/png
```

* Tips  
利用シーンによっては、S3に配置するファイルを下記のような形で  
```text
s3://backet-name/02012345678/20220101/image.jpg
```
デバイス毎のフォルダ配下にサブフォルダを設けて配置したいという方もいるはずです。  
その場合は、setMultiData APIをご利用頂くことで解決できます。  
詳細はsetMultiData APIに記載。  

## setMultiData

HTTPS POST  
```text
https://awsapi.gateway.com/api/setmultidata
```

setDeviceDataとputImageFileを同時に行うことが可能なAPI。  
multipart/form-data を利用することでDynamoDBとS3への格納を同時に行います。  
画像ファイルのアップロードのみに使用することも可能です。  
本APIを利用した場合、image contentsのContent-Dispositionのnameパラメータにpath付きでファイル名を指定することで、SIMキー情報配下の任意のS3
パスにデータを格納することが可能になります。  
例)  
```text
Content-Type: multipart/form-data; boundary=【user指定】
--【user指定】
Content-Disposition: form-data; name="yyyy/mm/dd/HH/MM/SS/image.jpeg" filename="file_yyyymmddHHMMSS.jpeg"
Content-Type: image/jpeg
【binary data】
--【user指定】
Content-Disposition: form-data; name="devicedata" filename="data.json"
Content-Type: application/json
{“lon”=123.456,”lat”=43.21,”id”=”RX-001”,”image”=”yyyy/mm/dd/HH/MM/SS/image.jpeg”}
```

## detectFire

HTTPS POST/PUT

イメージファイル保存と炎検知用のAPI。  
サービスアカウント固有に生成したS3バケットにデータを保存します。  
保存したイメージファイルに対して、AIを用いて炎検知を行います。  
炎検知対象を保存するディレクトリ以下に、データ送信端末の電話番号を利用してディレクトリを作成し、その配下にファイルを保存します。  
定期的にアップロードする場合、ユーザー側でタイムスタンプ情報等固有情報をファイル名に付与してください。

ファイル名はURLのラストパスに指定します。  
例)
```text
https://awsapi.gateway.com/api/detectfire/abc.jpeg
```

対応するContent-Typeは下記の通り。
```text
image/jpeg
image/png
```

内部処理でエラーが発生した場合、エラーコードを返します。
詳細は以下の通りです。

|エラーコード|内容|
|---|---|
|MDP_E_0000|送信元IPアドレスが指定されていない|
|MDP_E_0001|送信元がMEEQで管理されていないIPアドレス|
|MDP_E_0003|ファイル名が指定されていない|
|MDP_E_0004|Bodyにデータが設定されていない|

## getFile

HTTPS GET  

サービスアカウント固有に生成したS3バケットからファイルを取得するためのAPIです。  
デバイスのファームアップデートや設定変更ファイルのダウンロード等にご利用いただけます。  
取得可能なファイルは、データ送信端末の電話番号ディレクトリ配下のみに限定されています。  
URLにfilenameパラメータを付与し、値に取得するファイル名を指定します。  
例)  
```text
https://awsapi.gateway.com/api/getfile?filename=abc.jpeg
```

ご利用の際は、Acceptヘッダに application/octet-stream MIMEタイプを指定する必要があります。  

```text
Allow: application/octet-stream
```

# Dynamoテーブル

本サービス開始時にユーザーのAWSアカウントIDに対して、専用テーブルに対するアクセス権限を付与したARNを配布します。
ユーザーにはテーブルに対する全てのAction権限が付与されます。
```json
"Effect": "Allow",
"Action": [
"dynamodb:BatchGetItem",
"dynamodb:BatchWriteItem",
"dynamodb:DeleteItem",
"dynamodb:GetItem",
"dynamodb:PutItem",
"dynamodb:Query",
"dynamodb:UpdateItem"
],
```

## アクセス方法

Dynamoテーブルへアクセス可能にするロール権限に対してポリシーを付与させます。
ポリシーには以下を定義します。

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "【MEEQサービスから配布されたDynamoアクセス用のARN】"
  }
}
```
同ロール権限を設定したAWSアカウントIDをお伝え下さい。  
Dynamoテーブルのアクセスロールの信頼されたエンティティにユーザーのAWSアカウントIDを追加致します。


## QuickSight利用の提案

Amazon QuickSightを利用することで、DynamoDBに格納したデータ一覧を各種グラフ表示することが可能です。  
ユーザーへのQuickSightへのデータアクセス権限付与をサービスオプションとして提供しています。
ご利用になりたい方はお問い合わせください。

QuickSightについては下記を参照ください。  
https://aws.amazon.com/jp/quicksight/

利用事例の紹介  
https://aws.amazon.com/jp/blogs/news/quicksight-dashboard-analysis-retail/

# S3バケット

本サービス開始時にS3バケットを払い出します。
ユーザーのAWSアカウントIDに対してGet, Put, Delete権限を付与いたします。  
AWSアカウントIDを弊社にご連携ください。

## アクセス方法

aws clientコマンドを利用してS3バケットにアクセスする事が可能です。  
また、ご希望に応じて、S3バケットのレプリケーション先をユーザーの指定したS3バケットにすることも可能です。

# インターネットアクセス用API

インターネットからDynamoDBに格納したデータにアクセスするためのREST APIを提供致します。  
契約時に払い出したAPIライセンスキーを利用してAPIをご利用頂けます。

APIライセンスキーは、x-api-keyヘッダに指定してください。

## getDataBySimKey

Dynamoテーブルのパーティションキーを使用してデータ取得を行うAPI。  
getTimestamp, leTimestampパラメータをオプションで設定することで、任意の範囲でのデータ取得が可能です。  
DynamoDBの特性上、1クエリで取得可能なデータサイズが1Mbyteまでのため、それを超えるデータ取得の場合、
レスポンスにlastEvaluatedKeyパラメータを付与したJsonオブジェクトを返却します。  
利用者は、返却されたlastEvaluatedKeyパラメータを利用し、再度リクエスト情報にexclusiveStartKeyとして
パラメータ設定をすることで、続きのデータを取得することが出来ます。  

### URL

GET /getDataBySimKey

### リクエスト
Json

|パラメータ名|M/O|タイプ|説明|値の例|
|:---|:---:|:---:|:---|:---|
|key|M|String|Dynamoテーブルのパーティションキー名を指定する。|"key":"tel"|
|value|M|String|取得対象の値を指定する。|"value":"08012345678"|
|geTimestamp|O|Number|取得対象の開始時刻をタイムスタンプで指定する。|"geTimestamp":1629199059758|
|leTimestamp|O|Number|検索対象の終了時刻をタイムスタンプで指定する。|"leTimestamp":1629874511223|
|exclusiveStartKey|O|Map|検索結果にlastEvaluatedKeyパラメータが含まれていて、途中からデータを取得する際に指定する<br>lastEvaluatedKeyの内容をそのまま指定する。|"exclusiveStartKey": {<br>    "createtime": 1629285924711,<br>    "tel": "08012345678"<br>}|

### レスポンス
Json

|パラメータ名|M/O|タイプ|説明|
|:---|:---:|:---:|:---|
|record|O|Array|Dynamoテーブルから取得したレコード情報を同配列で返却する|
|lastEvaluatedKey|O|Map|データが取り切れなかった場合に設定されるパラメータ。<br>次のリクエストにexclusiveStartKeyパラメータとして受信内容を指定すると続きからデータを取得することが出来る。|

## getDataByDeviceData

Dynamoテーブルに登録しているdeviceData配下のデータを利用してデータ取得を行うAPI。  
getTimestamp, leTimestampパラメータをオプションで設定することで、任意の範囲でのデータ取得が可能です。  
DynamoDBの特性上、1スキャンで取得可能なデータサイズが1Mbyteまでのため、それを超えるデータ取得の場合、
レスポンスにlastEvaluatedKeyパラメータを付与したJsonオブジェクトを返却します。  
利用者は、返却されたlastEvaluatedKeyパラメータを利用し、再度リクエスト情報にexclusiveStartKeyとして
パラメータ設定をすることで、続きのデータを取得することが出来ます。

### URL

GET /getDataByDeviceData  

### リクエスト
Json

|パラメータ名|M/O|タイプ|説明|値の例|
|:---|:---:|:---:|:---|:---|
|key|M|String|DynamoテーブルのdeviceData配下のパラメータ名を指定する。|"key":"deviceName"|
|value|M|String|取得対象の値を指定する。|"value":"RX-0"|
|geTimestamp|O|Number|取得対象の開始時刻をタイムスタンプで指定する。|"geTimestamp":1629199059758|
|leTimestamp|O|Number|検索対象の終了時刻をタイムスタンプで指定する。|"leTimestamp":1629874511223|
|exclusiveStartKey|O|Map|検索結果にlastEvaluatedKeyパラメータが含まれていて、途中からデータを取得する際に指定する<br>lastEvaluatedKeyの内容をそのまま指定する。|"exclusiveStartKey": {<br>    "createtime": 1629285924711,<br>    "tel": "08012345678"<br>}|

### レスポンス
Json

|パラメータ名|M/O|タイプ|説明|
|:---|:---:|:---:|:---|
|record|O|Array|Dynamoテーブルから取得したレコード情報を同配列で返却する|
|lastEvaluatedKey|O|Map|データが取り切れなかった場合に設定されるパラメータ。<br>次のリクエストにexclusiveStartKeyパラメータとして受信内容を指定すると続きからデータを取得することが出来る。  

