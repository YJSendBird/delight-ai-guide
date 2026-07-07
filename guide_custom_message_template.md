# Custom Message Template 사용 가이드 (상품 옵션 선택 예제)

이 가이드는 `go-to-hanssem-mall-with-options` 메시지 템플릿을 기반으로 **상품 옵션 선택 화면**을 구현하는 방법을 설명합니다.

---

## 개요

**Custom Message Template란?**

- Sendbird에서 미리 정의한 템플릿이 아닌, **비즈니스 특화 UI**를 구현할 수 있음
- AI Agent 서버에서 구조화된 데이터(JSON)를 클라이언트로 전송
- 클라이언트에서 이 데이터를 받아 **자신의 UI 컴포넌트**로 렌더링
- 예제: 쿠폰, 상품 목록, 예약 폼, **상품 옵션 선택** 등

**`go-to-hanssem-mall-with-options` 템플릿의 목표:**

- AI Agent가 사용자에게 상품을 추천
- 사용자가 **색상, 사이즈, 수량** 등 옵션 선택
- "장바구니에 추가" 또는 "구매하기" 버튼 클릭
- 앱에서 선택 결과를 처리 → 한샘 쇼핑몰로 연동

---

## 데이터 구조

### 1. 메시지 페이로드 구조

AI Agent 서버에서 클라이언트로 전송되는 데이터:

```json
{
  "custom_message_templates": [
    {
      "id": "go-to-hanssem-mall-with-options",
      "response": {
        "status": 200,
        "content": "{\"productId\": \"PROD_12345\", \"productName\": \"소파\", \"productImage\": \"https://...\", \"price\": 599000, \"options\": {\"color\": [\"회색\", \"검정\", \"갈색\"], \"size\": [\"3인\", \"2인\"]}}"
      },
      "error": null
    }
  ]
}
```

### 2. CustomMessageTemplateData 인터페이스

```typescript
interface CustomMessageTemplateData {
  id: string; // "go-to-hanssem-mall-with-options"
  response: {
    status: number; // HTTP 상태코드 (200, 404 등)
    content: string | null; // JSON 문자열
  };
  error: string | null; // 에러 메시지
}

interface ExtendedMessagePayload {
  custom_message_templates?: CustomMessageTemplateData[];
}

// 상품 옵션 데이터 (content를 파싱한 후의 형태)
interface ProductWithOptions {
  productId: string;
  productName: string;
  productImage: string;
  price: number;
  description?: string;
  options: {
    [key: string]: string[]; // color: ["회색", "검정"], size: ["3인", "2인"]
  };
}
```

---

## Web (React) 구현

### 1단계: 커스텀 컴포넌트 생성

