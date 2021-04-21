---
title: 사람인 Android App Kotlin Converting
bigimg: "/img/android_kotlin_banner.jpeg"
author: 강덕현
tags:
- android
- kotlin
---

안녕하세요. 사람인HR 서비스인프라개발팀 안드로이드 앱 개발 담당 강덕현 연구원입니다. 벌써 4년이나 지난 2017년, 구글의 Kotlin 공식지원 발표 및 2019년 Kotlin first 선언 이후 많은 앱과 라이브러리들이 Kotlin으로 전환되었고, 현재도 전환과정에 있으며, 앞으로도 Kotlin의 비중은 커져갈 것으로 예상되고 있습니다. 이러한 추세 속에 사람인 안드로이드 앱도 지난 2020년 초 Kotlin으로 코드레벨 리뉴얼이 이루어 졌습니다.

여러명의 개발자들이 각자 파트를 나눠 신규 개발은 Kotlin으로, 기존 Java 소스는 점진적으로 작업하는 일반적인 경우와는 달리 저희는 소수의 인원이 짧은 기간동안 한 번의 작업으로 100퍼센트 변환을 하는 방식을 채택하게 되었습니다. 속전속결로 작업을 끝내고, 발생하는 이슈들을 모아 후속조치도 한방에 끝내자는 전략이었는데, 소수의 인원이고, 내포된 비즈니스 로직이 많지않은 웹 앱인데다 구성원 모두 코틀린 사용경험이 이미 있다는 특징을 잘 살려 러닝 커브나 전환 비용 등을 설득하는 과정이 딱히 없었기 때문에 가능했던 일이였던 것 같습니다.

<br>

# 작업 순서 정하기

가장 먼저 전체 프로젝트 내에서 변환을 진행할 순서를 패키지 단위로 정하기로 했는데, 이 때의 모토는 다음과 같았습니다.

> #### 각 파일당 한번씩만 작업해야 한다

app 모듈에는 여러 패키지가 있고, 각 패키지들은 최소 한개에서 수십개의 클래스들로 이루어져 있습니다. Kotlin은 Java와 JVM에서 100% 호환이 된다는 장점이 있는데 이는 파일단위로 작업하는 과정 중에 충분히 컴파일 및 런타임 테스트가 가능하다는 말이 됩니다. 이를 수행하려면 한 파일의 작업이 끝날 때 마다 이 파일과 의존관계에 있는 여러 클래스들을 문법적으로 상호 호환시켜야 하고, 한번 Kotlin 문법을 정확히 적용해 두어 이 파일을 의존중인 다른 파일을 변환할 때 믿을만한 기반이 되도록 해야 합니다. 따라서 편의에 따라 그때그때 이미 작업한 파일로 돌아가 끼워맞추는 식의 변환이 아닌 개발 정책과, 언어의 의도에 맞는 변환을 하려면 딱 한번 제대로 작업 한 후 그 결과를 기반으로 다른 작업을 이어나가는 것이 현명하다고 판단했습니다.

그럼 어떤 순서로 작업하는 것이 좋을까요? 아무래도 다른 패키지를 의존하는 비중은 적고, 다른 패키지가 의존하는 비중이 클 수록 앱 구조의 기저에 있어 보입니다. 구조상 아래쪽에 있고, 의존하는 패키지가 적을 수록 이미 작업을 마쳤는데 다른 패키지가 변경되었을 때 그에 대응하기 위해 다시 작업하러 돌아올 가능성이 적을 것입니다. 나중에 알게 되었는데 클린 아키텍처에서 이런 개념을 `컴포넌트의 안정성`이라고 소개하고 있습니다. 이 개념을 빌어 정리하면 패키지가 안정적인 순서대로 작업했다고 말할 수 있겠습니다.

<br>

> #### 불안정성 지표 I = 나가는 의존성 / 나가거나 들어오는 의존성

<br>
### 결정된 작업 package의 대략적인 순서

