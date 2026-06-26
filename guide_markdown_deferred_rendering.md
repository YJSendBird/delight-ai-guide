# Markdown 이미지 및 링크 렌더링 타이밍 가이드

이 가이드는 AI Agent가 메시지를 스트리밍할 때, Markdown 이미지와 링크가 완전히 로드될 때까지 숨기는 방법을 설명합니다.

---

## 개요

**문제:**

- AI Agent의 응답이 스트리밍될 때, 완성되지 않은 Markdown 토큰(예: `![image](`)이 잠시 노출됨
- 불완전한 마크다운 형식이 사용자에게 보여짐
- 최종 렌더링 완료 전에 부분적인 콘텐츠가 깜빡임

**해결책:**

- `deferredMarkdownElements` 설정을 사용하여 완성되지 않은 마크다운 토큰을 숨길 수 있음
- 이미지와 링크가 완전히 로드될 때까지 숨겼다가 완성되면 표시
- 사용자 경험 개선

---

## Web (React)

### 설정 방법

```tsx
import { FixedMessenger } from "@sendbird/ai-agent-messenger-react";

export function App() {
  return (
    <FixedMessenger
      appId="YOUR_APP_ID"
      aiAgentId="YOUR_AI_AGENT_ID"
      config={{
        conversation: {
          list: {
            // 이미지와 링크가 완성될 때까지 숨김
            deferredMarkdownElements: ["image", "link"],
          },
        },
      }}
    />
  );
}
```

### 옵션

| 옵션          | 설명                                                    |
| ------------- | ------------------------------------------------------- |
| `'image'`     | 마크다운 이미지 `![alt](url)` 형식이 완성될 때까지 숨김 |
| `'link'`      | 마크다운 링크 `[text](url)` 형식이 완성될 때까지 숨김   |
| `[]` (기본값) | 모든 마크다운 요소를 즉시 렌더링 (이전 동작)            |

### 예제: 이미지만 숨기기

```tsx
<FixedMessenger
  appId="YOUR_APP_ID"
  aiAgentId="YOUR_AI_AGENT_ID"
  config={{
    conversation: {
      list: {
        // 이미지만 완성될 때까지 숨김
        deferredMarkdownElements: ["image"],
      },
    },
  }}
/>
```

### 예제: AgentProviderContainer 사용

```tsx
import {
  AgentProviderContainer,
  Conversation,
} from "@sendbird/ai-agent-messenger-react";

export function App() {
  return (
    <AgentProviderContainer
      appId="YOUR_APP_ID"
      aiAgentId="YOUR_AI_AGENT_ID"
      config={{
        conversation: {
          list: {
            deferredMarkdownElements: ["image", "link"],
          },
        },
      }}
    >
      <Conversation />
    </AgentProviderContainer>
  );
}
```

### 스트리밍 동작

```
스트리밍 진행 중:
AI Agent: "여기 사진이 있습니다 ![a     <- 숨겨짐
AI Agent: "여기 사진이 있습니다 ![alt-text](ht   <- 숨겨짐
AI Agent: "여기 사진이 있습니다 ![alt-text](https://example.com/image.jpg)   <- 표시됨!

[완성된 이미지가 렌더링됨]
```

---

## Android

### 설정 방법

```kotlin
import com.sendbird.sdk.aiagent.messenger.AIAgentMessenger
import com.sendbird.sdk.aiagent.messenger.model.DeferredMarkdownElement

// 앱 초기화 시 설정
AIAgentMessenger.config.conversation.list.deferredMarkdownElements =
    setOf(
        DeferredMarkdownElement.IMAGE,
        DeferredMarkdownElement.LINK
    )
```

또는 Launcher 시작 시:

```kotlin
import com.sendbird.sdk.aiagent.messenger.LauncherSettingsParams

MessengerLauncher(
    context,
    "YOUR_AI_AGENT_ID",
    LauncherSettingsParams()
).attach()
```

### 옵션

```kotlin
// DeferredMarkdownElement enum
enum class DeferredMarkdownElement {
    IMAGE,  // 마크다운 이미지 형식 숨김
    LINK    // 마크다운 링크 형식 숨김
}
```

### 예제: 이미지만 숨기기

```kotlin
AIAgentMessenger.config.conversation.list.deferredMarkdownElements =
    setOf(DeferredMarkdownElement.IMAGE)
```

### 예제: 전체 화면 Messenger 시작

```kotlin
startActivity(
    MessengerActivity.newIntentForConversation(
        context = this,
        aiAgentId = "YOUR_AI_AGENT_ID"
    )
)

// 위의 설정이 적용됨
```

---

## React Native

### 설정 방법

```tsx
import {
  AIAgentProviderContainer,
  FixedMessenger,
} from "@sendbird/ai-agent-messenger-react-native";

export function App() {
  return (
    <AIAgentProviderContainer
      appId="YOUR_APP_ID"
      aiAgentId="YOUR_AI_AGENT_ID"
      userSessionInfo={userSessionInfo}
      nativeModules={nativeModules}
      config={{
        conversation: {
          list: {
            // 이미지와 링크가 완성될 때까지 숨김
            deferredMarkdownElements: ["image", "link"],
          },
        },
      }}
    >
      <FixedMessenger />
    </AIAgentProviderContainer>
  );
}
```

### 옵션

Web과 동일:

- `'image'`: 마크다운 이미지 숨김
- `'link'`: 마크다운 링크 숨김
- `[]`: 모든 요소 즉시 렌더링

### 예제: 커스텀 네비게이션

