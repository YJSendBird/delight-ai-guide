# 채팅창 상단에 상품 배너 다는 가이드

이 가이드는 Context Object를 통해 상품 ID를 전달받고, 이를 이용해 채팅창 상단에 상품 배너를 표시하는 방법을 설명합니다.

---

## 개요

**Flow:**

1. 앱에서 사용자가 보고 있는 상품의 ID를 파악
2. Context Object를 통해 AI Agent SDK에 상품 ID 전달
3. SDK 커스텀화를 통해 헤더 영역에 상품 배너 추가
4. 상품 정보를 API로부터 가져와 배너에 표시

---

## iOS

### 1단계: Context Object로 Product ID 전달

```swift
AIAgentMessenger.presentConversation(
    aiAgentId: "YOUR_AI_AGENT_ID"
) { params in
    // Context object를 통해 상품 ID 전달
    params.context = [
        "product_id": "{PRODUCT_ID}",
        "product_name": "{PRODUCT_NAME}"
    ]
    params.parent = self
    params.presentationStyle = .fullScreen
}
```

### 2단계: 상품 배너 뷰 만들기

```swift
import UIKit

final class ProductBannerView: UIView {
    private let imageView = UIImageView()
    private let nameLabel = UILabel()
    private let priceLabel = UILabel()

    override init(frame: CGRect) {
        super.init(frame: frame)
        backgroundColor = .secondarySystemBackground

        imageView.contentMode = .scaleAspectFill
        imageView.clipsToBounds = true
        imageView.layer.cornerRadius = 8

        nameLabel.font = .systemFont(ofSize: 15, weight: .semibold)
        nameLabel.numberOfLines = 2

        priceLabel.font = .systemFont(ofSize: 14, weight: .bold)

        let textStack = UIStackView(arrangedSubviews: [nameLabel, priceLabel])
        textStack.axis = .vertical
        textStack.spacing = 4

        let stack = UIStackView(arrangedSubviews: [imageView, textStack])
        stack.axis = .horizontal
        stack.spacing = 12
        stack.alignment = .center

        addSubview(stack)
        stack.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            imageView.widthAnchor.constraint(equalToConstant: 48),
            imageView.heightAnchor.constraint(equalToConstant: 48),
            stack.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 16),
            stack.trailingAnchor.constraint(equalTo: trailingAnchor, constant: -16),
            stack.topAnchor.constraint(equalTo: topAnchor, constant: 8),
            stack.bottomAnchor.constraint(equalTo: bottomAnchor, constant: -8)
        ])
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    func configure(name: String, price: String, image: UIImage?) {
        nameLabel.text = name
        priceLabel.text = price
        imageView.image = image
    }
}
```

### 3단계: 헤더 커스텀으로 배너 붙이기

`SBAConversationModule.Header`를 서브클래싱하고 모듈 셋에 등록하면 대화 화면 헤더를 확장할 수 있습니다.

> **1.15.0 주의**: 커스텀 컴포넌트 서브클래스에 **stored property를 선언하면 대화 진입 시 크래시**합니다 (SDK 내부 의존성 주입이 프로퍼티를 순회하다 dynamic cast 오류 발생, 다음 릴리즈에서 수정 예정). 아래처럼 뷰는 `layoutBody()`에서 만들고, 참조가 필요하면 `tag` + computed property로 접근하세요. computed property와 static은 안전합니다.

```swift
import SendbirdAIAgentMessenger
import UIKit

final class ProductBannerHeader: SBAConversationModule.Header {
    private static let bannerTag = 990_001

    // stored property 금지(1.15.0 크래시) — computed로 조회
    private var bannerView: ProductBannerView? {
        viewWithTag(Self.bannerTag) as? ProductBannerView
    }

    // 기본 헤더(layoutBody) 아래에 배너를 붙인다.
    override func layoutBody() -> UIView {
        let banner = ProductBannerView()
        banner.tag = Self.bannerTag
        banner.heightAnchor.constraint(equalToConstant: 64).isActive = true

        loadProductInfo(into: banner)

        // 배너가 필요한 대화에서만 vStack에 추가하세요.
        // 필요 없는 경우에는 super.layoutBody()만 반환하면 기본 헤더 그대로입니다.
        return SBALinearLayout.vStack {
            super.layoutBody()
            banner
        }
    }

    private func loadProductInfo(into banner: ProductBannerView) {
        // 진입 시 전달한 product_id로 상품 정보를 조회해서 배너에 반영
        fetchProductInfo(productId: "{PRODUCT_ID}") { product in
            DispatchQueue.main.async {
                banner.configure(
                    name: product.name,
                    price: "₩\(product.price)",
                    image: nil
                )
            }
        }
    }
}

// 대화 화면을 열기 전에 등록
SBAModuleSet.ConversationModule.HeaderComponent = ProductBannerHeader.self
```

