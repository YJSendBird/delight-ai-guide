# 폰트 커스텀 가이드

## iOS

### 기본 폰트 커스텀

iOS SDK에서는 `SBAFontSet`을 통해 폰트를 커스텀할 수 있습니다.

```swift
// 커스텀 폰트 패밀리 설정
SBAFontSet.fontFamily = "YourCustomFont"

// 예시: 시스템 폰트 사용
SBAFontSet.fontFamily = ".SFUI-Regular"

// 예시: 커스텀 폰트 파일 사용 (앱에 포함된 경우)
SBAFontSet.fontFamily = "NotoSansCJKkr-Regular"
```

### 폰트 커스텀 상태 확인

`SBAThemeFont` 인터페이스를 통해 폰트 커스텀 상태를 관리할 수 있습니다.

```swift
// 커스텀 폰트 적용 여부 확인
if SBAFontSet.isCustomized {
    // 커스텀 폰트가 적용됨
}

// 기본 폰트로 재설정
SBAFontSet.resetToDefault()
```

### 언어별 최적화

iOS SDK는 언어별로 자동으로 폰트와 레이아웃을 조정합니다. 다국어 지원 시 별도의 폰트 설정이 필요할 수 있습니다.

---

## Android

### 테마를 통한 커스텀

Android SDK에서는 `MessengerTheme`을 통해 UI 요소를 커스텀할 수 있습니다.

```kotlin
// MessengerTheme 커스텀 예시
AIAgentMessenger.setTheme { theme ->
    // 테마 속성 커스텀
    // 색상, 배경 등 설정 가능
}
```

### 스타일 리소스를 통한 폰트 설정

Android에서는 `res/values/styles.xml` 또는 `res/values/themes.xml`을 통해 폰트를 정의할 수 있습니다.

```xml
<!-- res/values/styles.xml -->
<resources>
    <!-- 커스텀 텍스트 스타일 정의 -->
    <style name="CustomTextStyle">
        <item name="fontFamily">@font/custom_font</item>
        <item name="android:textSize">14sp</item>
    </style>
</resources>
```

### 커스텀 폰트 파일 추가

1. `res/font/` 디렉토리 생성 (없는 경우)
2. `.ttf` 또는 `.otf` 파일 추가
3. 스타일에서 `@font/font_name`으로 참조

```xml
<!-- res/font/noto_sans_cjkkr_regular.xml -->
<?xml version="1.0" encoding="utf-8"?>
<font-family xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:android="http://schemas.android.com/apk/res/android">
    <font
        app:fontStyle="normal"
        app:fontWeight="400"
        app:font="@font/noto_sans_cjkkr_regular" />
</font-family>
```

---

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
