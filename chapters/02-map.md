# 第2章：地図アプリの基本

> 執筆者：匡鈺海
> 最終更新：2026-05-27

## この章で学ぶこと

この章では、MapKitを使って地図を表示し、観光スポットをマーカーで表示する方法を学ぶ。
また、カテゴリごとのフィルター機能や、地図上の情報をSwiftUIで見やすく表示する方法についても理解を深める。

## 模範コードの全体像



```swift
// ============================================
// 第2章（基本）：MapKitで地図を表示するアプリ
// ============================================
// 東京の観光スポットを地図上にマーカーで表示します。
// マーカーをタップすると詳細情報が表示されます。
// ============================================

import SwiftUI
import MapKit

// MARK: - データモデル

struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}

// MARK: - サンプルデータ

extension Landmark {
    static let sampleData: [Landmark] = [
        Landmark(
            name: "浅草寺",
            description: "東京都内最古の寺院。雷門が有名。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7148, longitude: 139.7967),
            category: .temple
        ),
        Landmark(
            name: "東京タワー",
            description: "1958年に完成した高さ333mの電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
            category: .tower
        ),
        Landmark(
            name: "東京スカイツリー",
            description: "高さ634mの世界一高い自立式電波塔。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7101, longitude: 139.8107),
            category: .tower
        ),
        Landmark(
            name: "明治神宮",
            description: "明治天皇と昭憲皇太后を祀る神社。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6764, longitude: 139.6993),
            category: .temple
        ),
        Landmark(
            name: "上野恩賜公園",
            description: "美術館や動物園がある広大な公園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.7146, longitude: 139.7732),
            category: .park
        ),
        Landmark(
            name: "新宿御苑",
            description: "都心にある広さ58.3ヘクタールの庭園。",
            coordinate: CLLocationCoordinate2D(latitude: 35.6852, longitude: 139.7100),
            category: .park
        ),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var cameraPosition: MapCameraPosition = .region(
        MKCoordinateRegion(
            center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
            span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
        )
    )
    @State private var selectedLandmark: Landmark?
    @State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

    var filteredLandmarks: [Landmark] {
        Landmark.sampleData.filter { selectedCategories.contains($0.category) }
    }

    var body: some View {
        ZStack(alignment: .bottom) {
            // 地図
            Map(position: $cameraPosition) {
                ForEach(filteredLandmarks) { landmark in
                    Marker(
                        landmark.name,
                        systemImage: landmark.category.iconName,
                        coordinate: landmark.coordinate
                    )
                    .tint(landmark.category.color)
                }
            }
            .mapStyle(.standard(elevation: .realistic))

            // カテゴリフィルター
            VStack(spacing: 8) {
                if let landmark = selectedLandmark {
                    LandmarkCard(landmark: landmark)
                        .transition(.move(edge: .bottom))
                }

                CategoryFilter(selectedCategories: $selectedCategories)
            }
            .padding()
        }
        .onMapCameraChange { context in
            // 地図の操作に応じた処理を追加できる
        }
    }
}

// MARK: - カテゴリフィルター

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                    .background(
                        selectedCategories.contains(category)
                            ? category.color.opacity(0.2)
                            : Color.gray.opacity(0.1)
                    )
                    .foregroundStyle(
                        selectedCategories.contains(category)
                            ? category.color
                            : .gray
                    )
                    .clipShape(Capsule())
                }
            }
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 16))
    }
}

// MARK: - ランドマーク詳細カード

struct LandmarkCard: View {
    let landmark: Landmark

    var body: some View {
        VStack(alignment: .leading, spacing: 6) {
            HStack {
                Image(systemName: landmark.category.iconName)
                    .foregroundStyle(landmark.category.color)
                Text(landmark.name)
                    .font(.headline)
                Spacer()
            }
            Text(landmark.description)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .background(.ultraThinMaterial)
        .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、MapKitを使って東京の観光スポットを地図上に表示するアプリである。
寺社・タワー・公園などのカテゴリごとにマーカーを表示し、フィルター機能によって表示内容を切り替えることができる。
また、スポットの名前や説明をカード形式で確認でき、地図アプリの基本的な使い方を学べる。

## コードの詳細解説

### データモデル（ランドマーク構造体）

```swift
// MARK: - データモデル

struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let description: String
    let coordinate: CLLocationCoordinate2D
    let category: Category

    enum Category: String, CaseIterable {
        case temple = "寺社"
        case tower = "タワー"
        case park = "公園"

        var iconName: String {
            switch self {
            case .temple: return "building.columns"
            case .tower: return "antenna.radiowaves.left.and.right"
            case .park: return "leaf"
            }
        }

        var color: Color {
            switch self {
            case .temple: return .red
            case .tower: return .blue
            case .park: return .green
            }
        }
    }
}
```

**何をしているか：**

この部分では、地図上に表示する観光スポットのデータを定義している。
Landmark構造体には、スポット名、説明、位置情報、カテゴリなどが保存されている。
また、Categoryという列挙型を使って、「寺社」「タワー」「公園」の種類を管理している。


**なぜこう書くのか：**

LandmarkをIdentifiableにすることで、SwiftUIのListやMap上で各データを区別しやすくなる。
また、カテゴリごとにアイコンや色を設定することで、地図上で視覚的に分かりやすく表示できる。
enumを使うことで、カテゴリの種類を安全かつ整理された形で管理できる。


**もしこう書かなかったら：**

Identifiableがない場合、SwiftUIが各データを正しく識別できず、ForEachなどで扱いにくくなる。
また、カテゴリを文字列だけで管理すると、タイプミスや不正な値が発生しやすくなる。
さらに、色やアイコンをカテゴリごとに分けていないと、地図上でスポットの種類を判別しにくくなる。


---

### 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .region(
    MKCoordinateRegion(
        center: CLLocationCoordinate2D(latitude: 35.6812, longitude: 139.7671),
        span: MKCoordinateSpan(latitudeDelta: 0.08, longitudeDelta: 0.08)
    )
)

var body: some View {
    Map(position: $cameraPosition) {
        ForEach(filteredLandmarks) { landmark in
            Marker(
                landmark.name,
                systemImage: landmark.category.iconName,
                coordinate: landmark.coordinate
            )
            .tint(landmark.category.color)
        }
    }
    .mapStyle(.standard(elevation: .realistic))
}
```

**何をしているか：**

この部分では、MapKitを使って地図を表示し、観光スポットをマーカーとして地図上に配置している。
また、cameraPositionを使って、最初に表示する地図の中心位置や拡大範囲を設定している。
ForEachによって、filteredLandmarksのデータを繰り返し表示し、それぞれのスポットをマーカーとして表示している。

**なぜこう書くのか：**

Map(position:)を使うことで、地図の表示位置やズーム状態をSwiftUI側で管理できる。
また、Markerを使うことで、観光スポットを分かりやすく地図上へ表示できる。
カテゴリごとにアイコンや色を変えることで、ユーザーがスポットの種類を直感的に理解しやすくなる。

**もしこう書かなかったら：**

cameraPositionを設定しない場合、地図の初期位置が分かりにくくなる可能性がある。
また、Markerを使わないと、地図上に観光スポットを表示できない。
さらに、カテゴリごとに色やアイコンを分けなかった場合、どのスポットが寺社・タワー・公園なのか判別しにくくなる。

---

### マーカーの表示

```swift
Map(position: $cameraPosition) {
    ForEach(filteredLandmarks) { landmark in
        Marker(
            landmark.name,
            systemImage: landmark.category.iconName,
            coordinate: landmark.coordinate
        )
        .tint(landmark.category.color)
    }
}
```

**何をしているか：**

この部分では、地図上に観光スポットのマーカーを表示している。
ForEachを使って観光スポットのデータを繰り返し処理し、それぞれの場所にMarkerを配置している。
また、カテゴリごとに異なるアイコンや色を設定している。

**なぜこう書くのか：**

ForEachを使うことで、複数の観光スポットを効率よく地図上へ表示できる。
Markerを使うことで、位置情報を視覚的に分かりやすく表示できる。
さらに、カテゴリごとにアイコンや色を変更することで、ユーザーがスポットの種類を直感的に判断しやすくなる。

**もしこう書かなかったら：**

ForEachを使わない場合、マーカーを一つずつ手動で書く必要があり、データ数が増えると管理が難しくなる。
また、Markerを使わなければ、観光スポットの位置を地図上に表示できない。
さらに、色やアイコンを設定しない場合、すべて同じ見た目になり、スポットの種類が分かりにくくなる。

---

### フィルター機能

