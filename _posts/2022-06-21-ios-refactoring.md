---
title: 사람인 iOS App Refactoring
---

안녕하세요.  
사람인 iOS 앱을 개발하고 있는 김은미입니다. 
이번 글에서는 지난 2021년에 진행한 iOS 아이엠그라운드앱 리팩토링 과정을 공유하려고 합니다.

<br>

## 들어가며 
아이엠그라운드 앱은 사람인에서 제공하고 있는 이직을 돕기 위해 면접 대비, 입사 제안, 담당자와의 대화를 돕는 모바일 앱입니다. 아이엠그라운드 앱은 처음 개발할 때 애플에서 기본적으로 지원하는 디자인 패턴인 Cocoa MVC로 개발되었습니다. 지속해서 새로운 기능이 계속 추가되며 업데이트가 이루어지고 있기 때문에 그에 따라 코드도 함께 증가하며 복잡해졌습니다. 코드가 증가함에 따라 부각되는 Cocoa MVC의 단점 및 레거시 코드의 문제점을 개선하기 위해 리팩토링을 하였고 리팩토링에서 사용한 아키텍처와 디자인패턴을 중점으로 어떻게 개선했는지 공유하려고 합니다. 

리팩토링 프로젝트에서는 `Clean Architecture`와 `MVVM`, `Coordinator` 패턴을 도입하였습니다. 프로젝트의 계층을 분리하여 계속 기능이 추가되더라도 파악하기 쉽게 하였고, 계층별 수정이 다른 계층에 영향이 없도록 구성하였습니다. 또한 UI와 비즈니스 로직을 분리하여 뷰 모델에서 처리할 수 있도록 개선함으로 단위 테스트가 가능하게 되었습니다. 그럼 리팩토링하며 왜 이런 선택을 하였는지, 어떻게 적용하였는지, 적용 후 어떻게 달라졌는지 좀 더 자세히 설명하겠습니다. 
<br><br>
## 문제점
### Cocoa MVC (Massive View Controller)
개념상의 MVC 디자인 패턴과 다르게 실제 개발을 하다 보면 모든 객체가 Model, View, Controller에 속하는 것이 아니고 View의 라이프 사이클이 Controller와 밀접하게 상관되어 iOS의 ViewController는 많은 로직을 가지게 되며 계속 커지는 문제를 안고 있습니다. 

아이엠그라운드 앱에서는 기업 면접, 인성 면접, 역량 면접 등 여러 유형의 모의 면접을 경험할 수 있는데 유형에 따라 영상 면접, 인성 검사, 적성 게임 등의 검사들이 진행되면서 각 검사도 여러 스텝으로 진행되기 때문에 하나의 ViewController에서 UI 변동이 컸으며 각 검사에 맞는 UI와 기능을 제공하기 위해 Custom View, UITableView, WKWebView, Network, AWS S3, AVAudioPlayer 등.. 많은 기능이 ViewController에 구현되어 있었습니다. 

MVC(Massive View Controller) 하나의 파일에 많은 코드 내용이 포함되어 가독성이 떨어져 기존 코드를 파악하는 비용이 발생했고 테스트가 어려워 계속해서 추가 기능 개발과 유지 보수를 진행할 때 버그를 발생시킬 수 있는 위험이 커졌습니다. 
### 단위 테스트
테스트는 개발 과정에 있어 빼놓을 수 없는 중요한 단계로 요구 사항에 맞게 개발되었는지 평가하고 확인하기 위한 과정입니다. 좋은 프로그램은 사용자에게 최신의 기능을 제공하기 위해 지속해서 업데이트가 필요하기 때문에 테스트에 들어가는 비용도 지속해서 발생하게 됩니다.  

일련의 단계 수행이 필요한 앱의 기능은 단위 테스트를 통해 테스트에 걸리는 시간을 줄일 수 있습니다. 모의 면접 기능은 유형에 따라 시작부터 종료까지 여러 종류의 검사들을 차례로 받기 때문에 하나의 검사가 완료되면 다음 검사로 이어지는 일련의 과정으로 진행됩니다. 각 상세 기능에 대한 추가 또는 수정이 발생하면 테스트 과정에서 앞의 단계를 거쳐야 수정된 기능을 테스트할 수 있었고 이러한 문제는 테스트 비용을 증가시켰습니다. 당연한 이야기지만 전체 기능을 테스트하는 것은 소수의 기능을 테스트하는 것보다 오랜 시간이 소요됩니다. 

단위 테스트는 독립적으로 실행될 수 있어야 합니다. Cocoa MVC 구조는 Controller가 커지면서 내부에 구현된 View와 Model 사이의 의존성 때문에 단위테스트가 어려울 수밖에 없습니다. 단위 테스트하기 위해선 클래스들이 분리되어야 하고 클래스와 클래스 간의 의존성이 없어야 가능합니다. 외부 리소스에도 종속되지 않아야 합니다. 두 요소가 관련되어 어떤 요소의 변경이 다른 요소에 영향을 미치지 않도록 해야 합니다.
### Singleton Pattern
singleton은 단일 오브젝트로 존재하며 애플리케이션의 여러 곳에서 데이터를 공유하는 경우에 주로 사용됩니다. ViewController나 Class 별로 일일이 전달하기가 번거롭기 때문에 singleton pattern으로 application 내에서 유일한 상태를 공유 및 접근하도록 설계할 수 있습니다. 

