# 클라이언트 한글언어 강제 설정 가이드

Delight AI agent SDK에서 사용자 디바이스의 시스템 설정에 관계없이 한글 언어로 강제 설정하는 방법을 설명합니다.

---

## iOS

### Full-screen 모드에서 한글 설정

```swift
// 전체 화면 모드로 대화 화면 표시
AIAgentMessenger.presentConversation(
    aiAgentId: "YOUR_AI_AGENT_ID"
) { params in
    params.language = "ko-KR"  // 한글(한국) 설정
    params.countryCode = "KR"
    params.parent = self
    params.presentationStyle = .fullScreen
}
```

### 언어 코드 형식

- **IETF BCP 47** 형식을 따릅니다.
- **한국**: `ko-KR`
- **기타**: `en-US`, `ja-JP`, `zh-CN` 등

> **참고**: `countryCode`는 ISO 3166 형식 (예: "KR", "US", "JP")

---

## Android

### 개요

언어는 화면을 열 때 전달하는 `ConversationSettingsParams.language`로 지정합니다. 이 값은 두 곳에 적용됩니다.

1. **SDK 화면 UI 언어** — Messenger 화면(`MessengerActivity`)이 해당 언어의 로케일로 표시됩니다.
2. **대화 메타데이터** — 서버로 전달되어 AI 에이전트 응답 언어 결정에 사용됩니다.

---

### 전체 화면 모드에서 한글 설정

```kotlin
import com.sendbird.sdk.aiagent.messenger.model.ConversationSettingsParams
import com.sendbird.sdk.aiagent.messenger.ui.activity.MessengerActivity

startActivity(
    MessengerActivity.newIntentForConversation(
        context = this,
        aiAgentId = "YOUR_AI_AGENT_ID",
        conversationSettingsParams = ConversationSettingsParams(
            language = "ko-KR",  // 한글(한국) 강제 설정
            country = "KR",      // 옵션: 국가 코드
        ),
    )
)
```

대화 목록 화면도 동일합니다.

```kotlin
startActivity(
    MessengerActivity.newIntentForConversationList(
        context = this,
        aiAgentId = "YOUR_AI_AGENT_ID",
        conversationSettingsParams = ConversationSettingsParams(
            language = "ko-KR",
            country = "KR",
        ),
    )
)
```

---

### 언어 코드 형식

- `language`: **IETF BCP 47** 형식 — 한국어는 `ko-KR` (그 외 `en-US`, `ja-JP` 등)
- `country`: **ISO 3166** 형식 — `KR`, `US`, `JP` 등
- 기본값: `language`를 지정하지 않으면 디바이스 로케일(`Locale.getDefault()`)이 사용됩니다.

---

### SDK 내장 UI 언어

SDK 화면의 버튼/안내 문구 등 내장 문자열은 아래 언어의 번역을 포함합니다.

| 언어       | 코드 |
| ---------- | ---- |
| 영어(기본) | `en` |
| 한국어     | `ko` |
| 일본어     | `ja` |
| 독일어     | `de` |
| 스페인어   | `es` |
| 프랑스어   | `fr` |
| 힌디어     | `hi` |
| 이탈리아어 | `it` |
| 포르투갈어 | `pt` |
| 터키어     | `tr` |

- 목록에 없는 언어를 지정하면 UI 문자열은 기본값(영어)으로 표시됩니다.
- SDK 문자열을 앱에서 직접 바꾸고 싶다면, 앱의 `strings.xml`에 **같은 키**로 재정의하면 됩니다 (예: `aa_text_...` 키). 앱 리소스가 SDK 리소스보다 우선 적용됩니다.

---

### 주의사항

- **화면을 열 때마다 명시적으로 전달**: `language`는 화면 실행 시점에 적용됩니다. 한글 고정이 목적이라면 `MessengerActivity`를 여는 모든 곳에서 `language = "ko-KR"`을 일관되게 전달하세요.
- **AI 응답 언어**: 에이전트의 응답 언어는 대시보드의 에이전트 설정과 함께 동작합니다. `language`를 전달해도 기대와 다르게 응답한다면 대시보드 설정을 함께 확인하세요.

## 웹 (JavaScript / React)

### FixedMessenger 컴포넌트에서 한글 설정

React SDK에서는 `language` prop으로 한글 언어를 강제 설정할 수 있습니다.

