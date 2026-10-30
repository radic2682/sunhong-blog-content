---
id: "01HHG7J8006SQE08MWA9AA6F5Y"
slug: "verilog-slot-machine-lfsr"
title: "Verilog 슬롯머신 설계: LFSR 난수 생성과 타이머 시드 (Basys3)"
category: "01M651PC00H3J4W7JXTZ32PR45"
tags: ["verilog","vivado"]
summary: "Timer를 이용한 슬롯머신 설계"
thumbnail: "/img/01HHG7J8006SQE08MWA9AA6F5Y/eacfeba19c01.webp"
note: false
created: "2023-12-13T09:00:00+09:00"
updated: "2026-09-15T02:10:29+09:00"
---
## 1. 설계 내용

우선 start 신호가 0인 경우 start 신호가 0인 경우 출력은 입력을 x1배 한 값이 출력되며, 3개의 random 값은 000으로 초기화된다. start 신호가 1인 경우에는 아래의 슬롯머신의 동작이 수행된다. 우선 Timer를 이용하여 Linear Feedback Shift Register에 Seed값을 부여하고 이를 이용해 0~9까지의 3개의 BCD 형태의 4비트 이진수 random 값을 3개를 출력한다. 입력으로는 10진수 1~100까지의 숫자를 이진수 형태로 7비트 입력으로 부여할 수 있으며, BCD 형태의 4비트 이진수 숫자가 3개가 일치하는 경우는 x10배의 출력을, 숫자 2개가 일치하는 경우 x5배의 출력을, 숫자가 1개가 일치하는 경우 x2배의 입력 대비 값이 이진수 형태로 출력된다.

## 2. 블록도 및 서브 블록의 정의
### 1) 블록도

![블록도|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/eacfeba19c01.webp)

![블록도 (2)|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/985b13fc2a7f.webp)

### 2) 서브 블록

#### (1) random module

```verilog
module random (clock, reset_b, start, q, random);
    input wire clock, reset_b, start;
    input wire [15:0] q;
    output reg [15:0] random;

    parameter magic = 16'd6;

    reg [15:0] count;
    reg randomed;

    always @(posedge clock, negedge reset_b)
    begin
        if (~reset_b) begin
            count <= 16'b0;
            random <= 16'b0;
            randomed <= 1'b0;
        end
        else if (~start) begin
            count <= 16'b0;
            random <= 16'b0;
            randomed <= 1'b0;
        end
        else if (start & (count == magic) & ~randomed) begin
            count <= count;
            random <= q;
            randomed <= 1'b1;
        end
        else begin
            count <= count + 1;
            random <= random;
            randomed <= randomed;
        end
    end
endmodule
```

`입력`: clock, reset_b, start, q

`출력`: random

random은 clock, reset_b, start, q를 입력으로 받아 random값을 출력하는 모듈이다. q는 lfsr의 출력값으로 16bit의 랜덤값이다. posedge에서 q는 매번 달라진다. 이는 lfsr 모듈의 설명에서 자세히 다룬다.

magic이라는 파라미터는 이 lfsr 모듈의 q를 6+1번 달라지게 한다. 즉, lfsr 모듈을 7번 뒤에, 7번의 posedge후에 나오는 값을 random의 출력으로 나올 수 있도록 해준다. count가 0부터 magic까지 증가하는데, 이는 else 구문에서 확인할 수 있다. 또한, 만약 count가 7에 도달했다면 randomed는 1이 되어 더 이상 반복되지 않는다.

magic값은 임의의 값으로 만약 15번 섞어주고 싶다면 14로 설정하면 된다.

#### (2) lfsr module

```verilog
module lfsr (clock, reset_b, seed_pulse, count, q);
    input wire clock, reset_b, seed_pulse; // seed_pulse: seed를 딱 한번만 주기 위해 사용
    input wire [15:0] count;

    output wire [15:0] q;

    reg [15:0] r_reg;
    wire [15:0] r_next;
    wire feedback_value;

    always @(posedge clock, negedge reset_b)
    begin
        if (~reset_b)
            r_reg <= 16'b1;
        else if (seed_pulse)
            r_reg <= count;
        else
            r_reg <= r_next;
    end

    assign feedback_value = r_reg[13] ^ r_reg[4] ^ r_reg[0]; //lfsr의 섞어주는 기능

    assign r_next = {feedback_value, r_reg[15:1]};
    assign q = r_reg;
endmodule
```

