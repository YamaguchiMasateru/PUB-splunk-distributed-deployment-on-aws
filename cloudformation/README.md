# CloudFormation sample

Splunk Enterpriseの検証基盤を、親スタック1個と子スタック8個で作成するサンプルです。CloudFormationが作成するのはAWS基盤までで、Splunk Enterpriseのインストールやクラスタ設定は含みません。

## Security considerations

検証用テンプレートには、次の制約があります。デプロイ前に利用環境の要件に合わせて変更してください。

- 全EC2のEBS暗号化が無効です（[`ec2-shc.yaml`](ec2-shc.yaml)、[`ec2-private.yaml`](ec2-private.yaml)）。
- EC2でIMDSv2を必須にする`MetadataOptions.HttpTokens: required`を指定していません。
- Search Head EC2はパブリックIP付きです。Security Groupで直接のインバウンド接続は許可していませんが、private subnetへの配置も検討してください（[`ec2-shc.yaml`](ec2-shc.yaml)）。
- 管理用ALBは`internet-facing`です。接続元CIDRで制限していますが、管理画面をインターネット経由で公開する設計です（[`admin-alb.yaml`](admin-alb.yaml)）。
- `mgmt`、`cm`、`indexer-*`など、管理ノードの役割が分かる名前を公開DNSに登録します（[`route53.yaml`](route53.yaml)）。
- ALBのアクセスログ、AWS WAF、削除保護は設定していません。

## Templates

| ファイル | 内容 |
| --- | --- |
| `main.yaml` | 親スタック |
| `vpc-network.yaml` | VPC、サブネット、ルート、Internet Gateway、NAT Gateway |
| `security-groups.yaml` | ALBとSplunkノード用Security Group |
| `iam-roles.yaml` | EC2用IAMロールとインスタンスプロファイル |
| `ec2-shc.yaml` | Search Head用EC2 2台 |
| `ec2-private.yaml` | 管理サーバー、Cluster Manager、Indexer 3台 |
| `alb.yaml` | Search HeadとHEC向け公開用ALB |
| `admin-alb.yaml` | 管理用Splunk Web向けALB |
| `route53.yaml` | 公開Aliasレコードと内部DNSレコード |
| `parameters.example.json` | 公開用の入力例。プレースホルダーの置換が必要 |
| `parameters.json` | ローカル検証用の実パラメーター。`.gitignore`により非追跡 |

## Prerequisites

- AWS CLIと実行用プロファイル
- 子テンプレートを格納する非公開S3バケット
- ALB用のACM証明書
- Route 53公開ホストゾーン
- 7台分のAmazon Linux 2023 AMI ID
- 管理画面とHECへの接続を許可するCIDR

はじめに`parameters.example.json`を`parameters.json`へコピーします。次に、`parameters.json`内の`<...>`を実環境の値へ置き換えてください。`example.com`、`203.0.113.0/24`、`example-splunk-deployment-bucket`もサンプル値です。

```bash
cp cloudformation/parameters.example.json cloudformation/parameters.json
```

`parameters.json`には公開できない環境固有値が入るため、Gitの追跡対象にしません。

## Validate

```bash
for template in cloudformation/*.yaml; do
  echo "Validating: ${template}"
  aws cloudformation validate-template \
    --template-body "file://${template}" \
    --profile <AWS_PROFILE> \
    --region ap-northeast-1
done
```

## Upload nested templates

リポジトリのルートで実行します。S3のキーは、`parameters.json`の`TemplatePrefix`と一致させます。

```bash
aws s3 sync cloudformation/ \
  s3://<TEMPLATE_BUCKET>/Cfn_Template/ \
  --exclude "main.yaml" \
  --exclude "parameters.json" \
  --exclude "parameters.example.json" \
  --exclude "README.md" \
  --profile <AWS_PROFILE> \
  --region ap-northeast-1
```

## Create stack

次の操作はEC2、ALB、NAT Gatewayなどを作成し、料金が発生します。

```bash
aws cloudformation create-stack \
  --stack-name splunk-cluster \
  --template-body file://cloudformation/main.yaml \
  --parameters file://cloudformation/parameters.json \
  --capabilities CAPABILITY_NAMED_IAM \
  --profile <AWS_PROFILE> \
  --region ap-northeast-1
```

## Delete stack

```bash
aws cloudformation delete-stack \
  --stack-name splunk-cluster \
  --profile <AWS_PROFILE> \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name splunk-cluster \
  --profile <AWS_PROFILE> \
  --region ap-northeast-1
```

S3バケット、Route 53公開ホストゾーン、ACM証明書、指定したAMIはスタック外のリソースであり、スタック削除後も残ります。
