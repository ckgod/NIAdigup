# 1.2.2 Convention Plugin

Convention Plugin은 여러 모듈에서 공통으로 사용되는 빌드 로직을 캡슐화하고 재사용할 수 있게 해주는 Custom Gradle 플러그인이다.

## 목적
- 빌드 로직의 중복을 줄인다.
  - 코드 중복 감소
- 프로젝트 전체에 걸쳐 일관된 빌드 설정을 유지한다.
  - 빌드 일관성 유지
- 빌드 설정의 유지보수를 용이하게 한다.
  - 모듈별 build.gradle.kts 파일 간결하게 유지 가능

Convention Plugin을 사용하면 대규모 멀티 모듈 프로젝트에서 빌드 설정을 효율적으로 관리할 수 있으며,
프로젝트의 확장성과 유지보수성을 크게 향상시킬 수 있다.
now in android 프로젝트에서는 이 개념을 적극적으로 활용하여 복잡한 빌드 설정을 체계적으로 관리하고 있다.

<seealso>
    <category ref="convention_plugin_post">
        <a href="https://developer.squareup.com/blog/herding-elephants/">Herding Elephants</a>
        <a href="https://github.com/jjohannes/idiomatic-gradle">Reorder topics in the TOC</a>
    </category>
</seealso>