`입력`: clock, reset_b, seed_pulse, count

`출력`: q

입력으로 clock, reset_b, seed_pulse, count를 받아, 출력으로 q를 출력한다. 여기서 출력 q는 위 ramdom module의 input의 활용된다.

lfsr은 선형 되먹임 시프트 레지스터이다. 시프트 레지스터의 일종으로, 레지스터에 입력되는 값이 이전 상태 값들의 선형 함수로 계산되는 구조를 가지고 있다. 이때 사용되는 선형 함수는 주로 배타적 논리합(XOR)이다. LFSR의 초기 비트 값은 시드(seed)라고 부른다. LFSR의 동작은 결정론적이기 때문에, LFSR로 생성되는 값의 수열은 그 이전 값에 의해 결정된다. 또한, 레지스터가 가질 수 있는 값의 개수는 유한하기 때문에, 이 수열은 특정한 주기에 의해 반복된다. 하지만 선형 함수를 잘 선택한다면 주기가 길고 무작위적으로 보이는 수열을 생성할 수 있다. 즉, 난수를 만들 수 있다.

다음상태에 영향을 주는 비트 위치의 목록은 탭 수열이라고 불린다. 여기서는 13, 4, 0을 선택했다. 만약 다른 값을 선택해도 괜찮다. 섞어주는 방식, 즉 섞어주는 방정식이 달라질 뿐이다.

seed값은 lsfr이 동작을 시작하는 시점에서 한번만 부여해야 하는 특징이 있다. 이를 구현하기 위해 seed_pulse를 이용한다. see_pulse는 counter module로부터 받아온다. 어떻게 한번만 주는 지는 counter module에서 기술했다. feedback_value는 섞인 값이며 선형 레지스터의 [15]에 이 값이 들어가게 되고 [0]에 있던 값은 밀려 사라지게 된다. 이를 위 코드에서는 assign문으로 기술하였고 r_next이다. 즉, clock이 한번 동작할 때마다 섞인 값이 [15]에 들어가고 한자리씩 밀린다. 이렇게 생긴 랜덤값은 r_reg에 할당되고 다시 q에 할당되어 출력값이 된다.

위 random module에서 magic은 이 lfsr을 몇 번 돌려줄지, 몇 번 섞어줄지를 결정한다. 그 수는 클럭의 posdege에서 동작하므로 클럭의 횟수와도 같다.

lfsr의 시드값으로는 counter의 값을 사용한다. 즉, 시스템이 작동 후 counter가 증가하게 되고, start가 1이 됐을 때 counter값이 lfsr에 들어가게 된다. 위에서 설명했듯이 이 행위는 초기화가 될 때까지 단 한번 발생한다.

![lfsr module|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/3f225fa9b01f.webp)

#### (3) counter

```verilog
module counter (clock, reset_b, start, count, seed_pulse);
    input wire clock, reset_b, start;
    output reg [15:0] count;
    output reg seed_pulse;

    always @(posedge clock, negedge reset_b)
    begin
        if (~reset_b)
            count <= 16'b0;
        else
            count <= count + 1;
    end

    reg pushed;

    // 딱한번만 줄때 유용하게 사용  
    // pushed 값 설정 <start가 0이 되기 전까지 1을 유지, start가 0이되면 pushed를 0으로 만듦>
    always @(posedge clock, negedge reset_b)
    begin
        if (~reset_b)
            pushed <= 1'b0;
        else if (~pushed & start)
            pushed <= 1'b1;
        else if (~start)
            pushed <= 1'b0;
    end

    // seed_pulse의 생성  => start가 1이 될때 딱 한번만 seed_pulse는 1이 되며, 다를 때는 0유지
    always @(posedge clock, negedge reset_b)
    begin
        if (~reset_b) // reset:1
            seed_pulse <= 1'b0;
        else if (~pushed & start) // pushed:0 start:1  => start가 0에서 1로 바뀔때, pushed는 0이므로 딱 한번 발생
            seed_pulse <= 1'b1;
        else // pushed:1
            seed_pulse <= 1'b0;
    end
endmodule
```

