---
title: "희소 테이블(Sparse Table)"
date: 2022-03-16 04:44:00 +0900
categories: ['알고리즘 & 자료구조']
tags: 희소테이블 자료구조
---
## 설명
희소 테이블이란 정적 데이터에서 구간 쿼리를 빠르게 계산할수 있는 자료 구조다. 예를 들어 `Array[0] - Array[N]`과 함수 $F$가 존재할 때, $0 \leq i \leq j \leq N$을 만족하는 $F(i, j)$를 빠르게 구할 수 있다.  
이를 사용하면 희소 테이블을 구성하는 데에는 $O(N \log N)$, 쿼리 당 $O(\log N)$의 처리가 가능해 문제를 $O(N \log N)$의 시간에 해결할 수 있다.

희소 테이블을 사용할 수 있는 조건은 다음과 같다.
* 결합 법칙이 성립: $F(a, b, c) = F(F(a, b), c) = F(a, F(b, c))$
* 데이터가 변하지 않음

이 두 조건을 만족하는 데이터와 함수, 그리고 쿼리 수가 많은 문제가 주어진다면 희소 테이블을 사용하는 것이 좋다.

최댓값, 최솟값 같은 특정 쿼리의 경우에 $O(1)$에 쿼리 처리가 가능하다.

## 예제
### 구간 합
#### 전처리
$F(a, b) =\sum_{i = a}^{b} Array[i]$

| i | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|--|--|--|--|--|--|--|--|--|
| Array[i] | 1 | 3 | 5 | 2 | 8 | 5 | 3 | 10 |

`table[k][i]` = $F(i,i+2^{k}-1)$
#### `table[0][i]`
`table[0][i]` = $F(i, i)$이므로 전부 `Array[i]`를 대입한다.
#### 나머지
$F(i, i + 2^{k} - 1) = F(i, i + 2^{k - 1} - 1) + F(i + 2^{k - 1}, i + 2^{k - 1} + 2^{k - 1} - 1)$  
`table[k][i] = table[k - 1][i] + table[k - 1][i + (1 << (k - 1))]`

한 행이 모두 있다면 다음 행은 밑 행을 기준으로 bottom-up하게 채울 수 있다.

| i | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| table[0][i] | 1 | 3 | 5 | 2 | 8 | 5 | 3 | 10 |
| table[1][i] | 4 | 8 | 7 | 10 | 13 | 18 | 13 |
| table[2][i] | 11 | 18 | 20 | 28 | 26 |
| table[3][i] | 37 |

