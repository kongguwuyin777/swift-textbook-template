# 第7章：センサーの活用

> 執筆者：匡鈺海
> 最終更新：2026-07-29

## この章で学ぶこと

この章では、CoreMotionのCMPedometerを使って歩数と移動距離を取得し、CoreLocationを使って現在の移動速度を計測する方法を学ぶ。取得したセンサーデータから経過時間、移動距離、速度、消費カロリーを計算し、活動トラッカーとして画面に表示する。

## 模範コードの全体像

（教員から配布された模範コードをここに貼り付ける）

```swift
// ============================================
// 第7章（応用）：歩数計・移動距離トラッカー
// ============================================
// CoreMotion（歩数計）とCoreLocation（移動距離）を
// 組み合わせて、今日の活動を記録するアプリです。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSMotionUsageDescription
//     値: "歩数を計測するためにモーションセンサーを使用します"
//   - NSLocationWhenInUseUsageDescription
//     値: "移動距離を計測するために位置情報を使用します"
// ============================================

import SwiftUI
import CoreMotion
import CoreLocation

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    // MARK: - 計算プロパティ

    var elapsedTime: TimeInterval {
        guard let start = startTime else { return 0 }
        return Date().timeIntervalSince(start)
    }

    var distanceInKm: Double {
        distance / 1000
    }

    var speedInKmh: Double {
        currentSpeed * 3.6
    }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var tracker = ActivityTracker()
    @State private var timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // タイマー表示
                    timerSection

                    // メイン統計
                    statsGrid

                    // スタート/ストップボタン
                    controlButton

                    // 速度メーター
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
            .onReceive(timer) { _ in
                // タイマーの更新をトリガー（UI再描画のため）
                if tracker.isTracking {
                    // @Observableなので自動で更新される
                }
            }
        }
    }

    // MARK: - タイマーセクション

    private var timerSection: some View {
        VStack(spacing: 4) {
            Text(formatTime(tracker.elapsedTime))
                .font(.system(size: 48, weight: .thin, design: .monospaced))

            if tracker.isTracking {
                Text("計測中")
                    .font(.caption)
                    .foregroundStyle(.green)
            }
        }
        .padding()
    }

    // MARK: - 統計グリッド

    private var statsGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible()),
        ], spacing: 16) {
            StatCard(
                icon: "figure.walk",
                value: "\(tracker.stepCount)",
                unit: "歩",
                color: .blue
            )
            StatCard(
                icon: "map",
                value: String(format: "%.2f", tracker.distanceInKm),
                unit: "km",
                color: .green
            )
            StatCard(
                icon: "flame",
                value: String(format: "%.0f", tracker.caloriesBurned),
                unit: "kcal",
                color: .orange
            )
            StatCard(
                icon: "speedometer",
                value: String(format: "%.1f", tracker.speedInKmh),
                unit: "km/h",
                color: .purple
            )
        }
    }

    // MARK: - コントロールボタン

    private var controlButton: some View {
        Button {
            if tracker.isTracking {
                tracker.stopTracking()
            } else {
                tracker.startTracking()
            }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    // MARK: - 時間フォーマット

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

// MARK: - 統計カード

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)

            Text(value)
                .font(.title)
                .bold()

            Text(unit)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(color.opacity(0.08))
        )
    }
}

// MARK: - 速度メーター

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度")
                .font(.caption)
                .foregroundStyle(.secondary)

            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))

                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)

                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}

#Preview {
    ContentView()
}

```

**このアプリは何をするものか：**

このアプリは、歩数、移動距離、消費カロリー、現在の速度、経過時間を計測する活動トラッカーです。「スタート」ボタンを押すと計測が始まり、歩いた結果が画面上の統計カードに表示されます。
計測中は現在の速度がメーターで表示され、速度に応じて色が緑、オレンジ、赤に変化します。「ストップ」ボタンを押すと、歩数と位置情報の計測が終了します。

## コードの詳細解説

### CoreMotionの基本（CMMotionManager）

```swift
private let pedometer = CMPedometer()
var stepCount: Int = 0
var distance: Double = 0
var isPedometerAvailable: Bool = false

override init() {
    super.init()
    isPedometerAvailable = CMPedometer.isStepCountingAvailable()
}

if isPedometerAvailable {
    pedometer.startUpdates(from: Date()) { [weak self] data, error in
        guard let self = self, let data = data else { return }

        DispatchQueue.main.async {
            self.stepCount = data.numberOfSteps.intValue

            if let dist = data.distance {
                self.distance = dist.doubleValue
            }
        }
    }
}
```

**何をしているか：**

