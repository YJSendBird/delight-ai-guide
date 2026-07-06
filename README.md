# Delight AI Agent 통합 가이드

Sendbird Delight AI Agent SDK 연동 시 바로 참고할 수 있는 가이드 모음입니다.

---

## 가이드 목록

| 가이드 | 설명 |
| --- | --- |
| [폰트 커스텀 가이드](./guide_font.md) | SDK 채팅창에 커스텀 폰트를 적용하는 방법 |
| [한글 언어 강제 설정 가이드](./guide_language.md) | 디바이스 시스템 설정에 관계없이 한글로 강제 설정하는 방법 |
| [채팅창 상단 상품 배너 가이드](./guide_product_banner.md) | Context Object로 상품 정보를 전달해 배너를 표시하는 방법 |
| [Markdown 렌더링 타이밍 가이드](./guide_markdown_deferred_rendering.md) | 스트리밍 중 Markdown 이미지와 링크를 완성될 때까지 숨기는 방법 |
| [Custom Message Template 가이드](./guide_custom_message_template.md) | 상품 옵션 선택 화면을 커스텀 메시지 템플릿으로 구현하는 방법 |
| [GA4 LiveMetric 통합 가이드](./guide_ga4_livemetric.md) | LiveMetric 핸들러로 실시간 이벤트를 GA4에 전송하는 방법 |

---

## 제품별 대화 연동 가이드

| 가이드 | 설명 |
| --- | --- |
| [제품별 대화 연동 가이드](./guide_product_conversation_connection.md) | 제품 페이지 진입 시 해당 제품의 대화를 다시 열고, 없으면 새로 생성하는 방법 |

> `openProductConversation` 헬퍼를 만들어 대화 진입을 감싸고, `product_id`는 Context Object로 전달합니다.
> iOS 대화 화면은 full screen 기준으로 엽니다. Android와 Web 구현 가이드는 순차적으로 추가 예정입니다.