1. models - VO들 (가장 안정)
2. constants - 전역적으로 사용되는 상수들
3. utils - 전역적으로 사용되는 유틸 클래스 및 함수들
4. localdata - SQLite DB, SharedPreference 등 로컬 스토리지 관련
5. remotedata - Retrofit, Cookie Managing 등 네트워킹 관련
6. webview - 커스텀된 웹뷰, 클라이언트, 브릿지 등
7. ui - Activity, Fragment, Dialog, Notification, App Widget (가장 불안정)

<br>

# 작업
실제로 위 순서를 지켜 작업을 해 나갔더니 예상했던 대로 일부 예외 케이스(정책 변경이나 실수 등)가 아니면 한번 작업을 마친 파일로 다시 돌아오는 경우는 없었고, 일정한 방향에 따라 진행할 수 있었습니다.

그럼 이제부터 실제로 Kotlin의 대표적인 특성과 기능들이 프로젝트에 어떻게 녹아 들었는지 보시겠습니다.

## 1. data class(models package)

일반적인 앱이라면 데이터를 저장하기 위한 VO클래스를 최소 1개이상 가지고 있을 겁니다. 앱이 복잡해지고 유스케이스가 늘어나면 필연적으로 데이터를 저장할 VO역시 급격히 늘어나게 되고, 데이터 속성이 많아질수록 해당 클래스 파일의 라인 수 역시 엄청난 속도로 늘어납니다. 초기값 설정과 필요에 따라 오버로딩된 수많은 생성자들, 각각의 get/set 메소드들은 일일이 다 만들어줘야 하고, inner class가 있다면 그 안에서도 똑같은 작업을 해 줘야 합니다. 일반적으로 아래와 같은 모양새가 되겠습니다.

{% highlight java %}
public class SomeResponseModel {

    private String responseCode;
    private String responseMessage;
    private Data data;

    public String getResponseCode() {
        return responseCode;
    }

    public void setResponseCode(String responseCode) {
        this.responseCode = responseCode;
    }

    public String getResponseMessage() {
        return responseMessage;
    }

    public void setResponseMessage(String responseMessage) {
        this.responseMessage = responseMessage;
    }

    public Data getData() {
        if (data == null) {
            data = new Data();
        }
        return data;
    }

    public void setData(Data data) {
        this.data = data;
    }

    public class Data {

        private String uuid;
        private String type;
        private String url;
        private String notify;
        private String date;

        public String getUUID() {
            return uuid;
        }

        public void setUUID(String uuid) {
            this.uuid = uuid;
        }

        public String getType() {
            return type;
        }

        public void setType(String type) {
            this.type = type;
        }

        public String getUrl() {
            return url;
        }

        public void setUrl(String url) {
            this.url = url;
        }

        public String getNotify() {
            return notify;
        }

        public void setNotify(String notify) {
            this.notify = notify;
        }

        public String getDate() {
            return date;
        }

        public void setDate(String date) {
            this.date = date;
        }
    }
}
{% endhighlight %}

평범한 Java 오브젝트인데 실제로 저장되는 값은 몇 개 안되지만 80줄 정도 차지하고 있습니다. 데이터 명세가 명확히 잡혀있다면 작업 자체는 쉽고 단순하지만 주로 시간만 소모하는 반복 노동이 되고, 이러한 단순 반복 작업은 실수라도 하게된다면 한참뒤에서야 깨닫는 경우가 많습니다.

Kotlin에서는 이런 길고 지루한 작업을 훨씬 짧고 간단하게 마칠 수 있도록 `data class`라는 특수한 형태의 클래스를 제공합니다.

{% highlight kotlin %}
data class SomeResponseModel(
        var responseCode: String? = null,
        var responseMessage: String? = null,
        var data: Data = Data()
) {
    data class Data (
            var uuid: String? = null,
            var type: String? = null,
            var url: String? = null,
            var notify: String? = null,
            var date: String? = null
    )
}
{% endhighlight %}

