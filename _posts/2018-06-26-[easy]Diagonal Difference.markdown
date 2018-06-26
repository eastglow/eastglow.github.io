---
layout: post
title:  "[easy]Diagonal Difference"
date:   2018-06-26 23:30:00
author: EastGlow
categories: HackerRank
---
## 문제

정사각형 행렬이 주어진다. 여기서 두 대각선(좌->우, 우->좌)의 합을 구하고 그 합 사이의 절대값을 구한다.

~~~
1 2 3
4 5 6
9 8 9
~~~

위 행렬을 예로 들면 1+5+9 = 15, 3+5+9 = 17 이며 두 합 사이의 절대값은 |15-17| = 2 이다.




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

    // Complete the diagonalDifference function below.
    static int diagonalDifference(int[][] arr) {
        int a = 0;
        int b = 0;
        
        for(int i=0; i<arr.length; ++i){     
            for(int j=0; j<arr[0].length; ++j){
                if(i==j) a += arr[i][j];
                if(i+j == arr.length-1) b += arr[i][j];
            }
        }
        
        return Math.abs(a-b);
    }

    private static final Scanner scanner = new Scanner(System.in);

    public static void main(String[] args) throws IOException {
        BufferedWriter bufferedWriter = new BufferedWriter(new FileWriter(System.getenv("OUTPUT_PATH")));

        int n = scanner.nextInt();
        scanner.skip("(\r\n|[\n\r\u2028\u2029\u0085])?");

        int[][] arr = new int[n][n];

        for (int i = 0; i < n; i++) {
            String[] arrRowItems = scanner.nextLine().split(" ");
            scanner.skip("(\r\n|[\n\r\u2028\u2029\u0085])?");

            for (int j = 0; j < n; j++) {
                int arrItem = Integer.parseInt(arrRowItems[j]);
                arr[i][j] = arrItem;
            }
        }

        int result = diagonalDifference(arr);

        bufferedWriter.write(String.valueOf(result));
        bufferedWriter.newLine();

        bufferedWriter.close();

        scanner.close();
    }
}
~~~
예시로 든 정사각형 행렬을 풀어보면 좌->우 대각선은 (0,0) (1,1) (2,2),  우->좌 대각선은 (0,2) (1,1) (2,0) 의 숫자들을 지칭하고 있다.



좌->우 대각선은 x, y 좌표가 서로 같을 때, 우->좌 대각선은 x,y 좌표가 행렬의 크기보다 -1 작을 때 라는 규칙을 띄고 있다. 다른 규칙이 있거나 다른 풀이 방법이 있을 수 있지만 나는 이러한 규칙을 발견하여 풀었다. easy 단계라 딱히 어려운 문제는 아니었던 것 같다.