# Lab02 BT2 - CloudFormation, CodeBuild, Taskcat và CodePipeline

## 1. Giới Thiệu

Đây là bài thực hành triển khai lại hạ tầng của Lab01 bằng AWS CloudFormation, sau đó tự động hóa quy trình kiểm tra và deploy bằng AWS CodePipeline.

Hạ tầng được tạo gồm:

- VPC.
- Public subnet và private subnet.
- Internet Gateway.
- NAT Gateway.
- Route tables cho public/private subnet.
- Security Groups.
- 2 EC2 instance: 1 public EC2 và 1 private EC2.

Phần CI/CD được tạo gồm:

- AWS CodeCommit làm source repository.
- AWS CodeBuild chạy `cfn-lint` và Taskcat để kiểm tra template.
- AWS CodePipeline gồm 3 stage: `Source`, `Build`, `Deploy`.
- CloudFormation deploy stack hạ tầng từ template đã được package.

## 2. Cấu Trúc Source Code

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

Trong đó:

- `infrastructure.yaml` là root stack. File này gọi các nested stack trong thư mục `modules/`.
- `modules/vpc.yaml` tạo VPC, subnet, Internet Gateway, NAT Gateway và route table.
- `modules/security-groups.yaml` tạo Security Group cho public EC2 và private EC2.
- `modules/ec2.yaml` tạo public EC2 và private EC2.
- `pipeline.yaml` tạo CodeBuild, CodePipeline, IAM roles, S3 artifact bucket và EventBridge trigger.
- `buildspec.yml` là file CodeBuild sẽ đọc để chạy các bước validate template.
- `.taskcat.yml` cấu hình Taskcat.
- `.cfnlintrc` cấu hình cfn-lint.

Lý do tách thành `modules/` là để source CloudFormation dễ đọc hơn, gần với cách tổ chức module trong Terraform ở Lab01.

## 3. Chuẩn Bị Môi Trường

### 3.1. Cài AWS CLI

Kiểm tra AWS CLI:

```bash
aws --version
```

Nếu chưa cài AWS CLI, có thể cài theo hướng dẫn chính thức của AWS:

```text
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
```

### 3.2. Cấu Hình AWS Account

Chạy lệnh:

```bash
aws configure
```

Nhập các thông tin:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name: ap-southeast-1
Default output format: json
```

Kiểm tra tài khoản đang dùng:

```bash
aws sts get-caller-identity
```

### 3.3. Kiểm Tra Region

Bài này dùng region Singapore:

```bash
aws configure get region
```

Nếu region chưa đúng, set lại:

```bash
aws configure set region ap-southeast-1
```

### 3.4. Tạo EC2 Key Pair

EC2 instance cần key pair để SSH. Nếu chưa có key pair `lab1-kp`, tạo bằng lệnh:

```bash
aws ec2 create-key-pair \
  --key-name lab1-kp \
  --region ap-southeast-1 \
  --query 'KeyMaterial' \
  --output text > lab1-kp.pem

chmod 400 lab1-kp.pem
```

Kiểm tra key pair:

```bash
aws ec2 describe-key-pairs \
  --key-names lab1-kp \
  --region ap-southeast-1
