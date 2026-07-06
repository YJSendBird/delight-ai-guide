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

### 제약사항

- 헤더 높이를 키우는 대신, 배너를 헤더 아래 별도 뷰로 얹는 구성도 가능합니다.
- 헤더의 기본 버튼(menu/close/handoff)은 `Header`의 `@LayoutSlot` 프로퍼티(`menuButton`, `closeButton`, `handoffButton`)로 노출되어 있어, `layoutLeftItems()` / `layoutRightItems()` 오버라이드로 배치를 조정할 수 있습니다.

---

## Android

### 개요

**Flow:**

1. 앱에서 사용자가 보고 있는 상품의 ID를 파악
2. Context Object(`context` 맵)를 통해 AI Agent SDK에 상품 ID 전달
3. `ConversationModule`을 확장해 헤더 아래에 배너 View 추가
4. 상품 정보를 API로 조회해 배너에 표시

```
┌─────────────────────────────────┐
│ Conversation Header             │
├─────────────────────────────────┤
│ [Product Image] Product Name    │  <- 상품 배너 (커스텀 View)
│                    ₩599,000     │
├─────────────────────────────────┤
│ AI Agent: 안녕하세요! ...        │
│ User: 이 상품의 크기가 ...       │
│                                 │
│ [입력창]                         │
└─────────────────────────────────┘
```

---

### 1단계: Context Object로 상품 ID 전달

화면을 열 때 `ConversationSettingsParams.context`에 상품 ID를 담아 전달합니다. 이 값은 대화의 context로 서버에 전달되어 AI 에이전트가 상품 맥락을 인지하는 데에도 사용됩니다.

```kotlin
import com.sendbird.sdk.aiagent.messenger.model.ConversationSettingsParams
import com.sendbird.sdk.aiagent.messenger.ui.activity.MessengerActivity

fun openConversationWithProduct(productId: String) {
    // 배너에서 사용할 수 있도록 앱 쪽에도 현재 상품 ID를 보관
    CurrentProduct.id = productId

    startActivity(
        MessengerActivity.newIntentForConversation(
            context = this,
            aiAgentId = "YOUR_AI_AGENT_ID",
            conversationSettingsParams = ConversationSettingsParams(
                context = mapOf("product_id" to productId),
            ),
        )
    )
}

// 배너가 참조할 앱 레벨 상태 (예시)
object CurrentProduct {
    var id: String? = null
}
```

> 상품별로 대화 자체를 분리하고 싶다면 [제품별 대화 연동 가이드](./guide_product_conversation_connection.md)를 함께 참고하세요.

---

### 2단계: ConversationModule 확장으로 배너 View 추가

대화 화면의 레이아웃은 `ConversationModule.onCreateContentView()`가 구성합니다. 이 클래스는 `open`이므로 상속해서 **헤더 바로 아래에 배너 View를 삽입**할 수 있습니다.

```kotlin
import android.content.Context
import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.widget.LinearLayout
import com.sendbird.sdk.aiagent.messenger.consts.MessengerThemeMode
import com.sendbird.sdk.aiagent.messenger.model.MessengerSettings
import com.sendbird.sdk.aiagent.messenger.model.theme.MessengerTheme
import com.sendbird.sdk.aiagent.messenger.modules.ConversationModule

class ProductBannerConversationModule(
    context: Context,
    currentTheme: MessengerTheme,
    currentThemeMode: MessengerThemeMode,
    settings: MessengerSettings?,
    args: Bundle,
) : ConversationModule(context, currentTheme, currentThemeMode, settings, args) {

    var bannerView: View? = null
        private set

    override fun onCreateContentView(context: Context, inflater: LayoutInflater, args: Bundle?): View {
        val root = super.onCreateContentView(context, inflater, args) as LinearLayout

        // 앱에서 만든 배너 레이아웃 inflate
        val banner = inflater.inflate(R.layout.view_product_banner, root, false)
        bannerView = banner

        // 헤더가 표시 중이면 헤더(0) 바로 아래(1), 아니면 최상단(0)에 삽입
        val insertIndex = if (params.isUsingHeader) 1 else 0
        root.addView(banner, insertIndex)

        bindProductBanner(banner)  // 3단계에서 구현
        return root
    }
}
```