기존 프로젝트에서도 빠르게 개발하기 위해 Application 내의 Class에서 쉽게 접근할 수 있도록 DebugModeManager, AppInfo, API, OAuthManager, PreferenceManager, KeychainManager, DeepLinkManager, Progress 등... 여러 기능이 singleton으로 구현되어 있었습니다. 

singleton으로 구현된 class 들은 여러 화면에서 사용되었고 때문에 수정이 필요한 경우 영향도가 application 전체로 번지게 되었습니다. 각 화면에서 사용하는 기능도 다르기 때문에 사용 위치를 추적하기 어려워 디버그에 어려움이 발생했습니다. 많은 기능이 singleton으로 개발되다 보니 singleton에서 singleton을 참조할 수 있는 위험도 생겼습니다. 이렇게 의존 관계가 생겨나고 이에 따라 코드를 분석하고 수정하는 것이 어려워졌습니다.

singleton은 class에 의존 관계이기 때문에 mock을 만들 수 없습니다. 이에 따라 단위 테스트를 작성할 때 singleton을 사용한 class 들은 의존성을 제거하기 어려워 singleton을 얻어오거나 테스트를 위한 설정이 많아지게 됩니다. singleton의 사용이 많아질수록 의존성을 가진 class 들의 테스트 코드 작성이 어려워져 변경에 대한 영향도가 큰 기능에 대해 테스트가 부족한 상태가 됩니다. 
<br><br>
## 개선 사항
### 리팩토링, 시작
리팩토링을 하기 위해서는 여러 가지 방향으로 접근할 수 있을 것 같습니다. 리팩토링에 대한 시급하게 요구 사항 있는 화면 또는 기능이 있어서 먼저 진행할 수도 있고, 리팩토링으로 인해 성능 개선이 바로 일어나도록 핵심적인 부분부터 진행할 수도 있고, 안정성을 고려해 서비스에 미치는 영향도가 가장 낮은 부분부터 진행할 수도 있을 것 같습니다. 

아이엠그라운드 앱의 경우에는 성능 이슈가 있었던 것이 아니고 유지보수와 이후 기능 추가를 위해 설계 및 구성 기반을 마련하고 코드 품질을 높이는 것이 목적이었습니다. Cocoa MVC의 기초적인 구조에서 발생할 수 있는 문제를 방지하고 유지보수 및 앞으로의 요구 사항을 수정할 수 있는 효율적인 구조로의 변경과 단위 테스트를 수행하길 원했습니다. 동일한 기능을 제공하지만 내부를 완전히 재구성하는 작업이기 때문에 기존 코드의 안정성을 유지하며 프로젝트 전체에 끼치는 영향도를 고려하여 진행할 수 있도록 계획하였습니다. 

리팩토링에는 비용이 들어갑니다. 그런데도 앞서 이야기한 이유로 필요한 과정이었고 기존 운영 및 추가개발 업무를 진행하며 함께 진행하였기 때문에 미치는 영향도가 가장 낮은 기능 화면부터 시작하였습니다. 리팩토링과 단위 테스트 후 iOS 동료 개발자분과 코드 리뷰를 통해 설계와 패턴이 제대로 적용되었는지 확인할 수 있도록 하였습니다. 이후 QA 진행 및 배포 순으로 7개월에 걸쳐 완료하였습니다. 

위에서 설명한 문제점들을 해결하기 위해 가장 먼저 필요한 것은 설계였습니다. 좋은 아키텍처란 무엇일까요? 여러 좋은 글들에서 이야기하는 조건들이 많지만 위에 설명한 문제점을 해결하기 위해 아키텍처를 선택하며 중점으로 생각한 부분은 아래와 같습니다. 

- 관심사의 분리로 하나의 책임만 담당하도록 구분되어 특정 프레임워크나 서비스를 사용하지만 종속되지 않아 언제든 교체하여 변화에 적응할 수 있는 설계 
- 쉽게 바꿀 수 있는 유연한 구조로 설계 
- 의존성을 최대한 제거, 단순하고 일정한 데이터의 흐름으로 코드의 이해와 디버그가 쉬움
- 의존성 제거로 가능해지는 단위 테스트 

<br>
### Clean Architecture
Robert C Martin(Uncle Bob)님이 제안하여 유명해진 [클린 아키텍처](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)는 소프트웨어를 계층으로 나누어 관심사 분리를 구현했습니다. 클린 아키텍처의 분리된 각 계층을 크게 세 개의 Layer로 구성할 수 있습니다.

![클린아키텍처](/img/neyo/iOS-CleanArchitecture.jpeg){: width="720" height="500"}
#### Domain Layer
클린 아키텍처에서 Entities, Use Cases를 의미합니다. 도메인 레이어는 다른 레이어들에 어떤 영향도 받지 않습니다. 앱에 필요한 주요한 비즈니스 로직과 비즈니스 로직에 의해 영향을 받게 되는 모델들이 해당합니다. 

**Entities**   
비즈니스 모델을 의미합니다. 핵심 규칙에 해당하는 영역으로 다른 구성 요소에 의존하지 않아 UseCases 등 다른 계층을 알지 못합니다. 

