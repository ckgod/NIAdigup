# 2.1 Hilt
Hilt는 Android 앱의 의존성 주입을 위한 Dagger 기반의 라이브러리이다.
Dagger의 복잡성을 줄이고 Android 특화된 기능을 제공하여 보다 쉽게 DI를 구현할 수 있게 해준다.

## 어노테이션

### 앱 레벨 어노테이션

<list>
<li>
<p><code>@HiltAndroidApp</code></p>
<list><li>Hilt를 사용하는 안드로이드 애플리케이션에 필수로 붙어야 하는 어노테이션</li>
<li>Application 클래스에 적용하며, 의존성 주입의 시작점이 된다.</li></list>
</li>
</list>


### 컴포넌트 관련 어노테이션

<list>
<li><code>@AndroidEntryPoint</code>
<list>
<li>Android 구성요소에 의존성 주입을 가능하게 하는 어노테이션</li>
<li>Activity, Fragment, View, Service, BroadcastReceiver 등에 사용</li>
</list>
</li>
</list>

```Kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject lateinit var someClass: SomeClass
}
```

### 모듈 관련 어노테이션

<list>
<li><code>@Module</code>
<list>
<li>의존성을 제공하는 방법을 정의하는 클래스에 사용</li>
</list>
</li>
<li><code>@InstallIn</code>
<list>
<li>모듈이 어느 컴포넌트에 설치될지 지정</li>
</list></li>
</list>

```Kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule { ... }
```

### 의존성 주입 어노테이션

<list>
<li><code>@Inject</code>
<list>
<li>의존성을 주입 받을 필드나 생성자에 사용</li>
</list></li>
<li><code>@Provides</code>
<list>
<li>모듈 내에서 의존성을 제공하는 메서드에 사용</li>
</list></li>
</list>

```Kotlin
@Provides
fun provideRepository(api: ApiService): Repository {
    return RepositoryImpl(api)
}
```

### 스코프 관련 어노테이션

<list>
<li><code>@Singleton</code>
<list>
<li>애플리케이션 전체에서 단일 인스턴스로 유지</li>
</list></li>
<li><code>@ActivityScoped</code>
<list>
<li>Activity 생명주기 동안 단일 인스턴스로 유지</li>
</list></li>
<li><code>@FragmentScoped</code>
<list>
<li>Fragment 생명주기 동안 단일 인스턴스로 유지</li>
</list></li>
<li><code>@ViewModelScoped</code>
<list>
<li>ViewModel 생명주기 동안 단일 인스턴스로 유지</li>
</list></li>
</list>

### 한정자 어노테이션

<list>
<li>
<code>@Qualifier</code>
<list><li>같은 타입의 다른 구현체를 구분하기 위해 사용</li></list>
</li>
</list>

```Kotlin
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class AuthInterceptorOkHttpClient

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class OtherInterceptorOkHttpClient
```

### 바인딩 어노테이션

<list>
<li>
<code>@Binds</code>
<list><li>인터페이스와 구현체를 바인딩할 때 사용</li></list>
</li>
</list>

```Kotlin
@Binds
abstract fun bindRepository(
    repositoryImpl: RepositoryImpl
): Repository
```

