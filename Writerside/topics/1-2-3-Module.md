# 1.2.3 Module

## 전체 모듈 리스트
![module_list2.png](module_list2.png)

app, app-nia-catalog 모듈과 나머지 모듈은 아이콘이 다른 것을 볼 수 있다.

### Application 모듈 (초록색 네모가 붙은 아이콘)
* AndroidManifest.xml에 <code>android:name=".Application"</code> 클래스가 정의되어 있는 모듈을 뜻한다.
* 실행 가능한 앱으로 빌드되는 모듈이다.
* build.gradle에 <code>plugins { id 'com.android.application' }</code> 적용
* APK/AAB로 빌드된다.

### Library 모듈 (파란색 네모가 붙은 아이콘)
* 단독으로 실행 불가능한 모듈이다.
* 재사용 가능한 컴포넌트로 동작한다.
* AAR(Android Archive)로 빌드된다.
* 안드로이드 모듈일 경우 build.gradle에 <code>plugins { id 'com.android.library' }</code> 적용 자바 코틀린 모듈일 경우 <code>plugins { id 'java-library' id 'org.jetbrains.kotlin.jvm' }</code> 적용 