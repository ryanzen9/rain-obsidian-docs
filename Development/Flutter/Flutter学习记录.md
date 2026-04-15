---
Title: Flutter学习记录
Draft: true
tags:
  - web
  - flutter
  - 前端
Author: Ruby Ceng
---

## 相关文档

> GSY 学习项目：`https://guoshuyu.cn/home/wx/Flutter-0.html`

> 文档源码：`https://github.com/rubyceng/github-client-flutter.git`

## 起步

### 安装 FVM 与 Flutter SDK

```bash
brew tap leoafarias/fvm
brew install fvm
```

相关命令：

```shell
# 安装sdk
fvm install 3.16.9

# 切换版本
fvm use 2.2.0
fvm use 2.2.0 --global

# 列出版本
fvm list

# 删除版本
fvm remove 2.2.0

# 环境配置版本
fvm use 2.2.0 --flavor dev
```

### 安装 Xcode 和相关配置

Appstore 安装 Xcode 后执行命令：

```bash
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch
```

初始化 Flutter 版本配置：

```bash
~/Work/Demo/Flutter  mkdir flutter_app                                                                                                                                                                                   ok | at 11:58:10

~/Work/Demo/Flutter  cd flutter_app                                                                                                                                                                                      ok | at 11:58:18

~/Work/Demo/Flutter/flutter_app  fvm use stable



~/Work/Demo/Flutter/flutter_app  ls -a                                                                                                                                                                  ok | stable flutter | at 12:00:40
.          ..         .fvm       .fvmrc     .gitignore

 ~/Work/Demo/Flutter/flutter_app  cat .fvmrc                                                                                                                                                             ok | stable flutter | at 12:00:41
{
  "flutter": "stable"
}%

 ~/Work/Demo/Flutter/flutter_app  cat .gitignore                                                                                                                                                         ok | stable flutter | at 12:00:48

# FVM Version Cache
.fvm/%

 ~/Work/Demo/Flutter/flutter_app  cd .fvm                                                                                                                                                                ok | stable flutter | at 12:00:57

 ~/Work/Demo/Flutter/flutter_app/.fvm  ls                                                                                                                                                                ok | stable flutter | at 12:01:02
flutter_sdk     fvm_config.json release         version         versions
```

初始化 flutter 项目：

```bash
# 初始化项目
fvm flutter create .

# 运行macos
fvm flutter run -d macos

# 运行ios
fvm flutter run -d ios

# 运行chrome
fvm flutter run -d chrome
```

### Xcode IDE 相关环境配置

配置 Simulator Runtimes 模拟器运行时：

> 模拟器是在您的 Mac 上运行的虚拟 iPhone 或 iPad，用来测试您的 iOS 应用。Flutter 需要知道您安装了哪些版本的模拟器才能在上面运行 App。

安装 Cocoapods apple 原生相关的 Sdk 库：

> **这是什么？** **CocoaPods** 是一个针对 macOS 和 iOS 项目的**依赖包管理器**，非常重要。许多 Flutter 插件（比如相机、地图、分享等）都需要依赖原生的库，而 CocoaPods 就是用来自动管理这些原生库的工具。

```bash
# 使用Homebrew安装cocoapods
brew install cocoapods
```

检查相关的环境配置：

```bash
fvm flutter doctor
```

### VSCode 插件

1. Flutter
2. Dart
3. Json to Dart Model
4. Awesome Flutter Snippets

### 创建 Flutter 示例项目

使用 CLI 进行快速创建：

```bash
fvm flutter create flutter_app
```

项目文件结构：

```
flutter_app/
├── lib/                    # 主要Dart代码目录
│   └── main.dart          # 应用入口文件
├── android/               # Android平台特定代码和配置
├── ios/                   # iOS平台特定代码和配置
├── web/                   # Web平台特定代码和配置
├── windows/               # Windows平台特定代码和配置
├── linux/                 # Linux平台特定代码和配置
├── macos/                 # macOS平台特定代码和配置
├── test/                  # 单元测试和集成测试
├── pubspec.yaml           # 项目配置和依赖管理
├── pubspec.lock          # 依赖版本锁定文件
└── analysis_options.yaml # 代码分析规则配置
```

```dart
// main.dart application entry point
import 'package:flutter/material.dart';

/*
创建一个简单的计算器应用

1. 搭建界面框架
Scaffold 组件：用于创建应用的框架，包含 AppBar 和 Body。
AppBar 组件：包含应用的标题。

Body 组件：包含显示屏和按钮区。
显示屏 (Display)：在顶部，用来显示输入的数字和计算结果。
按钮区 (Keyboard)：在下方，包含数字键和操作符键。

2. 实现显示屏
维护状态 out_put 来显示计算结果

3. 实现按钮区
Row 组件：水平布局组件
Column 组件：垂直布局组件
Expanded 组件：用于将子组件扩展到剩余空间
OutlinedButton 组件：用于创建按钮
判断按钮来处理逻辑

4. 实现计算逻辑
实现加减乘除运算 onPressed 方法
使用 setState 方法进行实时的更新
*/

void main() {
  runApp(const CalculatorApp());
}

class CalculatorApp extends StatelessWidget {
  const CalculatorApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Calculator',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const CalculatorHomePage(),
    );
  }
}

class CalculatorHomePage extends StatefulWidget {
  const CalculatorHomePage({super.key});

  @override
  State<CalculatorHomePage> createState() => _CalculatorHomePageState();
}

class _CalculatorHomePageState extends State<CalculatorHomePage> {
  String _output = "0";
  String output = "0";
  double num1 = 0.0;
  double num2 = 0.0;
  String operand = "";

  // 计算逻辑
  void buttonPressed(String buttonText) {
    if (buttonText == "CLEAR") {
      output = "0";
      num1 = 0.0;
      num2 = 0.0;
      operand = "";
    } else if (buttonText == "+" ||
        buttonText == "-" ||
        buttonText == "*" ||
        buttonText == "/") {
      num1 = double.parse(output);
      operand = buttonText;
      output = "0";
    } else if (buttonText == ".") {
      if (!output.contains(".")) {
        output = output + buttonText;
      }
    } else if (buttonText == "=") {
      num2 = double.parse(output);

      if (operand == "+") {
        output = (num1 + num2).toString();
      }
      if (operand == "-") {
        output = (num1 - num2).toString();
      }
      if (operand == "*") {
        output = (num1 * num2).toString();
      }
      if (operand == "/") {
        output = (num1 / num2).toString();
      }

      num1 = 0.0;
      num2 = 0.0;
      operand = "";
    } else {
      if (output == "0") {
        output = buttonText;
      } else {
        output = output + buttonText;
      }
    }

    // setState方法用于更新UI
    // 每当我们的状态变量（如 _output）发生改变，并且我们希望界面也随之更新时，就必须在 setState() 的回调函数中去改变它。
    setState(() {
      // 清理一下小数点
      _output = double.parse(output).toStringAsFixed(2);
      // 如果小数点后是 .00，就去掉
      if (_output.endsWith('.00')) {
        _output = _output.substring(0, _output.length - 3);
      }
    });
  }

  // 构建按钮
  Widget buildButton(String buttonText) {
    return Expanded(
      child: OutlinedButton(
        style: OutlinedButton.styleFrom(padding: const EdgeInsets.all(24.0)),
        child: Text(
          buttonText,
          style: const TextStyle(fontSize: 20.0, fontWeight: FontWeight.bold),
        ),
        onPressed: () => buttonPressed(buttonText),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('简单计算器')),
      // Column 组件：垂直布局组件
      body: Column(
        children: <Widget>[
          // 显示屏区域
          Container(
            padding: const EdgeInsets.all(16.0),
            alignment: Alignment.bottomRight,
            child: Text(_output),
          ),

          // 分割线
          const Expanded(child: Divider()),

          // 按钮区域
          Column(
            children: <Widget>[
              Row(
                children: <Widget>[
                  buildButton('7'),
                  buildButton('8'),
                  buildButton('9'),
                  buildButton('+'),
                ],
              ),
              Row(
                children: <Widget>[
                  buildButton('4'),
                  buildButton('5'),
                  buildButton('6'),
                  buildButton('-'),
                ],
              ),
              Row(
                children: <Widget>[
                  buildButton('1'),
                  buildButton('2'),
                  buildButton('3'),
                  buildButton('*'),
                ],
              ),
              Row(
                children: <Widget>[
                  buildButton('0'),
                  buildButton('.'),
                  buildButton('='),
                  buildButton('/'),
                ],
              ),
            ],
          ),
        ],
      ),
    );
  }
}
```

