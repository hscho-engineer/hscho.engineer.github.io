---
layout: post
title: "비동기 병렬 시스템의 병목: 위상 정렬과 임계 경로(Critical Path) 설계"
subtitle: "단순한 알고리즘을 넘어, 커널 스케줄러의 의존성 해결 아키텍처 부검하기"
description: "멀티스레드 병렬 처리 환경에서 발생하는 의존성(Dependency) 문제와, 가장 늦게 끝나는 프로세스가 전체의 병목이 되는 임계 경로(Critical Path)를 위상 정렬과 DP로 통제하는 로직을 분석합니다."
permalink: /critical-path-architecture/
date: 2026-02-26
categories: [Backend, Algorithm, OS, Architecture]
tags: [TopologicalSort, DP, CriticalPath, Asynchronous, Multithreading, Scheduling]
---

> "모든 스레드를 동시에 비동기로 던진다고 시스템이 무조건 빨라지는가?"

수많은 백엔드 서버들이 트래픽을 처리하기 위해 멀티스레드와 비동기(Asynchronous) 큐에 작업을 밀어 넣습니다. 하지만 작업 간에 선후 관계(Dependency)가 존재할 때, 시스템의 최종 완료 시간은 평균 처리 속도가 아니라 **'가장 늦게 끝나는 단 하나의 최악의 병목'**에 의해 지배당합니다.

이를 시스템 엔지니어링에서는 **임계 경로(Critical Path)**라고 부릅니다. 

이 글은 C++의 큐(Queue) 기반 위상 정렬(Topological Sort)과 동적 계획법(DP)을 결합하여, OS 스케줄러가 데드락(Deadlock) 없이 병렬 프로세스의 의존성을 해결하고 임계 경로를 찾아내는 로직을 로우 레벨에서 부검(Autopsy)한 기록입니다.

## 1. Core Architecture: 의존성 그래프와 위상 정렬

작업(Task) 간의 선후 관계를 통제하기 위해 시스템은 방향 비순환 그래프(DAG, Directed Acyclic Graph)를 구축해야 합니다.

* **진입 차수(In-degree):** 특정 프로세스가 실행되기 위해 반드시 먼저 끝나야 하는 선행 작업의 개수입니다.
* **Ready Queue:** 진입 차수가 0인 작업, 즉 '지금 당장 CPU에 올려 실행 가능한 프로세스'들을 담아두는 버퍼입니다.

작업이 하나 끝날 때마다 연결된 다음 작업들의 진입 차수를 1씩 깎아내며, 새롭게 0이 된 작업을 큐에 밀어 넣는 Kahn's Algorithm이 이 아키텍처의 뼈대입니다. 하지만 단순히 '순서'를 정하는 것만으로는 비동기 시스템의 소요 시간을 계산할 수 없습니다.

## 2. 병목의 발견: DP를 통한 임계 경로(Critical Path) 갱신

![critical path](https://github.com/user-attachments/assets/ca8a00d9-7596-4153-aa5e-def50d5f99f8)

A작업(10초)과 B작업(100초)이 모두 끝나야 C작업을 실행할 수 있다고 가정해 봅시다. A와 B를 별도의 스레드에서 동시에 실행하더라도, C는 A가 끝나는 10초 뒤가 아니라 **B가 끝나는 100초 뒤에야 비로소 실행이 가능해집니다.**

즉, 다음 노드로 진행할 때 **"나로 들어오는 여러 선행 작업 중, 가장 늦게 끝나는(Max) 최악의 병목 시간을 내 메모리에 덮어쓰며 캐싱(Caching)해야 한다"**는 완벽한 점화식이 성립합니다.

이를 C++ 코드의 코어 로직으로 표현하면 다음과 같습니다.

```cpp
// cur: 방금 끝난 선행 작업
// nxt: 다음에 실행할 작업
// dp: 해당 작업이 실행 가능해지기까지 걸리는 절대적인 누적 시간 캐시

dp[nxt] = max(dp[nxt], dp[cur] + build_time[nxt]);
```
이 한 줄의 코드가 전체 파이프라인의 멱등성(Idempotency)을 보장하며, 병렬 처리 환경의 가장 치명적인 병목 지점을 메모리에 덮어씁니다.
## 3. C++ Memory Control: 객체 재할당 억제

이 로직을 수백만 번의 I/O가 발생하는 테스트 환경(BOJ 1005: ACM Craft)에서 증명하기 위해, 메모리 통제에 극도로 신경을 썼습니다.

그래프를 나타내는 vector<int> adj 배열을 매번 지역 변수(Stack)로 선언하거나 동적 할당(Heap)하게 되면, 극심한 메모리 오버헤드와 스택 오버플로우(Stack Overflow)의 위험에 노출됩니다. 이를 방지하기 위해 전역 데이터 영역(Data Segment)에 넉넉한 메모리를 미리 확보하고, 이전 컨텍스트의 찌꺼기만 초기화하는 방식을 채택했습니다.
```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <algorithm>

using namespace std;

const int MAX_NODE = 1005;
int build_time[MAX_NODE];
int indegree[MAX_NODE];
int dp[MAX_NODE];
vector<int> adj[MAX_NODE]; // 스택 오버플로우 방지를 위한 전역 메모리 확보

int main() {
    ios::sync_with_stdio(0);
    cin.tie(0);
    
    int t;
    cin >> t;
    while(t--) {
        int n, k, w;
        cin >> n >> k;
        
        // 1. Memory Format: 이전 컨텍스트의 찌꺼기(Garbage)를 완벽하게 포맷
        fill(build_time, build_time + n + 1, 0);
        fill(indegree, indegree + n + 1, 0);
        fill(dp, dp + n + 1, 0);
        for(int i = 1; i <= n; i++) {
            adj[i].clear(); // Capacity는 유지하되 Size만 0으로 밀어 할당 오버헤드 억제
        }
        
        for(int i = 1; i <= n; i++) cin >> build_time[i];
        
        for(int i = 0; i < k; i++) {
            int u, v;
            cin >> u >> v;
            indegree[v]++;
            adj[u].push_back(v);
        }
        
        queue<int> q;
        for(int i = 1; i <= n; i++) {
            if(indegree[i] == 0) {
                q.push(i);
                dp[i] = build_time[i];
            }
        }
        
        cin >> w;
        
        // 2. Scheduler Runtime: 비동기 병목 탐색 시작
        while(!q.empty()) {
            int cur = q.front();
            q.pop();
            
            for(int nxt : adj[cur]) {
                indegree[nxt]--;
                if(indegree[nxt] == 0) q.push(nxt);
                
                // Critical Path 덮어쓰기: 가장 늦게 끝나는 선행 프로세스의 시간을 캐싱
                dp[nxt] = max(dp[nxt], dp[cur] + build_time[nxt]);
            }
        }
        
        cout << dp[w] << '\n';
    }
    return 0;
}
```
## 4. 결론
단순한 순서 정렬을 넘어, 각 노드의 소요 시간과 병렬 실행의 특성을 고려하여 시스템의 최종 병목 지점(Critical Path)을 찾아내는 아키텍처를 C++ 코드로 증명했습니다.

이러한 의존성 관리 로직은 리눅스의 make 빌드 시스템부터, 대용량 분산 시스템의 MSA(Microservices Architecture) 트랜잭션 추적에 이르기까지 코어 엔지니어링의 근본이 되는 필수 설계 패턴입니다.
