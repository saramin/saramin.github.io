---
title: 사람인 Android App Refactoring
author: 한동훈
tags:
- android
- kotlin
---

안녕하세요.
사람인HR 서비스인프라개발팀 안드로이드 앱 개발 담당 한동훈입니다.
해당 내용은 지난 2021년에 아이엠그라운드앱 구조개선을 위해 진행했던 앱리펙토링 공유하는 포스팅입니다. 

<br/>
## 필요성?
기존에 서비스중인 아이엠그라운드 앱은 MVC패턴구조로 아래와 같은 문제점들이 있었습니다.

<br/>
#### 기존 문제점
- MVC패턴을 사용하고 있어 Activity, Fragment가 Controller역할을 해서 view, model에 대해 프로세스를 정의하기 때문에 프로젝트가 진행될수록 코드가 비대해지고, 그 결과 가독성이 떨어지고 서비스가 확장될수록 유지보수의 비용이 많이 발생함. 
- Activity마다 같은 로직을 사용할 경우 각각에 구현되기 때문에 중복이 발생하고 수정사항 발생시 관련 부분 전부를 수정해야 하며 뷰와 종속적이기 때문에 화면마다 데이터와 뷰간의 로직이 상이하여 로직에 대한 재검증이 필요함. 
- repository와 api인터페이스가 비지니스로직 특성에 따라 구분되어 있지 않고, 하나의 class에 모두 나열되어 있음.
- 구조상 클래스간 의존성, 결합도가 높음.
- dataSource영역의 코드들이 구분없이 한 class와 interface에 정의되어 있어 분석에 비효율적
- 비지니스로직이 분리되어 있지 않아 UnitTest가 어려움.

<br/>

그래서 이를 해결하고자 리팩토링에서 MVVM패턴을 채용하였고,  MVVM패턴 구성과 함께 최적화되어 사용할수 있는  AAC(Android Architecture Components), DI(Depecndency Injection) dagger-hilt, DataBinding, ViewBinding, RXjava를 적용하여 개선하였습니다.

<br/>
#### 적용내용
- MVVM패턴(AAC Componnents(LiveData, Lifecycle, ViewModel, Room) 와 조합)
- Dagger-hilt
- DataBinding, ViewBinding
- RxJava

<br/>
## 1. MVVM
### 1) MVC 와 MVVM 차이
MVC는 안드로이드 개발에서 화면에 기본이 되는 Activity, Fragment가 직접 Controller가 되어 model과 뷰를 조합하여 화면을 구성하고 유저의 인터렉션을 정의하는 구조입니다. 그래서 controller에서 타이밍상 호출순서에 따라 화면이 갱신되지 않거나 하는경우도 있으며, 생명주기에 영향을 받기 때문에 메모리릭 발생확률도 있어 신경써주어야 합니다.

 반면 MVVM의 경우 Activity, Fragment는 뷰의 역할만 담당하고 viewModel에서 repository를 통해 비지니스로직의 정제된 결과값 데이터를 Rx나 liveData같이 observing 가능한 형태로 반환하고, 해당 viewModel의 결과를 사용하고자 하는 view에서 viewModel의 결과값을 observe(관찰)하여 비지니스 로직이 어떤구조로 돌아가는지 알필요없이 결과데이터만 가지고 화면을 구성할수 있게 되는 방식입니다. 이경우 데이터를 observe하는 형태이기 때문에 데이터가 변경시 화면을 구성하기 되어 타이밍문제로 누락이 발생하지 않고, 생명주기에 영향을 받지 않기 때문에 메모리릭 문제에서 자유로울수 있는 장점이 있습니다.

