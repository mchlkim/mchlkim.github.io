---
title: n8n으로 뉴스 피드 자동화하기
date: 2025-07-28 12:00:00 +0900
categories: [knowledge, article]
tags: [n8n]
---


# 개요
지난 게시글에서 n8n 서버를 직접 구축[^1]했었다.

이번 글에서는 RSS 피드를 단순 수집하는 것이 아닌 LLM을 사용해 요약 및 정리하여 메일로 전송하는 워크플로우에 대해 정리해보았다.


# 워크플로우
![Workflow](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/workflow.png)

1. 2개 이상의 RSS 피드를 읽어와서 Merge 노드를 통해 RSS 피드 데이터를 통합
2. 통합된 데이터를 Filter 노드를 통해 오늘 날짜에 발행된 글만 필터링
3. RSS 피드 구조의 데이터를 json 구조의 데이터로 변환
4. json으로 구조화된 데이터를 LLM으로 전달
5. LLM에서 시스템 프롬트를 통해 입력된 데이터를 가공
6. 가공된 데이터에서 json 구조 추출
7. 추출된 json 데이터를 다중 배열로 병합
8. 정렬된 데이터별로 메일 내용 분리 및 HTML 작성
9. 생성된 HTML을 기반으로 메일 전송

---
# 상세정보

n8n 버전 정보: 1.102.4

### 01. 트리거
![img.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/trigger.png)

| 노드  유형           | 노드 설명     | 등록 값 |
|------------------|-----------|------|
| Schedule Trigger | 일정 기반 트리거 | -    |

#### 설명

트리거는 Schedule 기반으로 매일 9시에 실행
    

### 02. RSS 피드 등록

![img_1.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/rss_01.png)
![img_2.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/rss_02.png)

#### 설명

| 노드  유형   | 노드 설명      | 등록 값                                       |
|----------|------------|--------------------------------------------|
| RSS Read | AWS 기술 블로그 | https://aws.amazon.com/ko/blogs/tech/feed  |
| RSS Read | AWS 한국 블로그 | https://aws.amazon.com/ko/blogs/korea/feed |

내가 수집하고 싶은 RSS 뉴스 피드를 등록하면 된다.

노드에 등록하기 전 내가 원하는 정보가 RSS 피드에 포함되는지 확인한다.   




### 03. 데이터 병합

![img_3.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/merge.png)

| 노드  유형 | 노드 설명      | 등록 값 |
|--------|------------|------|
| Merge  | AWS 기술 블로그 | -    |

#### 설명

등록된 RSS Read 노드 개수만큼 Number of Inputs를 지정해주면 된다.  
이 글에서는 2개의 RSS Read 노드가 등록되어 있기 때문에 2로 지정했다.  

### 04. 날짜 필터

![img_5.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/filter.png)

| 노드 유형 | 노드 설명 | 등록 값 |
|-----------|-----------|---------|
| Filter    | 오늘 날짜 이후에 발행된 글만 필터링 | {% raw %} {{ $json.pubDate.toDateTime().format('yyyy-MM-dd') }} {% endraw %} |
| -         | -         | {% raw %} {{ $today.format('yyyy-MM-dd') }} {% endraw %} |

  
필터에서는 다양한 유형의 기준을 제공한다. 
- 문자열
- 숫자
- 날짜 및 시간
- 논리형 (Boolean)
- 배열
- 오브젝트  

앞에 오는 값이 비교 값, 뒤에 오는 값이 기준 값이다.    

x = 비교 값, y는 기준 값으로 표현한다면 `x ≥ y`와 같다.  

#### 설명

```text 
{% raw %} {{ $json.pubDate.toDateTime().format('yyyy-MM-dd') }} {% endraw %}
```

RSS에서 제공되는 날짜는 가독성이 떨어지기 때문에 가독성을 위해 우리가 보기 편한 형식으로 변환해준다.    


