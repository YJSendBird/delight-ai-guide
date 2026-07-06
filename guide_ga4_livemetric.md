# GA4 LiveMetric 통합 가이드

이 가이드는 Sendbird Delight AI Agent SDK의 **LiveMetric** 핸들러를 사용하여 실시간 분석 데이터를 GA4(Google Analytics 4)로 전송하는 방법을 설명합니다.

---

## 개요

**LiveMetric이란?**

- Sendbird AI Agent SDK에서 발생하는 **실시간 이벤트 데이터**
- 대화 시작, 메시지 전송, 사용자 액션 등의 메트릭
- `onLiveMetric` 핸들러로 수신하여 GA4로 전송 가능
- 사용자 행동 분석 및 비즈니스 인사이트 수집에 활용

**주요 활용 사례:**

- 📊 대화 시작/종료 추적
- 🛒 상품 옵션 선택 및 구매 액션 추적
- 📈 사용자 engagement 분석
- 🔍 AI Agent 응답 품질 모니터링
- 💰 구매 퍼널 추적

---

## LiveMetric 데이터 구조

### LiveMetric 인터페이스

```typescript
interface LiveMetric {
  category: string; // 메트릭 카테고리 (e.g., "conversation", "message", "template_action")
  metricKey: string; // 메트릭 키 (e.g., "conversation:conversation_open", "custom_template:button_click")
  data: Record<string, any>; // 추가 데이터 객체
  timestamp?: number; // 이벤트 발생 시간 (밀리초)
}
```

### 주요 메트릭 예제

```typescript
// 대화 시작
{
  category: "conversation",
  metricKey: "conversation:conversation_open",
  data: {
    conversationId: "12345",
    timestamp: 1234567890000
  }
}

// Custom Message Template 렌더링
{
  category: "template",
  metricKey: "template:rendered",
  data: {
    templateId: "go-to-hanssem-mall-with-options",
    productId: "PROD_12345"
  }
}

// Custom Template 액션
{
  category: "template_action",
  metricKey: "template_action:add_to_cart",
  data: {
    templateId: "go-to-hanssem-mall-with-options",
    productId: "PROD_12345",
    quantity: 2,
    options: { color: "검정", size: "3인" }
  }
}
```

---

## Web (React) 구현

### 1단계: LiveMetric 핸들러 구현

```tsx
import {
  AgentProviderContainer,
  Conversation,
  type LiveMetric,
} from "@sendbird/ai-agent-messenger-react";

export function App() {
  // LiveMetric 핸들러
  const handleLiveMetric = (metric: LiveMetric) => {
    console.log("📊 LiveMetric 수신:", {
      category: metric.category,
      metricKey: metric.metricKey,
      data: metric.data,
    });

    // GA4로 전송
    sendToGA4(metric);

    // 커스텀 처리
    handleCustomMetrics(metric);
  };

  return (
    <AgentProviderContainer
      appId="YOUR_APP_ID"
      aiAgentId="YOUR_AI_AGENT_ID"
      handlers={{
        onLiveMetric: handleLiveMetric,
      }}
    >
      <Conversation />
    </AgentProviderContainer>
  );
}
```

### 2단계: GA4 전송 함수

```typescript
/**
 * LiveMetric을 GA4로 전송
 */
function sendToGA4(metric: LiveMetric): void {
  if (!window.gtag) {
    console.warn("GA4 (gtag) 라이브러리가 로드되지 않았습니다");
    return;
  }

  // 이벤트명 생성 (category:metricKey 형식)
  const eventName = `ai_agent_${metric.category}_${metric.metricKey.split(":")[1]}`;

  // GA4 이벤트 전송
  window.gtag("event", eventName, {
    category: metric.category,
    metric_key: metric.metricKey,
    ...metric.data,
  });
}
```

### 3단계: 커스텀 메트릭 처리

