---
id: "01M6K8DR5ES95EYKYDSYMGVVEC"
slug: "riscv-decoder"
title: "RISC-V 디코더 설계: opcode·funct3·funct7 표, 형식 판별, 불법 명령어 검출"
category: "01M6K19SKHFQ3SCEWS4D97Y48R"
tags: ["risc-v","decoder","rv32i","illegal instruction","objdump"]
summary: "RISC-V 15편. RV32I 디코드 표를 제어 신호까지 채우고 불법 명령어 조건 다섯 가지를 정한 뒤, 174,608개 단어로 GNU objdump와 대조했다. 165,923개가 같았고 다른 8,685개는 네 가지 원인으로 모두 설명됐다."
thumbnail: "/img/01M6K8DR5ES95EYKYDSYMGVVEC/6fe4e0df65fd.webp"
note: true
order: 15
created: "2026-11-04T21:27:04+09:00"
updated: "2026-11-04T21:27:04+09:00"
---
:::note
**이 글의 핵심 3줄**
- RV32I 디코더는 세 가지 일을 한다. 레지스터 번호를 고정 자리에서 꺼내 바로 보내고, opcode·funct3·funct7로 디코드 표의 한 줄(제어 신호 묶음)을 고르고, 표에 없는 조합을 불법 명령어로 표시한다. 주 opcode 32칸 중 RV32I가 쓰는 칸은 11개다.
- 불법 조건은 다섯 가지로 정리된다. 16비트 칸, 없는 opcode, 없는 funct3, 없는 funct7(시프트의 inst[25] 포함), SYSTEM의 나머지 값이다. 반대로 FENCE의 남는 칸과 rd=x0인 HINT는 통과시켜야 한다.
- Python으로 짠 디코더를 GNU objdump(rv32i로 제한)와 174,608개 단어로 대조했다. 165,923개가 같았고, 다른 8,685개는 모두 objdump 쪽이 명세와 다르게 보는 네 가지 경우로 설명됐다(설명 안 되는 불일치 0개).
:::

1~3편에서 RV32I의 모양(형식 여섯 가지와 즉시값 배치)과 정수 연산이 무엇을 계산하는지 봤다. 2편 끝의 체크리스트에서는 디코더가 지켜야 할 규칙을 몇 줄로 적어 두었다. 15편부터 시작하는 2부는 그 규칙을 실제 코어로 옮기는 이야기다. 첫 부품이 디코더다. 이 글에서는 RV32I 전체의 디코드 표를 제어 신호까지 채우고, 불법 명령어 검출 조건을 2편보다 촘촘하게 정한 뒤, 32비트 공간을 대량으로 훑어 GNU objdump와 판정을 맞춰 본다. 이 묶음(15~18편)은 RTL 대신 사이클 단위 Python 모델로 설계를 설명한다. 모델의 한 줄 한 줄이 나중에 RTL의 한 블록이 되도록 짰다.

## 1. 디코더가 하는 일

명령어 32비트가 들어오면 디코더는 다음을 내보낸다.

| 출력 | 어디로 | 언제 정해지나 |
|---|---|---|
| rs1·rs2·rd 번호 | 레지스터 파일 | 형식과 무관, 고정 자리 |
| 형식(I·S·B·U·J·R) | 즉시값 생성기 | opcode만 보고 |
| 제어 신호 | ALU·메모리·선택기 | opcode·funct3·funct7 |
| 불법 표시 | 예외 처리 | 표에 줄이 없을 때 |

![RV32I 디코더는 레지스터 번호를 고정 자리에서 바로 보내고, opcode·funct3·funct7로 디코드 표를 찾으며, 표에 없는 조합과 SYSTEM의 나머지 값을 불법으로 묶는다|full](/img/01M6K8DR5ES95EYKYDSYMGVVEC/6fe4e0df65fd.webp)

