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
        --template-body "file://./src/vpc-2az-6subnets.yaml" \
        --parameters "file://./src/ComputeVPC.json" \
        --capabilities CAPABILITY_IAM ;

# EcrVPC
aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-EcrVPC \
        --template-body "file://./src/vpc-2az-4subnets.yaml" \
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

### (2)-(b) TransitGateway接続
ComputeVPCとEcrVPCをTransitGatewayで接続します。
#### (i) TGW1の作成とComputeVPCのアタッチ
スタック作成に必要な情報を取得します。
```shell
#情報取得
EcrAccountID=$(aws --profile ${CONFMGR_PROFILE} --output text sts get-caller-identity --query 'Account')
EcrVPCCidr=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-EcrVPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')

EcrRegionS3PrefixList=($(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    ec2 describe-prefix-lists \
        --filters "Name=prefix-list-name,Values=com.amazonaws.${ECR_REGION}.s3" \
    --query 'PrefixLists[0].Cidrs' ))
#取得情報の確認
echo "EcrAccountID = ${EcrAccountID}
EcrVPCCidr   = ${EcrVPCCidr}
EcrRegionS3PrefixList = ${EcrRegionS3PrefixList[@]}"
```
取得した情報からCloudFormationのスタックに渡すパラメータファイルを作成し、そのパラメータファイルを利用しTGW1のスタックを作成します。
```shell
# TGW1のStack作成時のパラメータファイル(JSON)作成
echo '[' > Tgw1.json
for i in $(seq 1 ${#EcrRegionS3PrefixList[@]})
do
    echo -e '  {\n    "ParameterKey": "EcrRegionS3Prefix'"${i}"'",\n    "ParameterValue": "'"${EcrRegionS3PrefixList[$i]}"'"\n  },' >> Tgw1.json
done
echo '  {
    "ParameterKey": "EcrAwsAccountId",
    "ParameterValue": "'"${EcrAccountID}"'"
  },
  {
    "ParameterKey": "EcrVPCCidr",
    "ParameterValue": "'"${EcrVPCCidr}"'"
  }
]' >> Tgw1.json

# TGW1のCloudFormationスタック作成
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-TGW1 \
        --template-body "file://./src/TGW1_ComputeAccount.yaml" \
        --parameters "file://./Tgw1.json";
```

#### (ii) TGW2の作成・EcrVPCのアタッチ・TGW間接続
スタック作成に必要な情報を取得します。
```shell
#情報取得
Tgw1Id=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-TGW1 \
        --query 'Stacks[].Outputs[?OutputKey==`Tgw1Id`].[OutputValue]')
ComputeVPCCidr=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-ComputeVPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')
#取得情報の確認
echo "Tgw1Id         = ${Tgw1Id}
ComputeVPCCidr = ${ComputeVPCCidr}"
```
TGW2作成
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "Tgw1Region",
    "ParameterValue": "'"${MAIN_REGION}"'"
  },
  {
    "ParameterKey": "Tgw1Id",
    "ParameterValue": "'"${Tgw1Id}"'"
  },
  {
    "ParameterKey": "ComputeVPCCidr",
    "ParameterValue": "'"${ComputeVPCCidr}"'"
  }
]'

#スタックの作製
aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} \
    cloudformation create-stack \
        --stack-name FargetePoC-TGW2 \
        --template-body "file://./src/TGW2_ComputeAccount.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}";
```
TGW1側でインター・リージョン・ピアリングを承諾し、ピアリング作成完了までWaitします。
```shell
#InterRegionPeeringのID取得
InterRegionPeeringId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-TGW2 \
        --query 'Stacks[].Outputs[?OutputKey==`TgwInterRegionPeeringAttachmentId`].[OutputValue]')

#InterRegionPeeringの許可
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    ec2 accept-transit-gateway-peering-attachment \
        --transit-gateway-attachment-id "${InterRegionPeeringId}"

#InterRegionPeering作成完了までWait
while true
do
    STATUS=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
        ec2 describe-transit-gateway-peering-attachments \
            --filters "Name=transit-gateway-attachment-id,Values=${InterRegionPeeringId}" \
        --query 'TransitGatewayPeeringAttachments[0].Status.Code')
    echo "$(date '+%Y/%m/%d %H:%M:%S') Status = ${STATUS}"
    if [ "${STATUS}" = "available" ]; then break; fi
    sleep 5
done

#TGW1のルートテーブルへのInterRegionPeering関連付け
Tgw1RouteTableId=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-TGW1 \
        --query 'Stacks[].Outputs[?OutputKey==`Tgw1RouteTableId`].[OutputValue]')

aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    ec2 associate-transit-gateway-route-table \
        --transit-gateway-route-table-id ${Tgw1RouteTableId} \
        --transit-gateway-attachment-id ${InterRegionPeeringId}

#TGW2のルートテーブルへのInterRegionPeering関連付け
Tgw2RouteTableId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-TGW2 \
        --query 'Stacks[].Outputs[?OutputKey==`Tgw2RouteTableId`].[OutputValue]')

aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} \
    ec2 associate-transit-gateway-route-table \
        --transit-gateway-route-table-id ${Tgw2RouteTableId} \
        --transit-gateway-attachment-id ${InterRegionPeeringId}
```
TGWインター・リージョン・ピアリングに関するルーティング情報をTGWのルートテーブルに追加します。
```shell
#情報取得
Tgw1RouteTableId=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-TGW1 \
        --query 'Stacks[].Outputs[?OutputKey==`Tgw1RouteTableId`].[OutputValue]')
Tgw2RouteTableId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-TGW2 \
        --query 'Stacks[].Outputs[?OutputKey==`Tgw2RouteTableId`].[OutputValue]')
InterRegionPeeringId=$(aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-TGW2 \
        --query 'Stacks[].Outputs[?OutputKey==`TgwInterRegionPeeringAttachmentId`].[OutputValue]')
ComputeVPCCidr=$(aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} --output text \
    cloudformation describe-stacks \
        --stack-name FargetePoC-ComputeVPC \
        --query 'Stacks[].Outputs[?OutputKey==`VpcCidr`].[OutputValue]')
#取得情報の確認
echo "
Tgw1RouteTableId     = ${Tgw1RouteTableId}
Tgw2RouteTableId     = ${Tgw2RouteTableId}
InterRegionPeeringId = ${InterRegionPeeringId}
ComputeVPCCidr = ${ComputeVPCCidr}"

#TGW1のルートテーブルへの設定
aws --profile ${COMPUTE_PROFILE} --region ${MAIN_REGION} \
    ec2 create-transit-gateway-route \
        --transit-gateway-route-table-id ${Tgw1RouteTableId} \
        --destination-cidr-block "0.0.0.0/0" \
        --transit-gateway-attachment-id ${InterRegionPeeringId}

#TGW2のルートテーブルへの設定
aws --profile ${COMPUTE_PROFILE} --region ${ECR_REGION} \
    ec2 create-transit-gateway-route \
        --transit-gateway-route-table-id ${Tgw2RouteTableId} \
        --destination-cidr-block ${ComputeVPCCidr} \
        --transit-gateway-attachment-id ${InterRegionPeeringId}

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