`입력`: clock, reset_b, start

`출력`: count, seed_pulse

clock에 따라 증가하는 counter가 첫 번째 always 문에 기술되어 있다. 또한, lfsr에서 딱 한번의 seed값을 주기 위한 seed_pulse가 있고, seed_pluse를 만들기 위한 pushed를 만들기 위한 always문이 기술되어 있다.

pushed는 start가 0일때나 reset_b가 0이면 0이 입력되고, pushed가 0이고 start가 1이면 1이 입력된다. 즉, pushed가 0이고 start가 1이 될 때, seed_pulse는 1이 된다. 이후 pushed는 0이 되어 start가 1이 될 때, 단 한번만 작동한다.

#### (4) binary to BCD

```verilog
module binary_to_BCD(binary, digit10000, digit1000, digit100, digit10, digit0);
    input [15:0] binary;
    output reg [3:0] digit10000, digit1000, digit100, digit10, digit0;
    wire [3:0] c1,c2,c3,c4,c5,c6,c7,c8,c9,c10,c11,c12,c13,c14,c15,c16,c17,c18,c19,c20,c21,c22,c23,c24,c25,c26,c27,c28,c29,c30,c31,c32,c33,c34; // add3's output
    reg [3:0] d1,d2,d3,d4,d5,d6,d7,d8,d9,d10,d11,d12,d13,d14,d15,d16,d17,d18,d19,d20,d21,d22,d23,d24,d25,d26,d27,d28,d29,d30,d31,d32,d33,d34; // add3's input

    add3 a1(d1,c1); add3 a2(d2,c2); add3 a3(d3,c3); add3 a4(d4,c4); add3 a5(d5,c5);
    add3 a6(d6,c6); add3 a7(d7,c7); add3 a8(d8,c8); add3 a9(d9,c9); add3 a10(d10,c10);
    add3 a11(d11,c11); add3 a12(d12,c12); add3 a13(d13,c13); add3 a14(d14,c14); add3 a15(d15,c15);
    add3 a16(d16,c16); add3 a17(d17,c17); add3 a18(d18,c18); add3 a19(d19,c19); add3 a20(d20,c20);
    add3 a21(d21,c21); add3 a22(d22,c22); add3 a23(d23,c23); add3 a24(d24,c24); add3 a25(d25,c25);
    add3 a26(d26,c26); add3 a27(d27,c27); add3 a28(d28,c28); add3 a29(d29,c29); add3 a30(d30,c30);
    add3 a31(d31,c31); add3 a32(d32,c32); add3 a33(d33,c33); add3 a34(d34,c34);

    always @ (*) begin
        // 1st vertical line from right
        d1 <= {1'b0, binary[15:13]};
        d2 <= {c1[2:0], binary[12]};
        d3 <= {c2[2:0], binary[11]};
        d4 <= {c3[2:0], binary[10]};
        d5 <= {c4[2:0], binary[9]};
        d6 <= {c5[2:0], binary[8]};
        d7 <= {c6[2:0], binary[7]};
        d8 <= {c7[2:0], binary[6]};
        d9 <= {c8[2:0], binary[5]};
        d10 <= {c9[2:0], binary[4]};
        d11 <= {c10[2:0], binary[3]};
        d12 <= {c11[2:0], binary[2]};
        d13 <= {c12[2:0], binary[1]};

        // 2nd vertical line from right
        d14 <= {1'b0, c1[3], c2[3], c3[3]};
        d15 <= {c14[2:0], c4[3]};
        d16 <= {c15[2:0], c5[3]};
        d17 <= {c16[2:0], c6[3]};
        d18 <= {c17[2:0], c7[3]};
        d19 <= {c18[2:0], c8[3]};
        d20 <= {c19[2:0], c9[3]};
        d21 <= {c20[2:0], c10[3]};
        d22 <= {c21[2:0], c11[3]};
        d23 <= {c22[2:0], c12[3]};

        // 3rd vertical line from right
        d24 <= {1'b0, c14[3], c15[3], c16[3]};
        d25 <= {c24[2:0], c17[3]};
        d26 <= {c25[2:0], c18[3]};
        d27 <= {c26[2:0], c19[3]};
        d28 <= {c27[2:0], c20[3]};
        d29 <= {c28[2:0], c21[3]};
        d30 <= {c29[2:0], c22[3]};

        // 4th vertical line from right
        d31 <= {1'b0, c24[3], c25[3], c26[3]};
        d32 <= {c31[2:0], c27[3]};
        d33 <= {c32[2:0], c28[3]};
        d34 <= {c33[2:0], c29[3]};

        // output
        digit0 <= {c13[2:0], binary[0]};
        digit10 <= {c23[2:0], c13[3]};
        digit100 <= {c30[2:0], c23[3]};
        digit1000 <= {c34[2:0], c30[3]};
        digit10000 <= {c31[3], c32[3], c33[3], c34[3]};
    end
endmodule
```

