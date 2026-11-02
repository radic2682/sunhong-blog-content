---
id: "01HHG7J800QD6X2S7X3E1AB9PC"
slug: "wallace-tree-multiplier"
title: "Wallace Tree 곱셈기 Verilog 설계: 반가산기·전가산기로 8비트 병렬 곱셈"
category: "01M651PC00H3J4W7JXTZ32PR45"
tags: ["verilog","vivado"]
summary: "반가산기와 전가산기로 부분합을 단계별로 줄이는 Wallace Tree 구조로 8비트 병렬 곱셈기를 Verilog 설계했다. 16비트 곱을 내는 계산 과정, 설계 흐름, 테스트벤치와 전체 코드를 정리했다."
thumbnail: "/img/01HHG7J800QD6X2S7X3E1AB9PC/f7a0960134a5.webp"
note: false
created: "2023-12-13T09:00:00+09:00"
updated: "2026-09-15T02:10:29+09:00"
---
## 1. Topic

Wallace Tree Multiplier는 두 개의 정수를 곱하는 디지털 하드웨어 회로를 효율적으로 구현할 수 있는 구조로 1964년 오스트렐리아의 Chris Wallace에 의해 발명되었다. 설계한 8bit 곱셈기는 8bit의 승수와 피승수를 입력하면 <u>16bit의 곱셈 결과값</u>을 받을 수 있다.

설계는 반가산기와 전가산기를 이용해 각 스테이지별로 부분합을 하였다. 반가산기는 2개의 입력을 받아 Sum과 Carry를 출력으로 반환한다. 이와 달리 전가산기는 3개의 입력을 받아 Sum과 Carry를 반환한다. 이러한 반가산기와 전가산기를 이용하면 효율적으로 곱셈기를 만들 수 있다.

![Topic|medium](/img/01HHG7J800QD6X2S7X3E1AB9PC/f7a0960134a5.webp)

## 2. Design Flow

아래는 8bit wallace multiplier의 계산 과정을 보여준다. 2개씩 묶인 것은 입력이 2개인 반가산기를 이용하고 3개씩 묶인 것은 입력이 3개인 전가산기를 이용한다.

![Design Flow|medium](/img/01HHG7J800QD6X2S7X3E1AB9PC/a2a021ee0741.webp)

8bit wallace multiplier에서 stage0의 각 점들은 입력 A, B의 각 bit를 bit 자리수 별 &연산한 결과와 같다. 이는 코드에서 다음과 같이 구현하였다. p는 8bit를 가진 8개의 배열이다. ‘[ ]’안에 있는 숫자는 stage0의 줄 수를 나타낸다고 보면 알기 쉽다. 예를 들어 p[0]에는 A와 8{B[0]}의 각자리 별로 &연산한 결과가 들어가게 된다. 8{B[0]}는 B의 0번째 비트를 8번 반복하는 연산이다. 이는 stage0에서는 B의 0번째 자리만 A와의 연산에 참여하기 때문이다. 만약 n번째 staged에서는 B의 n번째 자리만 A와 연산할 것이다. 그림3은 이렇게 연산하여 p에 참여하는 a의 자리수, b의 자리수를 보여준다.

```verilog
//p multiplied by A and B (and gate)
wire [7:0] p[7:0];
assign p[0] = A & {8{B[0]}};
assign p[1] = A & {8{B[1]}};
assign p[2] = A & {8{B[2]}};
assign p[3] = A & {8{B[3]}};
assign p[4] = A & {8{B[4]}};
assign p[5] = A & {8{B[5]}};
assign p[6] = A & {8{B[6]}};
assign p[7] = A & {8{B[7]}};
```

![Design Flow (2)|medium](/img/01HHG7J800QD6X2S7X3E1AB9PC/0fb66c0765bf.webp)

이렇게 만들어진 p의 값은 반가산기와 전가산기에 의해 계산이 된다. 계산 과정은 아래와 같다. 이름은 full adder일 경우 ‘f’, half adder일 경우 ‘h’로 나타냈다. 그 뒤에 오는 ‘s’는 sum, ‘c’는 carry의 약자이며 가장 앞자리 수는 stage수+1로 나타내었다. 또한, 뒤에 오는 수는 adder의 고유 번호를 나타낸다.

![Design Flow (3)|medium](/img/01HHG7J800QD6X2S7X3E1AB9PC/f86bf56ecc08.webp){: width="80%" height="80%"}

![Design Flow (4)|medium](/img/01HHG7J800QD6X2S7X3E1AB9PC/849710127683.webp)
{: width="80%" height="80%"}

테스트벤치는 총 2가지 방법으로 구현해 보았다. 두 가지 방식 모두 a와 b를 입력으로 받아 일반 곱셈 연산자로 계산된 값과 wallace tree로 계산된 값을 비교해 같은지를 확인한다.