```tsx
import {
  AIAgentProviderContainer,
  Conversation,
} from "@sendbird/ai-agent-messenger-react-native";
import { NavigationContainer } from "@react-navigation/native";

export function App() {
  return (
    <AIAgentProviderContainer
      appId="YOUR_APP_ID"
      aiAgentId="YOUR_AI_AGENT_ID"
      userSessionInfo={userSessionInfo}
      nativeModules={nativeModules}
      config={{
        conversation: {
          list: {
            deferredMarkdownElements: ["image", "link"],
          },
        },
      }}
    >
      <NavigationContainer>
        {/* 네비게이션 구조 */}
        <Conversation />
      </NavigationContainer>
    </AIAgentProviderContainer>
  );
}
```

---

## iOS

**참고:** iOS SDK의 경우 아직 공식적인 `deferredMarkdownElements` 설정이 문서화되지 않았습니다.

iOS SDK 업그레이드나 새로운 기능 릴리스를 확인하려면 [iOS CHANGELOG](./ios/CHANGELOG.md)를 참고하세요.

현재는 기본적인 마크다운 렌더링이 지원되며, 향후 업데이트에서 이 기능이 추가될 수 있습니다.

---

## 사용 시나리오

### 시나리오 1: 긴 콘텐츠 응답

```
사용자: "이 제품의 상세 사진을 보여줄래?"

AI Agent 응답 스트리밍:
"다음은 제품 이미지입니다:

![Product Main View](https://api.example.com/images/product-1.jpg)
![Product Side View](https://api.example.com/images/product-2.jpg)
![Product Back View](https://api.example.com/images/product-3.jpg)

그리고 자세한 설명 링크:
[전체 사양 보기](https://example.com/specs)
[고객 리뷰](https://example.com/reviews)"

// deferredMarkdownElements: ['image', 'link'] 설정시:
// 1. AI가 이미지 마크다운을 작성하는 동안 숨겨짐
// 2. 모든 이미지 URL이 완성되면 렌더링 시작
// 3. 링크도 동일한 방식으로 처리됨
```

### 시나리오 2: 링크만 필요한 경우

```tsx
// 이미지는 많이 사용하지 않고 링크만 빨리 표시하고 싶은 경우
config={{
  conversation: {
    list: {
      deferredMarkdownElements: ['link'],  // 링크만 지연
    },
  },
}}
```

---

## 성능 고려사항

| 설정                | 장점                                         | 단점                        |
| ------------------- | -------------------------------------------- | --------------------------- |
| `['image', 'link']` | 깔끔한 사용자 경험, 불완전한 마크다운 미노출 | 이미지/링크 표시 약간 지연  |
| `['image']`         | 이미지만 완성 대기, 링크는 즉시 표시         | 링크는 부분 노출 가능       |
| `[]` (기본값)       | 최빠른 초기 렌더링                           | 불완전한 마크다운 일시 노출 |

### 추천 설정

- **쇼핑 앱**: `['image', 'link']` - 이미지가 중요하므로 완전히 로드될 때까지 대기
- **정보 제공 앱**: `['link']` - 링크만 완전히 로드될 때까지 숨김
- **빠른 응답 우선**: `[]` - 모든 요소 즉시 표시

---

## 호환성

| 플랫폼       | 최소 버전                   |
| ------------ | --------------------------- |
| Web (React)  | v1.31.0 이상                |
| Android      | v1.15.0 이상                |
| React Native | v1.19.0 이상                |
| iOS          | 미지원 (추후 업데이트 예정) |

---

## 마이그레이션 노트

### 이전 설정 (Deprecated)

```tsx
// 이전 방식 (v1.30.0 이하)
config={{
  conversation: {
    list: {
      markdownImageRenderMode: 'complete-only',
      markdownLinkRenderMode: 'complete-only'
    }
  }
}}
```

### 새로운 설정 (v1.31.0+)

```tsx
// 새로운 방식 (v1.31.0+)
config={{
  conversation: {
    list: {
      deferredMarkdownElements: ['image', 'link']
    }
  }
}}
```

**변경 이유:**

- 더 명확한 API
- 이미지와 링크를 동시에 제어
- 향후 다른 마크다운 요소 추가 가능성

---

## 문제 해결

### Q: 이미지가 표시되지 않습니다.

**A:** 다음을 확인하세요:

1. SDK 버전이 v1.31.0+ (React), v1.15.0+ (Android), v1.19.0+ (React Native)인지 확인
2. `deferredMarkdownElements` 설정이 올바르게 되어 있는지 확인
3. 이미지 URL이 유효한지 확인

### Q: 스트리밍 중 깜빡임이 있습니다.

**A:** 이는 정상적인 동작입니다:

- 마크다운 형식이 완성될 때까지 숨겨짐
- 완성되면 즉시 렌더링됨
- 불완전한 형식이 노출되는 것을 방지

### Q: 링크는 표시되는데 이미지는 안 보입니다.

**A:** `deferredMarkdownElements`에 `'image'`만 포함되어 있는지 확인하세요:

```tsx
// 잘못된 설정
deferredMarkdownElements: ["link"]; // 이미지 미포함

// 올바른 설정
deferredMarkdownElements: ["image", "link"];
```

---

## 관련 가이드

- [폰트 커스텀 가이드](./guide_font.md)
- [클라이언트 한글언어 강제 설정 가이드](./guide_language.md)
- [채팅창 상단에 상품 배너 다는 가이드](./guide_product_banner.md)

---

## 참고 자료

- [React CHANGELOG v1.31.0](https://github.com/sendbird/delight-ai-agent/blob/main/js/react/CHANGELOG.md)
- [Android CHANGELOG v1.15.0](https://github.com/sendbird/delight-ai-agent/blob/main/android/CHANGELOGS.md)
- [React Native CHANGELOG v1.19.0](https://github.com/sendbird/delight-ai-agent/blob/main/js/react-native/CHANGELOG.md)