![binary to BCD|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/100a53c189d0.webp)

`입력`: binary

`출력`: digit10000, digit1000, digit100, digit10, digit0

Binary to BCD, 즉 이진수를 BCD로 바꿔주는 회로를 기술하였다. BCD는 이진수 네 자리를 묶어 십진수 한 자리로 사용하는 기수법이다. 변환하는 방법은 다음과 같다.

① Binary 값을 차례로 MSB 비트부터 시프트하여 처리

② 5보다 크거나 같다면 3을 더하고 시프트 처리

③ 5보다 작다면 3을 더하지 않고 시프트 처리

이에 대해 for문으로 기술하면 다음과 같다. 하지만 for문은 합성이 안되기 때문에 아래 그림과 같이 조합회로로 다시 직접 기술하였다.

```verilog
module bin2bcd(
   input [13:0] bin,
   output reg [15:0] bcd
   );
   
integer i;
    
always @(bin) begin
    bcd=0;          
    for (i=0;i<14;i=i+1) begin                  //Iterate once for each bit in input number
        if (bcd[3:0] >= 5) bcd[3:0] = bcd[3:0] + 3;     //If any BCD digit is >= 5, add three
    if (bcd[7:4] >= 5) bcd[7:4] = bcd[7:4] + 3;
    if (bcd[11:8] >= 5) bcd[11:8] = bcd[11:8] + 3;
    if (bcd[15:12] >= 5) bcd[15:12] = bcd[15:12] + 3;
    bcd = {bcd[14:0],bin[13-i]};                //Shift one bit, and shift in proper bit from input 
    end
end
endmodule
```

![binary to BCD (2)|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/0a3ace8bf459.webp)

