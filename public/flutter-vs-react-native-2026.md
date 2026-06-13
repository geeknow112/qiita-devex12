---
title: 【2026年版】Flutter vs React Native 技術比較ガイド
tags:
  - Dart
  - クロスプラットフォーム
  - reactnative
  - Flutter
  - モバイルアプリ開発
private: false
updated_at: '2026-06-13T12:16:14+09:00'
id: 8c882b485bfcc6debd06
organization_url_name: null
slide: false
ignorePublish: false
---

## はじめに

クロスプラットフォーム開発の2大巨頭、FlutterとReact Native。2026年現在、どちらを選ぶべきか？

この記事では、アーキテクチャ、パフォーマンス、開発体験の観点から技術的に比較します。

## 市場シェア（2026年）

| フレームワーク | シェア |
|---|---|
| Flutter | 46% |
| React Native | 35% |
| その他 | 19% |

## アーキテクチャ比較

### Flutter：自前レンダリング

```
App Code (Dart)
    ↓
Flutter Framework
    ↓
Impeller Engine (GPU直接描画)
    ↓
Platform
```

- **Impellerエンジン**で全ピクセルを自前描画
- ネイティブUIコンポーネントを使用しない
- 全プラットフォームで完全に同一のUI

### React Native：ネイティブブリッジ

```
App Code (JS/TS)
    ↓
Hermes Engine
    ↓
Native Bridge (JSI)
    ↓
Native UI Components
```

- **Hermesエンジン**でJavaScript実行
- ブリッジ経由でネイティブUIを操作
- プラットフォームごとのネイティブ感

## 言語比較

### Dart（Flutter）

```dart
// Null Safety + 型推論
class UserService {
  final ApiClient _client;
  
  UserService(this._client);
  
  Future<User?> fetchUser(String id) async {
    try {
      final response = await _client.get('/users/$id');
      return User.fromJson(response.data);
    } catch (e) {
      return null;
    }
  }
}

// Riverpodでの状態管理
@riverpod
Future<User?> user(UserRef ref, String id) async {
  final service = ref.watch(userServiceProvider);
  return service.fetchUser(id);
}
```

### TypeScript（React Native）

```typescript
// 型定義
interface User {
  id: string;
  name: string;
  email: string;
}

// Custom Hook
const useUser = (id: string) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchUser(id)
      .then(setUser)
      .finally(() => setLoading(false));
  }, [id]);
  
  return { user, loading };
};

// Component
const UserProfile: React.FC<{ id: string }> = ({ id }) => {
  const { user, loading } = useUser(id);
  
  if (loading) return <ActivityIndicator />;
  if (!user) return <Text>User not found</Text>;
  
  return <Text>{user.name}</Text>;
};
```

## パフォーマンス実測

| 項目 | Flutter | React Native |
|---|---|---|
| アニメーションFPS | 59-60 | 54-58 |
| 起動時間 | 高速 | やや遅い |
| メモリ使用量 | 中程度 | やや多い |
| 最小APKサイズ | 10-15 MB | 7-12 MB |

### アニメーション実装例

**Flutter**

```dart
class AnimatedCard extends StatefulWidget {
  @override
  _AnimatedCardState createState() => _AnimatedCardState();
}

class _AnimatedCardState extends State<AnimatedCard>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _scaleAnimation;
  
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      duration: Duration(milliseconds: 300),
      vsync: this,
    );
    _scaleAnimation = Tween<double>(begin: 1.0, end: 1.1)
        .animate(CurvedAnimation(
          parent: _controller,
          curve: Curves.easeInOut,
        ));
  }
  
  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTapDown: (_) => _controller.forward(),
      onTapUp: (_) => _controller.reverse(),
      child: ScaleTransition(
        scale: _scaleAnimation,
        child: Card(child: Text('Tap me')),
      ),
    );
  }
}
```

**React Native（Reanimated 3）**

```typescript
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withSpring,
} from 'react-native-reanimated';

const AnimatedCard: React.FC = () => {
  const scale = useSharedValue(1);
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
  }));
  
  const handlePressIn = () => {
    scale.value = withSpring(1.1);
  };
  
  const handlePressOut = () => {
    scale.value = withSpring(1);
  };
  
  return (
    <Pressable onPressIn={handlePressIn} onPressOut={handlePressOut}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>Tap me</Text>
      </Animated.View>
    </Pressable>
  );
};
```

