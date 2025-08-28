---
title: Flutter GetX & Obx概念
date: 2025-05-14 10:01:00
tags:
  - App
  - Mobile
categories: 程式語言
---

# 正文

什麼是 GetX?

- 狀態管理(State Management)
- 路由管理(Route Management)
- 相依性注入(Dependency Injection)

什麼是 Obx()?
`Obx()` 是 GetX Observable Widget，用來改動的變數(Rx Variable)，當值改變時自動 re-render(更新 UI)。

例子:

```Dart
Obx(() => Text(controller.count.toString()));
```

當`controller.count`(一個 Rx 變數)改變時，Text()自動更新。

Rx 是什麼?
-> Observable Variable
當你想使用 Observable,你必須在定義變數的時候先把他定義成要觀察的變數，GetX 才會知道需要 trigger 它。

如何定義它？
在變數後面+ `.obs`

```Dart
final RxBool isUploading = false.obs
```

整合使用：

```Dart

import 'package:flutter/material.dart';
import 'package:get/get.dart';

class CounterController extends GetxController {
  final count = 0.obs;
  void increment() => count++;
}

void main(){
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  final CounterController controller = Get.put(CounterController());


  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(title: Text('GetX Obx Demo')),
        body: Center(
          child: Obx(() => Text('Count: ${controller.count}', style: TextStyle(fontSize: 30))),
        ),
        floatingActionButton: FloatingActionButton(
          onPressed: controller.increment,
          child: Icon(Icons.add),
        ),
      ),
    );
  }
}

```

結論：
`.obs`: 把變數變成可被監聽
`Obx()`: 監聽`.obs` 變數，當值改變就更新 UI
`Get.put()`: 注入控制器，方便全域使用

# 補充

和 React 哪部分概念相通？
| 概念 | Flutter + GetX | React |
| -------- | ------------------------------------------- | ---------------------------------------- |
| 宣告可響應的變數 | `var count = 0.obs;` | `const [count, setCount] = useState(0);` |
| UI 自動更新 | `Obx(() => Text('$count'))` | `return <p>{count}</p>` |
| 改變狀態 | `count++` 或 `count.value++` | `setCount(count + 1)` |
| 綁定狀態到 UI | `Obx()` 包住 Widget | JSX 中直接使用變數 |
| 控制邏輯集中 | Controller class (`extends GetxController`) | 通常放在 component 中或 custom hook |
| 全域注入 | `Get.put(MyController())` | React Context / Redux / Zustand 等 |

相通核心概念：

- 響應式設計： UI 綁定的是狀態(state)，狀態一改變 UI 自動更新。
- 分離邏輯： 把狀態控制和 UI 拆開寫，更容易維護(GetX 的 Controller v.s. React 的 hook/context)。
- 單向數據流： 使用者觸發事件 -> 改變狀態 -> UI 更新(單向 flow)。
- component-based：Flutter widget tree & React component tree 類似，都是組件化思維。

```jsx
const [count, setCount] = useState(0);

return (
  <div>
    <p>{count}</p>
    <button onClick={() => setCount(count + 1)}>Add</button>
  </div>
);
```

```Dart
final count = 0.obs;
Obx(()=> Text('$count'));
FloatingActionButton(onPressed: () => count++);

```
