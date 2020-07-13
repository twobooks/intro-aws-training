# AWS のシークレットキーの作成
AWSコンソールから

# AWS CLIのインストール
PS aws --version  
バージョン確認

# AWS Access Key等の設定
PS aws configure
下記の設定を行う
AWS Access Key ID
AWS Secret Access Key
Default region name # ap-northeast-1 (東京リージョン)を指定するの
Default output format # JSONを指定
```bash
PS cat ~/.aws/credentials
[default]
aws_access_key_id = XXXXXXXXXXXXXXXXXX
aws_secret_access_key = YYYYYYYYYYYYYYYYYYY

PS cat ~/.aws/config
[profile default]
region = ap-northeast-1
output = json
```

# S3バケットの作成
```bash
PS $bucketName="mybucket-twobooks20200712"
PS echo $bucketName
mybucket-twobooks20200712
PS echo "s3://${bucketName}"
s3://mybucket-twobooks20200712
PS aws s3 mb "s3://${bucketName}"
make_bucket: mybucket-twobooks20200712
PS aws s3 ls
2020-07-12 11:44:55 mybucket-twobooks20200712
```

# S3 bucket へのアップロード
```bash
PS echo "Hello world!" > hello_world.txt
PS aws s3 cp hello_world.txt "s3://${bucketName}/hello_world.txt"
```
# S3 bucket 内の確認
```bash
aws s3 ls "s3://${bucketName}" --human-readable
```

# S3 bucket の削除
デフォルトでは，バケットは空でないと削除できない．空でないバケットを強制的に削除するには --force のオプションを付ける．
```bash
aws s3 rb "s3://${bucketName}" --force
```

# CloudFormation
AWSでのリソースを管理するための仕組み.  
Infrastructure as Code (IaC)．  
CloudFormation を記述するには，基本的に JSON (JavaScript Object Notation) と呼ばれるフォーマットを使う．

# AWS CDK
CloudFormation 記述が複雑.  
基本的に変数やクラスといった概念が使えない　(厳密には，変数に相当するような機能は存在する)．  
また，記述の多くの部分は繰り返しが多く，自動化できる部分も多い．  
解決してくれるのが， AWS Cloud Development Kit (CDK) ．  
CDKは Python などのプログラミング言語を使って CloudFormation を自動的に生成してくれるツール．  
CDKを使うことで，CloudFormation に相当するクラウドリソースの記述を，より親しみのあるプログラミング言語を使って行うことができる．かつ，典型的なリソース操作に関してはパラメータの多くの部分を自動で決定してくれるので，記述しなければならない量もかなり削減される．

# SSH コマンドの基本的な使い方
```bash
$ ssh <user name>@<host name>
```
user name は接続する先のユーザー名．  
host name はアクセスする先のサーバーのIPアドレスやDNSによるホストネームが入る． 
SSH コマンドでは，ログインのために使用する秘密鍵ファイルを -i もしくは --identity_file のオプションで指定することができる．
```bash
# 基本書式
$ ssh -i Ec2SecretKey.pem <user name>@<host name>
# .sshフォルダからのSSH
C:\Users\2books\.ssh>ssh -i "HirakeGoma.pem" ec2-user@18.183.140.117
# Linuxと同じ形でも使える
$ ssh -i ~/.ssh/HirakeGoma.pem ec2-user@18.183.140.117
```

# chmod 400 をWindowsで
https://qiita.com/sumomomomo/items/28d54e35bfa5bc524cf5
PowerShellで下記
```bash
$path = ".ssh\SSH鍵名.pem"
icacls.exe $path /reset
icacls.exe $path /GRANT:R "$($env:USERNAME):(R)"
icacls.exe $path /inheritance:r
```

# Amazon EC2 キーペアの作成、表示、削除
https://docs.aws.amazon.com/ja_jp/cli/latest/userguide/cli-services-ec2-keypairs.html
あまりPowershell使いたくないからコンソールで作ったほうが早いかも（pemファイルのダウンロードまでされる）
```bash
PS C:\>aws ec2 create-key-pair --key-name MyKeyPair --query 'KeyMaterial' --output text | out-file -encoding ascii -filepath MyKeyPair.pem

# キーの削除
$ aws ec2 delete-key-pair --key-name "HirakeGoma"
# ローカルのキーも削除しとく
```