이 Kotlin 클래스는 저기 위에있는 Java 클래스와 일치하지만 라인수는 환상적으로 줄었고, 함수같이 생긴 것들은 하나도 보이지 않습니다. `data class`는 객체 생성과 동시에 모든 내부변수를 초기화 시키고, 생성자의 오버로딩이 필요없이 프로퍼티 지정만으로 그때그때 필요한 변수만 골라서 대입할 수 있으며 get/set메서드는 자동으로 포함되고, 물론 필요한 경우에는 오버라이드 할 수 있도록 지원해 줍니다.

VO를 Java에서 Kotlin으로 변환할 때는 어떤 멤버변수들이 non null이 될 가능성이 있는지, 있다면 non null로 설정하고 디폴트 값을 세팅할 지, 아니면 nullable로 남겨둘지 등을 주의해서 결정하셔야 합니다. 직접 객체를 생성하고 파라미터를 넣어 주는 경우에는 non null 변수에 null을 대입시키면 컴파일 에러가 발생하겠지만 예를 들어 서버에서 내려온 json string을 Gson 등을 이용해 자동 파싱하도록 하는 경우 당장 컴파일 에러는 발생하지 않더라도 서버에서 어떤 의도를 가지고 null을 내려줬는데 앱에서 이런 경우를 무시하고 초기값을 설정해 버리는 등의 가능성이 있기 때문입니다. 이런경우의 재수정은 추후 의존성 그래프의 최상단에까지 영향을 주게 될 수 있습니다.

## 2. object, top-level functions(utils package)

Kotlin에서 `object`는 또 다른 특수 형태의 클래스로써, 객체 정의를 해 두면 최초 참조시 딱 한번 인스턴스가 생성되고, 그 이후 참조하면 이미 생성된 인스턴스를 반환하는 클래스입니다. 객체 생성이 스레드로부터 안전하다고 하여 싱글톤 클래스로 사용하기 딱 좋습니다. 또한 Kotlin에는 Java의 static 키워드가 없으므로 top-level function, companion object등과 함께 전역적으로 사용할 수 있도록 해 주는 옵션중에 하나입니다.

사람인 앱에는 기기와 앱의 정보를 가져오는 유틸리티 클래스가 하나 있는데, 대략적으로 아래와 같습니다.

{% highlight java %}
public class OsUtil {

    public static int getAndroidVersion() {...}
    public static int getVersion(Context context) {...}
    ...
}
{% endhighlight %}

가지고있는 메서드는 모두 static이며 인스턴스 생성여부에 관계없이 사용되고 있습니다. 이런 클래스는 복수의 인스턴스를 생성할 일이 없고, 메서드만이 앱 전반에 걸쳐 사용되므로 object로 변환하여 사용할 수 있습니다.

{% highlight kotlin %}
object OsUtil {

    fun getAndroidVersion(): String {...}
    fun getVersion(context: Context): String? {...}
    ...
}
{% endhighlight %}

아래와 같이 java에서 static 메서드 호출하듯 사용합니다.

{% highlight kotlin %}
val device_version = OsUtil.getAndroidVersion().toString()
{% endhighlight %}

이 object 클래스를 바이트코드로 변환한 뒤 Java로 디컴파일 해 보면 아래와 같이 싱글톤 클래스로 구현되어 있음을 알 수 있습니다.

{% highlight java %}
public final class OsUtil {
   static {
     OsUtil var0 = new OsUtil();
     INSTANCE = var0;
   }

   @NotNull
   public static final OsUtil INSTANCE;

   private OsUtil() {
   }

   public final String getAndroidVersion() {...}
   public final String getVersion(Context context) {...}
   ...
}
{% endhighlight %}

만약 싱글톤 인스턴스를 만들 필요도 없고, 인스턴스가 가지고 있어야 할 필드가 전혀 어서 전역에서 static 함수만을 사용하면 된다면 `top-level functions`라는 대안을 사용할 수 있습니다. 아래와 같이 .kt파일의 최상위에 함수를 생성하여 줍니다.

