# 第5章：機能統合の実践

> 執筆者：匡鈺海
> 最終更新：2026-07-15

## この章で学ぶこと

（この章で扱うトピックの概要を2〜3行で書く。自分の言葉で。）

例：この章では、これまでに学んだカメラ・地図・データ保存の各機能を組み合わせて、「フォトマップ」アプリを実装する方法を学ぶ。具体的には撮影した写真をGPS位置情報と一緒に保存し、地図上に表示し、永続化したデータを検索・編集するアプリを題材にする。複数機能を統合するためのアーキテクチャ設計が重要になる。

## 模範コードの全体像

// ============================================
// 第5章：カメラ + 地図 + データ保存の統合アプリ
// ============================================
// 写真を撮影し、撮影場所を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
//   - NSCameraUsageDescription（実機の場合）
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}



**このアプリは何をするものか：**

このアプリは、写真と撮影場所を一緒に保存できる「フォトマップ」アプリです。
写真を選択して、タイトルやメモを入力すると、そのときの位置情報とともに記録できます。保存した記録は地図上のマーカーや一覧画面から確認できます。地図上の写真をタップすると、写真、メモ、保存日、撮影場所の詳細が表示されます。また、不要になった記録は一覧画面から削除できます。

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(
        title: String,
        memo: String = "",
        latitude: Double,
        longitude: Double,
        imageData: Data? = nil
    ) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(
            latitude: latitude,
            longitude: longitude
        )
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}
```

**何をしているか：**
写真のタイトル、メモ、緯度・経度、画像データ、作成日時を1件の記録として管理しています。また、保存した座標を地図で使える形式に変換し、画像データを画面に表示できる UIImage に変換しています。

**なぜこう書くのか：**
@Model を付けることで、SwiftDataを使って記録を端末内に保存できるからです。緯度と経度を Double 型で保存すると、SwiftDataで扱いやすく、必要なときに CLLocationCoordinate2D に変換できます。画像は未選択の場合もあるため、Data? にしています。

**もしこう書かなかったら：**
@Model がなければ、SwiftDataの保存対象として利用できません。また、緯度・経度や画像データを保存しなければ、アプリを終了した後に撮影場所や写真を再表示できなくなります。画像を必須の Data 型にすると、写真を選択していない記録を作成できません。

---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}
```

**何をしているか：**

画面下部に「マップ」と「一覧」の2つのタブを表示しています。「マップ」では記録した場所を地図上で確認でき、「一覧」では保存した記録をリスト形式で確認できます。

**なぜこう書くのか：**

地図から探す操作と一覧から探す操作では目的が異なるため、画面をタブで分けています。TabView を使うことで、利用者は画面下部のボタンから簡単に表示を切り替えられます。

**もしこう書かなかったら：**

TabView がなければ、地図画面と一覧画面を直接切り替えられません。別の画面へ移動するボタンなどを追加する必要があり、操作が分かりにくくなる可能性があります。
---

### カメラと位置情報の連携

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("写真を選択", systemImage: "photo")
}

.onChange(of: selectedItem) { _, newItem in
    Task {
        if let data = try? await newItem?.loadTransferable(type: Data.self) {
            selectedImageData = data

            if let uiImage = UIImage(data: data) {
                previewImage = Image(uiImage: uiImage)
            }
        }
    }
}

func saveRecord() {
    guard let location = locationManager.currentLocation else { return }

    let record = PhotoRecord(
        title: title,
        memo: memo,
        latitude: location.latitude,
        longitude: location.longitude,
        imageData: selectedImageData
    )

    modelContext.insert(record)
    dismiss()
}
```

**何をしているか：**

写真ライブラリから選択した画像をデータとして読み込み、画面にプレビュー表示しています。保存ボタンを押すと、画像、タイトル、メモ、現在地の緯度・経度を1件の記録として保存します。

**なぜこう書くのか：**

PhotosPicker を使うことで、SwiftUIから安全に写真を選択できます。また、画像と位置情報を同じ PhotoRecord に保存することで、どの写真がどの場所のものかを地図上で確認できます。guard を使い、位置情報が取得できていない状態で保存されることも防いでいます。

**もしこう書かなかったら：**

画像を読み込む処理がなければ、選択した写真を表示・保存できません。位置情報を一緒に保存しなければ、写真を地図上の正しい場所に表示できなくなります。また、位置情報を確認せず保存すると、座標のない不完全な記録が作られる可能性があります。
※このコードはカメラで直接撮影するのではなく、PhotosPicker を使って写真ライブラリから画像を選択する仕組みです。

---

### SwiftDataでの画像保存

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

func saveRecord() {
    guard let location = locationManager.currentLocation else { return }

    let record = PhotoRecord(
        title: title,
        memo: memo,
        latitude: location.latitude,
        longitude: location.longitude,
        imageData: selectedImageData
    )

    modelContext.insert(record)
    dismiss()
}
```

**何をしているか：**

選択した画像を Data 型にして、タイトルや位置情報などと一緒に PhotoRecord へ格納しています。その記録を modelContext.insert() でSwiftDataに保存します。表示するときは、保存された Data を UIImage に戻します。

**なぜこう書くのか：**

SwiftDataには Image や UIImage をそのまま保存できないため、保存可能な Data 型に変換する必要があります。また、画像を選択しない場合も考えられるため、imageData をオプショナルの Data? にしています。

**もしこう書かなかったら：**

画像を Data に変換しなければ、SwiftDataへ正しく保存できません。modelContext.insert() を行わなければ、作成した記録は一時的なものになり、画面を閉じたりアプリを再起動したりすると失われます。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`TabView` | 複数のビューをタブで切り替えるコンポーネント | `TabView { ... }.tabViewStyle(.page)` |
| 例：`CLLocationManager` | GPS位置情報を取得するAPIManager | `let location = manager.location?.coordinate` |

| `TabView` | 複数の画面をタブで切り替えるためのコンポーネント | `TabView { MapTab(); ListTab() }` |
| `CLLocationManager` | GPSを使って現在地を取得するAPI | `manager.startUpdatingLocation()` |
| `@Observable` | クラスの値の変化をSwiftUIの画面へ反映させる | `@Observable class LocationManager` |
| `@Model` | クラスをSwiftDataの保存対象にする | `@Model class PhotoRecord` |
| `@Query` | SwiftDataに保存されているデータを取得する | `@Query private var records: [PhotoRecord]` |
| `modelContext.insert()` | 新しいデータをSwiftDataへ保存する | `modelContext.insert(record)` |
| `modelContext.delete()` | SwiftDataからデータを削除する | `modelContext.delete(records[index])` |
| `PhotosPicker` | 写真ライブラリから画像を選択する | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| `loadTransferable` | 選択した写真を`Data`型として非同期で読み込む | `await newItem?.loadTransferable(type: Data.self)` |
| `Map` | SwiftUIの画面に地図を表示する | `Map(position: $cameraPosition) { ... }` |
| `Annotation` | 地図上の指定した座標に独自の表示を置く | `Annotation(record.title, coordinate: record.coordinate)` |
| `UserAnnotation` | 地図上に利用者の現在地を表示する | `UserAnnotation()` |
| `.sheet` | 現在の画面の上に別画面をモーダル表示する | `.sheet(isPresented: $isShowingAddSheet)` |
| `.onChange` | 指定した値が変化したときに処理を実行する | `.onChange(of: selectedItem) { ... }` |
| `guard` | 条件を満たさない場合に処理を早く終了する | `guard let location = locationManager.currentLocation else { return }` |
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
