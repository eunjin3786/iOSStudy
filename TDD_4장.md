## 1. XCTAssert 함수들 

- 같은지 : XCTAssertEqual, XCTAssertNotEqual 

- 참인지 : XCTAssertTrue, XCTAssertFalse 

- Nil 인지: XCTAssertNil, XCTAssertNotNil 

- 에러던지는지 : XCTAssertThrowsError, XCTAssertNoThrow 

- 비교: XCTAssertLessThan, XCTAssertGreaterThan 등  

  

\+ assert문이 없는 테스트는 돌리면 ‘성공’ 이라는 결과가 나온다. 



### 1.1 Testing Errors (p98)

1. Red

  ```swift
  func testModelWithNoGoal_whenStarted_throwsError() {
       XCTAssertThrowsError(try sut.start())
     }
  ```
  
   `AppModelTests.swift` 에 goal 설정없이 start하면 에러를 던지는 지 검증하는 테스트 작성

   

2. Green 

   ```swift
     class AppModel {
       let dataModel = DataModel()
   
       func start() throws {
          guard dataModel.goal != nil else {
            throw AppError.goalNotSet
          }
       }
     }
   ```

   AppModel에 다음과 같은 코드를 추가해서 테스트를 통과 시킨다



1. Green 

   ```swift
     func testStart_withGoalSet_doesNotThrow() {
       // given
       givenGoalSet()
   
       // then
       XCTAssertNoThrow(try sut.start())
     }
   ```

   반대로 goal을 설정하고 start를 하면 에러를 안던지는지 검증하는 테스트도 작성한다 

   이미 로직이 있기 때문에 테스트는 성공한다


   ==> TDD 중, 이렇게 테스트를 쓰다보면 실패없이 바로 통과되는 테스트를 쓰게 되기도 하는 군..!! 

   



## 2. 뷰컨트롤러 테스트 

뷰컨을 테스팅할때 중요한 점은 view와 control들을 directly test하지 않는점..! 이것은 UI test에서 하는 것이 더 좋다.

우리의 목표는 뷰컨의 'Logic' 과 'State' 를 체크하는 것 

MVC 패턴에서 테스트를 작성해나가는 것을 보면 썩 깔끔하지는 않은 것 같다.

=>  MVVM, VIPER, RIBs 역시 좋다.. 



### 2.1 Using the host App (p73)

`StepCountControllerTests.swift` 에서 다음과 같은 테스트를 작성하면 

```swift 
  func testChaseView_whenLoaded_isNotStarted() {
    let chaseView = sut.chaseView
    XCTAssertEqual(chaseView?.state, AppState.notStarted)
  }
```

chaseView가 nil이라서 실패한다.



그 이유는.. 

`StepCountController` 안에 `ChaseView` 는 `@IBOutlet` 으로 연결되어있다. 

실제 앱 플로우에서는 `StepCountController` 가 storyboard에서 만들어져서 앱코드가 실행할때 이미 로드되어있다.



하지만 테스트에서는 

```swift 
sut = StepCountController()
```

sut이 directly 만들어져서 실제 앱과 상황이 다르다 



#### 해결방법

일단 테스트타켓의 HostApplication을 확인하고 

![스크린샷 2019-11-13 오후 2 00 58](https://user-images.githubusercontent.com/9502063/68734544-06bc2300-061e-11ea-95c7-916f34db44df.png)



1) app window의 root뷰컨을 이용하여 원하는 뷰컨을 얻기 

```swift 
func loadRootViewController() -> RootViewController {
  let window = UIApplication.shared.windows[0]
  return window.rootViewController as! RootViewController
}

extension RootViewController {
  var stepController: StepCountController {
    return children.first { $0 is StepCountController } as! StepCountController
  }
}



let rootController = loadRootViewController()
sut = rootController.stepController
```



2) 스토리보드를 이용하기 

```swift
let storyboard = UIStoryborad(name: "Main", bundle: mil)
sut = storyboard.instantiateViewController(withIdentifier: "StepController") as! StepCountController

// load view 
sut.loadViewIfNeeded()
```





