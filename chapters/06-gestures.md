# 第6章：ジェスチャー操作

> 執筆者： 匡鈺海
> 最終更新：2026-07-29

## この章で学ぶこと

この章では、DragGestureを使ってユーザーのドラッグ操作を検出する方法を学ぶ。カードの移動距離に応じて位置、回転、文字の透明度を変化させ、アニメーションを組み合わせることで、Tinder風のスワイプUIを実装する。さらに、右スワイプを「好き」、左スワイプを「好きではない」として動物を分類する。

例：この章では、ユーザーの指の動きを検出するジェスチャー認識の方法を学ぶ。タップ・ロングプレス・ドラッグ・拡大縮小・回転など、各ジェスチャーの実装方法を学び、最終的にTinder風のスワイプUIで複数のジェスチャーを組み合わせた実装を題材にする。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第6章（応用）：Tinder風スワイプカードUI
// ============================================
// ドラッグジェスチャーとアニメーションを組み合わせて、
// カードを左右にスワイプして仕分けるUIを作ります。
// ============================================

import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、表示された動物のカードを左右にスワイプして、好きな動物と好きではない動物に分類するアプリです。右にスワイプすると「LIKE」、左にスワイプすると「NOPE」になり、それぞれの数が画面上部に表示されます。
カードは画面下部のハートボタンと×ボタンでも分類できます。すべてのカードを分類すると「完了！」と表示され、「もう一度」ボタンを押すと順番をランダムにして最初からやり直せます。
## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
// このコードでは、タップやロングプレス専用の
// TapGesture、LongPressGestureは使用していない。

Button {
    if let top = animals.last {
        removeCard(animal: top, direction: .left)
    }
} label: {
    Image(systemName: "xmark.circle.fill")
        .font(.system(size: 50))
        .foregroundStyle(.red)
}
```

**何をしているか：**

このアプリでは、タップ操作としてButtonを使用している。×ボタンをタップすると一番上の動物が「好きではない」に分類され、ハートボタンをタップすると「好き」に分類される。ロングプレス操作は実装されていない。

**なぜこう書くのか：**

単純なタップ操作にはButtonを使うと、処理の内容と画面表示を分かりやすく記述できる。また、アクセシビリティやタップ時の反応も標準で利用できるため、TapGestureよりボタンに適している。

**もしこう書かなかったら：**

手動操作用のボタンがなければ、カードを分類する方法がスワイプだけになる。スワイプ操作が難しい利用者は、動物を分類しにくくなる。なお、ロングプレスはこのアプリに必要な機能ではないため、省略しても現在の動作には影響しない。
---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
@State private var rotation: Double = 0

.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
            rotation = Double(value.translation.width / 20)
        }
        .onEnded { value in
            if value.translation.width > swipeThreshold {
                withAnimation(.easeOut(duration: 0.3)) {
                    offset = CGSize(width: 500, height: 0)
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                    onSwipe(.right)
                }
            } else if value.translation.width < -swipeThreshold {
                withAnimation(.easeOut(duration: 0.3)) {
                    offset = CGSize(width: -500, height: 0)
                }
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                    onSwipe(.left)
                }
            } else {
                withAnimation(.spring) {
                    offset = .zero
                    rotation = 0
                }
            }
        }
)
```

**何をしているか：**

DragGestureでカードをドラッグした距離を取得し、その値をoffsetに保存してカードを指の動きに合わせて移動させている。また、横方向の移動距離に応じてカードの角度も変えている。指を離したときは、移動距離によって右スワイプ、左スワイプ、元の位置に戻す処理を選択する。

**なぜこう書くのか：**

@Stateでoffsetとrotationを管理すると、値の変更がすぐに画面へ反映されるため、カードが指について動いているように見える。さらに、一定距離を超えた場合だけスワイプを成立させることで、少し触れただけで誤判定されることを防いでいる。

**もしこう書かなかったら：**

offsetを更新しなければ、指を動かしてもカードは画面上で移動しない。onEndedで判定しなければ、左右の分類を決められない。また、移動距離が足りないときに.zeroへ戻さなければ、カードが途中の位置に残ってしまう。

---

### 拡大縮小と回転

```swift
@State private var rotation: Double = 0

DragGesture()
    .onChanged { value in
        offset = value.translation
        rotation = Double(value.translation.width / 20)
    }
    .onEnded { value in
        if abs(value.translation.width) <= swipeThreshold {
            withAnimation(.spring) {
                offset = .zero
                rotation = 0
            }
        }
    }

.offset(offset)
.rotationEffect(.degrees(rotation))
```