![분석레포트](/img/neyo/ios-report.jpeg){: width="720" height="500"}    
분석 레포트 화면

아래는 분석 리포트 엔티티의 일부입니다. 모의 면접 완료 후 결과 확인을 위해 제공되는 분석 리포트의 구조 및 관련 함수가 정의되어 있습니다. 이 개체는 외부의 변경으로 변경되지 않습니다.  

```swift
struct Analysis: Codable {
    struct Result: Codable {
        var result: Report?
    }
    
    struct Report: Codable {
        var tips: Tips?
        var character: Character?
        var aptitude: Aptitude?
        var keywords: [String]?
        
        enum CodingKeys: String, CodingKey {
            case tips
            case character
            case aptitude
            case keywords = "keyword"
        }
    }
//...

```

**Use Cases**   
Entities의 데이터 흐름을 나타내며 애플리케이션의 모든 사용 사례를 구현하는 계층입니다. UseCase를 통해 밖의 레이어에서 Entities를 사용하기 때문에 의존도는 Entities를 향하며 Entities를 제외한 다른 계층은 알지 못합니다. 분석 리포트는 API를 통해 데이터를 전송받기 때문에 UseCase를 아래와 같이 작성하였습니다.

```swift
protocol AnalysisResultUseCase {
    func requestAnalysisReport(token: String, termNo: UInt) -> Observable<Model.Analysis.Result>
}
```
<br>
#### Data Layer
클린 아키텍처에서 DB, API를 의미합니다. Presentation Layer 계층으로 데이터를 불러오는 역할을 수행합니다. 네트워크를 통해 데이터를 받아오거나 DataBase를 통해 데이터를 가져오는 작업을 하는 것이 해당합니다.

Domain Layer에서 정의한 Use Cases 프로토콜들을 채택하여 실제 구현 부로 API, DB, Socket 등... 다양한 프레임워크를 통해 데이터를 처리한 뒤 모델로 변환하여 전달할 수 있도록 합니다. 실제 구현 부에서 사용한 프레임워크의 변경이 다른 Layer에 영향을 주지 않도록 하여 얼마든지 더 좋은 프레임워크로 교체가 가능합니다. 

Use Cases를 구현할 때는 API, DB 등의 프레임워크 사용을 구현한 클래스를 주입받아 사용하도록 하였습니다. 분석 리포트의 경우 API 통신을 통해 데이터를 전송받기 때문에 Alamofire 라이브러리를 이용하여 get, post 등의 통신을 구현한 Network 클래스를 주입하여 사용하였습니다. 전달받은 데이터는 Domain Layer에서 정의한 비즈니스 모델과 매핑하여 사용하였습니다.

```swift
// DomainLayer 정의된 AnalysisResultUseCase 실제 구현 
class NetworkAnalysisResultUseCase: AnalysisResultUseCase {

    private let network: Network<Model.Analysis.Result>

    init(network: Network<Model.Analysis.Result>) {
        self.network = network
    }
    
    func requestAnalysisReport(token: String, termNo: UInt) -> Observable<Model.Analysis.Result> {
        let router = Router.analysisReport(token: token, termNo: termNo)
        return network.getItem(router.path, parameters: router.parameters, headers: router.headers)
    }
}
```

```swift
class Network<T: Decodable> {
    
    private let endPoint: String
    private let scheduler: ConcurrentDispatchQueueScheduler
    
    init(_ endPoint: String) {
        self.endPoint = endPoint
        self.scheduler = ConcurrentDispatchQueueScheduler(qos: DispatchQoS(qosClass: DispatchQoS.QoSClass.background, relativePriority: 1))
    }
    
    private func makeHeaders(headers: HTTPHeaders? = nil) -> HTTPHeaders? {
        var makeHeaders = headers
        if let userAgent = HTTPHeaders.default.dictionary["User-Agent"]  {
            makeHeaders?.add(name: "User-Agent", value: userAgent)
        }
        return makeHeaders
    }
    
    func getItem(_ path: String,
                 parameters: [String: Any]? = nil,
                 encoding: ParameterEncoding = URLEncoding.default,
                 headers: HTTPHeaders? = nil) -> Observable<T> {
        let absolutePath = "\(endPoint)\(path)"
        return RxAlamofire
            .request(.get,
                     absolutePath,
                     parameters: parameters,
                     encoding: encoding,
                     headers: makeHeaders(headers: headers))
            .observeOn(scheduler)
            .responseData()
            .flatMap { response, data -> Observable<(String, [String: Any]?, HTTPHeaders?, HTTPURLResponse, Data)> in
                let temp = (absolutePath, parameters, headers, response, data)
                return Observable.just(temp)
            }
            .validationResultThrowError(ofType: T.self)
    }
//...

```
<br>
#### Presentation Layer
클린 아키텍쳐에서 Presenters와 UI를 의미합니다. 화면에 보이는 영역을 담당하는 레이어로 UI에 해당하는 UIViewController, SwiftUI 이 해당됩니다. 
MVVM을 적용해 View와 ViewModel로 구현하였는데 View(UIViewContoller)에서는 UI와 관련된 부분을 처리하였고 ViewModel에서는 Domain Layer의 Use Cases를 호출하여 비즈니스 로직을 처리하였습니다. 

