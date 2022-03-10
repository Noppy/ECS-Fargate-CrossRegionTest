# ECS-Fargate-CrossRegionTest


# 作成環境
## (1) 構成概要図
### (1)-(a) 全体構成


# 作成手順
## (1)事前設定
### (1)-(a) 作業環境の準備
下記を準備します。
* bashが利用可能な環境(LinuxやMacの環境)
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定

### (1)-(b) gitのclone
```shell
git clone https://github.com/Noppy/ECS-Fargate-CrossRegionTest.git
cd ECS-Fargate-CrossRegionTest
```

### (1)-(c) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export COMPUTE_PROFILE=<Fargaeを動かすComputeAccount用のプロファイルを指定>
export CONFMGR_PROFILE=<ECRを設置する構成管理Account用のプロファイルを指定>
export MAIN_REGION="ap-northeast-3"
export ECR_REGION="ap-southeast-2"

#プロファイルの動作テスト
#COMPUTE_PROFILE
aws --profile ${COMPUTE_PROFILE} sts get-caller-identity

#CONFMGR_PROFILE
aws --profile ${CONFMGR_PROFILE} sts get-caller-identity
```

## (2)Network準備
### (2)-(a) VPC作成
```shell
# ComputeVPC
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-ComputeVPC \
        --template-body "file://./src/vpc-2az-4subnets.yaml" \
        --parameters "file://./src/ComputeVPC.json" \
        --capabilities CAPABILITY_IAM ;

# EcrVPC
aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-EcrVPC \
        --template-body "file://./src/vpc-2az-2subnets.yaml" \
        --parameters "file://./src/EcrVPC.json" \
        --capabilities CAPABILITY_IAM ;

# DevVPC
aws --profile ${CONFMGR_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-DevVPC \
        --template-body "file://./src/vpc-2az-2subnets.yaml" \
        --parameters "file://./src/DevVPC.json" \
        --capabilities CAPABILITY_IAM ;
```

### (2)-(b) VPC Peering接続
ComputeVPCとEcrVPCをVPC Peeringで接続します。
```shell
#情報取得
EcrVpcId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-EcrVPC \
    --query 'Stacks[].Outputs[?OutputKey==`VpcId`].[OutputValue]')
EcrVpcCider=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-EcrVPC \
    --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')
echo "EcrVpcId    = ${EcrVpcId}
EcrVpcCider = ${EcrVpcCider}"

#VPC Peering作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "EcrRegion",
    "ParameterValue": "'"${ECR_REGION}"'"
  },
  {
    "ParameterKey": "EcrVpcId",
    "ParameterValue": "'"${EcrVpcId}"'"
  },
  {
    "ParameterKey": "EcrVpcCider",
    "ParameterValue": "'"${EcrVpcCider}"'"
  }
]'

aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-VpcPeering \
        --template-body "file://./src/VpcPeering.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}";
```
EcrVpcにVPC Peeringのルートを追加します。
```shell
#情報取得
ComputeVPCCidr=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-ComputeVPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')
EcrRouteTableId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-EcrVPC \
    --query 'Stacks[].Outputs[?OutputKey==`SubnetRouteTableId`].[OutputValue]')
VpcPeerId=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-VpcPeering \
        --query 'Stacks[].Outputs[?OutputKey==`VpcPeeringId`].[OutputValue]')
#確認
echo "
ComputeVPCCidr  = ${ComputeVPCCidr}
EcrRouteTableId = ${EcrRouteTableId}
VpcPeerId       = ${VpcPeerId}
"

#ECR VPCのルートテーブルにVPC Peeringのルート追加
aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} \
    ec2 create-route \
        --route-table-id ${EcrRouteTableId} \
        --destination-cidr-block ${ComputeVPCCidr} \
        --vpc-peering-connection-id ${VpcPeerId}
```

### (2)-(C) VPCエンドポイント作成
```shell
# Deploy VPCEs in the ComputeVPC.
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-ComputeVPC-VPCE \
        --template-body "file://./src/VPCE_ComputeVPC.yaml" ;

# Deploy VPCEs in the EcrVPC.
aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-EcrVPC-VPCE \
        --template-body "file://./src/VPCE_EcrVPC.yaml" ;
