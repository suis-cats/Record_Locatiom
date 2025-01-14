Qiitaの記事↓

https://qiita.com/suis-f/items/af800ea97919fb3c6e10



Qittaに載せた内容



SwiftUIを使って、ユーザーの位置情報を記録し、通った道をマップに青い線で表示するアプリを作る方法を解説します。iOS 17の新しい`Map` APIを活用します。


![0902-_3__4_.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3328398/c7f2a46a-ca9a-99f9-5182-466a8ec69607.gif)



## 1. プロジェクトのセットアップ

まず、Xcodeで新しいSwiftUIプロジェクトを作成しましょう。プロジェクト名は「TrackMyRoute」など、好きな名前でOKです。

### 1.1 位置情報の権限を設定する

位置情報を使用するには、`Info.plist`に位置情報の使用許可をリクエストするキーを追加する必要があります。

`Info.plist` に以下のキーを追加してください。

```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>アプリが使用中に位置情報を取得します。</string>
```

:bulb: `Info.plist`がない場合は、GUIからも追加できます。
![plist位置情報 (2).gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3328398/fb001408-1877-05ad-d2d1-3a6e81de1f77.gif)





これで、アプリが位置情報を取得するための許可をユーザーにリクエストできるようになります。

## 2. 位置情報を管理する `LocationManager` の作成

次に、位置情報を管理し、通った道を記録するための `LocationManager` クラスを作成します。

### 2.1 `LocationManager.swift` の作成

プロジェクト内の`ContentView.swift`などと同じ場所に、新しい `Swift` ファイルを作成し、`LocationManager.swift` と名前を付けます。以下のコードを追加します。

```swift
import Foundation
import CoreLocation

class LocationManager: NSObject, ObservableObject, CLLocationManagerDelegate {
    private var locationManager = CLLocationManager()
    
    @Published var location: CLLocation? = nil
    @Published var locationError: String? = nil
    @Published var authorizationStatus: CLAuthorizationStatus?
    @Published var locations: [CLLocationCoordinate2D] = []  // 位置情報の履歴を保持
    
    override init() {
        super.init()
        locationManager.delegate = self
        
        // 権限のリクエスト
        locationManager.requestWhenInUseAuthorization()
        
        // 現在の権限状態を初期化
        let status = locationManager.authorizationStatus
        self.authorizationStatus = status
        print("Location authorization status: \(status)")
        
        // 位置情報の取得を開始
        locationManager.startUpdatingLocation()
    }
    
    func locationManager(_ manager: CLLocationManager, didChangeAuthorization status: CLAuthorizationStatus) {
        DispatchQueue.main.async {
            self.authorizationStatus = status
            print("Authorization status changed: \(status.rawValue)")
            
            if status == .authorizedWhenInUse || status == .authorizedAlways {
                self.locationManager.startUpdatingLocation()
            } else {
                self.locationError = "位置情報の権限が許可されていません。設定を確認してください。"
            }
        }
    }
    
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        if let lastLocation = locations.last {
            DispatchQueue.main.async {
                self.location = lastLocation
                self.locationError = nil  // エラーメッセージをクリア
                self.locations.append(lastLocation.coordinate)  // 位置情報を記録
            }
            print("Updated location: \(lastLocation.coordinate.latitude), \(lastLocation.coordinate.longitude)")
        } else {
            print("No location data available")
        }
    }
    
    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        DispatchQueue.main.async {
            self.locationError = error.localizedDescription
        }
        
        if let clError = error as? CLError {
            switch clError.code {
            case .denied:
                print("Location access denied by the user")
                self.locationError = "位置情報へのアクセスが拒否されました。設定を確認してください。"
            case .locationUnknown:
                print("Location is currently unknown, but it may be available later")
                self.locationError = "現在の位置情報は取得できませんが、後ほど利用可能になるかもしれません。"
            default:
                print("Location Manager failed with error: \(error.localizedDescription)")
                self.locationError = "位置情報の取得に失敗しました。エラー: \(error.localizedDescription)"
            }
        } else {
            print("Failed to get location: \(error.localizedDescription)")
            self.locationError = "位置情報の取得に失敗しました。エラー: \(error.localizedDescription)"
        }
    }
}
```

このクラスは、ユーザーの位置情報を取得し、エラーをハンドリングしながら位置情報を記録します。

## 3. 位置情報を利用してマップを表示する

次に、位置情報を使用して地図上に通った道を描画する `ContentView` を作成します。

### 3.1 `ContentView.swift` の実装

`ContentView.swift` を以下のように修正します。

```swift
import SwiftUI
import CoreLocation
import MapKit

struct ContentView: View {
    
    @StateObject private var locationManager = LocationManager()
    @State var position: MapCameraPosition = .automatic
    
    var body: some View {
        VStack {
            if !locationManager.locations.isEmpty {
                Map(position: $position) {
                    // 通った道を描画
                    MapPolyline(coordinates: locationManager.locations)
                        .stroke(.blue, lineWidth: 5)
                }
                .mapControls {
                    MapUserLocationButton()
                }
                .onAppear {
                    if let firstLocation = locationManager.locations.first {
                        position = .region(MKCoordinateRegion(center: firstLocation, span: MKCoordinateSpan(latitudeDelta: 0.01, longitudeDelta: 0.01)))
                        print("Map region set to first location: \(firstLocation.latitude), \(firstLocation.longitude)")
                    }
                }
            } else if let error = locationManager.locationError {
                Text("位置情報の取得に失敗しました: \(error)")
                    .foregroundColor(.red)
                    .onAppear {
                        print("Error occurred: \(error)")
                    }
            } else {
                Text("位置情報を取得中...")
                    .onAppear {
                        print("Waiting for location data...")
                    }
            }
        }
        .padding()
    }
}

#Preview {
    ContentView()
}
```

このコードは、ユーザーの現在位置に基づいてマップを表示し、通った道を青色の線で描画します。

## 4. 最後に

これで、ユーザーの位置情報を記録し、通った道をマップに表示するアプリが完成しました。今回紹介した手法は、ランニングやサイクリングのアプリケーションでの利用にも応用可能です。

もしこの記事が役に立ったら、ぜひ「いいね」や「ストック」をお願いします。また、質問やフィードバックがあれば、コメント欄でお知らせください！


