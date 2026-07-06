# 폰트 커스텀 가이드

## iOS

### 기본 폰트 커스텀

iOS SDK에서는 `SBAFontSet`을 통해 폰트를 커스텀할 수 있습니다.

```swift
// 커스텀 폰트 패밀리 설정 (전체 폰트에 일괄 적용)
SBAFontSet.fontFamily = "{CUSTOM_FONT_FAMILY}"

// nil 이면 시스템 폰트를 사용합니다 (기본값)
SBAFontSet.fontFamily = nil
```

### 스타일별 폰트 커스텀

폰트 패밀리 일괄 적용 대신, 스타일별로 개별 폰트를 지정할 수도 있습니다.

```swift
SBAFontSet.h1 = UIFont.systemFont(ofSize: 20, weight: .bold)
SBAFontSet.h2 = UIFont.systemFont(ofSize: 18, weight: .semibold)
SBAFontSet.body1 = UIFont.systemFont(ofSize: 16, weight: .regular)
SBAFontSet.button1 = UIFont.systemFont(ofSize: 16, weight: .bold)
SBAFontSet.caption1 = UIFont.systemFont(ofSize: 12, weight: .medium)
```

### 테마 폰트 커스텀 상태 확인

테마의 개별 폰트는 `SBAThemeFont` 프로퍼티 래퍼로 관리됩니다. `$` 접두사로 접근해서 커스텀 여부 확인과 초기화를 할 수 있습니다.

```swift
// 커스텀 폰트 적용 여부 확인
if SBATheme.conversation.header.title.$titleFont.isCustomized {
    // 커스텀 폰트가 적용됨
}

// 기본 폰트로 재설정
SBATheme.conversation.header.title.$titleFont.resetToDefault()
```

### 유의사항

- 앱 번들에 포함된 폰트만 사용할 수 있습니다 (Info.plist의 `UIAppFonts` 등록 필요).
- `fontFamily`에 넣는 이름은 실제 폰트의 PostScript 이름과 일치해야 합니다.

---

## Android

### 개요

SDK의 모든 텍스트는 **텍스트 스타일 리소스**(`H1`, `Body1` 등)를 통해 렌더링됩니다. 커스텀 폰트 적용 방법은 두 가지입니다.

| 방법                                     | 적용 범위                       | 난이도 |
| ---------------------------------------- | ------------------------------- | ------ |
| A. SDK 스타일 리소스 오버라이드          | SDK 전체 텍스트 일괄 적용       | 쉬움   |
| B. 테마 Provider로 `TextAppearance` 교체 | 특정 컴포넌트 텍스트만 선택 적용 | 중간   |

> **앱 테마(`android:fontFamily`) 전역 방식이 적용되지 않는 이유**
> 일반적으로 앱 전체 폰트는 Application 테마에 `android:fontFamily`를 지정해 적용하지만, 이 방식은 SDK 화면에는 동작하지 않습니다.
> 1. `MessengerActivity`는 SDK manifest에 자체 테마가 지정되어 있어 앱 테마의 fontFamily가 전달되지 않고,
> 2. SDK의 모든 텍스트는 `fontFamily`가 명시된 TextAppearance 스타일(아래 표)로 그려지므로 테마 기본 폰트를 덮어씁니다.
>
> 따라서 SDK 화면 전체에 커스텀 폰트를 적용하려면 **방법 A(스타일 리소스 오버라이드)** 를 사용하세요.

---

### 사전 준비: 폰트 리소스 추가

1. `res/font/` 디렉토리 생성 (없는 경우)
2. `.ttf` 또는 `.otf` 파일 추가 (예: `res/font/noto_sans_kr_regular.ttf`, `res/font/noto_sans_kr_bold.ttf`)
3. 필요 시 font-family XML로 굵기별 파일을 묶음

```xml
<!-- res/font/noto_sans_kr.xml -->
<?xml version="1.0" encoding="utf-8"?>
<font-family xmlns:app="http://schemas.android.com/apk/res-auto">
    <font app:fontStyle="normal" app:fontWeight="400" app:font="@font/noto_sans_kr_regular" />
    <font app:fontStyle="normal" app:fontWeight="700" app:font="@font/noto_sans_kr_bold" />
</font-family>
```

