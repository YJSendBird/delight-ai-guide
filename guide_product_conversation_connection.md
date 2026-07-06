# 제품별 대화 연동 가이드

이 가이드는 제품 페이지에서 해당 제품의 대화를 다시 열고, 없으면 새로 만들도록 구성하는 방법을 설명합니다.

핵심은 `openProductConversation(productId:productName:from:)` 같은 헬퍼를 하나 만들어서, 화면에서는 그 함수만 호출하게 만드는 것입니다.

---

## 고객사 요구사항

- 제품 페이지에 들어가면 그 제품의 대화를 다시 연다.
- 이미 진행 중인 이전 제품 대화가 있으면 유지한다.
- 제품 식별값은 `product_id` 키로 Context Object에 전달한다.
- 대화 화면은 `openProductConversation` 헬퍼로 진입한다.
- 대화 목록의 새 대화 버튼은 현재 제품의 대화로 연결한다.
- 대화 종료 후 나타나는 새 대화 시작 버튼은 숨긴다.

---

## iOS

### 구현 흐름

1. 앱에서 현재 PDP의 `productId`를 확보한다.
2. `openProductConversation(productId:productName:from:)` 헬퍼를 호출한다.
3. 헬퍼 내부에서 `authenticate`로 세션 연결을 보장한다. (`searchConversation`은 연결된 상태를 요구한다 — 미연결 시 `800101 Connection required.`)
4. `searchConversation`으로 기존 대화를 찾는다.
5. 있으면 그 채널을 열고, 없으면 `createConversation`으로 새 대화를 만든다.
6. 대화 생성과 화면 진입 둘 다 `context`에 `product_id`와 필요한 정보를 넣어 제품 범위를 유지한다.

### 바로 붙여 넣는 코드

```swift
import UIKit
import SendbirdAIAgentMessenger

private let aiAgentId = ProcessInfo.processInfo.environment["SENDBIRD_AI_AGENT_ID"] ?? "YOUR_AI_AGENT_ID"

struct ProductConversationContext {
    let productId: String
    let productName: String

    var dictionary: [String: String] {
        [
            "product_id": productId,
            "product_name": productName
        ]
    }
}

func openProductConversation(productId: String, productName: String, from parent: UIViewController) {
    let conversationContext = ProductConversationContext(
        productId: productId,
        productName: productName
    )

    // searchConversation은 연결(connect)된 상태를 요구합니다 (미연결 시 800101 "Connection required.").
    // authenticate로 세션 연결을 보장한 뒤 검색합니다.
    AIAgentMessenger.authenticate(aiAgentId: aiAgentId) { result in
        switch result {
        case .success:
            searchProductConversation(context: conversationContext.dictionary, from: parent)
        case .failure(let error):
            print("authenticate failed: \(error)")
        }
    }
}

private func searchProductConversation(context: [String: String], from parent: UIViewController) {
    AIAgentMessenger.searchConversation(
        params: AIAgentMessenger.SearchConversationParams(
            aiAgentId: aiAgentId,
            context: context
        )
    ) { result in
        switch result {
        case .success(let channelURLs):
            if let channelURL = channelURLs.first {
                presentProductConversation(channelURL: channelURL, context: context, from: parent)
            } else {
                createProductConversation(context: context, from: parent)
            }
        case .failure(let error):
            print("searchConversation failed: \(error)")
        }
    }
}

private func createProductConversation(context: [String: String], from parent: UIViewController) {
    AIAgentMessenger.createConversation(
        aiAgentId: aiAgentId,
        paramsBuilder: { params in
            params.context = context
        }
    ) { result in
        switch result {
        case .success(let channelURL):
            presentProductConversation(channelURL: channelURL, context: context, from: parent)
        case .failure(let error):
            print("createConversation failed: \(error)")
        }
    }
}

private func presentProductConversation(channelURL: String, context: [String: String], from parent: UIViewController) {
    DispatchQueue.main.async {
        AIAgentMessenger.presentConversation(aiAgentId: aiAgentId, channelURL: channelURL) { params in
            params.parent = parent
            params.presentationStyle = .fullScreen
            params.context = context
        }
    }
}
```

### 사용 예시

```swift
final class ProductViewController: UIViewController {
    var productId: String = "{PRODUCT_ID}"
    var productName: String = "{PRODUCT_NAME}"

    @IBAction func didTapChatButton(_ sender: UIButton) {
        openProductConversation(
            productId: productId,
            productName: productName,
            from: self
        )
    }
}
```

### 새 대화 버튼 처리

