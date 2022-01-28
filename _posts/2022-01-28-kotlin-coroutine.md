---
layout: post
title: Kotlin Coroutine으로 비동기 프로그래밍 배우기
tags: [android, kotlin, coroutine,development]
comments: true
---
Kotlin Coroutine에 대해 공유하려 합니다.

코루틴에 대해 이야기하기 전에 비동기와 동기에 대해 이야기해야 합니다.

## 비동기식 vs 동기식
- **비동기식 코드**는 이전 코드가 완료될 때까지 기다리지 않고 실행할 수 있으며 동시에 여러 작업을 실행할 수 있습니다. 예를 들어, 실생활에서 비동기는 커피숍에서 커피를 기다리는 동안 커피를 주문하는 것과 같으며, 동료와 회의를 하거나 소셜 미디어를 스크롤하는 것과 같은 다른 일을 할 수 있습니다.
- **동기식 코드**는 이전 코드가 완료된 후 실행할 수 있습니다. 동기식을 사용하면 동시에 단일 작업만 실행할 수 있습니다. 예를 들어 실생활에서 동기는 계산원에서 줄을 서서 기다리는 것과 같으며 계산원 앞에서 서비스를 받은 후에 서비스를 받을 수 있습니다.
  
![sync vs async](/assets/img/coroutine/sync_async.png){: .mx-auto.d-block :}

위의 예에서 **동기식** 고객은 제품이 완료된 후에 실행할 수 있고 주문은 고객이 완료된 후에 실행할 수 있지만 **비동기식**에서는 제품과 고객을 동시에 실행할 수 있으므로 비동기식으로 많은 시간을 절약할 수 있습니다!

### 그렇다면 코틀린 코루틴은 무엇일까요?
코루틴은 네트워크 호출 또는 데이터베이스의 데이터 처리와 같은 장기 실행 작업을 실행하는 동안 앱의 응답성을 유지하는 비동기 작업을 실행하는 코드를 단순화하기 위해 Android에서 사용할 수 있는 동시성 디자인 패턴입니다.

장기 실행 작업을 관리하는 코루틴 도움말 및 UI(사용자 인터페이스)는 장기 실행 작업을 실행하는 동안 응답성이 뛰어나고 사용자에게 피드백을 제공하므로 사용자는 앱이 많은 시간이 소요되는 장기 실행 작업을 수행하고 있음을 알 수 있습니다.

코루틴은 다음과 같은 문제를 해결할 수 있습니다.

- **장기 실행 작업**(네트워크 요청, 데이터베이스에서 데이터 읽기)
- **Main-safety**, 메인 스레드에서 일시 중단 기능을 호출하기 위해 저장합니다.

### 장기 실행 작업 관리
코루틴은 두 개의 새로운 작업을 추가하여 일반 기능을 기반으로 합니다. `invoke`(또는 호출) 및 `return` 외에도 코루틴은 `suspend` 및 `resume`를 추가합니다.

**suspend** — 현재 코루틴의 실행을 일시 중지하고 모든 지역 변수를 저장합니다.

**resume** — 일시 중지된 위치에서 일시 중지된 코루틴을 계속합니다.

다른 `suspend` 함수나 `launch`과 같은 코루틴 빌더를 사용하여 새 코루틴을 시작하여 `suspend` 기능을 호출할 수 있습니다.

다음 예는 가상의 장기 실행 작업에 대한 간단한 코루틴 구현을 보여줍니다.