---

### 방법 A. SDK 스타일 리소스 오버라이드 (전체 일괄 적용)

SDK는 아래 12개의 텍스트 스타일을 사용합니다. **앱 모듈에 같은 이름의 스타일을 정의하면 SDK의 스타일을 덮어씁니다** (Android 리소스 병합 규칙 — 앱 리소스가 라이브러리 리소스보다 우선).

SDK 기본값:

| 스타일      | fontFamily        | textStyle | textSize |
| ----------- | ----------------- | --------- | -------- |
| `H1`        | sans-serif        | bold      | 18sp     |
| `H2`        | sans-serif        | bold      | 16sp     |
| `Subtitle1` | sans-serif-medium | normal    | 16sp     |
| `Subtitle2` | sans-serif        | normal    | 16sp     |
| `Body1`     | sans-serif        | normal    | 16sp     |
| `Body2`     | sans-serif-medium | normal    | 14sp     |
| `Body3`     | sans-serif        | normal    | 14sp     |
| `Button`    | sans-serif        | bold      | 14sp     |
| `Caption1`  | sans-serif        | bold      | 12sp     |
| `Caption2`  | sans-serif        | normal    | 12sp     |
| `Caption3`  | sans-serif        | bold      | 11sp     |
| `Caption4`  | sans-serif        | normal    | 11sp     |

앱의 `res/values/styles.xml`에 같은 이름으로 재정의합니다. **`fontFamily`만 바꾸고 `textStyle`/`textSize`는 SDK 기본값을 유지**하는 것을 권장합니다.

```xml
<!-- res/values/styles.xml (앱 모듈) -->
<resources>
    <style name="H1">
        <item name="android:fontFamily">@font/noto_sans_kr</item>
        <item name="android:textStyle">bold</item>
        <item name="android:textSize">18sp</item>
    </style>

    <style name="Body1">
        <item name="android:fontFamily">@font/noto_sans_kr</item>
        <item name="android:textStyle">normal</item>
        <item name="android:textSize">16sp</item>
    </style>

    <!-- 나머지 스타일(H2, Subtitle1/2, Body2/3, Button, Caption1~4)도 동일한 방식으로 재정의 -->
</resources>
```

> **주의**: `H1`, `Body1` 같은 이름은 범용적이라 **앱에서 이미 같은 이름의 스타일을 쓰고 있다면 충돌**합니다. 이 경우 앱 자체 스타일 이름을 변경하거나, 방법 B를 사용하세요.

---

### 방법 B. 테마 Provider로 `TextAppearance` 교체 (부분 적용)

특정 컴포넌트의 폰트만 바꾸고 싶다면 `AIAgentThemeProviders`로 테마를 교체하면서 원하는 `TextAppearance`의 `fontStyleRes`만 커스텀 스타일로 지정합니다.

먼저 앱에 TextAppearance용 스타일을 정의합니다.

```xml
<!-- res/values/styles.xml (앱 모듈) -->
<resources>
    <style name="MyMessengerTitle">
        <item name="android:fontFamily">@font/noto_sans_kr</item>
        <item name="android:textStyle">bold</item>
        <item name="android:textSize">18sp</item>
    </style>

    <style name="MyMessengerBody">
        <item name="android:fontFamily">@font/noto_sans_kr</item>
        <item name="android:textSize">16sp</item>
    </style>
</resources>
```

그리고 **Messenger 화면을 열기 전에** 테마 Provider를 교체합니다. Light/Dark 두 모드를 모두 지원한다면 둘 다 설정하세요.

