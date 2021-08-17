# MEEQデータプラットフォーム IoTストレージ

REST for Jsonを利用したデータ収集・保存サービスです。  
ユーザーは、サービス開始時に指定されたURLに対してHTTPSリクエストを送信するだけで、専用のDynamoテーブルにデータが保存されます。

MEEQ閉域SIMを利用したサービスのため、同サービスではインターネットを介しません。  
セキュアにデータ送信が可能なため、端末側のセキュリティを考慮する必要なくご利用頂けます。

# データ送信/保存

サービス開始時に指定されたURLに対してHTTPSリクエストを送信してください。  
Content-Typeには、application/jsonを指定し、ボディにはJson形式のデータを設定してください。

本サービスでは、受信データに対し、データ送信端末の電話番号とデータ受信時刻のTimestamp情報を自動的に付与します。  
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

ユーザーのAWSアカウントのDynamoテーブルへアクセスしたいロール権限に対してポリシーを付与させます。
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


# QuickSight利用の提案

Amazon QuickSightを利用することで、DynamoDBに格納したデータ一覧を各種グラフ表示することが可能です。  
ユーザーへのQuickSightへのデータアクセス権限付与をサービスオプションとして提供しています。
ご利用になりたい方はお問い合わせください。

QuickSightについては下記を参照ください。
https://aws.amazon.com/jp/quicksight/

利用事例の紹介
https://aws.amazon.com/jp/blogs/news/quicksight-dashboard-analysis-retail/
