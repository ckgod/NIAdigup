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

## ConventionPlugin 작성
모듈을 만들었을 때 생성되는 build.gradle.kts 파일의 빌드 관련 중복 코드를 제거하고, 
한 곳에서 관리하기 위해 ConventionPlugin을 구현해보자.

컴벤션 플러그인을 만들면 여러 가지 모듈에서 해당 플러그인만 적용하면 중복되는 코드 없이 빌드를 진행할 수 있게 된다.

### AndroidApplicationConventionPlugin.kt 구현
<code>AndroidApplicationConventionPlugin</code>을 만들고 build.gradle.kts(Module:app) 모듈에 적용해보자.

![convention_plugin_path](convention_plugin_path.png)

convention 모듈의 java 패키지에 AndroidApplicationConventionPlugin 클래스를 구현 해야한다.
컨벤션 플러그인은 <code>org.gradle.api.Plugin</code> 인터페이스를 구현해야하고, apply 함수에 빌드 관련 코드를 넣어주면 된다.

```Kotlin
// AndroidApplicationConventionPlugin.kt
class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        TODO("Not yet implemented")
    }
}
```

우선 컴벤션 플러그인 구현에서 version catalog를 편하게 사용하기 위한 Project 확장함수를 만든다.

```Kotlin
// ProjectExtensions.kt
val Project.libs
    get(): VersionCatalog = extensions.getByType<VersionCatalogsExtension>().named("libs")
```

Kotlin Android 프로젝트의 기본 설정을 구성하는 함수는 다른 컨벤션 플러그인에서도 자주 쓰이기 때문에 Project 인터페이스의 확장함수로 구현해준다.

```Kotlin
internal fun Project.configureKotlinAndroid(
    commonExtension: CommonExtension<*, *, *, *, *, *>,
) {
    commonExtension.apply {
        compileSdk = 34

        defaultConfig {
            minSdk = 26
        }

        compileOptions {
            isCoreLibraryDesugaringEnabled = true
            sourceCompatibility = JavaVersion.VERSION_17
            targetCompatibility = JavaVersion.VERSION_17
        }
    }

    configureKotlin<KotlinAndroidProjectExtension>()

    dependencies {
        add("coreLibraryDesugaring", libs.findLibrary("android.desugarJdkLibs").get())
    }
}

private inline fun <reified T : KotlinTopLevelExtension> Project.configureKotlin() = configure<T> {
    // Treat all Kotlin warnings as errors (disabled by default)
    // Override by setting warningsAsErrors=true in your ~/.gradle/gradle.properties
    val warningsAsErrors: String? by project
    when (this) {
        is KotlinAndroidProjectExtension -> compilerOptions
        is KotlinJvmProjectExtension -> compilerOptions
        else -> TODO("Unsupported project extension $this ${T::class}")
    }.apply {
        jvmTarget = JvmTarget.JVM_11
        allWarningsAsErrors = warningsAsErrors.toBoolean()
        freeCompilerArgs.add(
            // Enable experimental coroutines APIs, including Flow
            "-opt-in=kotlinx.coroutines.ExperimentalCoroutinesApi",
        )
    }
}
```

<tip title="isCoreLibraryDesugaringEnabled">
    <p>새로운 Java API를 오래된 Android 버전에서도 사용할 수 있게 해준다.</p>
    <p>사용하려면 dependencies 추가가 필요하다.</p>
</tip> 

AndroidApplicationConventionPlugin의 apply 메서드를 구현한다.
```Kotlin
// AndroidApplicationConventionPlugin.kt
class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.application")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<ApplicationExtension> {
                configureKotlinAndroid(this)
                defaultConfig {
                    applicationId = libs.findVersion("applicationId").get().toString()
                    targetSdk = libs.findVersion("targetSdk").get().toString().toInt()
                    versionCode = libs.findVersion("versionCode").get().toString().toInt()
                    versionName = libs.findVersion("versionName").get().toString()
                }
            }
            extensions.configure<ApplicationAndroidComponentsExtension> {
            }
        }
    }
}
// libs.version.toml
[versions]
# Project versions
applicationId = "app.ckg.androidlab"
versionName = "1.0"
versionCode = "1"
minSdk = "26"
targetSdk = "34"
compileSdk = "34"
```

### AndroidApplicationConventionPlugin 적용
우리가 작성한 Convention Plugin을 사용하기 위해선 Gradle이 컨벤션 플러그인을 특정 id로 인식할 수 있어야한다.

libs.versions.toml에 id를 정의해준다.

```Kotlin
[plugins]
...

# Convention Plugin
ckgdroidlab-android-application = { id = "ckgdroidlab.android.application" }
```

build-logic:convention 모듈에 컨벤션 플러그인을 등록해준다.
입력하는 id는 libs.versions.toml 파일에 작성한 id와 같아야 한다.

```Kotlin
// build.gradle.kts (:build-logic:convention)
gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "ckgdroidlab.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
    }
}
```

마지막으로 build.gradle.kts (:app) 모듈에 컨벤션 플러그인을 적용한다.

```Kotlin
plugins {
    alias(libs.plugins.ckgdroidlab.android.application)
}

android {
    namespace = "app.ckg.androidlab"

    defaultConfig {
        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
        vectorDrawables {
            useSupportLibrary = true
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    kotlinOptions {
        jvmTarget = "17"
    }
    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.1"
    }
    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
    }
}

dependencies {
    ...
}
```

코드를 보면 컨벤션 플러그인에서 지정했던 applicationId, targetSdk, compileSdk 등이 제거된 것을 확인할 수 있다.
이 외의 다른 블럭들도 다른 컨벤션 플러그인 구현으로 공통으로 관리할 수 있다.

<note title="build-logic을 적용 후 rebuild 시 빌드에러">
    <p>
        build-logic 모듈 적용 후 rebuild를 실행하면 <code>Unable to make progress running work. There are items queued for execution but none of them can be started</code>
        라는 에러가 발생한다. build-logic:convention:testClasses에 대해서 발생하는데 앱 프로젝트 settings.gradle.kts에 <code>gradle.startParameter.excludedTaskNames.addAll(listOf(":build-logic:convention:testClasses"))</code>
        추가로 임시로 해결 가능하다.
    </p>
</note>

[//]: # (https://dev-inventory.com/57)

[//]: # (https://velog.io/@hs4609/%EC%95%88%EB%93%9C%EB%A1%9C%EC%9D%B4%EB%93%9C-%EB%A9%80%ED%8B%B0-%EB%AA%A8%EB%93%88-build-logic-%EA%B5%AC%ED%98%84)

[//]: # (https://jinukeu.hashnode.dev/android-version-catalog-convention-plugin-buildgradle)

[//]: # (https://velog.io/@trasalby/Gradle-Kotlin-%EC%BB%A8%EB%B2%A4%EC%85%98-%ED%94%8C%EB%9F%AC%EA%B7%B8%EC%9D%B8%EC%9D%84-%EC%82%AC%EC%9A%A9%ED%95%9C-%EB%AA%A8%EB%93%88-%EA%B4%80%EB%A6%AC-1-%EB%A9%80%ED%8B%B0%EB%AA%A8%EB%93%88%EC%9D%98-%EB%8F%84%EC%9E%85)

[//]: # (https://developer.squareup.com/blog/herding-elephants/)