제품별 대화 흐름에서는 사용자가 임의로 맥락 없는 새 대화를 만들지 않도록 새 대화 버튼을 제어해야 합니다.

#### 대화 목록의 "새 대화" 버튼 — 현재 제품으로 연결

대화 목록 하단의 새 대화 버튼은 `SBAConversationListBottomView`를 서브클래싱해서 동작을 교체할 수 있습니다. 버튼(`button`)이 public으로 노출되고 `setupActions()`가 open이라, 기본 동작 대신 현재 제품의 대화 열기로 연결합니다.

```swift
import UIKit
import SendbirdAIAgentMessenger

// 현재 화면이 어떤 제품에 스코프되어 있는지 공급한다.
// 대화 목록을 띄우는 화면에서 값을 설정한다.
enum ProductConversationRouter {
    static var currentProductId: String?
    static var currentProductName: String?
    static var open: ((_ productId: String, _ productName: String) -> Void)?
}

// 새 대화 버튼이 제품 스코프 대화를 열도록 교체한 커스텀 하단 뷰
final class ProductScopedListBottomView: SBAConversationListBottomView {
    override func setupActions() {
        // SDK 기본 액션 대신 우리 동작으로 교체 (액션은 super 호출 없이 재정의)
        button?.removeTarget(nil, action: nil, for: .touchUpInside)
        button?.addTarget(self, action: #selector(didTapNewConversation), for: .touchUpInside)
    }

    @objc private func didTapNewConversation() {
        guard let productId = ProductConversationRouter.currentProductId,
              let productName = ProductConversationRouter.currentProductName else { return }
        // openProductConversation(productId:productName:from:) 호출로 연결
        ProductConversationRouter.open?(productId, productName)
    }
}

// 대화 목록을 열기 전에 등록
SBAConversationListModule.List.BottomView = ProductScopedListBottomView.self
```

사용 예시 — PDP에서 대화 목록을 열기 전에 라우터를 채워 둡니다.

```swift
ProductConversationRouter.currentProductId = "{PRODUCT_ID}"
ProductConversationRouter.currentProductName = "{PRODUCT_NAME}"
ProductConversationRouter.open = { [weak self] productId, productName in
    guard let self else { return }
    openProductConversation(productId: productId, productName: productName, from: self)
}
```

#### 대화 종료 후 "새 대화 시작" 버튼 — 숨김

대화가 종료되면 대화 화면 하단에 "새 상담 시작" 뷰가 나타납니다. 이 뷰는 config로 끌 수 있습니다.

```swift
// 대화 종료 후 나타나는 새 대화 시작(talk to agent) 뷰 숨김
AIAgentMessenger.config.conversation.isTalkToAgentViewEnabled = false
```

> 참고: 이전 이름인 `isConversationClosedViewEnabled`는 deprecated입니다. `isTalkToAgentViewEnabled`를 사용하세요.

#### 정리

| 위치 | 요구 동작 | 방법 |
| --- | --- | --- |
| 대화 목록 하단 새 대화 버튼 | 현재 제품의 대화로 연결 | `SBAConversationListBottomView` 서브클래스 + `SBAConversationListModule.List.BottomView` 등록 |
| 대화 종료 후 새 대화 시작 뷰 | 숨김 | `AIAgentMessenger.config.conversation.isTalkToAgentViewEnabled = false` |

### 구현 포인트

- `openProductConversation`은 화면에서 직접 길게 조립하지 않게 하려는 용도입니다.
- `searchConversation` 결과가 여러 개 나오면 가장 먼저 반환된 채널부터 여는 방식으로 시작할 수 있습니다.
- `params.context`는 대화 생성과 화면 진입 양쪽에 모두 넣어 두는 편이 안전합니다.
- `presentConversation` 호출은 메인 스레드에서 처리합니다.
- 제품이 바뀌면 새 `product_id`와 `productName`으로 다시 진입하면 됩니다.

### 최소 구성 요약

- 제품 페이지 진입 시 `openProductConversation(productId:productName:from:)` 호출
- 내부에서 `authenticate` → `searchConversation` → 없으면 `createConversation`
- `context["product_id"]`와 필요한 정보를 전달
- 화면은 `fullScreen` 기준으로 연다
- 대화 목록 새 대화 버튼은 현재 제품으로 연결, 종료 후 새 대화 시작 뷰는 숨김

---

## Android

### 0. 지원 현황 한눈에

