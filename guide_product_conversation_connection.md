# 제품별 대화 연동 가이드

이 가이드는 제품 페이지에서 해당 제품의 대화를 다시 열고, 없으면 새로 만들도록 iOS를 구성하는 방법을 설명합니다.

핵심은 `openProductConversation(productId:from:)` 같은 헬퍼를 하나 만들어서, 화면에서는 그 함수만 호출하게 만드는 것입니다.

---

## 고객사 요구사항

- 제품 페이지에 들어가면 그 제품의 대화를 다시 연다.
- 이미 진행 중인 이전 제품 대화가 있으면 유지한다.
- 제품 식별값은 `product_id` 키로 Context Object에 전달한다.
- 대화 화면은 `openProductConversation` 헬퍼로 진입한다.

---

## iOS 구현 흐름

1. 앱에서 현재 PDP의 `productId`를 확보한다.
2. `openProductConversation(productId:from:)` 헬퍼를 호출한다.
3. 헬퍼 내부에서 `authenticate`로 세션 연결을 보장한다. (`searchConversation`은 연결된 상태를 요구한다 — 미연결 시 `800101 Connection required.`)
4. `searchConversation`으로 기존 대화를 찾는다.
5. 있으면 그 채널을 열고, 없으면 `createConversation`으로 새 대화를 만든다.
6. 대화 생성과 화면 진입 둘 다 `context`에 `product_id`와 필요한 정보를 넣어 제품 범위를 유지한다.

---

## 바로 붙여 넣는 코드

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

---

## 사용 예시

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

---

## Android

### 구현 방법

> 작성 예정

### 코드 예시

```kotlin
// 작성 예정
```

---

## Web (JavaScript)

### 구현 방법

> 작성 예정

### 코드 예시

```javascript
// 작성 예정
```

---

## 구현 포인트

- `openProductConversation`은 화면에서 직접 길게 조립하지 않게 하려는 용도입니다.
- `searchConversation` 결과가 여러 개 나오면 가장 먼저 반환된 채널부터 여는 방식으로 시작할 수 있습니다.
- `params.context`는 대화 생성과 화면 진입 양쪽에 모두 넣어 두는 편이 안전합니다.
- `presentConversation` 호출은 메인 스레드에서 처리합니다.
- 제품이 바뀌면 새 `product_id`와 `productName`으로 다시 진입하면 됩니다.

---

## 최소 구성 요약

- 제품 페이지 진입 시 `openProductConversation(productId:productName:from:)` 호출
- 내부에서 `searchConversation` → 없으면 `createConversation`
- `context["product_id"]`와 필요한 정보를 전달
- 화면은 `fullScreen` 기준으로 연다
