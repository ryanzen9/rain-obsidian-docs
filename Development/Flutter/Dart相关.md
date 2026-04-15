---
Title: Dart相关
Draft: false
tags:
  - dart
  - flutter
  - language
Author: Ruby Ceng
---

## 相关文档

> Dart 语言 JavaScript 工程师迁移学习文档：`https://dart.dev/resources/coming-from/js-to-dart#conventions-and-linting`

## Dart 数据类型

| Dart 类型   | 说明              | 等价的 TypeScript 类型  | 关键区别与备注                                               |
| ----------- | ----------------- | ----------------------- | ------------------------------------------------------------ |
| `int`       | 整数              | `number`                | Dart 明确区分整数和浮点数。                                  |
| `double`    | 浮点数            | `number`                | TS 的 `number` 涵盖了 Dart 的 `int` 和 `double`。            |
| `num`       | `int` 或 `double` | `number`                | 这是与 TS `number` 最接近的对等体。                          |
| `String`    | 字符串            | `string`                | 功能几乎一样，多行字符串语法不同。                           |
| `bool`      | 布尔值            | `boolean`               | **Dart 更严格**，没有 "truthy/falsy" 的概念。                |
| `List<T>`   | 有序列表（数组）  | `Array<T>` 或 `T[]`     | **完全等同。**                                               |
| `Set<T>`    | 唯一项的无序集合  | `Set<T>`                | **完全等同。**                                               |
| `Map<K, V>` | 键值对集合        | `Map<K, V>` 或对象/接口 | Dart 更倾向于用 `Map` 表示动态对象，而 TS 常用 `interface`。 |

## Dart 变量关键字：const, final, var

### 核心定义

- **`const`**: 编译时常量。其值在代码编译那一刻就必须确定。用于性能优化。
- **`final`**: 运行时常量。变量只能被赋值一次，其值在程序运行时确定。保证了不可变性。
- **`var`**: 变量。它的值可以被多次修改。类型由第一次赋值时推断并确定。

### 使用方法

1. **`const`**

   Dart

   ```
   // 用于定义应用中永不改变的常量
   const Duration timeout = Duration(seconds: 5);
   const String appName = 'My Awesome App';

   // 用于不会改变的 Flutter Widget 以优化性能
   const Text('Hello, World!');
   ```

2. **`final`**

   Dart

   ```
   // 用于只需赋值一次的变量，例如 Widget 的属性或 API 返回值
   final String userId = fetchUserId(); // fetchUserId() 是一个运行时函数
   final apiResponse = await http.get(url);

   class MyWidget extends StatelessWidget {
     final String title; // Widget 属性必须是 final
     const MyWidget({super.key, required this.title});
   }
   ```

3. **`var`**

   Dart

   ```
   // 用于值需要改变的局部变量
   var counter = 0;
   counter = counter + 1;

   for (var i = 0; i < 5; i++) {
     print(i);
   }
   ```

**黄金法则 (何时使用):**

- **首选 `const`**：如果值在编译时就确定。
- **其次 `final`**：如果变量只需赋值一次。
- **最后 `var`**：如果变量的值确实需要改变。

## Dart 空安全 (Null Safety)

### 核心定义

一种语言特性，规定变量默认不可为空（non-nullable）。这意味着，除非你显式声明，否则变量不能持有 `null` 值。其目的是在编译时就消灭“空引用”错误。

- **不可空类型 (默认)**：变量不能为 `null`。
  Dart
  ```
  String name = "Dash"; // 正确
  // name = null; // 编译错误
  ```
- **可空类型 (需显式声明)**：在类型后加 `?`，表示该变量可以为 `null`。
  Dart
  ```
  String? address; // 正确，address 默认为 null
  address = "123 Main St"; // 正确
  ```

### 使用方法

在使用可空类型时，编译器会强制你进行安全检查，以防止在 `null` 上调用方法或属性。

1. **空值检查**

   Dart

   ```
   void printAddressLength(String? address) {
     if (address != null) {
       // 在这个代码块内，编译器确认 address 不为 null
       print(address.length);
     }
   }
   ```

2. **关键操作符**

   - `?.` **(可选链操作符)**: 如果对象不为 `null`，则调用其方法或属性；否则，整个表达式返回 `null`。
     Dart
     ```
     int? length = address?.length;
     ```
   - `!` **(非空断言)**: 告诉编译器“我确信这个变量此刻不为 `null`”。如果它实际上是 `null`，你的程序将在运行时崩溃。请谨慎使用。
     Dart
     ```
     int length = address!; // 如果 address 是 null，这里会抛出异常
     ```
   - `??` **(空值合并操作符)**: 如果左侧表达式的结果是 `null`，则返回右侧的值。
     Dart
     ```
     String currentAddress = address ?? "No Address Provided";
     ```
   - `late` **(延迟初始化)**: 向编译器承诺，你会在使用一个非空变量之前对它进行初始化。
     Dart

     ```

     class MyService {
       late final String _apiKey; // 声明一个非空变量

       void initialize() {
         _apiKey = _fetchApiKey(); // 在使用前进行初始化
       }

       String _fetchApiKey() => "some_secret_key";
     }
     ```
