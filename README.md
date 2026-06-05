# Lab02 BT2 - CloudFormation, CodeBuild, Taskcat va CodePipeline

## 1. Gioi thieu

Day la bai thuc hanh trien khai lai ha tang cua Lab01 bang AWS CloudFormation, sau do tu dong hoa quy trinh kiem tra va deploy bang AWS CodePipeline.

Ha tang duoc tao gom:

- VPC.
- Public subnet va private subnet.
- Internet Gateway.
- NAT Gateway.
- Route tables cho public/private subnet.
- Security Groups.
- 2 EC2 instance: 1 public EC2 va 1 private EC2.

Phan CI/CD duoc tao gom:

- AWS CodeCommit lam source repository.
- AWS CodeBuild chay `cfn-lint` va Taskcat de kiem tra template.
- AWS CodePipeline gom 3 stage: `Source`, `Build`, `Deploy`.
- CloudFormation deploy stack ha tang tu template da duoc package.

## 2. Cau truc source code

```text
lab02-bt2/
|-- infrastructure.yaml
|-- pipeline.yaml
|-- buildspec.yml
|-- .taskcat.yml
|-- .cfnlintrc
|-- .gitignore
|-- README.md
|-- modules/
|   |-- vpc.yaml
|   |-- security-groups.yaml
|   `-- ec2.yaml
`-- parameters/
    `-- lab2-parameters.json
```

Trong do:

- `infrastructure.yaml` la root stack. File nay goi cac nested stack trong thu muc `modules/`.
- `modules/vpc.yaml` tao VPC, subnet, Internet Gateway, NAT Gateway va route table.
- `modules/security-groups.yaml` tao Security Group cho public EC2 va private EC2.
- `modules/ec2.yaml` tao public EC2 va private EC2.
- `pipeline.yaml` tao CodeBuild, CodePipeline, IAM roles, S3 artifact bucket va EventBridge trigger.
- `buildspec.yml` la file CodeBuild se doc de chay cac buoc validate template.
- `.taskcat.yml` cau hinh Taskcat.
- `.cfnlintrc` cau hinh cfn-lint.

Ly do tach thanh `modules/` la de source CloudFormation de doc hon, gan voi cach to chuc module trong Terraform o Lab01.

## 3. Chuan bi moi truong

### 3.1. Cai AWS CLI

Kiem tra AWS CLI:

```bash
aws --version
```

Neu chua cai AWS CLI, co the cai theo huong dan chinh thuc cua AWS:

```text
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
```

### 3.2. Cau hinh AWS account

Chay lenh:

```bash
aws configure
```

Nhap cac thong tin:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name: ap-southeast-1
Default output format: json
```

Kiem tra tai khoan dang dung:

```bash
aws sts get-caller-identity
```

### 3.3. Kiem tra region

Bai nay dung region Singapore:

```bash
aws configure get region
```

Neu region chua dung, set lai:

```bash
aws configure set region ap-southeast-1
```

### 3.4. Tao EC2 key pair

EC2 instance can key pair de SSH. Neu chua co key pair `lab1-kp`, tao bang lenh:

```bash
aws ec2 create-key-pair \
  --key-name lab1-kp \
  --region ap-southeast-1 \
  --query 'KeyMaterial' \
  --output text > lab1-kp.pem

chmod 400 lab1-kp.pem
```

Kiem tra key pair:

```bash
aws ec2 describe-key-pairs \
  --key-names lab1-kp \
  --region ap-southeast-1
```

Khong commit file `.pem` len GitHub hoac CodeCommit.

### 3.5. Lay public IP cua may minh

Lay IP public:

```bash
curl https://checkip.amazonaws.com
```

Khi truyen vao tham so `MyIp`, them `/32`, vi du:

```text
14.169.22.156/32
```

Security Group cua public EC2 se chi cho phep SSH tu IP nay.

## 4. Kiem tra template truoc khi deploy