### 4단계: 상품 정보 조회

```swift
struct Product: Codable {
    let id: String
    let name: String
    let imageUrl: String
    let price: Double
    let description: String
}

func fetchProductInfo(productId: String, completion: @escaping (Product) -> Void) {
    let url = URL(string: "https://api.example.com/products/\(productId)")!

    URLSession.shared.dataTask(with: url) { data, _, _ in
        guard let data = data,
              let product = try? JSONDecoder().decode(Product.self, from: data) else { return }
        completion(product)
    }.resume()
}
```

### 5단계: 다른 대화에서는 배너 숨김 (예시)

배너는 제품 페이지에서 연 대화에만 의미가 있으므로, 대화 목록에서 다른 대화로 진입한 경우에는 숨기는 편이 좋습니다. 아래는 **channelURL 비교** 방식의 예시입니다 — 제품 대화를 열 때 채널 URL을 기록해 두고, 헤더의 현재 채널과 비교해서 다르면 숨깁니다.

> 현재 채널(`dataSource(with: .channel)`)은 비동기로 로드되므로, 배너를 기본 숨김으로 시작하고 채널이 로드된 뒤 판정하는 구성을 권장합니다. 판정 시점과 재시도 방식은 앱 구조에 맞게 조정하세요.

```swift
// 제품 대화를 여는 시점에 채널 URL 기록
enum ProductConversationRouter {
    static var activeChannelURL: String?
}

// 헤더에서 현재 채널과 비교 (예: didMoveToWindow 등 채널 로드 이후 시점)
private func updateBannerVisibility() {
    let channel: BaseChannel? = dataSource(with: .channel)
    guard let currentURL = channel?.channelURL else { return }
    bannerView?.isHidden = (currentURL != ProductConversationRouter.activeChannelURL)
}
```

### 제약사항

- 헤더 높이를 키우는 대신, 배너를 헤더 아래 별도 뷰로 얹는 구성도 가능합니다.
- 헤더의 기본 버튼(menu/close/handoff)은 `Header`의 `@LayoutSlot` 프로퍼티(`menuButton`, `closeButton`, `handoffButton`)로 노출되어 있어, `layoutLeftItems()` / `layoutRightItems()` 오버라이드로 배치를 조정할 수 있습니다.

---

## Android

### 1단계: Context Object로 Product ID 전달

```kotlin
MessengerLauncher(this, "YOUR_AI_AGENT_ID", LauncherSettingsParams(
    context = mapOf(
        "productId" to "PROD_12345",
        "productName" to "Sample Product"
    )
)).attach()
```

또는 전체 화면 모드:

```kotlin
startActivity(
    MessengerActivity.newIntentForConversation(
        context = this,
        aiAgentId = "YOUR_AI_AGENT_ID",
        conversationSettingsParams = ConversationSettingsParams(
            context = mapOf(
                "productId" to "PROD_12345",
                "productName" to "Sample Product"
            )
        )
    )
)
```

### 2단계: 상품 정보 조회

```kotlin
class ProductBannerManager {
    fun fetchProductInfo(productId: String, callback: (Product) -> Unit) {
        // Retrofit 또는 다른 HTTP 클라이언트로 API 호출
        val apiService = RetrofitClient.create(ProductApiService::class.java)

        apiService.getProduct(productId).enqueue(object : Callback<Product> {
            override fun onResponse(call: Call<Product>, response: Response<Product>) {
                response.body()?.let { callback(it) }
            }

            override fun onFailure(call: Call<Product>, t: Throwable) {
                // 에러 처리
            }
        })
    }
}

data class Product(
    val id: String,
    val name: String,
    val imageUrl: String,
    val price: Double,
    val description: String
)
```

### 3단계: 헤더 테마 커스텀화

Android에서는 `ConversationHeaderTheme`을 통해 헤더를 커스텀할 수 있습니다.

