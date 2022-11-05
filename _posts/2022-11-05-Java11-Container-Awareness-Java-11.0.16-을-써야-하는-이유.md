---
title: Java11 Container Awareness / Java 11.0.16 을 써야 하는 이유
author:
name: leeby
link: https://github.com/leeby-devel
date: 2022-11-05 16:54:00 +0900
categories: [Java, JVM]
tags: [Java, Java11, JVM, Kubernetes, docker, JVM Container Awareness]
---

## **저처럼 글 읽기를 싫어하시는 분들을 위한 결론**

> - Kubernetes 환경에서 Java 11 을 사용중이라면 **Java 11.0.16** 버전 사용을 권장한다.
> - 혹시라도 12 - 14 버전을 사용중이라면 15 이상의 버전 사용을 권장한다.
> - 그렇지 않으면 소중한 Pod 가 OOMKilled 당할 수 있다.
>
> 특히 `InitialRAMPercentage`, `MaxRAMPercentage` 옵션을 사용한다면 꼭!
>
> <img width="887" alt="스크린샷 2022-11-05 오후 2 13 46" src="https://user-images.githubusercontent.com/98696463/200102403-1ec6a33f-b1a6-4d51-8c90-4f4a4c3be153.png">

> 글이 깁니다. 이제부터 다음에 대해 구구절절 설명합니다.
> - K8s 환경에서 Java 11 구동시, 왜 Java 11.0.16 버전을 사용해야 하는지
> - 이 사람은 이것을 어쩌다 알게 되었는지
{: .prompt-warning }


## **"적절한" 리소스 산정을 위한 여정...**

최근 파트의 메인 프로젝트 배포를 앞두고 나름대로 배포 프로세스를 정비 중이었다.

배포할 애플리케이션은 서비스에서 발생하는 대부분의 트래픽을 처리하는 메인 애플리케이션이었고, 이 거대한 시스템을 새로운 언어로 포팅하여 마침내 배포를 앞둔 단계였다. 때문에 Kubernetes Pod 의 "적절한" 리소스 산정 작업이 꼭 필요했다.

**"적절한"** 리소스 산정은 어떻게 해야 할까?

<img width="291" alt="스크린샷 2022-11-05 오후 2 14 59" src="https://user-images.githubusercontent.com/98696463/200102438-fe04a755-9691-4fe7-b108-c6504a3834f7.png">

### _Pod 야.. OOMKilled 로부터 지켜주지 못해 미안해_

> 과거에 JVM 기반의 애플리케이션 리소스 산정을 위해 JMX 로 애플리케이션 모니터링도 하고 Heap 덤프도 했다.
>
> 나름의 삽질을 통해 리소스를 산정하여 k8s 위에 애플리케이션을 올렸지만 종종 알 수 없는 이유로 내 소중한 Pod 가 OOMKilled 당하는 모습을 목격해야 했고, 결국엔 리소스 상향이라는 미봉책으로 여차 저차 땜질했던 기억이 다시 떠올랐다.
>
> <img width="663" alt="스크린샷 2022-11-05 오후 2 25 13" src="https://user-images.githubusercontent.com/98696463/200103865-c62e94a8-57e6-46ed-ac82-972f739975f8.png">
>
> 정말 이상했다. Pod 가 왜 설정한 메모리 limit (8Gi) 보다 더 많은 메모리를 사용하다 죽는지 말이다.

~~분명 `MaxRAMPercentage` == 70(%) 으로 지정했는데 😥~~

## **InitialRAMPercentage, MaxRAMPercentage 그리고 Java 11**

지난번 부족했던 점을 복기하고자, 당시 회사 인트라넷에 작성한 삽질기에 전 파트 동료분이 작성해주신 댓글과 링크를 열심히 곱씹어 봤다.

<img width="654" alt="스크린샷 2022-11-05 오후 2 31 48" src="https://user-images.githubusercontent.com/98696463/200104138-f10b7f60-d70d-412b-ae84-c499facc7f91.png">

