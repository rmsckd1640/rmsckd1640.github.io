---
title: "위상 정렬(Topology Sort)"
date: 2026-05-07 23:47:28 +0900
series: "알고리즘"
tags: ["알고리즘"]
math: true
summary: "위상 정렬은 사이클이 없는 방향 그래프에서 모든 노드를 방향성에 거스르지 않도록 순서대로 나열하는 것을 의미한다. 쉽게 말해, '일의 순서'가 정해져 있는 작업을 차례대로 수행해야 할 때, 그 순서를 결정하기 위해 사용하는 알고리즘이다. 일반적으로 큐를 사용하여 구현하며, 과정은 다음과 같다. 그래프의"
---

위상 정렬은 사이클이 없는 방향 그래프에서 모든 노드를 방향성에 거스르지 않도록 순서대로 나열하는 것을 의미한다.

쉽게 말해, '일의 순서'가 정해져 있는 작업을 차례대로 수행해야 할 때, 그 순서를 결정하기 위해 사용하는 알고리즘이다.

### 동작 원리

일반적으로 큐를 사용하여 구현하며, 과정은 다음과 같다.

1. 그래프의 모든 노드에 대해 진입 차수를 계산한다.

2. 진입 차수가 0인 노드(선행 조건이 없는 노드)를 모두 큐에 넣는다.

3. 큐가 빌 때까지 다음 과정을 반복한다.

	- 큐에서 원소를 꺼내 출력(또는 결과 리스트에 추가)한다. 즉, 큐에서 꺼내지는 순서가 최종 결과이다.
    
    - 해당 노드와 연결된 모든 간선을 제거한다.
    
    - 간선 제거 후 진입 차수가 0이 된 노드를 새롭게 큐에 넣는다.
    
**주의: 모든 노드를 방문하기 전에 큐가 빈다면, 그래프에 사이클이 존재한다는 의미이다. 사이클이 존재한다면 위상 정렬이 불가능하다.**

### 시간복잡도

노드의 개수를 $$V$$, 간선의 개수를 $$E$$라고 할 때 모든 노드를 확인하며 각 노드에서 나가는 간선을 차례대로 제거하기 때문에(모든 노드와 간선을 한 번씩만 방문하기 때문에) $$O(V+E)$$이다.

### 문제

백준 2252번 줄 세우기

### 풀이

```java
import java.util.*;

public class Main{
	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		int n = sc.nextInt();
		int m = sc.nextInt();
		
		List<List<Integer>> graph = new ArrayList<>();
		
		for(int i=0; i<=n; i++) {
			graph.add(new ArrayList<>());
		}
		
		int[] degree = new int[n+1];
		
		for(int i=0; i<m; i++) {
			int from = sc.nextInt();
			int to = sc.nextInt();
			
			graph.get(from).add(to);
			
			degree[to]++;
		}
		
		Deque<Integer> q = new ArrayDeque<>();
		
		for(int i=1; i<=n; i++) {
			if(degree[i] == 0) q.offer(i);
		}
		
		StringBuilder sb = new StringBuilder();
		int cnt = 0; // 나중에 사이클 존재 여부 확인용
		while(!q.isEmpty()) {
			int cur = q.poll();
			cnt++;
			sb.append(cur+" ");
			
			//graph.get(cur).size()를 구해 for문을 돌리는 대신
			//for-each문을 사용하면 좋다.
			for(int temp : graph.get(cur)) {
				degree[temp]--;
				if(degree[temp] == 0) q.offer(temp);
				/**
				 *	리스트에서 실제로 간선을 삭제할 필요 없음.
				 *	이미 큐에서 꺼낸 노드는 다시 방문하지 않기 때문.
				 *	또한, 만약 리스트를 직접 건드리면 에러가 날 확률이 높아짐.
				 */
			}
		}
		
		if(cnt != n) System.out.println(-1);
		else System.out.println(sb.toString().trim());
	}
}
```
