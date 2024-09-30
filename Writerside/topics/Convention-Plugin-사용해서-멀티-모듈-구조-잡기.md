# Convention Plugin 사용해서 멀티 모듈 구조 구성하기
[Convention Plugin](1-2-2-Convention-Plugin.md)

멀티 모듈 프로젝트를 이해하기 위해 now in android 프로젝트 구조를 클론해보며 구현해보자.

## 모듈 생성
![create_module_1.png](create_module_1.png)

![create_module_2.png](create_module_2.png)

안드로이드에 의존적인 모듈을 만들 때 Android Library를 선택하고,
안드로이드에 의존적이지 않은 모듈의 경우, 순수 Java, Kotlin Library로 생성하면 된다.
![create_module_3.png](create_module_3.png){style="auto"}

위처럼 생성된다. (data - 안드로이드 라이브러리, domain - kotlin 라이브러리)

우선 now in android [의존성 그래프](프로젝트-구조-분석.md)를 참고해서 의존 관계가 적은 모듈 먼저 생성해보자.
![create_module_13451.png](create_module_13451.png){style="auto"}

## 모듈 관계 정의
모듈을 생성했으니 관계를 정의해야 한다.
의존성 그래프를 보면 의존 관계는 다음과 같다.
- data -> database, network
- database -> model
- network -> common

각 모듈에 build.gradle 파일의 dependencies 블록에 implementation 함수를 이용해서 모듈을 참조할 수 있다.
```Kotlin
dependencies {
    implementation(projects.core.database)
    implementation(projects.core.network)
}
```

### build-logic 모듈 생성하기