{% highlight kotlin %}
//  OsUtil.kt

import android.content.Context

fun getAndroidVersion(): String {...}
fun getVersion(context: Context): String? {...}

{% endhighlight %}

이 함수들은 최상위 함수들로써 어디서든지 접근자 없이 사용할 수 있습니다. 마찬가지로 Java코드로 디컴파일 해 보면 대략 아래와 같습니다.

{% highlight java %}
public final class OsUtilKt {
   
   @NotNull
   public static final String getAndroidVersion() {...}

   @Nullable
   public static final String getVersion(@NotNull Context context) {...}

}
{% endhighlight %}

OsUtil.kt파일이 OsUtilKt라는 클래스로 변신하여 top-level function으로 정의해 둔 함수들이 모두 이 클래스의 static 메소드로 구성되었습니다. 정리하자면 object class는 싱글톤 클래스를, top-level functions는 static 함수만을 가진 클래스를 생성하게 되는 것입니다.

현재 사람인 앱에서 사용중인 OsUtil.java는 유지해야 할 멤버 필드가 전혀 없기 때문에 두 방법 중 어느것을 사용하여 변환해도 괜찮습니다. 다만 두 방법중 어느 것이 나은지에 대한 고찰들이 많은데, OsUtil의 경우 함수가 클래스에 종속되어야 할 특별한 이유가 없어 top-level functions을 사용하는 것이 적당해 보이기도 하고, object가 namespace 이상의 기능을 하지 않는다면 top-level functions을 사용하라는 권고도 있습니다만 namespace로써 사용해도 크게 문제 될 것은 없어보입니다.