## Flutter 相关

> Flutter 中文文档：
> `https://docs.flutter.cn/get-started/fundamentals/widgets`

### 页面导航的实现（TabBar 与 PageView）

本章节深入探讨了 Flutter 中两种最核心的导航模式：顶部标签页（Top Tabs）和底部导航栏（Bottom Navigation）。我们不仅学习了它们的实现方式，还解决了在桌面端（macOS）遇到的手势冲突和适配问题。

#### 两种主流导航模式

在 Flutter 中，构建标签式导航主要依赖两套核心组件的组合。

| 特性         | **顶部标签页 (Top Tabs)**                                             | **底部导航栏 (Bottom Nav)**                                 |
| :----------- | :-------------------------------------------------------------------- | :---------------------------------------------------------- |
| **核心组件** | `TabBar` + `TabBarView`                                               | `BottomNavigationBar` + `PageView`                          |
| **控制器**   | `TabController` (通常由 `DefaultTabController` 自动管理)              | `PageController` (需要手动创建和管理)                       |
| **同步方式** | **自动同步** (只要 `TabBar` 和 `TabBarView` 共享一个 `TabController`) | **手动同步** (通过 `onTap` 和 `onPageChanged` 回调互相驱动) |
| **滑动性**   | `TabBarView` 天生支持滑动                                             | `PageView` 天生支持滑动 (前提是没有手势冲突)                |

---

#### 模式一：顶部标签页实现

顶部导航是信息分类展示的经典模式，常见于新闻、商品分类等应用场景。

**实现结构**: `DefaultTabController` -> `Scaffold` -> `AppBar` 的 `bottom` 属性放置 `TabBar` -> `body` 放置 `TabBarView`。

这种模式的优点是实现简单，Flutter 提供了 `DefaultTabController` 来自动完成 `TabBar` 和 `TabBarView` 之间的状态同步。

##### 代码示例

```flutter
import 'package:flutter/material.dart';

/// 顶部结构实现： scfflod 结构 appbar：tabbar + body： tabbarview
/// 使用 TabController 自动控制
class TopTabbarDemo extends StatelessWidget {
  const TopTabbarDemo({super.key});

  @override
  Widget build(BuildContext context) {
    // 使用 DefaultTabController 包裹，它会自动处理 TabController 的创建和销毁
    return DefaultTabController(
      length: 3, // 必须提供标签页的总数
      child: Scaffold(
        appBar: AppBar(
          title: Text('Top TabBar Demo'),
          bottom: TabBar( // TabBar 会自动寻找并使用 DefaultTabController
            tabs: [
              Tab(text: 'Tab 1', icon: Icon(Icons.access_alarm)),
              Tab(text: 'Tab 2', icon: Icon(Icons.android)),
              Tab(text: 'Tab 3', icon: Icon(Icons.ac_unit)),
            ],
          ),
        ),
        body: TabBarView( // TabBarView 也会自动同步
          children: [
            Container(color: Colors.blue),
            Container(color: Colors.red),
            Container(color: Colors.green),
          ],
        ),
      ),
    );
  }
}
```

---

#### 模式二：底部导航栏实现

底部导航是现代 App 的主流框架，用于承载应用的一级功能模块。

**实现结构**: `StatefulWidget` -> `Scaffold` -> `bottomNavigationBar` 属性放置 `BottomNavigationBar` -> `body` 放置 `PageView`。

这种模式需要**手动管理状态**：

1.  使用 `StatefulWidget` 来持有当前选中的索引 `_currentIndex`。
2.  创建并管理一个 `PageController` 来控制 `PageView` 的页面切换。
3.  通过 `onTap` 和 `onPageChanged` 回调，建立 `BottomNavigationBar` 和 `PageView` 之间的双向同步。

##### 代码示例

```flutter
import 'package:flutter/material.dart';

/// 底部结构实现： scfflod 结构 pageview + bottomNavigationBar
/// 使用 PageController 手动控制
class BottomTabbarDemo extends StatefulWidget {
  const BottomTabbarDemo({super.key});

  @override
  State<BottomTabbarDemo> createState() => _BottomTabbarDemoState();
}

class _BottomTabbarDemoState extends State<BottomTabbarDemo> {
  int _currentIndex = 0;
  // 对 PageView进行更新需要使用 PageController
  final PageController _pageController = PageController();

  @override
  void dispose() {
    _pageController.dispose(); // 必须在 dispose 方法中释放控制器
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      bottomNavigationBar: BottomNavigationBar(
        items: const [
          BottomNavigationBarItem(
            icon: Icon(Icons.access_alarm),
            label: 'Tab 1',
          ),
          BottomNavigationBarItem(icon: Icon(Icons.android), label: 'Tab 2'),
          BottomNavigationBarItem(icon: Icon(Icons.ac_unit), label: 'Tab 3'),
        ],
        currentIndex: _currentIndex, // UI 显示与状态同步
        onTap: (index) {
          // 点击底部 Tab，驱动 PageView 切换
          setState(() {
            _currentIndex = index;
            _pageController.jumpToPage(index);
          });
        },
      ),
      body: PageView(
        controller: _pageController,
        onPageChanged: (index) {
          // 滑动 PageView，驱动底部 Tab 高亮
          setState(() {
            _currentIndex = index;
          });
        },
        children: [
          Container(color: Colors.blue),
          Container(color: Colors.red),
          Container(color: Colors.green),
        ],
      ),
    );
  }
}
```

