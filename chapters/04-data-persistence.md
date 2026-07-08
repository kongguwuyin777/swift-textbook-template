# 第4章：データの永続化

> 執筆者：匡鈺海
> 最終更新：2026-06-10

## この章で学ぶこと

この章では、AppStorageとSwiftDataを使ったデータの永続化について学んだ。
メモアプリを作成し、SwiftDataによるメモの保存・編集・削除と、AppStorageによるユーザー設定の保存を実装した。
また、@Model、@Query、modelContextを利用したデータ管理の方法を理解した。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第4章：データの永続化（AppStorage + SwiftData）
// ============================================
// シンプルなメモアプリで、2つの永続化方法を学びます。
// - AppStorage：アプリ設定の保存
// - SwiftData：構造化データの保存
// ============================================

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - アプリのエントリポイント
// ※ @main のあるAppファイルに以下を記述してください：
//
// @main
// struct MemoApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: Memo.self)
//     }
// }

// MARK: - メインビュー

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button {
                        isShowingAddSheet = true
                    } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) {
                MemoAddView()
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            let memo = displayedMemos[index]
            modelContext.delete(memo)
        }
    }
}

// MARK: - メモの行

struct MemoRow: View {
    let memo: Memo

    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(memo.title)
                    .font(.headline)

                Text(memo.content)
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(2)

                Text(memo.createdAt, style: .date)
                    .font(.caption2)
                    .foregroundStyle(.tertiary)
            }

            Spacer()

            if memo.isFavorite {
                Image(systemName: "star.fill")
                    .foregroundStyle(.yellow)
            }
        }
        .padding(.vertical, 2)
    }
}

// MARK: - メモ追加画面

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") {
                    TextField("メモのタイトル", text: $title)
                }
                Section("内容") {
                    TextEditor(text: $content)
                        .frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        let memo = Memo(title: title, content: content)
                        modelContext.insert(memo)
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

// MARK: - メモ編集画面

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

```

**このアプリは何をするものか：**

このアプリは、メモを作成・保存・編集・削除できるメモ帳アプリである。
SwiftDataを利用してメモの内容を端末に保存し、アプリを再起動してもデータを保持できる。
また、AppStorageを利用してユーザー名や表示設定を保存できる。

## コードの詳細解説

### SwiftDataモデル（@Model）

```swift
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
```

**何をしているか：**

メモのデータ構造を定義している。タイトル、内容、作成日時、お気に入り状態を1つのデータとして管理し、SwiftDataで保存できるようにしている。

**なぜこう書くのか：**

@Modelを付けることで、このクラスをSwiftDataのデータモデルとして利用できる。メモに必要な情報をまとめて管理できるため、保存や取得が簡単になる。

**もしこう書かなかったら：**

@Modelを付けないとSwiftDataがデータモデルとして認識できず、メモを保存したり読み込んだりできなくなる。また、各プロパティがなければメモの情報を保持できない。

---

### データの追加・削除（modelContext）

```swift
@Environment(\.modelContext) private var modelContext

let memo = Memo(title: title, content: content)
modelContext.insert(memo)

func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        let memo = displayedMemos[index]
        modelContext.delete(memo)
    }
}
```

**何をしているか：**

modelContextを使ってSwiftDataのデータを追加・削除している。
新しいメモを保存するときはinsert()を使い、不要なメモを削除するときはdelete()を使う。

**なぜこう書くのか：**

SwiftDataではmodelContextがデータベースとの窓口となるためである。
データの追加や削除を簡単に行うことができ、画面の表示も自動的に更新される。

**もしこう書かなかったら：**

modelContextを使用しない場合、メモを保存したり削除したりすることができない。
そのため、アプリを再起動した後にデータが残らなくなる。

---

### @Queryによるデータ取得

```swift
@Query(sort: \Memo.createdAt, order: .reverse)
private var memos: [Memo]
```

**何をしているか：**

SwiftDataに保存されているMemoデータを取得している。
また、createdAtを基準に新しいメモが上に表示されるよう並び替えている。

**なぜこう書くのか：**

@Queryを使うことで保存されているデータを自動的に取得できるためである。
データが追加・削除・更新された場合も、画面が自動的に更新される。

**もしこう書かなかったら：**

保存されているメモを取得できなくなり、一覧に表示できない。
また、手動でデータを読み込む処理を書く必要があり、コードが複雑になる。

---

### @AppStorageによる設定保存

```swift
@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""
```

**何をしているか：**

AppStorageを利用してユーザー設定を保存している。
このアプリでは、ユーザー名と「お気に入りを上に表示」の設定を端末に保存し、アプリを再起動した後も保持できるようにしている。

**なぜこう書くのか：**

@AppStorageを使うことで、少ないコードで設定情報を保存・読み込みできるためである。
ユーザー設定のような簡単なデータを管理する場合に便利である。

**もしこう書かなかったら：**

アプリを終了すると設定内容が失われてしまう。
そのため、アプリを起動するたびにユーザー名や表示設定を再入力する必要がある。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| @Model       | SwiftDataでデータモデルを定義するためのマクロ            | `@Model class Memo { ... }`                                                |
| @Query       | SwiftDataからデータを取得し、自動で画面に反映するプロパティラッパー | `@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]` |
| @AppStorage  | ユーザー設定を端末に保存するためのプロパティラッパー             | `@AppStorage("userName") private var userName: String = ""`                |
| modelContext | SwiftDataのデータを追加・削除するためのコンテキスト         | `modelContext.insert(memo)`                                                |
| @Bindable    | SwiftDataのデータを画面と双方向に連携するための属性         | `@Bindable var memo: Memo`                                                 |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：メモのタイトルを「メモ帳」から「Myメモ」に変更した。
- 結果：画面上部のタイトルが「Myメモ」に変わった。
- わかったこと：navigationTitleを変更すると、画面のタイトルも変わることがわかった。

**実験2：**
- やったこと：お気に入りの星の色を.yellowから.redに変更した。
- 結果：お気に入りの星が赤色で表示された。
- わかったこと：foregroundStyleを変更すると、アイコンの色を変えられることがわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** @Stateと@Bindingの違いは何ですか。
   **得られた理解：**@Stateは自分のViewで状態を管理し、@Bindingは親Viewの状態を子Viewから変更するために使うと理解した。

2. **質問：** AppStorageとSwiftDataの違いは何ですか。
   **得られた理解：** AppStorageは簡単な設定の保存に使い、SwiftDataはメモなどの構造化されたデータの保存に使う

3. **質問：** @Queryは何のために使いますか。
   **得られた理解：** SwiftDataに保存されたデータを取得し、画面に表示するために使うと理解した。

## この章のまとめ
    この章では、@Stateや@Bindingによる状態管理、AppStorageとSwiftDataによるデータ保存を学んだ。特に、簡単な設定と構造化データでは保存方法が違うことが重要だと理解した。また、コードを実際に変更して試すことで、各機能の役割をより深く理解できた。
（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
