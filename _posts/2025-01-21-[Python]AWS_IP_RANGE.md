---
title: AWS 서비스 IP 범위 확인
date: 2025-01-21 12:00:00 +0900
categories: [Programming, Python]
tags: [AWS, Python] # TAG names should always be lowercase
---

## 개요

AWS 서비스에서 사용되는 퍼블릭 엔드포인트의 경우 여러 이유로 변동이 될 수 있다.

이로 인해 AWS에서 json[^1]으로 제공하지만 Python 코드를 사용해서 목록을 볼 수 있거나 코드 내에서 함수를 사용하여 좀 더 유연하게 사용이 가능하다.

## 패키지 설치[^2]

아래 명령어를 사용해서 python 패키지를 설치한다.

```bash
pip install awsipranges
```

## 예제

### 1. 서울 리전의 CodeBuild 퍼블릭 엔드포인트 조회

```python
import awsipranges

region = "ap-northeast-2"

def get_ip_range(regions, services):
    ip_ranges = awsipranges.get_ranges()

    filter_ip_ranges = ip_ranges.filter(regions=regions, services=services)
    ip_range_list = []

    for ip_range in filter_ip_ranges:
        ip_range_list.append(str(ip_range))

    return ip_range_list


res = get_ip_range(region, 'CODEBUILD')
print(res)
```

`awsipranges`의 `get_ranges()` 함수를 사용하여 IP 범위를 조회하고 `filter` 함수를 사용해서 원하는 서비스의 IP 범위를 조회할 수 있다.

위와 같이 서비스명은 CODEBUILD로 조회하면 해당 서비스의 IP 범위를 조회할 수 있다.

###### 출력 결과

```python
['3.38.90.8/29', '13.124.145.16/29']
```

### 2. 다중 리전의 CodeBuild, EC2 퍼블릭 엔드포인트 조회

```python
region = "ap-northeast-2"
services = ['CODEBUILD', 'EC2']


def get_ip_range(regions, services):
    ip_ranges = awsipranges.get_ranges()

    filter_ip_ranges = ip_ranges.filter(regions=regions, services=services)
    ip_range_list = []

    for ip_range in filter_ip_ranges:
        ip_range_list.append(str(ip_range))

    return ip_range_list


res = get_ip_range(region, services)
print(res)
```

`services` 변수에 조회하고 싶은 서비스를 리스트 형식으로 추가하면 해당 서비스의 IP 범위를 조회할 수 있다.
`regions`도 동일하게 리스트 형식으로 사용하면 조회할 수 있다.

###### 출력 결과

```python
['3.2.37.0/26', '3.5.140.0/22', '3.5.144.0/23', '3.5.184.0/21', '3.34.0.0/15', '3.36.0.0/14', '3.38.90.8/29', '13.124.0.0/16', '13.124.145.16/29', '13.125.0.0/16', '13.209.0.0/16', '15.164.0.0/15', '15.177.76.0/24', '15.193.9.0/24', '35.71.109.0/24', '43.200.0.0/14', '52.78.0.0/16', '52.79.0.0/16', '52.94.248.176/28', '52.95.252.0/24', '54.180.0.0/15', '99.77.141.0/24', '99.77.242.0/24', '99.150.24.0/21', '99.151.144.0/21', '151.148.40.0/24', '159.248.200.0/21', '159.248.216.0/21', '173.83.198.0/24', '2406:da00:2000::/40', '2406:da12::/36', '2406:da15::/36', '2406:da22::/36', '2406:da25::/36', '2406:da60:2000::/40', '2406:da61:2000::/40', '2406:da68:2000::/40', '2406:da69:2000::/40', '2406:da70:2000::/40', '2406:daf0:2000::/40', '2406:daf1:2000::/40', '2406:daf2:2000::/40', '2406:daff:2000::/40', '2600:f0f0:1:1000::/56', '2600:f0f0:82:900::/56']
```

## Ref.

[^1]: [AWS IP Range Json](https://aws-samples.github.io/awsipranges/index.html)
[^2]: [AWSIPRange 패키지 문서](https://github.com/aws-samples/awsipranges)