```tsx
import React, { useState } from "react";
import type { CustomMessageTemplateData } from "@sendbird/ai-agent-messenger-react";

interface Props {
  extendedMessagePayload?: {
    custom_message_templates?: CustomMessageTemplateData[];
  };
}

interface ProductWithOptions {
  productId: string;
  productName: string;
  productImage: string;
  price: number;
  description?: string;
  options: {
    [key: string]: string[];
  };
}

export const ProductOptionTemplate: React.FC<Props> = ({
  extendedMessagePayload,
}) => {
  const template = extendedMessagePayload?.custom_message_templates?.find(
    (t) => t.id === "go-to-hanssem-mall-with-options",
  );

  if (!template) return null;

  // 에러 처리
  if (template.error) {
    return (
      <div
        style={{
          padding: "16px",
          backgroundColor: "#fff3cd",
          borderRadius: "8px",
        }}
      >
        <p style={{ color: "#856404", margin: 0 }}>
          상품 정보를 불러올 수 없습니다.
        </p>
        <small>{template.error}</small>
      </div>
    );
  }

  // 상태 코드 확인
  if (template.response.status !== 200) {
    return (
      <div
        style={{
          padding: "16px",
          backgroundColor: "#fff3cd",
          borderRadius: "8px",
        }}
      >
        <p style={{ color: "#856404", margin: 0 }}>
          상품 정보를 불러올 수 없습니다. (오류: {template.response.status})
        </p>
      </div>
    );
  }

  let product: ProductWithOptions;
  try {
    product = JSON.parse(template.response.content ?? "{}");
  } catch (e) {
    return (
      <div
        style={{
          padding: "16px",
          backgroundColor: "#f8d7da",
          borderRadius: "8px",
        }}
      >
        <p style={{ color: "#721c24", margin: 0 }}>상품 데이터 파싱 오류</p>
      </div>
    );
  }

  return <ProductOptionCard product={product} />;
};

const ProductOptionCard: React.FC<{ product: ProductWithOptions }> = ({
  product,
}) => {
  const [selectedOptions, setSelectedOptions] = useState<
    Record<string, string>
  >({});
  const [quantity, setQuantity] = useState(1);

  const handleOptionChange = (optionName: string, value: string) => {
    setSelectedOptions((prev) => ({
      ...prev,
      [optionName]: value,
    }));
  };

  const handleAddToCart = () => {
    // 모든 옵션이 선택되었는지 확인
    const unselectedOptions = Object.keys(product.options).filter(
      (key) => !selectedOptions[key],
    );

    if (unselectedOptions.length > 0) {
      alert(`다음 옵션을 선택해주세요: ${unselectedOptions.join(", ")}`);
      return;
    }

    // 사용자 선택 결과 처리
    console.log("선택된 옵션:", {
      productId: product.productId,
      productName: product.productName,
      quantity,
      selectedOptions,
    });

    // GA4 또는 커스텀 분석 이벤트 발송
    if (window.gtag) {
      window.gtag("event", "add_to_cart", {
        product_id: product.productId,
        product_name: product.productName,
        quantity: quantity,
        options: JSON.stringify(selectedOptions),
      });
    }

    // 실제 한샘 쇼핑몰 API 호출 또는 페이지 이동
    // window.location.href = `https://hanssem.com/add-to-cart?productId=${product.productId}&options=${JSON.stringify(selectedOptions)}`;

    alert("장바구니에 추가되었습니다!");
  };

  return (
    <div
      style={{
        border: "1px solid #e0e0e0",
        borderRadius: "12px",
        padding: "16px",
        marginBottom: "12px",
        backgroundColor: "#fff",
      }}
    >
      {/* 상품 이미지 및 기본 정보 */}
      <div style={{ display: "flex", gap: "16px", marginBottom: "16px" }}>
        <img
          src={product.productImage}
          alt={product.productName}
          style={{
            width: "100px",
            height: "100px",
            borderRadius: "8px",
            objectFit: "cover",
          }}
        />
        <div style={{ flex: 1 }}>
          <h4 style={{ margin: "0 0 8px 0", fontSize: "16px" }}>
            {product.productName}
          </h4>
          {product.description && (
            <p style={{ margin: "0 0 8px 0", fontSize: "13px", color: "#666" }}>
              {product.description}
            </p>
          )}
          <p
            style={{
              margin: "0",
              fontSize: "18px",
              fontWeight: "bold",
              color: "#2c3e50",
            }}
          >
            ₩{product.price.toLocaleString()}
          </p>
        </div>
      </div>

      {/* 옵션 선택 */}
      <div style={{ marginBottom: "16px" }}>
        {Object.entries(product.options).map(([optionName, optionValues]) => (
          <div key={optionName} style={{ marginBottom: "12px" }}>
            <label
              style={{
                display: "block",
                marginBottom: "6px",
                fontWeight: "500",
                fontSize: "13px",
              }}
            >
              {optionName}
              {!selectedOptions[optionName] && (
                <span style={{ color: "#e74c3c", marginLeft: "4px" }}>*</span>
              )}
            </label>
            <select
              value={selectedOptions[optionName] ?? ""}
              onChange={(e) => handleOptionChange(optionName, e.target.value)}
              style={{
                width: "100%",
                padding: "8px",
                border: "1px solid #ddd",
                borderRadius: "4px",
                fontSize: "13px",
                fontFamily: "inherit",
              }}
            >
              <option value="">-- 선택해주세요 --</option>
              {optionValues.map((value) => (
                <option key={value} value={value}>
                  {value}
                </option>
              ))}
            </select>
          </div>
        ))}
      </div>

      {/* 수량 선택 */}
      <div style={{ marginBottom: "16px" }}>
        <label
          style={{
            display: "block",
            marginBottom: "6px",
            fontWeight: "500",
            fontSize: "13px",
          }}
        >
          수량
        </label>
        <input
          type="number"
          min="1"
          max="99"
          value={quantity}
          onChange={(e) =>
            setQuantity(Math.max(1, parseInt(e.target.value) || 1))
          }
          style={{
            width: "60px",
            padding: "8px",
            border: "1px solid #ddd",
            borderRadius: "4px",
            fontSize: "13px",
          }}
        />
      </div>

      {/* 액션 버튼 */}
      <div style={{ display: "flex", gap: "8px" }}>
        <button
          onClick={handleAddToCart}
          style={{
            flex: 1,
            padding: "12px",
            backgroundColor: "#2c3e50",
            color: "white",
            border: "none",
            borderRadius: "6px",
            fontSize: "14px",
            fontWeight: "bold",
            cursor: "pointer",
          }}
        >
          장바구니 추가
        </button>
        <button
          style={{
            flex: 1,
            padding: "12px",
            backgroundColor: "#e74c3c",
            color: "white",
            border: "none",
            borderRadius: "6px",
            fontSize: "14px",
            fontWeight: "bold",
            cursor: "pointer",
          }}
          onClick={() => {
            // 바로 구매 로직
            handleAddToCart();
            // window.location.href = `https://hanssem.com/checkout?productId=${product.productId}`;
          }}
        >
          바로 구매
        </button>
      </div>
    </div>
  );
};
```

### 2단계: 컴포넌트 등록

```tsx
import {
  AgentProviderContainer,
  IncomingMessageLayout,
} from "@sendbird/ai-agent-messenger-react";
import { ProductOptionTemplate } from "./ProductOptionTemplate";