---

#### 专题：解决桌面端手势冲突与适配

在将应用扩展到桌面端（如 macOS）时，会遇到 `PageView` 默认无法通过鼠标拖动来滑动的问题。

**根源**：Flutter 为了遵循平台原生体验，在桌面端的默认 `ScrollBehavior` (滚动行为) 中，并未将鼠标左键拖动视为"滚动"手势。

**解决方案**：自定义 `ScrollBehavior`，明确告诉 Flutter 允许鼠标进行拖动滚动。

##### 1. 创建自定义滚动行为类

```dart
import 'package:flutter/gestures.dart';
import 'package:flutter/material.dart';

// 自定义滚动行为，让应用在所有设备上都能通过拖动来滚动
class MyCustomScrollBehavior extends MaterialScrollBehavior {
  // 重写 dragDevices 属性，在默认设备基础上增加鼠标
  @override
  Set<PointerDeviceKind> get dragDevices => {
        PointerDeviceKind.touch,
        PointerDeviceKind.mouse,
        PointerDeviceKind.stylus,
        PointerDeviceKind.trackpad,
      };
}
```

##### 2. 应用滚动行为

###### 局部应用

用 `ScrollConfiguration` 组件包裹需要修改滚动行为的 Widget。

```dart
// ...
body: ScrollConfiguration(
  behavior: MyCustomScrollBehavior(), // 应用自定义行为
  child: PageView(
    // ...
  ),
),
//...
```

###### 全局应用（推荐）

在 `MaterialApp` 的根级别进行配置，使整个应用都支持鼠标拖动。

```dart
void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      // 在这里进行全局配置
      scrollBehavior: MyCustomScrollBehavior(),
      title: 'Flutter Demo',
      home: BottomTabbarDemo(),
    );
  }
}
```

### 状态管理

#### 基础状态管理 - setState

`setState`是 Flutter 最基础的状态管理方式，适用于组件内部的状态更新。它的特点是状态被完全限制在当前 Widget 内部，不会影响其他组件。

##### 概念特点

- **作用域**：仅限于当前 StatefulWidget 内部
- **适用场景**：独立管理自身状态，如计数器、开关按钮等
- **核心机制**：调用`setState()`触发 Widget 重建

##### 代码示例

```dart
class SelfStateCounter extends StatefulWidget {
  const SelfStateCounter({super.key});

  @override
  State<SelfStateCounter> createState() => _SelfStateCounterState();
}

class _SelfStateCounterState extends State<SelfStateCounter> {
  // 1. 状态作为State类的属性
  int _counter = 0;

  void _increment() {
    // 2. 调用setState()更新状态并触发UI重建
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.center,
      children: [
        Text(
          'Local State Counter: $_counter',
          style: const TextStyle(fontSize: 16),
        ),
        const SizedBox(width: 16),
        ElevatedButton(onPressed: _increment, child: const Icon(Icons.add)),
      ],
    );
  }
}
```

#### 状态提升 - 父子组件状态共享

当多个子组件需要共享同一个状态时，我们需要将状态"提升"到它们的共同父组件中进行管理。这种模式通过回调函数实现子组件向父组件的通信。

##### 概念特点

- **核心思想**：将共享状态提升到最近的共同父组件
- **通信方式**：
  - 父 → 子：通过构造函数传递状态数据
  - 子 → 父：通过回调函数触发父组件的`setState`
- **适用场景**：父子组件或兄弟组件间的状态共享

##### 代码示例

```dart
// 父组件，持有并管理状态
class LiftingStateUpExample extends StatefulWidget {
  const LiftingStateUpExample({super.key});
  @override
  State<LiftingStateUpExample> createState() => _LiftingStateUpExampleState();
}

class _LiftingStateUpExampleState extends State<LiftingStateUpExample> {
  bool _isActive = false;

  // 回调函数，用于接收子组件的事件
  void _handleSwitchChanged(bool newValue) {
    setState(() {
      _isActive = newValue;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        // 状态向下传递给子组件A
        StatusDisplay(isActive: _isActive),
        const SizedBox(height: 10),
        // 状态和回调函数都向下传递给子组件B
        ControlSwitch(
          isActive: _isActive,
          onChanged: _handleSwitchChanged, // 关键：传递回调
        ),
      ],
    );
  }
}

// 子组件A：只负责显示状态
class StatusDisplay extends StatelessWidget {
  final bool isActive;
  const StatusDisplay({super.key, required this.isActive});

  @override
  Widget build(BuildContext context) {
    return Text(
      isActive ? 'Active' : 'Inactive',
      style: TextStyle(
        color: isActive ? Colors.green : Colors.red,
        fontSize: 18,
      ),
    );
  }
}

// 子组件B：负责触发状态变更
class ControlSwitch extends StatelessWidget {
  final bool isActive;
  final ValueChanged<bool> onChanged; // 接收回调函数

  const ControlSwitch({
    super.key,
    required this.isActive,
    required this.onChanged,
  });

  @override
  Widget build(BuildContext context) {
    return Switch(
      value: isActive,
      onChanged: onChanged, // 关键：调用父组件传来的回调
    );
  }
}
```

#### 状态共享解决方案

状态共享遵循"单一数据源"(Single Source of Truth)的原则，即多个组件依赖并展示同一份数据。当数据改变时，所有依赖它的组件都会自动更新。

##### InheritedWidget - 官方解决方案

InheritedWidget 是 Flutter 官方提供的状态共享机制，通过 Widget 树高效地传递数据。

**特点：**

- **高效传递**：避免逐层传递数据的繁琐
- **自动更新**：依赖的子组件会自动重建
- **适用场景**：跨越多层的组件状态共享

```dart
// 1. 创建继承自InheritedWidget的类
class SharedColorWidget extends InheritedWidget {
  final Color color;
  final Function(Color) onColorChange;

  const SharedColorWidget({
    super.key,
    required this.color,
    required this.onColorChange,
    required super.child,
  });

  // 静态of方法，方便后代获取实例
  static SharedColorWidget of(BuildContext context) {
    final SharedColorWidget? result = context
        .dependOnInheritedWidgetOfExactType<SharedColorWidget>();
    assert(result != null, 'No SharedColorWidget found in context');
    return result!;
  }

  @override
  bool updateShouldNotify(SharedColorWidget oldWidget) {
    // 当color变化时，通知依赖它的后代重建
    return color != oldWidget.color;
  }
}

// 2. 在顶层管理状态并提供数据
class InheritedWidgetExample extends StatefulWidget {
  const InheritedWidgetExample({super.key});
  @override
  State<InheritedWidgetExample> createState() => _InheritedWidgetExampleState();
}

class _InheritedWidgetExampleState extends State<InheritedWidgetExample> {
  Color _color = Colors.blue;

  void _changeColor(Color newColor) {
    setState(() {
      _color = newColor;
    });
  }

  @override
  Widget build(BuildContext context) {
    return SharedColorWidget(
      color: _color,
      onColorChange: _changeColor,
      child: Column(
        children: [
          const ColorDisplay(),
          const SizedBox(height: 10),
          const ColorChangeButton(),
        ],
      ),
    );
  }
}

// 3. 后代组件获取并使用共享状态
class ColorDisplay extends StatelessWidget {
  const ColorDisplay({super.key});
  @override
  Widget build(BuildContext context) {
    final sharedColor = SharedColorWidget.of(context).color;
    return Container(width: 50, height: 50, color: sharedColor);
  }
}
```