```kotlin
// MessengerTheme 커스텀화
val customTheme = object : MessengerTheme {
    override val conversation: ConversationTheme
        get() = object : ConversationTheme {
            override val header: ConversationHeaderTheme
                get() = object : ConversationHeaderTheme {
                    // 헤더 배경색, 높이 등 커스텀화
                }
        }
}

AIAgentMessenger.setTheme(customTheme)
```

### 제약사항

- Android 커스텀화는 색상, 배경 등 제한적입니다.
- 상품 배너를 위한 완전히 새로운 View를 추가하려면, Messenger 위에 별도 Fragment나 View를 오버레이하는 것을 권장합니다.

---

## 웹 (JavaScript / React)

### 1단계: Context Object로 Product ID 전달

```tsx
import { FixedMessenger } from "@sendbird/ai-agent-messenger-react";

function App() {
  return (
    <FixedMessenger
      appId="YOUR_APP_ID"
      aiAgentId="YOUR_AI_AGENT_ID"
      contextData={{
        productId: "PROD_12345",
        productName: "Sample Product",
      }}
    />
  );
}
```

또는 AgentProviderContainer 사용 시:

```tsx
import {
  AgentProviderContainer,
  Conversation,
} from "@sendbird/ai-agent-messenger-react";

function App() {
  return (
    <AgentProviderContainer
      appId="YOUR_APP_ID"
      contextData={{
        productId: "PROD_12345",
        productName: "Sample Product",
      }}
    >
      <Conversation aiAgentId="YOUR_AI_AGENT_ID" />
    </AgentProviderContainer>
  );
}
```

### 2단계: 헤더 커스텀화를 통해 상품 배너 추가

```tsx
import {
  ConversationHeaderLayout,
  ConversationHeaderProps,
} from "@sendbird/ai-agent-messenger-react";
import { useState, useEffect } from "react";

// 상품 정보 인터페이스
interface Product {
  id: string;
  name: string;
  imageUrl: string;
  price: number;
  description: string;
}

// 상품 배너 컴포넌트
const ProductBanner = ({ productId }: { productId: string }) => {
  const [product, setProduct] = useState<Product | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Context에서 받은 productId로 상품 정보 조회
    fetch(`https://api.example.com/products/${productId}`)
      .then((res) => res.json())
      .then((data) => {
        setProduct(data);
        setLoading(false);
      })
      .catch((err) => {
        console.error("Failed to fetch product:", err);
        setLoading(false);
      });
  }, [productId]);

  if (loading) {
    return (
      <div style={{ padding: "12px", textAlign: "center" }}>
        상품 정보 로딩 중...
      </div>
    );
  }

  if (!product) {
    return null;
  }

  return (
    <div
      style={{
        padding: "12px 16px",
        borderBottom: "1px solid #e0e0e0",
        display: "flex",
        gap: "12px",
        backgroundColor: "#fafafa",
      }}
    >
      {/* 상품 이미지 */}
      <img
        src={product.imageUrl}
        alt={product.name}
        style={{
          width: "60px",
          height: "60px",
          borderRadius: "4px",
          objectFit: "cover",
        }}
      />

      {/* 상품 정보 */}
      <div
        style={{
          flex: 1,
          display: "flex",
          flexDirection: "column",
          justifyContent: "center",
        }}
      >
        <h4
          style={{ margin: "0 0 4px 0", fontSize: "14px", fontWeight: "600" }}
        >
          {product.name}
        </h4>
        <p style={{ margin: "0", fontSize: "12px", color: "#666" }}>
          {product.description}
        </p>
        <span
          style={{
            margin: "4px 0 0 0",
            fontSize: "14px",
            fontWeight: "700",
            color: "#2c3e50",
          }}
        >
          ${product.price.toFixed(2)}
        </span>
      </div>
    </div>
  );
};

// 커스텀 헤더 생성
const CustomConversationHeader = ({
  props,
}: {
  props: ConversationHeaderProps;
}) => {
  const { components } = ConversationHeaderLayout.useContext();

  // Context 데이터 접근 (앱에서 전달한 contextData)
  const productId = (props as any).contextData?.productId;

  return (
    <div>
      {/* 상품 배너 */}
      {productId && <ProductBanner productId={productId} />}

      {/* 기본 헤더 */}
      <div style={{ padding: "12px 16px", borderBottom: "1px solid #e0e0e0" }}>
        <components.Title {...props} />
      </div>
    </div>
  );
};