Neu muon chay kiem tra local truoc khi push code, cai cac package Python:

```bash
python -m pip install --upgrade pip
pip install "setuptools<81" cfn-lint taskcat
```

Chay cfn-lint:

```bash
cfn-lint --config-file .cfnlintrc *.yaml modules/*.yaml
```

Chay Taskcat lint:

```bash
taskcat lint
```

Trong pipeline that, CodeBuild se tu dong chay cac lenh nay nen buoc local khong bat buoc. Buoc local chu yeu dung de kiem tra nhanh truoc khi push.

## 5. Tao repository tren AWS CodeCommit

Kiem tra CodeCommit repository:

```bash
aws codecommit list-repositories \
  --region ap-southeast-1
```

Neu chua co repo `lab02-bt2`, tao moi:

```bash
aws codecommit create-repository \
  --repository-name lab02-bt2 \
  --repository-description "Lab02 BT2 CloudFormation source" \
  --region ap-southeast-1
```

Lay clone URL:

```bash
aws codecommit get-repository \
  --repository-name lab02-bt2 \
  --region ap-southeast-1 \
  --query 'repositoryMetadata.cloneUrlHttp' \
  --output text
```

## 6. Dua source code len CodeCommit

Clone repo CodeCommit:

```bash
git clone https://git-codecommit.ap-southeast-1.amazonaws.com/v1/repos/lab02-bt2
```

Copy source code cua bai vao repo vua clone. Khi copy nen bo qua `.git`, output tam va file key pair:

```bash
rsync -a \
  --exclude ".git" \
  --exclude ".taskcat_outputs" \
  --exclude "taskcat_outputs" \
  --exclude "packaged" \
  --exclude "*.pem" \
  ./ lab02-bt2/
```

Commit va push:

```bash
cd lab02-bt2
git add .
git commit -m "Add Lab02 CloudFormation source"
git push origin main
```

Sau khi push len CodeCommit, CodePipeline se tu dong duoc kich hoat neu pipeline stack da duoc tao.

## 7. Tao CodePipeline bang CloudFormation

Dung file `pipeline.yaml` de tao pipeline stack:

```bash
aws cloudformation deploy \
  --stack-name lab02-bt2-pipeline \
  --template-file pipeline.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    RepositoryName=lab02-bt2 \
    BranchName=main \
    TargetStackName=lab02-bt2-infrastructure \
    InfrastructureProjectName=lab02-bt2 \
    VpcCidr=10.0.0.0/16 \
    PublicSubnetCidr=10.0.1.0/24 \
    PrivateSubnetCidr=10.0.2.0/24 \
    AvailabilityZone=ap-southeast-1a \
    MyIp=YOUR_PUBLIC_IP/32 \
    InstanceType=t3.micro \
    KeyName=lab1-kp \
  --region ap-southeast-1
```

Can thay `YOUR_PUBLIC_IP/32` bang IP public cua may minh.

Lenh tren se tao:

- CodePipeline ten `lab02-bt2-pipeline`.
- CodeBuild project ten `lab02-bt2-pipeline-build`.
- S3 artifact bucket.
- IAM role cho CodePipeline, CodeBuild va CloudFormation.
- EventBridge rule de pipeline tu chay khi co commit moi tren CodeCommit.

Kiem tra output cua pipeline stack:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

## 8. Pipeline hoat dong nhu the nao

Pipeline gom 3 stage:

```text
Source -> Build -> Deploy
```

Y nghia tung stage:

- `Source`: lay source code tu CodeCommit branch `main`.
- `Build`: goi CodeBuild de chay `cfn-lint`, Taskcat va package nested stack.
- `Deploy`: dung CloudFormation de tao hoac cap nhat stack `lab02-bt2-infrastructure`.

Trong `buildspec.yml`, CodeBuild chay cac lenh chinh:

```bash
cfn-lint --config-file .cfnlintrc *.yaml modules/*.yaml
taskcat lint
aws cloudformation package --template-file infrastructure.yaml ...
taskcat test run -i .taskcat-packaged.yml
```

Neu buoc Build loi, pipeline se dung lai va khong deploy ha tang. Neu Build thanh cong, Deploy stage se chay va tao/cap nhat CloudFormation stack.

## 9. Chay pipeline

Thong thuong chi can push code len CodeCommit:

```bash
git add .
git commit -m "Update Lab02 source"
git push origin main
```

Pipeline se tu dong chay nho EventBridge trigger.

Neu muon chay pipeline thu cong:

```bash
aws codepipeline start-pipeline-execution \
  --name lab02-bt2-pipeline \
  --region ap-southeast-1
```

Kiem tra trang thai pipeline:

```bash
aws codepipeline get-pipeline-state \
  --name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query 'stageStates[].{Stage:stageName,Actions:actionStates[].{Action:actionName,Status:latestExecution.status,Summary:latestExecution.summary}}' \
  --output json
```

Ket qua mong doi:

```text
Source: Succeeded
Build:  Succeeded
Deploy: Succeeded
```

## 10. Deploy ha tang truc tiep bang AWS CLI

Bai nay deploy ha tang thong qua CodePipeline. Tuy nhien, neu can chay thu cong de chup minh chung hoac debug, khong deploy truc tiep file `infrastructure.yaml` ngay.

Ly do: `infrastructure.yaml` goi nested stack bang duong dan local:

```yaml
TemplateURL: modules/vpc.yaml
TemplateURL: modules/security-groups.yaml
TemplateURL: modules/ec2.yaml
```

CloudFormation tren AWS khong doc duoc duong dan local nay. Vi vay can package cac file trong `modules/` len S3 truoc.

Lay artifact bucket cua pipeline:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query "Stacks[0].Outputs[?OutputKey=='ArtifactBucketName'].OutputValue" \
  --output text
```

Package template:

```bash
mkdir -p packaged

aws cloudformation package \
  --template-file infrastructure.yaml \
  --s3-bucket YOUR_PACKAGE_BUCKET \
  --s3-prefix lab02-bt2/nested-templates \
  --output-template-file packaged/infrastructure.yaml \
  --region ap-southeast-1
```

Deploy file da package:

```bash
aws cloudformation deploy \
  --stack-name lab02-bt2-infrastructure \
  --template-file packaged/infrastructure.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ProjectName=lab02-bt2 \
    VpcCidr=10.0.0.0/16 \
    PublicSubnetCidr=10.0.1.0/24 \
    PrivateSubnetCidr=10.0.2.0/24 \
    AvailabilityZone=ap-southeast-1a \
    MyIp=YOUR_PUBLIC_IP/32 \
    InstanceType=t3.micro \
    KeyName=lab1-kp \
  --region ap-southeast-1
```

Neu chay `deploy` truc tiep voi `infrastructure.yaml`, co the gap loi:

```text
TemplateURL must be a supported URL
```

Loi nay co nghia la nested template chua duoc upload len S3.

## 11. Kiem tra ket qua trien khai

### 11.1. Kiem tra CloudFormation stack

Kiem tra stack pipeline:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Kiem tra stack ha tang:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-infrastructure \
  --region ap-southeast-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Ket qua dung la `CREATE_COMPLETE` hoac `UPDATE_COMPLETE`.

Xem output cua stack ha tang:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-infrastructure \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

### 11.2. Kiem tra CodeBuild

Lay build moi nhat:

```bash
aws codebuild list-builds-for-project \
  --project-name lab02-bt2-pipeline-build \
  --region ap-southeast-1 \
  --query 'ids[0]' \
  --output text
```

Xem trang thai build:

```bash
aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --region ap-southeast-1 \
  --query 'builds[0].{Status:buildStatus,Phase:currentPhase,Logs:logs.deepLink}' \
  --output table