**何をしているか：**

カードを横方向にドラッグした距離をもとにrotationを計算し、.rotationEffectでカードを回転させている。右へ動かすと一方向に、左へ動かすと反対方向に傾く。スワイプが成立しなかった場合は、回転角度を0に戻している。

**なぜこう書くのか：**

カードを移動させるだけでなく、ドラッグ方向に合わせて傾けることで、実際のカードを手で動かしているような自然な見た目になる。移動距離を20で割ることで、回転が大きくなりすぎないように調整している。

**もしこう書かなかったら：**

.rotationEffectがなければ、カードは傾かず平行に移動するだけになり、スワイプの動きが単調に見える。また、失敗したスワイプでrotationを0に戻さなければ、カードが傾いた状態で残る可能性がある。
※この模範コードでは、rotationEffectによる回転は実装されているが、scaleEffectやMagnifyGestureを使った拡大・縮小は実装されていない。

---

### ジェスチャーの組み合わせとアニメーション

```swift
DragGesture()
    .onChanged { value in
        offset = value.translation
        rotation = Double(value.translation.width / 20)
    }
    .onEnded { value in
        if value.translation.width > swipeThreshold {
            withAnimation(.easeOut(duration: 0.3)) {
                offset = CGSize(width: 500, height: 0)
            }

            DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                onSwipe(.right)
            }
        } else if value.translation.width < -swipeThreshold {
            withAnimation(.easeOut(duration: 0.3)) {
                offset = CGSize(width: -500, height: 0)
            }

            DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                onSwipe(.left)
            }
        } else {
            withAnimation(.spring) {
                offset = .zero
                rotation = 0
            }
        }
    }
```

**何をしているか：**

ドラッグ中はカードの位置と角度を同時に変化させている。指を離したとき、移動距離が100ポイントを超えていれば、カードを画面外まで移動させて分類する。距離が足りなければ、ばねのようなアニメーションで元の位置に戻す。

**なぜこう書くのか：**

.onChangedと.onEndedを組み合わせることで、ドラッグ中の表示と、操作終了後の処理を分けられる。スワイプ成功時には.easeOutを使用して自然に減速しながら退場させ、失敗時には.springを使用してカードが元の位置へ弾むように戻している。

**もしこう書かなかったら：**

アニメーションがなければカードの位置が突然変化し、操作が不自然に見える。また、カードが画面外へ移動する前に配列から削除すると、退場アニメーションが見えなくなる。そのため、asyncAfterで0.3秒待ってからonSwipeを実行している。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`DragGesture` | ドラッグジェスチャーを認識するジェスチャーレコグナイザー | `.gesture(DragGesture().onChanged { ... })` |
| 例：`MagnificationGesture` | ピンチジェスチャーで拡大・縮小を認識 | `.gesture(MagnificationGesture().onChanged { scale in ... })` |
| `DragGesture` | ドラッグ操作を認識するジェスチャー | `.gesture(DragGesture().onChanged { ... })` |
| `.onChanged` | ジェスチャー中に繰り返し処理を実行する | `.onChanged { value in offset = value.translation }` |
| `.onEnded` | ジェスチャーが終了したときに処理を実行する | `.onEnded { value in ... }` |
| `value.translation` | ドラッグを開始した位置からの移動距離を取得する | `offset = value.translation` |
| `.offset` | ビューの表示位置を移動させる | `.offset(offset)` |
| `.rotationEffect` | ビューを指定した角度だけ回転させる | `.rotationEffect(.degrees(rotation))` |
| `withAnimation` | 状態の変化にアニメーションを付ける | `withAnimation(.spring) { offset = .zero }` |
| `.spring` | ばねのように弾むアニメーションを作る | `withAnimation(.spring) { ... }` |
| `.easeOut` | 最後にゆっくり減速するアニメーションを作る | `.easeOut(duration: 0.3)` |
| `DispatchQueue.main.asyncAfter` | 指定した時間が経過した後に処理を実行する | `DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) { ... }` |
| `CGSize` | 横方向と縦方向の移動量をまとめて管理する | `CGSize(width: 500, height: 0)` |
| クロージャ | 子ビューから親ビューへ処理結果を渡す | `let onSwipe: (SwipeDirection) -> Void` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：
- 結果：
- わかったこと：

**実験2：**
- やったこと：
- 結果：
- わかったこと：

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