// 헤더 레이아웃 적용
export const CustomConversation = () => {
  return (
    <ConversationHeaderLayout.Template>
      <ConversationHeaderLayout.Header component={CustomConversationHeader} />
    </ConversationHeaderLayout.Template>
  );
};
```

### 3단계: 고급 예시 - 상품 배너 클릭 처리

```tsx
const ProductBannerWithLink = ({ productId }: { productId: string }) => {
  const [product, setProduct] = useState<Product | null>(null);

  useEffect(() => {
    fetch(`https://api.example.com/products/${productId}`)
      .then((res) => res.json())
      .then(setProduct)
      .catch(console.error);
  }, [productId]);

  const handleBannerClick = () => {
    // 상품 상세 페이지로 이동
    window.location.href = `/products/${productId}`;
  };

  if (!product) return null;

  return (
    <div
      onClick={handleBannerClick}
      style={{
        padding: "12px 16px",
        borderBottom: "1px solid #e0e0e0",
        backgroundColor: "#fafafa",
        cursor: "pointer",
        display: "flex",
        gap: "12px",
        transition: "background-color 0.2s",
      }}
      onMouseEnter={(e) => (e.currentTarget.style.backgroundColor = "#f0f0f0")}
      onMouseLeave={(e) => (e.currentTarget.style.backgroundColor = "#fafafa")}
    >
      <img
        src={product.imageUrl}
        alt={product.name}
        style={{
          width: "60px",
          height: "60px",
          borderRadius: "4px",
          objectFit: "cover",
        }}
      />
      <div style={{ flex: 1 }}>
        <h4
          style={{ margin: "0 0 4px 0", fontSize: "14px", fontWeight: "600" }}
        >
          {product.name}
        </h4>
        <span style={{ fontSize: "14px", fontWeight: "700", color: "#2c3e50" }}>
          ${product.price.toFixed(2)}
        </span>
      </div>
    </div>
  );
};
```

---

## 구현 체크리스트

### 모든 플랫폼

- [ ] Context Object에 productId 포함
- [ ] 상품 정보 API 엔드포인트 준비
- [ ] 오류 처리 로직 구현 (네트워크 오류, 404 등)
- [ ] 로딩 상태 UI 구현
- [ ] 배너 클릭 동작 정의 (상품 상세페이지로 이동 등)

### iOS

- [ ] AIAgentStarterKit 또는 SendbirdAIAgentMessenger 모듈 커스텀화 설정
- [ ] 상품 배너를 위한 UIView 구현
- [ ] 이미지 로딩 및 캐싱 처리

### Android

- [ ] ConversationHeaderTheme 커스텀화
- [ ] Glide 또는 Picasso로 이미지 로딩 처리
- [ ] RecyclerView 어댑터 구현 (여러 상품의 경우)

### Web/React

- [ ] ConversationHeaderLayout 커스텀화
- [ ] useContext 또는 props로 Context 데이터 전달
- [ ] 반응형 레이아웃 구현
- [ ] 이미지 로딩 상태 관리

---

## 주의사항

### API 호출 최소화

- Context에서 받은 productId로만 API를 호출하도록 제한
- 불필요한 API 중복 호출 방지 (useEffect 의존성 배열 확인)

### 네트워크 오류 처리

- 네트워크가 실패할 경우 배너를 표시하지 않거나 기본 메시지 표시
- 사용자에게 명확한 오류 메시지 제공

### 성능 고려사항

- **iOS/Android**: 이미지 캐싱 및 메모리 관리 필수
- **Web**: 초기 로딩 속도 최적화 (이미지 최적화, lazy loading)

### 배너 높이 제한

- 배너가 채팅창을 너무 많이 차지하지 않도록 고정 높이 설정 권장
- 모바일에서는 70-100px, 웹에서는 80-120px 정도 권장

### 다중 상품 지원

- 사용자가 상품을 전환할 때마다 Context를 업데이트하면 배너도 자동으로 갱신됨
- 각 상품별로 별도의 대화를 시작해야 하는지 확인 필요

---

## 예상 결과

구현 후 다음과 같은 화면을 볼 수 있습니다:

```
┌─────────────────────────────────┐
│ Conversation Header              │
├─────────────────────────────────┤
│ [Product Image] Product Name     │  <- 상품 배너
│                    $99.99        │
├─────────────────────────────────┤
│ AI Agent: 안녕하세요! 도움이...   │
│ User: 이 상품의 크기가 어떻게...  │
│ AI Agent: 이 상품은...           │
│                                   │
│ [입력창]                          │
└─────────────────────────────────┘
```