```

Ket qua dung:

```text
Status: SUCCEEDED
Phase:  COMPLETED
```

Trong log CodeBuild nen thay cac buoc:

- Cai `cfn-lint` va Taskcat.
- Chay `cfn-lint`.
- Chay `taskcat lint`.
- Chay `aws cloudformation package`.
- Chay `taskcat test run`.

### 11.3. Kiem tra EC2

Kiem tra EC2 instance duoc tao:

```bash
aws ec2 describe-instances \
  --region ap-southeast-1 \
  --filters "Name=tag:Project,Values=lab02-bt2" \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,State:State.Name,InstanceId:InstanceId,PublicIp:PublicIpAddress,PrivateIp:PrivateIpAddress,SubnetId:SubnetId}' \
  --output table
```

Ket qua mong doi:

- Public EC2 o trang thai `running` va co Public IP.
- Private EC2 o trang thai `running` va khong co Public IP.

### 11.4. Kiem tra VPC, subnet va NAT Gateway

Kiem tra VPC:

```bash
aws ec2 describe-vpcs \
  --region ap-southeast-1 \
  --filters "Name=tag:Project,Values=lab02-bt2" \
  --query 'Vpcs[].{VpcId:VpcId,CidrBlock:CidrBlock,State:State}' \
  --output table
```

Kiem tra NAT Gateway:

```bash
aws ec2 describe-nat-gateways \
  --region ap-southeast-1 \
  --filter "Name=tag:Project,Values=lab02-bt2" \
  --query 'NatGateways[].{NatGatewayId:NatGatewayId,State:State,SubnetId:SubnetId}' \
  --output table
```

Ket qua mong doi:

```text
NAT Gateway State: available
```

## 12. Goi y chup man hinh bao cao

Co the chup theo thu tu sau:

1. Cau truc thu muc source code.
2. `infrastructure.yaml` co 3 nested stack: VPC, Security Groups va EC2.
3. Cac file trong `modules/`.
4. `buildspec.yml` co `cfn-lint`, Taskcat va `aws cloudformation package`.
5. `pipeline.yaml` co 3 stage `Source`, `Build`, `Deploy`.
6. Terminal push code len CodeCommit.
7. CodePipeline `lab02-bt2-pipeline` chay thanh cong 3 stage.
8. CodeBuild `lab02-bt2-pipeline-build` co build `SUCCEEDED`.
9. CloudFormation stack `lab02-bt2-infrastructure` co trang thai `CREATE_COMPLETE` hoac `UPDATE_COMPLETE`.
10. Tab Resources cua CloudFormation hien nested stacks va cac resource chinh.
11. EC2 console co public EC2 va private EC2 dang `Running`.
12. VPC console co VPC, subnet, route table, NAT Gateway va Security Groups.

## 13. Don dep tai nguyen

NAT Gateway co the tinh phi, nen sau khi chup minh chung xong nen xoa stack.

Xoa stack ha tang truoc:

```bash
aws cloudformation delete-stack \
  --stack-name lab02-bt2-infrastructure \
  --region ap-southeast-1
```

Cho stack ha tang xoa xong:

```bash
aws cloudformation wait stack-delete-complete \
  --stack-name lab02-bt2-infrastructure \
  --region ap-southeast-1
```

Sau do xoa stack pipeline:

```bash
aws cloudformation delete-stack \
  --stack-name lab02-bt2-pipeline \
  --region ap-southeast-1
```

Neu S3 artifact bucket khong xoa duoc, vao S3 empty bucket truoc roi xoa lai stack pipeline.

## 14. Ghi chu

- GitHub repo dung de luu source code bai nop.
- AWS CodeCommit repo `lab02-bt2` la source that su cua CodePipeline theo yeu cau de bai.
- Khong commit file `.pem`, thu muc `packaged/`, `.taskcat_outputs/` hoac cac file output tam.