export function App() {
  return (
    <AgentProviderContainer appId="YOUR_APP_ID" aiAgentId="YOUR_AI_AGENT_ID">
      {/* Custom Message Template 등록 */}
      <IncomingMessageLayout.CustomMessageTemplate
        component={ProductOptionTemplate}
      />
    </AgentProviderContainer>
  );
}
```

---

## Android 구현

### 개요

**Custom Message Template이란?**

- Sendbird가 미리 정의한 템플릿이 아닌, **비즈니스 특화 UI**를 구현할 수 있는 기능
- AI Agent 서버가 구조화된 데이터(JSON)를 메시지의 `extendedMessagePayload`에 담아 전송
- 클라이언트는 이 데이터를 받아 **직접 만든 View**로 렌더링
- 예: 쿠폰, 상품 목록, 예약 폼, **상품 옵션 선택** 등

**`go-to-hanssem-mall-with-options` 템플릿의 목표:**

1. AI 에이전트가 사용자에게 상품을 추천
2. 사용자가 **색상, 사이즈, 수량** 등 옵션 선택
3. "장바구니 추가" / "바로 구매" 버튼 클릭
4. 앱이 선택 결과를 처리 → 한샘 쇼핑몰로 연동

---

### 데이터 구조

핸들러에는 SDK가 파싱한 `CustomMessageTemplateData` 목록이 전달됩니다. `response.content`에 템플릿별 데이터(JSON 문자열)가 들어 있습니다.

```kotlin
// com.sendbird.sdk.aiagent.messenger.model.conversation.CustomMessageTemplateData
data class CustomMessageTemplateData(
    val id: String,          // 대시보드에 설정한 템플릿 식별자
    val response: Response,
    val error: String?       // 실패 사유 (실패 시)
) {
    data class Response(
        val status: Int,     // HTTP 상태 코드
        val content: String? // 콘텐츠 페이로드 (JSON 문자열)
    )
}
```

#### 렌더링 위치

커스텀 템플릿은 메시지 본문(버블) 아래의 전용 슬롯에 렌더링됩니다.

```
┌─────────────────────────────┐
│        <MessageBubble>      │
└─────────────────────────────┘
┌─────────────────────────────┐
│   <CustomMessageTemplate>   │   <- 앱이 만든 View가 이 위치에 표시됨
└─────────────────────────────┘
┌─────────────────────────────┐
│        <Feedback> 등        │
└─────────────────────────────┘
```

---

### 1단계: `CustomMessageTemplateViewHandler` 구현

SDK가 커스텀 템플릿 데이터를 감지하면 이 핸들러를 호출하고, 앱은 View를 만들어 `callback.onViewReady(view)`로 돌려줍니다.

```kotlin
import android.content.Context
import android.view.LayoutInflater
import android.view.View
import android.widget.TextView
import com.sendbird.android.message.BaseMessage
import com.sendbird.sdk.aiagent.messenger.interfaces.CustomMessageTemplateViewCallback
import com.sendbird.sdk.aiagent.messenger.interfaces.CustomMessageTemplateViewHandler
import com.sendbird.sdk.aiagent.messenger.model.conversation.CustomMessageTemplateData
import org.json.JSONObject