분석레포트 화면은 모의면접을 치른 사용자에게 응답 데이터를 분석하여 결과를 제공하는 화면입니다. ViewModel에 Domain Layer의 UseCase를 주입받아 화면 로딩 시점에 UseCase에 정의된 분석레포트 데이터를 불러오는 기능을 수행하여 모델을 통해 데이터를 전달 받도록 하였습니다.

```swift
class ReportViewController: BaseViewController {
    
    @IBOutlet private weak var tableView: UITableView!
    
    private var dataSource: RxReloadDataSource<Section>!
    private var disposeBag = DisposeBag()
    
    var viewModel: ReportViewModel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
    }
    
    override func layout() {
        tableView.registerCell(AnalysisReportTitleTableViewCell.self)
        //...
    }

    override func bindView() {
        initDataSource()
        //...
    }

    override func bindViewModel() {
        
        let loadTrigger = Observable.just(())
        let totalReport = analysisReportSectionView.rx.segmentIndex
            .filter { $0 == 0 }
            .mapToVoid()

        let input = ReportViewModel.Input(loadTrigger: loadTrigger.asDriverOnErrorJustComplete(),
                                          totalReportTrigger: totalReport.asDriverOnErrorJustComplete(),
																					//...
                                          toHintTrigger: hintButtonClicked.asDriverOnErrorJustComplete())

        let output = viewModel.transform(input: input)

        output.sections
            .debug("sections")
            .drive(tableView.rx.items(dataSource: self.dataSource))
            .disposed(by: disposeBag)
        //...
    }
//...

```

```swift
final class ReportViewModel: BaseViewModel {

    // protocol
    private let navigator: CompletedInterviewNavigatorType
    private let popupNavigator: PopupNavigatorType
    private let analysisUseCase: AnalysisResultUseCase
    //...
    private var auth: AppAuthProtocol
    private var data: CompletedInterviewData
    
    private var disposeBag = DisposeBag()
		
    // 의존성 주입
    init(navigator: CompletedInterviewNavigatorType,
         popupNavigator: PopupNavigatorType,
         analysisUseCase: AnalysisResultUseCase,
         //...
         auth: AppAuthProtocol,
         data: CompletedInterviewData) {
        
        self.navigator = navigator
        self.popupNavigator = popupNavigator
        self.analysisUseCase = analysisUseCase
        //...
        self.auth = auth
        self.data = data
        super.init(serviceCheckUseCase: serviceCheckUseCase)
    }
}

extension ReportViewModel: ViewModel {
    
    enum InputTrigger: Trigger {
        case totalReport
    }
    
    struct Input {
        let loadTrigger: Driver<Void>
        let totalReportTrigger: Driver<Void>
        //...
        let toHintTrigger: Driver<HintType>
    }
    
    struct Output {
        let sections: Driver<[Section]>
        //...
        let error: Driver<(Trigger, Swift.Error)>
    }
    
    func transform(input: Input) -> Output {
        
        let activityIndicator = ActivityIndicator()
        activityIndicator.drive(App.progress.rx.animation).disposed(by: disposeBag)

        let sectionsBehaviorRelay = BehaviorRelay<[Section]>(value: [])
        let videoReportPublishRelay = PublishRelay<Void>()
        
        // MARK: API
        let refreshToken: ( (_ trigger: InputTrigger) -> Observable<Model.Auth>)
            = { [weak self] trigger in
                guard let `self` = self else { return Observable.empty() }
                return self.requestRefreshToken(authManager: self.auth,
                                           authUseCase: self.authUseCase,
                                           userUseCase: self.userUseCase,
                                           msgTokenUseCase: self.msgTokenUseCase,
                                           trigger: trigger,
                                           activityIndicator: activityIndicator,
                                           errorPublishRelay: self.errorPublishRelay)
            }
        
        let requestTotalReport = refreshToken(.totalReport)
            .flatMapLatest { [weak self] auth -> Observable<Model.Analysis.Result> in
                guard let `self` = self else {
                    return Observable.empty()
                }
                return self.analysisUseCase
                    .requestAnalysisReport(token: auth.accessToken,
                                           termNo: self.data.termNo)
                    .debug("requestAnalysisReport, 미디어 파일 요청")
                    .trackActivity(activityIndicator)
                    .catchErrorAcceptComplete(self.errorPublishRelay, InputTrigger.totalReport)
            }

        // MARK: Trigger
        input.loadTrigger
            .debug("loadTrigger")
            .drive(onNext: { [weak self] _ in
                guard let `self` = self else { return }
                let items: [Section] = [.section(items: [.title(self.data)])]
                sectionsBehaviorRelay.accept(items)
            }).disposed(by: disposeBag)
        
        input.totalReportTrigger.asObservable()
            .debug("totalReportTrigger")
            .flatMapLatest { _ in
                return requestTotalReport
            }.subscribe(onNext: { [weak self] result in
                guard let `self` = self, let result = result.result else { return }
                let reportDatas = ReportViewType.total(report: result, itemTypeCode: self.data.itemTypeCode).datas
                let section = self.createSection(interview: self.data, reportDatas: reportDatas)
                sectionsBehaviorRelay.accept(section)
                scrollPublishRelay.accept((section: 0, row: 0))
            }).disposed(by: disposeBag)
        
        //...
        return Output(sections: sectionsBehaviorRelay.asDriverOnErrorJustComplete(),
																					 error: errorPublishRelay.asDriverOnErrorJustComplete())
    }
//...

```


