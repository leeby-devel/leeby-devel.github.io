---
title: JVM Container Awareness Bug (cgroup v2) > JDK 11.0.16
author:
name: leeby
link: https://github.com/leeby-devel
date: 2022-11-05 16:54:00 +0900
categories: [JVM]
tags: [JDK, JDK11, JVM, Kubernetes, docker, Container Awareness, cgroup]
---

## **저처럼 글 읽기를 싫어하시는 분들을 위한 결론**

> - Kubernetes 환경에서 JDK 11 을 사용중이라면 **JDK 11.0.16** 이상의 버전 사용을 권장한다.
> - JDK 12 - 14 버전을 사용중이라면 15 이상의 버전 사용을 권장한다.
> - 그렇지 않으면 소중한 Pod 가 OOMKilled 당할 수 있다.
>
> 특히 `InitialRAMPercentage`, `MaxRAMPercentage` 옵션을 사용한다면 꼭!
>
> <img width="887" alt="스크린샷 2022-11-05 오후 2 13 46" src="https://user-images.githubusercontent.com/98696463/200102403-1ec6a33f-b1a6-4d51-8c90-4f4a4c3be153.png">

> 글이 깁니다. 다음 내용에 대해 구구절절 설명합니다.
> - k8s 환경에서 JDK 11 구동시, 왜 JDK 11.0.16 이상의 버전을 사용해야 하는지
> - 이 내용을 알아가게 된 과정
{: .prompt-warning }


## **"적절한" 리소스 산정을 위한 여정...**

최근 회사에서 메인 프로젝트 배포를 앞두고 나름대로 배포 프로세스를 정비 중이었다. 배포할 애플리케이션은 서비스에서 발생하는 대부분의 트래픽을 처리하는 메인 애플리케이션이었고, 이 거대한 시스템을 새로운 언어로 포팅하여 마침내 배포를 앞둔 단계였다. 때문에 Kubernetes Pod 의 "적절한" 리소스 산정 작업이 꼭 필요했다.

<img width="291" alt="스크린샷 2022-11-05 오후 2 14 59" src="https://user-images.githubusercontent.com/98696463/200102438-fe04a755-9691-4fe7-b108-c6504a3834f7.png">
_**"적절한"** 리소스 산정은 어떻게 해야 할까?_

### Pod 야.. OOMKilled 로부터 지켜주지 못해 미안해

> <img width="850" alt="스크린샷 2022-11-05 오후 2 25 13" src="https://user-images.githubusercontent.com/98696463/200103865-c62e94a8-57e6-46ed-ac82-972f739975f8.png">
> _Pod 가 왜 설정한 메모리 limit (8Gi) 보다 더 많은 메모리를 사용하다 죽는걸까? ~~분명 `MaxRAMPercentage` == 70(%) 으로 지정했는데 😥~~_
>
> 과거 JVM 기반의 애플리케이션 리소스 산정을 위해 JMX 로 애플리케이션 모니터링도 해보고 Heap 덤프도 했었다. 나름의 삽질을 통해 리소스를 산정하여 k8s 위에 애플리케이션을 올렸지만 종종 알 수 없는 이유로 내 소중한 Pod 가 OOMKilled 당하는 모습을 목격해야 했고, 결국엔 리소스 상향이라는 미봉책으로 여차 저차 땜질했던 기억이 다시 떠올랐다.



## **InitialRAMPercentage, MaxRAMPercentage 그리고 JDK 11**

<img width="850" alt="스크린샷 2022-11-05 오후 2 31 48" src="https://user-images.githubusercontent.com/98696463/200104138-f10b7f60-d70d-412b-ae84-c499facc7f91.png">