[출처](https://slideplayer.com/slide/7076551/)

test bench를 작성하여 확인하였고, 결과가 올바르게 나오는 것을 확인할 수 있었다. 16bit의 binary이기 때문에 BCD로 변경하게 되면 5자리가 나오게 된다. 우리가 필요한 것은 3자리이므로 나머지 자리는 버려주면 된다. verilog에서는 wire를 연결하지 않고 놔두어도 괜찮기 때문에 이를 메인 module에서는 floating 상태로 두었다. 하지만 지금 16bit binary to BCD module의 역할은 5자리를 출력하는 것이기 때문에 언제든지 파생할 수 있게, 코드를 재활용할 수 있도록 output은 5자리로 놔두었다.

#### (5) add3

```verilog
module add3(in,out);
    input [3:0] in;
    output [3:0] out;
    reg [3:0] out;

    always @ (in)
    case (in)
        4'b0000: out <= 4'b0000;
        4'b0001: out <= 4'b0001;
        4'b0010: out <= 4'b0010;
        4'b0011: out <= 4'b0011;
        4'b0100: out <= 4'b0100;
        4'b0101: out <= 4'b1000;
        4'b0110: out <= 4'b1001;
        4'b0111: out <= 4'b1010;
        4'b1000: out <= 4'b1011;
        4'b1001: out <= 4'b1100;
        default: out <= 4'b0000;
    endcase
endmodule
```

이 모듈은 binary to BCD module에서 쓰기 위한 하위 모듈이다. 5[0101(2)]를 넘어간다면 3을 더해주는 덧셈기이다. 예를 들어 7[0111(2)] -> 10[1010(2)], 7은 5를 넘으므로 3을 더한 10을 출력해준다.

### 3) 메인 블록

```verilog
module slot_machine (clock, start, reset_b, in_value1, in_value2, in_value3, in_money, out_random3, out_random2, out_random1, out_money);
    input wire clock, start, reset_b;
    input wire [3:0] in_value1, in_value2, in_value3;
    input wire [6:0] in_money;
    output wire [3:0] out_random3, out_random2, out_random1; // BCD의 5자리 중 뒷 3자리만 사용
    output reg [9:0] out_money;
```

input과 output은 위와 같다. in_value1, in_value2, in_value3는 슬롯머신 사용자가 입력하는 값으로 랜덤값과 비교하여 숫자가 일치한다면 그에 상응하는 돈을 출력하게 된다. 또한, 사용자의 in_money를 입력으로 받는다.

out_random1, out_random2, out_random3은 생성된 랜덤값으로 사용자에게 보여주기 위해 출력으로 지정하였다. 또한 사용자에게 돈을 돌려주기 위해 out_money라는 값을 출력으로 설정했다.

```verilog
    // connection
    reg [2:0] state;
    reg [2:0] next_state;
    reg [6:0] memory_in_money;
    reg memory_in_money_pushed;

    // parameter
    parameter S0 = 2'd0, S1 = 2'd1, S2 = 2'd2, S3 = 2'd3;

    // wire
    wire [15:0] random;
    wire [15:0] q; // lfsr 모듈의 출력값, random 모듈의 입력값
    wire seed_pulse; // 1bit짜리 seed pulse => clock 모듈에서 나오는 값 "한번만 실행할 때를 나타냄"
    wire [15:0] count; // counter 모듈의 count값으로 lfsr의 seed로 사용
    wire [3:0] digit1000, digit10000; //남은 값들 (BCD 1000자리, 10000자리 값들)

    // 파생
    random r1 (clock, reset_b, start, q, random);
    lfsr l1 (clock, reset_b, seed_pulse, count, q);
    counter c1 (clock, reset_b, start, count, seed_pulse);
    // binary_to_BCD 파생: ramdom 모듈의 16비트 random값을 받아 BCD 5자리 추출
    binary_to_BCD btb1 (.binary(random), .digit10000(digit10000), .digit1000(digit1000), .digit100(out_random3), .digit10(out_random2), .digit0(out_random1));
```

connection 주석 부분의 state와 nest_state는 reg 타입으로 각각 현재 상태와 다음 상태를 기억한다. 또한, memory_in_money는 start가 1이 되었을 때 1~100의 값이 올바르게 들어왔다면 그 때의 값을 저장하기 위해 사용된다. memory_in_money_pushed는 단 한번만 저장하기 위해서 사용된다. 계속 돈을 저장하게 되면 값이 바뀌기 때문이다.

```verilog
       // 클럭의 상승엣지에서만 작동됨 reset_b에 1이 들어오면 언제든지 초기화
    always @ (posedge clock, negedge reset_b)
    begin
        if (~reset_b) // reset_b:0 -> 상태 초기화
            state <= S0;
        else // 일반적으로 state가 다음 state로 변경
            state <= next_state;
    end
```

next_state의 값이 변경이 된다면 현재 상태 state의 값을 다음 상태로 바꿔주기 위해 사용된다. 이 state값은 밑에서 case문에 의해 상태 변화가 일어나는 것처럼 기술이 가능하다. 만약 reset_b의 값이 0이 된다면 상태를 초기화시키는 문장도 서술하였다.

```verilog
   // start:1 이고 1<=in_money<=100일때의 in_money를 memory_in_money에 단 한번 저장
    always @(posedge clock, negedge reset_b)
    begin
        if (~reset_b) begin
            memory_in_money_pushed <= 1'b0;
            memory_in_money <= 7'b0;
        end
        else if ((in_money >= 7'd1 & in_money <= 7'd100) & random == 0 & ~memory_in_money_pushed & start) begin
            memory_in_money_pushed <= 1'b1;
            memory_in_money <= in_money;

        end
        else if (~start) begin
            memory_in_money_pushed <= 1'b0;
            memory_in_money <= 7'b0;
        end
    end
```

in_money는 1에서 100의 값을 가져야 하고 만약 그 값이 아닌 다른 값이 입력으로 들어온다면 올바른 값이 들어올 때까지 대기해야 한다. 그리고 만약 올바른 값이 들어온다면 memory_in_money_pushed을 통해 memory_in_money에 그 값을 저장하게 된다.

```verilog
   // 상태천이를 위한 기술
    always @ (*)
    begin
        case(state)
            S0: begin
                if (~start) begin
                    out_money <= in_money;
                    next_state <= S0;
                end
                else begin // start: 1
                    out_money <= in_money;
                    next_state <= S1;
                end
            end
```

S0 상태에 대한 기술이다. 단순히 start가 1이 될 때까지 대기하면서, 입력의 돈을 출력으로 그대로 출력해준다.

```verilog
		     S1: begin
                if (random != 0)begin
                    next_state <= S2; // random값이 들어와야 S2로 넘어감
                end
                else if (in_money == 0 | in_money > 7'd100 & random == 0) begin // 1~100의 머니가 들어오지 않았을때
                    next_state <= S1;
                    out_money <= in_money;
                end
                else if ((in_money >= 7'd1 & in_money <= 7'd100) & random == 0) begin // 1~100의 머니이고 random이 0일때 
                    out_money <= 1'b0;
                    next_state <= S1;
                end

            end
```

S1 상태에 대한 기술이다. if문을 통해 여러 경우를 나누었다. 우선 랜덤이 0이면 S1 상태를 계속 유지한다. 그 이유는 랜덤값이 들어와야 사용자가 입력한 값과 비교를 할 수 있기 때문이다. 이 비교하는 상태가 S3인데, 랜덤값이 0일 때 S3가 되면 잘못된 값으로 비교할 수 있다는 문제를 가진다. S1 상태를 유지하면서 in_money가 1~100까지의 값을 받을 때까지 기다린다.

```verilog
            S2: begin
                if (out_random3 == in_value3 & out_random2 == in_value2 & out_random1 == in_value1) begin
                    out_money <= 4'b1010 * memory_in_money;
                    next_state <= S3;
                end
                else if ((out_random3 == in_value3 & out_random2 == in_value2) | (out_random3 == in_value3 & out_random1 == in_value1) | (out_random2 == in_value2 & out_random1 == in_value1)) begin
                    out_money <= 4'b101 * memory_in_money;
                    next_state <= S3;
                end
                else if (out_random3 == in_value3 | out_random2 == in_value2 | out_random1 == in_value1) begin
                    out_money <= 2'b10 * memory_in_money;
                    next_state <= S3;
                end
                else begin
                    out_money <= 1'b0 * memory_in_money;
                    next_state <= S3;
                end
            end
```

S2 상태는 각 자리를 비교하여 조건에 맞는 out_money를 설정해준다. 이진수 숫자가 3개가 일치하는 경우는 x10배의 출력을, 숫자 2개가 일치하는 경우 x5배의 출력을, 숫자가 1개가 일치하는 경우 x2배의 입력 대비 값이 이진수 형태로 출력된다. 또한, 다음 상태인 S3로 이동한다.

```verilog
            S3: begin // start가 1일때 S0으로 가는 것을 방지
                if (~start) // start:0 -> S0으로 이동
                    next_state <= S0;
                else begin
                    next_state <= S3;
                    out_money <= 7'b0;
                end
            end
            default state <= S0;
        endcase
    end
endmodule
```

S3에서는 start가 0인지 1인지 판단한다. 만약 start가 1일 때 S0로 간다면 바로 S1으로 넘어가서 초기화를 하지 못하는 문제가 발생할 수 있기에 이를 방지한다.

## 4. 시뮬레이션

```verilog
module tb;

    reg clock, reset_b, start;

    wire [15:0] count, q, random;
    wire seed_pulse;

    reg [3:0] in_value1, in_value2, in_value3;
    reg [6:0] in_money;
    wire [3:0] out_random3, out_random2, out_random1;
    wire [9:0] out_money;

    slot_machine sm1 (clock, start, reset_b, in_value1, in_value2, in_value3, in_money, out_random3, out_random2, out_random1, out_money);

    counter c1 (clock, reset_b, start, count, seed_pulse);
    lfsr c2 (clock, reset_b, seed_pulse, count, q);
    random c3 (clock, reset_b, start, q, random);

    initial
    clock = 0;
    always
    #5 clock = ~clock;

    initial
    begin
        reset_b = 0; // 0
        #10 reset_b =1; //10
        #380 reset_b = 0;
    end

    initial
    begin
        start = 0;
        #40 start =1;
        #100 start = 0;
        #80 start =1;
        #100 start = 0;
        #50 start = 1;
        #55 $finish;
    end

    initial
    begin
        in_value3 = 4'd1; in_value2 = 4'd3; in_value1 = 4'd4;
        #20 in_value3 = 4'd2; in_value2 = 4'd3; in_value1 = 4'd4;
        #70 in_value3 = 4'd2; in_value2 = 4'd3; in_value1 = 4'd4;
        #70 in_value3 = 4'd4; in_value2 = 4'd2; in_value1 = 4'd1;
        #100 in_value3 = 4'd4; in_value2 = 4'd4; in_value1 = 4'd1;
        #100 in_value3 = 4'd2; in_value2 = 4'd0; in_value1 = 4'd1;
    end

    initial
    begin
        in_money = 7'b1110;
        #10 in_money = 7'b11010;
        #30 in_money = 7'b111_1111;
        #20 in_money = 7'b1_1011;
        #10 in_money = 7'b0;
        #90 in_money = 7'b110;
        #20 in_money = 7'b1_1011;
        #60 in_money = 7'b0;
        #150 in_money = 7'b110;
    end

    initial
    $monitor("time:%3d  ||  start: %b, reset_b: %b, state: S%d  ||  count: %d  ||  lfsr: %b, random: %d  ||  BCD: %d, %d, %d, in_value: %d, %d, %d  ||  in_money: %d  ->  out_money: %d  ||", $time, start, reset_b, sm1.state, count, q, random, out_random3, out_random2, out_random1, in_value3, in_value2, in_value1, in_money, out_money);
endmodule
```

![시뮬레이션|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/e87c2f1e9f28.webp)

![시뮬레이션 (2)|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/7a94a521bc9f.webp)

![시뮬레이션 (3)|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/cadd6eceeb70.webp)

![시뮬레이션 (4)|medium](/img/01HHG7J8006SQE08MWA9AA6F5Y/7d09dc53a8bf.webp)

## 5. 고찰
Timer를 이용한 슬롯머신을 verilog HDL 언어를 통해 기술하였다. 추가적인 lfsr에 대해 알 수 있었고, random의 생성 방법에 대해 알 수 있었다. 또한, 순차회로의 기술 방법에 대해 실습을 하면서 더욱 잘 알 수 있었다. 이에 따라 회로를 기술하는 능력이 향상되었음을 느꼈다.

결과로는 시뮬레이션의 결과를 보면 알 수 있다. 붉은색의 start 신호가 가해졌을 때, lfsr에 timer의 seed값이 잘 인가되는 것을 확인할 수 있었다. 또한, 이에 따라 정해진 magic 값만큼 clock이 흘렀을 때, lfsr이 정해진 만큼 섞였을 때 random값이 출력되고 이에 BCD로 변환한 것의 아래에서부터 3자리를 in_value와 비교하는 것을 보여준다. 27의 입력을 받았는데, 1자리가 일치하기에 2배한 돈이 출력되는 것을 확인할 수 있다.

또한, 127의 돈이 입력이 되면 1~100에 해당이 되지 않기에 다시 입력을 받는 것도 확인할 수 있었다.

두 번째 start일 때는 27의 입력을 받았고, 3자리가 일치하기 때문에 10배한 270값이 출력되었다.

마지막은 reset_b가 0이 되었을 때, 비동기 리셋 명령이 실행되어 모든값이 초기화되는 것을 확인할 수 있다. 이때, 머신이 돈을 가져가면 안되기에 돈은 입력을 그대로 출력하였다.

위 결과에 따라 주어진 명세서의 내용을 만족하는 설계를 마쳤다.