> [static의 사용 사례별 대체법은 이 포스트에 잘 설명되어 있습니다.](https://jelmini.dev/post/from-java-to-kotlin-life-without-static/)

object를 사용할 때는 클래스 내의 필드를 지정할 때 주의하셔야 합니다. 생성된 object는 Method Area의 static필드에 저장되며 앱이 종료된 후에도 유지되므로 context를 멤버로 가지게 하거나(누우런 경고가 뜬다), 앱의 어떠한 실행 흐름에 따른 상태정보를 저장하는 용도로 사용하기에는 적합하지 않습니다. object의 멤버에 어떤 상태를 저장해 놓고 back 키를 눌러 앱을 종료한 후 다시 켰을 때도 이전 상태가 유지되는 것을 확인하실 수 있습니다. 사람인앱에도 전역적인 상태를 체크할 때 사용되는 static 변수들이 존재하여 일부는 인스턴스 변수로 전환하거나 아예 제거했지만 여전히 companion object로 변환해 사용중인 부분이 있는데, 아주 조심스레 관리중이며 점차 더 줄여나가는 중입니다.

## 3. 고차함수 : Collection API, scoping function(remotedata, ui package)

Java에서 Kotlin으로 개발언어를 전환할 때 가장 흔히들 말씀하시는 장점이 바로 '간결하다'는 것 입니다. 그리고 Kotlin의 문법을 간결하게 만들어주는 가장 큰 요소중 하나가 고차함수 원리를 이용한 Collection API, scoping function과 같은 편의성 API들임을 부정하시는 분은 없을 것이라 생각합니다. Collection API는 Collection 객체를 다루는 복잡한 로직을 메소드 체이닝을 이용하는 스트림으로 처리할 수 있도록 해 주고, scoping functions은 한 객체에 대한 여러가지 코드 플로우를 블록으로 묶어 훨씬 보기 편하게 해 줍니다.

아래 코드블럭은 Retrofit에 설정된 인터셉터에서 쿠키를 읽어들이는 과정입니다.

{% highlight java %}
StringBuilder cookieStr = new StringBuilder();
for (String header : originalResponse.headers("Set-Cookie")) {  //  블록 1겹
    String[] cookieList = header.split(";");
    for (int i = 0; i < cookieList.length; i++) {   //  블록 2겹
        String item = cookieList[i];
        String cookieLine = item.trim();
        if (!isExcludedCookie(cookieLine)) {    //  블록 3겹
            if (!cookieStr.toString().equals("")) { //  블록 4겹
                cookieStr.append(";");
            }
            cookieStr.append(cookieLine);
        }
    }
}
{% endhighlight %}

여러 String 형태의 쿠키를 가져와 분해하여 간단한 처리와 필터링을 한 후 다시 조립하는데, 아주 간단한 과정임에도 블록이 금방 4단계까지 깊어졌습니다. 반면 Kotlin에서 제공하는 Collection API는 람다와 결합해 훨씬 간결하게 표현하는 것이 가능합니다.

{% highlight kotlin %}
val cookieStr = StringBuilder()
originalResponse.headers("Set-Cookie")
        .flatMap { it.split(";") }
        .map { it.trim() }
        .filter { !isExcludedCookie(it) }
        .forEach {
            if (cookieStr.isNotEmpty()) cookieStr.append(";")
            cookieStr.append(it)
        }
{% endhighlight %}

블록의 깊이를 깊게 하지 않고, 체이닝 할 수 있도록 지원되는 다양한 스트림 함수와 람다식의 간결한 표현으로 불과 몇 줄만에 같은 작업을 처리하였습니다.

스트림을 활용한 Collection API의 스타일은 Kotlin에서 새롭게 만들어 낸 것은 아닙니다. 다만 Kotlin의 간결한 람다 표현식과 조합되면 소괄호가 따로 필요없고 중괄호 내부에서 it으로 알아서 표현해 주는 element를 곧바로 사용할 수 있는 등의 세세한 장점들 덕분에 쉽고 단순하게 사용할 수 있게 된 것입니다. 반면에 Kotlin에서 새롭게 제공하는 것들도 있는데 바로 범위 지정 함수(scoping function)입니다.

범위 지정 함수는 세상에 5가지나 있는데, 아래와 같습니다.

1. apply(block: T.() -> Unit): T : 인자 함수를 실행하고 자기 자신을 반환, this 참조
2. also(block: (T) -> Unit): T : 인자 함수를 실행하고 자기 자신을 반환, it 참조
3. let(block: (T) -> R): R : 인자 함수를 실행하고 결과값을 반환, it 참조
4. run(block: T.() -> R): R : 인자 함수를 실행하고 결과값을 반환, this 참조
5. with(receiver: T, block: T.() -> R): R : 인자 함수를 실행하고 결과값을 반환, this 참조

모두 사용방법은 비슷하지만 약간의 차이로 인해 공식문서에서 추천하는 보편적인 용도는 조금씩 다른 편입니다. 물론 강제되지도 않고, 생각하기에 따라 해석이 조금씩 다를 수 있으므로 꼭 목적을 맞추도록 많은 정성을 들이기보다는 팀 내의 컨벤션에 따라 관용어구처럼 사용하는 것이 더 좋을 것 같습니다.

아래는 사람인에서 FCM으로부터 받은 데이터 페이로드를 Notification의 형태로 나타내는 코드입니다.

{% highlight java %}
...

NotificationManager notificationManager = (NotificationManager) this
        .getSystemService(Context.NOTIFICATION_SERVICE);
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    if (!App.getInstance().gPref.getNotificationChannelCreated()) {
        NotificationChannel channelMessage = new NotificationChannel(
            CHANNEL_ID, CHANNEL_NAME, NotificationManager.IMPORTANCE_HIGH);
        if (notificationManager != null) {
            notificationManager.createNotificationChannel(channelMessage);
            App.getInstance().gPref.setNotificationChannelCreated(true);
        }
    }
}