```
### (2)-(d) ECR VPCのVPCエンドポイント用のPrivateHostedZone作成
#### (i) VPCエンドポイント(ECR)
```shell
#情報取得
#EcrApiVpce
EcrApiVpceId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-EcrVPC-VPCE \
        --query 'Stacks[].Outputs[?OutputKey==`EcrApiEndpointId`].[OutputValue]')
EcrApiVpceDnsName=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    ec2 describe-vpc-endpoints \
        --filters "Name=vpc-endpoint-id,Values=${EcrApiVpceId}" \
    --query 'VpcEndpoints[].DnsEntries[0].DnsName')
EcrApiVpceDnsHostedZoneId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    ec2 describe-vpc-endpoints \
        --filters "Name=vpc-endpoint-id,Values=${EcrApiVpceId}" \
    --query 'VpcEndpoints[].DnsEntries[0].HostedZoneId')
#EcrDkrVpce
EcrDkrVpceId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-EcrVPC-VPCE \
        --query 'Stacks[].Outputs[?OutputKey==`EcrDkrEndpointId`].[OutputValue]')
EcrDkrVpceDnsName=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    ec2 describe-vpc-endpoints \
        --filters "Name=vpc-endpoint-id,Values=${EcrDkrVpceId}" \
    --query 'VpcEndpoints[].DnsEntries[0].DnsName')
EcrDkrVpceDnsHostedZoneId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    ec2 describe-vpc-endpoints \
        --filters "Name=vpc-endpoint-id,Values=${EcrDkrVpceId}" \
    --query 'VpcEndpoints[].DnsEntries[0].HostedZoneId')

ConfMgrAccountID=$(aws --profile ${CONFMGR_PROFILE} --output text sts get-caller-identity --query 'Account')

#取得情報確認
echo "
EcrApiVpceDnsName         = ${EcrApiVpceDnsName}
EcrApiVpceDnsHostedZoneId = ${EcrApiVpceDnsHostedZoneId}
EcrDkrVpceDnsName         = ${EcrDkrVpceDnsName}
EcrDkrVpceDnsHostedZoneId = ${EcrDkrVpceDnsHostedZoneId}
ConfMgrAccountID          = ${ConfMgrAccountID}
"
```

```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "EcrRegion",
    "ParameterValue": "'"${ECR_REGION}"'"
  },
  {
    "ParameterKey": "EcrApiVpceDnsName",
    "ParameterValue": "'"${EcrApiVpceDnsName}"'"
  },
  {
    "ParameterKey": "EcrApiVpceDnsHostedZoneId",
    "ParameterValue": "'"${EcrApiVpceDnsHostedZoneId}"'"
  },
  {
    "ParameterKey": "EcrDkrVpceDnsName",
    "ParameterValue": "'"${EcrDkrVpceDnsName}"'"
  },
  {
    "ParameterKey": "EcrDkrVpceDnsHostedZoneId",
    "ParameterValue": "'"${EcrDkrVpceDnsHostedZoneId}"'"
  },
  {
    "ParameterKey": "ConfMgrAccountID",
    "ParameterValue": "'"${ConfMgrAccountID}"'"
  }
]'

aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-PHZ-Ecr \
        --template-body "file://./src/PHZ_ForEcrVpce.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" ;
```

#### (ii) VPCエンドポイント(S3)

```shell
#情報取得
S3EndpointInterfaceId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-EcrVPC-VPCE \
        --query 'Stacks[].Outputs[?OutputKey==`S3EndpointInterfaceId`].[OutputValue]')
S3VpceDnsName=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    ec2 describe-vpc-endpoints \
        --filters "Name=vpc-endpoint-id,Values=${S3EndpointInterfaceId}" \
    --query 'VpcEndpoints[].DnsEntries[0].DnsName')
S3VpceDnsHostedZoneId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    ec2 describe-vpc-endpoints \
        --filters "Name=vpc-endpoint-id,Values=${S3EndpointInterfaceId}" \
    --query 'VpcEndpoints[].DnsEntries[0].HostedZoneId')


