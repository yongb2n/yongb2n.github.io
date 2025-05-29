---
title: Firebase Cloud Functions로 OpenAI 프록시 서버 만들기
date: 2025-05-22 22:01:32 +09:00
categories: [Firebase, Cloud Functions]
tags: [openai, proxy, postman, typescript]
---
## 프록시 서버를 만든 이유
PECSPERT 팀에서 자폐 스펙트럼 아동을 위한 AI 문장 완성 기능을 개발하며 OpenAI API를 활용하고 있습니다. 

초기에는 React Native 프론트엔드에서 직접 API를 호출하려 했지만, 보안상의 위험을 발견했습니다.

API 키가 클라이언트 측에서 노출될 경우 악의적인 사용으로 이어질 수 있고, 부적절한 요금 청구가 발생할 가능성도 있었기 때문에, 이를 해결하고자 프록시 서버를 구축하기로 했습니다.

프록시 서버 구현에는 `Firebase Cloud Functions`를 선택했습니다. 

PECSPERT 팀은 아직 별도의 백엔드 서버를 구축하지 않은 상태로, 대부분의 필요한 기능(예: 인증, 데이터베이스, 호스팅)을 Firebase에서 처리하고 있습니다. 

이러한 환경적 제약과 팀의 기존 워크플로우를 고려했을 때, Firebase Cloud Functions는 빠르게 구현할 수 있는 최적의 선택이었습니다. 

이 중개 서버는 API 요청을 안전하게 관리하고, 응답 형식을 제어하며, 향후 파인튜닝 전 필터링 로직을 추가할 기반을 제공합니다.

## 어떻게 구현했는지
다음 단계들로 `Firebase Cloud Functions`를 활용해 프록시 서버를 구현했습니다. 

각 단계를 구체적으로 설명하겠습니다.

### 1. Firebase Functions 설정

프로젝트를 초기화하기 위해 `firebase init functions` 명령어를 사용해 새로운 Functions 환경을 설정했습니다. 

TypeScript를 선택해 타입 안전성을 확보하고, pnpm을 사용 중이므로 `functions/package.json`을 다음과 같이 구성했습니다.

```json
{
  "scripts": {
    "lint": "eslint .",
    "build": "tsc",
  },
  "dependencies": {
    "@google-cloud/functions-framework": "^4.0.0",
    "openai": "^4.96.2",
  }
}
```

`functions/tsconfig.json`은 `outDir: "lib"`을 지정해 빌드 결과를 lib/ 폴더에 출력하도록 했습니다. 

<img width="397" alt="스크린샷 2025-05-30 오전 12 17 22" src="https://github.com/user-attachments/assets/9e907674-087f-42e1-bffe-69fdc2455c5e" />

이는 배포 시 깔끔한 파일 구조를 유지하는 데 유용했습니다.

### 2. Secret 관리
API 키 노출을 방지하기 위해 `firebase functions:secrets:set OPENAI_API_KEY` 명령어로 시크릿을 등록했습니다. 

보안 강화를 위해 `.firebaserc`, `.firebase/debug.log` 등 민감한 파일을 `firebase.json`의 무시 목록에 추가했습니다.

```json
{
  "functions": [
    {
      "source": "functions",
      "predeploy": [
        "npm --prefix \"$RESOURCE_DIR\" run lint",
        "npm --prefix \"$RESOURCE_DIR\" run build"
      ],
      "ignore": [
        "node_modules",
        ".git",
        "firebase-debug.log",
        "firebase-debug.*.log",
        "*.local"
      ]
    }
  ]
}
```

### 3. 실제 코드 구성
프록시 함수는 `defineSecret`와 `onRequest`를 활용해 구현했습니다. 

아래는 핵심 구조로, PECSPERT 앱의 요구사항(자폐 스펙트럼 아동을 위한 자연스러운 문장 생성)을 반영한 코드입니다.