당시 애플리케이션의 Heap 메모리 조절을 위한 JVM 옵션 값으로 `InitialRAMPercentage`, `MaxRAMPercentage` 를 사용했다. `xmx`, `xms` 옵션보다 유동적이라는 이유에서였다.

### _Java 13 부터만 해당 옵셥을 사용할 수 있다면 왜일까?_

궁금해서 글에 있는 Java 13 이전 버전에서 발생하는 **[버그 리포트](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8223957){:target="_blank"}**를 쭉 읽어봤다.

> **리포트에서 지적한 문제점**
>
> <img width="704" alt="스크린샷 2022-11-05 오후 2 38 45" src="https://user-images.githubusercontent.com/98696463/200104272-5ce53656-e6a3-4999-8d9e-bb29039ea4ce.png">
>
> 1. 시스템(컨테이너) 메모리가 Java MaxRAM 보다 큰 경우 위 옵션은 시스템 메모리가 아닌 MaxRAM 을 기준으로 MazHeapSize 를 계산한다.
>   (`MaxHeapSize` = `MaxRAM` * `(MaxRAMPercentage) / 100`)
>
> 2. MaxRAMPercentage 로 설정된 MaxHeapSize 가 32g 를 넘어갈 수 없도록 로직이 짜여있다. 이는 기본적으로 켜져있는 `UseCompressedOops` (Java 애플리케이션 메모리 상한을 32g 로 제한) 옵션이 해당 옵션들 보다 우선되기 때문이다.
>
>
> *[* CompressedOops 는 왜 Java 애플리케이션 메모리를 최대 32g 로 제한할까?](https://brunch.co.kr/@alden/35){:target="_blank"}*

### _Java 11 에서도 사용할 수 있겠는데?_
위 내용을 정리하면, Java 13 에서 패치된 내용은 위 1, 2번의 문제를 개선했을 뿐, Java 11 에서 `InitialRAMPercentage`, `MaxRAMPercentage` 옵션을 사용하지 못하는 것은 아니었다.

그리고 다음과 같은 이유로 우리 파트 애플리케이션은 1, 2 번 한계점에 노출되지 않는 환경이었다.

> 1. Java MaxRAM 의 default 값은 128Gi 이다. 그런데 우리는 이보다 작은 10Gi 내외의 시스템 메모리 (= Pod resource limit) 를 사용한다. 시스템 메모리가 더 작을 땐 MaxRAM 이 아닌 시스템 메모리를 기준으로 MaxHeapSize 를 산출한다.
> <img width="1718" alt="스크린샷 2022-10-14 오전 2 09 53" src="https://user-images.githubusercontent.com/98696463/200104787-0729c5f0-cd25-474c-87e6-52c0273d5d9b.png">
>
> 2. 앞서 말했듯 시스템 메모리는 10Gi 내외로 산정할 것이기 때문에 힙사이즈가 32g 이상 될 일이 없다.
>

이에 따라 나는 배포할 애플리케이션에 `InitialRAMPercentage`, `MaxRAMPercentage` 옵션을 사용하는 것은 무리가 없다고 판단했다.

## **이상한 MaxHeapSize**

위 흐름을 이어 받아, 이전에 OOMKilled 로 고생하던 애플리케이션의 openjdk 버전에 `MaxRAMPercentage` 옵션을 넣고 MaxHeapSize 를 출력해봤다.

### _**docker-desktop 4.3.0**_
**openjdk11.0.4_11**
<img width="1321" alt="11 0 4-failed" src="https://user-images.githubusercontent.com/98696463/200109639-f75e41e6-1609-4565-9a16-139e506596fd.png">

컨테이너 메모리를 1GB 로 설정하고 `MaxRAMPercentage` == 80 으로 설정했다.

`MaxHeapSize` == 6.13G 라는 엉뚱한 수치가 나왔다. (..???)

**openjdk13.0.2_8**
<img width="1575" alt="스크린샷 2022-11-05 오후 4 48 47" src="https://user-images.githubusercontent.com/98696463/200109567-b836840e-bdfb-486c-8914-e443d4e230d9.png">

~~심지어~~ openjdk13 도 마찬가지였다.

##### _Java Container Awareness 버그_

이 이상한 현상에 대해 또 다시 구글링을 해봤다.

[아티클](https://kpow.io/articles/corretto-memory-issues/){:target="_blank"} 하나를 보고 이 또한 **[Java 버그](https://bugs.openjdk.org/browse/JDK-8230305){:target="_blank"}**임을 알게 됐다. 버그 내용을 요약하면, 다음과 같다.

> jvm 의 컨테이너 리소스 detection 은 cgroup v1 위에 설계됐지만 cgroup v1 을 사용하던 Docker 가 새 버전을 릴리즈하면서 일부 cgroup v2 feature 를 사용하게 되었다.
>
> 따라서 그 위에서 jvm 을 구동하면 **컨테이너 리소스 detection 에 문제가 생길 수 있다.**

> _cgroup은 단일 또는 태스크 단위의 프로세스 그룹에 대한 자원 할당을 제어하는 리눅스 커널 모듈이고 v1, v2 가 있다._
> _ref. [https://sonseungha.tistory.com/535](https://sonseungha.tistory.com/535){:target="_blank"}_
{: .prompt-note }

관련 내용은 **[도커 문서](https://docs.docker.com/desktop/release-notes/#docker-desktop-430){:target="_blank"}**에도 나와있다.
<img width="1380" alt="docker-doc" src="https://user-images.githubusercontent.com/98696463/200106581-da4eafd9-bfe2-4658-b810-d3acaec624ac.png">

docker-desktop 4.3.0 부터 cgroup v2 feature 를 사용한 것으로 보인다.

위 [Java 버그 리포트](https://bugs.openjdk.org/browse/JDK-8230305){:target="_blank"}에 따르면 이 버그는 Java 15 에서 패치됐지만 아주 다행히도 최근 (뒤늦게) **11.0.16 GA 버전에 백포팅됐다**고 한다. 아무래도 나름 크리티컬한 버그이고 11 은 LTS 여서 백포팅해준 듯 하다.

위 도커 테스트에서는 **docker-desktop 버전이 4.11.0** 으로 cgroup v2 feature 를 사용중일 것이고, Java 는 그에 대한 대응이 되지 않은 **11.0.4**, **13.0.2_8** 버전이어서 컨테이너 리소스 detection 이 제대로 되지 않았던 것이다.

그럼 15, 11.0.16 버전은 어떨까? 예상한대로 `MaxHeapSize` 를 잘 산출해낸다.

**openjdk15.0.2_7**
<img width="1352" alt="15-success" src="https://user-images.githubusercontent.com/98696463/200106867-5efa5eb0-0836-46b5-8694-28c86deeeafa.png">

**openjdk11.0.16.1_1**
<img width="1385" alt="11 0 16 - success" src="https://user-images.githubusercontent.com/98696463/200107033-ce85a711-4ffe-4556-b4c7-35ea31dc1a74.png">

### _**docker-desktop 3.6.0**_
자, 반대로 docker-desktop 버전을 내린다면 Java **11.0.4**, **13.0.2_8** 에서도 잘 동작해야 할 것이다.

**openjdk11.0.4_11**
<img width="1370" alt="11 0 4-success" src="https://user-images.githubusercontent.com/98696463/200107000-185571d7-7f5e-4220-8f67-014510999e95.png">

**openjdk13.0.2_8**
<img width="1350" alt="13-success" src="https://user-images.githubusercontent.com/98696463/200106991-22ca6f58-5305-4917-8db9-3aa8d90bf84d.png">

앞서 말했듯이 **docker-desktop 3.6.0** 은 cgroup v2 feature 를 사용하지 않는다.

따라서 cgroup v1 위에 설계된 Java **11.0.4**, **13.0.2_8** 의 메모리 detection 이 아까와는 다르게 잘 동작한다.


## **Kubernetes 와 JVM 애플리케이션의 OOMKilled**
그런데 아까 봤던 [아티클](https://kpow.io/articles/corretto-memory-issues/){:target="_blank"}에서 눈에 띄는 대목이 있다.
> <img width="1498" alt="third-party doc" src="https://user-images.githubusercontent.com/98696463/200107126-02bd76d7-2a86-4935-a54e-29283c10501c.png">
> Kubernetes 도 cgroup v2 feature 를 사용하는데, 위 [Container Awareness 버그](https://bugs.openjdk.org/browse/JDK-8230305){:target="_blank"}가 픽스되지 않은 버전의 JVM 을 구동하게 되면 (Pod 가 아닌) Node 기준으로 MaxHeapSize 를 산출할 수 있고 그렇게 되면 JVM 애플리케이션이 Pod resource limit 을 넘게되어 OOMKilled 당할 수도 있다.

> _위 도커 테스트에서 컨테이너 limit 인 1GB 와 관계없이 MaxHeapSize 가 쌩뚱맞은 6.13G 로 잡힌 것을 기억할 것이다._
{: .prompt-note }

과거 배포한 애플리케이션이 알 수 없는 이유로 OOMKilled 됐다고 했는데 급 PTSD 가 몰려왔다. 😇
동시에 애플리케이션으로 cgroup v2 대응이 되지 않은 openjdk 버전(11.0.4)을 사용한 것이 문제의 원인일 수도 있다는 생각이 들어 설레기도 했다.

그렇다면 도대체 Kubernetes 는 언제부터 cgroup v2 를 사용했고 무얼할 때 cgroup v2 를 사용하는지 궁금하여 [쿠버네티스 문서](https://kubernetes.io/docs/concepts/architecture/cgroups/){:target="_blank"}를 찾아봤다.
cgroup v2 를 사용한 건 **v1.25 [stable]** 버전부터인 것 같고 **Memory QoS** 에 cgroup v2 feature 를 사용한다고 한다.

> ### 쿠버네티스 문서
> <img width="887" alt="스크린샷 2022-11-05 오후 2 13 46" src="https://user-images.githubusercontent.com/98696463/200102403-1ec6a33f-b1a6-4d51-8c90-4f4a4c3be153.png">
> 그리고 문서 마지막에 대놓고 JDK 15 이후 버전 혹은 11.0.16 버전을 사용하라고 명시돼있다.
> (처음부터 이 문서를 봤었으면...😇)
>
> 그리고 [MemoryQoS 문서](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/){:target="_blank"}에는 다음과 같은 내용이 나와있다.
>
> <img width="1122" alt="memory QoS doc" src="https://user-images.githubusercontent.com/98696463/200108416-ab56e136-6ae3-4b76-b90a-f4f946ade4a2.png">

자, 어찌됐든 이제 새로 배포할 프로젝트 및 기존 Java11 애플리케이션의 마이너 버전을 변경 (**11.x.x -> 11.0.16**) 해야 할 이유를 찾은 것 같다!

## 정리
Container Awareness 가 제대로 이루어짐으로써 아래 두가지를 기대해 볼 수 있다.
- 이번에 배포할 Java 애플리케이션의 버전을 11.0.16 으로 업그레이드 한 뒤 위 JVM 옵션을 사용하면 "적절한" 애플리케이션 리소스를 산출할 때 더 정확한 도움을 받을 수 있을 것 같다. (사실 "기존 Java 버전의 resource detection 이 비정확했다"가 더 맞는 표현이다.)
- 더불어 Kubernetes 클러스터에서 알 수 없는 이유로 발생하던 OOMKilled 이슈도 단순 리소스를 상향 조정하는 미봉책 없이 알아서 해결될 수 있다고 살짝 기대해본다.