Intent notificationIntent = new Intent(this, MainActivity.class);
notificationIntent.putExtra(KEY_PUSH, true);
Bundle bundle = new Bundle();
for (Map.Entry<String, String> entry : data.entrySet()) {
    bundle.putString(entry.getKey(), entry.getValue());
}
notificationIntent.putExtras(bundle);
notificationIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
        | Intent.FLAG_ACTIVITY_CLEAR_TOP
        | Intent.FLAG_ACTIVITY_SINGLE_TOP);

PendingIntent pendingIntent = PendingIntent.getActivity(this, getPendingIntentId(),
        notificationIntent, PendingIntent.FLAG_UPDATE_CURRENT);

NotificationCompat.Builder notificationBuilder = new NotificationCompat.Builder(this, CHANNEL_ID)
        .setWhen(System.currentTimeMillis())
        .setSmallIcon(ic_launcher)
        .setContentTitle(title)
        .setContentText(message)
        .setContentIntent(pendingIntent)
        .setTicker(message)
        .setAutoCancel(true)
        .setPriority(NotificationCompat.PRIORITY_MAX)
        .setStyle(new NotificationCompat.BigTextStyle().bigText(message));

if (bitmap != null) {
    NotificationCompat.BigPictureStyle notiStyle = new NotificationCompat.BigPictureStyle();
    notiStyle.setSummaryText(message);
    notiStyle.bigPicture(bitmap);
    notificationBuilder.setStyle(notiStyle);
}

BitmapDrawable bitmapDraw = (BitmapDrawable) getResources().getDrawable(ic_launcher);
Bitmap launcherBitmap = bitmapDraw.getBitmap();
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    notificationBuilder.setSmallIcon(R.drawable.ic_push);
    notificationBuilder.setLargeIcon(launcherBitmap);
    notificationBuilder.setColor(0xff5B76F5);
}

Notification notification = notificationBuilder.build();
try {
    notificationManager.notify(getPendingIntentId(), notification);
    showBadge(data);
} catch (Exception e) {
}
{% endhighlight %}

알림 매니저 초기화부터 notify까지 설정할것도 많고, FCM으로 부터 받은 메시지에 첨부된 데이터들에 따라 훨씬 복잡해질 수도 있으니, Kotlin으로 변환하면서 관련 있는 부분들끼리 Scoping Function으로 묶어보았습니다.

{% highlight kotlin %}
...
val manager
        = (getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager)
        .apply {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                if (!Preferences.notificationChannelCreated) {
                    val channelMessage = NotificationChannel(
                        CHANNEL_ID, CHANNEL_NAME, NotificationManager.IMPORTANCE_HIGH)
                    createNotificationChannel(channelMessage)
                    Preferences.notificationChannelCreated = true
                }
            }
        }

val pendingIntent
        = Intent(this@FCMMessagingService, MainActivity::class.java)
        .apply {
            putExtra(KEY_PUSH, true)
            for ((key, value) in data) {
                putExtra(key, value)
            }
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                    or Intent.FLAG_ACTIVITY_CLEAR_TOP
                    or Intent.FLAG_ACTIVITY_SINGLE_TOP)
        }.let {
            PendingIntent.getActivity(this@FCMMessagingService, getPendingIntentId(),
                    it, PendingIntent.FLAG_UPDATE_CURRENT)
        }

NotificationCompat.Builder(this, CHANNEL_ID)
        .setWhen(System.currentTimeMillis())
        .setAutoCancel(true)
        .setPriority(NotificationCompat.PRIORITY_MAX)
        .setStyle(NotificationCompat.BigTextStyle().bigText(message))
        .apply {
            setContentIntent(pendingIntent)
            setContentTitle(title)
            setContentText(message)
            setSmallIcon(R.drawable.ic_push)
            val launcherBitmap = BitmapFactory.decodeResource(resources, icLauncher)
            setLargeIcon(launcherBitmap)
        }
        .apply{
            bitmap ?: return@apply
            val style = NotificationCompat.BigPictureStyle()
                    .setSummaryText(message)
                    .bigLargeIcon(null)
                    .bigPicture(bitmap)
            setStyle(style)
        }
        .build()
        .also {
            try {
                manager.notify(notificationId, it)
                showBadge()
            } catch (e: Exception) {
            }
        }
{% endhighlight %}

