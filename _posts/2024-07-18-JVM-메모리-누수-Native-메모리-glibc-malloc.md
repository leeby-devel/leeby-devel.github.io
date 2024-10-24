---
title: JVM 메모리 누수 > Native 메모리 > glibc malloc
author:
name: leeby
link: https://github.com/leeby-devel
date: 2024-07-18 20:54:00 +0900
categories: [JVM]
tags: [JDK, JVM, leak, C, glibc, malloc, jcmd]
---


몇 개월 전 파트 내 애플리케이션에서 발생한 메모리 누수를 트러블 슈팅 & 픽스한 적이 있고 [포스트](/posts/JVM-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EB%88%84%EC%88%98-Heap-lettuce/){:target="_blank"}로 다루었다. 누수를 잡아낸 후에도 미량이지만 애플리케이션 전체 메모리 사용량 (RSS 기준)이 계속해서 늘어나고 있었다.

원인을 파헤쳐보니 크리티컬한 내용은 아니나, GNU C 라이브러리 (glibc) 위에 구동되는 JVM 은 알게 모르게 영향을 받을 내용이라서 이곳에 그 내용을 공유해본다.

>_글의 내용이 glibc 를 사용하는 시스템을 전제로 하기 때문에 이 부분을 먼저 확인하시면 좋을 것 같습니다._
{: .prompt-note }
<img width="864" alt="스크린샷 2024-07-25 오후 8 39 45" src="https://github.com/user-attachments/assets/d4917d5a-4503-4242-b9b2-8779ded9f93d">
_대표적으로 많이 사용하는 OS, JDK 벤더 기준으로 표를 구성해봤다._

<details>
<summary>잡담</summary>

adoptopenjdk/alpine 의 경우 alpine 의 기본 C 라이브러리인 musl 의 안정성 검토가 다 되지 않아
<a href="https://github.com/AdoptOpenJDK/openjdk-docker/blob/master/15/jdk/alpine/Dockerfile.hotspot.releases.full">JDK 15 이전까지는 glibc 를 별도로 설치</a>해서 사용해왔다.
JDK 16 부터는 musl 을 사용한다고 하는데, 호환성이 좋을지는 모르겠다. 향후 파트 내 시스템을 JDK 21 로 마이그레이션할 계획인데 이땐 alpine 대신 debian-slim 을 사용해야겠다.
</details>

## 끝나지 않은 메모리 증가 현상
메모리 누수를 픽스한 이후 수 일간의 파드 모니터링 과정에서 파드가 죽고있거나 메모리 사용량이 눈에 띄게 증가하진 않았다. 하지만 왜인지 모를 이유로 메모리 사용량이 미량이지만 아주 꾸준하게 늘어나고 있다.

<img width="839" alt="스크린샷 2024-07-18 오후 7 24 17" src="https://github.com/user-attachments/assets/a4b882a7-a492-4476-914a-e68ef0f01063">
_도대체 뭘까?_

## Heap, Non-Heap, Direct Buffer 확인
우선 메모리의 어떤 영역이 늘어나는지 알고 싶다. pmap, grafana 를 통해 Heap, Non-Heap, DirectBuffer 를 확인해 봤다.

>`$ pmap -x 1 | sort -k 3 -n -r | more`
>
> <img width="839" alt="스크린샷 2024-07-23 오후 3 51 47" src="https://github.com/user-attachments/assets/003af26c-eaf9-4444-bf66-54c72460dc54">
>
>pmap > JVM Heap > RSS 가 이전부터 Reserved Heap 수치에 도달한 상태이다.
>- 증가분은 **Heap 메모리가 아니다**.
>
>grafana JVM Micrometer > Non-Heap 에 해당하는 메모리들 > 시계열상 사용량이 거의 일정하다.
>- 증가분은 **Non-Heap 메모리가 아니다**.
>
>grafana JVM Micrometer > Direct Buffer > 시계열상 사용량이 거의 일정하다.
>- 증가분은 Native Memory > **DirectBuffer 도 아니다**.
>
><img width="839" alt="스크린샷 2024-07-18 오후 8 24 43" src="https://github.com/user-attachments/assets/2fd1cbe6-9d56-4646-9db1-3ae29a412aa9">
>_글을 정리하고 생각해보니 Native 메모리에 속하는 Direct Buffer 도 현상에 일부 영향을 줬을 수 있을 것 같다._

