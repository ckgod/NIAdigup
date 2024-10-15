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
// build.gradle.kts (Module :core:data)
dependencies {
    implementation(projects.core.database)
    implementation(projects.core.network)
}
```

## build-logic 모듈 생성하기 {id="build-logic_1"}

![module5.png](module5.png)

위처럼 5개의 모듈이 생성되었고, 각 build.gradle.kts 에는 빌드와 관련된 중복 코드들이 존재한다.
앱이 커질수록 모듈과 중복 코드는 계속 증가할 것이다.

```Kotlin
// build.gradle.kts (Module :core:data)
plugins {
    alias(libs.plugins.android.library)
    alias(libs.plugins.jetbrains.kotlin.android)
}

android {
    namespace = "app.ckg.androidlab.core.data"
    compileSdk = 34
    ...
}
...
```

```Kotlin
// build.gradle.kts (Module :core:database)
plugins {
    alias(libs.plugins.android.library)
    alias(libs.plugins.jetbrains.kotlin.android)
}

android {
    namespace = "app.ckg.androidlab.core.database"
    compileSdk = 34
```
위와 같은 중복 코드를 제거하고 한 곳에서 관리하기 위한 방법중 하나가 build-logic 모듈을 만들어서 관리하는 방법이다.

### build-logic 모듈 생성
New -> Module을 선택 후 Java or Kotlin Library로 build-logic 모듈 생성

![create-build-logic.png](create-build-logic.png)

build-logic 모듈을 생성 후 Project settings.gradle.kts 파일에 <code>includeBuild("build-logic")</code>
을 작성해야 한다. 

해당 코드로 build-logic 디렉토리를 별도의 Gradle 프로젝트로 포함시키며, 메인 프로젝트와 'build-logic' 프로젝트가 함께 복합 빌드를 구성하게 된다.

```Kotlin
// settings.gradle.kts (Project Setting)
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google {
            content {
                includeGroupByRegex("com\\.android.*")
                includeGroupByRegex("com\\.google.*")
                includeGroupByRegex("androidx.*")
            }
        }
        ...
```

별도의 Gradle 프로젝트이기 때문에 settings.gradle.kts 파일과 gradle.properties 파일을 생성해준다.

```Kotlin
// settings.gradle.kts (:build-logic)
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}

rootProject.name = "build-logic"
include(":convention")
```

```Kotlin
// gradle.properties
# Gradle properties are not passed to included builds https://github.com/gradle/gradle/issues/2534
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configureondemand=true
```

### build-logic:convention 모듈 설정

```Kotlin
// build.gralde.kts (:build-logic:convention)
import org.jetbrains.kotlin.gradle.dsl.JvmTarget

plugins {
    `kotlin-dsl`
}

group = "app.ckg.androidlab.buildlogic"

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

kotlin {
    compilerOptions {
        jvmTarget = JvmTarget.JVM_17
    }
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.ksp.gradlePlugin)
}
```
convention 모듈은 kotlin dsl을 사용해서 convention plugin을 만들기 때문에 'kotlin-dsl' 플러그인을 추가해주어야 한다.

build-logic 모듈에서 생성하는 모든 플러그인은 컴파일 중에만 관련이 있고 런타임 중에는 아무것도 하지 않기 때문에 implementation이 아닌 compileOnly를 사용한다.


[//]: # (https://dev-inventory.com/57)

[//]: # (https://velog.io/@hs4609/%EC%95%88%EB%93%9C%EB%A1%9C%EC%9D%B4%EB%93%9C-%EB%A9%80%ED%8B%B0-%EB%AA%A8%EB%93%88-build-logic-%EA%B5%AC%ED%98%84)

[//]: # (https://jinukeu.hashnode.dev/android-version-catalog-convention-plugin-buildgradle)

[//]: # (https://velog.io/@trasalby/Gradle-Kotlin-%EC%BB%A8%EB%B2%A4%EC%85%98-%ED%94%8C%EB%9F%AC%EA%B7%B8%EC%9D%B8%EC%9D%84-%EC%82%AC%EC%9A%A9%ED%95%9C-%EB%AA%A8%EB%93%88-%EA%B4%80%EB%A6%AC-1-%EB%A9%80%ED%8B%B0%EB%AA%A8%EB%93%88%EC%9D%98-%EB%8F%84%EC%9E%85)

[//]: # (https://developer.squareup.com/blog/herding-elephants/)