NotificationManager와 PendingIntent, Notification을 사용하는 단계가 좀더 명확히 분리되어 보이고, 각각에 대한 코드가 어디서부터 어디까지인지 쉽게 알아 볼 수 있습니다. Notification에 여러 설정들을 적용시키는 과정을 각 apply블록에 넣어 줌으로써 Builder패턴 특유의 체이닝과도 잘 어울려 보입니다. 작업할 때 이외에 apply, let 블럭을 접어두면 고작 몇 줄로 축약해 둘 수 있기 때문에 위의 자바 소스와 비교하면 훨씬 정돈되어 보입니다.

범위지정 함수를 사용할 때는 this또는 it으로 컨텍스트가 되는 객체에 접근하는데, 범위지정 함수가 중첩되어 있거나, 함수 외부에 Activity처럼 this로 또 참조할 수 있는 객체가 있을 때는 블록 내의 지역변수명을 바꿔주는 등의 방법으로 필요한 객체를 제대로 참조하고 있는지 확인하시면서 작업하시는게 좋을 것 같습니다.

## 4. nullability, 초기화 지연(ui package)

자바와 코틀린의 가장 두드러진 차이는 null을 다루는 방식이지 않을까합니다. 기본 자료형, 참조 자료형이 있던 자바와는 달리 모든 자료형이 참조형인 코틀린은 놀랍게도 null을 기본값으로 인정하지 않습니다. null값을 가져야 하는 변수는 자료형 뒤에 `?`를 붙여 nullable 타입임을 명시해 줘야 하는데, 이 작은 표기법의 차이가 언어 전반에 어마어마한 영향을 끼칩니다. nullable 타입으로 선언된 변수는 해당 변수를 다루는 코드 전역에서 컴파일러와 lint가 null로 인한 예외 가능성을 끊임없이 경고하고, 변수명 뒤에 ?를 붙인 Safe Call이라는 함수 호출법을 통해 Null로 부터 안전한 상황에서만 호출하는 방법이 생겼으며, 반대로 Null인 경우는 다른 값을 대입하거나 실행할 수 있도록 엘비스 연산자를 제공하기도 합니다. 또한 Safe Call을 체이닝하여 중첩되는 Null 체크 조건문을 제거할 수 있게 되었습니다.

{% highlight java %}
/*
*   JAVA에서의 변수 선언
*/
@NonNull    //  NonNull타입 변수 선언
String name = "";

@Nullable   //  Nullable타입 변수 선언
String secondName = "";
{% endhighlight %}

{% highlight kotlin %}
/*
*   Kotlin에서의 변수 선언
*/
var name = ""   //  NonNull타입 변수 선언
var secondName: String? = ""   //  Nullable타입 변수 선언
{% endhighlight %}

{% highlight java %}
/*
*   JAVA에서의 중첩된 Null Check
*/
...
if (data != null && data.getExtras() != null) {
    String url = data.getExtras().getString("url");
    if (url != null) {
        mWebView.loadUrl(url);
    } else {
        mWebView.reload();
    }
} else {
    mWebView.reload();
}
...
{% endhighlight %}

{% highlight kotlin %}
/*
*   Kotlin에서의 중첩된 Null Check
*/
...
data?.extras?.getString("url")?.let { view.loadUrl(url) } ?: view.reloadUrl()
...
{% endhighlight %}

Java에서 참조형 변수를 선언하면 초기화를 해 주지 않아도 그 변수는 자동적으로 Null값을 가지게 됩니다. 그러나 Kotlin에서는 선언만 하고 초기화를 해주지 않으면 가차없이 `Property must be initialized`라고 오류가 발생하는데, Null을 기본적으로 허용하지 않기 때문입니다.