즉, **"JVM 에서 관찰할 수 없는 Native 메모리 어딘가"**에서 메모리 사용량 증가가 일어나고 있다고 짐작할 수 있다.

# 메모리 할당자 (malloc) 살펴보기

![KakaoTalk_Photo_2024-07-25-21-49-27](https://github.com/user-attachments/assets/1da0919b-dd4f-481d-a965-b64f891ba00c)
_앞선 포스트를 사내 게시판에도 게시했는데, 해당 포스트에 네이티브 메모리도 살펴보면 좋겠다는 동료 ~~(팀장님)~~ 분의 코멘트가 달린다._

![KakaoTalk_Photo_2024-07-19-20-48-05](https://github.com/user-attachments/assets/1138f550-d28a-46c8-9b1a-b75768876e10)
_뒤이어 옆 파트에서도 유사 사례가 있었다는 내용을 전달 받는다._

옆 파트에서 겪은 내용을 살펴보니 malloc (동적 메모리 할당자) 쪽에 **메모리 파편화 이슈**가 있다는 내용이다. JVM 을 사용하면서 나로선 전혀 생각지 못했을 영역의 이슈였다.

사실 Native 메모리가 단순히 부족해서 생기는 문제가 아닐까 (사용량이 끝없이 증가하진 않을거니깐?) 라며 엉뚱한 곳에 초점을 두고 트러블 슈팅 중이었는데, 동료 ~~(팀장님)~~ 가 중간중간 방향을 잡아준 덕에 **메모리 할당자 쪽에 영점을 맞춰** 이부분을 파헤쳐 봤다.

>포스팅에서 나도 이제 막 공부한 malloc 의 동작원리 따위를 다루어야 할지 말지 고민이 많았지만 문제를 이해하기 위해 필요한 내용이라고 판단했고 현상을 이해하기 위해 필요한 정도로만 다루어보기로 했습니다. 이에 대한 더 정확한 이해를 위해서는 **[glibc malloc 에 대한 공식 문서](https://sourceware.org/glibc/wiki/MallocInternals){:target="_blank"}**를 참고하시는 것을 권해드립니다.
{: .prompt-note }

## 갑자기 메모리 할당자 (malloc) 요?
OpenJDK 기준으로 JVM 은 OS 위에서 실행되는 C 프로세스이다. 때문에 C 메모리 할당 알고리즘의 영향을 받게 된다.

<img width="872" alt="스크린샷 2024-07-19 오후 10 12 21" src="https://github.com/user-attachments/assets/459c16df-a2d7-4b8a-ac7c-6ddf34ff6683">
_다음 그림은 C / JVM Memory 매핑을 도식화한 그림이다. ([출처](https://www.alibabacloud.com/blog/explaining-memory-issues-in-java-cloud-native-practices_599957){:target="_blank"})_

그림에서 보이듯 JVM 에서 Native / Non-heap 으로 표현되었던 메모리 영역들은 C 관점에서는 모두 Heap 에 위치한다.\
(JVM code, JVM Data, JVM Thread 의 Native Method Stack 등 일부 제외)

JVM 은 C Heap 안에서 논리적으로 메모리를 구분하여 사용하는 것 뿐이다. 따라서 위에서 말한 **"JVM 에서 관찰할 수 없는 Native 메모리 어딘가"**에서 발생한 문제가 C 메모리 할당자 (malloc) 이슈와 관련이 있을 수도 있다는 점이 이상해보이진 않아졌다.

## 메모리 할당자 종류

대부분의 unix 기반 시스템은 표준 C 라이브러리 구현체로 glibc 를 사용하는데 glibc 는 동적 메모리 할당자로 ptmalloc2 을 사용한다. 그러니 **대부분의 linux 배포판의 기본 메모리 할당자로 ptmalloc2 을 사용한다**고 이해하면 된다.\
_(alpine 등 일부 OS 는 제외 / 최상단 표 참고)_

그 밖에도 동적 메모리 (Heap) 할당자는 다양한 써드 파티에서 구현한 구현체들이 있다. 대표적인 몇가지만 보자면..

| 메모리 할당자       | Description                                                                                      | 사용처                           |
|---------------|--------------------------------------------------------------------------------------------------|-------------------------------|
| **ptmalloc2** | 주로 glibc의 메모리 할당기로 사용되며, C/C++ 언어에서 사용되는 대규모의 동적 메모리 할당과 해제를 효율적으로 처리하는 것을 목표로 한다.               | 대부분의 Linux 배포판 (glibc)        |
| tcmalloc      | 다중 스레드 환경에서 뛰어난 성능을 발휘하는 것이 특징. TCMalloc은 각 스레드별로 메모리 캐시를 사용하여 메모리 할당과 해제를 더욱 빠르게 처리한다.          | Google                        |
| jemalloc      | 다중 스레드 환경에서 효율적인 메모리 할당과 관리를 제공하며, 특히 대규모 멀티스레드 애플리케이션에서의 성능을 최적화한다. 낮은 메모리 오버헤드와 높은 확장성을 목표로 한다. | Facebook, Redis, Firefox, ... |

## ptmalloc2 동작 원리
뒤이어 설명할 이슈를 이해하기 위해서 ptmalloc2 의 동작 원리를 어느정도 알고 있는게 좋다.

| Terminology      | 설명                                                                                                           |
|------------------|--------------------------------------------------------------------------------------------------------------|
| Arena            | 하나 혹은 그 이상의 쓰레드에서 공유하는 영역으로, 아레나에 할당된 쓰레드들은 아레나의 freelist 들로부터 메모리를 할당한다.                                    |
| Heap             | 인접해있는 영역의 메모리로 Chunk 들의 집합으로 이루어져 있다. 개별 힙 세그먼트는 정확히 하나의 아레나에 속하게 된다.                                        |
| Chunk            | 크게 In-Use 상태 (**owned by app (JVM)**) 혹은 Free 상태인 (**owned by glibc**) Chunk 로 구분할 수 있고 이들의 집합이 Heap 을 구성한다. |
| freelist (= Bin) | Free 상태인 Chunk 들을 관리하기 위한 연결 리스트                                                                             |


스레드가 메모리를 요청 `malloc()` 하면 하나의 Arena 를 할당받는데, 여러개의 스레드가 동일한 Arena 를 할당 받을 수 있다. 한번 할당받은 Arena 는 몇몇 [예외 상황](https://sourceware.org/glibc/wiki/MallocInternals#Switching_arenas){:target="_blank"}을 제외하면 스위치되지 않는다. ptmalloc2 은 이런 Arena 를 기본적으로 여러개 유지하기에 스레드들간 경합을 최소화할 수 있다.

또한 하나의 Arena 에서 각 스레드는 별도의 Heap 세그먼트를 유지하기 때문에 이러한 Heap 을 유지하는 freelist (Bin) 도 별도로 유지된다. 이것이 멀티 스레딩 환경에서 동시에 malloc 요청이 오더라도 대개 즉각적으로 (= 경합 없이) freelist 로부터 메모리를 할당해 줄 수 있는 이유이다.


>`dlmalloc` 에서는 모든 스레드가 하나의 freelist 를 공유하기에 오직 하나의 스레드만이 임계 영역에 들어갈 수 있었다. 멀티스레딩 환경에서의 이런 경합을 개선하고자 나온것이 `dlmalloc` 을 포크한 `ptmalloc2` 이다.
{: .prompt-note }

## Arena 의 크기와 개수
Arena 크기와 개수는 어떻게 정해질까?

**Arena 의 최대 크기 = 64MiB**
```c
# 요즘은 대부분 64bit 시스템이므로 long == 8bytes 일 것이다.

define DEFAULT_MMAP_THRESHOLD_MAX (4 * 1024 * 1024 * sizeof(long))
define HEAP_MAX_SIZE (2 * DEFAULT_MMAP_THRESHOLD_MAX)

# "HEAP_MAX_SIZE" = 2 * (4 * 1024 * 1024 * 8)
# "HEAP_MAX_SIZE" = 67108864 "bit"
# "HEAP_MAX_SIZE" = 65536 "KiB"
# "HEAP_MAX_SIZE" = 64 "MiB"
```

**CPU 당 Arena 의 최대 개수 = 8**
```c
define NARENAS_FROM_NCORES(n) ((n) * (sizeof (long) == 4 ? 2 : 8))

# long == 8bytes 이므로, CPU 당 ARENA 의 개수는 8 개다.
```

즉, 64bit 시스템에서 **`CPU 하나당 Arena 의 최대 크기는 512MiB`** (64MiB * 8) 이다.\
우리 애플리케이션의 파드 cpu resource 는 10 이었다. 이는 Arena 의 개수가 최대 80개 까지 늘어날 수 있고 전체 크기는 최대 5GiB 이상 늘어날 수 있다는 이야기이다. 😱

<details>
<summary>잡담</summary>

여담이지만 cpu resource 리소스를 너무 많이 잡아 놓은 것 같다.<br /><br />
JVM 애플리케이션 특성상 앱 시작시 많은 CPU 자원을 소모하기 때문에 cpu 리소스를 많이 할당했지만 평시에는 cpu 자원을 거의 사용하지 않았다.
웜업 로직을 개선하든, 클러스터 버전업을 하면서 k8s 1.27 에 소개된 <a href="https://kubernetes.io/blog/2023/05/12/in-place-pod-resize-alpha/#java-processes-initialization-cpu-requirements">In-place Resource Resize</a> 라는 피쳐를 사용해보든 개선해야 할 사항이라 생각한다.
</details>

# ptmalloc2 의 메모리 파편화 문제
이제 위에서 언급한 malloc 의 메모리 파편화 이슈를 본격적으로 살펴본다.\
이슈를 두가지 면에서 살펴보고, 테스트를 통해 내용을 검증해보자.


## Point 1.

>**Q.**
>
>_위에서 계산한 수치는 Arena 의 "최대" 크기일 뿐 `malloc()` 후에 `free()` 만 잘 되고 있다면 문제될게 없지 않은가?_

>**A.**
>
>하지만 ptmalloc2 는 기대한 것처럼 동작하지 않는다. `free()` 가 호출되면 ptmalloc2 은 OS 로 (대체로) 메모리를 반환하지 않는다. 🤯
>대신 반환할 메모리를 향후 있을 수도 있을 사용을 위해 freelist 로 만드는데, freelist 는 OS 관점에서는 애플리케이션이 여전히 사용중인(RSS) 메모리일 뿐이다.
>
>좀 더 [복합적인 로직](https://sourceware.org/glibc/wiki/MallocInternals#Free_Algorithm){:target="_blank"}이 있지만 단순화하면,
>- `free()` 호출 시, 향후 있을 사용을 위해 OS 로 메모리를 반환하지 않고 해당 **Heap 을 freelist 로 할당**한다.
>- 이 말은 애플리케이션 메모리가 glibc 에 반환된 것이지, OS 에 반환된 것이 아니라는 말이다.
>- 이같은 이유로 위에서 Chunk 를 설명할 때 Free Chunk 를 **owned by glibc** 로 표현했다.

👉 정리하면 `free()` 커맨드는 OS 로 메모리를 반환시키는 명령어라고 볼 수 없고, glibc 는 `free()` 된 메모리를 freelist 로 만드는데, 이는 여전히 애플리케이션에서 사용중인 (RSS) 메모리로 인식된다.

## Point 2.

>**Q.**
>
>_OS 로 메모리가 반환되지 않더라도 freelist 재활용만 잘하면 문제될게 없지 않은가?_

>**A.**
>
>이 대목에서 glibc 의 메모리 점유와 관련된 버그 리포트들을 찾아볼 수 있다.\
>메모리 할당자가 기대처럼 freelist 재활용을 잘 하지 못하고, 새로운 Arena 를 생성하면서 메모리 점유량이 늘어난다는 내용이다.
>- [https://github.com/cloudfoundry/java-buildpack/issues/320](https://github.com/cloudfoundry/java-buildpack/issues/320){:target="_blank"}
>- [https://github.com/quarkusio/quarkus/issues/36204](https://github.com/quarkusio/quarkus/issues/36204){:target="_blank"}

👉 요약하면 다음과 같은 일이 반복적으로 일어난다는 내용이다.
1. 메모리 요청이 들어온다.
2. 재사용을 위해 남겨둔 freelist 를 사용하지 않고 또 다른 Arena 를 생성한다.
3. 메모리를 모두 사용한 뒤 (향후있게 될 사용을 위해) OS 에 반환하지 않고 freelist 로 만든다.
4. Arena 에 사용되지 않는 freelist 들이 계속 쌓여간다.

**`2번`**: freelist 가 실제로 재활용되지 않는 경우가 많다며 버그리포트에서 지적하는 내용이다.\
**`3번`**: glibc 문서에서 설명하고 있는 ptmalloc2 의 스펙이다.\
**`4번`**: 실제로 일어나고 있다면 메모리 (RSS)가 계속 증가하는 주범이 될 것이다. (**검증 필요**)

## malloc_trim()
glibc 에 이런 freelist 를 실제 OS 로 반환하는 명령어로 `malloc_trim()` 명령어가 있다. 이는 외부에서 C-heap 을 trim 할 수 있도록 설계된 glibc 의 API 이다.

JVM 에 위 같은 문제가 있자, `malloc_trim()` 명령어를 애플리케이션 레벨(JDK)에서 호출할 수 있게 해달라는 요청이 있었다. 이는 JDK `11.0.18`, `17.0.2` 에 백포팅돼서 jcmd 명령어를 통해 freelist 를 정리할 수 있게 되었다.

👉 `jcmd <pid> System.trim_native_heap`


- [https://bugs.openjdk.org/browse/JDK-8268893](https://bugs.openjdk.org/browse/JDK-8268893){:target="_blank"}
- [https://bugs.openjdk.org/browse/JDK-8273602](https://bugs.openjdk.org/browse/JDK-8273602){:target="_blank"}

또한 JVM 에서 이를 자동으로 호출하는 기능이 JDK `17.0.12`, `21.0.3` 부터 정식 기능으로 백포팅되어 `-XX:TrimNativeHeapInterval` 옵션을 통해 trim 주기를 설정할 수 있다.
초기에는 실험 기능으로 들어갔기 때문에 `-XX:+UnlockExperimentalVMOptions` 옵션을 함께 활성화 해야 했지만 지금은 공식 기능으로 채택되었다.
기본으로는 비활성화되어있고, 주기를 설정하면 활성화된다.


<img width="1305" alt="스크린샷 2024-07-23 오후 4 34 45" src="https://github.com/user-attachments/assets/671354c7-0655-4f72-97fe-cd796f7fedf3">

- [https://bugs.openjdk.org/browse/JDK-8293114](https://bugs.openjdk.org/browse/JDK-8293114){:target="_blank"}
- [https://bugs.openjdk.org/browse/JDK-8325496](https://bugs.openjdk.org/browse/JDK-8325496){:target="_blank"}

**_Autotrim 기능은 JDK 11 엔 백포팅 해주지 않았다..😡 놓아줄 때가 된 듯 하다._**

## 테스트 & 검증
테스트를 통해서 위에서 언급한 `4번 (Arena 에 사용되지 않는 freelist 들이 계속 쌓여간다.)` 을 검증해보자.

메모리 할당 상태를 살펴보기 위해 pmap 커맨드를 이용해도 해당 메모리가 Arena 에 할당된 메모리인지 알기 힘든데, 다행히도 [pmap 의 메모리 할당 상태를 분석해주는 java 애플리케이션](https://github.com/bric3/java-pmap-inspector){:target="_blank"}이 있다. inspector 를 사용하면 100% 는 아니지만 대략적으로 현재 메모리의 할당 상태를 확인해볼 수 있다. pmap 결과를 인자로 전달하면 inspector 가 메모리 할당 상태를 분석해준다.

분석 방법은 간단하다.

>1. **운영 파드(pod) 중 하나의 라벨을 변경하여 서비스에서 제외시킨다.**
>2. **`jcmd <pid> System.trim_native_heap` 전 후 pmap 을 비교한다.**
>3. **애플리케이션 RSS 수치 변동 여부를 확인한다.**
>
>**trim 전**
>```
>Memory mappings:
>         JAVA_HEAP count=1     reserved=12058624   rss=11036196
>           UNKNOWN count=168   reserved=960548     rss=728168
>      MALLOC_ARENA count=259   reserved=16982044   rss=2221852
>       JAVA_THREAD count=226   reserved=232328     rss=20364
>  UNKNOWN_SEGMENT1 count=35    reserved=107660     rss=73952
>  UNKNOWN_SEGMENT2 count=63    reserved=129024     rss=118456
>   NON_JAVA_THREAD count=15    reserved=15480      rss=208
>         CODE_HEAP count=3     reserved=245760     rss=179680
>```
>
>**trim 후**
>```
> $ jattach 1 jcmd System.trim_native_heap
> Connected to remote JVM
> JVM response code = 0
> Attempting trim...
> Done.
> Virtual size before: 30741876k, after: 30741748k, (-128k)
> RSS before: 14379488k, after: 13552404k, (-827084k)               <-- trimmed (단위: KB)
> Swap before: Ok, after: Ok, (Ok)
>
>
>Memory mappings:
>         JAVA_HEAP count=1     reserved=12058624   rss=11035946
>           UNKNOWN count=168   reserved=960548     rss=728168
>      MALLOC_ARENA count=259   reserved=16982044   rss=1395884     <-- about 800 MB trimmed from ARENA (단위: KiB)
>       JAVA_THREAD count=226   reserved=232328     rss=20284
>  UNKNOWN_SEGMENT1 count=35    reserved=107660     rss=73952
>  UNKNOWN_SEGMENT2 count=63    reserved=129024     rss=118456
>   NON_JAVA_THREAD count=15    reserved=15480      rss=208
>         CODE_HEAP count=3     reserved=245760     rss=179692
>```
>
>> `MALLOC_ARENA` 개수가 259개, `reserved` 메모리가 16982044KiB 이다.\
>> 16982044 KiB / 259 == 65567 KiB == 64.03 MiB 로 정확히 아레나의 최대 크기이다.
>>
>> 계산식 대로면 애플리케이션 할당 cpu 가 10개이기 때문에 최대 Arena 는 80개여야 하는데..?\
>> 개수가 훨씬 많아서 이상하다가도, reserved 크기를 보면 아레나가 맞다.
>>
>> 이부분은 왜인지 아직 모르겠지만, 개수보다는 RSS 수치가 중요하기 때문에 우선 넘어간다.
>{: .prompt-warning }
>
> **애플리케이션 RSS 수치 확인**
> <img width="808" alt="스크린샷 2024-07-23 오후 5 25 30" src="https://github.com/user-attachments/assets/5dd15ea2-a481-4c01-9c24-6f80d0cf3011">
>_수일간 점진적으로 증가해오던 메모리 수치가 trim 후, 훅 떨어졌다._
>
> 앞서 설명했듯이 JDK 17, 21 을 사용하면 이것을 일정주기로 auto trim 해줄 수 있다. JDK 11 에선 jcmd 명령어를 주기적으로 실행해주는 스크립트를 작성해도 문제가 없을 것 같지만 이 참에 JDK 버전을 올리는 방향이 더 좋아보인다.

이로써 실제 trim 을 통해 `MALLOC_ARENA` 에서 대략 800MB 가량의 메모리가 OS 로 반환되었음을 확인했다. 이말은 JVM `pid == 1` 에 820MB 가량의 메모리가 freelist (사용되지 않는 상태) 상태로 남아있었다는 말이다.

또한 애플리케이션의 에이징됨에 따라 정리되는 (trimmed) 메모리가 커지는 것을 관찰할 수 있었는데 이를 통해 freelist 가 시간이 지남에 따라 축적된다는 점을 알 수 있었고, 위에서 말한 **`4번 (Arena 에 사용되지 않는 Freelist 들이 계속 쌓여간다.)`을 경험적으로 검증**한 셈이다.

# 정리
- **현상**: JVM 수준에서 관측할 수 없는 Native 메모리 사용량의 지속적인 증가 현상
- **원인**: glibc 를 표준 C 라이브러리로 사용하는 리눅스 시스템 C 메모리 할당자 (= ptmalloc2) 의 메모리 파편화 문제

## 개선 방안
**👉 JDK `17.0.12`, `21.0.3` 이상 버전**
- `-XX:TrimNativeHeapInterval` 옵션을 활성화 하여 JVM 이 파편화된 freelist 메모리를 주기적으로 정리하도록 한다.

**👉 그 이전 버전**
- JDK `17.0.12`, `21.0.3` 이상 버전으로 업그레이드한다. 😄

- jcmd 의 `System.trim_native_heap` 를 호출하는 스크립트를 일정 주기로 실행시킨다.\
  JDK `11.0.18`, `17.0.2` 이상의 jcmd 에만 포함된 기능이다.

- 시스템의 메모리 할당자를 변경한다. (jemalloc, tcmalloc)\
  이런(사용해본적 없는 메모리 할당자를 사용하는) 부담을 가지고 갈바엔 JDK 업데이트를 하는게 나을 것 같다.

- 시스템 최대 ARENA 개수(`MALLOC_ARENA_MAX`)를 조정한다.\
  C Heap 을 제한하는 방식이라 퍼포먼스 이슈가 있을 수도 있다.\
  이 방식을 택하려면 상황을 잘 고려해서 사용해야 할 것 같다.

## Note.
1. **이론상 CPU 가 하나 추가될 때마다 Native 메모리가 512MiB 까지 더 필요할 수도 있다는 점**\
사실 ptmalloc2 이 항상 freelist 를 재활용하지 않는 것도 아니고, OS 로 메모리 반환을 항상 하지 않는 것도 아니다. 때문에 메모리 점유율은 증가하긴 하지만 완만하고 느리다.\
배포 주기가 매우 긴 조직이 아니면 모든 Arena 가 이론상의 최대 수치 만큼 도달하지는 않을 것 같다. 다만 중요한점은 문제를 인지하고 이에 대한 대비를 하느냐의 여부일 것 같다.

2. **ptmalloc2 는 (JVM 을 구동할 때?) 메모리 파편화 ~~이슈~~가 있다는 점**\
사실은 ptmalloc2 자체의 "이슈"라기 보다는 ptmalloc2 의 메모리 할당 알고리즘이 JVM 과 핏하지 않은 것 뿐이라는 생각이 더 크다.

3. **JVM `17.0.9`, `21.0.1` 이상 버전에서는 `-XX:TrimNativeHeapInterval` 을 세팅하는게 좋다는 점**\
위 세팅은 기본적으로 disable 되어있다. Native 메모리 파편화를 방지하기 위해서 세팅해두면 좋을 것 같다.
