# 第8章：ウィジェット

> 執筆者：匡鈺海
> 最終更新：2026-07-29

## この章で学ぶこと

この章では、WidgetKitを使ってiPhoneのホーム画面に表示できるウィジェットを作る方法を学ぶ。日替わりの名言を表示する機能を題材に、TimelineProviderによる更新予定の作成、ウィジェットのサイズに応じたレイアウト、メインアプリとのデータ共有について学ぶ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第8章：ウィジェットを作る
// ============================================
// 今日の名言をホーム画面に表示するウィジェットです。
// メインアプリとウィジェットの両方のコードを含みます。
//
// 【セットアップ手順】
// 1. Xcodeで File → New → Target → Widget Extension を選択
// 2. 「Include Configuration App Intent」のチェックを外す
// 3. Widget Extensionの名前を「QuoteWidget」にする
// 4. メインアプリとウィジェットで App Group を設定する
//    （Signing & Capabilities → App Groups）
// ============================================

// ============================================
// ■ メインアプリ側のコード（ContentView.swift）
// ============================================

import SwiftUI

// MARK: - 名言データ（アプリとウィジェットで共有）

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、是を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}


// ============================================
// ■ ウィジェット側のコード（QuoteWidget.swift）
// ============================================
// ※ Widget Extension ターゲット内のファイルに記述します。
// ※ QuoteStore は共有ファイルとして両ターゲットに追加するか、
//    同じコードをウィジェット側にもコピーしてください。
// ============================================

/*
import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
*/

```

**このアプリは何をするものか：**

このアプリは、毎日異なる名言を表示する「今日の名言」アプリです。メインアプリでは、その日の名言を大きく表示し、下の一覧から登録されているすべての名言と作者を確認できます。
ホーム画面にウィジェットを追加すると、アプリを開かなくても今日の名言を確認できます。ウィジェットは小サイズと中サイズに対応し、毎日0時を過ぎると新しい名言に更新されます。

## コードの詳細解説

### TimelineProviderの仕組み

```swift
struct QuoteProvider: TimelineProvider {
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    func getSnapshot(
        in context: Context,
        completion: @escaping (QuoteEntry) -> Void
    ) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    func getTimeline(
        in context: Context,
        completion: @escaping (Timeline<QuoteEntry>) -> Void
    ) {
        let currentDate = Date()
        let entry = QuoteEntry(
            date: currentDate,
            quote: QuoteStore.todaysQuote()
        )

        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(
                byAdding: .day,
                value: 1,
                to: currentDate
            )!
        )

        let timeline = Timeline(
            entries: [entry],
            policy: .after(tomorrow)
        )
        completion(timeline)
    }
}
```

**何をしているか：**

ウィジェットに表示するデータと更新予定を提供している。placeholderは読み込み中の仮表示、getSnapshotはウィジェットギャラリーなどのプレビュー表示、getTimelineは実際に表示する今日の名言と次回更新時刻を作成する。

**なぜこう書くのか：**

ウィジェットはアプリのように常に動作しているわけではないため、あらかじめ表示データと更新時刻をシステムへ渡す必要がある。.after(tomorrow)を指定することで、次の日の0時以降に新しい名言へ更新するようシステムへ依頼している。

**もしこう書かなかったら：**

TimelineProviderがなければ、ウィジェットは表示する名言や更新予定を取得できない。次回更新時刻を設定しなければ、日付が変わっても昨日の名言が残る可能性がある。また、completionを呼ばなければ、WidgetKitへデータが渡らず、ウィジェットを正しく表示できない。

---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry

    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }
}
```

**何をしているか：**

QuoteEntryは、ウィジェットに表示する日時と名言を1つのデータとして保持している。QuoteWidgetEntryViewはプロバイダから受け取ったエントリを使い、ウィジェットのサイズに応じて小サイズまたは中サイズのレイアウトを表示する。

**なぜこう書くのか：**

TimelineEntryに準拠することで、WidgetKitが「いつ、どのデータを表示するか」を管理できる。また、@Environment(\.widgetFamily)から現在のウィジェットサイズを取得することで、小サイズでは簡潔に、中サイズでは詳しく名言を表示できる。

**もしこう書かなかったら：**

TimelineEntryがなければ、TimelineProviderからウィジェットへ日時と名言を正しい形式で渡せない。また、サイズによる切り替えを行わなければ、すべてのサイズで同じレイアウトが使われ、文字が切れたり余白が不自然になったりする可能性がある。

---

### ウィジェットサイズごとのレイアウト

```swift
struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)
            Text(entry.quote.author)
                .font(.caption2)
        }
        .padding(12)
    }

    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()
                Text("— \(entry.quote.author)")
                    .font(.caption)
            }

            Spacer()
        }
        .padding()
    }
}
```

**何をしているか：**

@Environment(\.widgetFamily)で現在のウィジェットサイズを取得し、小サイズの場合はsmallWidget、中サイズの場合はmediumWidgetを表示している。小サイズでは縦方向に簡潔に並べ、中サイズでは横方向にアイコンと名言を配置している。

**なぜこう書くのか：**

ウィジェットはサイズによって利用できる表示領域が異なるためである。小サイズでは文字数と行数を制限し、中サイズでは広い横幅を使って、名言、作者、「今日の名言」という説明を読みやすく表示している。

**もしこう書かなかったら：**

すべてのサイズで同じレイアウトを使うと、小サイズでは文字が切れたり、内容が窮屈になったりする可能性がある。また、supportedFamiliesで許可したサイズに対応する表示を用意しなければ、見た目が不自然になる。