첫 줄이 RISC-V 디코더의 큰 장점이다. 2편에서 본 대로 rs1·rs2·rd는 어느 형식에서나 같은 비트에 있다. 그래서 디코더는 이 칸을 "해석"하지 않는다. 배선으로 레지스터 파일 주소에 연결하면 끝이다. U·J 형식에서는 rs1 자리에 즉시값 비트가 들어 있지만, 그때 읽은 값은 쓰지 않으면 된다.

다만 파이프라인을 만들 때는 주의할 점이 하나 생긴다. "이 명령어가 rs1을 정말 읽는가"를 모르면 해저드 검출이 헛짚는다. 예를 들어 `lui a0, 0x12345`의 rs1 자리에는 즉시값의 일부(여기서는 `0x08`)가 있다. 해저드 검출기가 이것을 진짜 rs1로 보면 x8을 쓰는 앞 명령어를 기다리느라 쓸데없이 멈춘다. 그래서 디코드 표에는 "rs1·rs2를 실제로 읽는가"라는 신호도 넣었다(17편에서 쓴다).

## 2. 주 opcode와 형식 판별

inst[1:0]이 `11`이면 32비트 명령어이고, 남은 inst[6:2] 5비트가 주 opcode(major opcode)다. 명세의 기본 opcode 표는 이 32칸을 inst[6:5](행)와 inst[4:2](열)로 펼쳐 보여 준다.

![주 opcode 32칸 중 RV32I가 쓰는 칸은 LOAD·STORE·BRANCH·JALR·JAL·MISC-MEM·OP-IMM·OP·AUIPC·LUI·SYSTEM의 11개이고 나머지 21칸은 RV32I 디코더에서 불법이다|full](/img/01M6K8DR5ES95EYKYDSYMGVVEC/15b0a3e70c94.webp)

RV32I가 쓰는 칸은 11개다. 나머지 21칸 중 17칸은 다른 표준 확장(F의 LOAD-FP, A의 AMO, RV64의 OP-32 등), 사용자 정의(custom-0~3), 예약이고, 마지막 열 4칸은 32비트보다 긴 명령어를 위해 비워 둔 자리다. RV32I만 구현한 코어에서는 21칸 모두 불법이다.

형식 판별은 opcode만으로 끝난다. funct3를 볼 필요가 없다.

| 주 opcode | 형식 | funct3로 가르는 것 | funct7을 보나 |
|---|---|---|---|
| LUI, AUIPC | U | 없음 | 아니오 |
| JAL | J | 없음 | 아니오 |
| JALR, LOAD, OP-IMM | I | 크기, 연산 | 시프트만 |
| STORE | S | 크기 | 아니오 |
| BRANCH | B | 비교 조건 | 아니오 |
| OP | R | 연산 | 예 |
| MISC-MEM, SYSTEM | I | FENCE / ECALL·EBREAK | 별도 규칙 |

이 표를 논리식으로 옮기면 형식 신호는 opcode 디코더 출력(one-hot 11개)의 OR 몇 개다. 예를 들어 `isI = JALR | LOAD | OPIMM | MISCMEM | SYSTEM`이다. 즉시값 생성기는 이 다섯 신호(I·S·B·U·J)만 받는다(16편).

손으로 한 줄 따라가 보자. `sub a0, a0, a1`은 `0x40b50533`이다.

```text
funct7  rs2   rs1   f3  rd    opcode
0100000 01011 01010 000 01010 0110011
```

opcode `0110011`은 OP, 그러니 R 형식이다. funct3 `000`은 add 또는 sub이고, funct7 `0100000`(inst[30]=1)이 sub를 고른다. rs1=10, rs2=11, rd=10은 형식과 상관없이 같은 자리에서 나온다. funct7만 `0000001`로 바꾼 `0x02b50533`은 M 확장의 `mul`이다. RV32I 디코더의 OP 줄에는 funct7 `0000001`이 없으니 불법이다.

## 3. 디코드 표: 제어 신호까지

RV32I 명령어 37개와 FENCE·ECALL·EBREAK를 합친 40줄이 표 전부다. 아래는 모델의 표(`15/rv32i.py`)를 그대로 찍은 것이다. 같은 opcode 안에서 비슷한 줄은 묶었다.