tb1은 총 6번의 입력을 줬을 때, 계산된 값이 모두 같았음을 볼 수 있었다. 예를 들어 a에 224, b에 227의 입력을 넣었을 때, 일반 곱셈 연산자로 계산된 값과 wallace tree로 계산된 값이 50848로 같았다.

tb2는 모든 입력을 다 넣어보는 테스트벤치이다. 이중 for문을 사용하여 a와 b에 모든 값을 넣어 주었다. 만약 값이 다르다면 값이 다른 만큼 error가 발생한다. 시뮬레이션 결과 error가 0이므로 설계가 잘 되었음을 알 수 있었다.

이번 설계를 통해 verilog 문법을 다시 한번 익히고 적용하며 부족한 점을 알 수 있었다. 또한, module을 정의하고 이해하는 데 도움이 되었으며 설계를 성공적으로 마침으로써 성취감을 얻었다. 한가지 아쉬운 점은 마지막 ripple carry adder부분을 module화 하고 싶었지만, 많은 입력을 처리하지 못하고 module화 하지 못했다는 점이다. module에 들어가는 입력을 결합 연산자로 묶어서 넣어 주었으나 실패하였다. 이는 더욱 공부해야 할 부분으로 보인다.

## 3. Test Bench

![Test Bench|medium](/img/01HHG7J800QD6X2S7X3E1AB9PC/d3ae6fde16ff.webp)


## 4. Code

