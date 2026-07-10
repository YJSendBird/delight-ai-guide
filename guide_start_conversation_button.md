# 대화 리스트 하단 '대화 시작' 버튼명 변경 가이드

대화 리스트(Conversation List) 화면 하단에 표시되는 **"Start a conversation"(대화 시작)** 버튼의 문구를 원하는 텍스트로 변경하는 방법을 설명합니다.

SDK의 UI 문자열은 플랫폼별 리소스 키로 관리되며, 앱에서 **같은 키로 재정의**하면 SDK 기본 문구보다 앱의 문구가 우선 적용됩니다.

---

## 변경 대상 문자열 키

| 플랫폼 | 문자열 키 | 기본값(영문) |
| --- | --- | --- |
| iOS | `SBA_ConversationList_Footer_talkToAgent` | Start a conversation |
| Android | `aa_text_talk_to_agent` | Start a conversation |

---

## iOS

### 1. `Localizable.strings` 파일 추가

앱 프로젝트에 `Localizable.strings` 파일이 없다면 먼저 생성합니다.

1. **Project Navigator**에서 타깃 그룹(폴더)을 우클릭
2. **New File from Template…** 선택
3. **Resource** 섹션에서 **Strings File(Legacy)** 선택
4. 파일 이름을 정확히 `Localizable.strings`로 지정 후 생성

### 2. Localization 활성화

한국어 등 지원 언어별 파일을 만들려면 프로젝트에 Localization을 추가합니다.

1. 프로젝트(파란색 아이콘) 선택 → **Project** 선택 → **Info** 탭
2. **Localizations** 섹션에서 **+** 버튼으로 언어 추가 (예: Korean)
3. 파일 선택 다이얼로그에서 `Localizable.strings` 체크
4. 언어별 `Localizable.strings` 파일이 생성됩니다

### 3. 버튼 문구 재정의

언어별 `Localizable.strings` 파일에 아래 키를 원하는 문구로 정의합니다.

```swift
// ko.lproj/Localizable.strings
"SBA_ConversationList_Footer_talkToAgent" = "새 상담 시작하기";
```

```swift
// en.lproj/Localizable.strings
"SBA_ConversationList_Footer_talkToAgent" = "Start a new chat";
```

키를 정의하면 SDK 내장 문구 대신 앱에서 정의한 문구가 우선 사용됩니다.

> **참고**: 종료된 대화 화면 하단에도 동일한 문구의 버튼이 표시됩니다. 이 버튼도 함께 변경하려면 `SBA_Conversation_List_Closed_talkToAgent` 키를 같은 방식으로 재정의하세요.

---

## Android

### 버튼 문구 재정의

앱의 `strings.xml`에 **같은 키**로 새 값을 정의하면 SDK 문자열이 재정의됩니다. 앱 리소스가 SDK 리소스보다 우선 적용됩니다.

```xml
<!-- res/values/strings.xml (기본) -->
<string name="aa_text_talk_to_agent">Start a new chat</string>
```

```xml
<!-- res/values-ko/strings.xml (한국어) -->
<string name="aa_text_talk_to_agent">새 상담 시작하기</string>
```

### 언어별 리소스 추가 (Translations Editor)

한국어 외 다른 언어도 함께 지원하려면 Android Studio의 Translations Editor를 사용합니다.

1. `res/values/strings.xml`을 우클릭하거나 🌐 아이콘 클릭 → **Translations Editor** 열기
2. **Add Locale(🌐➕)** 아이콘으로 언어 추가 (예: 일본어 `ja`)
3. 생성된 `res/values-ja/strings.xml`에 동일 키로 문구 정의

---

## 주의사항

- **언어별로 각각 재정의**: 지원하는 모든 언어의 리소스 파일에 키를 정의하세요. 특정 언어 파일에 키가 없으면 해당 언어에서는 SDK 기본 문구가 표시됩니다.
- **표시 언어 기준**: 어떤 언어 리소스가 사용될지는 화면 실행 시 전달한 `language` 값(미지정 시 디바이스 로케일)을 따릅니다. 한글 강제 설정과 함께 사용한다면 [한글 언어 강제 설정 가이드](./guide_language.md)를 참고하세요.
- **키 오타 주의**: 키가 정확히 일치해야 재정의됩니다. 전체 문자열 키 목록은 아래 리소스에서 확인할 수 있습니다.

---

## 추가 리소스

- [iOS 다국어 지원 가이드 (MULTILANGUAGE.md)](https://github.com/sendbird/delight-ai-agent/blob/main/ios/docs/MULTILANGUAGE.md)
- [Android 다국어 지원 가이드 (MULTILANGUAGE.md)](https://github.com/sendbird/delight-ai-agent/blob/main/android/docs/MULTILANGUAGE.md)
- [iOS 전체 문자열 키 목록 (Localizable.strings)](https://github.com/sendbird/delight-ai-agent/blob/main/ios/en.lproj/Localizable.strings)
- [Android 전체 문자열 키 목록 (strings.xml)](https://github.com/sendbird/delight-ai-agent/blob/main/android/res/values/strings.xml)
