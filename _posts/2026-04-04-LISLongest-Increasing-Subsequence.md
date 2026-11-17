---
title: "LIS(Longest Increasing Subsequence)"
date: 2026-04-04 11:14:05 +0900
series: "알고리즘"
tags: ["알고리즘"]
math: true
summary: "최장 증가 부분 수열 알고리즘은 어떤 수열이 주어졌을 때, 그 수열의 원소들을 순서를 유지하면서 뽑아내어 만든 수열 중 크기가 엄격하게 증가하는 가장 긴 수열을 찾는 문제이다. 백준 11053번 가장 긴 증가하는 부분 수열 백준 12015번 가장 긴 증가하는 부분 수열 2 각 원소를 마지막으로 하는"
---

## 최장 증가 부분 수열

최장 증가 부분 수열 알고리즘은 어떤 수열이 주어졌을 때, 그 수열의 원소들을 순서를 유지하면서 뽑아내어 만든 수열 중 **크기가 엄격하게 증가하는 가장 긴 수열**을 찾는 문제이다.

## 문제
1. [백준 11053번 가장 긴 증가하는 부분 수열](https://www.acmicpc.net/problem/11053)
2. [백준 12015번 가장 긴 증가하는 부분 수열 2](https://www.acmicpc.net/problem/12015)

## 풀이

### 1. 동적 계획법(DP)

각 원소를 마지막으로 하는 LIS의 길이를 저장하며 진행한다.

시간 복잡도: $$O(N^2)$$ (이중 루프 사용)

```java
import java.util.*;

public class Main{
	public static void main(String[] args) {
		int answer = Integer.MIN_VALUE;
		
		Scanner sc = new Scanner(System.in);
		
		int n = sc.nextInt();
		int[] arr = new int [n];
		
		for(int i=0; i<n; i++) {
			arr[i] = sc.nextInt();
		}
		
		int[] dp = new int[n];
		
		for(int i=0; i<n; i++) {
			dp[i] = 1; //기본 길이 1
			for(int j=0; j<i; j++) {
				if(arr[i] > arr[j]) dp[i] = Math.max(dp[j]+1, dp[i]);
			}
			
			answer = Math.max(answer, dp[i]);
		}
		
		System.out.println(answer);
	}
}
```

### 2. 이분 탐색(Binary Search)

별도의 배열을 유지하며, 현재 원소가 들어갈 적절한 위치를 이분 탐색으로 찾는다.

현재 원소가 배열의 마지막 값보다 크면 맨 뒤에 추가하고,

그렇지 않다면 현재 원소보다 크거나 같은 첫 번째 위치를 찾아 그 값을 현재 원소로 대체한다.

(길이는 항상 정확하지만 실제 LIS 수열의 구성 요소와는 다를 수 있다.)

시간 복잡도: $$O(NlogN)$$

```java
import java.util.*;

public class Main{
	static int[] temp;
	static int len = 0; //발견한 길이

	public static void binary(int num) {
		int lt=0, rt = len-1;
		while(lt <= rt) {
			int mid = (lt + rt) / 2;
			if(temp[mid] < num) {
				lt = mid + 1;
			}
			else rt = mid - 1;
		}
		temp[lt] = num;
		
		//삽입된 자리가 현재 길이와 같다면 새로운 최대값이 들어온 것 --> 길이 증가
		if(lt == len) len++;
	}
	
	public static void main(String[] args) {
		Scanner sc = new Scanner(System.in);
		int n = sc.nextInt();
		int[] arr = new int[n];
		for(int i=0; i<n; i++) {
			arr[i] = sc.nextInt();
		}
		
		temp = new int[n];
		
		for(int i=0; i<n; i++) {
			binary(arr[i]);
		}
		
		System.out.println(len);
	}
}
```