```typescript
/**
 * SDK 메트릭별 커스텀 처리
 */
function handleCustomMetrics(metric: LiveMetric): void {
  switch (metric.metricKey) {
    // 대화 시작
    case "conversation:conversation_open":
      console.log("✅ 사용자가 대화를 시작했습니다");
      trackConversationStart(metric.data);
      break;

    // 대화 종료
    case "conversation:conversation_close":
      console.log("❌ 사용자가 대화를 종료했습니다");
      trackConversationEnd(metric.data);
      break;

    // 메시지 수신
    case "message:message_received":
      console.log("💬 AI Agent 메시지 수신");
      trackMessageReceived(metric.data);
      break;

    // Custom Template 렌더링
    case "template:rendered":
      console.log("🎨 Custom Message Template 렌더링됨");
      trackTemplateRendered(metric.data);
      break;

    // Custom Template 액션
    case "template_action:add_to_cart":
      console.log("🛒 사용자가 장바구니 추가 클릭");
      trackAddToCart(metric.data);
      break;

    default:
      console.log("📝 기타 메트릭:", metric.metricKey);
  }
}

/**
 * 대화 시작 추적
 */
function trackConversationStart(data: Record<string, any>): void {
  // Amplitude, Mixpanel 등 다른 분석 도구로도 전송 가능
  console.log("대화 시작 데이터:", data);
}

/**
 * 대화 종료 추적
 */
function trackConversationEnd(data: Record<string, any>): void {
  console.log("대화 종료 데이터:", data);
}

/**
 * 메시지 수신 추적
 */
function trackMessageReceived(data: Record<string, any>): void {
  console.log("메시지 수신 데이터:", data);
}

/**
 * Template 렌더링 추적
 */
function trackTemplateRendered(data: Record<string, any>): void {
  console.log("Template 렌더링 데이터:", data);
}

/**
 * 장바구니 추가 추적
 */
function trackAddToCart(data: Record<string, any>): void {
  console.log("장바구니 추가 데이터:", data);
  // 비즈니스 로직 (한샘 쇼핑몰 연동 등)
}
```

### 4단계: GA4 태그 설정 (HTML)

```html
<!-- Google Analytics 4 -->
<script
  async
  src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"
></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag() {
    dataLayer.push(arguments);
  }
  gtag("js", new Date());
  gtag("config", "G-XXXXXXXXXX");
</script>
```

---

## Custom Message Template에서 추가 액션 추적

### Custom Template 컴포넌트에서 GA4 전송

