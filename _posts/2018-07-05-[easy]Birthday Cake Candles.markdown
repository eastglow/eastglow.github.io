---
layout: post
title:  "[easy]Birthday Cake Candles"
date:   2018-07-05 00:30:00
author: EastGlow
categories: 해커랭크
---
## 문제

당신의 생일날 생일 케이크가 하나 있다. 여기엔 초가 n개 꽂혀있는데 이 중에서 제일 키가 큰 초만 불어서 날려버릴 수 있다.

먼저 초 개수를 입력받고, 입력받은 개수만큼 초의 높이(정수)들을 입력받는다.

~~~
4
3 2 1 3
~~~

위 입력값은 4개의 초를 입력받으며 각각 높이가 3, 2, 1, 3 임을 의미한다. 여기서 날려버릴 수 있는 초의 개수를 출력하도록 한다.




## 풀이 1
~~~
import java.io.*;
import java.math.*;
import java.security.*;
import java.text.*;
import java.util.*;
import java.util.concurrent.*;
import java.util.regex.*;

public class Solution {

    // Complete the birthdayCakeCandles function below.
    static int birthdayCakeCandles(int[] ar) {
        int max = 0;
        int cnt = 1;
        
        max = ar[0];
        
        for(int j=1; j < ar.length; j++){            
            if(max < ar[j]){
                max = ar[j];
                cnt = 1;
            }else if(max == ar[j]){
                cnt++;
            }
        }
        
        return cnt;
    }

    private static final Scanner scanner = new Scanner(System.in);

    public static void main(String[] args) throws IOException {
        BufferedWriter bufferedWriter = new BufferedWriter(new FileWriter(System.getenv("OUTPUT_PATH")));

        int arCount = scanner.nextInt();
        scanner.skip("(\r\n|[\n\r\u2028\u2029\u0085])?");

        int[] ar = new int[arCount];

        String[] arItems = scanner.nextLine().split(" ");
        scanner.skip("(\r\n|[\n\r\u2028\u2029\u0085])?");

        for (int i = 0; i < arCount; i++) {
            int arItem = Integer.parseInt(arItems[i]);
            ar[i] = arItem;
        }

        int result = birthdayCakeCandles(ar);

        bufferedWriter.write(String.valueOf(result));
        bufferedWriter.newLine();

        bufferedWriter.close();

        scanner.close();
    }
}
~~~
예시로 든 3, 2, 1, 3에서 제일 높은 높이는 3이다. 3이 2개이므로 날려버릴 수 있는 초의 개수는 2가 나와야 한다. 처음 문제를 읽을 때 잘못 이해해서 한참 삽질을 하다가 다시 문제를 읽어보니 잘못 이해했던 걸 깨달았다. -_-

여튼, 쉬운 문제였지만 삽질한 기념으로 간단히 풀이해보자면 max 값에 배열의 첫 값을 저장해두고 그 다음 배열의 값부터 max와 비교해간다. 비교해서 배열의 값이 더 크면 max 값에 새로 담고, 날려버린 초의 개수(cnt)의 값을 1로 초기화해준다. 왜냐면 더 큰 수를 발견했다면 초의 개수는 1개부터 시작하기 때문. 그리고 max 값과 같은 값이 나온다면 초의 개수를 1개 늘려주면 된다. 문제를 푸는 능력도 중요하지만 문제를 이해하는 능력이 중요한 걸 다시 깨달은 문제이다...