##### Provider + ChangeNotifier - 社区推荐方案

Provider 配合 ChangeNotifier 提供了更加强大和灵活的状态管理解决方案。

**ChangeNotifier 核心思想：**
可以把它比作一个"中央公告板"：

- **公告板**(ChangeNotifier 实例)：负责记录和保管状态信息
- **发布者**(调用 notifyListeners()的方法)：在公告板上更新信息
- **订阅者**(context.watch 或 Consumer)：关心公告板的组件，信息更新时会收到通知并重建 UI

**实现步骤：**

1. 创建状态模型：继承 ChangeNotifier，状态变更后调用`notifyListeners()`
2. 提供状态：使用`ChangeNotifierProvider`在 Widget 树上层提供状态实例
3. 消费状态：使用`context.watch<T>()`或`Consumer<T>`监听状态变化

```dart
// 1. 状态模型：购物车
class CartModel extends ChangeNotifier {
  final List<String> _items = [];
  List<String> get items => _items;

  void add(String item) {
    _items.add(item);
    notifyListeners(); // 状态改变，通知所有监听者！
  }
}

// 2. 在顶层提供状态
class ChangeNotifierExample extends StatelessWidget {
  const ChangeNotifierExample({super.key});
  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (context) => CartModel(),
      child: const Column(
        children: [
          CartBadge(), // 显示数量
          SizedBox(height: 20),
          ProductList(), // 添加商品
        ],
      ),
    );
  }
}

// 3. 消费状态的组件A
class CartBadge extends StatelessWidget {
  const CartBadge({super.key});
  @override
  Widget build(BuildContext context) {
    // 使用watch监听CartModel的变化
    final cart = context.watch<CartModel>();
    return Chip(
      label: Text('购物车: ${cart.items.length} 件'),
      backgroundColor: Colors.orange,
    );
  }
}

// 4. 改变状态的组件B
class ProductList extends StatelessWidget {
  const ProductList({super.key});
  @override
  Widget build(BuildContext context) {
    // 使用read获取CartModel，只调用方法不监听变化
    final cart = context.read<CartModel>();
    final cartVM = context.watch<CartModel>();
    return ElevatedButton(
      child: Text('添加商品 "手机" ${cartVM.items.length}'),
      onPressed: () => cart.add('手机'),
    );
  }
}
```

#### 组件间通信

组件间通信关注的是"消息传递"，当一个组件发生某个事件时，需要通知另一个组件采取相应动作。这是一种命令式的通信方式。

##### Notification - 事件冒泡机制

Notification 提供了事件冒泡式的通信机制，子组件可以向上层祖先组件发送事件通知。

**核心特点：**

- **单向传递**：从子组件向祖先组件冒泡
- **事件驱动**：基于特定事件的触发机制
- **适用场景**：深层嵌套组件的事件通知，如滚动事件、自定义动作触发等

**实现步骤：**

1. 定义事件：创建继承自 Notification 的类，携带事件数据
2. 监听事件：使用`NotificationListener<T>`包裹祖先组件，提供回调处理
3. 派发事件：在子组件中创建通知实例并调用`dispatch(context)`

```dart
// 1. 定义通知类
class ColorChangeNotification extends Notification {
  final Color color;
  ColorChangeNotification(this.color);
}

// 2. 在顶层监听通知
class NotificationExample extends StatefulWidget {
  const NotificationExample({super.key});
  @override
  State<NotificationExample> createState() => _NotificationExampleState();
}

class _NotificationExampleState extends State<NotificationExample> {
  Color _containerColor = Colors.grey.shade200;

  @override
  Widget build(BuildContext context) {
    // 使用NotificationListener包裹子树
    return NotificationListener<ColorChangeNotification>(
      // 收到通知时的回调
      onNotification: (notification) {
        setState(() {
          _containerColor = notification.color;
        });
        // 返回true表示事件已处理，不再向上冒泡
        return true;
      },
      child: Container(
        color: _containerColor,
        padding: const EdgeInsets.all(20),
        child: const DeeplyNestedButton(), // 嵌套很深的子组件
      ),
    );
  }
}

// 3. 在深层子组件中派发通知
class DeeplyNestedButton extends StatelessWidget {
  const DeeplyNestedButton({super.key});
  @override
  Widget build(BuildContext context) {
    return Center(
      child: ElevatedButton(
        child: const Text('Change Parent Color'),
        onPressed: () {
          // 派发通知，上层的NotificationListener可以接收到
          ColorChangeNotification(Colors.teal.shade100).dispatch(context);
        },
      ),
    );
  }
}
```

#### 状态管理方案对比与选择

| 方案                    | 适用场景           | 优点                 | 缺点                   |
| ----------------------- | ------------------ | -------------------- | ---------------------- |
| setState                | 组件内部状态       | 简单直接，性能好     | 无法跨组件共享         |
| 状态提升                | 父子/兄弟组件      | 易理解，Flutter 原生 | 代码冗余，层级深时繁琐 |
| InheritedWidget         | 跨层级状态共享     | Flutter 官方，高效   | 样板代码多，使用复杂   |
| Provider+ChangeNotifier | 中大型应用状态管理 | 功能强大，社区成熟   | 学习成本，依赖第三方库 |
| Notification            | 事件通知           | 解耦性好，事件冒泡   | 仅适用于通知场景       |

#### 建议

1. **优先使用简单方案**：从 setState 开始，根据需求逐步升级
2. **合理选择作用域**：局部状态用 setState，跨组件状态考虑 Provider
3. **避免过度工程化**：不要为了使用状态管理而使用
4. **关注性能**：合理使用 context.watch 和 context.read，避免不必要的重建
5. **保持数据流清晰**：明确状态的流向和更新路径

### 用户输入

#### 🔘 按钮组件

##### 1. ElevatedButton - 浮动按钮

具有一定深度的按钮，适合为扁平布局增添立体感。

```dart
ElevatedButton(
  onPressed: _incrementCounter,
  style: ElevatedButton.styleFrom(
    textStyle: const TextStyle(fontSize: 16),
  ),
  child: const Text('ElevatedButton'),
)
```