### 2.2 Randomized Order (p79)

Product -> Scheme -> Edit Scheme 들어가서 

![스크린샷 2019-11-13 오후 2 01 42](https://user-images.githubusercontent.com/9502063/68734576-218e9780-061e-11ea-9ca3-df6551d602f2.png)



Randomize execution order  체크..!! 

=> 숨겨진 테스트 간의 dependency를 파악할 수 있을 것이다!!! 

(이걸 체크안하면 순서대로 테스트가 실행되는 것 같다)







## 3. 코드 커버리지 

Product -> Scheme -> Edit Scheme 들어가서  

![스크린샷 2019-11-13 오후 2 02 19](https://user-images.githubusercontent.com/9502063/68734604-3703c180-061e-11ea-85d7-d8ddd0c8b472.png)

Gather coverage 체크 한후, 테스트를 다시 돌리기 



Report navigator(말풍선 모양) 누르고 Coverage 클릭하면 

![스크린샷 2019-11-13 오후 2 02 56](https://user-images.githubusercontent.com/9502063/68734627-4d118200-061e-11ea-8784-cb1806efdaff.png)



각 파일의 커버리지를 볼 수 있다. 





그리고 각 함수의 커버리지를 볼 수 있다. 

![스크린샷 2019-11-13 오후 2 03 20](https://user-images.githubusercontent.com/9502063/68734641-5c90cb00-061e-11ea-9243-e15b2ef8d82a.png)



그중 startStopPause 함수를 더블클릭해서 들어가보자-! 



![스크린샷 2019-11-13 오후 2 04 32](https://user-images.githubusercontent.com/9502063/68734688-86e28880-061e-11ea-9ef4-8a831528536f.png)



- 빨갛게 나오는 부분은 테스트 안된 것 

- 초록색 부분은 테스트 된 것. 숫자는 저 라인이 몇번 돌아갔는지(실행되었는지) 를 의미한다. 

  

자 빨간 부분은 UI Test의 영역이라서 테스트를 작성하지 않을 것이다. 이런 식으로 TDD는 UI testing을 포함하지 않기 때문에 우리는 100프로 커버리지를 기대할 수 없다. 



➕ 빨간 체크 부분

(빨간 체크부분을 설명하기 위해 Test debugging에서 가져온 내용)



현재 있는 테스트는 distance가 0인 상황이여서 첫번째 조건 (distance > 0) 만 거치고 있다. 


이렇게 distance > 0 조건만 체크되는 테스트를 만들면 빨간 체크 부분(striped annotation) 이 뜬다. 

빨간 체크 부분은 해당 조건을 테스트하는 또다른 테스트를 만들라는 뜻 -! 



![스크린샷 2019-11-13 오후 2 05 19](https://user-images.githubusercontent.com/9502063/68734713-a2e62a00-061e-11ea-9d5c-96f3ddbc30ff.png)



```swift
  func testModel_whenUserAheadOfNessie_isNotCaught() {
    // given
    sut.distance = 1000
    sut.nessie.distance = 100

    // then
    XCTAssertFalse(sut.caught)
  }
```



그래서 이런 테스트를 추가하여서 빨간 체크부분을 거치게 만들어주었다 





(만약 빨강, 초록 안보인다면 Editor > Code coverage를 체크하기) 





## 4. Test debugging 

Breakpoint Naviagtor (오른쪽에서 두번째, 화살표 모양) 을 눌러서 +를 누르고 Test Failure Breakpoint를 추가하자 

![스크린샷 2019-11-13 오후 2 06 44](https://user-images.githubusercontent.com/9502063/68734761-d45ef580-061e-11ea-9e23-2fe71917c6dd.png)



그러면 테스트가 실패하는 지점에서 디버거는 멈출 것이다. 

이렇게 디테일한 정보를 보고 왜 테스트가 실패하는지를 알 수 있게 되서 도움이 된다 

![스크린샷 2019-11-13 오후 2 06 20](https://user-images.githubusercontent.com/9502063/68734737-c610d980-061e-11ea-90db-3d153f8700ef.png)





## 5. 이야기하고 싶은 것 

- code coverage가 잘 업데이트 안되는  것 같다..! 테스트를 주석처리하고 테스트를 다시 돌려도 커버리지가 업데이트 안되고 그대로인 경우가 많았다. 



- 테스트 코드는 문서로의 역할도 하는데, 예제의 테스트 코드는 잘 안읽히는 것 같다. 

  테스트 이름도 그렇고 테스트 함수랑 given을 위한 그냥 함수(?)가 같이 있는 것도 조금 헷갈리는 것 같다. 

  

  그래서 대표적으로  `AppMoelTests` 를 

  ```swift
  class AppModelTests: XCTestCase {
  
    var sut: AppModel!
  
    override func setUp() {
      super.setUp()
      sut = AppModel()
    }
  
    override func tearDown() {
      sut = nil
      super.tearDown()
    }
  
    // MARK: - Given
  
    func givenGoalSet() {
      sut.dataModel.goal = 1000
    }
  
    func givenInProgress() {
      givenGoalSet()
      try! sut.start()
    }
  
    // MARK: - Lifecycle
  
    func testAppModel_whenInitialized_isInNotStartedState() {
      let initialState = sut.appState
      XCTAssertEqual(initialState, AppState.notStarted)
    }
  
    // MARK: - Start
  
    func testAppModel_whenStarted_isInInProgressState() {
      // given
      givenGoalSet()
  
      // when started
      try? sut.start()
  
      // then it is in inProgress
      let newState = sut.appState
      XCTAssertEqual(newState, AppState.inProgress)
    }
  
    func testModelWithNoGoal_whenStarted_throwsError() {
      XCTAssertThrowsError(try sut.start())
    }
  
    func testStart_withGoalSet_doesNotThrow() {
      // given
      givenGoalSet()
  
      // then
      XCTAssertNoThrow(try sut.start())
    }
  
    // MARK: - Restart
    
    func testAppModel_whenReset_isInNotStartedState() {
      // given
      givenInProgress()
  
      // when
      sut.restart()
  
      // then
      XCTAssertEqual(sut.appState, .notStarted)
    }
  }
  ```

  

  

  `Quick` 을 써서 바꿔보았는데,

  테스트 코드가 더 체계적으로 잘 파악 되는 것 같다가도 

  괄호가 너무 많아서 오히려 가독성이 떨어지는 것 같다가도.. 🤔
  ( + XCTAssertNoThrow는 it안에 적으면 컴파일 에러나서 nimble의 expect 함수써줌 )

  

  ```swift
  class AppMoelTestsWithQuick: QuickSpec {
    
    var sut: AppModel!
    
    private func givenGoalSet() {
      sut.dataModel.goal = 1000
    }
    
    private func givenInProgress() {
      givenGoalSet()
      try! sut.start()
    }
  
    override func spec() {
      
      beforeEach {
        self.sut = AppModel()
      }
      
      afterEach {
        self.sut = nil
      }
  
      describe("Initialized 단계에서") {
        it("appState는 notStarted여야한다.") {
          // when
          let initialState = self.sut.appState
          // then
          XCTAssertEqual(initialState, AppState.notStarted)
        }
      }
      
      describe("Start 단계에서") {
        context("Goal을 설정했다면") {
          beforeEach {
            // given
            self.givenGoalSet()
          }
          
          it("에러를 던지면 안된다.") {
            // then
            expect { try self.sut.start() }.notTo(throwError())
          }
          
          it("appState는 inProgress여야한다.") {
            // when
            try? self.sut.start()
  
            // then
            let newState = self.sut.appState
            XCTAssertEqual(newState, AppState.inProgress)
          }
        }
        
        context("Goal을 설정안했다면") {
          it("에러를 던져야한다") {
            // then
            expect { try self.sut.start() }.to(throwError())
          }
        }
      }
      
      describe("Restart 단계에서") {
        it("appState는 notStarted 여야한다.") {
          // given
          self.givenInProgress()
          // when
          self.sut.restart()
          // then
          XCTAssertEqual(self.sut.appState, .notStarted)
        }
      }
    }
  }
  ```

  
