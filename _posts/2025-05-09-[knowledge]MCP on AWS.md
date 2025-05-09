---
title: Amazon Q Developer CLI
date: 2025-05-09 12:00:00 +0900
categories: [knowledege, article]
tags: [mcp, amazon q, amazon q developer, aws mcp]
---

## 개요

AWS에서 공식적으로 Amazon Q Developer CLI를 통해 MCP 지원을 발표[^1]했다.   

이를 통해 Claude Desktop이 아닌 Amazon Q Developer CLI를 통해 MCP를 사용할 수 있게 되었다.

---
## 설치 및 구성

Amazon Q Developer에서 지원하는 운영체제는 MacOS, WSL, Linux를 지원[^2]한다.
그리고, Builder ID가 있어야 하기 때문에 사용하기 전에 미리 가입을 해야한다.
나는 첫 출시 때 자동완성 기능을 사용해보기 위해 설치를 했었었다.  

설치를 하고나서 추가 설정하는 부분은 GUI에서 할 수 있기 때문에 크게 어려운 부분은 없었다.   

![img.png](/assets/img/posts/2025-05-09-MCPonAWS/1.png)

그리고 문제가 발생한 경우 위의 이미지와 같이 `q doctor`와 `q issue`를 통해 오류를 해결할 수 있다.  
나도 처음 설치할 때에는 문제가 있었지만 위의 명령어를 통해 해결했었다.   

위와 같이 기본적인 설정이 완료되었다면 mcp 서버를 설정해보겠다.

aws에서 제공하는 aws mcp 서버들은 awslabs github의 mcp 저장소[^3]에서 확인할 수 있다.

mcp 구성의 경우 json을 통해 설정하고 있으며 Amazon Q의 경우 `~/.aws/amazonq/mcp.json`에 위치한다.

```shell
vi ~/.aws/amazonq/mcp.json
```

나는 아래와 같이 aws 문서, 테라폼 mcp 서버를 추가했다.   

```json
{
  "mcpServers": {
    "awslabs.aws-documentation-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      "disabled": false
    },
    "awslabs.terraform-mcp-server": {
      "command": "uvx",
      "args": ["awslabs.terraform-mcp-server@latest"],
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      "disabled": false
    }
  }
}

```
{: file='mcp.json'}

---

## 실습

실제로 내가 추가한 mcp서버가 정상적으로 동작되는지 확인 해보자

사용 방법은 간단하다 아래 명령어를 터미널에서 실행하면 된다.

```shell
q
```

![img.png](/assets/img/posts/2025-05-09-MCPonAWS/2.png)

그러면 mcp서버를 먼저 설치하고 완료되면 amazon q 실행이 완료된다.   
실행을 하면 인터랙티브 모드로 실행이 되며 Chat GPT를 사용하는 것과 같이 AI 모델과 상호작용을 할 수 있다.

그러면 정상적으로 적용이 됐는지 실행해보자

aws api gateway에서 502 bad gateway가 발생하는 문제에 대해서 질문하면 🛠️  Using tool: search_documentation from mcp server awslabsaws_documentation_mcp_server과 같이 내가 추가한 mcp서버를 사용하는 것을 볼 수 있다.   
기본적으로 사용자의 수락없이는 다음 단계로 진행이 되지 않는다. 전체 단계에 대해서 수락 한다면 `t`를 입력, 다음 단계에 대해서 수락한다면 `y`를 입력하면 된다.   
![img.png](/assets/img/posts/2025-05-09-MCPonAWS/3.png)


내가 자연어로 질문을 기반으로 AWS 문서를 검색하고 문서를 찾아낸 것을 볼 수 있다.
![img.png](/assets/img/posts/2025-05-09-MCPonAWS/4.png)
![img.png](/assets/img/posts/2025-05-09-MCPonAWS/5.png)

관련된 문서를 모두 찾게되면 종합적인 내용을 정리하고 관련된 AWS 문서를 알려준다.
![img.png](/assets/img/posts/2025-05-09-MCPonAWS/6.png)

앞으로 구글 검색보다는 aws 문서 mcp 서버를 연동하여 확인하는 것이 좋을 것으로 생각한다.


---
## Ref.
[^1]: [Amazon Q Developer CLI, 모델 컨텍스트 프로토콜(MCP) 지원 시작](https://aws.amazon.com/ko/blogs/korea/extend-the-amazon-q-developer-cli-with-mcp/?trk=769a1a2b-8c19-4976-9c45-b6b1226c7d20&sc_channel=el)
[^2]: [Amazon Q CLI 설치](https://docs.aws.amazon.com/ko_kr/amazonq/latest/qdeveloper-ug/command-line-installing.html)
[^3]: [awslabs mcp repository](https://github.com/awslabs/mcp.git)
