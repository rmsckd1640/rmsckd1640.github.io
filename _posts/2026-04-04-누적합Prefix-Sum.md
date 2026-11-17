---
title: "누적합(Prefix Sum) 알고리즘"
date: 2026-04-04 11:42:36 +0900
series: "알고리즘"
tags: ["알고리즘"]
math: true
summary: "누적합 알고리즘은 배열의 특정 구간 합을 효율적으로 구하기 위한 기법이다. 매번 구간을 순회하며 합을 구하는 대신, 미리 누적합 배열을 만들어 각 구간의 합을 O(1)에 계산할 수 있도록 한다. 크기가 N인 원본 배열을 한 번 순회하며 누적합 배열을 생선한다. M개의 구간 합 요청에 대해 각각 뺄셈"
---

## 누적합(Prefix Sum)

누적합 알고리즘은 배열의 특정 구간 합을 효율적으로 구하기 위한 기법이다.

매번 구간을 순회하며 합을 구하는 대신, **미리 누적합 배열을 만들어** 각 구간의 합을 $$O(1)$$에 계산할 수 있도록 한다.

크기가 $$N$$인 원본 배열을 한 번 순회하며 누적합 배열을 생선한다.

$$M$$개의 구간 합 요청에 대해 각각 뺄셈 연산($$O(1)$$)만 수행한다.

이로 인해 전체 시간 복잡도는 $$O(N + M)$$으로, 질의가 많을수록 큰 성능 향상을 얻을 수 있다.

## 문제

[백준 11660번 구간 합 구하기 5](https://www.acmicpc.net/problem/11660)

## 풀이

2차원 누적합 배열을 구할 때,

현재까지 누적합 = 현재 값 + 왼쪽 누적 + 위쪽 누적 - 왼쪽 위 중복 영역

어떤 구간에 대한 합을 구할 때,

구간합 = 전체 - 위쪽 - 왼쪽 + 겹친 부분

```java
import java.util.*;

public class Main{
	public static void main(String[] args){
		Scanner sc = new Scanner(System.in);
		
		int n = sc.nextInt();
		int m = sc.nextInt();
		int[][] arr = new int[n+1][n+1];
		int[] answer = new int[m];
		
		for(int i=1; i<=n; i++) {
			for(int j=1; j<=n; j++) {
				arr[i][j] = sc.nextInt();
			}
		}
		
		int[][] S = new int[n+1][n+1];
		
		for(int i=1; i<=n; i++) {
			for(int j=1; j<=n; j++) {
				S[i][j] = arr[i][j] + S[i][j-1] + S[i-1][j] - S[i-1][j-1];
			}
		}
		
		
		for(int i=0; i<m; i++) {
			int x1 = sc.nextInt();
			int y1 = sc.nextInt();
			int x2 = sc.nextInt();
			int y2 = sc.nextInt();
			
			answer[i] = S[x2][y2] - S[x1-1][y2] - S[x2][y1-1] + S[x1-1][y1-1]; 
		}
		
		for(int i : answer) {
			System.out.println(i);
		}
	}
}
```