만든 모듈을 **화면을 열기 전에** Provider로 등록합니다.

```kotlin
import com.sendbird.sdk.aiagent.messenger.providers.AIAgentModuleProviders
import com.sendbird.sdk.aiagent.messenger.providers.ConversationModuleProvider

AIAgentModuleProviders.conversation =
    ConversationModuleProvider { context, currentTheme, currentThemeMode, settings, args ->
        ProductBannerConversationModule(context, currentTheme, currentThemeMode, settings, args)
    }
```

배너 레이아웃 예시:

```xml
<!-- res/layout/view_product_banner.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/productBanner"
    android:layout_width="match_parent"
    android:layout_height="80dp"
    android:orientation="horizontal"
    android:gravity="center_vertical"
    android:padding="12dp"
    android:visibility="gone">

    <ImageView
        android:id="@+id/productImage"
        android:layout_width="56dp"
        android:layout_height="56dp"
        android:scaleType="centerCrop" />

    <LinearLayout
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:layout_marginStart="12dp"
        android:orientation="vertical">

        <TextView
            android:id="@+id/productName"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:maxLines="1"
            android:ellipsize="end"
            android:textStyle="bold" />

        <TextView
            android:id="@+id/productPrice"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />
    </LinearLayout>
</LinearLayout>
```

---

### 3단계: 상품 정보 조회 및 배너 바인딩

앱이 보관한 상품 ID(`CurrentProduct.id`)로 상품 정보를 조회해 배너에 표시합니다. 조회/이미지 로딩은 앱에서 사용 중인 HTTP 클라이언트(Retrofit 등)와 이미지 라이브러리(Glide/Coil)를 그대로 사용하면 됩니다.

```kotlin
// ProductBannerConversationModule의 메서드로 구현
private fun bindProductBanner(banner: View) {
    val productId = CurrentProduct.id ?: return

    // 예시: 앱의 API로 상품 정보 조회 (Retrofit/코루틴 등 앱 표준 방식 사용)
    fetchProduct(productId) { product ->
        banner.findViewById<TextView>(R.id.productName).text = product.name
        banner.findViewById<TextView>(R.id.productPrice).text = "₩%,d".format(product.price)
        Glide.with(banner).load(product.imageUrl).into(banner.findViewById(R.id.productImage))

        banner.setOnClickListener {
            // 상품 상세 페이지로 이동 등
        }
        banner.visibility = View.VISIBLE
    }
}
```

---

### 구현 체크리스트

- [ ] 화면 실행 시 `ConversationSettingsParams.context`에 `product_id` 포함
- [ ] `AIAgentModuleProviders.conversation` 등록은 init 직후, 화면 열기 전 1회
- [ ] 상품 정보 API 오류 처리 (실패 시 배너 미표시 권장)
- [ ] 배너 클릭 동작 정의 (상품 상세 페이지 이동 등)
- [ ] 배너 높이 고정 (70~100dp 권장 — 채팅 영역을 과도하게 차지하지 않도록)

---

### 주의사항

- **배너는 앱 소유 View입니다**: SDK는 배너의 데이터/생명주기를 관리하지 않습니다. 상품 조회 실패, 이미지 로딩, 클릭 처리는 앱에서 구현합니다.
- **Light/Dark 테마**: 배너 배경/텍스트 색상은 SDK 테마 모드(`AIAgentMessenger.currentThemeMode`)에 맞춰 앱에서 지정하세요.
- **상품 전환**: 다른 상품 페이지에서 다시 화면을 열면 새 `context`가 전달됩니다. 상품별로 대화를 분리하려면 [제품별 대화 연동 가이드](./guide_product_conversation_connection.md)의 검색/생성 흐름을 사용하세요.

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
