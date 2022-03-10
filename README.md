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
aws --profile ${CONFMGR_PROFILE} --region ${ECR_REGION} \
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
### (2)-(d) ECR VPCのECRエンドポイント用のPrivateHostedZone作成
設定に必要なVPCエンドポイントの情報を取得します。
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
#取得情報確認
echo "
EcrApiVpceDnsName         = ${EcrApiVpceDnsName}
EcrApiVpceDnsHostedZoneId = ${EcrApiVpceDnsHostedZoneId}
EcrDkrVpceDnsName         = ${EcrDkrVpceDnsName}
EcrDkrVpceDnsHostedZoneId = ${EcrDkrVpceDnsHostedZoneId}"
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
  }

]'

aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-PHZ \
        --template-body "file://./src/PHZ_ForEcrVpce.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" ;
```












