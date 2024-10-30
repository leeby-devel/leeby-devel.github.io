---
title: JVM 메모리 누수 > Heap > lettuce
author:
name: leeby
link: https://github.com/leeby-devel
date: 2024-04-12 21:00:00 +0900
categories: [JVM]
tags: [JDK, JVM, leak, MAT, jcmd]
---

최근 회사 애플리케이션의 JVM 메모리 누수를 찾아낸 뒤 픽스했다.\
과정을 정리해두면 메모리 누수를 또 찾고 있을 미래의 나에게도 도움이 될 것 같아 정리해본다.

# 트러블 슈팅 배경
애플리케이션 파드가 일정 시간 이상 에이징이 되면 RESTART 되고 있다.

사내 메트릭 분석 툴에 힙 히스토그램을 시계열로 분석할 수 있는 기능이 있어 이를 활용해봤다.

<img width="850" alt="스크린샷 2024-07-26 오전 4 07 24" src="https://github.com/user-attachments/assets/75fa57ce-d85f-4877-9350-53a96f87318f">
<img width="850" alt="스크린샷 2024-07-26 오전 4 07 48" src="https://github.com/user-attachments/assets/b057b1d2-fd31-4ab4-97e2-29c9f2f44d60">
<img width="850" alt="스크린샷 2024-07-26 오전 4 08 01" src="https://github.com/user-attachments/assets/9b988f0a-24e5-4ec3-8f2c-07b0b559f1e9">

임의의 파드를 여러개 분석해 본 결과 `long[]` 타입 객체가 시간이 지나면서 힙에 계속 누적되는 현상을 알 수 있었는데, 문제는 이 객체들이 어디서부터 생성되는지다.

# 원인 파악 과정

우선 힙 덤프를 수행해야 한다. 운영 환경에서만 발생하는 문제이기 때문에 운영 환경 컨테이너에 힙 덤프를 수행해야 하는데, 힙 덤프를 수행하면 메모리 스냅샷을 기록하기 위해 모든 스레드가 중단되는 stop-the-world 가 발생한다.

운영 환경에 영향을 주지 않고 이를 수행하기 위해서 분석하고자 하는 **파드의 라벨을 변경하여 서비스에서 제외**시킨다음 힙 덤프를 수행했다.

## VisualVM > 힙 덤프
초기에는 VisualVM 으로 힙 덤프를 수행했으나, 덤프 크기를 최소화하기 위해서인지 **덤프 수행 전 항상 Full GC 를 수행**했다. 그 결과 분석하고자 하는 `long[]` 객체들도 메모리에서 모조리 해제되고 100MB 이하만 남게 되었다.

<img width="850" alt="스크린샷 2024-04-12 오후 5 23 13" src="https://github.com/user-attachments/assets/a1e1bf54-dd84-4e52-bae9-bde25ce8742c">
_그래도 혹시 모르니 그나마 남아있는 `long[]` 객체들의 레퍼런스를 확인해봤다._

`long[]` 의 레퍼런스 상당수가 lettuce 패키지와 관련되어 있다.

## jcmd > 힙 덤프