```tsx
import React, { useState } from "react";

interface Props {
  extendedMessagePayload?: {
    custom_message_templates?: CustomMessageTemplateData[];
  };
}

export const ProductOptionTemplate: React.FC<Props> = ({
  extendedMessagePayload,
}) => {
  const template = extendedMessagePayload?.custom_message_templates?.find(
    (t) => t.id === "go-to-hanssem-mall-with-options",
  );

  if (!template) return null;

  let product: ProductWithOptions;
  try {
    product = JSON.parse(template.response.content ?? "{}");
  } catch (e) {
    return <div>데이터 파싱 오류</div>;
  }

  return <ProductOptionCard product={product} />;
};

interface ProductWithOptions {
  productId: string;
  productName: string;
  productImage: string;
  price: number;
  options: {
    [key: string]: string[];
  };
}

const ProductOptionCard: React.FC<{ product: ProductWithOptions }> = ({
  product,
}) => {
  const [selectedOptions, setSelectedOptions] = useState<
    Record<string, string>
  >({});
  const [quantity, setQuantity] = useState(1);

  /**
   * 옵션 변경 시 GA4 추적
   */
  const handleOptionChange = (optionName: string, value: string) => {
    setSelectedOptions((prev) => ({
      ...prev,
      [optionName]: value,
    }));

    // GA4 커스텀 이벤트
    if (window.gtag) {
      window.gtag("event", "custom_template_option_change", {
        template_id: "go-to-hanssem-mall-with-options",
        product_id: product.productId,
        option_name: optionName,
        option_value: value,
      });
    }
  };

  /**
   * 수량 변경 시 GA4 추적
   */
  const handleQuantityChange = (newQuantity: number) => {
    setQuantity(newQuantity);

    if (window.gtag) {
      window.gtag("event", "custom_template_quantity_change", {
        template_id: "go-to-hanssem-mall-with-options",
        product_id: product.productId,
        quantity: newQuantity,
      });
    }
  };

  /**
   * 장바구니 추가 시 GA4 추적
   */
  const handleAddToCart = () => {
    // 옵션 유효성 검사
    const unselectedOptions = Object.keys(product.options).filter(
      (key) => !selectedOptions[key],
    );

    if (unselectedOptions.length > 0) {
      alert(`다음 옵션을 선택해주세요: ${unselectedOptions.join(", ")}`);
      return;
    }

    // GA4 구매 이벤트 (E-commerce)
    if (window.gtag) {
      window.gtag("event", "add_to_cart", {
        currency: "KRW",
        items: [
          {
            item_id: product.productId,
            item_name: product.productName,
            price: product.price,
            quantity: quantity,
            item_category: "furniture",
            item_variant: JSON.stringify(selectedOptions),
          },
        ],
      });
    }

    alert("장바구니에 추가되었습니다!");
  };

  /**
   * 바로 구매 시 GA4 추적
   */
  const handleBuyNow = () => {
    // 옵션 유효성 검사
    const unselectedOptions = Object.keys(product.options).filter(
      (key) => !selectedOptions[key],
    );

    if (unselectedOptions.length > 0) {
      alert(`다음 옵션을 선택해주세요: ${unselectedOptions.join(", ")}`);
      return;
    }

    // GA4 구매 시작 이벤트
    if (window.gtag) {
      window.gtag("event", "begin_checkout", {
        currency: "KRW",
        items: [
          {
            item_id: product.productId,
            item_name: product.productName,
            price: product.price,
            quantity: quantity,
            item_variant: JSON.stringify(selectedOptions),
          },
        ],
      });
    }

    alert("구매 페이지로 이동합니다!");
    // window.location.href = `https://hanssem.com/checkout?...`;
  };

  return (
    <div
      style={{
        border: "1px solid #e0e0e0",
        borderRadius: "12px",
        padding: "16px",
      }}
    >
      {/* 상품 이미지 및 정보 */}
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
          <p style={{ margin: "0", fontSize: "18px", fontWeight: "bold" }}>
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
                <span style={{ color: "#e74c3c" }}>*</span>
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
            handleQuantityChange(Math.max(1, parseInt(e.target.value) || 1))
          }
          style={{
            width: "60px",
            padding: "8px",
            border: "1px solid #ddd",
            borderRadius: "4px",
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
          }}
        >
          장바구니 추가
        </button>
        <button
          onClick={handleBuyNow}
          style={{
            flex: 1,
            padding: "12px",
            backgroundColor: "#e74c3c",
            color: "white",
            border: "none",
            borderRadius: "6px",
          }}
        >
          바로 구매
        </button>
      </div>
    </div>
  );
};
```

---

## Android 구현

### 1단계: LiveMetric 리스너 설정

```kotlin
import com.sendbird.sdk.aiagent.messenger.AIAgentMessenger
import com.sendbird.sdk.aiagent.messenger.model.LiveMetric
import com.google.firebase.analytics.FirebaseAnalytics

class MainActivity : AppCompatActivity() {
    private lateinit var firebaseAnalytics: FirebaseAnalytics

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Firebase Analytics 초기화
        firebaseAnalytics = FirebaseAnalytics.getInstance(this)

