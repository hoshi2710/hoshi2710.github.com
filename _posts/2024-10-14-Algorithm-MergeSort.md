---
layout: post
title: "Algorithm - [Sort] Merge Sort(병합 정렬)"
date: 2024-10-14
author: Hoshi
cover: "/assets/img/posts_img/MergeSort.png"
tags: Algorithm Sort
categories: "Algorithm"
---

# 병합 정렬?

정렬하고자 하는 데이터들을 다 쪼갠다음 쪼갠 요소들끼리 서로 비교하고 병합하고 병합한 데이터 그룹끼리 또 비교해서 새로운 그룹을 만들고 이를 반복하여 완전한 데이터 배열로 돌아올때까지 반복하는 정렬 방법이다.

# 정렬 과정

## 정렬 시각화(예시)

![MergeSortUML.png]({{site.url}}/assets/img/posts_img/MergeSortUML.png)

## 과정 풀이

![MergeSort_Step1.png]({{site.url}}/assets/img/posts_img/MergeSort_Step1.png)

![MergeSort_Step2.png]({{site.url}}/assets/img/posts_img/MergeSort_Step2.png)

![MergeSort_Step3.png]({{site.url}}/assets/img/posts_img/MergeSort_Step3.png)

1. 각 요소들을 모두 쪼갠다음 두개씩 그룹을 지어 비교한다.
   위 예시처럼 오름차순으로 정렬을 한다고 할때 묶은 그룹 안에서 작은 수를 먼저 좌측에 두고, 그다음 이어서 나머지 데이터를 넣어
   새로운 그룹으로 병합시켜준다.
   (그룹이 지어지지 못한 데이터는 그대로 내려서 병합 시켜 준다.)
2. 위 병합작업이 완료후 다시 두개의 그룹을 비교한다.
   이번에는 양 그룹을 비교할때 좌측에 있는 부분을 우선적으로 비교하여 병합을 진행한다. 현재 그림에서는 42, 14가 우선적으로 비교대상이 된다.
   14가 더 작음으로 먼저 배치가 된다.
   그 다음 42가 배치가 되는 것이 아닌 다시 14 다음에 있는 89와 42를 비교하는 작업이 바로 이어서 진행되게 된다.
3. 이 과정을 모든 데이터 그룹이 병합되어 온전한 배열을 이루기까지 반복한다.

# 병합 정렬의 특징

## Stable sort(안정 정렬)

안정정렬이란 정렬을 진행하는 도중 중복되는 값이 발생했을때 그 순서가 유지가 되는가의 유무이다.
병합 정렬은 중복되는 값이 발생해도 원래 배열의 인덱스의 상하가 변하지 않으며 이에 따라 복수 기준을 활용하여 정렬 하거나 이미 정렬된 배열을 다시 정렬을 해도 원래 정렬한 내용이 유지가 된다.

## 시간 복잡도의 상대적인 우위

병합정렬은 그룹이 병합되고 정렬이 이루어지는 과정이 반복되기때문에 과정마다 그룹의 갯수가 1/2배씩 줄어들게 되면서 $$O(nlog(n))$$의 시간복잡도를 가지며 비슷한 시간복잡도를 가진 다른 정렬방법과 달리 최악, 최고의 케이스의 시간복잡도가 거의 $$O(nlog(n))$$ 와 크게 차이가 없어 일관된 속도의 정렬이 가능하다.

## 메모리 사용량이 큼

새로운 그룹을 지으면서 새로운 배열을 생성하고 이를 병합하고 하는 과정에서 타 정렬 방식에 비해 더 많은 메모리가 사용되므로 알고리즘 문제의 조건에서 메모리 조건이 조금 가혹하게 설정되어있다면 사용하기 어려울 수 있다.

# 예시 코드

```java
import java.util.Arrays;

public class Main
{
    // 정렬하고자 하는 배열
    static int[] arr= {90,42,89,14,16,95,93,72};
    static void mergeSortTask(int start, int end, int unit) {
        // 만들어질 수 있는 그룹의 갯수
        int groupEa = (int) ((end-start+1)/unit) + ((end-start+1)%unit != 0 ? 1:0);
        // 병합된 데이터 배열을 담을 배열
        int[] tempArr = new int[8];
        for(int i=0; i<(groupEa-(groupEa%2)); i+=2) { // 짝이 지어지는 그룹끼리 먼저 정렬
            // 배열 내에서 각 그룹의 인덱스 범위 {startIdx, endIdx}
            // 두번째 그룹에 경우 데이터 갯수 부족으로 첫번째 그룹과 데이터 갯수가 안맞을 수 있으므로 별도의 예외처리 진행
            int[] groupA = {i*unit,(i+1)*unit-1}, groupB = {(i+1)*unit, Math.min((i + 2) * unit - 1, end)};
            // 각 그룹의 길이
            int lengthA = groupA[1]-groupA[0]+1, lengthB = groupB[1] - groupB[0]+1;
            int cursor = groupA[0], cursorA=groupA[0],cursorB=groupB[0];
            for(int j=0; j<lengthA+lengthB; j+=1) {
                // 비교, 병합 작업중 첫번째 그룹에서 더이상 비교할 데이터가 없다면
                // 두번째 그룹의 나머지 데이터들을 차례대로 새로운 그룹에 배열하고 작업 종료
                if(cursorA > groupA[1]) {
                    tempArr[cursor] = arr[cursorB];
                    cursorB += 1;
                    cursor+=1;
                    continue;
                }
                // 두번째 그룹에서 더이상 비교할 데이터가 없다면
                // 첫번째 그룹에 있는 나머지 데이터들을 차례대로 새로운 그룹에 배열하고 작업 종료
                if(cursorB > groupB[1]) {
                    tempArr[cursor] = arr[cursorA];
                    cursorA += 1;
                    cursor+=1;
                    continue;
                }
                // 두 그룹의 값을 서로 비교 하여 새로운 그룹으로 병합
                if(arr[cursorA] > arr[cursorB]) {
                    tempArr[cursor] = arr[cursorB];
                    cursorB +=1;
                }else {
                    tempArr[cursor] = arr[cursorA];
                    cursorA +=1;
                }
                cursor+=1;
            }
        }
        // 그룹이 지어지지 못한 데이터들이 있다면
        // 차례대로 새 배열에 배치
        if(groupEa%2 == 1)
            for (int i=unit*(groupEa-1); i<=end; i+=1) {
            tempArr[i] = arr[i];
            }
        //병합된 새 배열로 기존 배열 업데이트
        arr = tempArr;
        // 아직 병합이 완전히 이루어지지 않았다면
        // 이어서 재귀 작업 진행
        if(unit*2 < end-start+1) mergeSortTask(start,end,unit*2);
    }
    public static void main(String[] args) {
        System.out.println("정렬 전 : "+ Arrays.toString(arr)); // 정렬 전 배열 출력
        mergeSortTask(0,7,1); // 정렬 작업 진행
        System.out.println("정렬 후 : "+ Arrays.toString(arr)); // 정령 후 배열 출력
    }
}
```

![MergeSort_Example.png]({{site.url}}/assets/img/posts_img/MergeSort_Example.png)
