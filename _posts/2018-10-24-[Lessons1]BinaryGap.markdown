---
layout: post
title:  "[Lessons1]BinaryGap"
date:   2018-10-24 22:00:00
author: EastGlow
categories: 코딜리티
---
## 문제

[https://app.codility.com/programmers/lessons/1-iterations/binary_gap/](https://app.codility.com/programmers/lessons/1-iterations/binary_gap/)

## 풀이
~~~
public class Solution {
    public int solution(int N) {
        // write your code in Java SE 8

        boolean flag = true;
        int biggerNum = 0;
        int tempNum = 0;
        String binaryStr = Integer.toBinaryString(N);
        char[] binaryArray = binaryStr.toCharArray();

        for(char ch : binaryArray){
            if('1' == ch){
                if(tempNum > 0){
                    if(tempNum > biggerNum){
                        biggerNum = tempNum;
                        flag = true;
                    }
                }        		
                tempNum = 0;
            }else{
        		    tempNum++;
            }
        }

        if(flag){
            return biggerNum;
        }else{
            return 0;
        }
    }
}
~~~
한 달 전쯤에 풀어봤었던 문제인데 당시엔 정답률이 40% 밖에 안 나왔었다. 그때 당시 계속 고민하고 코드 수정해서 제출해봤는데 40%를 넘지 못해서 그냥 넘어갔던 문제이다.

코딜리티 문제를 다시 풀어보기 위해 이 문제도 다시 접하게 되었는데 다시 코드를 짜서 제출하니 이번엔 2번 만에 100%를 맞았다.

로직은 간단하게 전달받은 숫자를 2진수로 변환 후, char 배열로 바꿔준다. char 배열로 루프문을 돌려서 "1"일 때와 "0"일 때를 먼저 체크해주고 "1"일 때는 tempNum이 0보다 클 때와 아닐 때를 나눠준다.

0보다 크다면 "1"을 만난 후에 "0"을 1번이라도 만났다는 얘기이기 때문에 만약 그 tempNum이 제일 큰 수를 담는 biggerNum보다 크다면 biggerNum에 tempNum을 담아준다.

"1"을 최소 2번은 만나야 flag 값이 true가 되므로 flag가 true일 때만 biggerNum을 리턴해주고 아니면 0을 리턴해준다.
