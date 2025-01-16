---
title: AMI VM Import/Export
date: 2025-01-10 12:00:00 +0900
categories: [AWS, EC2, EBS]
tags: [AMI, VM] # TAG names should always be lowercase
---

## 개요

AMI를 가상 머신에서 사용가능하도록 추출하거나 반대로 가상 머신 이미지를 AMI로 생성하는 방법을 알아보자.

#### 사전 조건[^1]

- AWS CLI 설치
- Amazon S3 버킷
- `vmimport` IAM 역할

#### 고려사항[^2]

여러가지 고려사항이 있지만 우선적으로 확인해야할 고려사항만 간추렸다.

- VM Import/Export는 동일한 계정의 S3 버킷으로 VM 내보내기만 지원
- 블록 디바이스가 암호화된 EBS 볼륨은 내보내기 불가
- Windows 이미지, SQL Server, AWS Marketplace 이미지는 내보내기 불가
- 다른 AWS 계정에서 공유받은 이미지는 내보내기 불가
- 1TiB보다 큰 볼륨은 VM 지원되지 않음

---

## 작업

#### 1. S3 버킷 생성

VM Export된 이미지를 저장한 S3 버킷 생성이 필요하다.

버킷이름: `vmimport-123456789012`

#### 2. IAM 역할 생성

##### 신뢰 정책

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vmie.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:Externalid": "vmimport"
        }
      }
    }
  ]
}
```

##### IAM 정책

정책 후 생성 후 역할에 연결한다.

Resource는 위에서 생성한 S3 버킷 arn을 지정한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetBucketLocation", "s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::<vmimport-123456789012>",
        "arn:aws:s3:::<vmimport-123456789012>/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetBucketLocation",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject",
        "s3:GetBucketAcl"
      ],
      "Resource": [
        "arn:aws:s3:::<vmimport-123456789012>",
        "arn:aws:s3:::<vmimport-123456789012>/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:ModifySnapshotAttribute",
        "ec2:CopySnapshot",
        "ec2:RegisterImage",
        "ec2:Describe*"
      ],
      "Resource": "*"
    }
  ]
}
```

#### 3. 이미지 내보내기

`export-image` 명령어를 사용해서 이미지를 내보낸다.

이 때 지원되는 디스크 이미지 포맷은 VMDK, RAW, VHD 세가지이다.

| 포맷 | 설명                                      | 용도                                                                     |
| ---- | ----------------------------------------- | ------------------------------------------------------------------------ |
| VMDK | VMWare 가상 머신 디스크 이미지            | VMWare vSphere/ESxi, Workstation, Fusion에서 사용                        |
| RAW  | 블록 디바이스 이미지                      | 클라우드 및 온프레미스에서 직접 사용하거나 QEMU/KVM과 같은 환경에서 사용 |
| VHD  | Microsoft Hyper-V 가상 머신 디스크 이미지 | Hyper-V 환경으로 마이그레이션 또는 Azure 클라우드에서 사용               |

###### export-image 명령어 실행

VMDK를 사용해서 VMWare에서 사용 가능한 이미지를 내보내기한다.

```bash
aws ec2 export-image --image-id <ami-id> --disk-image-format VMDK --s3-export-location S3Bucket=<vmimport-123456789012>,S3Prefix=export/ --region <region>
```

###### 출력 결과

```json
{
  "DiskImageFormat": "VMDK",
  "ExportImageTaskId": "export-ami-abcd1234",
  "ImageId": "ami-abcd1234",
  "Progress": "0",
  "S3ExportLocation": {
    "S3Bucket": "vmimport-123456789012",
    "S3Prefix": "export/"
  },
  "Status": "active",
  "StatusMessage": "validating"
}
```

###### 진행률 확인

```bash
aws ec2 describe-export-image-tasks --export-image-task-ids export-ami-abcd1234 --region <region>
```

###### 출력 결과

```json
{
    "ExportImageTasks": [
        {
            "ExportImageTaskId": "export-ami-abcd1234",
            "ImageId": "ami-abcd1234",
            "Progress": "60",
            "S3ExportLocation": {
                "S3Bucket": "vmimport-123456789012",
                "S3Prefix": "export/"
            },
            "Status": "active",
            "StatusMessage": "updating",
            "Tags": []
        }
    ]
```

## Ref.

[^1]: [이미지 내보내기 사전 조건](https://docs.aws.amazon.com/ko_kr/vm-import/latest/userguide/prerequisites-image-export.html)
[^2]: [이미지 내보내기 고려사항](https://docs.aws.amazon.com/ko_kr/vm-import/latest/userguide/limits-image-export.html)