##### 2. OutlinedButton - 边框按钮

带有文本和可见边框的按钮，用于重要但非主要的操作。

```dart
OutlinedButton(
  onPressed: _incrementCounter,
  style: OutlinedButton.styleFrom(
    textStyle: const TextStyle(fontSize: 16),
  ),
  child: const Text('OutlinedButton'),
)
```

##### 3. FilledButton - 填充按钮

用于完成流程的重要、最终操作，例如保存、确认等。

```dart
FilledButton(
  onPressed: _incrementCounter,
  style: FilledButton.styleFrom(
    textStyle: const TextStyle(fontSize: 16),
  ),
  child: const Text('FilledButton'),
)
```

##### 4. IconButton - 图标按钮

用于执行简单操作的图标按钮。

```dart
IconButton(
  onPressed: _incrementCounter,
  icon: const Icon(Icons.add),
  style: IconButton.styleFrom(
    iconSize: 30,
    backgroundColor: Colors.blue,
    foregroundColor: Colors.white,
  ),
  tooltip: 'IconButton 图标按钮',
)
```

##### 5. FloatingActionButton - 浮动操作按钮

用于执行主要或高频操作的浮动按钮。

```dart
FloatingActionButton(
  onPressed: _incrementCounter,
  child: Icon(Icons.add, size: 30, color: Colors.white),
)
```

#### ✏️ 文本输入组件

文本组件是信息展示和用户输入的重要载体。

##### 1. Text - 普通文本

最基础的文本显示组件。

```dart
Text('Hello, World!')
```

##### 2. SelectableText - 可选择文本

允许用户选择和复制的文本组件。

```dart
SelectableText('Hello, World!')
```

##### 3. RichText - 富文本

支持多种样式混合的富文本组件。

```dart
RichText(
  text: TextSpan(
    text: 'Hello, World!',
    style: TextStyle(
      fontWeight: FontWeight.bold,
      fontSize: 20,
      color: Colors.black54,
    ),
  ),
)
```

##### 4. TextField - 文本输入框

基础的文本输入组件。

```dart
TextField(
  decoration: const InputDecoration(
    border: OutlineInputBorder(),
    labelText: '请输入文本',
  ),
)
```

##### 5. TextFormField - 表单文本输入框

带有验证功能的表单输入组件。

```dart
TextFormField(
  decoration: const InputDecoration(hintText: '请输入'),
  validator: (String? value) {
    if (value == null || value.isEmpty) {
      return '请输入内容';
    }
    return null;
  },
)
```

#### 🎯 选择类组件

选择组件帮助用户从预定义选项中进行选择。

##### 1. SegmentedButton - 分段按钮

用于在几个选项中进行单选的分段控制器。

```dart
SegmentedButton(
  segments: items.map((item) => ButtonSegment(
    value: item,
    label: Text(item),
    icon: Icon(Icons.toc),
  )).toList(),
  selected: {selectedItem},
  onSelectionChanged: (value) => onSelectionChange(value.first),
)
```

##### 2. Chip - 标签

用于显示信息或选择的紧凑元素。

```dart
Chip(
  label: Text('标签文本'),
  avatar: CircleAvatar(backgroundColor: Colors.blue),
)
```

##### 3. DropdownMenu - 下拉菜单

节省空间的下拉选择组件。

```dart
DropdownMenu(
  initialSelection: selectedItem,
  onSelected: (value) => onSelectionChange(value),
  dropdownMenuEntries: items
      .map((item) => DropdownMenuEntry(value: item, label: item))
      .toList(),
)
```

##### 4. Slider - 滑块

用于从连续范围中选择值的滑块组件。

```dart
Slider(
  value: sliderValue,
  max: 100,
  divisions: 10,
  label: sliderValue.toString(),
  onChanged: (value) => setState(() => sliderValue = value),
)
```

#### 🔄 切换类组件

切换组件用于在两个或多个状态之间进行切换。

##### 1. Switch - 开关

二元状态切换开关。

```dart
Switch(
  activeColor: Colors.red,
  value: isSelected,
  onChanged: (value) => setState(() => isSelected = value),
)
```

##### 2. Checkbox - 复选框

用于多选场景的复选框。

```dart
Checkbox(
  activeColor: Colors.red,
  value: isSelected,
  onChanged: (value) => setState(() => isSelected = value),
)
```

##### 3. Radio - 单选框

用于单选场景的单选按钮。

```dart
Radio(
  value: Character.musician,
  onChanged: (value) => setState(() => selectedCharacter = value),
  groupValue: selectedCharacter,
  activeColor: Colors.red,
)
```

##### 4. CheckboxListTile - 复选框列表

包含复选框的列表项。

```dart
CheckboxListTile(
  title: Text('选项标题'),
  value: isSelected,
  onChanged: (value) => setState(() => isSelected = value),
  activeColor: Colors.red,
  secondary: Icon(Icons.person),
)
```

##### 5. SwitchListTile - 开关列表

包含开关的列表项。

```dart
SwitchListTile(
  value: isSelected,
  onChanged: (value) => setState(() => isSelected = value),
  title: Text('开关标题'),
)
```

#### 📅 日期时间选择

日期时间选择组件用于时间相关的用户输入。

##### 1. DatePickerDialog - 日期选择器

系统级别的日期选择对话框。

```dart
Future<DateTime?> showDatePickerDialog() async {
  return await showDatePicker(
    context: context,
    initialEntryMode: DatePickerEntryMode.calendar,
    initialDate: DateTime.now(),
    firstDate: DateTime(2019),
    lastDate: DateTime(2030),
  );
}
```

##### 2. TimePickerDialog - 时间选择器

系统级别的时间选择对话框。

```dart
Future<TimeOfDay?> showTimePickerDialog() async {
  return await showTimePicker(
    context: context,
    initialTime: TimeOfDay.now(),
    initialEntryMode: TimePickerEntryMode.dial,
  );
}
```

#### 👆 滑动操作组件

滑动组件提供基于手势的交互方式。

##### Dismissible - 可滑动删除

允许用户通过滑动手势删除项目的组件。

```dart
Dismissible(
  background: Container(color: Colors.green),
  key: ValueKey<int>(items[index]),
  onDismissed: (DismissDirection direction) {
    setState(() {
      items.removeAt(index);
    });
  },
  child: ListTile(title: Text('Item ${items[index]}')),
)
```

### 用户交互

#### 1. SnackBar 消息提示

##### 概述

SnackBar 是 Flutter 中用于显示简短消息的组件，通常用于提供用户操作的反馈。它会从屏幕底部滑入，并在一段时间后自动消失。

##### 基本实现