#取得情報確認
echo "
S3EndpointInterfaceId = ${S3EndpointInterfaceId}
S3VpceDnsHostedZoneId = ${S3VpceDnsHostedZoneId}
S3VpceDnsName         = ${S3VpceDnsName}
"
```

```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "EcrRegion",
    "ParameterValue": "'"${ECR_REGION}"'"
  },
  {
    "ParameterKey": "S3VpceDnsName",
    "ParameterValue": "'"${S3VpceDnsName}"'"
  },
  {
    "ParameterKey": "S3VpceDnsHostedZoneId",
    "ParameterValue": "'"${S3VpceDnsHostedZoneId}"'"
  }
]'

aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-PHZ-S3 \
        --template-body "file://./src/PHZ_ForS3Vpce.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" ;
```

## (3) ECRとDockerイメージの準備
### (i) DockerRepository準備
```shell
# 情報取得
ComputeAccountID=$(aws --profile ${COMPUTE_PROFILE} --output text sts get-caller-identity --query 'Account')

#DockerRepository作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "ComputeAccountID",
    "ParameterValue": "'"${ComputeAccountID}"'"
  }
]'
aws --profile ${CONFMGR_PROFILE} --region ${ECR_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-EcrRepository \
        --template-body "file://./src/EcrRepository.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" ;
```

### (ii) DockerBuild環境の準備
```shell
#最新のAmazon Linux2のAMI IDを取得
AL2_AMIID=$(aws --profile ${CONFMGR_PROFILE} --region ${MAIN_REGION} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' );
echo "
AL2_AMIID = ${AL2_AMIID}"

#インスタンス作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  }
]'

aws --profile ${CONFMGR_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-DockerBuilder \
        --template-body "file://./src/DockerBuildInstance.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" \
        --capabilities CAPABILITY_IAM ;
```

### (iii) デモ用のdockerイメージを作成
DockerBuildインスタンスで、デモ用のphpのwebサーバのコンテナを作成し、ECRリポジトリに登録します。
- 手順
    - Systems Manager - Session ManagerでDockerBuildインスタンスのOSにログイン
    ```shell
    aws --profile ${CONFMGR_PROFILE} --region ${MAIN_REGION} ssm start-session --target "<DockerBuildインスタンスのID>"
    ```
    - 下記コマンドを実行し環境を準備し、dokcerイメージ作成 & ECR登録を行う

#### DockerBuildインスタンス環境準備
```shell
#ec2-userにスイッチ
sudo -u ec2-user -i

# Setup AWS CLI
REGION="<$ECR_REGION のリージョンコードを手動設定>"
aws configure set region ${REGION}
aws configure set output json

#動作テスト(作成したECRレポジトリがリストに表示されることを確認)
aws ecr describe-repositories

#dockerテスト(下記コマンドでサーバ情報が参照できることを確認)
docker info
```
#### Dockerイメージ作成
```shell
#コンテナイメージ用のディレクトリを作成し移動
mkdir httpd-container
cd httpd-container

#データ用フォルダを作成
mkdir src

#dockerコンテナの定義ファイルを作成
cat > Dockerfile << EOL
# setting base image
FROM php:8.1-apache

RUN set -x && \
    apt-get update 

COPY src/ /var/www/html/
EOL

#
cat > src/index.php << EOL
<html>
  <head>
    <title>PHP Sample</title>
  </head>
  <body>
    <?php echo gethostname(); ?>
  </body>
</html>
EOL

#Docker build
docker build -t httpd-sample:ver01 .
docker images

#コンテナの動作確認
docker run -d -p 8080:80 httpd-sample:ver01
docker ps #コンテナが稼働していることを確認

#接続確認
# <title>PHP Sample</title>という文字が表示されたら成功！！
curl http://localhost:8080
```
#### ECR登録
```shell
REPO_URL=$( aws --output text \
    ecr describe-repositories \
        --repository-names fargatepoc-repo \
    --query 'repositories[].repositoryUri' ) ;
echo "
REPO_URL = ${REPO_URL}
"
# ECR登録用のタグを作成
docker tag httpd-sample:ver01 ${REPO_URL}:latest
docker images #作成したtagが表示されていることを確認

#ECRログイン
#"Login Succeeded"と表示されることを確認
aws ecr get-login-password | docker login --username AWS --password-stdin ${REPO_URL}

#イメージのpush
docker push ${REPO_URL}:latest

#ECR上のレポジトリ確認
aws ecr list-images --repository-name fargatepoc-repo
```
#### ログアウト
```shell
exit  #ec2-userからの戻る
exit  #SSMからのログアウト
```

## (4) EC２インスタンス単独でのdocker pullテスト
### (i) FowardProxy設置(yumアップデート用)
```shell
# Amazon Linux2のyumリポジトリのみ許可
AllowUrlsList="amazonlinux-2-repos-${MAIN_REGION}.s3.dualstack.${MAIN_REGION}.amazonaws.com,amazonlinux-2-repos-${MAIN_REGION}.s3.${MAIN_REGION}.amazonaws.com,"

#最新のAmazon Linux2のAMI IDを取得
AL2_AMIID=$(aws --profile ${CONFMGR_PROFILE} --region ${MAIN_REGION} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' );
#ComputeVpcのCIDR取得
ComputeVPCCidr=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-ComputeVPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')

echo "
AllowUrlsList = ${AllowUrlsList}
AL2_AMIID = ${AL2_AMIID}
ComputeVPCCidr = ${ComputeVPCCidr}
"

# Set JSON
CFN_STACK_PARAMETERS='
[
    {
        "ParameterKey": "AllowUrlsList",
        "ParameterValue": "'"${AllowUrlsList}"'"
    },
    {
        "ParameterKey": "ProxyAmiId",
        "ParameterValue": "'"${AL2_AMIID}"'"
    },
    {
        "ParameterKey": "AllowCidrBlockForProxy",
        "ParameterValue": "'"${ComputeVPCCidr}"'"
    }
]'

# Deploy Foward Proxy in the ComputeVPC.
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-FowardPorxy \
        --template-body "file://./src/forward_proxy.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" \
        --capabilities CAPABILITY_IAM ;
```
### (ii) ECR PULLテスト用インスタンス設置
#### インスタンスの作成
```shell
#最新のAmazon Linux2のAMI IDを取得
AL2_AMIID=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' );
FowardProxyDns=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text\
    cloudformation describe-stacks \
        --stack-name FargetePoC-FowardPorxy \
    --query 'Stacks[].Outputs[?OutputKey==`LoadBalancerDns`].[OutputValue]'
)
echo "
AL2_AMIID = ${AL2_AMIID}
FowardProxyDns = ${FowardProxyDns}
"

#インスタンス作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  },
  {
    "ParameterKey": "FowardProxyDns",
    "ParameterValue": "'"${FowardProxyDns}"'"
  }
]'

#インスタンス作成
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-DockerTestInstance \
        --template-body "file://./src/DockerTestInstance.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" \
        --capabilities CAPABILITY_IAM ;
```
#### 情報の取得
この後の作業に必要となる情報を取得する
```shell
#テスト用インスタンスのインスタンスID確認
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation describe-stacks \
        --stack-name FargetePoC-DockerTestInstance \
    --query 'Stacks[].Outputs[?OutputKey==`InstanceId`].[OutputValue]'

#レポジトリURLの取得
aws --profile ${CONFMGR_PROFILE} --region ${ECR_REGION} \
    ecr describe-repositories \
        --repository-names fargatepoc-repo \
    --query 'repositories[].repositoryUri' ;
```
#### インスタンスのセットアップ(docker & aws cli)
 - Systems Manager - Session Managerで確認したインスタンスIDのOSにログイン
     ```shell
    aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} ssm start-session --target "<DockerテストインスタンスのID>"
    ```
```shell
#ec2-userにスイッチ
sudo -u ec2-user -i

#Dockerセットアップ
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user

#Dockerのテスト(ec2-userで再ログインしてdockerサーバ情報が取得できるか確認)
exit
sudo -u ec2-user -i
docker info

# Setup AWS CLI
REGION="<$ECR_REGION のリージョンコードを手動設定>"
aws configure set region ${REGION}
aws configure set output json
```
#### Docker pullテスト

```shell
REPO_URL="<予め確認したfargatepoc-repoのURLを設定>"

#ECRログイン
#"Login Succeeded"と表示されることを確認
aws ecr get-login-password | docker login --username AWS --password-stdin ${REPO_URL}

#DockerイメージのPull
docker pull "${REPO_URL}:latest"

#pullしたDockerイメージの確認
docker images
```

## (5) Faragetでのテスト
```shell

```