```text
이름        형식 rw A B op    mem  wb 분기/점프
lui          U   1  z i add   -    a  -
auipc        U   1  p i add   -    a  -
jal          J   1  - - -     -    4  jal
jalr         I   1  r i add   -    4  jalr
beq…bgeu     B   0  r r -     -    -  eq ne lt ge ltu geu
lb lh lw     I   1  r i add   r1 r2 r4  m  -
lbu lhu      I   1  r i add   r1u r2u   m  -
sb sh sw     S   0  r i add   w1 w2 w4  -  -
addi…andi    I   1  r i (f3)  -    a  -
slli srli    I   1  r i sll srl -  a  -
srai         I   1  r i sra   -    a  -
add…and      R   1  r r (f3,f7) -  a  -
fence        I   0  - - -     -    -  (아무것도 안 함)
ecall ebreak I   0  - - -     -    -  트랩
```

칸의 뜻은 다음과 같다.

| 신호 | 값 | 뜻 |
|---|---|---|
| rw | 0/1 | 레지스터 쓰기(RegWrite) |
| A, B | r·p·z / r·i | ALU 입력: rs1·pc·0 / rs2·즉시값 |
| mem | r·w, 크기 | 로드·스토어, 바이트 수, u=부호 없음 |
| wb | a·m·4 | 쓰기 되돌림: ALU·메모리·pc+4 |

몇 가지 선택을 설명해 둔다. `lui`는 ALU 입력 A를 0으로 두고 "0 + 즉시값"을 계산하게 했다. 덧셈기 하나를 그대로 쓰기 위해서다(3편 체크리스트). `auipc`는 A에 pc를 넣는다. `jal`은 ALU를 쓰지 않는다. 목적지 pc+imm은 분기용 덧셈기가 따로 계산하고, rd에는 pc+4가 들어간다. `jalr`은 ALU로 rs1+imm을 계산한 뒤 최하위 비트를 지운다. 분기는 ALU 대신 비교기가 조건을 판단한다(16편에서 왜 그렇게 나눴는지 본다).

OP-IMM에서 funct7 자리를 보는 것은 시프트 세 개뿐이다. `addi`의 inst[30]은 즉시값 비트일 뿐이라, 이것을 sub 선택에 쓰면 3편에서 짚은 흔한 버그가 된다. 그래서 OP-IMM의 ALU 연산은 funct3로만 고르고, funct3가 `101`일 때만 inst[30]으로 srli·srai를 가른다.

## 4. 불법 명령어 검출 조건

2편 체크리스트를 디코더 회로의 조건으로 다시 쓰면 다섯 줄이다.

| 조건 | 예 | 근거 |
|---|---|---|
| ① inst[1:0] ≠ `11` | `0x00000000` | 1.5 길이 인코딩 |
| ② opcode가 11개 밖 | `0xffffffff` | 기본 opcode 표 |
| ③ funct3 없음 | `ld`(LOAD `011`) | 2.4~2.6 |
| ④ funct7 없음 | `mul`, 시프트 inst[25]=1 | 2.4 |
| ⑤ SYSTEM 나머지 | `0x00200073` | 2.8 |

① 모두 0인 단어(`0x00000000`)는 명세 1.5가 불법으로 정해 둔 값이다. 지워진 메모리로 뛰어든 실수를 잡으려는 것이다. C 확장이 없으면 inst[1:0]이 `00`인 단어는 32비트 명령어가 될 수 없으니 이 조건에 자연스럽게 걸린다. C 확장을 넣으면 이야기가 달라진다. 16비트 `0x0000`이 따로 불법이어야 하므로 압축 해제기에서 별도로 잡아야 한다(21편).

② 모두 1인 단어도 명세 1.5가 불법으로 정했다(ILEN=32인 RV32I에서). inst[6:0]=`1111111`은 긴 명령어 자리라 opcode 검사에 그대로 걸린다. 결과는 같지만 모델에서는 이유를 따로 셌다.

