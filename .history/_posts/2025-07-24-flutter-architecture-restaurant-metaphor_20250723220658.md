---
title: 'Demystifying Flutter Architecture: Understanding iOS Menu System Through a Restaurant Metaphor'
date: 2025-07-24
permalink: /posts/2025/07/flutter-architecture-restaurant-metaphor/
tags:
  - flutter
  - architecture
  - iOS
  - framework
  - engine
---

Let's use a restaurant kitchen metaphor to understand how Flutter's menu system works, specifically the complex interactions between iOS app, Framework, and Engine! — Inspired by Justin

<!--more-->

**Who is this for?** This article is for Flutter developers (from beginners to intermediate) who want to understand how Flutter works under the hood, especially the interaction between Framework and Engine layers.

💡 **Building your own Flutter Engine?** Check out my guide: [Your First Step to Contributing: Building Flutter Engine and Framework Development](/posts/2025/07/flutter-engine-guide/)

Think about what happens when we walk into a restaurant and enjoy a delicious meal. Right, as customers we need to order food, the waiter records our request and passes it to the kitchen, the kitchen receives the order and delegates it to the appropriate chef based on the dish type! Once the chef finishes cooking, we get to enjoy our meal! So let's imagine Flutter's process works like ordering food at a restaurant!

Just a heads up, this might seem like a silly comparison at first, but trust me, by the end of this article you'll never look at `FlutterTextInputPlugin.mm` the same way again. (And yes, that's a real file name - I know it sounds like someone fell asleep on their keyboard!)

**Customer (iOS Developer) orders → Waiter (Framework) takes the order → Kitchen (Engine) cooks → Customer enjoys the meal (System Menu)**

![Flutter Restaurant Metaphor - Customer to Kitchen Flow](../images/flutter-restaurant-metaphor.png)
*The complete flow: Customer (iOS Developer) → Waiter (Flutter Framework) → Kitchen (Flutter Engine) → System Menu*

## The Architecture Flow

Here's a visual representation of how the system works:

```
`[iOS App Developer]`
    ↓ (uses `TextField`/`CupertinoTextField`)
`[Flutter Framework - Dart]`
    ↓ (Platform Channel: `"ContextMenu.showSystemContextMenu"`)
`[Flutter Engine - Objective-C]`
    ↓ (`FlutterPlatformPlugin.mm` → `FlutterTextInputPlugin.mm`)
`[iOS Native System Menu]`
    ↓ (`UIMenuController`)
`[User sees the menu!]`
```

## Step 1: Customer Places Order (iOS App Development Layer)

Most developers use `TextField` or `CupertinoTextField` to create editable text fields. But under the hood, both of these widgets rely on a core component: **`EditableText`**.

Developers can customize the menu that pops up after long-pressing a text field through properties like `TextField.contextMenuBuilder` or `CupertinoTextField.contextMenuBuilder`. It's like a customer telling the waiter: "I don't want the default meal! I want to customize my order!"