# VPC(Virtual Private Cloud)
VPCはAWS上にプライベートな仮想ネットワーク環境を構築するツール．  
複数のサーバーを連動させて計算を行う場合に互いのアドレスなどを管理する必要があり，そのような場合にVPCは有用．  
EC2インスタンスは必ずVPCの中に配置されなければならない，という制約がある.ハンズオンでもミニマルなVPCを構成している．
```python
from aws_cdk import (core, aws_ec2 as ec2,)
import os

class MyFirstEc2(core.Stack):

    def __init__(self, scope: core.App, name: str, key_name: str, **kwargs) -> None:
        super().__init__(scope, name, **kwargs)

        # <1>
        vpc = ec2.Vpc(
            self, "MyFirstEc2-Vpc",
            max_azs=1, #avaialibility zoneの設定．障害などを気にしないなら1でOK
            cidr="10.10.0.0/23", # VPC内のIPv4のレンジを指定．
            # 10.10.0.0/23 は10.10.0.0 ~ 10.10.1.255の512個のアドレス範囲．
            subnet_configuration=[ #サブネットの設定．
            # priavte subnet と public subnet の二種類ある.
                ec2.SubnetConfiguration(
                    name="public",
                    subnet_type=ec2.SubnetType.PUBLIC,
                )
            ],
            nat_gateways=0,
            # これを0にしておかないと，NAT Gateway の利用料金が発生してしまうので，注意！
        )
```

# Security Group
EC2インスタンスに付与することのできる仮想ファイアーウォール  
特定のIPアドレスから来た接続を許したり　(インバウンド・トラフィックの制限) ，逆に特定のIPアドレスへのアクセスを禁止したり (アウトバウンド・トラフィックの制限) することができる．  
```python
from aws_cdk import (core, aws_ec2 as ec2,)
import os

class MyFirstEc2(core.Stack):

    def __init__(self, scope: core.App, name: str, key_name: str, **kwargs) -> None:
        super().__init__(scope, name, **kwargs)

        # <2>
        sg = ec2.SecurityGroup(
            # EC2にログインしたのち，ネットからプログラムなどをダウンロードできるよう，
            # allow_all_outbound=True のパラメータを設定している．
            self, "MyFirstEc2Vpc-Sg",
            vpc=vpc,
            allow_all_outbound=True,
        )
        sg.add_ingress_rule(
            # SSHによる外部からの接続を許容
            # すべてのIPv4アドレスからのポート22番へのアクセスを許容している．
            peer=ec2.Peer.any_ipv4(),
            connection=ec2.Port.tcp(22),
        )
```

# EC2
t2.micro であれば月に750時間までは無料で利用することができる．  
t2.micro の $0.0116 / hour という金額は，on-demandインスタンスというタイプを選択した場合の価格である．  
Spot instance と呼ばれるインスタンスも存在する．Spot instance は，AWSのデータセンターの負荷が増えた場合，AWSの判断により強制シャットダウンされる可能性があるが，その分大幅に安い料金設定になっている．  
科学計算で，コストを削減する目的で，このSpot Instanceを使う事例も報告されている
```python
from aws_cdk import (core, aws_ec2 as ec2,)
import os

class MyFirstEc2(core.Stack):

    def __init__(self, scope: core.App, name: str, key_name: str, **kwargs) -> None:
        super().__init__(scope, name, **kwargs)

                # <3>
        host = ec2.Instance(
            self, "MyFirstEc2Instance",
            instance_type=ec2.InstanceType("t2.micro"), # t2.micro を設定
            machine_image=ec2.MachineImage.latest_amazon_linux(),# OSはAmazon Linux 
            vpc=vpc,
            vpc_subnets=ec2.SubnetSelection(subnet_type=ec2.SubnetType.PUBLIC),
            security_group=sg,
            key_name=key_name
        )
```

# CDKのdeployまで
```bash
$ md cdkdir
$ cd cdkdir
$ cdk init app --language=python
$ .env\Scripts\activate.bat
# ここから仮想環境
(.env)$ pip install -r requirements.txt
(.env)$ cdk deploy -c key_name="HirakeGoma"  # deploy完了　
# stackの削除
$ cdk destroy
```