④ 시프트 즉시값의 inst[25]는 RV64I에서 shamt의 6번째 비트다. RV32I의 명령어 목록에서는 이 자리가 `0000000`/`0100000`으로 정해져 있어 inst[25]=1은 표에 없는 인코딩, 즉 예약(reserved) 인코딩이 된다. 명세 2.2는 예약 인코딩을 만났을 때의 동작을 정하지 않는다(UNSPECIFIED). 흔한 선택은 불법 명령어 예외다. 플랫폼이 표준용 예약 opcode에 예외를 요구할 수 있다는 주석도 같은 자리에 있다.

⑤ SYSTEM opcode에서 RV32I가 정한 값은 `ecall`(`0x00000073`)과 `ebreak`(`0x00100073`) 정확히 둘이다. CSR 명령어(funct3 ≠ 0)는 Zicsr 확장(7장)이고, `mret`·`wfi` 같은 값은 특권 명세에서 온다. 이 글의 디코더는 RV32I만 받으므로 둘 다 불법으로 둔다. 실제 M 모드 코어라면 Zicsr와 특권 명령어 줄을 표에 더하게 된다(23~26편).

반대로 **불법으로 만들면 안 되는 것**이 두 가지다.

- **FENCE의 남는 칸.** 명세 2.7은 FENCE의 rs1·rd를 미래용으로 남기면서 "기본 구현은 이 칸을 무시해야 한다"고 정했다. fm과 pred·succ의 예약 조합도 "fm=0000인 보통 FENCE로 처리하라"고 했다. 그래서 MISC-MEM에서는 funct3=`000`만 확인하면 된다. FENCE.I(funct3 `001`)는 Zifencei 확장(6장)이라 RV32I만 구현하면 불법이다.
- **HINT.** rd=x0인 연산 명령어(`addi x0, x5, 3`, `add x0, x0, x5` 등)와 pred나 succ가 0인 FENCE는 HINT다(명세 2.9). 평소처럼 실행하고 x0 쓰기만 버리면 된다. 디코더 입장에서는 아무 처리도 하지 않는 것이 정답이다. 이 "아무 처리도 안 하는" HINT가 17편 파이프라인에서 함정이 된다.

## 5. 실습: objdump와 판정 대조

디코더의 "합법/불법" 판정을 무엇과 대조할까. 2편처럼 GNU 도구를 기준으로 쓰되, 이번에는 어셈블러가 아니라 디스어셈블러 objdump다. 32비트 단어를 넣었을 때 objdump가 명령어 이름을 찍으면 합법, `.insn`으로 찍으면 불법으로 본다.

여기에는 함정이 둘 있었다.

1. `.word 0x…`로 넣으면 objdump가 데이터로 보고 디스어셈블하지 않는다. 어셈블러가 데이터 위치를 표시하는 매핑 심볼 `$d`를 붙이기 때문이다. `objcopy --strip-symbol='$d'`로 지워야 명령어로 읽는다.
2. objdump가 어떤 확장까지 디코드할지는 ELF의 `Tag_RISCV_arch` 속성이 정한다. `-march=rv32i`로 어셈블하면 이 값이 `rv32i2p1`이 되고, objdump는 `-M max`를 주지 않는 한 이 범위만 디코드한다. `-M max`를 주면 같은 표본에서 58,361개가 더 "합법"이 됐다(가장 많은 것은 F·D·Q·Zfh의 `fmadd` 계열). 이 objdump 버전에는 `-M`으로 확장을 고르는 옵션이 따로 없으니, 범위 제한은 속성으로 하는 것이 맞다.

```python
def objdump(ws):
    src = '\n'.join(f'.word 0x{w:08x}' for w in ws)
    open('a.s', 'w').write(src)
    run(['riscv64-elf-as', '-march=rv32i',
         '-o', 'a.o', 'a.s'])
    run(['riscv64-elf-objcopy', '--strip-symbol=$d',
         'a.o', 'b.o'])
    out = run(['riscv64-elf-objdump', '-d',
               '-M', 'no-aliases', 'b.o'])
    # 줄마다 주소 → 첫 낱말(명령어 이름 또는 .insn)
```