        // LiveMetric 리스너 설정
        AIAgentMessenger.setLiveMetricListener { metric ->
            handleLiveMetric(metric)
        }
    }

    /**
     * LiveMetric 핸들러
     */
    private fun handleLiveMetric(metric: LiveMetric) {
        Log.d("LiveMetric", "수신: ${metric.category} / ${metric.metricKey}")
        Log.d("LiveMetric", "데이터: ${metric.data}")

        // GA4 (Firebase Analytics)로 전송
        sendToFirebaseAnalytics(metric)

        // 커스텀 처리
        handleCustomMetrics(metric)
    }

    /**
     * Firebase Analytics (GA4)로 전송
     */
    private fun sendToFirebaseAnalytics(metric: LiveMetric) {
        val bundle = Bundle().apply {
            putString("category", metric.category)
            putString("metric_key", metric.metricKey)

            // 추가 데이터 전송
            metric.data.forEach { (key, value) ->
                when (value) {
                    is String -> putString(key, value)
                    is Int -> putInt(key, value)
                    is Long -> putLong(key, value)
                    is Double -> putDouble(key, value)
                    else -> putString(key, value.toString())
                }
            }
        }

        val eventName = "ai_agent_${metric.metricKey.replace(":", "_")}"
        firebaseAnalytics.logEvent(eventName, bundle)
    }

    /**
     * 메트릭별 커스텀 처리
     */
    private fun handleCustomMetrics(metric: LiveMetric) {
        when (metric.metricKey) {
            "conversation:conversation_open" -> {
                Log.d("Conversation", "대화 시작됨")
                trackConversationStart(metric.data)
            }

            "conversation:conversation_close" -> {
                Log.d("Conversation", "대화 종료됨")
                trackConversationEnd(metric.data)
            }

            "template:rendered" -> {
                Log.d("Template", "Custom Message Template 렌더링됨")
                trackTemplateRendered(metric.data)
            }

            "template_action:add_to_cart" -> {
                Log.d("Template", "장바구니 추가 클릭")
                trackAddToCart(metric.data)
            }
        }
    }

    private fun trackConversationStart(data: Map<String, Any>) {
        Log.d("Analytics", "대화 시작: $data")
    }

    private fun trackConversationEnd(data: Map<String, Any>) {
        Log.d("Analytics", "대화 종료: $data")
    }

    private fun trackTemplateRendered(data: Map<String, Any>) {
        Log.d("Analytics", "Template 렌더링: $data")
    }

    private fun trackAddToCart(data: Map<String, Any>) {
        Log.d("Analytics", "장바구니 추가: $data")
    }
}
```

### 2단계: Custom Message Template에서 액션 추적

```kotlin
import com.google.firebase.analytics.FirebaseAnalytics
import org.json.JSONObject

class ProductOptionTemplateHandler : CustomMessageTemplateViewHandler {
    override fun onCreateCustomMessageTemplateView(
        context: Context,
        message: BaseMessage,
        data: List<CustomMessageTemplateData>,
        callback: CustomMessageTemplateViewCallback
    ) {
        val template = data.firstOrNull { it.id == "go-to-hanssem-mall-with-options" }
            ?: return

        try {
            val content = template.response.content ?: return
            val product = JSONObject(content)
            val firebaseAnalytics = FirebaseAnalytics.getInstance(context)

            val view = createProductOptionView(context, product, firebaseAnalytics)
            callback.onViewReady(view)
        } catch (e: Exception) {
            Log.e("ProductOption", "Error: ${e.message}")
        }
    }