```

Không commit file `.pem` lên GitHub hoặc CodeCommit.

### 3.5. Lấy Public IP Của Máy Mình

Lấy IP public:

```bash
curl https://checkip.amazonaws.com
```

Khi truyền vào tham số `MyIp`, thêm `/32`, ví dụ:

```text
14.169.22.156/32
```

Security Group của public EC2 sẽ chỉ cho phép SSH từ IP này.

## 4. Kiểm Tra Template Trước Khi Deploy

Nếu muốn chạy kiểm tra local trước khi push code, cài các package Python:

```bash
python -m pip install --upgrade pip
pip install "setuptools<81" cfn-lint taskcat
```

Chạy cfn-lint:

```bash
cfn-lint --config-file .cfnlintrc *.yaml modules/*.yaml
```

Chạy Taskcat lint:

```bash
taskcat lint
```

Trong pipeline thật, CodeBuild sẽ tự động chạy các lệnh này nên bước local không bắt buộc. Bước local chủ yếu dùng để kiểm tra nhanh trước khi push.

## 5. Tạo Repository Trên AWS CodeCommit

Kiểm tra CodeCommit repository:

```bash
aws codecommit list-repositories \
  --region ap-southeast-1
```

Nếu chưa có repo `lab02-bt2`, tạo mới:

```bash
aws codecommit create-repository \
  --repository-name lab02-bt2 \
  --repository-description "Lab02 BT2 CloudFormation source" \
  --region ap-southeast-1
```

Lấy clone URL:

```bash
aws codecommit get-repository \
  --repository-name lab02-bt2 \
  --region ap-southeast-1 \
  --query 'repositoryMetadata.cloneUrlHttp' \
  --output text
```

## 6. Đưa Source Code Lên CodeCommit

Clone repo CodeCommit:

```bash
git clone https://git-codecommit.ap-southeast-1.amazonaws.com/v1/repos/lab02-bt2
```

Copy source code của bài vào repo vừa clone. Khi copy nên bỏ qua `.git`, output tạm và file key pair:

```bash
rsync -a \
  --exclude ".git" \
  --exclude ".taskcat_outputs" \
  --exclude "taskcat_outputs" \
  --exclude "packaged" \
  --exclude "*.pem" \
  ./ lab02-bt2/
```

Commit và push:

```bash
cd lab02-bt2
git add .
git commit -m "Add Lab02 CloudFormation source"
git push origin main
```

Sau khi push lên CodeCommit, CodePipeline sẽ tự động được kích hoạt nếu pipeline stack đã được tạo.

## 7. Triển Khai Thực Tế

Phần này là luồng chạy chính của bài: tạo pipeline bằng CloudFormation, đưa source code lên CodeCommit, sau đó để CodePipeline tự build và deploy hạ tầng.

### 7.1. Deploy Pipeline Stack

Dùng file `pipeline.yaml` để tạo các tài nguyên phục vụ CI/CD:

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

Cần thay `YOUR_PUBLIC_IP/32` bằng IP public của máy đang dùng, ví dụ `14.169.22.156/32`.

Lệnh trên sẽ tạo:

- CodePipeline tên `lab02-bt2-pipeline`.
- CodeBuild project tên `lab02-bt2-pipeline-build`.
- S3 artifact bucket.
- IAM role cho CodePipeline, CodeBuild và CloudFormation.
- EventBridge rule để pipeline tự chạy khi có commit mới trên CodeCommit.

Kiểm tra output của pipeline stack sau khi deploy:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

### 7.2. Push Source Code Lên CodeCommit

Nếu đang ở repo CodeCommit đã clone, commit và push source code:

```bash
git add .
git commit -m "Update Lab02 CloudFormation source"
git push origin main
```

Sau khi push lên branch `main`, EventBridge sẽ kích hoạt CodePipeline.

Pipeline chạy theo thứ tự:

```text
Source -> Build -> Deploy
```

Trong đó:

- `Source`: lấy source code từ CodeCommit.
- `Build`: dùng CodeBuild để chạy `cfn-lint`, Taskcat và package nested stack.
- `Deploy`: dùng CloudFormation để tạo hoặc cập nhật stack `lab02-bt2-infrastructure`.

Trong stage `Build`, CodeBuild đọc file `buildspec.yml` và chạy các lệnh chính:

```bash
cfn-lint --config-file .cfnlintrc *.yaml modules/*.yaml
taskcat lint
aws cloudformation package --template-file infrastructure.yaml ...
taskcat test run -i .taskcat-packaged.yml
```

Nếu stage `Build` thành công, pipeline mới chuyển sang stage `Deploy`.

### 7.3. Chạy Lại Pipeline Khi Cần

Thông thường chỉ cần push code mới lên CodeCommit là pipeline sẽ tự chạy. Nếu muốn chạy lại pipeline thủ công, dùng lệnh:

```bash
aws codepipeline start-pipeline-execution \
  --name lab02-bt2-pipeline \
  --region ap-southeast-1
```

### 7.4. Deploy Hạ Tầng Trực Tiếp Bằng AWS CLI

Bài này triển khai hạ tầng chính thông qua CodePipeline. Tuy nhiên, trong trường hợp muốn chạy thủ công bằng AWS CLI để kiểm tra template hoặc cập nhật stack trực tiếp, cần package template trước khi deploy.

Lý do là `infrastructure.yaml` đang dùng nested stack:

```yaml
TemplateURL: modules/vpc.yaml
TemplateURL: modules/security-groups.yaml
TemplateURL: modules/ec2.yaml
```

CloudFormation trên AWS cần các nested template này nằm trên S3. Vì vậy, trước tiên lấy tên artifact bucket của pipeline:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query "Stacks[0].Outputs[?OutputKey=='ArtifactBucketName'].OutputValue" \
  --output text
```

Sau đó package template:

```bash
mkdir -p packaged

aws cloudformation package \
  --template-file infrastructure.yaml \
  --s3-bucket YOUR_PACKAGE_BUCKET \
  --s3-prefix lab02-bt2/nested-templates \
  --output-template-file packaged/infrastructure.yaml \
  --region ap-southeast-1
```

Cuối cùng deploy file đã package:

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

## 8. Kiểm Tra Kết Quả Trên AWS Console

Sau khi pipeline chạy xong, có thể kiểm tra trực tiếp trên AWS Console.

### 8.1. CodeCommit

Vào AWS Console:

```text
Developer Tools -> CodeCommit -> Repositories -> lab02-bt2
```

Kiểm tra source code đã có các file `infrastructure.yaml`, `pipeline.yaml`, `buildspec.yml`, `.taskcat.yml` và thư mục `modules/`.

### 8.2. CodePipeline

Vào:

```text
Developer Tools -> CodePipeline -> Pipelines -> lab02-bt2-pipeline
```

Kết quả đúng là 3 stage đều thành công:

```text
Source: Succeeded
Build: Succeeded
Deploy: Succeeded
```

### 8.3. CodeBuild

Vào:

```text
Developer Tools -> CodeBuild -> Build projects -> lab02-bt2-pipeline-build
```

Mở build mới nhất và kiểm tra trạng thái `Succeeded`. Trong build log có thể thấy các bước cài `cfn-lint`, cài Taskcat, chạy `cfn-lint`, chạy `taskcat lint`, chạy `aws cloudformation package` và chạy `taskcat test run`.

### 8.4. CloudFormation

Vào:

```text
CloudFormation -> Stacks
```

Kiểm tra các stack:

```text
lab02-bt2-pipeline
lab02-bt2-infrastructure
```

Stack `lab02-bt2-infrastructure` cần ở trạng thái `CREATE_COMPLETE` hoặc `UPDATE_COMPLETE`. Trong tab Resources có thể thấy các nested stack như `VpcStack`, `SecurityGroupsStack` và `Ec2Stack`.

### 8.5. EC2 Và VPC

Trong EC2 console, kiểm tra có 2 instance được tạo:

- Public EC2: có Public IP, nằm trong public subnet.
- Private EC2: không có Public IP, nằm trong private subnet.

Trong VPC console, kiểm tra các tài nguyên mạng:

- VPC của project `lab02-bt2`.
- Public subnet và private subnet.
- Route table cho public subnet đi qua Internet Gateway.
- Route table cho private subnet đi qua NAT Gateway.
- NAT Gateway ở trạng thái `Available`.
- Security Group của public EC2 và private EC2.

## 9. Kiểm Tra Kết Quả Bằng AWS CLI

Ngoài AWS Console, có thể dùng AWS CLI để kiểm tra nhanh toàn bộ kết quả triển khai.

### 9.1. Kiểm Tra Pipeline

Kiểm tra trạng thái pipeline:

```bash
aws codepipeline get-pipeline-state \
  --name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query 'stageStates[].{Stage:stageName,Actions:actionStates[].{Action:actionName,Status:latestExecution.status,Summary:latestExecution.summary}}' \
  --output json
```

Xem các execution gần nhất:

```bash
aws codepipeline list-pipeline-executions \
  --pipeline-name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --max-results 5 \
  --query 'pipelineExecutionSummaries[].{Id:pipelineExecutionId,Status:status,Commit:sourceRevisions[0].revisionId,Message:sourceRevisions[0].revisionSummary}' \
  --output table
```

### 9.2. Kiểm Tra CodeBuild

Lấy build mới nhất của project:

```bash
aws codebuild list-builds-for-project \
  --project-name lab02-bt2-pipeline-build \
  --region ap-southeast-1 \
  --query 'ids[0]' \
  --output text
```

Xem chi tiết build:

```bash
aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --region ap-southeast-1 \
  --query 'builds[0].{Status:buildStatus,Phase:currentPhase,Logs:logs.deepLink}' \
  --output table
```

Kết quả đúng là:

```text
Status: SUCCEEDED
Phase:  COMPLETED
```

### 9.3. Kiểm Tra CloudFormation

Kiểm tra trạng thái pipeline stack:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-pipeline \
  --region ap-southeast-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Kiểm tra trạng thái infrastructure stack:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-infrastructure \
  --region ap-southeast-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Xem output của infrastructure stack:

```bash
aws cloudformation describe-stacks \
  --stack-name lab02-bt2-infrastructure \
  --region ap-southeast-1 \
  --query 'Stacks[0].Outputs' \
  --output table
```

Xem các resource trong stack:

```bash
aws cloudformation list-stack-resources \
  --stack-name lab02-bt2-infrastructure \
  --region ap-southeast-1 \
  --query 'StackResourceSummaries[].{LogicalId:LogicalResourceId,Type:ResourceType,Status:ResourceStatus}' \
  --output table
```

### 9.4. Kiểm Tra EC2

Kiểm tra các EC2 instance của project:

```bash
aws ec2 describe-instances \
  --region ap-southeast-1 \
  --filters "Name=tag:Project,Values=lab02-bt2" \
  --query 'Reservations[].Instances[].{Name:Tags[?Key==`Name`]|[0].Value,State:State.Name,InstanceId:InstanceId,PublicIp:PublicIpAddress,PrivateIp:PrivateIpAddress,SubnetId:SubnetId}' \
  --output table
```

Kết quả mong đợi:

- Public EC2 ở trạng thái `running` và có Public IP.
- Private EC2 ở trạng thái `running` và không có Public IP.

### 9.5. Kiểm Tra VPC, Subnet Và NAT Gateway

Kiểm tra VPC:

```bash
aws ec2 describe-vpcs \
  --region ap-southeast-1 \
  --filters "Name=tag:Project,Values=lab02-bt2" \
  --query 'Vpcs[].{VpcId:VpcId,CidrBlock:CidrBlock,State:State}' \
  --output table
```

Kiểm tra NAT Gateway:

```bash
aws ec2 describe-nat-gateways \
  --region ap-southeast-1 \
  --filter "Name=tag:Project,Values=lab02-bt2" \
  --query 'NatGateways[].{NatGatewayId:NatGatewayId,State:State,SubnetId:SubnetId}' \
  --output table
```

Kết quả mong đợi:

```text
NAT Gateway State: available
```