그러나 각 화면, 뷰 등 모든 컴포넌트에 생명주기가 존재하고 수시로 바뀌어대는 안드로이드 환경에서는, 늦은 초기화가 필요할 때가 상당히 많은데 이러한 경우에는 또 다른 방법이 있습니다. 바로 by lazy와 lateinit입니다.

{% highlight kotlin %}
//  lateinit
lateinit var deps: Deps
...

public override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    deps = DaggerDeps.builder()
            .networkModule(NetworkModule(cacheFile))
            .build()
}
{% endhighlight %}

{% highlight kotlin %}
//  by lazy
protected val deps: Deps by lazy<Deps> {
    val cacheFile = File(PATH_PARENT, PATH_CHILD)
    DaggerDeps.builder()
            .networkModule(NetworkModule(cacheFile))
            .build()
}
{% endhighlight %}

lateinit 키워드를 이용하면 var로 선언된 프로퍼티의 초기화를 선언 시점이 아니라 나중으로 미룰 수 있습니다. 이 경우 Nullable 변수로 선언하지 않아도 컴파일이 가능하기 때문에 접근할 때 마다 null check를 할 필요가 없지만 초기화가 되지 않아도 컴파일 에러가 발생하지 않아 접근하여 사용하기 전에 초기화가 되어 있음을 개발자가 책임져야 합니다.

반면 한번 초기화 되면 수정될 필요가 없는 val로 선언된 프로퍼티는 선언과 동시에 초기화가 되어야만 하는데, by lazy {...} 블록을 통해 프로퍼티가 사용되는 최초시점에 초기화 되도록 지연시킬 수가 있습니다. by lazy를 통해 getter를 실행할 람다를 컴파일시 생성되는 Lazy 객체가 전달받아, 최초 접근시에 이를 실행하여 반환함으로써 지연 초기화의 기능을 수행하게 됩니다. lateinit과는 달리 초기화 시점을 고민하지 않아도 되겠습니다.

# 마무리하며

이렇게 사람인 앱에 코틀린이 어떤 계획에 따라 어떻게 적용되었는지 대표적인 코틀린의 특성들을 기준으로 소개를 해 보았습니다. Kotlin이라는 언어 자체가 편의성을 극대화 하면서도 안정성에도 많은 장점이 있고, 변환 과정에서도 간결함에 취해 안정성을 놓치지 않도록 신경 쓴 결과 배포 전 QA에서도 사소한 몇 가지 오류를 제외하면 큰 결함 사항이 없었습니다(휴..). 이후 미처 미리 알지 못한 이슈들을 제거하는 짧은 후속 조치 기간을 가진 후에는 Crash 리포트가 눈에 띄게 줄었고, 이전 보다 앱의 결함에 대한 리뷰나 문의사항도 줄어들었으며, 다른 부서로부터 컨버팅에 대한 긍정적인 피드백도 있었고, 동시에 총 코드 라인수도 33% 정도 감소되었습니다. 많은 보일러 플레이트가 줄어 생산성도 향상되었고, 읽기 편하니 코드 리뷰하기도 쉬워진 것 같습니다.

새로운 기술을 배우고 적용시키는 일은 늘 새롭고 설레는 일이지만 동시에 많은 스트레스와 우려를 동반합니다. 그럼에도 불구하고 결과적으로 앱이 긍정적인 방향으로 변화됨이 느껴져 앞으로 새로운 것을 배워 적용시키는 것에 좀 더 긍정적인 인상을 갖게 되었습니다. 이 정도의 큰 규모의 변화는 당분간 없겠지만, 앞으로도 안드로이드 앱 프로그래밍의 패러다임의 변화를 주시하다 적당한 때에 적당한 기술을 유감없이 받아들이면 좋겠습니다. Kotlin도입을 계획중이신 분들께 참고가 되는 글이 되길 바랍니다. 감사합니다.