```swift
@State private var selectedCategories: Set<Landmark.Category> = Set(Landmark.Category.allCases)

var filteredLandmarks: [Landmark] {
    Landmark.sampleData.filter { selectedCategories.contains($0.category) }
}

struct CategoryFilter: View {
    @Binding var selectedCategories: Set<Landmark.Category>

    var body: some View {
        HStack(spacing: 8) {
            ForEach(Landmark.Category.allCases, id: \.self) { category in
                Button {
                    if selectedCategories.contains(category) {
                        selectedCategories.remove(category)
                    } else {
                        selectedCategories.insert(category)
                    }
                } label: {
                    HStack(spacing: 4) {
                        Image(systemName: category.iconName)
                        Text(category.rawValue)
                    }
                    .font(.caption)
                    .padding(.horizontal, 10)
                    .padding(.vertical, 6)
                }
            }
        }
    }
}
```

**何をしているか：**

この部分では、観光スポットをカテゴリごとに絞り込むフィルター機能を実装している。
selectedCategoriesで現在選択されているカテゴリを管理し、filteredLandmarksで選択されたカテゴリのみを表示している。
また、Buttonを押すことでカテゴリの追加・削除を切り替えられるようになっている。

**なぜこう書くのか：**

Setを使うことで、カテゴリの重複を防ぎながら効率よく管理できる。
filterを使うことで、条件に一致する観光スポットだけを簡単に抽出できる。
さらに、カテゴリごとにボタンを作成することで、ユーザーが表示内容を自由に切り替えられるようになる。

**もしこう書かなかったら：**

フィルター機能がない場合、すべての観光スポットが常に表示され、地図が見づらくなる可能性がある。
また、selectedCategoriesを管理しないと、どのカテゴリが選択されているか判断できなくなる。
さらに、filterを使わなかった場合、表示データを手動で管理する必要があり、コードが複雑になってしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| ------------------------ | --------------------------- | ---------------------------------------------------------- |
| `Map`                    | SwiftUIで地図を表示するためのコンポーネント   | `Map(position: $cameraPosition)`                           |
| `Marker`                 | 地図上に位置情報を表示するためのコンポーネント     | `Marker("東京タワー", coordinate: coordinate)`                  |
| `MapCameraPosition`      | 地図の表示位置やズーム範囲を管理する型         | `@State private var cameraPosition`                        |
| `MKCoordinateRegion`     | 地図の中心座標と表示範囲を設定する構造体        | `MKCoordinateRegion(center: ..., span: ...)`               |
| `CLLocationCoordinate2D` | 緯度・経度を表す位置情報の型              | `CLLocationCoordinate2D(latitude: 35.6, longitude: 139.7)` |
| `ForEach`                | 複数のデータを繰り返し表示するための構文        | `ForEach(filteredLandmarks) { landmark in ... }`           |
| `Set`                    | 重複しないデータを管理するコレクション型        | `Set(Landmark.Category.allCases)`                          |
| `filter`                 | 条件に一致するデータだけを取り出すメソッド       | `sampleData.filter { ... }`                                |
| `@Binding`               | 親Viewと子Viewで状態を共有するための仕組み   | `@Binding var selectedCategories`                          |
| `Identifiable`           | SwiftUIでデータを一意に識別するためのプロトコル | `struct Landmark: Identifiable`                            |
| `enum`                   | 関連する値をまとめて管理する型             | `enum Category: String, CaseIterable`                      |
| `CaseIterable`           | enumの全ケースを取得できるプロトコル        | `Category.allCases`                                        |
| `mapStyle`               | 地図の表示スタイルを変更する修飾子           | `.mapStyle(.standard(elevation: .realistic))`              |
| `@State`                 | View内で状態を保持するためのプロパティラッパー   | `@State private var selectedLandmark`                      |


## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：MKCoordinateSpan の latitudeDelta と longitudeDelta の値を変更して、地図の拡大率を変えてみた。
- 結果：値を小さくすると地図が拡大表示され、大きくすると広い範囲が表示された。
- わかったこと：MKCoordinateSpan を変更することで、地図のズーム範囲を調整できることが分かった。

**実験2：**
- やったこと：フィルター機能で「寺社」だけを選択し、他のカテゴリを非表示にしてみた。
- 結果：地図上には寺社カテゴリのマーカーだけが表示された。
- わかったこと：filter と Set を使うことで、条件に一致するデータだけを簡単に表示できることが分かった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   **得られた理解：**

2. **質問：**
   **得られた理解：**

3. **質問：**
   **得られた理解：**

## この章のまとめ

（この章で学んだ最も重要なことを、未来の自分が読み返したときに役立つように書く）