`DataStore` 는 기본 설정이나 응용 프로그램 상태와 같은 소량의 데이터를 안전하고 일관되게 저장하는 방법을 제공하는 Jetpack 데이터 저장소 라이브러리입니다. 비동기 데이터 저장을 가능하게 하는 **Kotlin 코루틴과 Flow**를 기반으로 합니다. 스레드로부터 안전하고 차단되지 않기 때문에 `SharedPreferences`를 대체하는 것을 목표로 합니다. 두 가지 다른 구현을 제공합니다. 유형이 지정된 개체([프로토콜 버퍼](https://developers.google.com/protocol-buffers) 지원)를 저장하는 [Proto DataStore](https://developer.android.com/topic/libraries/architecture/datastore?gclid=CjwKCAiA55mPBhBOEiwANmzoQtX8aFaxx5WFTDOpYVN429tF3U8X3BnZu8ZMfJhRqGtyme_PzaypHhoCQDsQAvD_BwE&gclsrc=aw.ds#datastore-typed) 와 키-값 쌍을 저장하는 [Preferences DataStore](https://developer.android.com/topic/libraries/architecture/datastore?gclid=CjwKCAiA55mPBhBOEiwANmzoQtX8aFaxx5WFTDOpYVN429tF3U8X3BnZu8ZMfJhRqGtyme_PzaypHhoCQDsQAvD_BwE&gclsrc=aw.ds#datastore-preferences) 입니다. 앞으로는 `DataStore`를 사용할 때 달리 지정하지 않는 한 두 구현을 모두 참조합니다.

```kotlin
suspend fun fetchDocs() {                             
    val result = get("https://developer.android.com")
    show(result)                                   
}

suspend fun get(url: String) = withContext(Dispatchers.IO) { 
/* ... */ 
}
```


### 안전을 위한 코루틴
Kotlin에서 모든 코루틴은 기본 스레드에서 실행 중일 때도 디스패처에서 실행되어야 합니다. 코루틴은 스스로를 일시 중단할 수 있으며 디스패처는 코루틴을 재개할 책임이 있습니다.

코루틴이 실행되어야 하는 위치를 지정하기 위해 Kotlin은 사용할 수 있는 세 가지 디스패처를 제공합니다.

- **Dispatchers.Main** — 이 디스패처를 사용하여 기본 Android 스레드에서 코루틴을 실행합니다. UI와 상호 작용하고 빠른 작업을 수행하는 데만 사용해야 합니다. 예에는 `suspend` 함수 호출, Android UI 프레임워크 작업 실행, [LiveData](https://developer.android.com/topic/libraries/architecture/livedata) 개체 업데이트가 포함됩니다.
- **Dispatchers.IO** — 이 디스패처는 기본 스레드 외부에서 디스크 또는 네트워크 I/O를 수행하도록 최적화되어 있습니다. 예를 들어 [Room 구성 요소](https://developer.android.com/topic/libraries/architecture/room) 사용, 파일 읽기 또는 쓰기, 네트워크 작업 실행 등이 있습니다.
- **Dispatchers.Default** — 이 디스패처는 메인 스레드 외부에서 CPU 집약적인 작업을 수행하도록 최적화되어 있습니다. 사용 사례의 예에는 목록 정렬 및 JSON 구문 분석이 포함됩니다.
이 블로그 게시물에서는 **DataStore** 의 작동 방식, 제공하는 구현 및 개별 사용 사례에 대해 자세히 살펴보겠습니다. `SharedPreferences` 또한 이것이 가져오는 이점과 개선 사항과 `DataStore`가 가치 있는 이유를 살펴보겠습니다.


## 코루틴 시작하기
다음 두 가지 방법 중 하나로 코루틴을 시작할 수 있습니다.

- `launch`는 새로운 코루틴을 시작하고 호출자에게 결과를 반환하지 않습니다. "*한번 사용되고 잊어버리는*" 것으로 간주되는 모든 작업은 실행은 `launch`를 사용하여 시작할 수 있습니다.
- `async`는 새 코루틴을 시작하고 **await**이라는 일시 중단 함수를 사용하여 결과를 반환할 수 있도록 합니다.

다음은 MVVM 아키텍처와 함께 코루틴을 사용하는 예입니다.
```kotlin
   fun signIn(email: String, password: String) {
        loading.postValue(true)
        viewModelScope.launch(Dispatchers.IO){
            try {
                auth?.let { login->
                    login.signInWithEmailAndPassword(email, password)
                        .addOnCompleteListener { task: Task<AuthResult> ->
                            if (task.isSuccessful) {
                                _signInStatus.postValue(true)
                                // Sign in success, update UI with the signed-in user's information
                            } else {
                                _signInStatus.postValue(false)
                            }
                            loading.postValue(false)
                        }
                }
            } catch (e: Exception){
                loading.postValue(false)
            }
        }
    }
```

사용자를 인증하기 위해 네트워크 요청을 사용하므로 **Dispatchers.Main** 대신 **Dispatchers.IO**를 사용합니다.

Android의 모범 사례 코루틴은 [Android의 코루틴 모범 사례 | 안드로이드 개발자](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)

[원본 글](https://medium.com/codelabs-unikom/learn-asynchronous-programming-with-kotlin-coroutines-5afb16766230)