On iOS, if developers **don't customize the menu**, `TextField` defaults to using a built-in menu called **`SystemContextMenu`**. Its job is very specific - just taking orders. It doesn't cook the menu itself, it's just a messenger. It's like the customer telling the waiter: "Hey, I'll just take the default menu, let the iOS kitchen prepare a standard native menu!" (Because let's be honest, sometimes we're too lazy to customize everything, and that's totally fine!)

## Step 2: Waiter Takes Order and Passes to Kitchen (From Framework to Engine)

Now the "waiter" (Framework) has received the instruction to "show native menu". They need to accurately convey this instruction to the "kitchen" (Engine). This communication process uses a tool called **Platform Channels** (Flutter's mechanism for Dart-native communication). Think of it as a walkie-talkie, specifically managed by text_input.dart, for communication between Dart code (in Framework) and native code (Objective-C, in Engine).

1. `SystemContextMenu` uses a controller called `SystemContextMenuController` (manages native iOS menu display)
2. Through platform channels, it sends a message named `"ContextMenu.showSystemContextMenu"`

This message is like the waiter shouting into the walkie-talkie: "Table 7 wants the salmon!" (Except in our case it's more like "Table UITextField wants the SystemContextMenu!" - not as appetizing, I know.)

## Step 3: Kitchen Prepares the Dish (Engine Layer)

Now the message reaches the "kitchen floor manager" through the walkie-talkie — `FlutterPlatformPlugin.mm`, which is the "first stop" in the engine for receiving all platform messages. It's super busy, receiving all kinds of instructions (not just text-related). Think of it as that one manager who's running around with three phones, a clipboard, and somehow still knows exactly where everything goes. When it receives an instruction, it figures out which "chef" should handle it.

When it hears `"ContextMenu.showSystemContextMenu"`, it knows to pass this task to the chef specialized in text input: **Head Text Chef: `FlutterTextInputPlugin.mm`**, the expert who handles all "text input" related work.

Inside `FlutterTextInputPlugin.mm`, there's a crucial part. It **decides** which buttons should appear on the final menu based on the current state. For example, the Lookup button: [View source code](https://github.com/flutter/flutter/blob/b5dbfc3a425435d126e629ed4c73d78507a27291/engine/src/flutter/shell/platform/darwin/ios/framework/Source/FlutterTextInputPlugin.mm#L997)

## Real Kitchen Story: Adding a New Dish (Live Text) to the Menu

Let me share a real story from the Flutter kitchen! Just like how a popular dish might disappear from a restaurant menu and customers complain, iOS users noticed that the **Live Text** option (a feature that lets you extract text from images) disappeared from text field menus after a Secure Paste update. (Yeah, sometimes fixing one thing breaks another - welcome to software development!)

### The Problem: Missing Dish on the Menu
In PR [#170969](https://github.com/flutter/flutter/pull/170969), I had to add the Live Text option back to the iOS system context menu. It's like customers saying: "Hey, where did my favorite Live Text dish go? We want it back!"

### The Solution: Recipe Development

Just like adding a new dish requires coordination between the dining room and kitchen, here's how I fixed it:

#### 1. Update the Menu (Framework Side)
First, I created a new menu item class in the Framework:
```dart
// Like adding a new item to the restaurant's menu board
class IOSSystemContextMenuItemDataLiveText extends IOSSystemContextMenuItemData {
  const IOSSystemContextMenuItemDataLiveText({this.title});
  final String? title;
  // This tells the waiter: "We now serve Live Text!" 
  // (and hey, you can even customize the dish name!)
}
```

#### 2. Kitchen Preparation (Engine Side)
Then in the kitchen (`FlutterTextInputPlugin.mm`), here's the fun part - I actually SIMPLIFIED the recipe! The previous chef was overthinking it with unnecessary checks. I just told them:
```objc
// Simplified implementation - just let iOS handle it!
- (void)captureTextFromCamera:(id)sender {
    if (@available(iOS 15.0, *)) {
        [super captureTextFromCamera:sender];
    }
}
// That's it! Sometimes the best recipes are the simplest ones.
```

#### 3. Serving the Dish
The beauty of this fix? Once both the Framework (waiter) and Engine (kitchen) knew about Live Text, iOS users could once again enjoy this feature in their text field menus!

And here's the kicker - the whole fix was actually about REMOVING complexity, not adding it. Sometimes the best code is the code you delete. (Don't tell my manager I said that!)

#### 4. The Return Journey (When Users Click Live Text)
But wait, there's more! When a user actually clicks the Live Text button:
1. **iOS takes the order directly**: The click on the native menu goes straight to iOS
2. **Kitchen gets to work**: `captureTextFromCamera:` is called in our Engine code (which simply calls `[super captureTextFromCamera:sender]` - remember, we simplified this!)
3. **Special delivery**: iOS opens the camera with text recognition
4. **Kitchen updates the waiter**: Even though iOS handled the camera directly, the Framework still gets notified of the result
5. **Happy customer**: The recognized text flows back through the Engine → Framework → Your app!

It's like ordering a special dish that requires the chef to come out and prepare it at your table, but the waiter still needs to write it down in the order book to keep track of everything!

This real-world example shows exactly how the restaurant metaphor works in practice. Every new feature or bug fix in Flutter follows this same pattern: customer request → waiter (Framework) → kitchen (Engine) → happy customer!

## Bonus: What Happens When Users Click? (The Reverse Process)

This is where the story gets really interesting! The reverse process shows how user actions flow back through our restaurant.

### Flutter Input Field Clicks
When you click on a Flutter input field, this event stays entirely in the Dart world - the Flutter Framework handles it directly without needing to go to the native kitchen.

### Native Menu Button Clicks
But when you click a native menu button (like "Copy" or "Look Up"), something different happens:

1. **iOS Captures First**: The menu is actually a native `UIMenuController` element drawn by iOS itself, not by Flutter. So iOS captures the click event first - it's like the customer directly telling the kitchen staff what they want, bypassing the waiter!

2. **Kitchen Notifies the Waiter**: Here's the crucial part - even though the kitchen got the order directly, it still needs to notify our "waiter" (Framework) through a **callback mechanism**. The "head text chef" (`FlutterTextInputPlugin.mm`) says: "Hey waiter, the customer at table 7 just ordered the 'Look Up' special - update your order book!"

3. **Engine Processes**: The FlutterTextInputPlugin receives this callback and translates it into an action Flutter can understand.

4. **Framework Updates State**: The Engine sends a message back through Platform Channels to the Framework, which then updates your app's state. This ensures the Framework remains the **single source of truth** for what's happening in your app. (The waiter's order book must always be accurate!)

This completes the entire interaction loop - from customer order to kitchen preparation to serving the final dish!

---

## Practical Tips: How to Find These Files?

You don't need to memorize full filenames. You need to learn how to **deduce** and **search**.

### Deduction
- "My problem is related to text input on iOS..."
  - `iOS` → Maps to `.../platform/darwin/ios/...` in engine code directory
  - `Text Input` → Filename definitely contains `TextInput`
  - **Combine them:** Search for `TextInput` in the `ios` directory, and you'll immediately find `FlutterTextInputPlugin.mm`

### Search
**This is every developer's superpower!** (Seriously, half of programming is just knowing what to search for. The other half is copying from Stack Overflow - kidding! Sort of...)
- **Search by function:** Search for `"Look Up"` string in the entire Flutter Engine code, you'll find it in `FlutterTextInputPlugin.mm`, thus locating the "head text chef"
- **Search by channel name:** Search for `"ContextMenu.showSystemContextMenu"` message name in the code, you'll find both sender (`text_input.dart`) and receiver (`FlutterPlatformPlugin.mm`)

### Learn to Trace the Source

This is a more advanced technique. In your IDE, use "global search" on a method name (like `showSystemContextMenu`) to find related files.

You can jump from an Engine method step by step to see who calls it, tracing all the way back to the Framework layer. Or vice versa. This helps you build a complete view of the call chain.

## Deep Understanding

`FlutterTextInputPlugin.mm` is Flutter Engine's "diplomat" stationed on the iOS platform. While it works for Flutter Engine, it must use iOS native code as the "local language" to communicate effectively with the operating system. It's basically bilingual - speaking both Dart and Objective-C. (Show-off!)

Your app is built using Flutter's official Dart building blocks (Framework), and both the blocks and the model are made of Dart.

Just a fair warning, understanding this architecture takes time (think weeks, not hours), but once you get it, you'll be cooking up Flutter contributions like a master chef! So pour yourself some coffee, dive into the code, and enjoy the journey. Trust me, it gets easier after the first few PRs!

---

## Related Articles

Want to build your own Flutter Engine? Check out: [Your First Step to Contributing: Building Flutter Engine and Framework Development](/posts/2025/07/flutter-engine-guide/)

---

## License

This article is licensed under [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)

💡 **Found this helpful?** Follow me on [GitHub](https://github.com/jingshao-code) and share this article with your Flutter community!