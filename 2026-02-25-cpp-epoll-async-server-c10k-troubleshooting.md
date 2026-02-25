---
layout: post
title: "[C++ 백엔드 딥다이브] 프레임워크 없이 epoll 비동기 서버 구축하고 11만 RPS 돌파하기"
date: 2026-02-25
categories: [Backend, C++, OS, Network]
tags: [epoll, C10K, Multithreading, MemoryLeak, Troubleshooting, C++]
---

> "비동기 서버가 어떻게 동작하는지 진짜로 이해하고 있는가?"

수많은 백엔드 이력서들이 Java/Spring 생태계나 Node.js의 마법에 기대어 비동기(Asynchronous)와 동시성(Concurrency)을 논합니다. 하지만 런타임 환경과 가비지 컬렉터(GC)가 제공하는 보호막을 걷어냈을 때, 커널 레벨에서 발생하는 I/O 스로틀링과 메모리 경합을 직접 통제할 수 있는 엔지니어는 극소수입니다.

이 한계를 깨부수기 위해, 어떠한 웹 프레임워크의 도움도 없이 오직 **C++14와 Linux 커널 API(`epoll`), 그리고 POSIX 스레드만으로 C10K(1만 동시 접속) 문제를 돌파하는 Event-Driven 비동기 서버**를 밑바닥부터 설계하고 구현했습니다. 

이 글은 그 과정에서 마주한 멀티스레딩의 잔혹한 비결정성(Non-determinism)과, 이를 하드코어하게 튜닝해 낸 트러블슈팅(Autopsy) 기록입니다.

## 1. Core Architecture: Reactor Pattern

초고트래픽 서버의 심장을 뛰게 만들기 위해 철저한 역할 분담 아키텍처를 채택했습니다.


* **Event Demultiplexer (Main Thread):** Linux의 `epoll`을 엣지 트리거(`EPOLLET`) 모드로 가동하여 O(1)에 가까운 I/O 이벤트 감지 성능을 확보했습니다. 메인 스레드는 오직 클라이언트의 접속(`accept`)과 I/O 이벤트 감지만을 담당하며 시스템 콜 오버헤드를 극한으로 줄입니다.
* **Thread Pool (Worker Threads):** `std::condition_variable`과 `std::mutex`를 활용한 생산자-소비자(Producer-Consumer) 패턴을 구현하여, 메인 스레드가 던져주는 작업을 4개의 워커 스레드가 Non-blocking으로 처리합니다.

이론적인 뼈대는 완벽했지만, 대용량 파이프라인(10MB 연속 전송)을 꽂아 넣자마자 멀티스레드 프로그래밍의 지옥도가 열렸습니다.

## 2. 트러블슈팅 1: Race Condition과 Segmentation Fault

**[증상]** 단일 클라이언트가 10MB 데이터를 전송하다가 급작스럽게 연결을 끊었을 때 서버가 `Segmentation Fault`를 뱉으며 즉사했습니다.

**[부검 및 원인 분석]**
커널 I/O 로그를 트래킹한 결과, 워커 스레드가 커널 버퍼를 쥐어짜는(Drain) 도중 클라이언트가 `FIN/RST`를 날려버리는 타이밍 이슈였습니다. 
1. 클라이언트 연결 종료 시, 메인 스레드의 `epoll_wait`가 즉시 `EPOLLHUP`을 감지.
2. 메인 스레드가 맵(Map)에서 세션 원시 포인터(`*`)를 `delete`로 강제 해제.
3. 찰나의 순간 뒤, 데이터를 다 읽은 워커 스레드가 이미 날아간 메모리(Dangling Pointer)의 `fd`에 접근(`close`)하면서 OS가 프로세스를 강제 종료시켰습니다.

**[해결: 메모리 생명주기 동기화]**
소켓 생명주기를 커널(`epoll`)에만 의존하지 않고, 애플리케이션 레벨에서 **`std::shared_ptr`의 Reference Counting**을 전면 도입했습니다.