    private fun createProductOptionView(
        context: Context,
        product: JSONObject,
        firebaseAnalytics: FirebaseAnalytics
    ): View {
        val view = LayoutInflater.from(context).inflate(R.layout.custom_product_option_template, null)

        // 상품 정보 설정
        val productId = product.optString("productId")
        val productName = product.optString("productName", "상품명 없음")
        val price = product.optLong("price", 0)

        // 옵션 Spinner 설정
        val optionsContainer = view.findViewById<android.widget.LinearLayout>(R.id.optionsContainer)
        val options = product.optJSONObject("options") ?: JSONObject()
        val selectedOptions = mutableMapOf<String, String>()

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
                    override fun onItemSelected(parent: android.widget.AdapterView<*>?, v: View?, position: Int, id: Long) {
                        val value = if (position > 0) valuesList[position] else ""
                        selectedOptions[optionName] = value

                        // GA4 추적
                        val bundle = Bundle().apply {
                            putString("product_id", productId)
                            putString("option_name", optionName)
                            putString("option_value", value)
                        }
                        firebaseAnalytics.logEvent("custom_template_option_change", bundle)
                    }

                    override fun onNothingSelected(parent: android.widget.AdapterView<*>?) {}
                })
            }

            optionsContainer.addView(spinner)
        }

        // 수량 입력
        val quantityInput = view.findViewById<EditText>(R.id.quantityInput).apply {
            setText("1")
            inputType = android.text.InputType.TYPE_CLASS_NUMBER
        }

        // 장바구니 추가 버튼
        view.findViewById<Button>(R.id.addToCartButton).setOnClickListener {
            val quantity = quantityInput.text.toString().toIntOrNull() ?: 1

            // GA4 추적 (E-commerce)
            val bundle = Bundle().apply {
                putString("currency", "KRW")
                putString("item_id", productId)
                putString("item_name", productName)
                putLong("price", price)
                putInt("quantity", quantity)
                putString("item_variant", selectedOptions.toString())
            }
            firebaseAnalytics.logEvent("add_to_cart", bundle)

            android.widget.Toast.makeText(context, "장바구니에 추가되었습니다!", android.widget.Toast.LENGTH_SHORT).show()
        }

        return view
    }
}
```

---

## iOS 구현

### 1단계: LiveMetric 핸들러 설정

`AIAgentMessenger.onLiveMetricHandler`에 핸들러를 등록하면 SDK가 발생시키는 모든 LiveMetric을 받을 수 있습니다.

```swift
import UIKit
import SendbirdAIAgentMessenger
import FirebaseAnalytics

class AppDelegate: UIResponder, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        // Firebase Analytics 초기화
        FirebaseApp.configure()

        // AI Agent SDK 초기화 후 LiveMetric 핸들러 설정
        AIAgentMessenger.onLiveMetricHandler = { [weak self] metric in
            self?.handleLiveMetric(metric)
        }

        return true
    }

    // MARK: - LiveMetric 핸들러

    private func handleLiveMetric(_ metric: AIAgentMessenger.LiveMetric) {
        print("[LiveMetric] [\(metric.category.rawValue)] \(metric.metricKey) — \(metric.data)")

        // Firebase Analytics (GA4)로 전송
        sendToFirebaseAnalytics(metric)

        // 커스텀 처리
        handleCustomMetrics(metric)
    }
}
```

### LiveMetric 데이터 구조 (iOS)

```swift
// AIAgentMessenger.LiveMetric
public class LiveMetric {
    public enum Category: String {
        case handoff        // 핸드오프 시작/성공/실패
        case conversation   // 대화 초기화/열림/닫힘
        case message        // 메시지 송수신
        case connection     // 연결 상태
    }

    public let category: Category         // 메트릭 카테고리
    public let metricKey: String          // 예: "conversation_open"
    open var data: [String: String] { }   // 카테고리별 payload
}
```

주요 `metricKey` 값:

| 카테고리 | metricKey |
| --- | --- |
| conversation | `conversation_initialized`, `conversation_open`, `conversation_closed` |
| handoff | `handoff_started`, `handoff_succeeded`, `handoff_failed` |
| message | `incoming_message_received`, `outgoing_message_<sendingStatus>` |
| connection | `connection_connected` 등 |

### 2단계: Firebase Analytics (GA4) 전송 함수

```swift
// MARK: - Firebase Analytics 전송

private func sendToFirebaseAnalytics(_ metric: AIAgentMessenger.LiveMetric) {
    // 이벤트명 생성 (category_metricKey 형식, 최대 40자)
    let eventName = String("ai_agent_\(metric.metricKey)".prefix(40))

    // 파라미터 구성
    var params: [String: Any] = [
        "category": metric.category.rawValue,
        "metric_key": metric.metricKey,
    ]

    // 추가 데이터 병합 (data는 [String: String])
    for (key, value) in metric.data {
        params[key] = value
    }

    Analytics.logEvent(eventName, parameters: params)
}
```

### 3단계: 커스텀 메트릭 처리

```swift
// MARK: - 메트릭별 커스텀 처리