| 요구사항                                                        | 대응                        |
| --------------------------------------------------------------- | --------------------------- |
| ① 제품 페이지 진입 → 제품 대화 열기 (이전 대화 있으면 재개, 없으면 새로 생성) | **지원** (본 가이드)        |
| ② 대화 목록의 "새 대화" 버튼 → 현재 제품으로                    | SDK 작업/배포 후 가이드 제공 |
| ③ 대화 종료 후 "새 대화" 버튼 → 현재 제품으로                   | **버튼 숨김** (설정)        |

> **핵심 원칙** — `product_id`는 항상 대화의 **context 객체**에 담습니다 (검색/생성/표시 시 동일하게). context는 키-값 맵이라 필요한 다른 값도 함께 담을 수 있습니다. 이렇게 해야 같은 제품의 대화를 다시 찾을 수 있습니다.

> **사전 준비** — 앱 초기화 직후 `AIAgentMessenger.updateSessionInfo(...)`로 **사용자 정체(익명/인증)** 를 1회 설정해 두어야 합니다 — `SessionInfo.AnonymousSessionInfo()` 또는 `SessionInfo.ManualSessionInfo(...)`. ([README의 공통 사전 준비](./README.md) 참고)

---

### 1. 제품 대화 띄우기 (필수)

제품 페이지에 진입할 때 호출합니다.

`product_id`로 대화를 **검색**하고 — 있으면 그 대화를 열고, 없으면 같은 context로 **새 대화를 생성**해서 엽니다.

```kotlin
import androidx.lifecycle.lifecycleScope
import com.sendbird.sdk.aiagent.messenger.AIAgentMessenger
import com.sendbird.sdk.aiagent.messenger.model.ConversationCreateParams
import com.sendbird.sdk.aiagent.messenger.model.ConversationSettingsParams
import com.sendbird.sdk.aiagent.messenger.model.SearchConversationParams
import com.sendbird.sdk.aiagent.messenger.ui.activity.MessengerActivity
import kotlinx.coroutines.launch

// Activity 메서드. aiAgentId는 화면이 보유한 nullable 필드.
private fun openProductConversation(productId: String) {
    val aiAgentId = aiAgentId ?: return
    val context = mapOf(
        "product_id" to productId
        // 필요 시 다른 키도 함께 담을 수 있습니다 ("키" to 값, ...)
    )
    lifecycleScope.launch {
        runCatching {
            with(AIAgentMessenger) {
                val existing = awaitSearchConversation(              // 기존 대화 검색
                    SearchConversationParams(aiAgentId, context)
                ).firstOrNull()
                val url = existing ?: awaitCreateConversation(       // 없으면 새로 생성
                    ConversationCreateParams(aiAgentId = aiAgentId, context = context)
                )
                startActivity(
                    MessengerActivity.newIntentForConversation(
                        this@YourActivity, aiAgentId, url,
                        ConversationSettingsParams(context = context) // 새 대화도 제품 스코프
                    )
                )
            }
        }.onFailure {
            // 네트워크 오류 등 — 재시도 UI 또는 무시
        }
    }
}
```

동작 요약:

- `awaitSearchConversation` — `context`가 **정확히 일치**하는 대화의 채널 URL 목록을 반환합니다 (없으면 빈 리스트).
- `awaitCreateConversation` — 해당 `context`로 새 대화를 만들고 채널 URL을 반환합니다.
- `MessengerActivity.newIntentForConversation(..., url, ...)` — 해당 대화를 전체 화면으로 엽니다.

---

### 2. 대화 종료 후 "새 대화" 버튼 제어

대화가 종료되면 대화 화면 하단에 "새 대화 시작" 버튼이 나타납니다. 이 버튼으로 시작한 새 대화는 현재 제품 스코프가 보장되지 않으므로, 아래 설정으로 **버튼을 숨김** 처리해주세요.

이 설정은 **대화 화면을 열기 전에** 적용되어 있어야 합니다. init 직후 1회 설정을 권장합니다.

```kotlin
// init 직후, 화면 열기 전 1회
AIAgentMessenger.config.conversation.list.shouldShowMessageFooterView = false
```

---

### 체크리스트

1. **product_id는 context에.** 검색/생성/표시 모든 호출에서 동일하게 `mapOf("product_id" to ...)`.
2. **제품 페이지 진입 시** `openProductConversation` 호출 (1단계).
3. **대화 종료 후 "새 대화" 버튼**은 설정으로 숨김 (2단계).
4. **전역 설정/등록**은 init 직후, 화면 열기 전에 1회.

---

## Web (JavaScript)

### 구현 방법

> 작성 예정

### 코드 예시

```javascript
// 작성 예정
```
