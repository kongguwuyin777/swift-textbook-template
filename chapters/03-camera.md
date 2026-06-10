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

このアプリは、選択した写真にフィルターをかけて楽しむアプリである。
セピアやモノクロなどの効果を適用し、加工した写真を保存することができる。


## コードの詳細解説

### PhotosPickerによる写真選択

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
Label("写真を選ぶ", systemImage: "photo")
}
.buttonStyle(.bordered)

.onChange(of: selectedItem) { _, newItem in
Task {
await loadOriginalImage(from: newItem)
}
}

```

**何をしているか：**

PhotosPickerを使用してフォトライブラリから写真を選択する機能を実装している。
写真が選択されると、onChangeが検知し、loadOriginalImage()を呼び出して画像を読み込み、画面に表示する。


**なぜこう書くのか：**

PhotosPickerはSwiftUIで写真選択を簡単に実装できるため使用している。
また、onChangeを利用することで、ユーザーが新しい写真を選択したタイミングで自動的に画像を読み込むことができる。


**もしこう書かなかったら：**

PhotosPickerを使用しなければ、ユーザーは写真を選択できなくなる。
また、onChangeがなければ写真を選択しても画像の読み込み処理が実行されず、画面に表示されない。


---

### 画像の非同期読み込み

```swift
.onChange(of: selectedItem) { _, newItem in
Task {
await loadOriginalImage(from: newItem)
}
}

func loadOriginalImage(from item: PhotosPickerItem?) async {
guard let item = item else { return }

```
do {
    if let data = try await item.loadTransferable(type: Data.self),
       let uiImage = UIImage(data: data) {
        originalUIImage = uiImage
        currentFilter = .original
        displayImage = Image(uiImage: uiImage)
    }
} catch {
    print("画像読み込みエラー: \(error)")
}
```

}

```

**何をしているか：**

選択された画像を非同期で読み込み、UIImageに変換して画面へ表示している。
読み込み完了後に状態を更新することで、ユーザーは画像を確認しながらフィルター加工を行うことができる。


**なぜこう書くのか：**

画像データの読み込みには時間がかかる場合があるため、async/awaitを利用して非同期処理として実行している。
これにより、画像の読み込み中でもアプリの画面が固まらず、快適に操作できる。

**もしこう書かなかったら：**

非同期処理を使用しない場合、大きな画像を読み込む際に画面が一時的に停止する可能性がある。
その結果、操作性が低下し、ユーザー体験が悪くなる。


---

### UIViewControllerRepresentableによるカメラ連携

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

### Coordinatorパターン

```swift
// 該当部分のコードを抜粋して貼る
```

**何をしているか：**

**なぜこう書くのか：**

**もしこう書かなかったら：**

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`PhotosPicker` | フォトライブラリから画像を選択するコンポーネント | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| 例：`UIImagePickerController` | カメラまたはフォトライブラリにアクセスするUIKitコンポーネント | `picker.sourceType = .camera` |
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