클린 아키텍처의 `The Dependency Rule`은 각 레이어는 분리된 영역을 나타내며 안쪽을 향해서만 의존이 가능하며 외부 레이어가 내부 레이어에 영향을 끼치지 않도록 설계되어 있습니다. 안쪽일수록 고수준의 영역으로 추상화되고 바깥의 레이어일수록 구체화합니다.

각 계층 간의 의존성 방향이 아래와 같이 흐르게 됩니다. 
#### Presentation Layer -> Domain Layer <- Data Layer

그런데 실제 흐름을 살펴보면 Presentation Layer에서 Domain Layer의 Use Case를 통해 Data Layer에 요청하는 것을 볼 수 있습니다. 
1. View에서 ViewModel로 UI Life Cycle 및 사용자의 Action 신호를 전달합니다. 
2. ViewModel에서 DomainLayer의 UseCase를 호출합니다.
3. UseCase를 통해 Data Layer의 API, DB 등의 실제 구현 부를 호출하여 데이터를 가져옵니다. 
4. 데이터는 Domain Layer에 정의된 모델로 정의되어 ViewModel을 통해 View로 전달되고 화면에 나타납니다. 

**의존성의 방향을 유지하며 Flow를 가능하게 하려면 Domain Layer와 Data Layer 사이에 DIP(Dependency Inversion Principle) 의존성 역전이 필요합니다.**
#### DIP(Dependency Inversion Principle) 

'고차원 모듈은 저차원 모듈에 의존하면 안 된다. 이 모듈 모두 다른 추상화된 것에 의존해야 한다. 추상화된 것은 구체적인 것에 의존하면 안 된다. 구체적인 것이 추상화된 것에 의존해야 한다.' (Martin, Robert C.) 

즉, 바깥쪽 레이어의 객체를 내부에서 생성하지 않고 추상화된 것에 의존하도록 하는 것입니다. Domain Layer에서 Data Layer에 의존하지 않고 API나 DB 등... 데이터를 받아와야 합니다. 이런 경우 Domain Layer와 Data Layer 사이에 Protocol을 두어 처리할 수 있습니다. Domain Layer의 Use Case를 Protocol로 정의하고 실제 구현은 Data Layer에서 Domain Layer의 UseCase를 채택하여 구현 함으로써 의존성의 방향은 지키면서 요청을 할 수 있게 됩니다. 이를 위하여 ViewModel에는 Domain Layer의 UseCase를 정의하였고 ViewModel을 생성할 때 Data Layer의 UseCase를 생성하여 주입하였습니다. 

이에 따라 각 계층 레이어에서 사용하는 프레임워크 외부 라이브러리들은 다른 레이어와는 무관해지고 UI와 비즈니스 로직에서 분리되어 얼마든지 쉽게 교체하여 변화에 빠르게 적용할 수 있게 되었습니다. 의존성 규칙을 정함으로 비즈니스 로직이 분리되어 UI와 상관없이 테스트할 수 있었고 DB, API 등의 데이터는 mock을 이용하여 테스트할 수 있게 되었습니다. 

### Coordinator Pattern
Coordinator 패턴은 UIViewController가 보유하던 책임 중 Navigation과 관련된 부분을 다른 인스턴스에서 책임지도록 하는 패턴입니다. 

아이엠그라운드 앱에서 모의 면접을 제공할 때 유형에 따라 순서가 달라지는 경우 하나의 화면에서 여러 개의 화면 전환이 필요하고, 한 화면으로의 전환이 여러 화면에서 요청되기도 했습니다. ViewController에서 직접적으로 화면 전환을 호출해 왔는데 이런 경우 호출하는 ViewController 간에 커플링이 발생할 수밖에 없습니다. 화면이 많아질수록 화면 전환 코드가 여러 곳에 흩어지며 중복이 발생합니다. 파악 관리하기 어려우며 디버그 및 테스트에 비용이 들어갈 수 있습니다. 

ViewController에서 화면 이동 코드를 분리하여 Coordinator를 통해 화면 전환을 요청하도록 하였습니다. 하나의 UINavigationController로 관리하려는 화면을 그룹으로 묶어 하나의 Coordinator로 관리하였습니다. 먼저 Protocol로 필요한 화면 이동을 정의한 뒤 해당 프로토콜을 채택하여 실제 화면 이동을 구현하였습니다. Coordinator에서 ViewController와 ViewModel 및 모든 종속성을 생성했습니다. ViewController와 ViewModel 생성 시 Data Layer의 API, DB, Socket 등.. 서비스를 인스턴스화 해서 필요에 따라 주입하였고 ViewModel 내부에서는 Protocol을 선언하여 기능을 호출했습니다. 

화면 전환 로직을 ViewController에서 분리하여 중복이 제거되고 ViewController가 가벼워졌고 화면 전환 코드의 수정이 쉬워졌으며 ViewModel에서 화면 이동을 호출할 때 Protocol을 호출하게 하여 화면 이동 코드와 관계없이 비즈니스 로직 테스트가 가능하게 되었습니다. 

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?
    
    private var disposeBag = DisposeBag()

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        //...
        AppNavigator.shared.start(window: window)
    }