## プロジェクト構成比較

### Flutter

```
my_app/
├── lib/
│   ├── main.dart
│   ├── app/
│   │   ├── app.dart
│   │   └── routes.dart
│   ├── features/
│   │   ├── auth/
│   │   │   ├── data/
│   │   │   ├── domain/
│   │   │   └── presentation/
│   │   └── home/
│   └── shared/
│       ├── widgets/
│       └── utils/
├── test/
├── pubspec.yaml
└── analysis_options.yaml
```

### React Native

```
my_app/
├── src/
│   ├── App.tsx
│   ├── navigation/
│   ├── features/
│   │   ├── auth/
│   │   │   ├── api/
│   │   │   ├── hooks/
│   │   │   ├── screens/
│   │   │   └── components/
│   │   └── home/
│   └── shared/
│       ├── components/
│       └── utils/
├── __tests__/
├── package.json
└── tsconfig.json
```

## 状態管理比較

### Flutter（Riverpod）

```dart
// providers.dart
@riverpod
class AuthNotifier extends _$AuthNotifier {
  @override
  AsyncValue<User?> build() => const AsyncValue.data(null);
  
  Future<void> signIn(String email, String password) async {
    state = const AsyncValue.loading();
    state = await AsyncValue.guard(() async {
      return ref.read(authServiceProvider).signIn(email, password);
    });
  }
  
  Future<void> signOut() async {
    await ref.read(authServiceProvider).signOut();
    state = const AsyncValue.data(null);
  }
}

// 使用側
class LoginScreen extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authNotifierProvider);
    
    return authState.when(
      data: (user) => user != null ? HomeScreen() : LoginForm(),
      loading: () => CircularProgressIndicator(),
      error: (e, _) => Text('Error: $e'),
    );
  }
}
```

### React Native（Zustand）

```typescript
// store.ts
interface AuthState {
  user: User | null;
  loading: boolean;
  signIn: (email: string, password: string) => Promise<void>;
  signOut: () => Promise<void>;
}

const useAuthStore = create<AuthState>((set) => ({
  user: null,
  loading: false,
  
  signIn: async (email, password) => {
    set({ loading: true });
    try {
      const user = await authService.signIn(email, password);
      set({ user, loading: false });
    } catch (e) {
      set({ loading: false });
      throw e;
    }
  },
  
  signOut: async () => {
    await authService.signOut();
    set({ user: null });
  },
}));

// 使用側
const LoginScreen: React.FC = () => {
  const { user, loading } = useAuthStore();
  
  if (loading) return <ActivityIndicator />;
  if (user) return <HomeScreen />;
  return <LoginForm />;
};
```

## ビルド・デプロイ

### Flutter

```bash
# 開発
flutter run

# ビルド
flutter build apk --release
flutter build ios --release
flutter build web

# テスト
flutter test
flutter test --coverage
```

### React Native

```bash
# 開発
npx react-native start
npx react-native run-android
npx react-native run-ios

# ビルド（EAS Build）
eas build --platform android
eas build --platform ios

# テスト
npm test
```

## CI/CD設定例

### Flutter（GitHub Actions）

```yaml
name: Flutter CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
      
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test --coverage
      - run: flutter build apk --release
```

### React Native（GitHub Actions）

```yaml
name: React Native CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage
      - run: eas build --platform android --non-interactive
```

## 選定基準まとめ

### Flutterを選ぶ

- UIの一貫性が最優先
- 高度なカスタムUI/アニメーション
- Web/Desktop展開も視野
- パフォーマンス重視

### React Nativeを選ぶ

- 既存React/JSチーム
- ネイティブルック＆フィール重視
- 既存ネイティブアプリへの組み込み
- アプリサイズ最小化

## 参考リンク

- [Flutter公式](https://flutter.dev/)
- [React Native公式](https://reactnative.dev/)
- [Riverpod](https://riverpod.dev/)
- [Zustand](https://zustand-demo.pmnd.rs/)
- [Reanimated](https://docs.swmansion.com/react-native-reanimated/)

