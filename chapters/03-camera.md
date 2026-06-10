# 第3章：カメラの利用

> 執筆者：匡鈺海
> 最終更新：2026-05-27

## この章で学ぶこと

この章では、写真を選んでフィルター加工を行うアプリを作成した。
CoreImageを利用した画像加工や、加工後の写真を保存する方法を学んだ。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、
// 画面に表示します。シミュレータでも動作します。
// ============================================

import SwiftUI
import PhotosUI

// MARK: - メインビュー

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア

    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み

    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、フォトライブラリから写真を選択したり、カメラで撮影した写真を表示したりするアプリである。
ユーザーは好きな写真を簡単に確認することができる。



## コードの詳細解説

### PhotosPickerによる写真選択

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)

.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}

```

**何をしているか：**

PhotosPickerを使ってフォトライブラリから写真を選択し、選択された画像を画面に表示している。

**なぜこう書くのか：**

PhotosPickerはSwiftUIで簡単にフォトライブラリへアクセスできるため使用している。また、ユーザーが写真を選択したタイミングで自動的に画像を読み込むことができる。


**もしこう書かなかったら：**

ユーザーがフォトライブラリから写真を選択できなくなる。また、写真を選択しても画像が読み込まれず、画面に表示されない。

---

### 画像の非同期読み込み

```.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}

func loadImage(from item: PhotosPickerItem?) async {
    guard let item = item else { return }

    do {
        if let data = try await item.loadTransferable(type: Data.self),
           let uiImage = UIImage(data: data) {
            selectedImage = Image(uiImage: uiImage)
        }
    } catch {
        print("画像の読み込みに失敗: \(error.localizedDescription)")
    }
}

```

**何をしているか：**

選択された写真を非同期で読み込み、UIImageに変換して画面に表示している。


**なぜこう書くのか：**

画像の読み込みには時間がかかる場合があるため、async/awaitを使って画面が固まらないようにしている。

**もしこう書かなかったら：**

大きな画像を読み込むときに画面が止まったり、写真を選択しても正しく表示されない可能性がある。

---

### UIViewControllerRepresentableによるカメラ連携

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }
}
```

**何をしているか：**

SwiftUIからUIKitのUIImagePickerControllerを使えるようにし、カメラで写真を撮影できるようにしている。

**なぜこう書くのか：**

カメラ機能はUIKitのUIImagePickerControllerを使うため、UIViewControllerRepresentableでSwiftUIに対応させている。

**もしこう書かなかったら：**

SwiftUIの画面からカメラを起動できず、写真を撮影して表示する機能が使えなくなる。

---

### Coordinatorパターン

```swift
func makeCoordinator() -> Coordinator {
    Coordinator(self)
}

class Coordinator: NSObject,
                   UIImagePickerControllerDelegate,
                   UINavigationControllerDelegate {

    let parent: CameraView

    init(_ parent: CameraView) {
        self.parent = parent
    }

    func imagePickerController(
        _ picker: UIImagePickerController,
        didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey : Any]
    ) {
        if let image = info[.originalImage] as? UIImage {
            parent.capturedImage = image
        }
        parent.dismiss()
    }

    func imagePickerControllerDidCancel(
        _ picker: UIImagePickerController
    ) {
        parent.dismiss()
    }
}
```

**何をしているか：**

Coordinatorを使ってUIImagePickerControllerのイベントを受け取り、撮影した写真をSwiftUI側へ渡している。また、撮影完了時やキャンセル時にカメラ画面を閉じる処理も行っている。

**なぜこう書くのか：**

UIImagePickerControllerはUIKitのDelegate方式で動作するため、SwiftUIと連携するにはCoordinatorが必要である。Coordinatorを介することで、撮影結果をSwiftUIの状態変数へ安全に反映できる。

**もしこう書かなかったら：**

撮影した写真を取得できなくなり、SwiftUIの画面へ表示できない。また、撮影後やキャンセル後にカメラ画面を閉じることもできなくなる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 項目                                | 説明                                         | 使用例                                                            |
| --------------------------------- | ------------------------------------------ | -------------------------------------------------------------- |
| `UIViewControllerRepresentable`   | SwiftUIからUIKitのViewControllerを利用するためのプロトコル | `struct CameraView: UIViewControllerRepresentable`             |
| `@Binding`                        | 親ビューと子ビューでデータを共有する仕組み                      | `@Binding var capturedImage: UIImage?`                         |
| `Task`                            | 非同期処理を開始するために使用する                          | `Task { await loadImage(from: newItem) }`                      |
| `async / await`                   | 非同期処理をわかりやすく記述する構文                         | `func loadImage(...) async`                                    |
| `fullScreenCover`                 | 画面全体を覆うモーダル画面を表示する                         | `.fullScreenCover(isPresented: $isShowingCamera)`              |
| `onChange`                        | 値の変化を監視して処理を実行する                           | `.onChange(of: selectedItem)`                                  |
| `UIImagePickerControllerDelegate` | カメラ撮影結果を受け取るDelegate                       | `class Coordinator: NSObject, UIImagePickerControllerDelegate` |
| `Coordinator`                     | UIKitのDelegateとSwiftUIを連携するためのクラス          | `func makeCoordinator() -> Coordinator`                        |

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