```tsx
import { FixedMessenger } from "@sendbird/ai-agent-messenger-react";

function App() {
  return (
    <FixedMessenger
      appId="YOUR_APP_ID"
      aiAgentId="YOUR_AI_AGENT_ID"
      language="ko-KR" // 한글(한국) 설정
      countryCode="KR" // 옵션: 국가 코드
    />
  );
}
```

### 런타임에 언어 동적 변경

```tsx
import { useState } from "react";
import { FixedMessenger } from "@sendbird/ai-agent-messenger-react";

function App() {
  const [language, setLanguage] = useState("ko-KR");

  const handleLanguageChange = (newLanguage: string) => {
    setLanguage(newLanguage);
  };

  return (
    <>
      <button onClick={() => handleLanguageChange("ko-KR")}>한국어</button>
      <button onClick={() => handleLanguageChange("en-US")}>English</button>

      <FixedMessenger
        appId="YOUR_APP_ID"
        aiAgentId="YOUR_AI_AGENT_ID"
        language={language}
        countryCode={language === "ko-KR" ? "KR" : "US"}
      />
    </>
  );
}
```

### AgentProviderContainer 사용 시

```tsx
import {
  AgentProviderContainer,
  Conversation,
} from "@sendbird/ai-agent-messenger-react";

function App() {
  return (
    <AgentProviderContainer
      appId="YOUR_APP_ID"
      language="ko-KR" // 한글 설정
      countryCode="KR"
    >
      <Conversation aiAgentId="YOUR_AI_AGENT_ID" />
    </AgentProviderContainer>
  );
}
```

---

## CDN 버전 (JavaScript)

### HTML에서 직접 사용

```html
<script src="https://cdn.sendbird.com/ai-agent-messenger/v1/ai-agent-messenger.js"></script>

<div id="app"></div>

<script>
  const container = document.getElementById("app");

  SendbirdAIAgent.init({
    appId: "YOUR_APP_ID",
    aiAgentId: "YOUR_AI_AGENT_ID",
    language: "ko-KR", // 한글(한국) 설정
    countryCode: "KR",
  });

  SendbirdAIAgent.render(container);
</script>
```

---

## 지원되는 언어 코드

SDK에서 지원하는 언어 코드는 다음과 같습니다:

| 언어         | 코드    |
| ------------ | ------- |
| 한국어       | `ko-KR` |
| 영어         | `en-US` |
| 일본어       | `ja-JP` |
| 중국어(간체) | `zh-CN` |
| 중국어(번체) | `zh-TW` |
| 독일어       | `de-DE` |
| 스페인어     | `es-ES` |
| 프랑스어     | `fr-FR` |
| 이탈리아어   | `it-IT` |
| 포르투갈어   | `pt-PT` |
| 터키어       | `tr-TR` |
| 힌디어       | `hi-IN` |

---

## 주의사항

### 시스템 언어와의 충돌

- **iOS/Android**: 설정한 언어로 UI가 표시되지만, 시스템 언어 설정이 변경되면 재시작 시 영향을 받을 수 있습니다.
- **해결책**: 앱 시작 시마다 명시적으로 언어를 설정하세요.

### 미지원 언어

- 위 표에 없는 언어는 기본값(영어)으로 표시됩니다.
- 커스텀 문자열 설정으로 다른 언어 지원 가능합니다 (각 플랫폼의 다국어 가이드 참고).

### AI Agent 응답 언어

- `language` 설정은 UI 텍스트에만 영향을 미칩니다.
- AI Agent의 응답 언어는 별도로 `context` 객체에서 설정해야 할 수 있습니다.

```swift
// iOS 예시
AIAgentMessenger.presentConversation(aiAgentId: "YOUR_AI_AGENT_ID") { params in
    params.language = "ko-KR"
    params.context = ["language": "ko"]  // AI 응답 언어
    params.parent = self
    params.presentationStyle = .fullScreen
}
```

```kotlin
// Android 예시
MessengerLauncher(this, "YOUR_AI_AGENT_ID", LauncherSettingsParams(
    language = "ko-KR",
    context = mapOf("language" to "ko")  // AI 응답 언어
)).attach()
```

---

## 추가 리소스

- [iOS 다국어 지원 가이드](./ios/MULTILANGUAGE.md)
- [Android 다국어 지원 가이드](./android/docs/MULTILANGUAGE.md)
- [React 다국어 지원 가이드](./js/react/docs/MULTILANGUAGE.md)
