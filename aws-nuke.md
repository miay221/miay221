
# AWS 리소스 전부 삭제하는 방법: AWS-nuke

정말 어려웠다... **AWS Budgets 예산 알람**을 켜두고 **Cost Explorer로 비용을 추적**하고 있었지만, 최근 조금 바빠서 놓쳤더니 비용이 발생했었다.

**Elastic Beanstalk**, **VPC** 연결, **Auto Scaling** 등 여러 서비스를 사용한 뒤 **제때 삭제하지 않아 비용이 발생**한 것이니,  

여러분도 사용 후 반드시 **제때 정리**하시길!








## 1. 준비 작업

### 1.1 AWS CLI와 AWS Nuke 설치 확인

- **AWS CLI 설치 확인:**
```bash
aws --version
```

- **AWS Nuke 설치 확인:**
```bash
aws-nuke --version
```

만약 설치가 되어 있지 않다면?
- [AWS CLI 설치 링크](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [AWS Nuke 다운로드 링크](https://github.com/rebuy-de/aws-nuke/releases)

---

### 1.2 AWS 자격증명 설정

```bash
aws configure
```

- **Access Key ID**: 루트 또는 IAM 사용자의 키 입력
- **Secret Access Key**: 루트 또는 IAM 사용자의 비밀 키 입력
- **Region**: 기본 리전 설정 (예: `ap-northeast-2`)

자격이 올바른지 확인:

```bash
aws sts get-caller-identity
```

출력 예:
```json
{
  "UserId": "ABC123EXAMPLE",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:root"
}
```

---

## 2. AWS Nuke 설정 파일 작성

**YAML 설정 파일**을 통해 삭제할 리소스를 정해야 하는데 이건 각자 상황에 맞게 만드셔야 합니다!

저는 테스팅 목적의 계정이라 모든 리소스 삭제를 위해 filters 를 비워뒀어요

**예시 config.yaml:**
```yaml
regions:
  - "global"  # 글로벌 서비스 (IAM 등)
  - "ap-northeast-2"  # 서울 리전
  - "us-east-1"  # 추가 리전

account-blacklist: []  # 블랙리스트에 계정을 추가하지 않음

accounts:
  "123456789012":  # AWS 계정 번호
    filters: {}  # 모든 리소스를 삭제
    resource-types:
      exclude: []  # 예외 리소스 없음
```

---

## 3. AWS 리소스 확인 및 삭제 실행

### 3.1 어떤 리소스를 사용하고 있는지 전체 조회

```powershell
$regions = (aws ec2 describe-regions --query "Regions[].RegionName" --output text).Split()

foreach ($region in $regions) {
    Write-Output "Checking resources in region: $region"
    $vpcs = aws ec2 describe-vpcs --region $region --query "Vpcs[].VpcId" --output text
    if ($vpcs) {
        Write-Output "VPCs found in $region: $vpcs"
    } else {
        Write-Output "No VPCs in $region"
    }
}
```

### 3.2 AWS Nuke로 모든 리소스 삭제

```powershell
aws-nuke --config "여기는 config.yaml 파일의 경로 작성"
```

---

## 4. 삭제 중 발생할 수 있는 문제 해결

- **DependencyViolation 오류**:
  - 인터넷 게이트웨이, 서브넷, 보안 그룹 등이 남아 있으면 삭제가 안될 수 있습니다!
  - 먼저 해당 리소스를 해제한 후에 다시 삭제해야 합니다.

- **IAM 권한 문제**:
  - 루트 사용자 또는 삭제 권한이 있는 IAM 역할 권한이 있어야 해요. 계정 확인하시고 반드시 alias 설정하기!
  - (aws-nuke 가 모든 리소스를 삭제하는 위험한 작업이다보니 alias를 입력해서 사용자를 식별하더라구요)

---

## 5. 삭제 확인

### 5.1 EC2 인스턴스 확인
```bash
aws ec2 describe-instances --query "Reservations[].Instances[].InstanceId" --output text
```

### 5.2 S3 버킷 확인
```bash
aws s3api list-buckets --query "Buckets[].Name" --output text
```

### 5.3 Lambda 함수 확인
```bash
aws lambda list-functions --query "Functions[].FunctionName" --output text
```

---

## 6. 계정 해지 전 최종 점검

- [AWS Billing 콘솔](https://console.aws.amazon.com/billing/home)에서 비용 트래킹하면서 해지 확인하세요
- Support 플랜이 **Basic**인지 확인합니다. (베이직이 무료 플랜임)

---

## 7. 계정 해지 (선택 사항)

[AWS 계정 해지 절차](https://docs.aws.amazon.com/ko_kr/awsaccountbilling/latest/aboutv2/close-account.html)

---

## 8. 배운 것

AWS Nuke가 없었다면...모든 리전마다 리소스 찾아서 해제하고 삭제하고 너무 번거로웠을 것 같은데
다행히 활성화 중인 서비스가 없어서 전부 삭제할 수 있었어요;-; 여러분도 

**빠르게 확인하고 정리**하는거 잊지 마시고 과금없이 서버 잘 배포해요!
