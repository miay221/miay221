
# AWS 리소스 전부 삭제하는 방법: AWS-nuke

정말 어려웠다... **AWS Budgets 예산 알람**을 켜두고 **Cost Explorer로 비용을 추적**하고 있었지만, 최근 조금 바빠서 놓쳤더니 비용이 발생했었다.

**Elastic Beanstalk**, **VPC** 연결, **Auto Scaling** 등 여러 서비스를 사용한 뒤 **제때 삭제하지 않아 비용이 발생**한 것이니,  

여러분도 사용 후 반드시 **제때 정리**하시길!

---

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

만약 설치가 되어 있지 않다면:
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

AWS Nuke는 **YAML 설정 파일**을 통해 삭제할 리소스를 정의합니다.

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

### 2.1 config.yaml 파일 작성

Windows에서 **config.yaml** 파일 작성 경로:

```bash
C:\Users\<사용자 이름>\aws-nuke\config.yaml
```

PowerShell로 config.yaml을 작성하는 명령어:

```powershell
Set-Content -Path "C:\Users\<사용자 이름>\aws-nuke\config.yaml" -Value @"
regions:
  - "global"
  - "ap-northeast-2"
  - "us-east-1"

account-blacklist: []

accounts:
  "123456789012":
    filters: {}
    resource-types:
      exclude: []
"@
```

---

## 3. AWS 리소스 확인 및 삭제 실행

### 3.1 모든 리소스 조회

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
aws-nuke --config "C:\Users\<사용자 이름>\aws-nuke\config.yaml"
```

---

## 4. 삭제 중 발생할 수 있는 문제 해결

- **DependencyViolation 오류**:
  - 인터넷 게이트웨이, 서브넷, 보안 그룹 등이 남아 있을 수 있습니다.
  - 해당 리소스를 해제한 후 삭제해야 합니다.

- **IAM 권한 문제**:
  - 루트 사용자 또는 삭제 권한이 있는 IAM 역할로 실행해야 합니다.

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

- [AWS Billing 콘솔](https://console.aws.amazon.com/billing/home)에서 모든 비용을 확인합니다.
- 예약 인스턴스 및 Savings Plans 확인:
```bash
aws ec2 describe-reserved-instances --query "ReservedInstances[].ReservedInstancesId" --output text
aws savingsplans describe-savings-plans --query "SavingsPlans[].SavingsPlanId" --output text
```
- 지원 플랜이 **Basic**인지 확인합니다.

---

## 7. 계정 해지

모든 리소스를 삭제했으면 [AWS 계정 해지 절차](https://docs.aws.amazon.com/ko_kr/awsaccountbilling/latest/aboutv2/close-account.html)를 따라 계정을 해지합니다.

---

## 8. 결론

AWS Nuke를 활용해 **모든 리소스를 안전하게 삭제**하고 계정을 정리할 수 있습니다.  
계정을 정리할 때는 **비용 발생 여부를 꾸준히 모니터링**하는 것이 중요합니다.

**빠르게 확인하고 정리**하는 습관을 잊지 마세요!
