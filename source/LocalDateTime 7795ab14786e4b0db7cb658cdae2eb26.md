# LocalDateTime

Create: 2021년 10월 1일 오후 12:52
Tag: android

## 안드로이드에서 시간표시 방법

[https://developer.android.com/reference/java/time/package-summary](https://developer.android.com/reference/java/time/package-summary)

안드로이드 26에서 부터 새로 추가된 LocalDateTime을 기준으로 안드로이드에 시간 표시 방법에 대하여 이야기 해보려고 한다.

## 안드로이드에서 시간

### currentTimeMillis

UTC 1970-01-01 00:00:00을 기준으로 시작하는 시간 변수가 Long 값에 저장되기 때문에 대략 100년정도의 시간을 기록할 수 있다.

milliseconds 단위

```kotlin
System.currentTimeMillis()
```

### elapsedRealtime

안드로이드 시작 후 지금까지의 흘러간 시간

milliseconds 단위

```kotlin
SystemClock.elapsedRealtime()
```

### LocalDateTime

ISO-8601 calendar system 에서 사용하는 날짜 시간 표시방법이다.

`2007-12-03T10:15:30` 와 같이 표시하는 것이 대표적인 표시방법이고

이표기법에는 타임존이 들어가 있지 않다.

여기서 시간대를 추가, 타임존 추가 여부에 따라 사용되는 class가 OffsetDatatime, ZonedDataTime 등으로 달라지게 되며 System.currentTimeMillis()과의 상호 변환을 위한 class 및 시간간격, 시간주기 를 표시하기위한 class들이 추가로 있다.

## Android SDK 26 이하에서 사용

우선 시작하기 전에 안드로이드 26에서 추가된것이기 때문에 안드로이드 26미만에서 사용하는 방법에 대하여 알아 보자. 아직까지는 필요한 사항이다.

### ThreeThenABP

이전에 사용한 JakeWharton님의 라이브러리가 있지만 여기서는 다른 방법을 소개한다.

[GitHub - JakeWharton/ThreeTenABP: An adaptation of the JSR-310 backport for Android.](https://github.com/JakeWharton/ThreeTenABP)

### Java 8+ API 이용

안드로이드에서는 Java 8+ API desugaring support (Android Gradle Plugin 4.0.0+)를 이용한 java.time.* 에서 지원하는 코드를 사용할수 있다.

[https://developer.android.com/studio/write/java8-support#library-desugaring](https://developer.android.com/studio/write/java8-support#library-desugaring)

아래코드를 삽입하면 사용할수 있다.

```groovy
android {
    defaultConfig {
        multiDexEnabled true
    }
    compileOptions {
        coreLibraryDesugaringEnabled true
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = "1.8"
    }
}

dependencies {
    coreLibraryDesugaring 'com.android.tools:desugar_jdk_libs:1.1.5'
}
```

!!! note
    사용해본 결과는 모든 함수를 지원하지는 않는다.  
    [https://developer.android.com/studio/write/java8-support-table](https://developer.android.com/studio/write/java8-support-table)

## TL;DR

=== "gradle"

    ```groovy
    android {
        compileOptions {
            coreLibraryDesugaringEnabled true
        }
    }

    dependencies {
        coreLibraryDesugaring 'com.android.tools:desugar_jdk_libs:1.1.5'
    }
    ```

=== "kotlin"

    ```kotlin
    android {
            compileOptions {
            coreLibraryDesugaringEnabled = true
        }
    }

    dependencies {
        coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:1.1.5")
    }
    ```
<!-- 
- === "gradle"
    
    ```groovy
    android {
        compileOptions {
            coreLibraryDesugaringEnabled true
        }
    }
    
    dependencies {
        coreLibraryDesugaring 'com.android.tools:desugar_jdk_libs:1.1.5'
    }
    ```
    

   

- === "kotlin"
    
    ```kotlin
    android {
    		compileOptions {
            coreLibraryDesugaringEnabled = true
        }
    }
    
    dependencies {
        coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:1.1.5")
    }
    ``` -->