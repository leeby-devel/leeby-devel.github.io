---
title: JEP 346 > unused (미사용) committed memory 를 OS 로 반환
author:
name: leeby
link: https://github.com/leeby-devel
date: 2024-11-10 09:39:00 +0900
categories: [JVM, GC]
tags: [JEP, JVM, JDK, G1GC, GC]
---

## 들어가며...
최근 회사에서 JDK, Spring Boot, Kotlin, Gradle 등 기반 시스템 버전 업그레이드하는 작업을 주도했다.
그 중 JDK 버전은 `11 → 17`, `17 → 21` 두 단계로 진행할 계획이었고, 계획대로 JDK 17 부터 버전 업데이트를 했다.
업데이트 후 애플리케이션 메트릭을 살펴보는데 Heap 메모리 풋프린트가 이전과 다른 양상을 보였고, 이번 포스트에서 해당 내용을 다뤄보고자 한다.

## JVM Heap 메모리
먼저 JVM 힙메모리 상태에 대한 용어부터 짚고 가는 게 좋을 것 같다. Grafana JVM Micrometer 기준으로 JVM 힙 메모리는 `used`, `committed`, `max` (= reserved) 로 나뉜다.

<img width="800" alt="jvm-micrometer" src="https://github.com/user-attachments/assets/b6ce583d-4a84-41eb-b208-276722b76c82">
_committed == max 인 상태_

> - `used`: 애플리케이션 힙의 라이브 오브젝트들에 의해 점유되고 있는 메모리이다.
> - `RSS (of heap)`: 애플리케이션이 물리적으로 (실제로) 점유하고 있는 메모리 영역이다. committed 까지 확장될 수 있다. Grafana 에서 표현되지 않으므로 가상의 선을 그었다.
> - `committed`: 애플리케이션이 OS 로 부터 미리 할당받아 놓은 메모리 영역이다. 실제로 사용(RSS)되고 있을 수도 있고 아닐 수도 있다. reserved 까지 확장될 수 있다.
> - `reserved` (= max): 애플리케이션이 OS 로부터 할당받을 수 있는 메모리의 최대 크기이다.

## 기존 (JDK 11) 까지의 메모리 설정
JDK 11 까지는 `InitialRAMPercentage` 를 `MaxRAMPercentage` 와 같게 두거나, `xms` 값을 `xmx` 값과 같게 두는 것이 권장되었다.
그 이유는 다음과 같은 오버헤드를 피하기 위해서였다.

```
committed 크기가 작다.
→ GC 주기가 짧아지고 이는 오버헤드로 작용한다.

committed 크기가 reserved 와 달라 필요에 따라 reserved 까지 점진적으로 크기가 확장된다.
→ OS 로부터 메모리를 할당받는 과정에서 오버헤드가 발생한다.
```
<details>
<summary><strong>👉 부연 설명</strong></summary>
(G1GC 기준) GC Cycle 은 committed 메모리 범위 내에서 수행된다. 따라서 committed 메모리 크기가 작으면 그 만큼 GC 주기가 짧아진다.
엄밀하게는 GC Cycle 은 사용중인 메모리 (RSS) 범위 내에서 수행되고 추가 메모리가 필요하면 RSS 가 committed 만큼 확장되는 것이다.
이후에 committed 메모리로도 부족하면 OS 에 추가 메모리를 요청하고, 이 크기는 reserved 만큼 늘어날 수 있다.
</details>
\
덧붙여 JDK 11 까지는 **한번 committed 된 메모리는 OS 로 반환되지 않았다**.

즉, 애플리케이션이 에이징됨에 따라 committed 영역의 크기가 늘어났지만 이후에 실제로 메모리가 사용되지 않더라도 OS 로 반환되지는 않았다.

## JEP 346 > 메모리 관리 패턴 개선
이러한 메모리 사용 패턴은 사용량만큼 비용을 지불하는 클라우드 환경에서는 큰 단점으로 작용한다.
JEP 346 에서는 이러한 점을 개선하기 위해, 사용되지 않는 메모리는 OS 로 반환되도록 G1GC 의 메모리 사용 패턴에 변화를 줬다.
위 내용은 [JEP 346](https://openjdk.org/jeps/346){:target="_blank"} 의 Motivation 에 잘 나와있다.

JEP 346 개선사항은 **JDK 12 부터 적용**되었기 때문에 `JDK 11 → 17` 업데이트 이후에 현상을 발견한 것이었다.

### **개선 전후 메트릭 비교**
다음은 JDK 버전 외에 JVM 설정 값 등 모든 환경이 같은 동일 애플리케이션 메트릭이다.

<img width="800" alt="before-jdk12" src="https://github.com/user-attachments/assets/09f9470a-16ff-47c5-a4a0-a8ee2c49fa7d">
_JDK <= 11_

<img width="800" alt="after-jdk12" src="https://github.com/user-attachments/assets/f93354b0-0d04-4198-8a63-952e3408c468">
_JDK >= 12_

두 환경 모두 `InitialRAMPercentage`, `MaxRAMPercentage` 두 값을 동일하게 75% 로 두었다.
눈여겨 볼 점은 **JDK >= 12 에서는 `InitialRAMPercentage` 옵션이 기대와 같이 동작하지 않는 다는 점**이다.
예상 대로면 committed, reserved (= max) 풋프린트가 같아야 하기 때문이다.

### **InitialRAMPercentage 사용성 이해**
이는 **JEP 346 에서 메모리 관리 기능이 변경되었고 이 기능이 우선**되어 `InitialRAMPercentage` 를 아무리 높게 설정하여도 실제 사용되지 않는 메모리는 committed 풋프린트로 잡히지 않게 된 것이다.

따라서 **JDK 12 이후부터는 `InitialRAMPercentage` 값을 `MaxRAMPercentage` 값과 같게 (= 높게) 두는 것은 아무 의미가 없어졌다.**

다만 `InitialRAMPercentage` 옵션은 여전히 초기 힙 메모리를 설정하는 옵션으로서 의미가 있다.
JEP 346 개선사항으로 인해 `InitialRAMPercentage` 옵션이 직접적으로 변경된 것은 아니지만 간접적인 영향을 받았고, **옵션의 사용 맥락에 있어 약간의 변화가 생겼다고 이해**하면 될 것 같다.

