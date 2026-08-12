# Splunk Distributed Deployment on AWS

AWS上にSplunk Enterpriseの分散構成を構築するための、匿名化した公開用サンプルです。

## Contents

- [`cloudformation/`](cloudformation/): AWS基盤を作成するCloudFormationテンプレートとデプロイ手順

SSMによるインストール手順とSplunk設定例は、匿名化と検証が完了した段階で追加します。

## Important

- このリポジトリは学習・検証用です。本番環境の可用性、暗号化、バックアップ、監視、サイジングを保証しません。
- プレースホルダーを実環境の値へ置き換えてから使用してください。
- EC2、ALB、NAT Gateway、Route 53などの利用料金が発生します。
- Splunk Enterprise本体、ライセンス、秘密鍵、証明書原本は含みません。