```dart
import 'package:flutter/material.dart';

class SnackBarPage extends StatelessWidget {
  const SnackBarPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: ElevatedButton(
        onPressed: () {
          final snackBar = SnackBar(
            content: const Text('Yay! A SnackBar!'),
            action: SnackBarAction(
              label: 'Undo',
              onPressed: () {
                // 撤销操作的代码
              },
            ),
            showCloseIcon: true,
          );

          // 使用 ScaffoldMessenger 显示 SnackBar
          ScaffoldMessenger.of(context).showSnackBar(snackBar);
        },
        child: const Text('Show SnackBar'),
      ),
    );
  }
}
```

##### 关键特性

- **自动消失**: 默认 4 秒后自动隐藏
- **操作按钮**: 可添加 `SnackBarAction` 提供快速操作
- **关闭图标**: `showCloseIcon: true` 显示关闭按钮
- **全局管理**: 通过 `ScaffoldMessenger` 进行管理

##### 使用场景

- 操作成功/失败反馈
- 撤销操作提示
- 网络状态通知
- 表单提交结果

---

#### 2. 输入表单处理

##### 2.1 基础文本输入

###### TextEditingController 管理

```dart
class TextInputExample extends StatefulWidget {
  const TextInputExample({super.key});

  @override
  State<TextInputExample> createState() => _TextInputExample();
}

class _TextInputExample extends State<TextInputExample> {
  final myController = TextEditingController();
  late FocusNode myFocusNode;

  _printText() {
    final text = myController.text;
    print('Current text: $text');
  }

  @override
  void initState() {
    super.initState();
    myFocusNode = FocusNode();
    // 监听文本变化
    myController.addListener(_printText);
  }

  @override
  void dispose() {
    // 重要：释放资源
    myController.dispose();
    myFocusNode.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextField(
          controller: myController,
          onChanged: (e) {
            print('Text changed: $e');
          },
          autofocus: true,
        ),
        TextFormField(
          controller: myController,
          focusNode: myFocusNode
        ),
      ],
    );
  }
}
```

###### 核心概念

- **TextEditingController**: 控制文本输入的内容
- **FocusNode**: 管理输入框的焦点状态
- **监听器**: `addListener()` 监听文本变化
- **资源管理**: 在 `dispose()` 中释放控制器

##### 2.2 表单验证

```dart
class FormInputExample extends StatefulWidget {
  const FormInputExample({super.key});

  @override
  State<FormInputExample> createState() => _FormInputExample();
}

class _FormInputExample extends State<FormInputExample> {
  final GlobalKey<FormState> _formKey = GlobalKey<FormState>();

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      child: Column(
        children: [
          TextFormField(
            validator: (value) {
              if (value == null || value.isEmpty) {
                return 'Please enter some text';
              }
              return null;
            },
          ),
          ElevatedButton(
            onPressed: () {
              // 验证表单
              if (_formKey.currentState!.validate()) {
                ScaffoldMessenger.of(context).showSnackBar(
                  const SnackBar(content: Text('Processing Data')),
                );
              }
            },
            child: const Text('Submit'),
          ),
        ],
      ),
    );
  }
}
```

###### 表单验证要点

- **GlobalKey**: 用于访问 Form 的状态
- **validator**: 定义验证规则
- **validate()**: 触发所有字段验证
- **反馈机制**: 结合 SnackBar 提供用户反馈

#### 3. 用户手势交互

##### 3.1 水波纹效果 (InkWell)

```dart
class InkWellExample extends StatelessWidget {
  const InkWellExample({super.key});

  @override
  Widget build(BuildContext context) {
    return Center(
      child: InkWell(
        onTap: () {
          // 点击事件处理
        },
        child: Container(
          padding: EdgeInsets.all(16),
          child: Text('可点击的组件'),
        ),
      ),
    );
  }
}
```

##### 3.2 滑动删除 (Dismissible)

```dart
class ListCleanExample extends StatefulWidget {
  const ListCleanExample({super.key});

  @override
  State<ListCleanExample> createState() => _ListCleanExampleState();
}

class _ListCleanExampleState extends State<ListCleanExample> {
  List<ListTile> _tiles = [];

  @override
  void initState() {
    super.initState();
    _tiles = List.generate(
      100,
      (index) => ListTile(title: Text('title $index'))
    );
  }

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: _tiles.length,
      itemBuilder: (context, index) {
        final item = _tiles[index];
        return Dismissible(
          key: Key(item.title.toString()),
          child: item,
          onDismissed: (direction) {
            setState(() {
              _tiles.removeAt(index);

              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(
                  content: Text('${item.title} 已删除'),
                  backgroundColor: Colors.red,
                  duration: Duration(milliseconds: 1000),
                ),
              );
            });
          },
        );
      },
    );
  }
}
```

###### Dismissible 关键特性

- **唯一 Key**: 每个 Dismissible 需要唯一的 key
- **滑动方向**: 可设置水平或垂直滑动
- **删除回调**: `onDismissed` 处理删除逻辑
- **用户反馈**: 结合 SnackBar 提供删除确认

##### 3.3 拖拽操作 (Drag & Drop)

```dart
class AppDragExample extends StatefulWidget {
  const AppDragExample({super.key});

  @override
  State<AppDragExample> createState() => _AppDragExampleState();
}

class _AppDragExampleState extends State<AppDragExample> {
  void _itemDroppedOnCustomerCart({
    required Item item,
    required Customer customer,
  }) {
    setState(() {
      customer.items.add(item);
    });
  }

  Widget _buildMenuItem({required Item item}) {
    return LongPressDraggable<Item>(
      data: item,
      dragAnchorStrategy: pointerDragAnchorStrategy,
      feedback: DraggingListItem(
        dragKey: _draggableKey,
        photoProvider: item.imageProvider,
      ),
      child: MenuListItem(
        name: item.name,
        price: item.formattedTotalItemPrice,
        photoProvider: item.imageProvider,
      ),
    );
  }

  Widget _buildPersonWithDropZone(Customer customer) {
    return DragTarget<Item>(
      builder: (context, candidateItems, rejectedItems) {
        return CustomerCart(
          hasItems: customer.items.isNotEmpty,
          highlighted: candidateItems.isNotEmpty,
          customer: customer,
        );
      },
      onAcceptWithDetails: (details) {
        _itemDroppedOnCustomerCart(
          item: details.data,
          customer: customer
        );
      },
    );
  }
}
```

###### 拖拽实现要点

- **LongPressDraggable**: 长按开始拖拽
- **DragTarget**: 定义放置目标
- **feedback**: 拖拽过程中显示的组件
- **数据传递**: 通过泛型传递拖拽数据
- **视觉反馈**: 高亮显示拖拽目标

---

#### 4. 最佳实践总结

##### 4.1 资源管理

```dart
@override
void dispose() {
  // 控制器
  myController.dispose();
  // 焦点节点
  myFocusNode.dispose();
  // 定时器
  timer?.cancel();
  super.dispose();
}
```

##### 4.2 性能考虑

###### 列表优化

```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    // 只构建可见项目
    return buildItem(items[index]);
  },
)
```