private func handleCustomMetrics(_ metric: AIAgentMessenger.LiveMetric) {
    switch metric.metricKey {
    case "conversation_open":
        print("✅ 사용자가 대화를 시작했습니다")
        trackConversationStart(metric.data)

    case "conversation_closed":
        print("❌ 사용자가 대화를 종료했습니다")
        trackConversationEnd(metric.data)

    case "incoming_message_received":
        print("💬 AI Agent 메시지 수신")
        trackMessageReceived(metric.data)

    case "handoff_started":
        print("🤝 상담원 핸드오프 시작")
        trackHandoffStarted(metric.data)

    default:
        print("📝 기타 메트릭: \(metric.metricKey)")
    }
}

private func trackConversationStart(_ data: [String: String]) {
    print("대화 시작 데이터: \(data)")
}

private func trackConversationEnd(_ data: [String: String]) {
    print("대화 종료 데이터: \(data)")
}

private func trackMessageReceived(_ data: [String: String]) {
    print("메시지 수신 데이터: \(data)")
}

private func trackHandoffStarted(_ data: [String: String]) {
    print("핸드오프 시작 데이터: \(data)")
}
```

### 4단계: Custom Message Template에서 액션 추적

커스텀 템플릿 뷰의 버튼 액션에서 GA4 E-commerce 이벤트를 직접 전송합니다. (템플릿 뷰 구현은 [Custom Message Template 가이드](./guide_custom_message_template.md)의 iOS 섹션 참고)

> **1.15.0 주의**: 커스텀 컴포넌트 서브클래스에 stored property를 선언하면 크래시합니다. 상품 정보는 static 저장소에 보관합니다 (static은 SDK의 프로퍼티 순회 대상이 아니라 안전).

```swift
import UIKit
import SendbirdAIAgentMessenger
import FirebaseAnalytics

final class ProductOptionTemplateView: SBACustomMessageTemplateView {
    // stored property 금지(1.15.0 크래시) — static 저장소 사용
    private struct TrackedProduct {
        var id = ""
        var name = ""
        var price = 0
    }
    private static var trackedProduct = TrackedProduct()

    override func configure(with customMessageTemplates: [SBACustomMessageTemplateData]?) {
        super.configure(with: customMessageTemplates)

        guard let template = customMessageTemplates?.first(where: {
            $0.templateId == "go-to-hanssem-mall-with-options"
        }),
              let content = template.response.content,
              let jsonData = content.data(using: .utf8),
              let product = try? JSONSerialization.jsonObject(with: jsonData) as? [String: Any]
        else { return }

        Self.trackedProduct = TrackedProduct(
            id: product["productId"] as? String ?? "",
            name: product["productName"] as? String ?? "",
            price: product["price"] as? Int ?? 0
        )
    }

    @objc private func addToCartTapped() {
        let product = Self.trackedProduct

        // GA4 E-commerce 이벤트 (add_to_cart)
        Analytics.logEvent(AnalyticsEventAddToCart, parameters: [
            AnalyticsParameterCurrency: "KRW",
            AnalyticsParameterItems: [[
                AnalyticsParameterItemID: product.id,
                AnalyticsParameterItemName: product.name,
                AnalyticsParameterPrice: product.price,
                AnalyticsParameterQuantity: 1,
                AnalyticsParameterItemCategory: "furniture",
            ]],
        ])

        // 앱 후속 처리로 이벤트 전달
        sendEvent(name: "add_to_cart", data: ["productId": product.id])
    }