```cpp
// 메인 스레드의 무자비한 살인 멈추기
// delete session; (X)
sessions.erase(active_fd); // 맵에서만 제거

// 워커 스레드로 작업 투척 시 복사 캡처
std::shared_ptr<Session> session = sessions[active_fd]; 
threadPool.enqueue([session, epoll_fd]() {
    handle_client_io(session, epoll_fd); // 참조 카운트가 유지되어 절대 메모리가 날아가지 않음
});
```
워커 스레드의 작업 큐(Task Queue)와 메모리 참조 카운트를 완벽하게 동기화하여, "누군가 세션을 참조하고 있다면 절대 삭제되지 않는다"는 룰을 강제했습니다.

## 3. 트러블슈팅 2: Re-arming Window 틈새의 Heap Corruption
[증상] 세그폴트를 잡고 나니, 이번엔 double free or corruption (!prev) 에러와 함께 프로세스가 중단(Aborted)되었습니다.

[부검 및 원인 분석]
EPOLLONESHOT 플래그를 사용해 동일 소켓에서 이벤트가 중복 발생하는 것을 막았으나, 커널의 방어막이 풀리는 찰나의 틈새(Re-arming Window)가 문제였습니다.
워커 스레드 A가 처리를 마치고 EPOLL_CTL_MOD로 소켓을 재장전(Re-arming)하는 즉시 커널의 감시가 재개됩니다. 하지만 스레드 A가 함수 스코프를 완전히 벗어나기 전(0.0001초의 틈), 파이썬 스크립트의 10MB 폭격을 감지한 메인 스레드가 작업을 다시 큐에 던지고 스레드 B가 동일 세션에 진입해 버립니다. 두 스레드가 하나의 힙 메모리에 동시 접근하여 double free가 발생했습니다.

[해결: 이중 방어선 구축 (Belt and Suspenders)]
커널 레벨(EPOLLONESHOT)의 방어에만 의존하지 않고, 유저 레벨 방어막(std::mutex)을 추가했습니다.


```cpp
void handle_client_io(std::shared_ptr<Session> session, int epoll_fd) {
    // 2차 방어선: 데이터 경합 및 Heap Corruption 원천 차단
    std::lock_guard<std::mutex> lock(session->session_mutex);
    
    // ... I/O 처리 및 커널 버퍼 Drain ...

    // 1차 방어선: 처리가 무사히 끝난 후 커널 감시 재장전
    if (!is_disconnected) {
        struct epoll_event event;
        event.data.fd = session->fd;
        event.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
        epoll_ctl(epoll_fd, EPOLL_CTL_MOD, session->fd, &event);
    }
}
```
객체 자체에 락(Lock)을 걸어 스레드 오버랩(Overlapping Execution)을 물리적으로 차단하는 완벽한 Thread-safety를 달성했습니다.

## 4. 압도적인 성능 증명: 11만 RPS 돌파
모든 논리적 버그를 박살 낸 후, wrk 부하 테스트 툴을 사용하여 C10K 스트레스 테스트를 진행했습니다.

테스트 환경: 4 Threads, 1000 Concurrent Connections, 30s Duration

벤치마크 결과:

```text
Running 30s test @ [http://127.0.0.1:8080/](http://127.0.0.1:8080/)
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     8.76ms    3.98ms  51.66ms   80.78%
    Req/Sec    28.66k     5.23k   90.32k    75.10%
  3424409 requests in 30.10s, 169.82MB read
Requests/sec: 113770.06
Transfer/sec:      5.64MB
```
초당 113,770건의 요청을 에러율 0%로 방어해 냈습니다. 1,000명의 동시 접속자가 몰려드는 상황에서도 힙 오염이나 메모리 누수 없이 평균 지연 시간(Latency) 8.76ms를 기록하며 극강의 로드밸런싱을 증명했습니다.

프레임워크의 오버헤드를 제거한 순수 C++과 리눅스 커널 API의 압도적인 물리력을 확인한 순간입니다.

## 5. 결론 및 Next Step
'동작하는 코드'에 만족하지 않고, 커널의 동작 방식과 멀티스레드의 치명적인 취약점을 로우 레벨에서 부검하고 통제하는 경험을 얻었습니다.

현재 시스템콜 횟수 최적화 및 accept 병목(Thundering Herd) 현상 등의 한계를 분석 중이며, 향후 OSTEP 커널 이론을 결합하여 이 11만 RPS의 한계를 박살 내는 2단계 아키텍처 고도화를 진행할 예정입니다.

🔗 전체 소스 코드 (GitHub): https://github.com/hscho-engineer/epoll-async-c10k-server