```kotlin
import com.sendbird.sdk.aiagent.messenger.model.theme.DarkTheme
import com.sendbird.sdk.aiagent.messenger.model.theme.LightTheme
import com.sendbird.sdk.aiagent.messenger.providers.AIAgentThemeProviders
import com.sendbird.sdk.aiagent.messenger.providers.ThemeProvider

AIAgentThemeProviders.light = ThemeProvider {
    LightTheme().apply {
        // 대화 화면 헤더 타이틀
        conversation.header.titleTextAppearance.fontStyleRes = R.style.MyMessengerTitle
        // 상대(에이전트) 메시지 본문
        conversation.messageList.messageStyles.otherUserMessageStyle.textAppearance.fontStyleRes = R.style.MyMessengerBody
        // 내 메시지 본문
        conversation.messageList.messageStyles.myUserMessageStyle.textAppearance.fontStyleRes = R.style.MyMessengerBody
    }
}

AIAgentThemeProviders.dark = ThemeProvider {
    DarkTheme().apply {
        conversation.header.titleTextAppearance.fontStyleRes = R.style.MyMessengerTitle
        conversation.messageList.messageStyles.otherUserMessageStyle.textAppearance.fontStyleRes = R.style.MyMessengerBody
        conversation.messageList.messageStyles.myUserMessageStyle.textAppearance.fontStyleRes = R.style.MyMessengerBody
    }
}
```

테마 트리에서 커스텀 가능한 주요 `TextAppearance` 위치:

- `conversation.header.titleTextAppearance` — 대화 화면 헤더 타이틀
- `conversation.messageList.messageStyles.myUserMessageStyle` / `otherUserMessageStyle` — 메시지 본문
- `conversation.messageList.messageStyles.suggestedRepliesStyle`, `csatMessageStyle`, `citationStyle`, `messageFeedbackStyle` 등 — 부가 UI
- `conversationList.*` — 대화 목록 화면

> 전체 목록은 `MessengerTheme` / `MessageStyles` 인터페이스를 참고하세요.

---

### 주의사항

- **적용 시점**: 스타일 리소스 오버라이드(방법 A)는 빌드 시 적용되므로 별도 코드가 필요 없습니다. 테마 Provider(방법 B)는 반드시 **Messenger 화면 생성 전에** 등록하세요.
- **파일 크기**: 한글 폰트는 용량이 크므로 필요한 가중치(weight)만 포함하는 것을 권장합니다.
- **Light/Dark**: 방법 B 사용 시 두 테마 모드에 동일하게 적용해야 모드 전환 시 폰트가 유지됩니다.

## 웹 (JavaScript/React)

### CSS를 통한 폰트 커스텀

React SDK에서는 CSS를 통해 폰트를 커스텀할 수 있습니다.

```css
/* 글로벌 폰트 설정 */
body,
[class*="AIAgent"],
[class*="Sendbird"] {
  font-family: "Your Custom Font", "NotoSansCJKkr", sans-serif;
}

/* 특정 컴포넌트 폰트 커스텀 */
.sbai-message__text {
  font-family: "Your Custom Font", sans-serif;
  font-size: 14px;
}

.sbai-header__title {
  font-family: "Your Custom Font", sans-serif;
  font-weight: 600;
}
```

### 컴포넌트 커스텀을 통한 폰트 설정

템플릿 기반 레이아웃 시스템을 사용하여 커스텀 컴포넌트에 폰트를 적용할 수 있습니다.

```tsx
import {
  IncomingMessageLayout,
  IncomingMessageProps,
} from "@sendbird/ai-agent-messenger-react";

const CustomMessageBody = (props: IncomingMessageProps) => {
  return (
    <div style={{ fontFamily: "Your Custom Font", fontSize: "14px" }}>
      {props.message}
    </div>
  );
};

export const CustomMessage = () => {
  return <IncomingMessageLayout.MessageBody component={CustomMessageBody} />;
};
```

### 웹 폰트 import

Google Fonts나 다른 CDN의 웹 폰트를 사용할 수 있습니다.

```html
<!-- HTML head에 추가 -->
<link
  href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;600;700&display=swap"
  rel="stylesheet"
/>
```

```css
body {
  font-family: "Noto Sans KR", sans-serif;
}
```

---

## 주의사항

- **iOS**: 앱에 포함된 폰트 파일만 사용 가능 (Info.plist에 등록 필요)
- **Android**: 파일 크기를 고려하여 필요한 가중치만 포함
- **웹**: 폰트 파일 로딩 성능을 고려하여 최적화 필요
- **다국어**: 한글, 중국어, 일본어 등 동아시아 문자는 폰트 파일이 커질 수 있음
