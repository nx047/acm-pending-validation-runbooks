# ACM DNS Validation Checker - RunCommand 버전 요구사항

## 개요

이 버전은 SSM Automation의 `aws:runCommand` action을 사용하여 대상 EC2 instance에서 DNS 진단 스크립트를 실행합니다.

DNS 조회는 EC2 내부의 `dig` 명령을 사용합니다. 따라서 이 버전에서는 `dig`가 필수입니다.

## 실행 방식

- SSM Automation document가 `AWS-RunShellScript`를 호출합니다.
- 대상 EC2 instance에서 Python script가 실행됩니다.
- ACM 조회는 대상 EC2 instance에 설치된 `aws cli`를 통해 수행됩니다.
- DNS 조회는 대상 EC2 instance에 설치된 `dig`를 통해 수행됩니다.

## 입력 파라미터

- `AutomationAssumeRole`: SSM Automation이 사용할 IAM role ARN
- `CertificateArn`: 진단할 ACM certificate ARN
- `InstanceId`: 명령을 실행할 EC2 instance ID

## 필수 사전조건

대상 EC2 instance에는 다음 조건이 필요합니다.

- SSM Agent online 상태
- 인터넷 또는 public DNS resolver 접근 가능
- `python3` 설치
- `aws cli` 설치 및 실행 가능
- `dig` 설치
  - Amazon Linux/RHEL 계열: `bind-utils`
  - Ubuntu/Debian 계열: `dnsutils`

## IAM 권한

Automation role 또는 대상 instance profile에는 다음 권한이 필요합니다.

- `ssm:SendCommand`
- `ssm:GetCommandInvocation`
- `ec2:DescribeInstances`
- `acm:DescribeCertificate`

환경에 따라 SSM document 실행을 위한 추가 Systems Manager 권한이 필요할 수 있습니다.

## 진단 항목

- Certificate ARN 형식 검증
- ACM `DescribeCertificate` 호출
- EMAIL validation 인증서 조기 종료
- `ISSUED` 또는 `EXPIRED` 상태 조기 종료
- CNAME validation record 존재 여부 확인
- CNAME value mismatch 확인
- TXT conflict 확인
- CAA record에 의한 Amazon CA 차단 여부 확인
- NS delegation과 authoritative NS 불일치 확인
- SAN 인증서의 모든 `DomainValidationOptions` 순회
- renewal failure 여부 표시

## 장점

- `dig`를 사용하므로 DNS 조회 결과가 운영자가 수동으로 확인하는 방식과 유사합니다.
- parent nameserver에 직접 질의하는 NS delegation 점검을 구현하기 쉽습니다.
- 외부 Python dependency packaging이 필요 없습니다.

## 단점

- EC2 instance가 필요합니다.
- instance에 `dig`, `aws cli`, `python3`가 설치되어 있어야 합니다.
- Automation role 또는 instance profile 권한 범위가 커집니다.
- 진단 결과가 대상 EC2의 네트워크 환경에 영향을 받을 수 있습니다.

## 권장 사용 상황

- 이미 운영용 bastion 또는 utility EC2가 있는 환경
- `dig` 기반 DNS 질의를 그대로 재현하고 싶은 경우
- parent delegation NS 확인까지 비교적 정확히 수행하고 싶은 경우
