# TDD 9장

9장에서는 TDD로 진짜 네트워킹을 하는 코드를 작성 + 테스트 쪽에서는 Mock 객체로 테스트하는 코드를 작성 

// For App 

- Add a shared instance to DogPatchClient
- Add a property for the network client on ListingsViewController

---

// For Test

- Create a network client protocol
- Create a mock network client using the protocol
- Use the mock to stub and validate behavior

# [1] Creating a shared instance

<details>
<summary> 1.1 DogPathClient안에 shared를 만들기 </summary>
<DogPathClientTests.swift>

```swift
func test_shared_setsBaseURL() {
    // given
    let baseURL = URL(string: "https://dogpatchserver.herokuapp.com/api/v1/")!

    // then
    XCTAssertEqual(DogPatchClient.shared.baseURL, baseURL)
 }
```

```swift
func test_shared_setsSession() {
    // given
    let session = URLSession.shared

    // then
    XCTAssertEqual(DogPatchClient.shared.session, session)
}
```

```swift
func test_shared_setsResponseQueue() {
    // given
    let responseQueue = DispatchQueue.main

    // then
    XCTAssertEqual(DogPatchClient.shared.responseQueue, responseQueue)
}
```

이 세가지 테스트를 통해서 DogPatchClient안에 이런 생김새를 가지는 shared를 가지게 되었다. 

```swift
static let shared = DogPatchClient(baseURL: URL(string: "https://dogpatchserver.herokuapp.com/api/v1/")!, session: .shared, responseQueue: .main)
```
</details>

# [2] Adding a network client property

<details>
<summary> 2.1 ListingsViewController 안에 networkClient를 만들기  </summary>

<ListingsViewControllerTest.swift> 

```swift
// sut은 ListingsViewController
func test_networkClient_setToDogPatchClient() {  
  XCTAssertTrue(sut.networkClient === DogPatchClient.shared)
}
```

이 테스트를 통해서 ListingsViewController는 이런 모양의 networkClient를 가지게 되었다. 

```swift
var networkClient = DogPatchClient.shared
```
</details>

# [3] Using the network client
<details>
<summary> 3.1  Mock Network Client가 필요한 이유 </summary> <br/>

DogPatchClient를 너의 유닛테스트에 직접적으로 썼을 때 다음과 같은 약점이 있다.

- real network call을 한다고 했을 때 인터넷 커넥션이 필요하다.
- 인터넷 연결이 가능하지 않거나 서버가 다운되었을 때, 네트워크 콜은 실패한다.
- 그리고 유닛테스트가 네트워크 응답을 기다려야하기 때문에 느려질 것이다.

mock network client를 사용해라. 그러면 너는 real network call을 하는 것을 피할수 있으면서도 완전히 response 결과를 컨트롤 할 수 있을 것이다.
</details>

<details>
<summary> 3.2  Mock Network Client를 만드는 두가지 방법 </summary>  

1. **DogPatchClient를 서브클래싱해서 mock을 만들기. 각 메소드를 오버라이딩 하기.**  
하지만...  
👿 몇몇 메소드를 오버라이딩 하는 것을 까먹으면 real network call을 할 수도 있는 위험이 있다  
👿 fake network reponse를 캐싱할 위험도 있다.  

2. **Network Client Protocol을 만들고 이 프로토콜을 따르는 Mock 오브젝트를 만들기.**  
👉 DogPatchClient를 직접 만드는 것이 아니다  
👉 1번 방식의 위험성을 방지할 수 있는 Nice한 방법이다.  
👉이 방식의 약점이라면 프로토콜을 따로 만들어야한다 정도 되겠는데, 너는 빠르고 쉽게 만들 수 있을 것이다.   

3.  실제 URL로 request한다. 하지만 request를 가로채서 내가 원하는 응답 코드와 값을 주기.   
(OHHTTStubs같은 라이브러리 이용) ⇒ [추가 3.3] 으로 내려가시오  
👿 1번 방식과 동일한  위험있음  
</details>

# [4] Creating the network client protocol
<details>
<summary> 4.1 DogPatchService 프로토콜 만들기 </summary>

<DogPathClientTests.swift>
```swift
// sut는 DogPatchClient
// DogPatchClient가 DogPathService 프로토콜을 따르고 있는지 테스트. 
func test_conformsTo_DogPatchService() {
    XCTAssertTrue((sut as AnyObject) is DogPatchService)
}
```
이 테스트를 거쳐 이런 구조가 되었다.

```swift
protocol DogPatchService {

} 
```

```swift
extension DogPatchClient: DogPatchService {

}
```
</details>

<details>
<summary> 4.2 DogPatchService 프로토콜 안에 함수 선언하기 </summary>