class ProductOptionTemplateHandler : CustomMessageTemplateViewHandler {

    override fun onCreateCustomMessageTemplateView(
        context: Context,
        message: BaseMessage,
        data: List<CustomMessageTemplateData>,
        callback: CustomMessageTemplateViewCallback,
    ) {
        val template = data.firstOrNull { it.id == "go-to-hanssem-mall-with-options" }
            ?: return callback.onViewReady(createFallbackView(context))

        // 에러 처리
        if (template.error != null) {
            return callback.onViewReady(createErrorView(context, "상품 정보를 불러올 수 없습니다."))
        }
        // 상태 코드 확인
        if (template.response.status != 200) {
            return callback.onViewReady(
                createErrorView(context, "상품 정보를 불러올 수 없습니다. (오류: ${template.response.status})")
            )
        }

        val view = try {
            val content = template.response.content
                ?: return callback.onViewReady(createFallbackView(context))
            createProductOptionView(context, JSONObject(content))
        } catch (e: Exception) {
            createErrorView(context, "상품 데이터 파싱 오류")
        }
        // onViewReady는 반드시 메인 스레드에서 호출해야 합니다.
        callback.onViewReady(view)
    }

    private fun createProductOptionView(context: Context, product: JSONObject): View {
        // 파싱한 product JSON(productId/productName/price/options 등)으로
        // 앱의 UI 컴포넌트(상품 카드, 옵션 선택, 장바구니/구매 버튼 등)를 자유롭게 구성합니다.
        // 버튼 클릭 시 쇼핑몰 연동·GA4 추적은 앱에서 처리 (guide_ga4_livemetric.md 참고)
        val view = LayoutInflater.from(context)
            .inflate(R.layout.view_product_option_template, null)
        // ... 앱 구현 ...
        return view
    }

    private fun createErrorView(context: Context, message: String): View =
        TextView(context).apply {
            text = message
            setPadding(32, 32, 32, 32)
        }

    private fun createFallbackView(context: Context): View =
        TextView(context).apply {
            text = "지원하지 않는 템플릿입니다"
            setPadding(32, 32, 32, 32)
        }
}
```

---

### 2단계: 핸들러 등록

핸들러는 대화 어댑터 Provider를 통해 등록합니다. **init 직후, 화면 열기 전에 1회** 수행하세요.

```kotlin
import com.sendbird.sdk.aiagent.messenger.providers.AIAgentAdapterProviders
import com.sendbird.sdk.aiagent.messenger.providers.ConversationAdapterProvider
import com.sendbird.sdk.aiagent.messenger.ui.recyclerview.ConversationMessageListAdapter

AIAgentAdapterProviders.conversation =
    ConversationAdapterProvider { channel, uiParams, containerGenerator ->
        uiParams.customMessageTemplateViewHandler = ProductOptionTemplateHandler()
        ConversationMessageListAdapter(channel, uiParams, containerGenerator)
    }