#### 쿼리
$F(a,b$를 구할 때, 범위 $[a, b]$의 길이는 $b - a + 1$이고, 이는 2의 제곱으로 나타낼 수 있다. 2의 제곱수만큼의 길이로 분리된 $F(a, b)$를 계산하면 $O(\log N)$의 시간에 쿼리를 구할 수 있다.

$diff = a - b + 1 = 2^{x} + 2^{y} + 2^{z}$  
$F(a, b) = \\ F(a, a + 2^{x} - 1) + F(a + 2^{x}, a + 2^{x} + 2^{y} - 1) + F(a + 2^{x} + 2^{y}, a + 2^{x} + 2^{y} + 2^{z} - 1) + F(a + 2^{x} + 2^{y} + 2^{z}, b)$

```c++
int partialSum(vector<vector<int>> &table, int a, int b) {
    int diff = b - a + 1;
    int sum = 0;
    for (int i = table.size() - 1, j = a; i >= 0; i--) {
        if (diff & (1 << i)) {
            sum += table[i][j];
            j += (1 << i);
        }
    }
    return sum;
}
```
#### 문제
[구간 합 구하기 4](https://www.acmicpc.net/problem/11659)

위 문제를 희소 테이블로 해결 가능하다.
```c++
#include <iostream>
#include <vector>
#include <cmath>

using namespace std;

int partialMin(vector<vector<int>> &table, int a, int b) {
    int diff = b - a + 1;
    int sum = 0;
    for (int i = table.size() - 1, j = a; i >= 0; i--) {
        if (diff & (1 << i)) {
            sum += table[i][j];
            j += (1 << i);
        }
    }
    return sum;
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);
    cout.tie(NULL);
    int N, M;
    cin >> N >> M;
    int maxLog = static_cast<int>(log2(N));
    vector<int> arr(N);
    vector<vector<int>> table(maxLog + 1, vector<int>(N));
    for (int i = 0; i < N; i++) {
        cin >> arr[i];
        table[0][i] = arr[i];
    }
    for (int k = 1; k <= maxLog; k++) {
        for (int i = 0; i + (1 << (k - 1)) < N; i++) {
            table[k][i] = table[k - 1][i] + table[k - 1][i + (1 << (k - 1))];
        }
    }
    while (M-- > 0) {
        int i, j;
        cin >> i >> j;
        cout << partialMin(table, i - 1, j - 1) << '\n';
    }
    return 0;
}
```
그런데 사실 위 문제의 경우 누적합으로 구하는 것이 더 빠르고 편리하다.
### 구간 최솟값
희소 테이블은 쿼리가 $O(\log N)$일 때도 좋지만 $O(1)$에 처리가 될 때 굉장히 유용하다. 구간 최솟값은 그러한 문제 중 하나다.
#### 전처리
$F(a, b) = Min(a, b)$

| i | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|--|--|--|--|--|--|--|--|--|
| Array[i] | 1 | 3 | 5 | 2 | 8 | 5 | 3 | 10 |

$F(i, i + 2^{k} - 1) = Min(F(i, i + 2^{k - 1} - 1), F(i + 2^{k - 1}, i + 2^{k - 1} + 2^{k - 1} - 1))$

`table[k][i] = min(table[k - 1][i], table[k - 1][i + (1 << (k - 1))])`

구간 합과 비슷한 방법으로 테이블을 채울 수 있다.

| i | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| table[0][i] | 1 | 3 | 5 | 2 | 8 | 5 | 3 | 10 |
| table[1][i] | 1 | 3 | 2 | 2 | 5 | 3 | 3 |
| table[2][i] | 1 | 2 | 2 | 2 | 3 |
| table[3][i] | 1 |

#### 쿼리
구간 합을 구할 때는 전체 구간을 2의 제곱수의 크기로 나누어 하나 하나 더해주었지만, 최솟값의 경우 그렇게 할 필요가 없다. 일단 구간의 경계 a, b가 주어지면,$a + 2^{x} \leq b$를 만족하는 $x$의 최솟값을 찾는다.$x = \log (b - a + 1)$로 구할 수 있다.

<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1" width="641px" viewBox="-0.5 -0.5 641 411" style="max-width:100%;max-height:411px;"><defs></defs><g><rect x="100" y="70" width="60" height="60" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 101px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><font style="font-size: 23px">3</font></div></div></div></foreignObject><text x="130" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">3</text></switch></g><rect x="160" y="70" width="60" height="60" fill="#d5e8d4" stroke="#82b366" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 161px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><font style="font-size: 23px">4</font></div></div></div></foreignObject><text x="190" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">4</text></switch></g><rect x="220" y="70" width="60" height="60" fill="#d5e8d4" stroke="#82b366" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 221px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><font style="font-size: 23px">1</font></div></div></div></foreignObject><text x="250" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">1</text></switch></g><rect x="280" y="70" width="60" height="60" fill="#d5e8d4" stroke="#82b366" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 281px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><font style="font-size: 23px">2</font></div></div></div></foreignObject><text x="310" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">2</text></switch></g><rect x="400" y="70" width="60" height="60" fill="#d5e8d4" stroke="#82b366" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 401px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><font style="font-size: 23px">3</font></div></div></div></foreignObject><text x="430" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">3</text></switch></g><rect x="460" y="70" width="60" height="60" fill="#d5e8d4" stroke="#82b366" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 461px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><font style="font-size: 23px">4</font></div></div></div></foreignObject><text x="490" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">4</text></switch></g><rect x="520" y="70" width="60" height="60" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 521px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><font style="font-size: 23px">8</font></div></div></div></foreignObject><text x="550" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">8</text></switch></g><rect x="580" y="70" width="60" height="60" fill="rgb(255, 255, 255)" stroke="rgb(0, 0, 0)" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 581px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 12px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;"><span style="font-size: 23px">10</span></div></div></div></foreignObject><text x="610" y="104" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="12px" text-anchor="middle">10</text></switch></g><rect x="0" y="85" width="60" height="30" fill="none" stroke="none" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 100px; margin-left: 1px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 23px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">Array</div></div></div></foreignObject><text x="30" y="107" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="23px" text-anchor="middle">Array</text></switch></g><path d="M 340 100 L 400 100" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" stroke-dasharray="3 3" pointer-events="stroke"></path><path d="M 190 188 L 380 188 M 380 192 L 190 192 M 380 192" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"></path><rect x="160" y="140" width="60" height="30" fill="none" stroke="none" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 155px; margin-left: 161px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 23px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">a</div></div></div></foreignObject><text x="190" y="162" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="23px" text-anchor="middle">a</text></switch></g><rect x="460" y="140" width="60" height="30" fill="none" stroke="none" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 58px; height: 1px; padding-top: 155px; margin-left: 461px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 23px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">b</div></div></div></foreignObject><text x="490" y="162" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="23px" text-anchor="middle">b</text></switch></g><path d="M 300 268 L 490 268 M 490 272 L 300 272 M 490 272" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"></path><rect x="150" y="200" width="280" height="30" fill="none" stroke="none" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 278px; height: 1px; padding-top: 215px; margin-left: 151px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 23px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">min(a, a + 2<sup>x</sup>-1)</div></div></div></foreignObject><text x="290" y="222" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="23px" text-anchor="middle">min(a, a + 2x-1)</text></switch></g><rect x="240" y="280" width="356" height="30" fill="none" stroke="none" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 354px; height: 1px; padding-top: 295px; margin-left: 241px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 23px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">min(b - (2<sup>x</sup>&nbsp;- 1) , b)</div></div></div></foreignObject><text x="418" y="302" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="23px" text-anchor="middle">min(b - (2x&nbsp;- 1) , b)</text></switch></g><rect x="200" y="0" width="330" height="30" fill="none" stroke="none" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 328px; height: 1px; padding-top: 15px; margin-left: 201px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 23px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">x = log(b - a + 1)</div></div></div></foreignObject><text x="365" y="22" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="23px" text-anchor="middle">x = log(b - a + 1)</text></switch></g><path d="M 190 358 L 490 358 M 490 362 L 190 362 M 490 362" fill="none" stroke="rgb(0, 0, 0)" stroke-miterlimit="10" pointer-events="all"></path><rect x="34" y="380" width="606" height="30" fill="none" stroke="none" pointer-events="all"></rect><g transform="translate(-0.5 -0.5)"><switch><foreignObject pointer-events="none" width="100%" height="100%" requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility" style="overflow: visible; text-align: left;"><div xmlns="http://www.w3.org/1999/xhtml" style="display: flex; align-items: unsafe center; justify-content: unsafe center; width: 604px; height: 1px; padding-top: 395px; margin-left: 35px;"><div data-drawio-colors="color: rgb(0, 0, 0); " style="box-sizing: border-box; font-size: 0px; text-align: center;"><div style="display: inline-block; font-size: 23px; font-family: Helvetica; color: rgb(0, 0, 0); line-height: 1.2; pointer-events: all; white-space: normal; overflow-wrap: normal;">min(min(a, a + 2<sup>x</sup>-1),&nbsp;min(b - (2<sup>x</sup>&nbsp;- 1) , b))</div></div></div></foreignObject><text x="337" y="402" fill="rgb(0, 0, 0)" font-family="Helvetica" font-size="23px" text-anchor="middle">min(min(a, a + 2x-1),&nbsp;min(b - (2x&nbsp;- 1) , b))</text></switch></g></g><switch><g requiredFeatures="http://www.w3.org/TR/SVG11/feature#Extensibility"></g><a transform="translate(0,-5)" xlink:href="https://www.diagrams.net/doc/faq/svg-export-text-problems" target="_blank" rel="noopener"><text text-anchor="middle" font-size="10px" x="50%" y="100%">Text is not SVG - cannot display</text></a></switch></svg>

구간을 여러 개가 아닌 크기 $2^{x}$인 구간 두 개만 있어도 a와 b 사이의 원소는 전부 커버할 수 있다. 따라서 $O(1)$ 안에 구간 최솟값을 구할 수 있다.

#### 문제
[최솟값과 최댓값](https://www.acmicpc.net/problem/2357)

위 문제는 구간 최솟값과 최댓값을 구하는 문제인데, 최댓값도 최솟값과 마찬가지로 희소 테이블을 사용해 $O(1)$ 시간에 구할 수 있다.

```c++
#include <iostream>
#include <vector>
#include <cmath>
#include <algorithm>

using namespace std;

int partialMin(vector<vector<int>> &minTable, int a, int b) {
    int x = log2(b - a + 1);
    return min(minTable[x][a], minTable[x][b - (1 << x) + 1]);
}

int partialMax(vector<vector<int>> &maxTable, int a, int b) {
    int x = log2(b - a + 1);
    return max(maxTable[x][a], maxTable[x][b - (1 << x) + 1]);
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);
    cout.tie(NULL);
    int N, M;
    cin >> N >> M;
    int maxLog = static_cast<int>(log2(N));
    vector<int> arr(N);
    vector<vector<int>> minTable(maxLog + 1, vector<int>(N));
    vector<vector<int>> maxTable(maxLog + 1, vector<int>(N));
    for (int i = 0; i < N; i++) {
        cin >> arr[i];
        minTable[0][i] = arr[i];
        maxTable[0][i] = arr[i];
    }
    for (int k = 1; k <= maxLog; k++) {
        for (int i = 0; i + (1 << (k - 1)) < N; i++) {
            minTable[k][i] = min(minTable[k - 1][i], minTable[k - 1][i + (1 << (k - 1))]);
            maxTable[k][i] = max(maxTable[k - 1][i], maxTable[k - 1][i + (1 << (k - 1))]);
        }
    }
    while (M-- > 0) {
        int i, j;
        cin >> i >> j;
        cout << partialMin(minTable, i - 1, j - 1) << ' ' << partialMax(maxTable, i - 1, j - 1) << '\n';
    }
    return 0;
}
```