<DogPathClientTests.swift>

```swift
// DogPathService 프로토콜이 getDogs 라는 함수를 선언하고 있는 지 테스트.
// 컴파일 에러가 안나면 선언하고 있는 것.
func test_dogPatchService_declaresGetDogs() {
  // given
  let service = sut as DogPatchService

  // then
  _ = service.getDogs() { _, _ in }
}
```
이 테스트를 거쳐 DogPathServices는 이런 함수를 선언하고 있게 되었다. 

```swift
protocol DogPathService {
   func getDogs(completion: @escaping ([Dog]?, Error?) -> Void) -> URLSessionDataTask
}
```
</details>

# [5] Creating the mock network client
<details>
<summary> 5.1 MockDogPatchService 만들기 </summary>

<MockDogPatchService.swift> 파일 생성하고

네트워킹을 실제로 안하는 MockDogPatchService를 만들었다.

```swift
@testable import DogPatch
import Foundation

class MockDogPatchService: DogPatchService {

  var getDogsCallCount = 0
  var getDogsDataTask = URLSessionDataTask()
  var getDogsCompletion: (([Dog]?, Error?) -> Void)!

  func getDogs(
    completion: @escaping ([Dog]?, Error?) -> Void) -> URLSessionDataTask {
    getDogsCallCount += 1
    getDogsCompletion = completion
    return getDogsDataTask
  }
}
```
</details> 

# [6] Using the mock network client

<details>
<summary>  6.1 ListingsViewController의 networkClient 변수를 프로토콜 타입으로 & refreshData에 로직넣어주기 </summary>

```swift
<ListingsViewControllerTests.swift>

    // sut은 ListingsViewController
    func test_refreshData_setsRequest() {
        // given
        let mockNetworkClient = MockDogPatchService()
        sut.networkClient = mockNetworkClient

        // when
        sut.refreshData()

        // then
        XCTAssertEqual(sut.dataTask, mockNetworkClient.getDogsDataTask)
      }
```
위의 테스트함수로 ListingsViewContoller가 바뀌었다.   

// given을 위해 
👉 networkClient의 타입을 프로토콜로  
예전: 
```swift
var networkClient = DogPatchClient.shared
```
현재:
```swift
var networkClient: DogPatchService = DogPatchClient.shared
```

// then을 위해
👉 dataTask를 선언 후, refreshData하면 객체 넣어주게
```swift
var dataTask: URLSessionDataTask?

// MARK: - Refresh

@objc func refreshData() {

// TODO: - Write this
dataTask = networkClient.getDogs(completion: { (dogs, error) in

})
```
</details>

<details>
<summary> 6.2  리프레쉬가 되고 있는데, 리프레쉬가 또 불려도 API Call을 또 하지 않도록 하기 </summary>

```swift
func test_refreshData_ifAlreadyRefreshing_doesntCallAgain() {
    // given
    let mockNetworkClient = MockDogPatchService()
    sut.networkClient = mockNetworkClient

    // when
    sut.refreshData()
    sut.refreshData()

    // then
    XCTAssertEqual(mockNetworkClient.getDogsCallCount, 1)
 }
```
이 테스트의 통과를 위해 ✅ 한 부분이 추가되었다. 
```swift
// MARK: - Refresh
  @objc func refreshData() {
    // TODO: - Write this
    guard dataTask == nil else { return }
    ✅ dataTask = networkClient.getDogs(completion: { (dogs, error) in 

    })
  }
```
</details> 



<details>
<summary> 6.3 테스트 코드를 리팩토링하기 </summary>
```swift
 var mockNetworkClient: MockDogPatchService!

 func givenMockNetworkClient() {
   mockNetworkClient = MockDogPatchService()
   sut.networkClient = mockNetworkClient
 }

 override func tearDown() {
   sut = nil
   mockNetworkClient = nil
   super.tearDown()
 }

 func test_refreshData_setsRequest() {
   // given
   givenMockNetworkClient()

   // when
   sut.refreshData()

   // then
   XCTAssertEqual(sut.dataTask, mockNetworkClient.getDogsDataTask)
 }

 func test_refreshData_ifAlreadyRefreshing_doesntCallAgain() {
   // given
   givenMockNetworkClient()

   // when
   sut.refreshData()
   sut.refreshData()

   // then
   XCTAssertEqual(mockNetworkClient.getDogsCallCount, 1)
 }
```
</details>

<details>
<summary> 6.4 getDogs 에 관한 completion이 불리면 dataTask가 nil로 다시 set 되도록하기 </summary>

