### 프로젝트 소개

---

- 날씨 API를 호출하여, 현재 날씨와, 사용자가 원하는 위치를 입력받아 해당 지역의 날씨를 보여주는 어플리케이션 입니다.

### 프로젝트 참여자

---

| SwainYun | Howard |
| --- | --- |
| <img src="https://avatars.githubusercontent.com/u/99116619?v=4" width=200 height=200/> | <img src="https://github.com/Remaked-Swain/ios-weather-forecast/assets/63297236/6e1c3ea4-5cf3-481e-9da2-bcc372ba1d71" width=200 height=200 />|

### 프로젝트 동작 화면

---

![WeatherForecast](https://github.com/Remaked-Swain/ios-weather-forecast/assets/63297236/3136c747-ee2a-4c65-b46d-f4a1255f5141)


### 주요 학습 포인트

---

1. `URLSession` 을 사용해 범용성, 재사용성 높은 네트워킹 타입 만들어서 사용하기
2. `CoreLocation` 을 사용하여 현재 위치 가져오기
3. `CLGeocoder` 를 사용하여 상세 위치 가져오기 
4. `CollectionView` 사용법 숙달 
5. `Custom CollectionViewCell` , `Custom CollectionViewHeader` 사용하기 
6. `RefreshControl` 을 통한 새로고침 구현
7. `UIAlertController` 를 통하여 사용자 입력값 받기 
8. `NSCache` 를 사용한 이미지 캐싱 & 성능 개선 
9. `CoreGraphics` 를 활용하여 그래프 그리기 

### Trouble Shooting

---

- **날씨 정보 DTO 계층구조화**
    
    ```swift
    // 개선 전
    extension ForecastModel {
    		struct City: Codable {
    				let id: Int?
    				let name: String?
    				let coord: Coord?
    				let country: String?
    				let population, timezone, sunrise, sunset: Int?
    		}
    }
    
    // 개선 후
    struct ForecastCity: Codable {
    		let id: Int?
    		let name: String?
    		let coordinate: Coordinate?
    		let country: String?
    		let population, timezone, sunrise, sunset: Int?
    }
    ```
    
    ForecastModel 내에 City가 있는 계층구조를 위와 같이 표현하였다. 이 경우 코드의 Deep depth 문제, 그리고 각 모델 객체의 독립성 저하 문제를 지적받았다.
    
    ForecastModel이라는 최상위 모델 객체부터 점 표기법으로 접근하면서 depth가 깊어지는 문제에 대해서 충분히 이해하고 공감할 수 있었다.
    
    JSON으로 응답받은 데이터를 [Quicktype.io](http://Quicktype.io) 같은 파싱사이트를 사용해 DTO로 변환해본 결과 WeatherModel과 ForecastModel의 계층 내 몇몇의 모델 객체의 네이밍이 동일해 재선언 오류가 발생한 관계로 extension을 통해 최상위 모델에 Nested 시켜서 이를 해결했으나 모델 객체의 독립성 저하를 불러오는 결과를 낳았다.
    
    일일단위 최근 날씨 정보와 5일치 예보를 구분짓는 네이밍으로 수정해서 이름이 겹치지 않게 되자 extension으로 가둬둔 것도 풀 수 있었고 최종적으로는 각 객체를 독립적으로 사용할 수 있게 되었다.
    
- **에러 발생 지점에서는 throw로 에러를 던지고, 최종적으로 활용하는 곳에서 에러 핸들링하기**
    
    ```swift
    // 개선 전
    networkManager.downloadData(url: url) { [weak self] downloadedData in
    		guard let data = downloadedData else {
    				return
    		}
    
    		guard let weatherModel = try? JSONDecoder().decode(WeatherModel.self, from: data) else {
    				return
    		}
    }
    ```
    
    단순 `try?` 문으로 디코딩 실패 시 nil값이 반환된 채 진행되는 위와 같은 로직에서는 디코딩 실패의 원인을 파악하기 힘들고 더 나아가 어디서 어떤 에러 때문에 정상적으로 동작하지 못하는지 알아채기 힘들다.
    
    따라서 에러 발생 가능성이 있는 곳에서 최대한 확실히 에러를 인지할 수 있도록 `throw` 문을 통해 관련있는 코드로 에러를 전파하고 최종적으로 결과를 활용하는 곳에서 에러 발생에 대한 대처를 할 수 있게 해주는 것이 좋다.
    
    ```swift
    final class NetworkManager {
        enum NetworkingError: Error, CustomDebugStringConvertible {
            case unknown
            case taskingError
            case badClientRequest(statusCode: Int)
            case badServerResponse(statusCode: Int)
            case corruptedData
            
            var debugDescription: String {
                switch self {
                case .unknown: "알 수 없는 에러입니다."
                case .taskingError: "DataTask 작업 중 에러가 발생했습니다."
                case .badClientRequest(let statusCode): "클라이언트 측 에러입니다. Code: \(statusCode)"
                case .badServerResponse(let statusCode): "서버 측 에러입니다. Code: \(statusCode)"
                case .corruptedData: "손상된 데이터입니다."
                }
            }
        }
        
        // 생략...
        
        static func downloadData(url: URL, _ completionHandler: @escaping (Result<Data, NetworkingError>) -> Void) {
            var request = URLRequest(url: url)
            request.httpMethod = "GET"
            
            URLSession.shared.dataTask(with: request) { data, response, error in
                guard error == nil else {
                    return completionHandler(.failure(.taskingError))
                }
                
                guard let response = response as? HTTPURLResponse else { return completionHandler(.failure(.unknown)) }
                
                guard (200..<300).contains(response.statusCode) else {
                    return completionHandler(.failure(.badServerResponse(statusCode: response.statusCode)))
                }
                
                guard let data = data else {
                    return completionHandler(.failure(.corruptedData))
                }
                
                completionHandler(.success(data))
            }.resume()
        }
    }
    // -----------------------------------------------------------------------------
    extension WeatherForecastDataService {
        // 생략...
        
        private func downloadData(url: URL, serviceType: ServiceType) {
            NetworkManager.downloadData(url: url) { [weak self] result in
                switch result {
                case .success(let data):
                    let model = self?.decodeJSONToSwift(data, serviceType: serviceType)
                    DispatchQueue.main.async {
                        guard let self = self else { return }
                        self.model = model
                        self.delegate?.notifyModelDidUpdate(dataService: self, model: model)
                    }
                case .failure(let error): print(error)
                }
            }
        }
        
        // 생략...
        
        private func decodeJSONToSwift(_ data: Data?, serviceType: ServiceType) -> Decodable? {
            do {
                guard let data = data else { return nil }
                let model = try JSONDecoder().decode(serviceType.decodingType, from: data)
                return model
            } catch {
                guard let error = error as? DecodingError else { return nil }
                print("Failed to Decoding JSON: \(error)")
            }
            
            return nil
        }
    }
    ```
    
    발생한 에러 종류가 무엇인지 Completion Handler를 통해 내보내고, 다운로드 작업의 결과물을 활용해야하는 DataService 쪽에서 성공과 실패 여부에 따라 적절한 코드 진행을 할 수 있게 되었다. 따라서 어디서 에러가 발생했고 무엇이 원인인지 알기 쉬워졌다.
    
- **위치 확인 속도 개선**
    
    ```swift
    final class LocationManager: NSObject {
        private let locationManager: CLLocationManager = CLLocationManager()
        weak var delegate: LocationManagerDelegate?
        
        override init() {
            super.init()
            locationManager.desiredAccuracy = kCLLocationAccuracyKilometer
            locationManager.distanceFilter = kCLDistanceFilterNone
            locationManager.requestWhenInUseAuthorization()
            locationManager.delegate = self
        }
    }
    ```
    
    날씨 정보와 CLPlacemark를 받아오는 과정이 정상적으로 동작하는지 실제 기기로 테스트한 결과, 화면의 Label에 주소가 나타나기 까지 약 8초가 소요되는 것으로 확인되었다.
    
    CLLocationManager의 정확도를 Best에서 Kilometer급으로 낮추니 약 1초대 까지 속도를 개선할 수 있었다.
    
    요청한 정확도 수준에 맞는 위치 정보 구성을 위하여 디바이스는 가용할 수 있는 자원을 모두 사용할텐데, 용도에 맞는 정확도 수준으로 요청하는 것도 중요함을 깨달을 수 있었다.
    

### 프로젝트 후기

---

- Swain Yun
    - 지금까지 배운 모든 지식을 총동원해서, 또 그리고 새로운 것도 계속 시도 적용해야 겨우 완성할 수 있는 프로젝트였다.
    - 새롭고 신기한 개념을 배우면서도 이미 배웠던 내용을 까먹지 않게 다시 복습할 수 있었던 재밌는 경험.
- Howard
    - 새롭고 중요한 내용을 많이 배우게 된 프로젝트였습니다. 특히 이번 프로젝트를 진행하면서, Protocol을 활용한 추상화 작업과, 범용성이 높은 모델들을 만드는 것에 신경을 많이 썼고, NetworkManager, CacheManager, LocationManager 등을 잘 만들어서 다양한 상황에서 활용할 수 있게 만든것 같습니다.
    - 문법적인 부분에서 가장 크게 배운것은 `escaping closure` 의 사용과 `custom delegate` 의 사용인것 같습니다. API 호출 결과에 따른 순차적인 코드 실행을 시키는데 많은 도움이 된 것 같습니다.

### 프로젝트 진행 참고 자료

---

- 공식문서
    - [UICollectionView - 공식문서](https://developer.apple.com/documentation/uikit/uicollectionview)
    - [API Collection - 공식문서](https://developer.apple.com/documentation/uikit/views_and_controls/collection_views)
    - [Collection View Programming Guide for IOS](https://developer.apple.com/documentation/uikit/views_and_controls/collection_views)
    - [Core Location Framework - 공식문서](https://developer.apple.com/documentation/foundation/runloop)
    - [CLLocationManager Class - 공식문서](https://developer.apple.com/documentation/corelocation/cllocationmanager#overview)
    - [NSCache - 공식문서](https://developer.apple.com/documentation/foundation/nscache)
    - [UIView - 공식문서](https://developer.apple.com/documentation/uikit/uiview)
    - [draw 메소드 - 공식문서](https://developer.apple.com/documentation/uikit/uiview/1622529-draw)
    - [Drawing and Printing Guide for IOS - Archive](https://developer.apple.com/library/archive/documentation/2DDrawing/Conceptual/DrawingPrintingiOS/GraphicsDrawingOverview/GraphicsDrawingOverview.html)
    - [Quartz 2D Programming Guide - Archive](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html#//apple_ref/doc/uid/TP40007533-SW1)
- WWDC
    - [2018 WWDC - High Performance Auto Layout](https://www.youtube.com/watch?v=hSWKCGLheeE)
