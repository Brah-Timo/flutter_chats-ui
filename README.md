# flutter_chats_ui

**Production-grade Flutter chat UI framework** — plugin-based message types, command-pattern undo/redo, transport-agnostic, optimistic UI, offline outbox, threads, reactions, voice notes, polls, and more.

[![pub version](https://img.shields.io/badge/pub-v1.0.0-blue)](https://pub.dev/packages/flutter_chats_ui)
[![Flutter](https://img.shields.io/badge/Flutter-%E2%89%A53.19.0-blue)](https://flutter.dev)
[![Dart SDK](https://img.shields.io/badge/Dart-%E2%89%A53.3.0-blue)](https://dart.dev)
[![License: MIT](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

<img width="393" height="720" alt="flutter_chats_ui_showcase" src="https://github.com/user-attachments/assets/8b0f4f47-1d5a-4331-8bb4-38763fb67a6a" />




## Features

| Feature | Details |
|---|---|
| **Clean Architecture** | Domain → Engine → BLoC → UI — no layer leaks |
| **MVVM / BLoC** | `ChatBloc` + `ChatEvent` surface; `StreamBuilder`-friendly `ChatState` |
| **Command Pattern** | Undo/redo stacks (max 50), `SendMessage`, `EditMessage`, `Delete`, `React`, `Pin`, `MarkRead`, `Reply`, `Batch` |
| **12 Message Plugins** | Text, Image, Video, Audio, Voice Note, File, Location, Poll, Contact, Code, Sticker, System Event |
| **Transport Adapters** | In-Memory (tests/demos), WebSocket, REST, Firestore stub |
| **Offline Outbox** | Queued sends with automatic flush on reconnect |
| **Headless Engine** | `ChatEngine` has zero Flutter dependency — pure Dart, fully testable |
| **Threads & Replies** | First-class thread support with reply counts and `ThreadPanel` |
| **Reactions** | Per-message emoji reactions with toggle (add/remove) and undo |
| **Full-Text Search** | `SearchEngine` — case-insensitive, registry-aware fallback text |
| **Theming** | `ChatTheme` with light/dark presets, `copyWith`, and adaptive factory |
| **Accessibility** | `RepaintBoundary` on every bubble; `SafeArea`-aware composer |

---

## Getting Started

### Installation

Add to your `pubspec.yaml`:

```yaml
dependencies:
  flutter_chats_ui: ^1.0.0
```

Then run:

```bash
flutter pub get
```

### Minimal Example

```dart
import 'package:flutter_chats_ui/flutter_chats_ui.dart';
import 'package:flutter_chats_ui/plugins.dart';
import 'package:flutter_chats_ui/transports.dart';

final conversation = Conversation(
  id: 'conv-1',
  kind: ConversationKind.direct,
  members: [me, peer],
  messages: [],
);

// FlutterChatsUi is the ready-made StatefulWidget entry point.
FlutterChatsUi(
  initialConversation: conversation,
  currentUserId: me.id,
  transport: MemoryTransport(seed: conversation),
)
```

### Using ChatScreen directly

```dart
import 'package:flutter_chats_ui/flutter_chats_ui.dart';
import 'package:flutter_chats_ui/plugins.dart';
import 'package:flutter_chats_ui/transports.dart';

class MyChatPage extends StatefulWidget { /* ... */ }

class _MyChatPageState extends State<MyChatPage> {
  late final ChatBloc _bloc;

  @override
  void initState() {
    super.initState();
    _bloc = ChatBloc(
      initialConversation: widget.conversation,
      transport: MyWebSocketTransport(url: 'wss://chat.example.com'),
      currentUserId: widget.me.id,
    );
  }

  @override
  void dispose() {
    _bloc.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ChatScreen(
      bloc: _bloc,
      registry: MessageTypeRegistry.defaults(),
      theme: ChatTheme.adaptive(context),
    );
  }
}
```

<img width="395" height="742" alt="1" src="https://github.com/user-attachments/assets/d06ada07-dd8e-4e68-9c9c-26ffdf9672c4" />


<img width="395" height="742" alt="8" src="https://github.com/user-attachments/assets/afcd0cf5-c982-4686-8eeb-7985ec44dee0" />


<img width="395" height="742" alt="7" src="https://github.com/user-attachments/assets/449b1a47-8a04-4062-b91b-d89b160ce99d" />


<img width="395" height="742" alt="6" src="https://github.com/user-attachments/assets/082bbc74-2f24-4d59-ae55-0ef227e29ea6" />

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                          UI Layer                           │
│  ChatScreen  ChatAppBar  ChatMessageList  MessageComposer   │
│  ChatBubble  ThreadPanel  PinnedMessagesPanel  ...          │
└───────────────────────────┬─────────────────────────────────┘
                            │ ChatEvent / ChatState
┌───────────────────────────▼─────────────────────────────────┐
│                        BLoC Layer                           │
│                ChatBloc   ChatEvent   ChatState             │
└──────┬────────────────────┬──────────────────────┬──────────┘
       │ Commands           │ Engine               │ Transport
┌──────▼──────┐  ┌──────────▼─────────┐  ┌────────▼─────────┐
│CommandMgr   │  │   ChatEngine        │  │TransportAdapter  │
│ undo stack  │  │ MessageGrouper      │  │ Memory           │
│ redo stack  │  │ DateSeparator       │  │ WebSocket        │
│             │  │ SearchEngine        │  │ REST             │
│             │  │ UnreadMarker        │  │ Firestore stub   │
└─────────────┘  └────────────────────┘  └──────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                       Domain Layer                          │
│  ChatUser  Message  Conversation  Reaction  Attachment      │
│  Thread    MessagePayload  MessageTypePlugin  ChatCommand   │
└─────────────────────────────────────────────────────────────┘
```

---

## Message Plugins

Register built-in plugins or add custom ones:

```dart
// Use all 12 built-in plugins
final registry = MessageTypeRegistry.defaults();

// Or start empty and add selectively
final registry = MessageTypeRegistry.empty()
  ..register(TextMessagePlugin())
  ..register(ImageMessagePlugin())
  ..register(MyCustomPlugin());
```

### Custom Plugin

```dart
class MyStarredPayload extends MessagePayload {
  final String note;
  const MyStarredPayload(this.note);

  @override String get typeId => 'starred';
  @override Map<String, Object?> toJson() => {'note': note};
}

class MyStarredPlugin extends MessageTypePlugin {
  const MyStarredPlugin();

  @override String get id => 'starred';
  @override String get displayName => 'Starred Note';
  @override IconData get icon => Icons.star_rounded;

  @override
  Widget bubbleBuilder(BuildContext ctx, Message msg, MessageRenderContext rCtx) {
    final p = msg.payload as MyStarredPayload;
    return Text('⭐ ${p.note}');
  }

  @override
  Widget previewBuilder(BuildContext ctx, Message msg) =>
      const Text('Starred note');

  @override
  Map<String, Object?> serialize(MessagePayload p) => (p as MyStarredPayload).toJson();

  @override
  MessagePayload deserialize(Map<String, Object?> json) =>
      MyStarredPayload(json['note'] as String);

  @override
  String fallbackText(Message msg) => '⭐ ${(msg.payload as MyStarredPayload).note}';
}
```

---

## Transport Adapters

### WebSocket

```dart
final transport = WebSocketTransport(
  url: 'wss://chat.example.com/ws',
  conversationId: 'conv-1',
  currentUserId: me.id,
);
```

### REST (polling)

```dart
final transport = RestTransport(
  baseUrl: 'https://api.example.com',
  conversationId: 'conv-1',
  pollInterval: const Duration(seconds: 5),
);
```

### Firestore (stub)

Wire the `FirestoreTransport` stub to `cloud_firestore` in your host app.  
The file is not exported by default; copy and adapt `lib/src/transport/adapters/firestore_transport.dart`.

### Custom Transport

```dart
class MyTransport implements TransportAdapter {
  @override String get id => 'my_transport';

  @override Future<void> connect() async { /* ... */ }
  @override Future<void> disconnect() async { /* ... */ }
  @override Stream<TransportEvent> get events => _controller.stream;

  @override Future<DeliveryStatus> sendMessage(Message m) async {
    // call your API
    return DeliveryStatus.ok(serverAssignedId);
  }
  // implement editMessage, deleteMessage, fetchHistory, etc.
}
```

---

## Theming

```dart
// Built-in presets
ChatTheme.light   // default
ChatTheme.dark    // dark mode
ChatTheme.adaptive(context)  // follows system brightness

// Custom theme
const myTheme = ChatTheme(
  ownBubbleColor: Color(0xFF6200EE),
  otherBubbleColor: Color(0xFFF3E5F5),
  backgroundColor: Color(0xFFFAFAFA),
  showAvatars: true,
  showSenderNames: false,
);

// Incremental overrides
final myTheme = ChatTheme.light.copyWith(
  ownBubbleColor: Colors.deepPurple,
  maxBubbleWidthFactor: 0.65,
);
```

---

## Command Pattern (Undo / Redo)

```dart
// Dispatch via BLoC (recommended)
bloc.add(const UndoEvent());
bloc.add(const RedoEvent());

// Check availability
if (bloc.canUndo) { /* show undo button */ }
if (bloc.canRedo) { /* show redo button */ }

// Label for undo toast
final label = bloc.undoLabel; // e.g. "Undo send"

// Direct engine access
engine.undo();
engine.redo();
```

---

## Offline Mode

```dart
final bloc = ChatBloc(
  initialConversation: conv,
  transport: transport,
  currentUserId: me.id,
  config: const ChatConfig(offlineMode: true),
);
// Failed sends are enqueued in the Outbox and flushed on reconnect.
```

---

## File Structure

```
lib/
├── flutter_chats_ui.dart        ← main public barrel
├── plugins.dart                 ← 12 built-in plugins barrel
├── transports.dart              ← transport adapters barrel
└── src/
    ├── bloc/
    │   ├── chat_bloc.dart
    │   └── chat_event.dart
    ├── cache/
    │   ├── cache_adapter.dart
    │   ├── memory_cache.dart
    │   └── outbox.dart
    ├── core/
    │   ├── attachment.dart
    │   ├── chat_config.dart
    │   ├── chat_exception.dart
    │   ├── chat_state.dart
    │   ├── chat_user.dart
    │   ├── conversation.dart
    │   ├── message.dart
    │   ├── reaction.dart
    │   ├── thread.dart
    │   └── history/
    │       ├── command.dart
    │       ├── command_manager.dart
    │       └── commands/
    │           ├── batch_command.dart
    │           ├── delete_message_command.dart
    │           ├── edit_message_command.dart
    │           ├── mark_read_command.dart
    │           ├── pin_command.dart
    │           ├── react_command.dart
    │           ├── reply_command.dart
    │           └── send_message_command.dart
    ├── engine/
    │   ├── chat_engine.dart
    │   ├── date_separator_inserter.dart
    │   ├── message_grouper.dart
    │   ├── search_engine.dart
    │   └── unread_marker.dart
    ├── integration/
    │   ├── debouncer.dart
    │   ├── encryption_adapter.dart
    │   ├── i18n_adapter.dart
    │   ├── media_adapter.dart
    │   └── notification_bridge.dart
    ├── messages/
    │   ├── message_type_plugin.dart
    │   ├── message_type_registry.dart
    │   └── plugins/
    │       ├── audio_message_plugin.dart
    │       ├── code_plugin.dart
    │       ├── contact_plugin.dart
    │       ├── file_message_plugin.dart
    │       ├── image_message_plugin.dart
    │       ├── location_plugin.dart
    │       ├── poll_plugin.dart
    │       ├── sticker_plugin.dart
    │       ├── system_event_plugin.dart
    │       ├── text_message_plugin.dart
    │       ├── video_message_plugin.dart
    │       └── voice_note_plugin.dart
    ├── transport/
    │   ├── delivery_status.dart
    │   ├── transport_adapter.dart
    │   ├── transport_event.dart
    │   ├── transport_registry.dart
    │   └── adapters/
    │       ├── firestore_transport.dart
    │       ├── memory_transport.dart
    │       ├── rest_transport.dart
    │       └── websocket_transport.dart
    └── ui/
        ├── flutter_chats_ui.dart
        ├── theming/
        │   ├── chat_colors.dart
        │   ├── chat_theme.dart
        │   └── chat_typography.dart
        └── widgets/
            ├── chat_app_bar.dart
            ├── chat_bubble.dart
            ├── chat_message_list.dart
            ├── chat_screen.dart
            ├── emoji_picker_widget.dart
            ├── error_banner_widget.dart
            ├── long_press_actions_sheet.dart
            ├── mention_overlay.dart
            ├── message_composer.dart
            ├── pinned_messages_panel.dart
            ├── presence_indicator.dart
            ├── scroll_to_bottom_fab.dart
            ├── search_bar_widget.dart
            ├── swipe_to_reply.dart
            ├── thread_panel.dart
            ├── typing_dots.dart
            ├── unread_chip.dart
            └── waveform_widget.dart
test/
├── chat_engine_test.dart
├── command_manager_test.dart
├── message_grouper_test.dart
├── outbox_test.dart
├── search_engine_test.dart
├── transport_adapter_test.dart
├── integration/
│   └── send_receive_cycle_test.dart
└── widget/
    ├── chat_bubble_test.dart
    └── chat_screen_test.dart
```

---

## Requirements

| Dependency | Version |
|---|---|
| Dart SDK | `>=3.3.0 <4.0.0` |
| Flutter | `>=3.19.0` |
| `equatable` | `^2.0.5` |
| `uuid` | `^4.4.0` |
| `collection` | `^1.18.0` |
| `meta` | `^1.12.0` |

---

## Contributing

1. Fork the repository
2. Create your feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -m 'feat: add my feature'`
4. Push: `git push origin feature/my-feature`
5. Open a Pull Request

Please follow the existing code style (Clean Architecture, immutable domain models, `const` everywhere possible).

---

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

© 2026 timsoft-org