###### 状态管理

- 避免不必要的 `setState()` 调用
- 使用 `const` 构造函数优化性能
- 合理使用 `GlobalKey`

##### 4.3 可访问性

```dart
Semantics(
  button: true,
  label: '删除按钮',
  child: IconButton(
    onPressed: onDelete,
    icon: Icon(Icons.delete),
  ),
)
```

##### 4.4 错误处理

```dart
try {
  await submitForm();
  _showSuccessSnackBar();
} catch (e) {
  _showErrorSnackBar(e.toString());
}
```

### 路由导航

#### 1 基础路由导航

Flutter 使用 `Navigator` 类来管理页面间的导航，它采用栈（Stack）的方式管理路由：

- `Navigator.push()` - 推入新页面
- `Navigator.pop()` - 弹出当前页面
- `Navigator.pushReplacement()` - 替换当前页面

```dart
class FirstRoute extends StatelessWidget {
  const FirstRoute({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('First Route')),
      body: Center(
        child: ElevatedButton(
          child: const Text('Open route'),
          onPressed: () {
            // 导航到第二个页面
            Navigator.push(
              context,
              MaterialPageRoute(
                builder: (context) => const SecondRoute(),
                settings: RouteSettings(arguments: 'Hello from FirstRoute'),
              ),
            );
          },
        ),
      ),
    );
  }
}

class SecondRoute extends StatelessWidget {
  const SecondRoute({super.key});

  @override
  Widget build(BuildContext context) {
    // 接收传递的参数
    final String? _message =
        ModalRoute.of(context)?.settings.arguments as String?;

    return Scaffold(
      appBar: AppBar(title: const Text('Second Route')),
      body: Center(
        child: ElevatedButton(
          onPressed: () {
            // 显示接收到的消息
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text(_message ?? 'No message'))
            );
            // 返回上一页面
            Navigator.pop(context);
          },
          child: const Text('go back'),
        ),
      ),
    );
  }
}
```

MaterialPageRoute

- 提供平台特定的过渡动画
- iOS：右滑进入，Android：底部上滑

RouteSettings

- 用于传递路由参数和配置
- `arguments` 属性用于传递数据

ModalRoute.of(context)

- 获取当前路由信息
- 访问传递的参数

#### 2. 抽屉导航（Drawer）

抽屉导航提供了侧边菜单功能，通常用于应用的主要导航选项。它会从屏幕左侧滑出，提供便捷的导航体验。

```dart
class DrawerExample extends StatelessWidget {
  const DrawerExample({super.key});

  @override
  Widget build(BuildContext context) {
    return Drawer(
      child: ListView(
        padding: EdgeInsets.all(8.0),
        children: [
          // 抽屉头部
          const DrawerHeader(
            decoration: BoxDecoration(color: Colors.blue),
            child: Text(
              'Drawer Header',
              style: TextStyle(color: Colors.white, fontSize: 24),
            ),
          ),
          // 菜单项
          ListTile(
            title: const Text('Item 1'),
            onTap: () {
              Navigator.pop(context); // 关闭抽屉
            },
          ),
          ListTile(
            title: const Text('Item 2'),
            onTap: () {
              Navigator.pop(context); // 关闭抽屉
            },
          ),
        ],
      ),
    );
  }
}
```

集成抽屉到页面

```dart
class DrawerExamplePage extends StatelessWidget {
  const DrawerExamplePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Drawer Example')),
      drawer: const DrawerExample(), // 添加抽屉
      body: Center(
        child: Builder(
          builder: (context) {
            return ElevatedButton(
              onPressed: () {
                // 通过代码打开抽屉
                Scaffold.of(context).openDrawer();
              },
              child: const Text('Open Drawer'),
            );
          },
        ),
      ),
    );
  }
}
```

打开方式

1. **自动**: AppBar 自动显示菜单图标
2. **手势**: 从左边缘右滑
3. **编程**: `Scaffold.of(context).openDrawer()`

DrawerHeader 组件

- 提供抽屉顶部的标题区域
- 可自定义背景颜色和内容
- 通常显示应用 Logo 或用户信息

#### 3. 页面间数据传递

Flutter 提供多种方式在页面间传递数据：

- 正向传递：通过构造函数或 RouteSettings
- 反向传递：通过 Navigator.pop()返回值

##### 3.1 异步数据返回示例

```dart
class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Returning Data Demo')),
      body: const Center(child: SelectionButton()),
    );
  }
}

class SelectionButton extends StatefulWidget {
  const SelectionButton({super.key});

  @override
  State<SelectionButton> createState() => _SelectionButtonState();
}

class _SelectionButtonState extends State<SelectionButton> {
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () {
        _navigateAndDisplaySelection(context);
      },
      child: const Text('Pick an option, any option!'),
    );
  }

  // 异步导航并等待返回结果
  Future<void> _navigateAndDisplaySelection(BuildContext context) async {
    ScaffoldMessenger.of(context).removeCurrentSnackBar();

    // Navigator.push 返回 Future，在调用 Navigator.pop 时完成
    final result = await Navigator.push(
      context,
      MaterialPageRoute(builder: (context) => const SelectionScreen()),
    );

    if (!context.mounted) return;

    // 处理返回的结果
    if (result != null) {
      ScaffoldMessenger.of(context)
        ..removeCurrentSnackBar()
        ..showSnackBar(SnackBar(content: Text('You selected: $result')));
    }
  }
}
```

##### 3.2 选择页面实现

```dart
class SelectionScreen extends StatelessWidget {
  const SelectionScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Pick an option')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Padding(
              padding: const EdgeInsets.all(8),
              child: ElevatedButton(
                onPressed: () {
                  // 返回数据给上一个页面
                  Navigator.pop(context, 'Yep!');
                },
                child: const Text('Yep!'),
              ),
            ),
            Padding(
              padding: const EdgeInsets.all(8),
              child: ElevatedButton(
                onPressed: () {
                  // 返回数据给上一个页面
                  Navigator.pop(context, 'Nope.');
                },
                child: const Text('Nope.'),
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

##### 3.3 数据传递方式总结

###### 1. 正向传递（父 → 子）

```dart
// 方式1：构造函数传递
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => DetailPage(data: myData),
  ),
);

// 方式2：RouteSettings传递
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => DetailPage(),
    settings: RouteSettings(arguments: myData),
  ),
);
```

###### 2. 反向传递（子 → 父）

```dart
// 子页面：返回数据
Navigator.pop(context, resultData);

// 父页面：接收数据
final result = await Navigator.push(context, route);
if (result != null) {
  // 处理返回的数据
}
```

---

#### 4. 最佳实践与总结

##### 4.1 导航最佳实践

###### 1. 异步安全检查

```dart
Future<void> navigate() async {
  final result = await Navigator.push(context, route);
  if (!context.mounted) return; // 检查context是否仍然有效
  // 使用result...
}
```

###### 2. 统一路由管理

```dart
class AppRoutes {
  static const String home = '/home';
  static const String detail = '/detail';

