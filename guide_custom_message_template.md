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

### 1단계: 커스텀 View Handler 구현

```kotlin
import android.content.Context
import android.view.LayoutInflater
import android.view.View
import android.widget.Button
import android.widget.EditText
import android.widget.ImageView
import android.widget.Spinner
import android.widget.TextView
import com.sendbird.sdk.aiagent.messenger.model.CustomMessageTemplateData
import com.sendbird.sdk.aiagent.messenger.interfaces.CustomMessageTemplateViewHandler
import com.sendbird.sdk.aiagent.messenger.interfaces.CustomMessageTemplateViewCallback
import com.bumptech.glide.Glide
import org.json.JSONObject

class ProductOptionTemplateHandler : CustomMessageTemplateViewHandler {
    override fun onCreateCustomMessageTemplateView(
        context: Context,
        message: BaseMessage,
        data: List<CustomMessageTemplateData>,
        callback: CustomMessageTemplateViewCallback
    ) {
        // "go-to-hanssem-mall-with-options" 템플릿 찾기
        val template = data.firstOrNull { it.id == "go-to-hanssem-mall-with-options" }
            ?: return callback.onViewReady(createFallbackView(context))

        // 에러 처리
        if (template.error != null) {
            return callback.onViewReady(createErrorView(context, template.error))
        }

        // 상태 코드 확인
        if (template.response.status != 200) {
            return callback.onViewReady(
                createErrorView(context, "상품 정보를 불러올 수 없습니다. (상태: ${template.response.status})")
            )
        }

        try {
            val content = template.response.content ?: return callback.onViewReady(createFallbackView(context))
            val product = JSONObject(content)
            val view = createProductOptionView(context, product)
            callback.onViewReady(view)
        } catch (e: Exception) {
            callback.onViewReady(createErrorView(context, "데이터 파싱 오류: ${e.message}"))
        }
    }

    private fun createProductOptionView(context: Context, product: JSONObject): View {
        val view = LayoutInflater.from(context).inflate(R.layout.custom_product_option_template, null)

        // 상품 이미지
        val imageView = view.findViewById<ImageView>(R.id.productImage)
        Glide.with(context)
            .load(product.optString("productImage"))
            .placeholder(R.drawable.ic_placeholder)
            .error(R.drawable.ic_error)
            .into(imageView)

        // 상품명
        view.findViewById<TextView>(R.id.productName).apply {
            text = product.optString("productName", "상품명 없음")
        }

        // 가격
        view.findViewById<TextView>(R.id.productPrice).apply {
            text = "₩${product.optLong("price", 0).let { String.format("%,d", it) }}"
        }

        // 설명
        product.optString("description").let { description ->
            if (description.isNotEmpty()) {
                view.findViewById<TextView>(R.id.productDescription).apply {
                    text = description
                }
            }
        }

        // 옵션들 추가
        val optionsContainer = view.findViewById<android.widget.LinearLayout>(R.id.optionsContainer)
        val options = product.optJSONObject("options") ?: JSONObject()

        val spinners = mutableMapOf<String, Spinner>()
        val spinnerValues = mutableMapOf<String, String>()

        options.keys().forEach { optionName ->
            val optionValues = options.getJSONArray(optionName)
            val valuesList = mutableListOf<String>()
            valuesList.add("-- 선택해주세요 --")

            for (i in 0 until optionValues.length()) {
                valuesList.add(optionValues.getString(i))
            }

            val spinner = Spinner(context).apply {
                adapter = android.widget.ArrayAdapter(
                    context,
                    android.R.layout.simple_spinner_item,
                    valuesList
                )
                setOnItemSelectedListener(object : android.widget.AdapterView.OnItemSelectedListener {
                    override fun onItemSelected(parent: android.widget.AdapterView<*>?, view: View?, position: Int, id: Long) {
                        spinnerValues[optionName] = if (position > 0) valuesList[position] else ""
                    }

                    override fun onNothingSelected(parent: android.widget.AdapterView<*>?) {}
                })
            }

            spinners[optionName] = spinner

            // 옵션 라벨 + 스피너
            val optionLabel = TextView(context).apply {
                text = "$optionName *"
                textSize = 13f
                setTextColor(context.resources.getColor(android.R.color.black))
            }

            optionsContainer.addView(optionLabel)
            optionsContainer.addView(spinner)
        }

        // 수량 입력
        val quantityInput = view.findViewById<EditText>(R.id.quantityInput).apply {
            setText("1")
            inputType = android.text.InputType.TYPE_CLASS_NUMBER
        }

        // 액션 버튼
        view.findViewById<Button>(R.id.addToCartButton).setOnClickListener {
            val unselectedOptions = spinnerValues.filterValues { it.isEmpty() }.keys
            if (unselectedOptions.isNotEmpty()) {
                android.widget.Toast.makeText(
                    context,
                    "다음 옵션을 선택해주세요: ${unselectedOptions.joinToString(", ")}",
                    android.widget.Toast.LENGTH_SHORT
                ).show()
                return@setOnClickListener
            }

            val quantity = quantityInput.text.toString().toIntOrNull() ?: 1
            val productId = product.optString("productId")
            val selectedData = mapOf(
                "productId" to productId,
                "quantity" to quantity.toString(),
                "options" to spinnerValues.toString()
            )

            // GA4 트래킹
            // FirebaseAnalytics.getInstance(context).logEvent("add_to_cart") { ... }

            android.widget.Toast.makeText(context, "장바구니에 추가되었습니다!", android.widget.Toast.LENGTH_SHORT).show()
        }

        return view
    }

    private fun createErrorView(context: Context, message: String): View {
        return TextView(context).apply {
            text = message
            setPadding(16, 16, 16, 16)
            setTextColor(android.graphics.Color.RED)
        }
    }

    private fun createFallbackView(context: Context): View {
        return TextView(context).apply {
            text = "지원하지 않는 템플릿입니다"
            setPadding(16, 16, 16, 16)
            setTextColor(context.resources.getColor(android.R.color.darker_gray))
        }
    }
}
```