​```swift
func test_refreshData_completionNilsDataTask() {
    // given
    // 1
    givenMockNetworkClient()
    let dogs = givenDogs()

    // when
    // 2
    sut.refreshData()

    // 3
    mockNetworkClient.getDogsCompletion(dogs, nil)

    // then
    // 4
    XCTAssertNil(sut.dataTask)
  }
```
그래서 ✅ 부분이 추가되었음. 

```swift
// MARK: - Refresh
@objc func refreshData() {
  // TODO: - Write this
  guard dataTask == nil else { return }
  dataTask = networkClient.getDogs(completion: { (dogs, error) in
    ✅ self.dataTask = nil
  })
}
```
</details>

<details>
<summary> 6.5 refreshData에서 API Call이 성공하면 뷰모델이 업데이트 되는 지 확인하기 </summary>

👉 여기서 뷰모델은 화면당 하나가 아니라 테이블뷰 셀 당 하나임  
👉 뷰모델은 Equatable을 따르고 있어서 "같은 dog을 가지고 있는 뷰모델은 같다" 라고 비교된다.   
```swift 
// DogViewModel은 이렇게 Equatable을 구현해서 저렇게 비교하면 같다고 나옴. 
// MARK: - Equatable
extension DogViewModel: Equatable {
  static func == (lhs: DogViewModel, rhs: DogViewModel) -> Bool {
    return lhs.dog == rhs.dog
  }
}
```
👉 ListingsViewControllerTests 안의 givenDogs함수. 
```swift
func givenDogs(count: Int = 3) -> [Dog] {
    return (1 ... count).map { i in
      let dog = Dog(
        id: "id_\(i)",
        sellerID: "sellderID_\(i)",
        about: "about_\(i)",
        birthday: Date(timeIntervalSinceNow: -1 * Double(i).years),
        breed: "breed_\(i)",
        breederRating: Double(i % 5),
        cost: Decimal(i * 100),
        created: Date(timeIntervalSinceNow: -1 * Double(i).hours),
        imageURL: URL(string: "http://example.com/\(i)")!,
        name: "name_\(i)")
      return dog
    }
  }
```

```swift
func test_refreshData_givenDogsResponse_setsViewModels() {
  // given
  // 1
  givenMockNetworkClient()
  let dogs = givenDogs()  
  let viewModels = dogs.map { DogViewModel(dog: $0) }

  // when  
  // 2
  sut.refreshData()
  mockNetworkClient.getDogsCompletion(dogs, nil)

  // then
  // 3
  XCTAssertEqual(sut.viewModels, viewModels)
}
```
이 테스트의 통과를 위해 ✅이 추가됨. 

```swift
@objc func refreshData() {
    // TODO: - Write this
    guard dataTask == nil else { return }
    dataTask = networkClient.getDogs(completion: { (dogs, error) in
      self.dataTask = nil
      self.viewModels = dogs?.map { DogViewModel(dog: $0) } ?? []
    })
  }
```
</details>

<details>
<summary> 6.6 refreshData에서 API Call이 성공하고 뷰모델이 업데이트 된 후, tableview reloadData가 호출되는 지 확인하기 </summary>