`$json.pubDate`는 RSS 피드에서 제공되는 pubDate 값은 RFC2822 날짜 포맷(Mon, 28 Jul 2025 05:05:35 +0000)으로 제공  
`toDateTime()`는 RFC2822 포맷을 ISO 8601 포맷으로 변환해주는 함수  
`format('yyyy-MM-dd')`는 ISO 8601 포맷에서 연-월-일 형식으로 출력해주는 함수  

이를 통해 날짜 데이터를 가공하여 처리할 수 있다.  
  
  

                  
```text 
{% raw %} {{ $today.format('yyyy-MM-dd') }} {% endraw %}
```

`$today`는 n8n에서 제공되는 오늘 날짜를 출력해주는 함수  
`format('yyyy-MM-dd')`을 통해 포맷을 맞춰준다.

### 05. JSON 포맷으로 구조화

![json_structure](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/json_structure.png)

| 노드 유형 | 노드 설명 | 등록 값 |
|--------|-------------------------------|------------------------------|
| Code   | RSS 피드 데이터에서 필요한 정보만 json 구조화 | `json_structure.js` 코드 블럭 참고 |
  
```js
const results = [];

for (const item of items) {
  results.push({
    json: {
      title: item.json.title || '',
      content: item.json['content:encodedSnippet'] || '',
      categories0: item.json.categories?.[0] || '',
      categories1: item.json.categories?.[1] || '',
      link: item.json.link || ''
    }
  });
}

return results;
```
{: file='json_structure.js'}

#### 설명

json 구조로 변경하는 이유는 가공을 통해 필요한 데이터만 추출하기 위함이다.    

LLM을 사용하는 경우 가공을 하지 않고 RSS 피드 내용을 그대로 입력하게 되면 입력 토큰의 낭비로 이어질 수 있다.  
이러한 이유로 필요한 데이터만 json구조로 재가공 하는 것이다.  

### 06. LLM으로 내용 요약

![llm_summary.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/llm_summary.png)


| 노드  유형          | 노드 설명                     | 등록 값              |
|-----------------|---------------------------|-------------------|
| Basic LLM Chain | RSS 피드의 콘텐츠를 3000자 이하로 요약 | `prompt` 코드 블럭 참고 |

```
{% raw %} 
# 요청 사항
"{{ $json.content }}"를 3000자 이하로 요약해서 출력 예제 형식에 맞게 출력


#조건 
1. 출력에 대한 불필요한 대답은 하지 않음
2. 가시성을 위해 문단이 끝나는 경우 줄바꿈 적용

# 출력 예제

return [
  {
    json: {
      title: "{{ $json.title }}",
      content: "<요약된 content>",
      categories0: "{{ $json.categories0 }}",
      categories1: "{{ $json.categories1 }}",
      link: {{ $json.link }}
    }
  }
];
{% endraw %} 
```
{: file='prompt'}

#### 설명

Basic LLM Chain 노드를 사용하면 아래와 같이 Model을 연결할 수 있다.  

![basic_llm_chain.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/basic_llm_chain.png)

모델의 자격 증명은 아래와 같이 API Key를 입력해주면 된다.
![gemini.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/gemini.png)

무료로 제공되는 구글 Gemini API[^2]를 사용하여 모델을 연결해주면 된다.   


흐름은 Basic LLM Chain에 등록된 Prompt를 모델 노드로 요청하게 되는 구조이고 LLM에서 처리가 완료된 경우 다음 단계로 넘어가게 된다.  

### 07. LLM 응답 추출

![export_json.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/export_json.png)

| 노드 유형 | 노드 설명 | 등록 값 |
|--------|-------------------------------------------|---------------------------|
| Code   | LLM의 출력값에서 json 구조를 파싱 및 가공하여 json 데이터 추출 | `export_json.js` 코드 블럭 참고 |

