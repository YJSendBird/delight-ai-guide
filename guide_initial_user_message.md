# initialUserMessage

새 대화의 첫 유저 발화를 호스트가 지정합니다.

## iOS SDK 
1.21.0으로 버전 업데이트 후, AIAgentMessenger.presentConversation 메소드에 initialUserMessage 사용. 

### Version
```
1.21.0
```
 
### 쓰는 법

`presentConversation`의 parameter `channelURL`이 non-nil로 주어져야합니다.
필요하면, `AIAgentMessenger.createConversation(aiAgentId:)`로 채널을 생성해서 넘겨주세요.

```swift
AIAgentMessenger.createConversation(aiAgentId: agentId) { result in
    guard case .success(let channelURL) = result else { return }
    AIAgentMessenger.presentConversation(
        aiAgentId: agentId,
        channelURL: channelURL,
        initialUserMessage: "커튼 사이즈 재는 법이 궁금해요"
    )
}
```

`initialUserMessage`를 안 쓸 때는 기존대로 `presentConversation(channelURL: nil)`로 사용하시면 됩니다.

### 참고 

- **대화가 새로 만들어질 때만 적용됩니다.** SDK 는 `known_active_channel_url` 로 채널을 먼저 확정합니다. 그 채널에 대화가 이미 있으면 값은 버려집니다. 에러는 안 납니다.
- **single conversation 모드(`is_multiple_active_conversations_enabled == false`)에서는 항상 무시됩니다.** 서버가 늘 기존 대화를 돌려줍니다. 그 유저의 첫 대화를 만드는 순간에만 들어갑니다.
- **`channelURL: nil` 로 열면 보장되지 않습니다.** `channelURL` 을 넘기지 않으면 SDK 가 캐시된 active 채널 URL 을 `known_active_channel_url` 로 보냅니다. 그 채널에 대화가 있으면 무시됩니다.
- **`shouldUseCurrentActiveChannelURL` 은 바뀌지 않습니다.** `initialUserMessage` 유무가 이 플래그의 의미를 건드리지 않습니다.
- **버전:** 1.21.0

## Android SDK
1.19.0 버전으로 올린 후, MessengerActivity.newIntentForConversation 메소드에 새로 추가된 initialUserMessage 사용.

### SDK version
```
1.19.0
```

###
```kotlin
// Activity 메서드. aiAgentId 는 화면이 보유한 nullable 필드.
private fun openNewConversation(callbackUserMessage: String) {
  val aiAgentId = aiAgentId ?: return
  lifecycleScope.launch {
    runCatching {
      with(AIAgentMessenger) {
        authenticate(aiAgentId)                                   // 세션 establish (suspend)
        val url = awaitCreateConversation(                        // 클릭마다 새 대화 생성
          ConversationCreateParams(aiAgentId = aiAgentId))
        startActivity(MessengerActivity.newIntentForConversation(
          this@YourActivity, aiAgentId, url,
          initialUserMessage = callbackUserMessage))              // 서버가 첫 유저 메시지로 게시
      }
    }
  }
}
```