//...

```

```swift
final class AppNavigator {
    
    static let shared = AppNavigator()
    // ViewModel에 의존성 주입하기 위한 객체 정의 (API, DB, Socket 등..) 
    private let networkProvider: UseCaseProvider
    
    private init() {
        networkProvider = NetworkUseCaseProvider()
    }
    
    func start(window: UIWindow) {
        let intro = UIStoryboard.init(storyboard: .main).instantiateViewController(ofType: RIntroViewController.self)
        
        let navigationController = BaseNavigationController(rootViewController: intro)
        
        let homeNavigator = HomeNavigator(networkProvider: networkProvider,
                                          parentNavigationController: navigationController,
                                          navigationController: nil,
                                          storyBoard: .init(storyboard: .home))
        
        let popupNavigator = PopupNavigator(networkProvider: networkProvider,
                                            parentNavigationController: navigationController,
                                            navigationController: nil,
                                            storyBoard: .init(storyboard: .popup))
        
        intro.viewModel = IntroViewModel(homeNavigator: homeNavigator,
                                         popupNavigator: popupNavigator,
                                         serviceCheckUseCase: networkProvider.makeServiceCheckUseCase(),
                                         authUseCase: networkProvider.makeAuthUseCase(),
                                         versionsUseCase: networkProvider.makeVersionsUseCase(),
                                         //...
                                         remoteConfig: RemoteConfigManager())
        window.rootViewController = navigationController
        window.makeKeyAndVisible()
    }
}

```

```swift
protocol HomeNavigatorType {
    func pop(animated: Bool, completion: @escaping () -> Void)
    func dismissToMain(animated: Bool, completion: (() -> Void)?)
    
    func toHome(_ type: TransitionType)
    func toLogin(_ type: TransitionType)
    //...
}

final class HomeNavigator: BaseNavigator, HomeNavigatorType {
    
    override init(networkProvider: UseCaseProvider,
                  parentNavigationController: UINavigationController?,
                  navigationController: BaseNavigationController?,
                  storyBoard: UIStoryboard) {

        super.init(networkProvider: networkProvider,
                   parentNavigationController: parentNavigationController,
                   navigationController: navigationController,
                   storyBoard: storyBoard)
    }
    
    func toHome(_ type: TransitionType) {
        
        let home = storyBoard.instantiateViewController(ofType: HomeViewController.self)
    
        //...
        let contentViewControllers: [BaseViewController] = [interview, main, msgList]
        home.viewModel = HomeViewModel(navigator: self,
                                       serviceCheckUseCase: networkProvider.makeServiceCheckUseCase(),
                                       msgTokenUseCase: networkProvider.makeMsgTokenUseCase(),
                                       authUseCase: networkProvider.makeAuthUseCase(),
                                       userUseCase: networkProvider.makeUserUseCase(),
                                       homeUseCase: networkProvider.makeHomeUseCase(),
                                       msgUnreadUseCase: networkProvider.makeMsgUnreadUseCase(),
                                       freeApplyUseCase: networkProvider.makeFreeApplyUseCase(),
                                       auth: App.rAuth,
                                       homeDataManager: App.homeManager,
                                       contentViewControllers: contentViewControllers)
        toNavigation(type, home)
    }

    func toLogin(_ type: TransitionType) {
        let navigator = PopupNavigator(networkProvider: networkProvider,
                                       parentNavigationController: navigationController,
                                       navigationController: nil,
                                       storyBoard: .init(storyboard: .popup))
        navigator.toLogin(type, loginFromEvent: nil, dismissIntro: nil)
    }
//...
}

```

```swift
final class HomeViewModel: BaseViewModel {
    
  private let navigator: HomeNavigatorType
	private let homeUseCase: HomeUseCase
	private var auth: AppAuthProtocol
	//...
 
    init(navigator: HomeNavigatorType,
         serviceCheckUseCase: ServiceCheckUseCase,
         homeUseCase: HomeUseCase,
         //...
         auth: AppAuthProtocol) {
        
        self.navigator = navigator
        self.authUseCase = authUseCase
        self.homeUseCase = homeUseCase
        //...
        self.auth = auth
        super.init(serviceCheckUseCase: serviceCheckUseCase)
    }
}

extension HomeViewModel: ViewModel {
    
    enum InputTrigger: Trigger {
        case home
        //...
    }

    struct Input {
        let loadHomeTrigger: Driver<Bool>
        let unreadMsgTrigger: Driver<Bool>
        let retryFreeApplyTrigger: Observable<Void>
        let deepLinkWithLoginTrigger: BehaviorSubject<TargetType?>
        //...
    }

    struct Output {
        //...
        let error: Driver<(Trigger, Swift.Error)>
    }