jcmd 로 [덤프 실행 시점 그대로의 메모리 스냅샷을 얻어낼 수 있는 방법](https://stackoverflow.com/questions/23393480/can-heap-dump-be-created-for-analyzing-memory-leak-without-garbage-collection){:target="_blank"}이 있어서 이 방법대로 힙 덤프를 수행했다. 전체 덤프룰 수행하니 덤프 파일 용량이 9G 에 육박했다.

## MAT > 덤프파일 분석
1. `long[]` 객체가 의미 있는 단위로 증가할 때마다 힙 덤프를 수행\
   (120MB, 450MB, 1.3GB)
2. 덤프 파일을 파드에서 로컬로 복사하여 MAT 를 활용하여 덤프 파일 분석
3. `long[]` 타입 객체의 [Immediate Dominator](https://help.eclipse.org/latest/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Freference%2Finspections%2Fimmediate_dominators.html){:target="_blank"} 를 파악하여 어디에 기인하는 객체인지 파악

<img width="850" alt="120 복사본" src="https://github.com/user-attachments/assets/b5b64baa-e23f-472d-a702-69993300e3b6">
_long[] <= 120MB 인 시점_

<img width="850" alt="400 복사본" src="https://github.com/user-attachments/assets/5953995e-8577-4653-8373-e691f1ac69cf">
_long[] <= 450MB 인 시점_

<img width="850" alt="1300 복사본" src="https://github.com/user-attachments/assets/c9727f18-1b3c-4ee9-bd2a-5e842c9f86e2">
_long[] <= 1.3GB 인 시점_

세 덤프 파일 모두 `long[]` 타입 객체들의 [Immediate Dominator](https://help.eclipse.org/latest/index.jsp?topic=%2Forg.eclipse.mat.ui.help%2Freference%2Finspections%2Fimmediate_dominators.html){:target="_blank"} 를 따라가보니 [Lettuce CommandLatencyRecorder](https://lettuce.io/core/release/api/io/lettuce/core/metrics/CommandLatencyRecorder.html){:target="_blank"} 에서 사용하는 컴포넌트들 (**Hstogram**, **AtomicHistogram**, **LatencyUtils**) 의 파이가 앱이 에이징됨에 따라 점점 누적되고 있었다. lettuce 가 용의자에서 범인이 되는 순간이다. 😊

> [유사 사례](https://github.com/redis/lettuce/issues/1210){:target="_blank"}

# 해결 방법
lettuce 의 [Command Latency Metrics](https://github.com/redis/lettuce/wiki/Command-Latency-Metrics#command.latency.metrics.builtin){:target="_blank"} 스펙은 우리 파트에서 직접 사용하지 않고 앞으로 사용하지 않을 것 같은 lettuce 의 디버깅, 모니터링 관련 스펙이라고 판단했고 이에 따라 기능을 disable 처리했다.

```kotlin
...

val clientResources = DefaultClientResources.create().mutate()
    .commandLatencyRecorder(CommandLatencyRecorder.disabled()) // recorder 기능을 disabled 시킨다.
    .build()

return LettucePoolingClientConfiguration.builder()
    .clientResources(clientResources)  // 위에서 mutate 한 client resources 를 빌더에 추가한다.
    .build()

...
```

## 패치 이후 메트릭 모니터링
<img width="850" alt="스크린샷 2024-07-26 오전 4 26 01" src="https://github.com/user-attachments/assets/d50f65c2-3c09-43d7-92ea-86d8b722627c">
<img width="850" alt="스크린샷 2024-07-26 오전 4 26 21" src="https://github.com/user-attachments/assets/aacecd45-185d-4f47-90a3-af698e333a2c">
앱이 에이징 되어도 이전처럼 `long[]` 타입 객체가 누적되지 않음을 확인했다.


# 부록
## 전체 덤프 파일을 불러와도 MAT 에서 힙 분석 결과를 간소화 시키는 경우
<img width="850" alt="스크린샷 2024-07-26 오전 4 31 03" src="https://github.com/user-attachments/assets/34ac6b36-1663-4ae1-a852-1b055eef7efb">
MAT 에서 **Unreachable Objects** 까지 모두 트래킹하도록 옵션을 변경해줘야 한다.
>Preferences (Setting) > Memory Analyzer > **Keep unreachable objects**

<img width="300" alt="스크린샷 2024-07-26 오전 4 32 23" src="https://github.com/user-attachments/assets/f5504127-dce8-4dd1-84f9-59993a40c2c3">
<img width="850" alt="스크린샷 2024-07-26 오전 4 34 13" src="https://github.com/user-attachments/assets/3bb0e593-42cd-4375-9ce0-38c0de8b2778">
이렇게 하면 나처럼 9GB 짜리 덤프 파일을 불러오다가 힙 부족으로 계속 실패를 할 수도 있는데 😅 그땐 MAT 애플리케이션에서 패키지 내용 보기 > Contents > Eclipse > MemoryAnalyzer.ini 파일을 열고 xmx 을 늘려주면 된다.