### 2단계: Handler 등록

```kotlin
import com.sendbird.sdk.aiagent.messenger.AIAgentMessenger
import com.sendbird.sdk.aiagent.messenger.model.ConversationMessageListUIParams

// MessengerLauncher 생성 시
MessengerLauncher(context, "YOUR_AI_AGENT_ID").apply {
    // Custom message template handler 설정
    params = LauncherSettingsParams().apply {
        // Launcher 시작 시에 커스텀 handler를 설정할 수 없으므로,
        // Fragment 내에서 ConversationMessageListUIParams를 통해 설정해야 함
    }
}.attach()

// 또는 AIAgentAdapterProviders를 통해 전역 설정
AIAgentAdapterProviders.conversation = { channel, uiParams, containerGenerator ->
    uiParams.customMessageTemplateViewHandler = ProductOptionTemplateHandler()
    ConversationMessageListAdapter(channel, uiParams, containerGenerator)
}
```

---

## iOS 구현

iOS SDK의 경우, 아직 공식 custom message template 지원이 문서화되지 않았습니다.

향후 iOS 업데이트를 기다리거나, [iOS CHANGELOG](./ios/CHANGELOG.md)에서 최신 기능을 확인하세요.

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

## 관련 가이드

- [GA4 LiveMetric 통합 가이드](./guide_ga4_livemetric.md)
- [채팅창 상단에 상품 배너 다는 가이드](./guide_product_banner.md)
- [폰트 커스텀 가이드](./guide_font.md)

---

## 참고 자료

- [React Custom Message Template 문서](https://github.com/sendbird/delight-ai-agent/blob/main/js/react/docs/features/messages.md#custom-message-template)
- [Android Custom Message Template 문서](https://github.com/sendbird/delight-ai-agent/blob/main/android/docs/features/messages.md#custom-message-template)
- [React Native Custom Message Template 문서](https://github.com/sendbird/delight-ai-agent/blob/main/js/react-native/docs/features/messages.md#custom-message-template)