표본은 세 묶음(시드 15)이다. (a) opcode 7비트·funct3·funct7의 17비트 조합 전부에 나머지 칸을 무작위로 채운 것 4벌, (b) 32비트 균등 무작위 20만 개, (c) SYSTEM과 MISC-MEM의 12비트 즉시값 칸 4,096가지 전부(rs1·rd가 0인 것과 아닌 것). 겹치는 것을 빼면 740,630개다. 이 중 inst[1:0]≠`11`이거나 긴 명령어 자리인 566,022개는 objdump가 32비트로 읽지 않으므로 내 디코더만 불법으로 판정하고, 나머지 174,608개를 objdump와 맞췄다.

| 내 디코더 | objdump | 개수 |
|---|---|---|
| 합법 | 합법 | 32,132 |
| 불법 | 불법 | 133,791 |
| 합법 | 불법 | 8,633 |
| 불법 | 합법 | 52 |

다른 8,685개를 원인별로 나눴다.

| 원인 | 개수 | 누가 맞나 |
|---|---|---|
| FENCE의 rd·rs1·fm 칸 | 8,633 | 명세(내 디코더) |
| 특권 명령어 | 38 | 범위 차이 |
| rv32에서 6비트 shamt | 13 | 명세(내 디코더) |
| `unimp`(`0xc0001073`) | 1 | 범위 차이 |

- **FENCE.** objdump는 rd·rs1이 0이고 fm이 `0000`이거나 정확한 FENCE.TSO(`0x8330000f`)일 때만 `fence`로 찍는다. 다른 값은 `.insn`이다. 불일치 8,633개 중 rd≠0이 4,771개, rs1≠0이 4,780개, 예약 fm이 8,337개였다(겹침 있음). 명세 2.7에 따르면 이것들은 모두 보통 FENCE로 실행해야 한다. 디스어셈블러는 "표준 소프트웨어가 쓰는 모양"만 보여 주는 도구라 이렇게 정한 것으로 보인다(추정). 디코더를 objdump에 맞춰 고치면 명세를 어기게 된다.
- **특권 명령어.** objdump는 `-march`와 상관없이 `mret`·`sret`·`wfi`·`dret`·`sfence.vma`를 디코드했다(32개가 `sfence.vma`). 지금은 쓰이지 않는 `uret`·`hret`도 나왔다. RV32I만 구현한 디코더와 범위가 다를 뿐이다.
- **6비트 shamt.** inst[25]=1인 시프트 714개 중 701개는 objdump도 불법으로 봤다. 13개가 갈렸는데, 모두 funct7 자리가 `0000001`(slli·srli)이나 `0100001`(srai)인 경우였다. RV64식 6비트 shamt 모양이면 rv32에서도 받아 주는 것이다. 예: `0x02009013`을 `slli zero,ra,0x20`으로 찍었다. RV32에서 32비트 시프트는 있을 수 없는 값이다.
- **`unimp`.** CSR 쓰기 `csrrw x0, cycle, x0`는 "구현되지 않은 명령어" 관용구로 `unimp`라는 이름이 붙어 있다. objdump는 Zicsr가 없어도 이 한 값만은 이름으로 찍었다.

네 가지를 빼면 설명되지 않는 불일치는 0개다. 같은 표본에서 디코더가 합법으로 본 단어 중 560개가 HINT였고, 디코더는 이것들을 그대로 통과시켰다.

무작위 32비트 단어 199,973개만 따로 보면 RV32I로 합법인 것은 8,967개(4.5%)였다. 4분의 3(150,087개)은 inst[1:0]≠`11`에서 걸렸고, 없는 opcode가 32,585개, 없는 funct3가 6,138개, 없는 funct7이 1,798개, 시프트 inst[25]가 202개, SYSTEM 나머지가 196개였다. 합법 공간이 이렇게 좁아서, 엉뚱한 주소로 뛰었을 때 불법 명령어 예외로 빨리 걸릴 확률이 높다.

## 6. 설계 체크리스트: 디코더