지난번 부족했던 점을 복기하고자, 당시 사내 게시판에 작성한 삽질기에 전 파트 동료분이 작성해주신 댓글과 링크를 열심히 곱씹어 봤다. 당시 애플리케이션의 Heap 메모리 조절을 위한 JVM 옵션 값으로 `InitialRAMPercentage`, `MaxRAMPercentage` 를 사용하고 있었는데, `xmx`, `xms` 옵션보다 유동적이라는 이유에서였다.
> 힙을 비율로 지정하면 k8s 메모리 리소스를 조정할 때 따로 힙 크기를 손대지 않아도 되기 때문이다.

### **JDK 13 부터 해당 옵션을 사용할 수 있다? (X)**

<img width="850" alt="스크린샷 2022-11-05 오후 2 38 45" src="https://user-images.githubusercontent.com/98696463/200104272-5ce53656-e6a3-4999-8d9e-bb29039ea4ce.png">
_궁금해서 코멘트에 참조된 **[버그 리포트](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=8223957){:target="_blank"}**를 쭉 읽어봤다._

>1. **_시스템(컨테이너) 메모리가 JVM MaxRAM 보다 큰 경우_** 위 옵션은 시스템 메모리가 아닌 MaxRAM 을 기준으로 MazHeapSize 가 결정된다.
>       (`MaxHeapSize` = `MaxRAM` * `(MaxRAMPercentage) / 100`)
>
>2. 1번 문제가 해결되더라도 MaxRAMPercentage 로 설정된 **_MaxHeapSize 가 [32g 를 넘어갈 수 없도록](https://brunch.co.kr/@alden/35){:target="_blank"}_** 되어있다. 이는 기본적으로 활성화돼있는 `UseCompressedOops` 옵션이 해당 옵션들 보다 우선되기 때문이다.

내용을 정리하면, **JDK 13 이하 버전에서 `InitialRAMPercentage`, `MaxRAMPercentage` 옵션을 사용할 수 없다는 내용이 아니라**, 일부 조건 하에서 Heap 비율 산정이 제대로 이루어지지 않는 다는 내용의 버그리포트이다.

### **JDK 11 에서 사용 가능하다.**
특정 조건이란 다음 두가지인데 우리 애플리케이션은 이에 해당되지 않음을 확인했다.
1. 시스템 메모리가 `MaxRAM` 보다 큰 경우
2. Heap 사이즈가 32g 를 넘어가는 경우

> <img width="1718" alt="스크린샷 2022-10-14 오전 2 09 53" src="https://user-images.githubusercontent.com/98696463/200104787-0729c5f0-cd25-474c-87e6-52c0273d5d9b.png">
> _확인해 보니 `MaxRAM` 의 default 값은 128Gi 이다._
> 1. 우리는 `MaxRAM` 보다 작은 10Gi 내외의 시스템 메모리 (= 파드 리소스 기준) 를 사용한다. 시스템 메모리가 더 작을 땐 MaxRAM 이 아닌 시스템 메모리를 기준으로 MaxHeapSize 를 산출한다.
>
> 2. 앞서 말했듯 시스템 메모리는 10Gi 내외로 산정할 것이기 때문에 힙사이즈가 32g 이상 될 일이 없다.

이를 통해 `InitialRAMPercentage`, `MaxRAMPercentage` 옵션 사용에 문제가 없음을 확인했다.

## **이상한 MaxHeapSize**

위 흐름을 이어 받아, 이전에 OOMKilled 로 고생하던 애플리케이션의 openjdk 버전에 `MaxRAMPercentage` 옵션을 넣고 MaxHeapSize 를 출력해봤다.

- 컨테이너 메모리: 1GB
- `MaxRAMPercentage`: 80

### **docker-desktop 4.3.0**

<img width="1321" alt="11 0 4-failed" src="https://user-images.githubusercontent.com/98696463/200109639-f75e41e6-1609-4565-9a16-139e506596fd.png">
_**openjdk11.0.4_11**_

`MaxHeapSize` == 6.13G 라는 엉뚱한 수치가 나왔다. (..???)

<img width="1575" alt="스크린샷 2022-11-05 오후 4 48 47" src="https://user-images.githubusercontent.com/98696463/200109567-b836840e-bdfb-486c-8914-e443d4e230d9.png">
_**openjdk13.0.2_8**_

~~심지어~~ JDK 13 도 마찬가지다.

### **JVM Container Awareness 버그**

이 이상한 현상에 대해 또 다시 구글링을 해봤다.

[아티클](https://kpow.io/articles/corretto-memory-issues/){:target="_blank"} 하나를 보고 이 또한 **[버그](https://bugs.openjdk.org/browse/JDK-8230305){:target="_blank"}**임을 알게 됐다. 버그 내용을 요약하면, 다음과 같다.

> jvm 의 컨테이너 리소스 detection 은 cgroup v1 위에 설계됐지만 cgroup v1 을 사용하던 Docker 가 새 버전을 릴리즈하면서 일부 cgroup v2 feature 를 사용하게 되었다.
> 따라서 그 위에서 **cgroup v2 호환성 대응이 되어있지 않은 버전의 jvm 을 구동하면 컨테이너 리소스 detection 에 문제가 생길 수 있다.**

> _cgroup은 단일 또는 태스크 단위의 프로세스 그룹에 대한 자원 할당을 제어하는 리눅스 커널 모듈이고 v1, v2 가 있다._
> _ref. [https://sonseungha.tistory.com/535](https://sonseungha.tistory.com/535){:target="_blank"}_
{: .prompt-note }

<img width="1380" alt="docker-doc" src="https://user-images.githubusercontent.com/98696463/200106581-da4eafd9-bfe2-4658-b810-d3acaec624ac.png">
_관련 내용이 **[도커 문서](https://docs.docker.com/desktop/release-notes/#docker-desktop-430){:target="_blank"}**에도 나와있다. docker-desktop 4.3.0 부터 cgroup v2 feature 를 사용한 것으로 보인다._

위 [openJDK 버그 리포트](https://bugs.openjdk.org/browse/JDK-8230305){:target="_blank"}에 따르면 이 버그는 JDK 15 에서 패치됐지만 아주 다행히도 최근 **11.0.16 GA 버전에 백포팅됐다**고 한다. 아무래도 컨테이너 환경에서 구동되는 애플리케이션들이 많은 요즘에는 크리티컬한 버그이고 11 은 LTS 여서 백포팅해준 듯 하다.

위 도커 테스트에서는 **docker-desktop 버전이 4.11.0** 으로 cgroup v2 feature 를 사용중일 것이고, JVM 는 그에 대한 대응이 되지 않은 **11.0.4**, **13.0.2_8** 버전이어서 컨테이너 리소스 detection 이 제대로 되지 않았던 것이다.

그럼 JDK 15 와 11.0.16 버전은 힙 사이즈 산출을 제대로 할까?

<img width="1352" alt="15-success" src="https://user-images.githubusercontent.com/98696463/200106867-5efa5eb0-0836-46b5-8694-28c86deeeafa.png">
_**openjdk15.0.2_7**_

<img width="1385" alt="11 0 16 - success" src="https://user-images.githubusercontent.com/98696463/200107033-ce85a711-4ffe-4556-b4c7-35ea31dc1a74.png">
_**openjdk11.0.16.1_1**_

예상대로 `MaxHeapSize` 를 잘 산출해낸다.

### **docker-desktop 3.6.0**
그럼 반대로 docker-desktop 버전을 내린다면 JDK **11.0.4**, **13.0.2_8** 에서도 잘 동작해야 할 것이다.

<img width="1370" alt="11 0 4-success" src="https://user-images.githubusercontent.com/98696463/200107000-185571d7-7f5e-4220-8f67-014510999e95.png">
_**openjdk11.0.4_11**_

<img width="1350" alt="13-success" src="https://user-images.githubusercontent.com/98696463/200106991-22ca6f58-5305-4917-8db9-3aa8d90bf84d.png">
_**openjdk13.0.2_8**_

앞서 말했듯이 **docker-desktop 3.6.0** 은 cgroup v2 feature 를 사용하지 않는다. 따라서 cgroup v1 위에 설계된 JDK **11.0.4**, **13.0.2_8** 의 메모리 detection 이 아까와는 다르게 잘 동작한다.


## **Kubernetes 와 JVM 애플리케이션의 OOMKilled**
그런데 아까 봤던 [아티클](https://kpow.io/articles/corretto-memory-issues/){:target="_blank"}에서 눈에 띄는 대목이 있다.
> <img width="1498" alt="third-party doc" src="https://user-images.githubusercontent.com/98696463/200107126-02bd76d7-2a86-4935-a54e-29283c10501c.png">
> Kubernetes 도 cgroup v2 feature 를 사용하는데, 위 [Container Awareness 버그](https://bugs.openjdk.org/browse/JDK-8230305){:target="_blank"}가 픽스되지 않은 버전의 JVM 을 구동하게 되면 (Pod 가 아닌) Node 기준으로 MaxHeapSize 를 산출할 수 있고 그렇게 되면 JVM 애플리케이션이 Pod resource limit 을 넘게되어 OOMKilled 당할 수도 있다.

> _도커 테스트에서 컨테이너 limit 인 1GB 와 관계없이 MaxHeapSize 가 쌩뚱 맞게 6.13G 로 잡힌 것을 기억할 것이다._
{: .prompt-note }

앞서 과거에 배포한 애플리케이션이 알 수 없는 이유로 OOMKilled 됐다고 했는데, 문제의 원인이 cgroup v2 대응이 되지 않은 openjdk 버전(11.0.4)을 사용한 것일 수도 있다는 생각이 들어 두근두근(?)하기도 했다.

그렇다면 Kubernetes 는 언제부터 cgroup v2 를 사용했고 무얼할 때 cgroup v2 를 사용하는지 궁금하여 [쿠버네티스 문서](https://kubernetes.io/docs/concepts/architecture/cgroups/){:target="_blank"}를 찾아봤다.
cgroup v2 를 사용한 건 **v1.25 [stable]** 버전부터인 것 같고 **Memory QoS** 에 cgroup v2 feature 를 사용한다고 한다.

> ### 쿠버네티스 문서
> <img width="887" alt="스크린샷 2022-11-05 오후 2 13 46" src="https://user-images.githubusercontent.com/98696463/200102403-1ec6a33f-b1a6-4d51-8c90-4f4a4c3be153.png">
> _문서 마지막에 대놓고 JDK 15 이후 버전 혹은 11.0.16 버전을 사용하라고 명시돼있다. (처음부터 이 문서를 봤었으면...😇)_
>
> <img width="1122" alt="memory QoS doc" src="https://user-images.githubusercontent.com/98696463/200108416-ab56e136-6ae3-4b76-b90a-f4f946ade4a2.png">
> _그리고 [MemoryQoS 문서](https://kubernetes.io/blog/2021/11/26/qos-memory-resources/){:target="_blank"}에는 위와 같은 내용이 나와있다._

어찌됐든 이제 새로 배포할 프로젝트 및 기존 JDK 11 애플리케이션의 마이너 버전을 변경 (**11.x.x -> 11.0.16**) 해야 할 이유를 찾았다.

## 정리
Container Awareness 가 제대로 이루어짐으로써 아래 두가지를 기대해 볼 수 있다.
- 이번에 배포할 애플리케이션 JDK 버전을 11.0.16 으로 업그레이드 하고 위 JVM 옵션을 사용하면 "적절한" 리소스 산출시 보다 더 정확한 도움을 받을 수 있을 것 같다. (이전 JDK 버전의 resource detection 이 부정확했기 때문에)

- 더불어 Kubernetes 클러스터에서 알 수 없는 이유로 발생하던 OOMKilled 이슈도 단순 리소스를 상향 조정하는 미봉책 없이 자연스럽게 해결될 수 있다고 살짝 기대해본다.