```verilog
module HA(sum, carry, a, b);

  output sum, carry;

    input a, b;

  

    xor (sum, a, b);

    and (carry, a, b);

endmodule

  

module FA(sum, carry, a, b, c);

    output sum, carry;

    input a, b, c;

  

    xor (sum, a, b, c);

    and (t1, a, b);

    xor (t2, a, b);

    and (t3, t2, c);

    or (carry, t1, t3);

endmodule

module multiplier(A,B,out);

    //inputs and outputs

    input [7:0] A,B;

    output [15:0] out;

    //connecting wire

    wire sh11, sh12, sh13, sh14, sh21, sh22, sh23, sh31, sh32, sh33, sh34, sh41, sh42, sh43, sh44;

    wire sf11, sf12, sf13, sf14, sf15, sf16, sf17, sf18, sf19, sf110, sf111, sf112;

    wire sf21, sf22, sf23, sf24, sf25, sf26, sf27, sf28, sf29, sf210, sf211, sf212, sf213;

    wire sf31, sf32, sf33, sf34, sf35, sf36;

    wire sf41, sf42, sf43, sf44, sf45, sf46, sf47;

    wire ch11, ch12, ch13, ch14, ch21, ch22, ch23, ch31, ch32, ch33, ch34, ch41, ch42, ch43, ch44;

    wire cf11, cf12, cf13, cf14, cf15, cf16, cf17, cf18, cf19, cf110, cf111, cf112;

    wire cf21, cf22, cf23, cf24, cf25, cf26, cf27, cf28, cf29, cf210, cf211, cf212, cf213;

     wire cf31, cf32, cf33, cf34, cf35, cf36;

     wire cf41, cf42, cf43, cf44, cf45, cf46, cf47;

   wire c0, c1, c2, c3, c4, c5, c6, c7, c8, c9, c10;

  

   //p multiplied by A and B (and gate)

    wire [7:0] p[7:0];

   assign  p[0] = A & {8{B[0]}};  

   assign  p[1] = A & {8{B[1]}};  

   assign  p[2] = A & {8{B[2]}};  

   assign  p[3] = A & {8{B[3]}};  

   assign  p[4] = A & {8{B[4]}};  

   assign  p[5] = A & {8{B[5]}};  

   assign  p[6] = A & {8{B[6]}};  

   assign  p[7] = A & {8{B[7]}};  

  

    // stage 0 _ first line

   assign out[0] = p[0][0];

   HA h11 (out[1], ch11, p[1][0], p[0][1]);

   FA f11 (sf11, cf11, p[2][0], p[1][1], p[0][2]);

     FA f12 (sf12, cf12, p[2][1], p[1][2], p[0][3]);

     FA f13 (sf13, cf13, p[2][2], p[1][3], p[0][4]);

     FA f14 (sf14, cf14, p[2][3], p[1][4], p[0][5]);

     FA f15 (sf15, cf15, p[2][4], p[1][5], p[0][6]);

     FA f16 (sf16, cf16, p[2][5], p[1][6], p[0][7]);

     HA h12 (sh12, ch12, p[2][6], p[1][7]);

    // stage 0 _ second line

     HA h13 (sh13, ch13, p[4][0], p[3][1]);

    FA f17 (sf17, cf17, p[5][0], p[4][1], p[3][2]);

    FA f18 (sf18, cf18, p[5][1], p[4][2], p[3][3]);

     FA f19 (sf19, cf19, p[5][2], p[4][3], p[3][4]);

     FA f110 (sf110, cf110, p[5][3], p[4][4], p[3][5]);

     FA f111 (sf111, cf111, p[5][4], p[4][5], p[3][6]);

     FA f112 (sf112, cf112, p[5][5], p[4][6], p[3][7]);

     HA h14 (sh14, ch14, p[5][6], p[4][7]);

    // stage 1 _ first line

     HA h21 (out[2], ch21, ch11, sf11);

     FA f21 (sf21, cf21, cf11, p[3][0], sf12);

     FA f22 (sf22, cf22, cf12, sf13, sh13);

     FA f23 (sf23, cf23, cf13, sf14, sf17);

     FA f24 (sf24, cf24, cf14, sf15, sf18);

     FA f25 (sf25, cf25, cf15, sf16, sf19);

     FA f26 (sf26, cf26, cf16, sh12, sf110);

     FA f27 (sf27, cf27, ch12, p[2][7], sf111);

    // stage 1 _ second line

    HA h22 (sh22, ch22, cf17, p[6][0]);

   FA f28 (sf28, cf28, cf18, p[6][1], p[7][0]);

   FA f29 (sf29, cf29, cf19, p[6][2], p[7][1]);

    FA f210 (sf210, cf210, cf110, p[6][3], p[7][2]);

    FA f211 (sf211, cf211, cf111, p[6][4], p[7][3]);

    FA f212 (sf212, cf212, cf112, p[6][5], p[7][4]);

    FA f213 (sf213, cf213, p[5][7], p[6][6], p[7][5]);

    HA h23 (sh23, ch23, p[6][7], p[7][6]);

    // stage 2

    HA h31 (out[3], ch31, ch21, sf21);

    HA h32 (sh32, ch32, cf21, sf22);

    FA f31 (sf31, cf31, cf22, ch13, sf23);

    FA f32 (sf32, cf32, cf23, sf24, sh22);

    FA f33 (sf33, cf33, cf24, sf25, sf28);

    FA f34 (sf34, cf34, cf25, sf26, sf29);

    FA f35 (sf35, cf35, cf26, sf27, sf210);

    FA f36 (sf36, cf36, cf27, sf112, sf211);

    HA h33 (sh33, ch33, sh14, sf212);

    HA h34 (sh34, ch34, ch14, sf213);

    // stage 3

    HA h41 (out[4], ch41, ch31, sh32);

    HA h42 (sh42, ch42, ch32, sf31);

    HA h43 (sh43, ch43, cf31, sf32);

    FA f41 (sf41, cf41, cf32, ch22, sf33);

    FA f42 (sf42, cf42, cf33, cf28, sf34);

    FA f43 (sf43, cf43, cf34, cf29, sf35);

    FA f44 (sf44, cf44, cf35, cf210, sf36);

    FA f45 (sf45, cf45, cf36, cf211, sh33);

    FA f46 (sf46, cf46, ch33, cf212, sh34);

    FA f47 (sf47, cf47, ch34, cf213, sh23);

    HA h44 (sh44, ch44, ch23, p[7][7]);

    // stage 4 ripple carry adder

    HA h_a (out[5], c0, ch41, sh42);

     FA f_a (out[6], c1, c0, ch42, sh43);

   FA f_b (out[7], c2, c1, ch43, sf41);  

    FA f_c (out[8], c3, c2, cf41, sf42);  

   FA f_d (out[9], c4, c3, cf42, sf43);  

   FA f_e (out[10], c5, c4, cf43, sf44);

     FA f_f (out[11], c6, c5, cf44, sf45);

    FA f_g (out[12], c7, c6, cf45, sf46);

     FA f_h (out[13], c8, c7, cf46, sf47);

     FA f_i (out[14], c9, c8, cf47, sh44);

   HA h_b (out[15], c10, c9, ch44);

  

endmodule

  

module tb;

  

  reg [7:0] a, b;

  wire [15:0] mout;

  

  multiplier m1 (a, b, mout);

  wire [15:0] test;

  assign test = a * b;

  

  initial

  begin

     a = 4'd0;  b=4'd0;

  #1 a = 4'd0;  b=4'd1;

  #1 a = 4'd7;  b=4'd5;

  #1 a = 4'd10; b=4'd10;

  #1 a = 4'd15; b=4'd15;

  #1 a = 4'd13; b=4'd12;

  

  end

  initial

  begin

    $monitor("a*b: %d, wallace: %d", test, mout);

    $dumpfile("dump.vcd"); $dumpvars;

  end

endmodule

```