```js
// 결과를 저장할 배열
const output = [];

// 모든 입력 아이템 처리
for (const item of items) {
  const rawText = item.json.text;

  // 코드 블록 제거
  let code = rawText.replace(/^```json\s*|\s*```$/g, '').trim();

  try {
    let parsed;

    // "return"이 포함되어 있으면 JavaScript 객체 코드로 간주
    if (code.startsWith('return')) {
      // return 키워드 제거
      code = code.replace(/^return\s*/, '').trim();
      if (code.endsWith(';')) {
        code = code.slice(0, -1).trim();
      }

      // eval로 자바스크립트 객체 실행 (괄호로 감싸야 eval이 객체로 인식)
      parsed = eval('(' + code + ')');
    } else {
      // JSON으로 처리
      parsed = JSON.parse(code);
    }

    // 결과 추가
    if (Array.isArray(parsed)) {
      for (const p of parsed) {
        // n8n이 요구하는 포맷: { json: {...} }
        output.push({
          json: p.json || p  // p가 { json: {...} } 형태이거나 그냥 {...}일 경우 모두 대응
        });
      }
    } else {
      output.push({
        json: parsed.json || parsed
      });
    }

  } catch (error) {
    console.error('Parsing failed:', error.message);
  }
}

// 여러 아이템 반환
return output;
```
{: file='export_json.js'}

#### 설명
LLM의 출력값에서 json 구조를 파싱 및 가공을 통해 순수 json 형태의 데이터로 변환합니다. 


### 08. JSON 데이터 병합

![merge_json.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/merge_json.png)

| 노드 유형 | 노드 설명 | 등록 값 |
|--------|-------------------------------------------|---------------------------|
| Code  | LLM 응답 JSON 데이터 병합 | `merge_json.js` 코드 블럭 참고 |

```js
return [
  {
    json: {
      items: items.map(item => item.json)
    }
  }
];
```
{: file='merge_json.js'}

#### 설명 
n8n에서는 입력으로 들어간 값 만큼 노드가 반복으로 실행되기 때문에 한번의 작업으로 여러개의 json을 처리하기 위해서는 병합이 필요하다.


### 09. 메일 본문 작성

![mail_body.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/mail_body.png)

| 노드 유형 | 노드 설명 | 등록 값 |
|--------|-------------------------------------------|---------------------------|
| Code  | 메일 본문 작성 | `mail_body.js` 코드 블럭 참고 |

```js
const items = $json.items;

const html = items.map((item, index) => `
  <div style="margin-bottom: 30px;">
    <h2>${index + 1}. ${item.title}</h2>
    <p><strong>카테고리:</strong> ${item.categories0}, ${item.categories1}</p>
    <p>${item.content.replace(/\n/g, '<br>')}</p>
    <p><strong>링크:</strong> ${item.link}</p>
    <hr>
`).join('\n');

return [
  {
    json: {
      emailBody: html
    }
  }
];
```
{: file='mail_body.js'}

#### 설명

메일 본문을 그대로 전달하게 된다면 가독성이 떨어질 수 있기 때문에 HTML을 사용하여 가독성을 높여준다.
이때 주의사항으로는 Gmail 같은 경우 지원이 안되는 태그나 스타일을 사용하지 않도록 주의해야 한다.

### 10. 메일 전송

![gmail.png](/assets/img/posts/2025-06-11-n8n-rss-feed-automation/gmail.png)

| 노드 유형 | 노드 설명 | 등록 값 |
|--------|-------------------------------------------|---------------------------|
| Gmail  | 메일 전송 | `Message`, `Send` |

#### 설명

Gmail 노드를 사용하여 메일을 전송할 수 있다.

Google OAuth 방식이 아닌 API 방식만 지원되기 때문에 Google Cloud Platform에 접근해서 API[^3]를 활성화하고 키를 발급 받아야한다.


# 리뷰

처음 만들어본 n8n 워크플로우지만 만들면서 문제를 해결하는 부분 덕분에 기본적인 동작 구조를 이해하게 되었다.
완벽하지는 않았지만 내 생각대로 동작하는 워크플로우를 만들 수 있었다.


---
## Ref.
[^1]: [나만의 n8n 서버 만들기](https://mchlkim.github.io/posts/knowledge-n8n%EA%B5%AC%EC%B6%95/)
[^2]: [Gemini API](https://ai.google.dev/gemini-api/docs?hl=ko)
[^3]: [GCP Gmail API](https://console.cloud.google.com/marketplace/product/google/gmail.googleapis.com)