<br/>
### 2) MVVM
MVVM은 Model-View-ViewModel의 줄임말이고, 간단히 view(activity,fragment)는 화면 UI부분이고 viewModel 데이터를 관찰하여 화면을 갱신하는 역할을 하고, model은 요청받은 데이터를 반환하는 역할이고(DB,Api를 통해), viewModel은 view와 model사이에서 view가 요청한 데이터를 model에 요청하고 필요에 따라 중간에서 가공해서 받은데이터를 view과 관찰할수 있는 데이터로 보관하고 있는 역할을 합니다.

<br/>
#### 동작방식
<p align="center">
	  <img src="{{site.url}}/img/neyo/neyo_img_mvvm_v2.png" width="640px" height="193px" />
</p>

이미지 출처 :  [https://ko.wikipedia.org/wiki/모델-뷰-뷰모델](https://ko.wikipedia.org/wiki/모델-뷰-뷰모델)

<br/>
### 3)  AAC(Android Architecture Component) 
AAC(Android Architecture Component)는 안드로이드에서 제공하는 아키텍쳐로 LiveData, ViewModel, Room이 있고, AAC가 MVVM패턴에 최적화되어 있어서 리펙토링시 이부분도 도입하였습니다. 이중에 AAC의 ViewModel의 명명으로 인해 MVVM패턴 개념의 ViewModel과 이름이 같아 헷갈릴수 있는 부분이 있었는데, AAC에서 제공하는 ViewModel은 아래 그림과 같습니다.

<p align="center">
	  <img src="{{site.url}}/img/neyo/neyo_img_aac_pic.png" width="459px" height="506px" />
</p>


이미지 출처 : [https://developer.android.com/topic/libraries/architecture/viewmodel](https://developer.android.com/topic/libraries/architecture/viewmodel)


MVVM의 ViewModel 개념은 view와 model사이에서 데이터를 관리하는 역할을 이라면, AAC의 viewModel의 경우 mvvm패턴에서 말하는 viewmodel과 다르게, 화면 회전이나 화면변화로 redraw하거나할때 데이터가 유실되어 초기화 될수가 있는데 그런 환경에서 데이터를 보관하고 화면에 대한 Lifecycle을 알고있어 개발자가 별도 관리 및 처리 필요없이 OnDestroy될때 onClear함수로 데이터 해제 역할을 담당합니다. MVVM패턴의 ViewModel과 AAC에서 말하는 ViewModel은 차이가 있지만 AAC의 viewmodel을 mvvm의 ViewModel처럼 구성하여 사용하면 패턴과 AAC의 유용한 기능을 함께 사용할수 있기 때문에 이번에 도입하게 되었습니다.

``` java
open class BaseViewModel(
    private val globalErrorSubject: RxGlobalErrorSubject = RxGlobalErrorSubject
) : ViewModel(){

    protected val _progressDialogVisibilityState = PublishSubject.create<Boolean>()
    val progressDialogVisibilityState: Observable<Boolean> = _progressDialogVisibilityState.observeOn(AndroidSchedulers.mainThread())

    protected val compositeDisposable = CompositeDisposable()
    protected val networkErrorMsg = "오류가 발생했습니다.\\n새로고침 해주세요."

    fun addDisposable(disposable: Disposable) {
        compositeDisposable.add(disposable)
    }

    override fun onCleared() {
        compositeDisposable.clear()
        super.onCleared()
    }

    fun <T> Single<T>.subscribeDefault(
        consumer: Consumer<T>,
        errorConsumer: (Throwable) -> Unit,
        showProgress: Boolean = true
    ): Disposable = if (showProgress) {
        this.doOnSubscribe {
            _progressDialogVisibilityState.onNext(true)
        }.doOnEvent { _, _ ->
            _progressDialogVisibilityState.onNext(false)
        }
    } else {
        this
    }.subscribe(consumer, BaseErrorConsumer(errorConsumer))

        .
        .
        .
}

```

적용한 ViewModel은 개발자가 Activity,Fragment의 생명주기와 메모리릭 신경을 쓰지 않도록  AAC ViewModel을 상속받아 구현하였고, 비동기 처리부분을 addDisposable로 등록하고 동작중 화면종료가 발생하였을때 메모리해제되도록 onCleared에 compositeDisposable.clear()를 정의하였습니다.



``` java
@HiltViewModel
class CategorySelectViewModel @Inject constructor(
    private val repository: CategorySelectRepository
) : BaseViewModel() {

    val companyInfoListState          = PublishSubject.create<Array<CompanyInfoListResponse>>()
    val jobCategoryResponseModelState = PublishSubject.create<JobCategoryResponseModel>()
    val jobCodeResponseModelState     = PublishSubject.create<JobCodeResponseModel>()

        .
        .
        .

    val errorState                    = PublishSubject.create<ErrorState>()

    fun getCompanyInfoList() {
        addDisposable(repository.getCompanyInfoList()
            .subscribeDefault({
                companyInfoListState.onNext(it.body() ?: arrayOf())
            },{
                errorState.onNext(ErrorState.GetCompanyInfoList(it.message!!))
            }))
    }

    fun getJobCategory() {
        addDisposable(repository.getJobCategory()
            .subscribeDefault({
                jobCategoryResponseModelState.onNext(it.body() ?: JobCategoryResponseModel())
            },{
                errorState.onNext(ErrorState.GetJobCategory(it.message!!))
            }))
    }
		
}

```

각 ViewModel들은 이 BaseViewModel들을 상속받았고, 예시에서처럼 Api 호출후하거나, DB를 호출하여 결과값을 정제하고 RxJava PublishSubject클래스 형태로 담아둡니다. 그리고 해당 viewModel의 데이터를 이용하는 view에서는 결과값들을 subscribe하도록 설정해두고 UI구성할수 있도록 구성하였습니다. 


``` java

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        .
        .
        .

        initViewModel()
        viewModel.getJobCategory()
        .
        .
        .
    }

    private fun initViewModel() {
        compositeDisposable += viewModel.companyInfoListState.subscribe {
            i("")
            majorCompanyInfoMap = it.filter { it.comScleCd == "1" }.map { it.comNm to it.comNo }.toMap()
            midsizeCompanyInfoMap = it.filter { it.comScleCd == "2" }.map { it.comNm to it.comNo }.toMap()
            publicCompanyInfoMap = it.filter { it.comScleCd == "3" }.map { it.comNm to it.comNo }.toMap()
            binding.majorCompanyTab.performClick()
        }

        compositeDisposable += viewModel.jobCategoryResponseModelState.subscribe { model ->
            i("")
            model.data?.let { list ->
                jobCategoryMap = list.map { it!!.codeNm to it.code }.toMap()
            }
            initJobSelectView()
        }
         
        compositeDisposable += viewModel.paymentApplyState.subscribe { model ->
            i("")
            model.prdReanQty?.let {
                if (it > 0) {
                    goVideoInterviewGuideActivity()
                } else {
                    var msg   = "응시횟수가 부족합니다.\n응시횟수를 충전해 주시기 바랍니다."
                    Alert.showCustomAlert(context = this, msg = msg, confirmText = "충전하기", cancelText = "취소",
                                          confirmListener = {
                                              val purchaseListActivityIntent = Intent(this@CategorySelectActivity, PurchaseListActivity::class.java)
                                              startActivity(purchaseListActivityIntent)
                                          })
                }
            }
        }
        .
        .
        .
    }

```

<p align="center">
	  <img src="{{site.url}}/img/neyo/neyo_img_di_package.png" width="321px" height="435px" />
</p>

<br/>
## 2. DI(Dependency Injection) Dagger-hilt 도입
DI는 말그대로 Dependency Injection(의존성 주입)으로 객체간 의존성을 없애고 필요로 하는 객체를 외부에서 생성해서 필요한부분에서 주입해서 사용하는 개념입니다.
예를들어 아래와 같은 Car객체의 경우 HyundaiEngine()이라는 클래스를 가져다 쓰고 있는것이 의존성이고
실제 코드를 작성하다보면 이런부분이 자주 발생하기도 됩니다. 이럴경우 만약 Car를 생성하려고 할때 HyundaiEngine()이 아닌 BenzEngine()을 쓰는 차를 만들고 싶다고 하면 또다른 Car객체를 만들고 
BentzEngine()을 사용하는 또다른 Class를 생성하게 됩니다.

``` java
class Car {
  private val engine = HyunDaiEngine()
  private val tire   = KHTire()
}

class Car {
  private val engine = BenzEngine()
  private val tire   = KHTire()
}
```

이럴경우 Car클래스는 그대로 두고 외부에서 Engine을 생성해서 주입하게 되면 여러 Car객체를 별도로 만들 필요가 없이 Engine을 extend한 회사별 Engine을 외부에서 생성해서 주입해주면 됩니다. 이렇게 해서 클래스간 의존성을 줄이고 엔진을 바꾼다던가 , 타이어를 바꾼다던가 할때 차 전체를 수정할 필요없이 각 부품들만 수정해서 주입해주면 되기 때문에 효율적입니다.


``` java 
open class Engine() {

}

class HyunDaiEngine() : Engine() {

}

class Car(engine : Engine) {
  fun drive() {
      engine.drive()
  }
}

fun main() {
    val hyunDaiEngine = HyunDaiEngine() 
    val car = Car(hyunDaiEngine)
    car.drive()

    val benzEngine = BenzEngine()
    val benz = Car(benzEngine)
    benz.drive()
}
```

 위와 같이 문제점들을 없애고, 유지보수의 효율성을 증대하기위해 구글에서는 dagger2오픈소르를 밀고 있었는데, 기존사용방식의 경우 초기세팅이 어렵고 사용하려면 러닝커브가 높은편이라 쉽지 않았지만, 구글에서는 이런 초기 세팅의 어려움을 없애고, 쉽게 사용할수 있도록 android전용 DI 라이브러리인 dagger-hilt라이브러리를 발표하였고, 현재 리팩토링할때 사용하게 되었습니다. Dagger-hilt를 이용해서 위와같은 과정들이 자동화되었습니다. 
 

``` java  
// 기존 API 인터페이스 형태, 단일 Retofit library 생성방식 
interface Api {

    @POST("/api/.../...-auth")
    fun postAppAuths(@Body parameters: HashMap<String, Any>): Single<Response<ResponseBody>>
                   
                   ........


    @POST("/api/.../..../...-preview-connect")
    fun putConnect(
            @Header("Authorization") authorization: String,
            @Body parameters: HashMap<String, Any>)
            : Single<Response<ResponseBody>>

}


class RetroApi {

    companion object {

        private fun defaultRetrofit(): Retrofit {
            return Retrofit.Builder()
                    .baseUrl(App.getInstance().gPref.baseServer)
                    .addConverterFactory(GsonConverterFactory.create())
                    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                    .client(createOkHttpClient())
                    .build()
        }


        fun create(): Api {
            return defaultRetrofit().create(Api::class.java)
        }

        .
        .
        .
    }
}

```

위와 코드를 보면 기존에는 api 코드를 보면 하나의 Api 인터페이스에 모든 api코드가 다 정의되어 있고, api통신을 할때 사용하는 retrofit클래스를 전역에서 접근할수 이도록 구성되어 있었습니다. 그래서 한 class나 interface가 비대해져 가독성이 떨어지고 분석에 효율성이 떨어졌는데, 이부분을 해결하고 DI를 효율적으로 이용하고자 Api인터페이스를 특성별로 카테고리화하여 분산 시켰습니다. 

<p align="center">
	  <img src="{{site.url}}/img/neyo/neyo_img_package.png" width="321px" height="435px" />
</p>

<br/>
 그리고 각 비지니스 로직별 repository가 생성될때 필요하게될 각각의 Api 인터페이스를 외부주입 될수 있도록 DI를 이용하였고, 외부주입에 필요한 클래스나 인터페이스들을 ApplicaionModule에 명세하였습니다. Api통신에 사용하는 retrofit라이브러리의 경우 기본세팅을 하여 정의하였고, 각각의 Api별로 매핑이 달라져서 주입되어야 하므로, Api인터페이스들의 형태를 정의하였습니다.
``` java
@Module
@InstallIn(SingletonComponent::class)
class ApplicationModule {

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(App.getInstance().gPref.baseServer)
            .addConverterFactory(NullOnEmptyConverterFactory())
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(LiveDataCallAdapterFactory())
            .addCallAdapterFactory(RxJava3CallAdapterFactory.createSynchronous())
            .client(okHttpClient)
            .build()
    }

    @Provides
    @Singleton
    fun provideOkHttpClient(headerInterceptor: Interceptor): OkHttpClient {
        val interceptor = HttpLoggingInterceptor()

        if (App.getInstance().gPref.isDebug) {
            interceptor.level = HttpLoggingInterceptor.Level.BODY
        } else {
            interceptor.level = HttpLoggingInterceptor.Level.NONE
        }

        return OkHttpClient.Builder()
            .addNetworkInterceptor(interceptor)
            .addNetworkInterceptor(StethoInterceptor())
            .addInterceptor(headerInterceptor)
            .build()
    }

        .
        .
        .

    @Provides
    @Singleton
    fun providePreferences() : PreferencesInterface = App.getInstance().gPref

    @Provides
    @Singleton
    fun provideCommonApis(retrofit: Retrofit) : CommonAPIs       = retrofit.create(CommonAPIs::class.java)

    @Provides
    @Singleton
    fun provideAppStartApis(retrofit: Retrofit) : AppStartAPIs   = retrofit.create(AppStartAPIs::class.java)

    @Provides
    @Singleton
    fun provideMessageDatabase(@ApplicationContext context: Context): MessageDatabase = MessageDatabase.getInstance(context)

    @Provides
    @Singleton
    fun provideMessageDao(db: MessageDatabase) : MessageDao = db.messageDao()

    @Provides
    @Singleton
    fun provideMessageLocalRepository(dao: MessageDao) : MessageLocalRepository = MessageLocalRepository(dao)

    @Provides
    @Singleton
    fun provideSocketManager() : SocketManager = SocketManager()
        .
        .
        .
}

```

위처럼 api외에도 외부주입때 필요한 DB접근 Dao, socket처리를 담당하는 SocketManager등을 프로젝트가 확장될때마다 추가하여 진행하였습니다. 그리고 화면을 구성하는 Activity,Fragment에서는 viewModel에서 이용하는 repository들이 객체주입이 필요하므로 @AndroidEntryPoint를 써서 자동 member injection되도록 구성하였고, 


* Repository란? : 데이터 출처(로컬 DB인지 API응답인지 등)와 관계 없이 동일 인터페이스로 데이터에 접속할 수 있도록 만드는 것을 Repository 패턴이라고 한다. 레포지토리는 데이터 소스에 액세스하는 데 필요한 논리를 캡슐화하는 클래스 또는 구성 요소이다.


``` java
@AndroidEntryPoint
class CategorySelectActivity : BaseActivity<ActivityCategorySelectBinding>(), View.OnTouchListener {

    private val viewModel: CategorySelectViewModel by viewModels()

    override val layoutResID: Int = R.layout.activity_category_select
    override val color: Int       = R.color.white

    private var itvTpCd     = ""   
    private var crerStCd    = ""   
    private var jobCategory = ""    
    private var octpCd      = "" 
        .
        .
        .
}

```


viewModel에서는  repository클래스들을 이용해서 Datasource에 접근하는데, 각 repository의 경우 성격에 따라 카테고리화한 Api 종류에따라 필요한 Api나 DB접근을 위한 Dao등을 생성자로 넣어줘야할 필요성이 있는데 이부분에서 @Inject 어노테이션을 정의해 주므로써 ApplicationModule에서 미리 정의한 모듈들이 개발자가 그때그때 직접생성할 필요없이 자동주입되도록 적용하였습니다.


``` java
@HiltViewModel
class CategorySelectViewModel @Inject constructor(
    private val repository: CategorySelectRepository
) : BaseViewModel() {

    val companyInfoListState          = PublishSubject.create<Array<CompanyInfoListResponse>>()
    val jobCategoryResponseModelState = PublishSubject.create<JobCategoryResponseModel>()
    val jobCodeResponseModelState     = PublishSubject.create<JobCodeResponseModel>()
    val partnershipState              = PublishSubject.create<PartnershipCertifyInfoResponseModel>()
    val paymentApplyState             = PublishSubject.create<PaymentApplyResponseModel>()
    val errorState                    = PublishSubject.create<ErrorState>()

    fun getCompanyInfoList() {
        addDisposable(repository.getCompanyInfoList()
            .subscribeDefault({
                companyInfoListState.onNext(it.body() ?: arrayOf())
            },{
                errorState.onNext(ErrorState.GetCompanyInfoList(it.message!!))
            }))
    }
        .
        .
        .
}
```


``` java
class CategorySelectRepository @Inject constructor(
        private val api         : CategorySelectApi,
        private val commonApi   : CommonAPIs,
        private val pref        : PreferencesInterface = App.getInstance().gPref
) : CommonRepository(api = commonApi, pref = pref) {

        fun getCompanyInfoList(): Single<Response<Array<CompanyInfoListResponse>>> {
                return execute(api.getCompanyInfoList())
                        .subscribeOn(Schedulers.io())
                        .observeOn(AndroidSchedulers.mainThread())
        }

        fun getJobCategory(): Single<Response<JobCategoryResponseModel>> {
                return execute(api.getJobCategory())
                        .subscribeOn(Schedulers.io())
                        .observeOn(AndroidSchedulers.mainThread())
        }

        .
        .
        . 
}
```

<br/>
## 3. databinding, viewbinding 
기존 프로젝트에서는 화면 레이아웃을 그린 xml을 Activity나 Fragment에서 사용할때 아래처럼 
findViewById를 이용해 연결해서 사용하였는데 리팩토링전에는 kotlin-android-extension을 사용하고 있었고, 이 라이브러리가 같은 id값 사용하고 해당 xml이아닌 잘못된 xml의 동일 id값으로 연결될 가능성도 높고 이럴경우도 에러가 나야하지만 빌드가 되기 때문에 런타임 에러 발생확률이 높으며, recyclerview에서 viewHolder를 구성할때 사용할수 없는 문제들이 있어 deprecated되어 교체하게 되었습니다.
``` java
//xml파일 activity_main.xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    tools:context=".ui.mypage.profile.ProfileActivity">

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#fff"
        android:fitsSystemWindows="true">

            <TextView
                android:id="@+id/tvName"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                tools:text="테스트" />

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        //과거 방식
        findViewById<TextView>(R.id.tvName).text = "Hi"
        //리팩토링 전방식 android-extension라이브러리 사용 
        //class상단 import kotlinx.android.synthetic.main.activity_main.*
        //xml id직접연결가능 tvName이 다른화면의 activity_sample등 다른xml에 존재시
        //연결할수 있음 대신 구동시 런타임에러가 나타남.
        tvName.text= "Hi"
        
    }
        .
        .
        .
}

```

리팩토링시 mvvm패턴을 도입하였기 때문에 관련 라이브러리를 사용할때는 databinding,호환성이 좋아 교체 하였고, xml과 연동하여 사용하였습니다. viewBinding은 양방향 바인딩이 지원되지 않아(xml과 연동사용하는 방식) 이런 구현이 필요없는 부분에서는 viewBinding을 사용하였습니다. 


``` java
abstract class BaseAppCompatActivity<T: ViewDataBinding>: AppCompatActivity() {
    lateinit var binding: T
    abstract val layoutResID : Int

        .
        .
        .
    
    override fun onCreate(savedInstanceState: Bundle?, persistentState: PersistableBundle?) {
        super.onCreate(savedInstanceState, persistentState)
        onWindowFocusChanged(true)
        setBinding()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        window.allowEnterTransitionOverlap = true
        onWindowFocusChanged(true)
        setBinding()
    }

    private fun setBinding() {
        binding = DataBindingUtil.setContentView(this, layoutResID)
        binding.lifecycleOwner = this@BaseAppCompatActivity

        .
        .
        .
    }
}

@AndroidEntryPoint
class ChatRoomActivity : BaseAppCompatActivity<ActivityChatRoomBinding>() {

    override val layoutResID = R.layout.activity_chat_room

    private val  viewModel: ChatRoomViewModel by viewModels()
    lateinit var adapter: ChatRoomAdapter
    
        .
        .
        .
  
    override fun onCreate(savedInstanceState: Bundle?) {
        .
        .
        .
        binding.lifecycleOwner = this
        binding.viewModel = this.viewModel
    }
}
```


 모든 activity에서 사용하는 부분을 공통으로 정의한 Base를 만들고 각 Activity나 Fragment들을 해당하는 클래스를 상속하여 사용하였고, 공통적으로 databing을 하기 때문에 
layoutResID와 Generics로 Binding될 xml파일을 정의하고 DataBindingUtil을 통해 
contentView를 연결하도록 하고, xml에 liveData를 연동해서 사용할수 있는부분이 있기 때문에 lifecycleOwner를 명시해주었습니다. 그리고 본 Activity에서는 xml에 viewModel을 연결하거나 데이터를 연결하여 xml상에서 relay가능하도록 구현하였습니다.


``` xml

    <data>
        <import type="androidx.databinding.ObservableArrayList" />
        <import type="kr.co.saramin.img.network.model.response.VideoInterviewApplyResponseModel.TemVideo" />
        <variable name="completeVideoList" type="ObservableArrayList&lt;TemVideo&gt;" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:fitsSystemWindows="false">

        .
        .
        .

        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/playList"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_marginLeft="24dp"
            android:elevation="10dp"
            android:scrollbars="vertical"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintTop_toBottomOf="@id/playListTitleContainer"
            tools:bindCompletedVideoList="@{completeVideoList}"
            tools:listitem="@layout/completed_video_list_item" />

        .
        .
        .
    
    =======================================

    <data>
        <variable
            name="viewModel"
            type="kr.co.saramin.img.ui.chatroom.declare.DeclareViewModel"
            />
         <variable
            name="content"
            type="kr.co.saramin.img.network.model.socketmessage.AutoMsgTypeCancel"
            />
    </data>
        .
        .
        .
        <LinearLayout
            android:id="@+id/radioButton1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginStart="48dp"
            android:layout_marginTop="32dp"
            android:gravity="center_vertical"
            android:onClick='@{() -> viewModel.setSelectedState(102)}'
            android:tag="102"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toBottomOf="@id/titleText"
            > 
        .
        .
        .
        <TextView
        android:id="@+id/messageImageView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:background="@drawable/shape_background_rounded_rect_message_system"
        android:paddingHorizontal="8dp"
        android:gravity="center_horizontal"
        android:includeFontPadding="false"
        android:fontFamily="@font/noto_sans_regular"
        android:textColor="#5a657f"
        android:letterSpacing="-0.04"
        app:htmlText="@{content.header}"
        tools:htmlText="@{content.header}"
        />
 
        .
        .
        .

```


그래서 기존에 위의 예시처럼 xml과 databinding을 연결하여 실시간 반영될수 있도록 구성하던가, 
listner를 바로 연결하여 동작을 실행하도록 연결하는 방식들을 적용하였습니다. 위의 방식을 조합하므로써 구현해야할 코드량을 여러부분줄일수 있었습니다.

그리고 양방향 binding이 필요없는 부분에서는 viewBinding을 사용하였습니다. viewBinding과 databinding을 비교해보면 일단 ViewBinding이 빌드속도가 더 빠르고 xml에 <layout> 태그가 필요없다는 장점이 있지만 xml과 data를 연동해서 사용하는 위와같은 방식들을 지원하지 않아, 필요에 따라 프로젝트에는 섞어서 사용하였습니다.

<br/>
## 4. 프로젝트 구성
	
<p align="center">
	  <img src="{{site.url}}/img/neyo/neyo_img_mvvm_structure.png" width="722px" height="618px" />
</p>

	
  지금까지 위에서 부분적으로 설명한 적용된 MVVM 구조화된 형태를 보면 위의 그림과 같은 형태로 구성하였습니다. 다만 livedata를 사용하는 부분은 기존에 비슷하게 지원하는 RxJava3로 사용하고 있었기 때문에 대체 하여 사용하였습니다. 
  화면구성을 담당하는 Activity/Fragment에서 비지니스로직이 필요한 viewModel을 연동하고 
  viewModel에서는 repository인터페이스를 통해 데이터를 api나 DB등을 접근하여 가져오도록 하고 결과 데이터를 viewModel이 결과데이터를 가지고 있고, Activity,Fragment들은 화면구성에 필요한 결과데이터들을 관찰(Observing)하여 데이터의 변화가 있으면 즉각 미리정의해둔 UI구성이 적용되도록 구현하였습니다.  

<br/>
## 5. 마무리하며
	
 리팩토링을 통해 서비스 중인 앱의 구조를 변경해 볼 수 있는 좋은 기회를 얻었고, 당시 신규 라이브러리 등을 적용해볼 수 있어 뜻깊은 시간이었던거 같습니다. 보통 서비스가 확장되다 보면 복잡도가 올라가고 비대해져서 분석에 시간도 더 걸리고 유지보수도 비용도 점차 많이 들게 되기 마련인데, 이번 구조 개선할 기회가 생겨서 좀 더 세분화하고 유지보수가 용이하게 할 수 있게 되어 이후 개발에도 많은 도움이 되었습니다. 현재도 앱의 성능을 올릴 수 있고 체계적으로 개발할 수 있는 방법론이나 라이브러리들이 나오고 있는데 점차 현재 서비스에 적용하여 발전할 수 있도록 노력하도록 하겠습니다.

긴글 읽어주신 모든분들께 감사드립니다.

	
<br/>
### 참고자료
[https://docs.microsoft.com/ko-kr/xamarin/xamarin-forms/enterprise-application-patterns/mvvm](https://docs.microsoft.com/ko-kr/xamarin/xamarin-forms/enterprise-application-patterns/mvvm)<br/>
[https://developer.android.com/topic/libraries/architecture?hl=ko](https://developer.android.com/topic/libraries/architecture?hl=ko)
[https://developer.android.com/jetpack/guide?hl=ko](https://developer.android.com/jetpack/guide?hl=ko)
[https://developer.android.com/topic/libraries/architecture/viewmodel](https://developer.android.com/topic/libraries/architecture/viewmodel)
[https://developer.android.com/training/dependency-injection/hilt-android?hl=ko]([https://developer.android.com/training/dependency-injection/hilt-android?hl=ko)
[https://developer.android.com/topic/libraries/data-binding?hl=ko](https://developer.android.com/topic/libraries/data-binding?hl=ko)
[https://developer.android.com/topic/libraries/view-binding?hl=ko](https://developer.android.com/topic/libraries/view-binding?hl=ko)
