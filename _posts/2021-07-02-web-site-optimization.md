---
title: 웹 사이트 성능 최적화, 새로운 경험
subtitle: 사람인 & 베트남 Topdev 웹 사이트 성능 향상을 위해 적용한 방법은?
tags:
- lazyload
- HtmlMinimizer
- gtm
- Reflow
- SkeletonUI
- 성능최적화
---

안녕하세요. 사람인HR IT연구소 개발1팀 권종성입니다 :)

저는 웹 개발자로 사람인 채용포털 전반에 걸친 서비스를 개발하고 있으며, 이번에 좋은 기회로 사람인 모바일 페이지와 베트남 자회사인 앱랜서에서 IT직무에 특화된 채용포털 사이트인 [Topdev](https://topdev.vn) 웹 사이트 성능을 최적화하는 작업을 하게 되었습니다. 지금부터 기존에 어떤 성능 이슈들이 있었고, 어떻게 최적화를 하였는지 하나씩 알아보겠습니다.
<br><br><br>
# 시작은?
사람인HR IT연구소에는 '길드'라는 아주 특별한 모임이 있습니다. '길드'는 IT연구소 연구원들의 주도하에 실험적인 도전이나 다양한 문제들을 해결하기 위해 결성되는 모임인데요. 저는 '모바일 서비스 성능개선' 길드원을 모집한다는 공지를 보고선 '뭔지는 잘 모르겠지만.. 뭔가 멋진데?'라는 생각과 함께 참여하겠다는 메일을 보내고 길드에 참여하게 되었습니다(속전속결!).

결과적으로 여러 개선 방법들을 통해 웹 사이트를 최적화하여 사용자에게 더 나은 서비스를 경험할 수 있도록 하였고, 저 또한 웹 개발자로서 한층 더 성장하는 뜻 깊은 시간이었습니다.
<br><br><br>
# 현재의 문제들

| Page Load Time(seconds) | Bounce Rate(%) |
| -------- | -------- |
| 1     | 7     |
| 2     | 6     |
| 3     | 11     |
| 4     | 24     |
| 5     | 38     |
| 6     | 46     |
| 7     | 33     |
| ...     | ...     |

[Does Page Load Time Really Affect Bounce Rate?(2018)](https://www.pingdom.com/blog/page-load-time-really-affect-bounce-rate/) 포스트에서는 페이지 로드 시간에 따른 사용자 이탈률의 상관관계에 대한 내용이 작성되어 있습니다. 특히 페이지 로드까지 **4초**가 소요되면 이탈률은 **24%**로 급격히 증가한다고 나와 있는데요. 이와 같이 사용자는 페이지의 응답, 콘텐츠의 노출, 사용자가 마우스나 손을 통해 페이지를 동작할 수 있을때까지 4초를 넘어가게 되면 사용자는 적지 않은 불쾌감을 느끼게 되며, 더욱 심각한 건 사이트에 대한 불신이 생길 수도 있습니다. 특히, 한번 이탈한 사용자는 언제 돌아올지 알 수 없습니다. 불쾌하더라도 정말 보고싶은 콘텐츠나 사용하고 싶은 서비스가 있기 전까진 말이죠.(좋은 서비스를 만드는 건 다른 얘기입니다!!)

![saramin main](/img/web-site-optimization1.png)

사람인 모바일 메인페이지는 접속자 수가 TOP 5 안에 들지만, 많은 콘텐츠와 기능들로 인해 무거워져 있어 최적화가 가장 시급하여 첫 최적화 작업 대상으로 사람인 모바일 메인페이지를 선정하였습니다.

![google analytics1](/img/web-site-optimization2.png)
google analytics에서 확인해본 결과 지난 한 달간 모바일 메인페이지의 평균 로드 시간은 **4.13초**, 이탈률(Bounce Rate)는 **5%** 대 였는데요. 위에서 소개한 이탈률과는 괴리가 있는데요. 이는 사람인은 많은 구직자가 접속하는 사이트로 사이트 성능이 좋지 않아도 그에 상응하는 구직과 관련된 콘텐츠와 서비스가 있어 참아내며 이용하는 것으로 추정하였습니다. 그렇다고 내버려 두면 안되겠죠?!

웹 페이지를 최적화 하는데 일반적으로 사용할 수 있는 방법들은 다양하게 있는데요. 저희는 프론트 엔드를 분석하여 아래와 같이 최적화 할 수 있는 것들을 검토하였습니다.
* 방문 페이지 HTML 압축
* 인라인 CSS, Javascript 파일로 통합
* Javascript 지연 로드
* 콘텐츠 지연 로드

이 외에도 CSS, Javascript를 webpack로 번들하여 여러 개의 파일을 하나로 묶어 리소스 요청을 줄이고, minify로 리소스 용량도 함께 줄이는 작업도 검토하였지만 작업량이 많은데 반해 최소한의 인원으로 빠르게 성과를 내기 위해 향후 다시 검토하기로 하였습니다.
<br><br><br>
# 상세 구현 내용

## 1. 방문 페이지 HTML 압축
사용자는 브라우저 주소창에 URL을 입력하면 서버에서 클라이언트의 화면을 구성할 수 있는 HTML을 응답하게 됩니다. 이 HTML에는 개발자가 유지보수하기 위해 필요한 주석이나 공백이 있지만, 이는 사용자에겐 필요하지 않습니다. 불필요한 주석, 공백을 제거하게 되면 HTML의 용량을 축소할 수 있으며 당연하게도 사용자는 더욱 적은 용량의 HTML을 다운로드 받기 때문에 결과적으로 초기 렌더링이 더 빨라 집니다.
![network1](/img/web-site-optimization3.png)

크롬 개발자 도구를 통해 압축 전 모바일 메인페이지 HTML 용량을 400kb로 확인하였습니다. 

사람인은 PHP 7.x와 Zend Framework로 서비스를 운영중인데요. WordPress의 [Better WordPress Minify](http://betterwp.net/wordpress-plugins/bwp-minify/)를 참고하여 개발한 HtmlMinimizer 플러그인은 응답하는 HTML을 정규식으로 불필요한 주석이나 공백을 찾아 제거합니다. 플러그인 코드는 최소한으로만 공개하는 점 양해 부탁드리며 Better WordPress Minify를 참고해 주시면 되겠습니다.([https://github.com/OddOneOut/bwp-minify/blob/master/min/lib/Minify/HTML.php](https://github.com/OddOneOut/bwp-minify/blob/master/min/lib/Minify/HTML.php))

{% highlight php linenos %}
class HtmlMinimizer extends Zend_Controller_Plugin_Abstract
{
    ...
 
    private function minimizeHtml($bodyContent)
    {
        $minimizedHtml = str_replace("\r\n", "\n", trim($bodyContent));
 
        // replace scripts (and minimize) with placeholders
        $minimizedHtml = preg_replace_callback(
            '/(\\s*)<script(\\b[^>]*?>)([\\s\\S]*?)<\\/script>(\\s*)/i'
            , [$this, 'removeScriptCB']
            , $minimizedHtml);
 
        ...
 
        // trim each line.
        $minimizedHtml = preg_replace('/^\\s+|\\s+$/m', '', $minimizedHtml);
 
        ...
 
        $this->getResponse()->setBody($minimizedHtml);
    }
}
{% endhighlight %}

![diff1](/img/web-site-optimization4.png)
결과적으로 HtmlMinimizer 플러그인을 적용하여 HTML의 리소스를 **400kb → 300kb** 로  **25%** 가까운 용량을 축소하였습니다.(손 안대고 코 푼 느낌이군요..)
백엔드에서 HTML 압축 방식은 대량의 HTML을 분석하여 제거하고 응답하기 때문에 서버의 부하가 발생할 수도 있고, 잘못된 처리로 인해 의도치 않게 요소가 노출 되지 않거나 기능이 동작되지 않는 문제가 있을 수 있습니다. 부하 테스트, 기능 테스트도 꼭 해줘야하는거 잊지 마세요!

Laravel Framework를 사용하신다면 DNS Prefetch를 포함한 다양한 기능이 있는 [laravel-page-speed](https://github.com/renatomarinho/laravel-page-speed)를 고려하면 좋을 거 같습니다. 이 외에도 웹서버(Apache, Nginx)에서 HTML을 minify 할 수 있는 방법도 있으니 검색해 보시면 좋을거 같네요!
<br><br>
## 2. 인라인 CSS, Javascript 파일로 통합
인라인 CSS와 Javascript 코드는 파일로 통합하여 브라우저 캐시 사용으로 최적화 할 수 있다면 방문 페이지 HTML의 용량을 축소할 수 있습니다.
일정상 모든 인라인 Javascript 코드를 파일로 통합할 순 없었지만 **13kb** 정도의 Javascript 코드를 파일로 통합하여 최적화하는데 기여했습니다.
<br><br>
## 3. Javascript 지연 로드
서버에서 응답한 HTML을 브라우저는 한줄 한줄 파싱해 나가기 시작합니다. 그러다 CSS나 Javascript 리소스를 만나게 되면 파싱을 중지하고 리소스를 다운로드하고 실행을 완료할 때까지 기다리게되어 결과적으로 렌더링 시간이 지연되게 됩니다.
그렇다고 CSS나 Javascript를 사용하지 않을 수는 없는데요. 이런 경우 페이지 로드가 완료된 후 로드해도 되는 리소스는 지연 로드하여 최적화할 수 있습니다.
분석한 결과 아래 서드파티 라이브러리를 페이지 로드 이후에 로드되도록 수정하여 최적화 할 수 있었습니다.
* Google Adsense
* Google Analytics
* Google Tag Manager
* Naver Analytics
* Saramin Wiselog

⚠️[Google Tag Manager 설치 가이드](https://developers.google.com/tag-manager/devguide)에서는 gtm.js를 <head>에 설치하는 것을 권장하고 있습니다. 페이지 로드 이후에 gtm.js를 로드한다는 건 페이지 로드 이후에 사용자의 행동을 수집하는 것입니다. 즉, 페이지 로드 이후 Google Tag Manager가 로드 되기 전까지는 사용자의 행동이 수집되지 않는 점 유의해 주세요.
	
페이지 로드가 완료되면 콜백이 실행되며 라이브러리를 로드하는 방식으로 구현해야 하는데, 이러면 비동기로 리소스를 로드하게 되어 만약 의존하는 라이브러리가 먼저 로드되지 않거나, 라이브러리가 로드된 이후에 실행해야하는 알고리즘이 있을 수 있는데요. 이러한 문제는 [LoadJS](https://github.com/muicss/loadjs) 라이브러리를 통해 해결할 수 있습니다.
{% highlight html linenos %}
<script src="//unpkg.com/loadjs@latest/dist/loadjs.min.js"></script>
<script>
window.addEventListener("load", function() {
	loadjs(['//wcs.naver.net/wcslog.js'], function() {
		// 리소스 로드가 완료된 후 실행
	
	});
});
</script>
{% endhighlight %}
loadJS() 함수의 첫 번째 인자는 의존하는 스크립트의 주소를 넣고, 두 번째 인자에는 스크립트의 로드가 완료되면 호출되는 콜백 함수를 추가하여 사용할 수 있도록 하였습니다.
이 외에 독립적으로 수행되는 라이브러리의 경우 페이지 로드가 완료된 후 로드되도록 구현하였습니다

{% highlight javascript linenos %}
// Google Tag Manager
window.addEventListener("load", function() {
    (function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
            new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
        j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
        'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
    })(window,document,'script','dataLayer','gtm_id');
});
{% endhighlight %}
<br><br>
	
## 4. 콘텐츠 지연 로드
사람인 모바일 메인 페이지에서는 아래와 같은 콘텐츠를 초기에 렌더링하고 있었습니다.
* 상단 영역
* 추천 영역 5개
* 광고 영역 7개
* 광고 배너 2개
* 기타 영역 3개
* 푸터 영역

우와! 직접 세어 보니 꽤 많네요! 게다가 특정 영역은 숨겨진 콘텐츠를 펼쳐볼 수 있거나, swiper로 탐색할 수 있도록 구현되어 있습니다.
![main2](/img/web-site-optimization5.gif)

저희는 메인페이지에 처음 접속했을때 보여질 상단 영역을 제외한 총 8개의 영역에 대해 Intersection Observer API로 브라우저 뷰포트가 지연 로드 될 콘텐츠 영역에 근접하게 되면 비동기 통신으로 해당 콘텐츠가 렌더링되도록 구현하기로 결정했습니다.

콘텐츠 지연 로드로 다음과 같은 이점을 얻을 수 있습니다.
* 방문 페이지(HTML) 리소스 축소
* 지연 로드 영역의 이미지 또한 함께 지연 로드
* 각 영역에 해당하는 javascript 알고리즘 또한 지연 실행
* 지연 로드 영역의 백엔드 알고리즘 또한 지연 실행
	
⚠️지연 로드는 Javascript로 동작하기 때문에 SEO에 최적화 되어야 하는 콘텐츠는 피하는게 좋습니다.
<br><br>
### 4-1. Skeleton UI로 콘텐츠 영역 준비하기

| 로드 전 | 로드 후 |
| -------- | -------- |
| ![skeleton-before](/img/web-site-optimization6.png)   | ![skeleton-after](/img/web-site-optimization7.png)   |

사용자의 네트워크 환경이나 기기의 사양, 서버의 응답속도와 같은 다양한 원인으로 인해 지연 로드되는 콘텐츠는 언제 화면에 노출될지 알 수 없습니다. 저희는 다양한 환경의 사용자를 위해 미리 어떤 콘텐츠가 노출될지 예상할 수 있으며, 암시적으로 '로드 중 입니다'를 인지할 수 있도록 하기 위해 Skeleton UI를 사용하기로 하였습니다.
그리고, Skeleton UI를 통해 지연 로드 영역을 미리 확보해 놓는다면 Reflow 발생을 방지할 수 있습니다.
	
우선 view파일의 태그에 있던 백엔드 코드를 제거하고 태그 내용은 텅 비워 두고, 영역을 관찰하기 위한 이벤트를 바인딩 하기 위해 class 속성에 'lazy-load'와 data-category 속성을 만들어 해당 영역의 키 값(예: platinum_m)을 추가하였습니다.
```html
<section class="content lazy-load" id="platinum-m" data-category="platinum_m">
    <h2 class="tit">인기 채용공고</h2>
    <div class="wrap_job wrap_job_platinum">
        <div class="item">
            <em class="subj"></em>
            <div class="wrap_desc">
                <span class="logo"></span>
                <strong class="corp"></strong>
                <div class="wrap_scrap_btn"></div>
                <span class="desc"></span>
                <span class="date"></span>
            </div>
        </div>
    </div>
    ...
</section>
```

CSS로 비워둔 공간에 대한 박스와 배경색을 채워줌으로서 간단하게 구현을 할 수 있었습니다.
{% highlight css linenos %}
/* 박스 배경은 회색으로 정의합니다. */
.wrap_job_platinum .item {background:#e1e1e3 50% 50% no-repeat;}
 
/* 비어있는 태그는 컨텐츠가 들어갈만한 임의의 크기로 사이즈를 지정하고 배경은 흰색으로 정의합니다. */
.wrap_job_platinum .subj:empty {width:50%;height:15px;background-color:#fff}
.wrap_job_platinum .corp:empty {margin-bottom:3px;padding-bottom:0;width:30%;height:27px;background-color:#fff}
.wrap_job_platinum .wrap_scrap_btn:empty {margin:10px 5px;padding:0;width:20px;height:20px;background-color:#fff}
.wrap_job_platinum .desc:empty {margin-top:1px;padding-top:0;height:19px;background-color:#fff}
.wrap_job_platinum .date:empty {width:45px;height:12px;background-color:#fff}
{% endhighlight %}
<br><br>
### 4-2. 콘텐츠 지연 로드 구현
콘텐츠 지연 로드 과정은 다음과 같습니다.

1. Intersection Observer API로 지연 로드 영역이 관찰되도록 바인딩
2. 뷰 포트가 지연 로드 영역에 근접하게 되면 콜백이 동작하여 비동기 통신 서버에 콘텐츠 호출
3. 서버에서 응답한 콘텐츠 HTML을 Skeleton UI와 대체
4. 추가된 콘텐츠에 맞는 이벤트 바인딩
5. Intersection Observer API의 unobserve() 메서드로 해당 요소 관찰 중지
	
실제 구현한 코드와는 조금 다르지만 불필요한 알고리즘은 제거하여 이해하기 쉬운 예제를 준비해봤습니다. 
{% highlight javascript linenos %}
// IntersectionObserver 인스턴스 생성
var observer = new IntersectionObserver(function(entries, observer) {
    // 등록된 요소 순회
    $.each(entries, function(index, entry) {
        if (entry.isIntersecting) { // 뷰포트가 영역 가시권에 들어왔을때
            var $target = $(entry.target);
            var category = $target.data('category');
 
            $.ajax('/get-section', {
                data: {category: category},
                success: function(html) {
                    // 응답받은 콘텐츠로 대체
                    $target.html(html);
                    // 이벤트 바인딩
                    bindSectionEvents(category);
                }
            });
        }
    });
}, {rootMargin: '300px'}); // 브라우저 뷰포트의 여백 설정
 
// 관찰해야할 요소 등록
$('.lazy-load').each(function() {
    observer.observe(this);
});
{% endhighlight %}
* 2줄: IntersectionObserver 생성자로 인스턴스를 생성합니다. 첫 번째 인자는 브라우저 뷰포트가 가시권에 들어왔을 때 호출되는 콜백 함수를 주입할 수 있으며, 두 번째 인자는 옵션을 설정할 수 있습니다.
* 4줄: 하나 이상의 등록된 요소를 순회 합니다.
* 5줄: 하나 이상의 등록된 요소 중 가시권에 들어온 요소인지 확인 합니다.
* 6~17줄: 가시권에 들어온 요소에 해당하는 알고리즘 수행합니다. 비동기 통신으로 응답받은 HTML 콘텐츠를 렌더링하고 이벤트 또한 바인딩합니다.
* 20줄: 옵션으로 뷰포트의 바깥 여백 설정을 설정합니다. 사용성 테스트를 통해 적절한 범위를 설정하시는게 좋습니다. 범위를 과하게 넓히면 불필요하게 영역이 미리 로드될 수 있습니다.
<br><br><br>
	
## 결과
### 1. 방문 페이지 HTML
![skeleton-after](/img/web-site-optimization8.png)
방문 페이지 HTML은 최적화 전 400kb에서 공백, 주석 제거와 콘텐츠 지연 로드를 통해 160kb로 줄여 약 **60%** 축소되었습니다.
<br><br>

### 2. 페이지 로드
**최적화 전**
![skeleton-after](/img/web-site-optimization9.png)

**최적화 후**
![skeleton-after](/img/web-site-optimization10.png)
**Javascript 지연 로드** 및 **콘텐츠 지연 로드**로 페이지를 최적화하여 더욱 빠르게 페이지 로드 되는것을 확인할 수 있습니다.
<br><br>
	
### 3. 구글 애널리틱스 수집 결과
![skeleton-after](/img/web-site-optimization11.png)
최적화 전과 후로 사용자의 평균 페이지 로드 속도가 4초에서 **2초대**로 더 빨라졌으며, 이탈률 또한 줄어든 것을 확인할 수 있습니다.
<br><br><br>
	
# 하나 더!
성공적으로 첫 최적화 작업을 완료하고 길드를 해체하곤 각자 평온한(?) 시간들을 보내고 있던 중에 이전 경험을 살려 베트남 자회사 앱랜서가 운영중인 [Topdev](https://topdev.vn)의 메인 페이지 최적화 프로젝트도 하게되었습니다!(이 길이 내 길인가?..)

[Topdev](https://topdev.vn)에서도 다음과 같은 최적화 작업을 하였는데요.
* 방문 페이지(HTML) 압축
* 컨텐츠 지연 로드
* 자바스크립트 지연 로드
* 동기 통신으로 인한 렌더링 지연

이중 위에서 설명한 것과 중첩된 내용은 제외하고 알아볼께요~
<br><br><br>

## 방문 페이지(HTML) 압축
topdev.vn 사이트는 PHP 7.x와 Laravel Framework로 서비스를 운영중이어서 laravel-page-speed 패키지로 아주 쉽게 HTML을 압축할 수 있었습니다.
github laravel-page-speed 페이지에 설치 가이드가 잘 나와 있는데요. 우선 메인페이지만 최적화 하기로해서 미들웨어 그룹에 정의하고 메인페이지 라우터에서 지정하였습니다.
{% highlight php linenos %}
// app/Http/Kernel.php
class Kernel extends HttpKernel
{
    ...
 
    protected $middlewareGroups = [
        ...
 
        'minify_web' => [
            \RenatoMarinho\LaravelPageSpeed\Middleware\InlineCss::class,
            \RenatoMarinho\LaravelPageSpeed\Middleware\ElideAttributes::class,
            \RenatoMarinho\LaravelPageSpeed\Middleware\InsertDNSPrefetch::class,
            //\RenatoMarinho\LaravelPageSpeed\Middleware\RemoveComments::class,
            //\RenatoMarinho\LaravelPageSpeed\Middleware\TrimUrls::class,
            //\RenatoMarinho\LaravelPageSpeed\Middleware\RemoveQuotes::class,
            \RenatoMarinho\LaravelPageSpeed\Middleware\CollapseWhitespace::class, // Note: This middleware invokes "RemoveComments::class" before it runs.
            //\RenatoMarinho\LaravelPageSpeed\Middleware\DeferJavascript::class,
        ],
    ];
}
	
// Modules/Home/Routes/web.php
Route::get('/', 'HomeController@index')->name('index')->middleware('minify_web');
{% endhighlight %}
<br><br><br>
	
## 동기 통신으로 인한 렌더링 지연
크롬 개발자 도구에서 사이트 퍼포먼스를 측정하여 확인 하던 중 특정 Javascript 알고리즘이 **500~700ms** 가까이 렌더링을 지연시키고 있는 이슈를 확인하였습니다.
![skeleton-after](/img/web-site-optimization12.png)

{% highlight javascript linenos %}
function getTokenValue(name) {
    var value;
    $.ajax({
        method: 'GET',
        url: '/ajaxs/' + name,
        async: false,
        success: function(res) {
            value = res.data;
        }
    });
    return value;
}
 
var token = getToken('token');
var user_token = getToken('user_token');
{% endhighlight %}
getTokenValue() 함수를 확인해 보니 jQuery ajax의 async 옵션을 **false**로 설정하여 동기적으로 서버와 통신하여 통신이 완료될때까지 브라우저를 얼어붙게 만들고 있었습니다.
이를 해소하기 위해 서버와 통신할때 비동기로 통신하도록 수정하고, 서버에서 응답받은 값을 의존하는 알고리즘들은 비동기 통신이 완료된 후 실행되도록 수정하였습니다.
{% highlight javascript linenos %}
function getTokenValue(name) {
    return $.ajax('/ajaxs/' + name);
}
 
var token = '';
var user_token = '';
 
$.when(getTokenValue('token'), getTokenValue('user_token'))
    .then(function(res1, res2) {
        token = res1[0].data;
        user_token = res2[0].data;
 
        // token, user_token 데이터에 의존하고 있는 알고리즘 실행
        ...
    });
{% endhighlight %}
동기 통신을 통해 순차적으로 코드가 실행되게 되어 개발자에게는 가독성이 좋을 수 있겠지만, 브라우저를 얼려버리고 렌더링도 함께 지연되게되어 사용자 경험에 안 좋은 영향을 끼치게 됩니다. 웹 사이트를 개발하실 때 서버와의 동기 통신은 지양해 주세요~
	
만약 동기 통신을 사용해야 한다면 페이지 로드가 완료된 후나 타임아웃 옵션을 설정해서 장기간 서버응답이 지연될 때를 대비하는게 좋을거 같네요.
<br><br><br>

## 결과
[webpagetest.org](https://webpagetest.org)는 세계 여러 곳의 지역에서 웹 사이트의 로딩 속도를 테스트할 수 있는 서비스를 제공하는데요. 이번에는 [webpagetest.org](https://webpagetest.org)에서 최적화 전과 후에 대해 성능 테스트를 진행하였습니다. 테스트 옵션은 다음과 같습니다.
* **Test Location**: Seoul, Korea - EC2
* **Browser**: Chrome
* **Connection**: Cable (5/1 Mbps 28ms RTT)
* **Number of Tests to Run**: 9

**최적화 전**
![skeleton-after](/img/web-site-optimization13.png)

**최적화 후**
![skeleton-after](/img/web-site-optimization14.png)	

* First Byte: 0.689s -> **0.519s**
* Start Render: 1.400s -> **1.100s**
* First Contentful Paint: 1.428s -> **1.109s**
* domContentLoaded: 5.302s -> **1.482s**
* loadEvent: 8.017s -> **1.998s**
<br><br><br>
	
# 끝으로
지금까지 작업했던 최적화 방법들은 기존에 우리가 알던 'CSS는 <head> 안에 Javascript는 HTML문서 최하단에 배치' 같은 기초적인 방법들을 넘어 많은 콘텐츠를 압축하고 초기에 노출되지 않아도 되는 콘텐츠를 지연시켜, 더욱 빠르게 렌더링되도록 하여 사용자가 더욱 쾌적하게 서비스를 이용할 수 있도록 하였습니다. 지금까지 소개한 방법들은 많은 비용이 필요하지 않으므로 웹 개발자라면 개발하실때 꼭 한번 고려해 보시면 좋을거 같습니다. :)
<br><br><br>
	
# Reference
* [https://www.pingdom.com/blog/page-load-time-really-affect-bounce-rate/](https://www.pingdom.com/blog/page-load-time-really-affect-bounce-rate/)
* [https://web.dev/rail/](https://web.dev/rail/)
* [https://ui.toast.com/fe-guide/ko_PERFORMANCE](https://ui.toast.com/fe-guide/ko_PERFORMANCE)
* [https://helloinyong.tistory.com/297](https://helloinyong.tistory.com/297)
* [https://web.dev/optimize-lcp/](https://web.dev/optimize-lcp/)
* [https://bearjin90.tistory.com/21](https://bearjin90.tistory.com/21)