  static Route<dynamic> generateRoute(RouteSettings settings) {
    switch (settings.name) {
      case home:
        return MaterialPageRoute(builder: (_) => HomePage());
      case detail:
        return MaterialPageRoute(builder: (_) => DetailPage());
      default:
        return MaterialPageRoute(builder: (_) => NotFoundPage());
    }
  }
}
```

##### 4.2 抽屉导航最佳实践

###### 1. 响应式设计

```dart
class ResponsiveDrawer extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    if (MediaQuery.of(context).size.width > 600) {
      return NavigationRail(); // 桌面端使用导航栏
    } else {
      return Drawer(); // 移动端使用抽屉
    }
  }
}
```

###### 2. 状态管理

```dart
class DrawerMenu extends StatelessWidget {
  final String currentRoute;

  const DrawerMenu({required this.currentRoute});

  @override
  Widget build(BuildContext context) {
    return Drawer(
      child: ListView(
        children: [
          DrawerHeader(/*...*/),
          ListTile(
            title: Text('Home'),
            selected: currentRoute == '/home',
            onTap: () {
              Navigator.pop(context);
              if (currentRoute != '/home') {
                Navigator.pushNamed(context, '/home');
              }
            },
          ),
        ],
      ),
    );
  }
}
```

##### 4.3 性能优化

###### 1. 延迟加载

```dart
Navigator.push(
  context,
  MaterialPageRoute(
    builder: (context) => const LazyLoadedPage(),
    settings: RouteSettings(name: '/lazy'),
  ),
);
```

###### 2. 预加载关键页面

```dart
void preloadImportantPages() {
  precacheImage(AssetImage('assets/important_image.png'), context);
}
```

### 数据持久化

Flutter 提供了多种数据持久化方案，每种方案都有其特定的适用场景。本学习记录总结了四种主要的持久化技术：SharedPreferences、FlutterSecureStorage、本地文件存储和 SQLite 数据库。

#### 1. SharedPreferences - 轻量级键值对存储

##### 📦 依赖配置

```yaml
dependencies:
  shared_preferences: ^2.0.15
```

##### 🎯 核心特点

- **平台实现**：Android 使用 SharedPreferences API，iOS 使用 NSUserDefaults
- **数据格式**：简单键值对（String、int、bool、double、List\<String\>）
- **存储方式**：XML 文件（Android）或 plist 文件（iOS）
- **访问方式**：异步操作，通过 Platform Channels 通信

##### 💡 适用场景

- ✅ 用户偏好设置（主题、语言、字体大小）
- ✅ 状态标记（是否看过引导页、是否同意协议）
- ✅ 轻量级缓存（搜索历史关键词）
- ❌ 不适合大量或复杂数据

##### 🔧 核心 API 使用

```dart
// 获取实例
final prefs = await SharedPreferences.getInstance();

// 读取数据
int counter = prefs.getInt('counter') ?? 0;

// 写入数据
await prefs.setInt('counter', counter);
```

#### 2. Flutter Secure Storage - 安全敏感数据存储

##### 📦 依赖配置

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
```

##### 🎯 核心特点

- **安全性**：使用系统级加密存储
- **数据保护**：适用于密码、API 密钥等敏感信息
- **访问控制**：提供生物识别等额外安全层
- **平台集成**：深度集成系统安全机制

##### 💡 适用场景

- ✅ 用户登录凭证（用户名、密码、Token）
- ✅ API 密钥和数据库密码
- ✅ 个人隐私信息
- ❌ 不适合频繁读写的数据

##### 🔧 核心 API 使用

```dart
// 创建实例
const storage = FlutterSecureStorage();

// 写入敏感数据
await storage.write(key: 'api_secret_key', value: secretValue);

// 读取数据
String? value = await storage.read(key: 'api_secret_key');

// 删除数据
await storage.delete(key: 'api_secret_key');
```

#### 3. 本地文件存储 - 灵活的文件操作

##### 📦 依赖配置

```yaml
dependencies:
  path_provider: ^2.0.0 # 获取系统路径
```

##### 🎯 核心特点

- **灵活性**：支持任意格式数据（文本、JSON、二进制）
- **跨平台**：使用 path_provider 解决路径差异
- **直接控制**：完全控制文件读写过程
- **存储位置**：应用专属目录

##### 💡 适用场景

- ✅ 缓存 API 响应（复杂 JSON 数据）
- ✅ 用户生成内容（笔记、绘图数据、日志）
- ✅ 应用配置文件
- ✅ 离线数据缓存

##### 🔧 核心 API 使用

```dart
// 获取应用目录
final directory = await getApplicationDocumentsDirectory();
final file = File('${directory.path}/data.json');

// 写入文件
await file.writeAsString(jsonEncode(data));

// 读取文件
if (await file.exists()) {
  final contents = await file.readAsString();
  final data = jsonDecode(contents);
}

// 删除文件
await file.delete();
```

#### 4. SQLite 数据库 - 结构化数据管理

##### 📦 依赖配置

```yaml
dependencies:
  sqflite: ^2.0.3
  path: ^1.8.2
```

##### 🎯 核心特点

- **关系型数据库**：支持复杂的 SQL 查询操作
- **高性能**：针对大量数据优化
- **ACID 特性**：保证数据一致性和完整性
- **灵活查询**：支持联表、聚合、索引等高级功能

##### 💡 适用场景

- ✅ 数据密集型应用（记账、TODO、日记）
- ✅ 需要复杂查询的应用（多条件筛选、排序）
- ✅ 离线优先应用（本地缓存+同步）
- ✅ 多表关联数据

##### 🔧 核心实现模式

数据模型设计

```dart
class Dog {
  final int id;
  final String name;
  final int age;

  Dog({required this.id, required this.name, required this.age});

  Map<String, dynamic> toMap() {
    return {'id': id, 'name': name, 'age': age};
  }
}
```

数据库工具类（单例模式）

```dart
class DataBaseHelper {
  static final DataBaseHelper _instance = DataBaseHelper._internal();
  factory DataBaseHelper() => _instance;
  static Database? _database;
  DataBaseHelper._internal();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDatabase();
    return _database!;
  }
}
```

#### 🎯 选择指南

| 存储方案          | 数据量 | 复杂度 | 安全性 | 查询能力 | 适用场景   |
| ----------------- | ------ | ------ | ------ | -------- | ---------- |
| SharedPreferences | 小     | 简单   | 一般   | 键值查找 | 用户设置   |
| Secure Storage    | 小     | 简单   | 高     | 键值查找 | 敏感信息   |
| 本地文件          | 中等   | 中等   | 一般   | 文件操作 | 缓存数据   |
| SQLite            | 大     | 复杂   | 一般   | SQL 查询 | 结构化数据 |
