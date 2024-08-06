# RIBS 사용해보기

## 프로젝트 구조

<img src = "https://github.com/HwangWoonChun/Study-AppArchitecture/blob/main/image/md04_01.png" width = 651 height = 331>

* RIBS 는 Router, Interaction, Builder 이 있는데 이를 따 RIBS 라고 칭한다. 이를 묶어 갈빗대(폭립)에서 본따 리블렛이라고 표현하기도 한다.
 
### 1. Builder 살펴보기
* Builder 는 리블렛 객체를 생성하는 역할을 한다.
* Dependency 를 인자값으로 받는다.
* 빌드 함수를 통해 리블렛에 필요한 객체를 생성한다.
```swift
import ModernRIBs

public protocol AppHomeDependency: Dependency {
}

final class AppHomeComponent: Component<AppHomeDependency>, TransportHomeDependency {
}

// MARK: - Builder

public protocol AppHomeBuildable: Buildable {
    func build(withListener listener: AppHomeListener) -> ViewableRouting
}

public final class AppHomeBuilder: Builder<AppHomeDependency>, AppHomeBuildable {
    
    public override init(dependency: AppHomeDependency) {
        super.init(dependency: dependency)
    }

    //AppHomeListener : AppHome 리블렛이 부모 리블렛에게 이벤트를 전달 할 떄 사용 한다.(단순 델리게이트 로직)
    public func build(withListener listener: AppHomeListener) -> ViewableRouting {
        let component = AppHomeComponent(dependency: dependency) // 로직 추가시 필요한 객체를 담는 바구니 역할
        let viewController = AppHomeViewController()
        let interactor = AppHomeInteractor(presenter: viewController) //비즈니스 로직이 들어가는 두뇌 역할
        interactor.listener = listener
        
        let transportHomeBuilder = TransportHomeBuilder(dependency: component)
        //router 를 리턴하는데 이렇게 리턴된 값은 부모 리블렛이 사용한다.
        //router 는 리블렛간 화면 이동을 담당하며, RIBS 의 수직구조로 자식과 부모간 리블렛을 뗏다 붙였다 할 수 있다.
        return AppHomeRouter(
            interactor: interactor,
            viewController: viewController,
            transportHomeBuildable: transportHomeBuilder
        )
    }
}
```

### 2. AppRoot 는 최상위 리블렛
* 자식 리블렛으로 Home, Financial, Profile 을 지닌다.
* attachChild -> setViewControllers
* 각각의 라우터로 부터 viewControllable 을 받아와서 탭바 컨트롤러에 샛팅 하고 있다.
```swift
import ModernRIBs

protocol AppRootInteractable: Interactable,
                              AppHomeListener,
                              FinanceHomeListener,
                              ProfileHomeListener {
    var router: AppRootRouting? { get set }
    var listener: AppRootListener? { get set }
}

protocol AppRootViewControllable: ViewControllable {
    func setViewControllers(_ viewControllers: [ViewControllable])
}

final class AppRootRouter: LaunchRouter<AppRootInteractable, AppRootViewControllable>, AppRootRouting {
    
    private let appHome: AppHomeBuildable
    private let financeHome: FinanceHomeBuildable
    private let profileHome: ProfileHomeBuildable
    
    private var appHomeRouting: ViewableRouting?
    private var financeHomeRouting: ViewableRouting?
    private var profileHomeRouting: ViewableRouting?
    
    init(
        interactor: AppRootInteractable,
        viewController: AppRootViewControllable,
        appHome: AppHomeBuildable,
        financeHome: FinanceHomeBuildable,
        profileHome: ProfileHomeBuildable
    ) {
        self.appHome = appHome
        self.financeHome = financeHome
        self.profileHome = profileHome
        
        super.init(interactor: interactor, viewController: viewController)
        interactor.router = self
    }
    
    func attachTabs() {
        let appHomeRouting = appHome.build(withListener: interactor)
        let financeHomeRouting = financeHome.build(withListener: interactor)
        let profileHomeRouting = profileHome.build(withListener: interactor)
        
        attachChild(appHomeRouting)
        attachChild(financeHomeRouting)
        attachChild(profileHomeRouting)
        
        let viewControllers = [
            NavigationControllerable(root: appHomeRouting.viewControllable),
            NavigationControllerable(root: financeHomeRouting.viewControllable),
            profileHomeRouting.viewControllable
        ]
        
        viewController.setViewControllers(viewControllers)
    }
}
```

### 3. 정리
* 하나의 로직의 단위는 라우터, 빌터, 인터렉터, 뷰로 이루어 지며 그 단위를 리블렛이라 부른다.
* 빌더는 리블렛 객체들을 생성하고 관리하고 라우터를 리턴 한다.
* 인터렉터는 비즈니스 로직을 담는 두뇌 이다.
* 라우터는 리블렛 트리를 만들고 뷰의 라우팅 역할을 수행하며, 자식 리블렛을 붙이고 싶을 때 자식 리블렛의 빌더를 만들고 빌더 메소드를 통해 라우터를 받아와 어태치 차일드를 하고 UI 를 띄운다.