CMPedometerを使って、計測開始後の歩数と移動距離を取得している。最初に端末が歩数計測に対応しているか確認し、対応している場合だけstartUpdatesで計測を開始する。取得したデー
タはstepCountとdistanceへ保存する。）

**なぜこう書くのか：**

CMPedometerは歩数や歩行距離の取得に特化しているため、加速度データを自分で解析する必要がない。また、センサーからの結果はバックグラウンドスレッドで返される可能性があるため、画面に関係する値の更新をDispatchQueue.main.async内で行っている。[weak self]を使うことで、クロージャがActivityTrackerを強く保持し続けることも防いでいる。


**もしこう書かなかったら：**

利用可能か確認せずに計測を開始すると、歩数計測に対応していない端末で正しい結果を取得できない。メインスレッドを使わずに画面用の値を更新すると、UIの更新が不安定になる可能性がある。また、pedometer.stopUpdates()を呼ばなければ、ストップ後も歩数計の更新が続く可能性がある。

---

### デバイスの姿勢データ（pitch/roll/yaw）

```swift
// このコードでは、デバイスの姿勢データを取得する
// CMMotionManagerやCMDeviceMotionは使用していない。

func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations newLocations: [CLLocation]
) {
    guard let location = newLocations.last else { return }

    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

**何をしているか：**

このアプリでは、pitch、roll、yawなどの端末の姿勢は取得していない。代わりに、CLLocationManagerから最新の位置情報を受け取り、現在の速度と移動した場所の座標を記録している。

**なぜこう書くのか：**

このアプリの目的は歩数、移動距離、速度を計測することであり、端末の傾きや向きは計測に必要ないためである。max(0, location.speed)によって、位置情報から有効な速度を取得できなかった場合の負の値を0にしている。

**もしこう書かなかったら：**

位置情報の更新処理がなければ、現在の速度や移動した座標を記録できない。一方、姿勢データを追加しても、このアプリの歩数や移動距離の計測には直接影響しない。姿勢を取得する場合は、別途CMMotionManagerとstartDeviceMotionUpdatesを実装する必要がある。

---

### 歩数計（CMPedometer）

```swift
private let pedometer = CMPedometer()
var stepCount: Int = 0
var distance: Double = 0
var isPedometerAvailable: Bool = false

override init() {
    super.init()
    isPedometerAvailable = CMPedometer.isStepCountingAvailable()
}

if isPedometerAvailable {
    pedometer.startUpdates(from: Date()) { [weak self] data, error in
        guard let self = self, let data = data else { return }

        DispatchQueue.main.async {
            self.stepCount = data.numberOfSteps.intValue

            if let dist = data.distance {
                self.distance = dist.doubleValue
            }
        }
    }
}
```

**何をしているか：**

CMPedometerを使い、計測を開始した時点からの歩数と移動距離を取得している。取得した歩数はstepCount、距離はdistanceに保存し、画面の統計カードへ表示する。

**なぜこう書くのか：**

最初にisStepCountingAvailable()で端末が歩数計測に対応しているか確認することで、利用できる場合だけ計測を開始できる。[weak self]で循環参照を防ぎ、DispatchQueue.main.asyncでUIに関係するデータを安全に更新している。

**もしこう書かなかったら：**

CMPedometerを使用しなければ、歩数や歩行距離を自動で取得できない。利用可能か確認しなければ、非対応端末でデータを取得できない可能性がある。また、計測終了時にpedometer.stopUpdates()を呼ばなければ、ストップボタンを押した後も更新が続く可能性がある。

---

### CoreLocationとの連携

```swift
private let locationManager = CLLocationManager()
var currentSpeed: Double = 0
var locations: [CLLocationCoordinate2D] = []

override init() {
    super.init()
    locationManager.delegate = self
    locationManager.desiredAccuracy = kCLLocationAccuracyBest
    locationManager.requestWhenInUseAuthorization()
}

func startTracking() {
    locationManager.startUpdatingLocation()
}

func stopTracking() {
    locationManager.stopUpdatingLocation()
}