| 항목 | 규칙 |
|---|---|
| 레지스터 번호 | 해석하지 말고 고정 자리에서 배선. "실제로 읽는가" 신호는 따로 만든다 |
| 형식 | opcode만 보고 결정. funct3를 기다리지 않는다 |
| OP-IMM의 inst[30] | funct3=`101`일 때만 의미가 있다. `addi`에서는 즉시값 비트 |
| 시프트 inst[25] | RV32에서는 예약 인코딩. 흔한 선택은 불법 예외 |
| FENCE | funct3=`000`만 확인. rs1·rd·fm·pred·succ는 무시(예약값 포함) |
| FENCE.I | Zifencei가 없으면 불법(funct3 `001`) |
| SYSTEM | RV32I만이면 `0x00000073`·`0x00100073` 외 전부 불법. Zicsr·특권을 넣으면 표에 줄 추가 |
| 0과 모두 1 | 둘 다 불법. C 확장이 있으면 16비트 `0x0000`을 따로 잡아야 한다 |
| HINT | rd=x0 연산, pred/succ가 0인 FENCE. 예외 없이 통과 |
| 예약 인코딩 | 동작은 UNSPECIFIED. 대부분 불법 예외로 정한다(검증 기준과 맞출 것) |
| 대조 기준 | objdump는 FENCE 칸·6비트 shamt·특권 명령어에서 명세와 다르다. 기준으로 쓸 때 이 넷을 걸러야 한다 |

마지막 줄은 33~34편의 검증 이야기와 이어진다. 기준 모델이 틀릴 수도 있다는 것을 알고 있어야, 불일치가 났을 때 내 설계를 무작정 고치지 않는다.

## 정리

- 디코더는 레지스터 번호를 고정 자리에서 바로 보내고, opcode로 형식을, opcode·funct3·funct7로 제어 신호 한 줄을 고른다. RV32I는 주 opcode 32칸 중 11칸만 쓴다.
- 제어 신호는 RegWrite, ALU 입력 A·B 선택, ALU 연산, 메모리 읽기·쓰기와 크기, 쓰기 되돌림, 분기 조건, 점프 종류, 그리고 "rs1·rs2를 실제로 읽는가"다.
- 불법 조건은 16비트 칸, 없는 opcode·funct3·funct7(시프트 inst[25] 포함), SYSTEM 나머지 값이다. FENCE의 남는 칸과 HINT는 통과시켜야 한다.
- objdump와 174,608개를 대조해 165,923개가 같았고, 다른 8,685개는 FENCE 칸·특권 명령어·6비트 shamt·`unimp`의 네 가지로 모두 설명됐다.

**다음 글:** 16편에서는 디코더가 고른 형식을 받아 즉시값을 만드는 즉시값 생성기를 2편의 선택기 표에서 바로 회로로 옮기고, PC·명령어 메모리·레지스터 파일·ALU·데이터 메모리·쓰기 되돌림 선택기를 이어 단일 사이클 데이터패스를 완성한다. 실제로 컴파일한 프로그램 다섯 개를 그 모델로 돌려 QEMU와 명령어 단위로 맞춰 본다.

## 참고 문헌

- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. 1.5 명령어 길이 인코딩(0과 모두 1), 2.2 기본 명령어 형식(예약 인코딩은 UNSPECIFIED), 2.4 정수 연산(시프트 인코딩), 2.7 메모리 순서 명령어(FENCE의 무시할 칸), 2.8 ECALL·EBREAK, 2.9 HINT, 6장 Zifencei, 7장 Zicsr, 명령어 목록 장(RV32/64G Instruction Set Listings)의 기본 opcode 표. https://github.com/riscv/riscv-isa-manual
- D. Harris, S. Harris, *Digital Design and Computer Architecture: RISC-V Edition*, 2021. 7.3 단일 사이클 프로세서의 제어 장치(주 디코더·ALU 디코더)
- GNU Binutils 2.47 objdump 문서, RISC-V 디스어셈블러 옵션(`-M max`, `no-aliases`). https://sourceware.org/binutils/docs/binutils/objdump.html