    func transform(input: Input) -> Output {
        let activityIndicator = ActivityIndicator()
        activityIndicator.drive(App.progress.rx.animation).disposed(by: disposeBag)
       
        //...
        input.deepLinkWithLoginTrigger
            .filter { $0 != nil }
            .map { $0! }
            .subscribe(onNext: { [weak self] type in
                guard let `self` = self, let user = self.auth.getUser(), self.auth.isValid() else {
                    self?.navigator.dismissToMain(animated: true, completion: nil)
                    return
                }
                
                switch type {
                case .analysisComplete(let memIdx, let termNo):
                    guard let memIdx = memIdx, let termNo = termNo, user.memIdx == memIdx else {
                        self.navigator.dismissToMain(animated: true, completion: nil)
                        return
                    }
                    let data = CompletedInterviewData(termNo: termNo,
                                                      careerName: nil,
                                                      occupationName: nil,
                                                      companyName: nil,
                                                      regDateString: nil,
                                                      itemTypeCode: nil,
                                                      programLevel: nil)
                    self.navigator.toCompletedInterview(.present(true),
                                                        data: data,
                                                        accessDeepLink: true)

                case .expirataion1d, .expirataion3d, .expirataion7d:
                    self.navigator.toMyPage(.present(true), accessDeepLink: true)
                //...
                default:
                    break
                }
            }).disposed(by: disposeBag)
        
        return Output(error: errorPublishRelay.asDriverOnErrorJustComplete())
    }
    //...
}

```
<br>
### MVVM Pattern 
MVVM 패턴은 Model-View-ViewModel의 약자로 UI 로직(View)을 비즈니스 로직(ViewModel)과 분리하고 reactive 하게 UI를 동작하게 구성하는 디자인 패턴입니다. 

![mvvm](/img/neyo/iOS-mvvm.png){: width="720" height="250"}

기존 ViewController에서 Model과 비즈니스 로직(ViewModel)을 분리하여 라이프 사이클과 강력하게 연결된 UIViewController를 View로 분류하고, UIKit과 관련된 코드 없이 비즈니스 로직을 작성하여 ViewModel로 분류합니다. ViewController는 비즈니스 로직이 빠지며 가벼워지고 ViewModel는 UI와 독립적으로 테스트할 수 있습니다. 테스트 코드를 작성할떄도 UITest, UnitTest로 분리하여 작성하게 됩니다. 

MVVM패턴의 동작을 살펴보면 View에서는 화면에 그려지는 요소들을 정의하고 발생하는 event를 ViewModel로 전달합니다. ViewModel은 event가 발생했을 때 통신, 계산 비즈니스 로직을 수행하여 Model을 View에 전달하고 View에서 변경된 데이터를 업데이트하게 됩니다. 이렇게 일정한 방향으로 이벤트가 흐르게 되면 코드를 읽기 쉬우며 복잡도가 낮아지게 됩니다. 

View에서 비즈니스 로직이 분리되면 업데이트되는 Model과 UI와 싱크를 맞추기 위해서 Data Binding이 필요합니다. 분리된 View와 ViewModel를 연결하기 위해 RxSwift를 이용하여 reactive 하게 처리할 수 있습니다. Model에 변경이 생기게 되면 UI도 업데이트가 이루어져 데이터가 일관성을 유지하게 됩니다. 


![메시지](/img/neyo/ios-chat.png){: width="510" height="510"}    
메시지 화면

아래는 기업과 사용자 간의 메시지를 보낼 수 있는 화면에서 상대가 보낸 메시지를 신고할 수 있는 기능에 관련된 코드 중 일부입니다. 테이블뷰에 바인딩 된 각각의 메시지에서 사용자의 swipe action이 일어나면 ViewController의 TableView에서 event를 받아 ViewModel로 전달하게 됩니다. ViewModel은 해당 event를 받으면 신고하기 팝업을 노출하고 API 통신을 처리하는 로직을 수행합니다. 신고하기 API의 성공 여부를 확인하여 다시 ViewController로 결과를 전달하면 UI 관련 업데이트(성공 후 화면 이동 또는 실패 얼럿 노출)를 처리합니다.

```swift
class MsgViewController: BaseViewController {
   
    @IBOutlet weak var tableView: MsgTableView!
    private let msgDeclarePublishRelay = PublishRelay<MessageObject>()
    //...

    private var disposeBag = DisposeBag()
    
    var viewModel: MsgViewModel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        //...
    }

    override func layout() {
        tableView.refreshControlInit()
        tableView.registerCell(MsgDateCell.self)
        //...
    }
    
    override func bindView() {
        //...
        tableView.declareClosure = { [weak self] cellData in
            guard let message = cellData.message else { return }
            self?.msgDeclarePublishRelay.accept(message)
        }
    }
    
    override func bindViewModel() {
        //...
        let input = MsgViewModel.Input(messageTrigger: messagePublishRelay.asDriverOnErrorJustComplete(),
																																									 //...
																																									 declareTrigger: msgDeclarePublishRelay.asDriverOnErrorJustComplete())

        let output = viewModel.transform(input: input)
        
        output.openWeb
            .drive(onNext: { [weak self] url in
                UIApplication.shared.open(url, options: [:], completionHandler: nil)
            }).disposed(by: disposeBag)
        
        output.alert.asObservable()
            .flatMap{ [weak self] type -> Observable<Bool> in
                guard let `self` = self else { return Observable.empty() }
                return App.showAlert(type: type, onView: self)
            }.subscribe(onNext: { _ in
                
            }).disposed(by: disposeBag)
        //...
    }
    //...
}

```

```swift
final class MsgViewModel: BaseViewModel {
    
    //...
    private var disposeBag = DisposeBag()
    