```

---

### 주의사항

#### 1. 에러/폴백 처리 필수

- `template.error != null` → 실패 사유 표시 또는 배너 미표시
- `template.response.status != 200` → 오류 UI
- JSON 파싱 실패 → try-catch로 방어
- **등록하지 않은 템플릿 ID**가 내려올 수 있으므로 폴백 View를 반환해 앱이 깨지지 않게 하세요.

#### 2. 스레드

- `callback.onViewReady(view)`는 **반드시 메인 스레드에서 호출**해야 합니다.
- 핸들러 내부에서 네트워크 등 비동기 작업을 한다면, 완료 후 메인 스레드로 전환해 `onViewReady`를 호출하세요.

#### 3. 성능

- 메시지 목록(RecyclerView) 안에 렌더링되므로 View 생성을 가볍게 유지하세요.
- 이미지: Glide/Coil 등의 캐싱 라이브러리 사용 권장.

#### 4. 하나의 메시지에 여러 템플릿

- `data`는 리스트입니다. 한 메시지에 여러 템플릿이 올 수 있으니 `id`로 필터링해 처리하세요.

## iOS 구현

iOS SDK에서는 `SBACustomMessageTemplateView`를 서브클래싱해서 커스텀 템플릿 UI를 구현하고, `SBAModuleSet`에 등록합니다. 검증 기준 SDK 버전은 `SendbirdAIAgentCore 1.15.0`입니다.

> **1.15.0 주의 — 서브클래스에 stored property 금지**
>
> 커스텀 컴포넌트 서브클래스에 **stored property를 선언하면 대화 진입 시 크래시**합니다. SDK 내부 의존성 주입(`MessengerInfoInjectable`)이 `Mirror`로 프로퍼티를 순회하다 dynamic cast 오류를 내기 때문이며, 다음 릴리즈에서 수정 예정입니다. 아래처럼 뷰는 `layoutBody()`에서 만들고, 참조가 필요하면 `tag` + computed property로 접근하세요.

### 렌더링 위치

커스텀 템플릿 뷰는 메시지 버블 아래의 전용 슬롯에 렌더링됩니다.

```
┌─────────────────────────────┐
│        <MessageBubble>      │
└─────────────────────────────┘
┌─────────────────────────────┐
│  <CustomMessageTemplateView>│   <- layoutBody() 가 반환한 뷰가 이 위치
└─────────────────────────────┘
┌─────────────────────────────┐
│        <Feedback> 등        │
└─────────────────────────────┘
```

### 사이즈와 여백 (full-width 렌더링)

웹/Android처럼 콘텐츠 영역을 꽉 채우려면, `layoutBody()`가 반환하는 뷰의 폭과 내부 요소의 늘어남 규칙을 명시해야 합니다.

**1) 폭은 루트 뷰에 직접 준다**

- 커스텀 템플릿 뷰의 최대 폭 상한은 `SBAConstant.messageCellMaxWidth`(기본 244pt)입니다. `SBACustomMessageTemplateView.setupLayouts()`가 이 값으로 `<=` 제약을 겁니다.
- iOS Auto Layout은 폭 제약이 없으면 내부 콘텐츠의 intrinsic content size(예: 라벨 텍스트 폭)를 따라갑니다. 라벨/버튼만 넣으면 **텍스트 폭으로만** 그려지고 가로로 늘어나지 않습니다.
- 따라서 `layoutBody()` 루트 뷰에 폭을 직접 고정합니다.

```swift
stack.translatesAutoresizingMaskIntoConstraints = false
stack.widthAnchor.constraint(
    equalToConstant: AIAgentMessenger.config.conversation.messageCellMaxWidth
).isActive = true
```

- 셀 최대 폭 자체를 넓히려면 화면을 열기 전에 조정합니다(기본 244pt).

```swift
AIAgentMessenger.config.conversation.messageCellMaxWidth = 288.0
```

**2) 내부 요소는 늘어나도록 우선순위를 낮춘다**

- 루트 폭을 고정해도, 내부 요소가 hugging을 유지하면 자기 텍스트 폭만큼만 그려집니다.
- 가로로 함께 채우려면 스택 정렬을 `.fill`로 두고, 내부 요소의 hugging/compression 우선순위를 낮춥니다. 높이는 고정합니다.

```swift
button.heightAnchor.constraint(equalToConstant: 44).isActive = true
button.setContentHuggingPriority(.defaultLow, for: .horizontal)
button.setContentCompressionResistancePriority(.defaultLow, for: .horizontal)

let stack = SBALinearLayout.vStack(alignment: .fill) { button }
```

**3) 좌우/상하 여백**

- 셀은 커스텀 템플릿 뷰를 붙일 때 **왼쪽 정렬 여백을 이미 적용**합니다(`messageAreaLeftSpacing`). `layoutBody()` 안에서 좌우 패딩을 또 주면 이중으로 밀립니다. 특별한 이유가 없으면 좌우 패딩은 생략하세요.
- 위/아래 간격이 필요하면 `SBALinearLayout`의 `.set(padding:)`으로 `SBAPadding`을 줍니다. 좌우를 직접 넣어야 하는 경우(셀 여백을 우회하는 커스텀 셀 등)에는 **left와 right를 같은 값으로** 줘 대칭을 맞춥니다.

```swift
SBALinearLayout.vStack(alignment: .fill) {
    button.set(padding: .init(top: 8, bottom: 8)) // 상하 간격만
}
```

### 1단계: 커스텀 템플릿 뷰 생성

```swift
import UIKit
import SendbirdAIAgentMessenger