```swift
// sut은 ListingsViewController
func test_refreshData_givenDogsResponse_reloadsTableView() {
    // given
    givenMockNetworkClient( givenMockNetworkClient(` givenMockNetworkClient( givenMockNetworkClient(`)
    let dogs = givenDogs()

    // 1
    // reloadData가 불렸는지 확인하고 싶어서 만든 MockTableView. 
    class MockTableView: UITableView {
      var calledReloadData = false
      override func reloadData() {
        calledReloadData = true
      }
    }

    // 2
    let mockTableView = MockTableView()
    sut.tableView = mockTableView

    // when
    sut.refreshData()
    mockNetworkClient.getDogsCompletion(dogs, nil)

    // then

    // 3
    XCTAssertTrue(mockTableView.calledReloadData)
  }
```
이 테스트의 통과를 위해 ✅이 추가됨. 
```swift 
@objc func refreshData() {
    // TODO: - Write this
    guard dataTask == nil else { return }
    dataTask = networkClient.getDogs(completion: { (dogs, error) in
      self.dataTask = nil
      self.viewModels = dogs?.map { DogViewModel(dog: $0) } ?? []
      self.tableView.reloadData()
    })
  }
```
</details>

<details>
<summary>  6.7 refreshData할때 refreshControl이 잘 동작하는 지 확인하기 </summary> 
```swift
func test_refreshData_beginsRefreshing() {
    // given
    givenMockNetworkClient()

    // when
    sut.refreshData()
    
    // then
    XCTAssertTrue(sut.tableView.refreshControl!.isRefreshing)
 }
```

​```swift
func test_refreshData_givenDogsResponse_endsRefreshing() {
  // given
  givenMockNetworkClient()
  let dogs = givenDogs()

  // when
  sut.refreshData()
  mockNetworkClient.getDogsCompletion(dogs, nil)

  // then
  XCTAssertFalse(sut.tableView.refreshControl!.isRefreshing)
}
```

두 테스트를 거치며 이렇게 바뀌었음.스트를 거치며 이렇게 바뀌었음.

```swift
@objc func refreshData() {
    guard dataTask == nil else { return }
    ✅ self.tableView.refreshControl?.beginRefreshing()
    dataTask = networkClient.getDogs() { dogs, error in
      self.dataTask = nil
      self.viewModels = dogs?.map { DogViewModel(dog: $0)} ?? []
      ✅ self.tableView.refreshControl?.endRefreshing()
      self.tableView.reloadData()
    }
  }
```
</details>

# [추가 3.3] 사실 mock network client를 쓰는 방법은 한가지 더 있다

- [민소네님 블로그](http://minsone.github.io/ios/mac/ios-mock-network-request) 에서 알게 된, [OHHTTPStubs](https://github.com/AliSoftware/OHHTTPStubs)
- 해당 URL로 한 request를 가로채서(?) 내가 원하는 응답 코드와 데이터를 준다. 
👉 비슷한 친구들:  (기억이 안남..)
<details>
<summary> 예제 코드 (티스토리) </summary>   

```swift
import XCTest
import OHHTTPStub

@testable import Tistory

class FollowingTests: XCTestCase {

    var viewModel: FollowingViewModel!
    var viewController: FollowingViewController!
    
    override func setUp() {
        super.setUp()
        viewModel = FollowingViewModel(blogName: "")
        viewController = FollowingViewController.create(viewModel: viewModel)
        viewController.loadViewIfNeeded()
    }
    
    override func tearDown() {
        viewModel = nil
        viewController = nil
        super.tearDown()
    }
    
    func test_when_following_list_empty() {
        // given
        let expect = expectation(description: "구독하는 블로거가 없으면 empty cell을 보여줘야한다.")
    
        let host = "appapi-development.test.dev.tistory.com"
        let path = "/app/v2/feed/followings"
        stub(condition: isHost(host) && isPath(path)) { (request) -> OHHTTPStubsResponse in
            return OHHTTPStubsResponse(jsonObject: ["items": []], statusCode: 200, headers: nil)
        }
    
        // when
        var visibleCells: [UITableViewCell] = []
        ApiManager.shared.requestFollowingList(page: 1, sortOption: .recency) { (code, response) in
            self.viewModel.followingList = response?.list ?? []
            self.viewModel.followingStream.onNext(code)
            visibleCells = self.viewController.tableView.visibleCells
            expect.fulfill()
        }
    
        // then
        wait(for: [expect], timeout: 1)
        XCTAssertEqual(visibleCells.count, 1)
        XCTAssert(visibleCells.first is EmptyTitleTableViewCell)
    }
    
    func test_when_following_list_not_empty() {
        // given
        let expect = expectation(description: "구독하는 블로거를 보여주어야한다.")
    
        let host = "appapi-development.test.dev.tistory.com"
        let path = "/app/v2/feed/followings"
        stub(condition: isHost(host) && isPath(path)) { (request) -> OHHTTPStubsResponse in
            return OHHTTPStubsResponse(jsonObject: ["items": [["id": "0", "name": "그렌"],
                                                              ["id": "1" , "name": "제드"],
                                                              ["id": "2" , "name": "지니"]]],
                                       statusCode: 200,
                                       headers: nil)
        }
    
        // when
        var visibleCells: [UITableViewCell] = []
        ApiManager.shared.requestFollowingList(page: 1, sortOption: .recency) { (code, response) in
            self.viewModel.followingList = response?.list ?? []
            self.viewModel.followingStream.onNext(code)
            visibleCells = self.viewController.tableView.visibleCells
            expect.fulfill()
        }
    
        // then
        wait(for: [expect], timeout: 1)
        XCTAssertEqual(visibleCells.count, 3)
        for cell in visibleCells {
            XCTAssertTrue(cell is BlogProfileTableViewCell)
        }
    }
}
```
</details>

# [추가] 🤔🤔🤔

1. Null을 제목에 넣어서 글을 발행하면 어떤 응답을 주는지 리얼 api콜로 테스트하면 좋을텐데
⇒ 근데 이건 서버쪽에서 작성해야하는 테스트인 것 같음
2. 본문에 아무것도 작성한 된 글을 띄울때 잘 띄우는지 테스트 작성해보기
⇒ 근데 이건 keditor쪽에서 작성해야하는 테스트같음