func locationManager(
    _ manager: CLLocationManager,
    didUpdateLocations newLocations: [CLLocation]
) {
    guard let location = newLocations.last else { return }

    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

**何をしているか：**

CLLocationManagerを使って位置情報の取得許可を求め、活動の計測中に現在地を取得している。新しい位置情報を受け取るたびに、現在の速度をcurrentSpeedへ保存し、移動した座標をlocations配列へ追加する。

**なぜこう書くのか：**

delegateを設定することで、位置情報が更新されたときにdidUpdateLocationsを呼び出せる。kCLLocationAccuracyBestを指定することで、移動速度や移動経路の計測に必要な精度を優先している。また、max(0, location.speed)によって、速度を取得できなかった場合の負の値を0として扱っている。

**もしこう書かなかったら：**

delegateを設定しなければ、更新された位置情報を受け取れない。利用許可を求めなければ、位置情報を取得できず、現在の速度や移動座標を記録できない。また、停止時にstopUpdatingLocation()を呼ばなければ、計測終了後も位置情報の取得が続き、バッテリーを余分に消費する可能性がある。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`CMMotionManager` | 加速度・ジャイロ・気圧などのセンサーデータを取得 | `motionManager.startDeviceMotionUpdates(to: .main) { ... }` |
| 例：`CMPedometer` | 歩数や歩行距離をカウント | `pedometer.queryPedometerData(from: startDate, to: Date())` |
| `CMPedometer` | 歩数や歩行距離を計測する | `pedometer.startUpdates(from: Date()) { data, error in ... }` |
| `isStepCountingAvailable()` | 端末が歩数計測に対応しているか確認する | `CMPedometer.isStepCountingAvailable()` |
| `stopUpdates()` | 歩数データの更新を停止する | `pedometer.stopUpdates()` |
| `CLLocationManager` | 位置、移動速度、座標などを取得する | `private let locationManager = CLLocationManager()` |
| `CLLocationManagerDelegate` | 位置情報の更新結果を受け取る | `func locationManager(_:didUpdateLocations:)` |
| `startUpdatingLocation()` | 位置情報の取得を開始する | `locationManager.startUpdatingLocation()` |
| `location.speed` | 現在の移動速度をm/sで取得する | `currentSpeed = max(0, location.speed)` |
| `@Observable` | オブジェクトの値の変化をSwiftUIへ反映する | `@Observable class ActivityTracker` |
| `[weak self]` | クロージャによるオブジェクトの強い保持を防ぐ | `{ [weak self] data, error in ... }` |
| `DispatchQueue.main.async` | メインスレッドで画面用の値を更新する | `DispatchQueue.main.async { self.stepCount = ... }` |
| `Timer.publish` | 一定間隔で処理を発生させるタイマーを作る | `Timer.publish(every: 1, on: .main, in: .common)` |
| `LazyVGrid` | ビューを格子状に並べる | `LazyVGrid(columns: columns) { ... }` |
| `.trim` | 円などの図形の一部分だけを表示する | `Circle().trim(from: 0, to: 0.75)` |
| | | |
| | | |
| | | |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
- やったこと：消費カロリーの計算式を、1歩あたり0.04 kcalから0.05 kcalへ変更した。
  var caloriesBurned: Double {
    Double(stepCount) * 0.05
}
- 結果：同じ歩数でも、変更前より消費カロリーが多く表示された。
- わかったこと：消費カロリーは歩数をもとにした簡易的な推定値であり、係数を変更すると計算結果も変わることがわかった。より正確にするには、体重や歩行速度なども考慮する必要がある。

**実験2：**
- やったこと：速度メーターの色が変わる基準を変更した。
  var speedColor: Color {
    if speed < 3 { return .green }
    if speed < 6 { return .orange }
    return .red
}
- 結果：元の設定より低い速度で、メーターの色が緑からオレンジ、オレンジから赤へ変わるようになった。
- わかったこと：条件分岐の数値を変更することで、利用目的に合わせて速度の評価基準を調整できることがわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：CMPedometerは、どのように歩数と移動距離を取得しているのか。**
   **得られた理解：startUpdatesを使うことで、指定した時刻からの歩数と歩行距離を継続的に取得できる。加速度センサーの値を自分で分析しなくても、numberOfStepsとdistanceから結果を利用できる。**

2. **質問：なぜDispatchQueue.main.asyncの中で歩数を更新するのか。**
   **得られた理解：CMPedometerの結果はメインスレッド以外で返される場合がある。画面に表示する値はメインスレッドで更新することで、UIを安全かつ正しく更新できる。**

3. **質問：CoreMotionとCoreLocationは、どのように役割を分けているのか。**
   **得られた理解：CoreMotionのCMPedometerは歩数と歩行距離を取得し、CoreLocationは現在の速度と移動座標を取得する。それぞれの得意なデータを組み合わせることで、より詳しい活動情報を表示できる。**

## この章のまとめ

この章では、CoreMotionのCMPedometerから歩数と移動距離を取得し、CoreLocationから現在の速度と位置座標を取得する方法を学んだ。計測の開始時には各データをリセットしてセンサーの更新を開始し、終了時には更新を停止することが重要である。
また、取得したデータをキロメートルや時速へ変換し、歩数から消費カロリーを推定できる。@Observableを使ってデータの変化を画面に反映し、タイマー、統計カード、速度メーターを組み合わせることで、活動状況を分かりやすく表示できる。