---

### メインアプリとの連携

```swift
struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(
            id: 1,
            text: "為せば成る、為さねば成らぬ何事も",
            author: "上杉鷹山"
        ),
        // その他の名言
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(
            of: .day,
            in: .year,
            for: Date()
        ) ?? 0

        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}


// メインアプリ側
let todaysQuote = QuoteStore.todaysQuote()

// ウィジェット側
let quote = QuoteStore.todaysQuote()
```

**何をしているか：**

メインアプリとウィジェットの両方から、共通のQuoteとQuoteStoreを利用している。どちらもtodaysQuote()を呼び出すことで、同じ日には同じ名言を表示する。

**なぜこう書くのか：**

共通のデータモデルと名言選択処理を共有することで、アプリ側とウィジェット側に同じ処理を重複して書かずに済む。また、名言の追加や選択方法の変更も、QuoteStoreの修正だけで両方へ反映できる。

**もしこう書かなかったら：**

アプリとウィジェットに別々の名言データや選択処理を書くと、内容が一致しなくなる可能性がある。また、修正時に両方のコードを変更する必要があり、管理が複雑になる。
※この模範コードでは、App Groupの設定手順は書かれているが、App GroupのUserDefaultsを使ったデータ保存はまだ実装されていない。現在はQuoteStoreの共有ファイルを両方のターゲットに追加することで連携している。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TimelineProvider` | ウィジェットを更新するタイミングとコンテンツを定義 | `struct QuoteProvider: TimelineProvider { ... }` |
| 例：`@main` + `WidgetConfiguration` | ウィジェットのエントリーポイント | `@main struct QuoteWidget: Widget { ... }` |
| `TimelineProvider` | ウィジェットに表示する内容と更新予定を提供する | `struct QuoteProvider: TimelineProvider { ... }` |
| `TimelineEntry` | 表示日時とウィジェット用データをまとめる | `struct QuoteEntry: TimelineEntry { let date: Date }` |
| `placeholder` | ウィジェットの読み込み中に表示する仮データを作る | `func placeholder(in context: Context) -> QuoteEntry` |
| `getSnapshot` | ウィジェットギャラリーなどで使う表示データを提供する | `func getSnapshot(in:completion:)` |
| `getTimeline` | 実際の表示データと次回更新予定を作成する | `func getTimeline(in:completion:)` |
| `Timeline` | エントリの一覧と更新方針をWidgetKitへ渡す | `Timeline(entries: [entry], policy: .after(tomorrow))` |
| `@Environment(\.widgetFamily)` | 現在のウィジェットサイズを取得する | `@Environment(\.widgetFamily) var family` |
| `StaticConfiguration` | ユーザー設定を必要としないウィジェットを定義する | `StaticConfiguration(kind: kind, provider: QuoteProvider())` |
| `supportedFamilies` | ウィジェットが対応するサイズを指定する | `.supportedFamilies([.systemSmall, .systemMedium])` |
| `containerBackground` | ウィジェット用の背景を設定する | `.containerBackground(.fill.tertiary, for: .widget)` |
| `@main`と`Widget` | ウィジェットの起動点となる型を定義する | `@main struct QuoteWidget: Widget { ... }` |
| `Codable` | データを保存・共有しやすい形式へ変換可能にする | `struct Quote: Identifiable, Codable` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：今日の名言を選ぶ計算に1を足した。
- let index = (dayOfYear + 1) % quotes.count
- 結果：同じ日でも、変更前とは異なる次の名言が表示された。
- わかったこと：日付から求めた数値を名言の件数で割った余りが、表示する名言の位置として使われていることがわかった。%を使うことで、配列の範囲を超えずに名言を繰り返し表示できる。

**実験2：**
- やったこと：対応するウィジェットサイズから.systemMediumを外した
- .supportedFamilies([.systemSmall])
- 結果：ウィジェット追加画面で、小サイズだけを選択できるようになった。
- わかったこと：supportedFamiliesで、利用者が選択できるウィジェットサイズを制限できることがわかった。中サイズにも対応するには、.systemMediumの指定と、それに適したレイアウトの両方が必要である。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：TimelineProviderのplaceholder、getSnapshot、getTimelineには、どのような違いがあるのか。**
   **得られた理解：placeholderは読み込み中の仮表示、getSnapshotはウィジェットギャラリーなどのプレビュー、getTimelineは実際の表示内容と更新予定を提供するために使われる。**

2. **質問：なぜTimelineで次の日の0時を指定するのか。**
   **得られた理解：日付が変わったときに新しい名言へ更新するためである。.after(tomorrow)は、次の日の0時以降にウィジェットの更新をシステムへ依頼する指定であり、正確な実行時刻はiOSが管理する。**

3. **質問：メインアプリとウィジェットは、どのように同じ名言を表示しているのか。**
   **得られた理解：両方のターゲットから共通のQuoteとQuoteStoreを利用し、同じtodaysQuote()を呼び出している。そのため、同じ日には同じ名言を表示できる。**

## この章のまとめ

この章では、WidgetKitを使ってホーム画面に日替わりの名言を表示する方法を学んだ。ウィジェットではTimelineEntryに表示データをまとめ、TimelineProviderから現在の内容と次回更新予定を提供する。
また、widgetFamilyを使って小・中サイズに適したレイアウトを切り替え、StaticConfigurationでウィジェットの名前、説明、対応サイズを設定できる。メインアプリとウィジェットでデータモデルや処理を共有することも重要である。
