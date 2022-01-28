---
layout: post
title: Jetpack DataStore 소개
tags: [android, jetpack, development]
thumbnail-img: /assets/img/datastore/android_developers.png
cover-img: /assets/img/datastore/android_developers.png
share-img: /assets/img/datastore/android_developers.png
comments: true
---
`DataStore` 는 기본 설정이나 응용 프로그램 상태와 같은 소량의 데이터를 안전하고 일관되게 저장하는 방법을 제공하는 Jetpack 데이터 저장소 라이브러리입니다. 비동기 데이터 저장을 가능하게 하는 **Kotlin 코루틴과 Flow**를 기반으로 합니다. 스레드로부터 안전하고 차단되지 않기 때문에 `SharedPreferences`를 대체하는 것을 목표로 합니다. 두 가지 다른 구현을 제공합니다. 유형이 지정된 개체([프로토콜 버퍼](https://developers.google.com/protocol-buffers) 지원)를 저장하는 [Proto DataStore](https://developer.android.com/topic/libraries/architecture/datastore?gclid=CjwKCAiA55mPBhBOEiwANmzoQtX8aFaxx5WFTDOpYVN429tF3U8X3BnZu8ZMfJhRqGtyme_PzaypHhoCQDsQAvD_BwE&gclsrc=aw.ds#datastore-typed) 와 키-값 쌍을 저장하는 [Preferences DataStore](https://developer.android.com/topic/libraries/architecture/datastore?gclid=CjwKCAiA55mPBhBOEiwANmzoQtX8aFaxx5WFTDOpYVN429tF3U8X3BnZu8ZMfJhRqGtyme_PzaypHhoCQDsQAvD_BwE&gclsrc=aw.ds#datastore-preferences) 입니다. 앞으로는 `DataStore`를 사용할 때 달리 지정하지 않는 한 두 구현을 모두 참조합니다.

이 블로그 게시물에서는 **DataStore** 의 작동 방식, 제공하는 구현 및 개별 사용 사례에 대해 자세히 살펴보겠습니다. `SharedPreferences` 또한 이것이 가져오는 이점과 개선 사항과 `DataStore`가 가치 있는 이유를 살펴보겠습니다.

## DataStore vs SharedPreferences
앱에서 `SharedPreferences`를 사용했을 가능성이 큽니다. 또한 **재현하기 어려운** `SharedPreferences` 문제를 경험했을 수도 있습니다. 포착되지 않은 예외로 인해 분석에서 이상한 충돌이 발생하거나 호출할 때 UI 스레드가 차단되거나 앱 전체에서 일관되지 않고 지속되는 데이터가 발생합니다. **DataStore**는 이러한 모든 문제를 해결하기 위해 구축되었습니다.

`SharedPreferences`와 `DataStore`간의 직접적인 비교를 살펴보겠습니다.
![datastore vs shared](/assets/img/datastore/datastore_shared.png){: .mx-auto.d-block :}

## Async API
대부분의 데이터 저장소 API를 사용하면 데이터가 수정될 때 **비동기식**으로 알림을 받아야 하는 경우가 많습니다. `SharedPreferences`는 일부 비동기 지원을 제공하지만 [OnSharedPreferenceChangeListener](https://developer.android.com/reference/android/content/SharedPreferences.OnSharedPreferenceChangeListener) 를 통해 변경된 값에 대한 업데이트만 제공합니다. 그러나 이 콜백은 여전히 **메인 스레드**에서 호출됩니다. 마찬가지로, 파일 저장 작업을 백그라운드로 오프로드하려면 `SharedPreferences` `apply()`를 사용할 수 있지만, 이렇게 하면 `fsync()`의 **UI 스레드가 차단**되어 잠재적으로 버벅거림 및 ANR을 유발할 수 있습니다. 이는 서비스가 시작 또는 중지되거나 활동이 일시 중지 또는 중지될 때마다 발생할 수 있습니다. 이에 비해 `DataStore`는 Kotlin 코루틴과 Flow의 강력한 기능을 사용하여 데이터 검색 및 저장을 위한 **완전 비동기식 API**를 제공하여 UI 스레드를 차단할 위험을 줄입니다. Kotlin Flows에 익숙하지 않은 사람들에게는 비동기식으로 계산할 수 있는 값의 흐름일 뿐입니다.

## Synchronous work
`SharedPreferences` API는 즉시 사용 가능한 동기 작업을 지원합니다. 그러나 지속 데이터를 수정하기 위한 동기식 `commit()`은 UI 스레드에서 호출하는 것이 안전해 보일 수 있지만 실제로는 더 많은 **I/O 작업을 수행**합니다. 이는 ANR 및 UI 버벅거림으로 이어질 수 있고 종종 발생하는 위험한 시나리오입니다. 이를 방지하기 위해 `DataStore`는 **즉시 사용할 수 있는 동기 지원을 제공하지 않습니다.** `DataStore`는 환경 설정을 파일에 저장하고 내부적으로는 달리 지정되지 않는 한 **Dispatchers.IO**에 대한 모든 데이터 작업을 수행하여 UI 스레드를 차단되지 않은 상태로 유지합니다.

그러나 나중에 살펴보겠지만 코루틴 빌더의 약간의 도움으로 DataStore와 동기 작업을 결합하는 것이 가능합니다.

## Error handling
`SharedPreferences`는 런타임 예외로 구문 분석 오류를 발생시켜 앱을 충돌에 취약하게 만들 수 있습니다. 예를 들어 `ClassCastException`은 **잘못된 데이터 유형**이 요청될 때 API에서 발생하는 일반적으로 발생하는 예외입니다. `DataStore`는 Flow의 오류 신호 메커니즘에 의존하여 데이터를 읽거나 쓸 때 발생하는 **모든 예외를 포착**하는 방법을 제공합니다.

## Type safety
데이터를 저장하고 검색하기 위해 Map 키-값 쌍을 사용하는 것은 유형 안전 보호를 제공하지 않습니다. 그러나 Proto DataStore를 사용하면 데이터 모델에 대한 스키마를 미리 정의하고 **전체 유형 안전성**의 추가 이점을 얻을 수 있습니다.

## Data consistency
`SharedPreferences`의 원자성 보장 부족은 데이터 수정 사항이 항상 어디서나 반영되는 것에 의존할 수 없다는 것을 의미합니다. 이것은 특히 이 API의 요점이 **지속되는 데이터 저장소**이기 때문에 위험할 수 있습니다. 이에 비해 DataStore의 **완전한 트랜잭션 API**는 **원자적 읽기-수정-쓰기** 작업으로 데이터가 업데이트되므로 강력한 [ACID](https://en.wikipedia.org/wiki/ACID) 보장을 제공합니다. 또한 완료된 모든 업데이트가 읽기 값에 반영된다는 사실을 반영하여 *"쓰기 후 읽기"* 일관성을 제공합니다.

## Migration support
`SharedPreferences`에는 마이그레이션 메커니즘이 내장되어 있지 않습니다. 지루하고 오류가 발생하기 쉬운 값을 이전 저장소에서 새 저장소로 다시 매핑한 다음 정리하는 것은 사용자의 몫입니다. 이 모든 것은 데이터 유형 불일치 문제가 쉽게 발생할 수 있으므로 **runtime exception**의 가능성을 높입니다. 그러나 DataStore는 `SharedPreferences`에서 DataStore로의 마이그레이션을 위해 제공된 구현과 함께 데이터를 **쉽게 마이그레이션**하는 방법을 제공합니다.

## Preferences vs Proto DataStore
`DataStore`가 `SharedPreferences`보다 어떤 이점을 제공하는지 살펴보았으므로 **Preferences와 Proto DataStore**의 두 가지 구현 중에서 선택하는 방법에 대해 이야기해 보겠습니다.

[Preferences DataStore](https://developer.android.com/topic/libraries/architecture/datastore?gclid=CjwKCAiA55mPBhBOEiwANmzoQtX8aFaxx5WFTDOpYVN429tF3U8X3BnZu8ZMfJhRqGtyme_PzaypHhoCQDsQAvD_BwE&gclsrc=aw.ds#datastore-preferences) 는 스키마를 미리 정의하지 않고 **키-값 쌍**을 기반으로 데이터를 읽고 씁니다. `SharedPreferences`와 유사하게 들릴 수 있지만 DataStore가 제공하는 위에서 언급한 모든 개선 사항을 염두에 두십시오. 명명에 "*기본 설정*"을 함께 사용하는 것에 속지 마십시오. 이들은 공통점이 없으며 완전히 별개의 두 API에서 제공됩니다.

[Proto DataStore](https://developer.android.com/topic/libraries/architecture/datastore?gclid=CjwKCAiA55mPBhBOEiwANmzoQtX8aFaxx5WFTDOpYVN429tF3U8X3BnZu8ZMfJhRqGtyme_PzaypHhoCQDsQAvD_BwE&gclsrc=aw.ds#datastore-typed) 는 **유형이 지정된 개체**를 저장하고 [프로토콜 버퍼](https://developers.google.com/protocol-buffers) 로 지원되어 유형 안전성을 제공하고 키가 필요하지 않습니다. Protobuf는 XML 및 기타 유사한 데이터 형식보다 빠르고, 작고, 간단하고, 덜 모호합니다. 이전에 사용하지 않았다면 두려워하지 마십시오! 이것들은 배우기 매우 간단합니다. Proto DataStore를 사용하려면 새로운 직렬화 메커니즘을 배워야 하지만 그 이점, 특히 **유형 안전성**이 그만한 가치가 있다고 생각합니다.

![datastore vs proto](/assets/img/datastore/datastore_proto.png){: .mx-auto.d-block :}

둘 중 하나를 선택할 때 다음 사항을 고려해야 합니다.

- 데이터 읽기 및 쓰기를 위한 **키-값 쌍**으로 작업하고 DataStore의 개선 사항을 계속 활용하면서 최소한의 변경으로 SharedPreferences에서 **빠르게 마이그레이션**하고 유형 안전성 검사 없이 충분히 자신감을 느끼고 싶다면 **Preferences DataStore**를 사용할 수 있습니다.
- 가독성 향상의 추가 이점을 위해 프로토콜 버퍼를 배우고 싶거나 데이터가 열거형이나 목록과 같은 **더 복잡한 클래스**로 작업해야 하고 그렇게 하는 동안 **전체 유형 안전 지원**을 받으려면 **Proto DataStore**를 사용해 볼 수 있습니다.

## DataStore vs Room
"*음, 데이터를 저장하기 위해 [Room](https://developer.android.com/training/data-storage/room) 을 사용하는 것이 어떻습니까?*"라고 물을 수 있습니다. 그리고 그것은 공정한 질문입니다! 그럼 이 모든 것에서 Room이 어디에 적합한지 봅시다.

수십 KB보다 **큰 복잡한 데이터 세트**로 작업해야 하는 경우 서로 다른 데이터 테이블 간에 **부분 업데이트** 또는 **참조 무결성**이 필요할 수 있습니다. 이 경우 **Room**을 사용하는 것을 고려해야 합니다.

그러나 기본 설정이나 앱 상태와 같은 **더 작고 단순한 데이터 세트**로 작업하고 있으므로 부분 업데이트나 참조 무결성이 필요하지 않은 경우 **DataStore**를 선택해야 합니다.

![datastore vs room](/assets/img/datastore/datastore_room.png){: .mx-auto.d-block :}

지금까지 **DataStore**의 작동 방식, 변경 사항 및 개선 사항, 두 가지 구현 중에서 결정하는 방법에 대해 자세히 알아보았습니다.

###### 출처 : https://medium.com/androiddevelopers/introduction-to-jetpack-datastore-3dc8d74139e7