    @objc private func buyNowTapped() {
        let product = Self.trackedProduct

        // GA4 E-commerce 이벤트 (begin_checkout)
        Analytics.logEvent(AnalyticsEventBeginCheckout, parameters: [
            AnalyticsParameterCurrency: "KRW",
            AnalyticsParameterItems: [[
                AnalyticsParameterItemID: product.id,
                AnalyticsParameterItemName: product.name,
                AnalyticsParameterPrice: product.price,
                AnalyticsParameterQuantity: 1,
            ]],
        ])

        sendEvent(name: "buy_now", data: ["productId": product.id])
    }
}
```

### Firebase Analytics SPM 설정

`Package.swift` 또는 Xcode SPM에서 Firebase Analytics 추가:

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/firebase/firebase-ios-sdk", from: "10.0.0"),
],
targets: [
    .target(
        name: "YourApp",
        dependencies: [
            .product(name: "FirebaseAnalytics", package: "firebase-ios-sdk"),
        ]
    ),
]
```

`GoogleService-Info.plist`를 프로젝트에 추가하고, `AppDelegate`에서 `FirebaseApp.configure()`를 호출하세요.

---

## GA4 이벤트 분석

### 권장 이벤트 구조

```typescript
// 표준 E-commerce 이벤트
window.gtag("event", "add_to_cart", {
  currency: "KRW",
  items: [
    {
      item_id: "PROD_12345",
      item_name: "한샘 프리미엄 소파",
      price: 599000,
      quantity: 1,
      item_category: "furniture",
      item_variant: "색상: 검정, 사이즈: 3인",
    },
  ],
});

// 구매 완료 이벤트
window.gtag("event", "purchase", {
  currency: "KRW",
  transaction_id: "TX_12345",
  value: 599000,
  items: [
    {
      item_id: "PROD_12345",
      item_name: "한샘 프리미엄 소파",
      price: 599000,
      quantity: 1,
    },
  ],
});

// 커스텀 이벤트
window.gtag("event", "ai_agent_conversation_start", {
  ai_agent_id: "YOUR_AI_AGENT_ID",
  conversation_id: "CONV_12345",
  timestamp: Date.now(),
});
```

### GA4 대시보드에서 확인

1. **GA4 콘솔** → 보고서 → 전환
2. **수익화** → E-commerce 구매
3. **사용자** → 개요
4. **커스텀 정의** → 커스텀 이벤트 생성

---

## 주의사항

### 1. GA4 라이브러리 확인

```typescript
// GA4 (gtag) 라이브러리가 로드되었는지 확인
if (window.gtag) {
  window.gtag("event", "test_event");
} else {
  console.warn("GA4 라이브러리가 로드되지 않았습니다");
}
```

### 2. 개인 정보 보호

- PII (Personally Identifiable Information) 전송 금지
- 민감한 정보 (비밀번호, 신용카드 등) 제외
- GA4 이용약관 준수

### 3. 이벤트 네이밍 규칙

- 소문자 및 언더스코어 사용 (snake_case)
- 최대 40자 제한
- 명확하고 일관된 이름 사용

```typescript
// ✅ 좋은 예
window.gtag("event", "custom_template_add_to_cart");

// ❌ 나쁜 예
window.gtag("event", "ADD_TO_CART_BUTTON_CLICKED_BY_USER");
```

### 4. 실시간 확인

GA4 Real-time 보고서에서 이벤트가 즉시 수집되는지 확인:

```
GA4 콘솔 → 보고서 → 실시간 → 이벤트 수 확인
```

---

## 참고 자료

- [JavaScript GA4 상세 가이드 (Logger 구현)](https://gist.github.com/bang9/40a93bd2b1cdbd84e4a7ac6d6e33f7cf#file-logging-md)
- [LiveMetric 타입 정의 (React SDK)](https://github.com/sendbird/delight-ai-agent/blob/main/js/react/CHANGELOG.md)
- [Google Analytics 4 문서](https://support.google.com/analytics)
- [Firebase Analytics 문서 (Android)](https://firebase.google.com/docs/analytics)
- [GA4 E-commerce 이벤트](https://developers.google.com/analytics/devguides/collection/ga4/ecommerce)