final class ProductOptionTemplateView: SBACustomMessageTemplateView {
    private static let nameTag = 990_101
    private static let priceTag = 990_102

    // stored property 금지(1.15.0 크래시) — computed로 조회
    private var nameLabel: UILabel? { viewWithTag(Self.nameTag) as? UILabel }
    private var priceLabel: UILabel? { viewWithTag(Self.priceTag) as? UILabel }

    override func layoutBody() -> UIView? {
        let nameLabel = UILabel()
        nameLabel.tag = Self.nameTag
        nameLabel.font = .systemFont(ofSize: 16, weight: .semibold)
        nameLabel.numberOfLines = 2

        let priceLabel = UILabel()
        priceLabel.tag = Self.priceTag
        priceLabel.font = .systemFont(ofSize: 15, weight: .bold)

        let addToCartButton = UIButton(type: .system)
        addToCartButton.setTitle("장바구니 추가", for: .normal)
        addToCartButton.addTarget(self, action: #selector(addToCartTapped), for: .touchUpInside)

        // full-width 로 그리려면 루트 뷰에 폭 제약을 준다 (위 "사이즈와 여백" 참고).
        // 셀이 왼쪽 정렬 여백을 이미 주므로 여기선 좌우 패딩을 넣지 않는다.
        let stack = SBALinearLayout.vStack(alignment: .fill) {
            nameLabel
            priceLabel
            addToCartButton
        }
        stack.translatesAutoresizingMaskIntoConstraints = false
        stack.widthAnchor.constraint(
            equalToConstant: AIAgentMessenger.config.conversation.messageCellMaxWidth
        ).isActive = true
        return stack
    }

    // 서버가 내려준 custom_message_templates 데이터로 화면 구성
    override func configure(with customMessageTemplates: [SBACustomMessageTemplateData]?) {
        super.configure(with: customMessageTemplates)

        guard let template = customMessageTemplates?.first(where: {
            $0.templateId == "go-to-hanssem-mall-with-options"
        }) else {
            // 등록하지 않은/조건에 안 맞는 템플릿이면 노출하지 않는다
            self.isHidden = true
            return
        }
        self.isHidden = false

        // 에러/상태 코드 처리
        guard template.error == nil, template.response.status == 200,
              let content = template.response.content,
              let jsonData = content.data(using: .utf8),
              let product = try? JSONSerialization.jsonObject(with: jsonData) as? [String: Any]
        else {
            nameLabel?.text = "상품 정보를 불러올 수 없습니다."
            return
        }

        nameLabel?.text = product["productName"] as? String
        if let price = product["price"] as? Int {
            priceLabel?.text = "₩\(price)"
        }
    }

    @objc private func addToCartTapped() {
        // 선택 결과를 뷰 컨트롤러로 전달
        sendEvent(name: "add_to_cart", data: ["templateId": "go-to-hanssem-mall-with-options"])
    }
}
```

### 2단계: 커스텀 뷰 등록

```swift
// 대화 화면을 열기 전에 등록
SBAModuleSet.ConversationModule.List.Cell.CustomMessageTemplateView = ProductOptionTemplateView.self
```

### 3단계: 이벤트 처리

커스텀 뷰에서 `sendEvent(name:data:)`로 보낸 이벤트는 `SBAConversationViewController`의 delegate 이벤트로 전달됩니다.

먼저 `SBAConversationViewController`를 상속한 클래스에서 이벤트를 받고,

```swift
final class MyConversationViewController: SBAConversationViewController {
    override func conversationModule(
        _ listComponent: SBAConversationModule.List,
        didReceiveEvent event: SBAConversationModule.List.DelegateEvent
    ) {
        switch event {
        case .didReceiveCustomMessageTemplateAction(let name, let data, let message):
            handleCustomTemplateAction(name: name, data: data, message: message)
        default:
            super.conversationModule(listComponent, didReceiveEvent: event)
        }
    }