    init(navigator: MsgNavigatorType,
         popupNavigator: PopupNavigatorType,
         serviceCheckUseCase: ServiceCheckUseCase,
         //...
         msgMessageUseCase: MsgMessageUseCase,
         room: Model.Room) {
        
        self.navigator = navigator
        self.popupNavigator = popupNavigator
        //...
        self.msgMessageUseCase = msgMessageUseCase
        self.room = room
        super.init(serviceCheckUseCase: serviceCheckUseCase)
    }
}

extension MsgViewModel: ViewModel {
    enum InputTrigger: Trigger {
        case msgList(index: Int)
        //...
    }

    struct Input {
        var messageTrigger: Driver<Int> //index
        var declareTrigger: Driver<MessageObject>
        //...
    }

    struct Output {
        let roomInfo: Driver<Model.Room>
        //...
        let openWeb: Driver<URL>
        let alert: Driver<AlertMessageType>
        let error: Driver<(Trigger, Swift.Error)>
    }

    func transform(input: Input) -> Output {
        let activityIndicator = ActivityIndicator()
        activityIndicator.drive(App.progress.rx.animation).disposed(by: disposeBag)

        let openWebPublishRelay = PublishRelay<URL>()
        let alertPublishRelay = PublishRelay<AlertMessageType>()
        //...

        let compeleDeclareBlock: ((_ url: URL?) -> ()) = { [weak self] url in
            self?.navigator.pop(animated: true)
            if let url = url {
                openWebPublishRelay.accept(url)
            } else {
                alertPublishRelay.accept(.declareComplete)
            }
        }
        
        input.declareTrigger.asObservable()
            .subscribe(onNext: { [weak self] message in
                guard let `self` = self else { return }
                self.navigator.toDeclare(.push(true, .push), message: message, completeDeclare: compeleDeclareBlock)
         }).disposed(by: disposeBag)
        //...

        return Output(roomInfo: Observable.just(room).asDriverOnErrorJustComplete(),
                      //...
                      openWeb: openWebPublishRelay.asDriverOnErrorJustComplete(),
                      alert: alertPublishRelay.asDriverOnErrorJustComplete(),
                      error: errorPublishRelay.asDriverOnErrorJustComplete())
    }
}

```
<br> 
## 리팩토링 결과 
기능별 구조가 명확해지고 의존성이 사라져 테스트가 가능한 구조로 UnitTest UITest가 가능해졌습니다. 새로운 기능을 추가해도 부분 기능 테스트가 가능하고 사이드 이펙트가 일어나지 않도록 쉽게 확인할 수 있게 되었습니다. 

데이터의 흐름이 단일화 되어 이해하기 쉬워졌습니다. 코드 구조가 일정하게 유지되어 협업 시에도 파악과 이해가 쉬워졌습니다. 오류 발생 시 디버깅이 쉬워져 유지 보수 및 추가 기능 개발 속도가 향상되었습니다.

의존성 규칙을 따라 계층별 관심사가 분리되어 특정 프레임워크나 라이브러리에 종속되지 않고 쉽게 수정할 수 있는 유연함이 생겼습니다.

## 마치며
리팩토링을 진행하며 문제점들을 해결하기 위해 우리에게 필요한 아키텍처가 무엇일까 고민하였습니다. 평가해보면 우리에게 맞는 아키텍처를 선택했던 것 같고 이를 통해 문제들이 해결되며 구체적이고 명확하게 역할이 나눠지고, 데이터의 흐름이 단순해지고, 쉽게 바꿀 수 있게 되고, 테스트가 쉬워지는 코드가 되었습니다. 결과적으로는 개발 속도가 향상되었습니다. 

여전히 개선하고 싶은 점이 많이 있습니다. Coordinator를 통해 의존성을 주입하였지만 더 쉽게 할 수 있는 방법은 없을까? UseCase를 더 간결하게 관리할 수 없을까? Singleton을 사용할 수 있는 더 좋은 방법은 없을까? 등 이미 적용한 아키텍처에 대해서도 부족한 부분 미흡한 부분을 개선하는 방법을 계속 생각해봐야 할 필요가 있고, 이 외의 아키텍처에 대해서도 더 잘 맞는 아키텍처가 있는지 찾아보고 리팩토링을 통해 개선해 나갈 계획입니다. 

감사합니다.
<br><br>

## 참고 
- [https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html)
- [https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [https://academy.realm.io/kr/posts/krzysztof-zablocki-mDevCamp-ios-architecture-mvvm-mvc-viper/](https://academy.realm.io/kr/posts/krzysztof-zablocki-mDevCamp-ios-architecture-mvvm-mvc-viper/)
- [https://github.com/kudoleh/iOS-Clean-Architecture-MVVM](https://github.com/kudoleh/iOS-Clean-Architecture-MVVM)
- [https://www.youtube.com/watch?v=4GjXq2Sr55Q&feature=youtu.be](https://www.youtube.com/watch?v=4GjXq2Sr55Q&feature=youtu.be)
- [https://cskime.github.io/2022-03-05/mvvm-ios](https://cskime.github.io/2022-03-05/mvvm-ios)
- [https://develogs.tistory.com/19](https://develogs.tistory.com/19)
- [https://www.jessesquires.com/blog/2017/02/10/refactoring-singletons-in-swift/](https://www.jessesquires.com/blog/2017/02/10/refactoring-singletons-in-swift/)