```ts
import { defineSecret } from "firebase-functions/params";
import { onRequest } from "firebase-functions/v2/https";

import OpenAI from "openai";

import { correctSentencePromptWithRules } from './services/llm/correctSentencePromptWithRules';
import { SentenceGenerationOptions } from './types/llmTypes';

const openaiApiKey = defineSecret("OPENAI_API_KEY");

export const proxyOpenAI = onRequest(
  { secrets: [openaiApiKey] },
  async (req, res) => {
    // POST 요청만 허용, 다른 메서드는 405 오류 반환
    if (req.method !== "POST") {
      res.status(405).send("Method Not Allowed");
      return;
    }

    try {
      const { cardTexts } = req.body as SentenceGenerationOptions;

      // cardTexts가 유효한 배열인지 검증
      if (!Array.isArray(cardTexts) || cardTexts.length === 0) {
        res.status(400).json({ error: "cardTexts is invalid." });
        return;
      }

      // OpenAI 클라이언트 초기화
      const openai = new OpenAI({ apiKey: openaiApiKey.value() /* 시크릿 키는 환경 변수로 안전하게 관리 */ });

      // 단어 배열을 기반으로 프롬프트 생성
      const prompt = correctSentencePromptWithRules({ cardTexts });

      // OpenAI API 호출, 모델은 파인튜닝한 모델을 사용
      const completion = await openai.chat.completions.create({
        model: "ft:gpt-4o-2024-08-06:pecspert-v1", // 파인튜닝 모델, 보안상 세부 생략
        messages: [
          {
            role: "system",
            content: `
                      당신은 PECSPERT 앱에서 사용하는 AI 문장 생성 전문가입니다.
                      PECSPERT는 자폐 스펙트럼 아동이 그림 카드를 통해 쉽고 자연스럽게 말할 수 있도록 돕는 AAC(보완대체의사소통) 서비스입니다.
                      ... 생략
                     `.trim(),
          },
          {
            role: "user",
            content: prompt,
          },
        ],
        temperature: 0.7, 
        max_tokens: 100,
      });

      // 응답 처리 및 반환
      const result = completion.choices[0].message?.content ?? null;
      if (!result) {
        res.status(502).json({ error: "No response." });
        return;
      }
      res.json({ result: JSON.parse(result) });
    } catch (e) {
      // 상세 에러 로그는 생략
      res.status(500).json({ error: "An error occurred while calling OpenAI." });
    }
  }
);
```

이 코드는 사용자가 보낸 단어 배열(예: ["사과", "먹다"])을 받아 자연스러운 문장("사과를 먹어요")으로 변환합니다.

`correctSentencePromptWithRules` 함수는 단어 배열을 기반으로 프롬프트를 생성하며, 시스템 메시지는 PECSPERT의 요구사항(친숙한 구어체, 조사 교정 등)을 명확히 전달합니다. 

응답은 JSON 형식으로 반환되어 React Native 앱에서 쉽게 처리할 수 있습니다.

### 4. 배포 시 고려사항
배포 과정에서 `@google-cloud/functions-framework`가 누락되면 실패하므로, 의존성을 철저히 확인했습니다. 

또한 IAM 권한 문제로 여러 번 에러가 발생했는데, `Cloud Functions Admin`과 `Secret Manager Accessor` 권한을 부여해 문제를 해결했습니다.

배포가 성공하면, 아래와 같은 엔드포인트를 확인할 수 있습니다.

![스크린샷 2025-05-29 오후 11 23 10](https://github.com/user-attachments/assets/62845a2d-f3ee-4337-9d97-f2764e55573c)

### 5. 배포 후 테스트
앱에서 프록시를 호출하는 코드는 다음과 같습니다.

```ts
const response = await fetch("https://us-central1-***.cloudfunctions.net/***", { /* 배포된 실제 URL, 보안상 세부 생략*/
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ cardTexts: ['사과', '먹다'] }),
});
const result = await response.json();
console.log(result); // "사과를 먹어요" 등의 응답 확인
```

또한 `Postman`으로 테스트 결과, 정상 응답을 통해 서버 동작을 검증했습니다.

<img width="1082" alt="스크린샷 2025-05-29 오후 11 52 35" src="https://github.com/user-attachments/assets/cdf1d059-e346-4c25-a358-55eaefb27d62" />

## 마무리하며
- API 키 보호: 클라이언트 측에서 API 키가 노출되지 않아 보안성이 높아졌습니다.
- 응답 형식 제어: LLM의 임의 응답을 방지하고, JSON 형식으로 제한해 안정성을 확보했습니다.
- 파인튜닝 준비: 응답이 원하는 형식이 아닐 때 필터링하거나, 교정 로직을 삽입해 파인튜닝 전 단계로 활용 가능합니다.

구현 과정에서 가장 큰 도전은 IAM 권한 문제로 인한 배포 실패였습니다. 초기 배포 시 `"Permission Denied"` 에러가 발생했는데, 이는 `Cloud Functions Admin` 권한 부족 때문이었습니다. 

Firebase 문서를 참고해 필요한 권한(`roles/cloudfunctions.admin`, `roles/secretmanager.secretAccessor`)을 추가하고, 권한 설정을 조정한 후 문제를 해결했습니다.

또한, 소량 데이터(30개)로 파인튜닝된 모델을 사용하면서 과적합 우려가 있었지만, 테스트 결과 98%의 조사 교정 정확도를 확인하며 안정성을 검증했습니다. 

이 경험은 백엔드 인프라 설계에 중요한 교훈을 주었고, 데이터 확장을 통해 더 나은 성능을 기대할 수 있을 것이라고 생각합니다.