    private func handleCustomTemplateAction(name: String, data: Any?, message: BaseMessage) {
        switch name {
        case "add_to_cart":
            // 한샘 몰 이동 등 후속 처리
            break
        default:
            break
        }
    }
}
```

이 커스텀 VC를 **2단계와 같은 지점(화면 열기 전)에서** 등록해야 실제로 사용됩니다. 등록하지 않으면 기본 `SBAConversationViewController`가 쓰여 위 override가 호출되지 않습니다.

```swift
SBAViewControllerSet.ConversationViewController = MyConversationViewController.self
```

### 데이터 구조 (iOS)

```swift
// SBACustomMessageTemplateData
public struct SBACustomMessageTemplateData: Codable {
    public let templateId: String        // JSON의 "id"
    public let response: Response        // status + content(JSON 문자열)
    public let error: String?

    public struct Response: Codable {
        public let status: Int
        public let content: String?
    }
}
```

### 주의사항 (iOS)

- **에러/상태 코드** — `template.error != nil`, `template.response.status != 200`, JSON 파싱 실패를 모두 방어하세요.
- **이벤트 등록 필수** — 이벤트를 받으려면 커스텀 뷰(2단계)뿐 아니라 커스텀 VC(3단계, `SBAViewControllerSet.ConversationViewController`)도 등록해야 합니다.
- **하나의 메시지에 여러 템플릿** — `configure(with:)`의 인자는 배열입니다. 한 메시지에 여러 템플릿이 올 수 있으니 `templateId`로 필터링해 처리하세요.

---

## 사용 흐름

### 1단계: 사용자 - 상품 상담

```
사용자: "소파 추천해줄래?"
AI Agent: "한샘의 프리미엄 소파를 추천합니다!"
[Custom Message Template 렌더링]
- 상품 이미지
- 상품명: "한샘 프리미엄 소파"
- 가격: ₩599,000
- 옵션 선택: 색상(회색, 검정, 갈색), 사이즈(3인, 2인)
- 수량 입력
- [장바구니 추가] [바로 구매] 버튼
```

### 2단계: 사용자 - 옵션 선택

```
사용자가 다음을 선택:
- 색상: 검정색
- 사이즈: 3인
- 수량: 2
```

### 3단계: 앱 처리

```typescript
// React에서 선택 결과 처리
const selectedData = {
  productId: "PROD_12345",
  productName: "한샘 프리미엄 소파",
  quantity: 2,
  selectedOptions: {
    color: "검정",
    size: "3인",
  },
};

// GA4 트래킹
window.gtag("event", "add_to_cart", {
  product_id: selectedData.productId,
  product_name: selectedData.productName,
  quantity: selectedData.quantity,
});

// 한샘 쇼핑몰 연동
// window.location.href = `https://hanssem.com/add-to-cart?...`;
```

---

## 주의사항

### 1. 모든 옵션 유효성 검사

```tsx
const unselectedOptions = Object.keys(product.options).filter(
  (key) => !selectedOptions[key],
);

if (unselectedOptions.length > 0) {
  alert(`필수: ${unselectedOptions.join(", ")}`);
  return;
}
```

### 2. 에러 핸들링

- API 실패 시 (`template.error` 확인)
- 상태 코드 확인 (`template.response.status`)
- JSON 파싱 에러 (try-catch)

### 3. 이미지 최적화

```tsx
// React에서 이미지 로딩 최적화
<img
  src={product.productImage}
  alt={product.productName}
  loading="lazy"
  style={{
    width: "100px",
    height: "100px",
    objectFit: "cover",
  }}
/>;

// Android에서 Glide 사용
Glide.with(context)
  .load(product.optString("productImage"))
  .placeholder(R.drawable.ic_placeholder)
  .error(R.drawable.ic_error)
  .into(imageView);
```

### 4. 성능 최적화

- 불필요한 리렌더링 방지 (useMemo, useCallback)
- 네트워크 요청 캐싱
- 이미지 사이즈 최적화

---

## 참고 자료

- [React Custom Message Template 문서](https://github.com/sendbird/delight-ai-agent/blob/main/js/react/docs/features/messages.md#custom-message-template)
- [Android Custom Message Template 문서](https://github.com/sendbird/delight-ai-agent/blob/main/android/docs/features/messages.md#custom-message-template)
- [React Native Custom Message Template 문서](https://github.com/sendbird/delight-ai-agent/blob/main/js/react-native/docs/features/messages.md#custom-message-template)
- [iOS Messages 문서](https://github.com/sendbird/delight-ai-agent/blob/main/ios/docs/features/messages.md)
