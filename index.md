[작성 완료된 가이드]

- [폰트 커스텀 가이드](./guide_font.md)
- [클라이언트 한글언어 강제 설정 가이드](./guide_language.md)
- [채팅창 상단에 상품 배너 다는 가이드](./guide_product_banner.md)
- [Markdown 이미지 및 링크 렌더링 타이밍 가이드](./guide_markdown_deferred_rendering.md)
- [Custom Message Template 사용 가이드 (상품 옵션 선택)](./guide_custom_message_template.md)
- [GA4 LiveMetric 통합 가이드](./guide_ga4_livemetric.md)

---

[SDK 업데이트 후 업데이트 할 가이드]

- 이전 대화 유지, 이전 상품 유지 가이드 및 새 대화 버튼 숨김처리
  - 제품 페이지 진입시 > 해당 제품에 맞는 대화가 나타나야 함 (대화가 있는 경우 이전 대화, 없으면 새 대화 생성)
  - 대화 목록에서, 새 대화 생성시 > 현재 보고 있는 제품으로 새 대화가 시작되어야 함
  - 대화 종료 시점에, 새 대화 생성시 > 현재 보고 있는 제품으로 새 대화가 시작되어야 함 (JS 는 커스텀으로 처리가 어려워, 일단 숨김처리 해놓음)
