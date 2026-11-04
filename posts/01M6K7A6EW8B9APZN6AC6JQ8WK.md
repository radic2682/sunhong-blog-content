---
id: "01M6K7A6EW8B9APZN6AC6JQ8WK"
slug: "riscv-abi-calling-convention-elf"
title: "어셈블리와 ABI: 레지스터 이름, 호출 규약, 스택 프레임, 의사 명령어, ELF"
category: "01M6K19SKHFQ3SCEWS4D97Y48R"
tags: ["risc-v","abi","calling convention","elf","linker"]
summary: "RISC-V 6편. psABI 호출 규약과 스택 프레임을 실제 C 함수로 따라갔다. work() 프레임은 80바이트, 9번째 인자부터는 스택으로 갔다. ABI별 인자 위치와 재배치를 확인했고, 링커 완화로 .text가 800바이트에서 712바이트로 줄었다."
thumbnail: "/img/01M6K7A6EW8B9APZN6AC6JQ8WK/a026845dfc84.webp"
note: true
order: 6
created: "2026-11-04T21:07:39+09:00"
updated: "2026-11-04T21:07:39+09:00"
---
:::note
**이 글의 핵심 3줄**
- 하드웨어는 `x1`~`x31`을 똑같이 다루지만, 소프트웨어는 psABI라는 약속으로 이름과 역할을 정한다. 인자는 `a0`~`a7`, 반환은 `a0`·`a1`, `s0`~`s11`은 함수가 지켜 줘야 하고, `sp`는 언제나 16바이트 정렬이다. 실제 C 함수 `work()`의 프레임은 80바이트였고 `sp`는 `0x800043b0`이었다.
- 9번째 이후 인자는 스택 `0(sp)`·`4(sp)`로, 16바이트 구조체는 RV32에서 숨은 포인터(`a0`)로 돌아온다. `ilp32`·`ilp32f`·`ilp32d`·`lp64d`는 같은 `double` 인자를 정수 레지스터 쌍, 정수 레지스터 하나, FP 레지스터로 다르게 넘기고, 섞어 링크하면 링커가 거부했다.
- 객체 파일의 `call`·`la`는 빈칸(재배치)이 있는 `auipc` 쌍이다. 링커가 거리를 채우고, 가까우면 줄인다(완화). 같은 프로그램의 `.text`가 800바이트에서 712바이트로 줄었다. 실행·디스어셈블 결과 18개를 Python으로 검산해 모두 맞았다.
:::

2~5편은 명령어 하나하나를 봤다. 하지만 코어를 검증하다 보면 명령어 하나보다 "이 함수가 왜 여기서 `sp`를 80 줄이지", "`a0`에 들어 있는 이 값은 무엇이지"를 읽어야 하는 때가 더 많다. 시뮬레이션 파형과 디스어셈블을 나란히 놓고 버그를 쫓으려면 컴파일러가 레지스터와 스택을 어떻게 쓰는지 알아야 한다. 6편은 그 약속, 즉 어셈블리 문법과 ABI(응용 이진 인터페이스), 그리고 ELF 파일이 링크되는 과정을 다룬다.

실습은 RV32I(`-march=rv32i -mabi=ilp32 -O1`)를 기본으로 하고, ABI 비교에는 RV32IMAFD와 RV64IMAFD를 썼다. 실행은 QEMU 11.1.3 `virt`, 도구는 `riscv64-elf-gcc`·`objdump`·`readelf` 16.2다.

## 1. 레지스터 이름과 역할

2편에서 본 ABI 이름이 여기서 의미를 갖는다. RISC-V psABI(ELF psABI 명세)의 정수 호출 규약은 다음과 같다.

| 레지스터 | 이름 | 호출을 넘어 보존? | 쓰임 |
|---|---|---|---|
| `x0` | `zero` | — | 항상 0 |
| `x1` | `ra` | 아니오 | 복귀 주소 |
| `x2` | `sp` | 예 | 스택 포인터 |
| `x3`, `x4` | `gp`, `tp` | — | 전역·스레드 포인터 |
| `x5`~`x7`, `x28`~`x31` | `t0`~`t6` | 아니오 | 임시값 |
| `x8` | `s0` / `fp` | 예 | 보존값, 프레임 포인터 |
| `x9`, `x18`~`x27` | `s1`~`s11` | 예 | 보존값 |
| `x10`~`x17` | `a0`~`a7` | 아니오 | 인자, `a0`·`a1`은 반환값 |

"보존(callee-saved)"은 함수가 그 레지스터를 쓰려면 원래 값을 저장해 두었다가 돌아가기 전에 되살려야 한다는 뜻이다. "보존 안 함(caller-saved)"은 불린 함수가 마음대로 덮어써도 되니, 부른 쪽이 필요하면 저장해야 한다는 뜻이다. `ra`가 보존 안 함인 것이 눈에 띈다. 함수를 부르는 순간 `jal`이 `ra`를 덮어쓰기 때문에, 다른 함수를 부르는 함수는 자기 `ra`를 스택에 저장한다.

`gp`와 `tp`는 일반 할당 대상이 아니다. `gp`는 링커 완화에 쓰는 전역 포인터(5절), `tp`는 스레드 지역 저장소를 가리킨다. 레지스터 번호 배치에도 이유가 있다. 인자와 보존 레지스터가 `x8`~`x15`에 모여 있는데, C 확장의 16비트 명령어가 3비트로 부를 수 있는 레지스터가 바로 이 여덟 개다(11편).

## 2. 스택 프레임: 실제 함수를 따라가기

규칙부터 보자. 스택은 높은 주소에서 낮은 주소로 자란다. 함수에 들어올 때 `sp`는 **16바이트 정렬**이어야 한다(psABI). 그래서 함수가 잡는 프레임 크기도 16의 배수가 된다.

실험 함수는 이렇다. 반복문 안에서 다른 함수를 부르므로 `x`와 `i`가 호출을 넘어 살아야 하고, 지역 배열이 있고, 인자 10개짜리 함수를 부른다.

```c
int work(int x) {
  int buf[4];
  for (int i = 0; i < 4; i++)
    buf[i] = mkpair(x + i).b;
  return sum10(buf[0], buf[1], buf[2], buf[3],
               x, x, x, x, 1, 2);
}
```

GCC가 만든 프롤로그와 에필로그다.

```text
work:
  addi sp,sp,-80      # 프레임 80바이트
  sw   ra,76(sp)      # 복귀 주소
  sw   s0,72(sp)      # 쓸 보존 레지스터 5개
  sw   s1,68(sp)
  sw   s2,64(sp)
  sw   s3,60(sp)
  sw   s4,56(sp)
  ...                 # 본문
  lw   ra,76(sp)      # 에필로그: 거꾸로 되살림
  lw   s0,72(sp)
  ...
  addi sp,sp,80
  ret
```

![work()의 스택 프레임은 80바이트로, 위에서부터 ra·s0~s4 저장(24바이트), buf[4], 맨 아래 9·10번째 인자가 놓였고 sp는 0x800043b0이었다|full](/img/01M6K7A6EW8B9APZN6AC6JQ8WK/a026845dfc84.webp)

QEMU에서 실행해 실제 주소를 찍었다. `main`이 `work`를 부를 때 `sp`는 `0x80004400`, `work` 안에서는 `0x800043b0`(80 아래)이었다. 둘 다 16의 배수다. `buf`의 주소는 `0x800043c8`로 `sp + 24`였다. 컴파일러는 `x`와 반복 변수를 `s1`·`s2`에, `buf`의 시작과 끝 주소를 `s3`·`s4`에 두었다. `mkpair` 호출이 `a`·`t` 레지스터를 덮어써도 이 값들은 살아남는다. 그 대가로 프롤로그에서 다섯 개를 저장했다.

프레임 포인터를 켜면(`-fno-omit-frame-pointer`) 프롤로그에 `addi s0,sp,80`이 더해진다. `s0`(=`fp`)가 함수에 들어올 때의 `sp`를 가리키게 되어, 디버거가 `fp`를 따라 호출 사슬을 거슬러 올라갈 수 있다. 기본(`-O1` 이상)은 `fp`를 쓰지 않아 `s0`도 일반 보존 레지스터다.

## 3. 인자가 많을 때, 구조체를 돌려줄 때

**인자 9개 이상.** 정수 인자는 `a0`부터 `a7`까지 8개만 레지스터로 간다. 그 뒤는 부른 쪽이 자기 프레임 맨 아래, 즉 `0(sp)`부터 차례로 둔다. 위 그림의 맨 아래 두 칸이 그것이다. `work`는 `sw a5,0(sp)`(i=1)와 `sw a5,4(sp)`(j=2)를 하고 `sum10`을 불렀다. `sum10`은 프레임이 없는 함수라서 자기 `sp`가 곧 부른 쪽의 `sp`이고, `lw a5,0(sp)`·`lw a5,4(sp)`로 꺼냈다. `sum10(1, …, 10)`의 결과 `0x000a0924`는 Python으로 계산한 1+…+8 + (9≪8) + (10≪16) = 657,700과 같았다.

**구조체 반환.** psABI의 규칙은 크기로 갈린다.

| 반환하는 값 | RV32 (ilp32) | RV64 (lp64) |
|---|---|---|
| `struct {int a, b;}` 8바이트 | `a0`·`a1` | `a0` 하나에 묶음 |
| `struct {int v[4];}` 16바이트 | 숨은 포인터 | `a0`·`a1` |
| `struct {float x, y;}` | `a0`·`a1` | `a0` 하나에 묶음 |
| 같은 것, `ilp32f`·`lp64d` | `fa0`·`fa1` | `fa0`·`fa1` |

2×XLEN 이하는 레지스터 두 개로, 그보다 크면 **부른 쪽이 메모리를 마련하고 그 주소를 숨은 첫 인자로 `a0`에 넣는다.** RV32에서 16바이트는 2×XLEN(8바이트)을 넘는다. 그래서 `mkquad(100)`은 `sw a1,0(a0)`, `sw a4,4(a0)` …처럼 `a0`가 가리키는 곳에 쓰고, 원래 첫 인자 `x`는 `a1`으로 밀려났다. 부른 `main`은 `addi a0,sp,24`로 자기 프레임 안의 자리를 넘겼다. RV64에서는 같은 16바이트가 2×XLEN이라 `a0`·`a1`에 두 개씩 묶여 돌아왔다(`slli`·`or`로 32비트 둘을 합침). 실행 결과 `mkpair(40)`은 (40, 41), `mkquad(100).v[3]`은 103이었다.

## 4. ABI 이름: `ilp32`, `ilp32f`, `lp64d`

`-mabi` 이름은 두 부분이다. 앞부분은 C 타입 크기다. `ilp32`는 int·long·pointer가 32비트, `lp64`는 long·pointer가 64비트(int는 32비트)라는 뜻이다. 컴파일러로 확인하니 `ilp32`에서 `sizeof(long)`과 `sizeof(void *)`가 4, `lp64`에서 8이었다. 뒷글자는 부동소수점 인자를 어디로 넘기는지다. 없으면 정수 레지스터(소프트 플로트 ABI), `f`면 `float`까지 FP 레지스터, `d`면 `double`까지 FP 레지스터다.

같은 함수 `double mix(double x, float y, int z)`를 FPU가 있는 같은 하드웨어(`rv32imafd`, `rv64imafd`)용으로 ABI만 바꿔 컴파일했다.

| ABI | `x` (double) | `y` (float) | `z` (int) |
|---|---|---|---|
| `ilp32` | `a0`·`a1` | `a2` | `a3` |
| `ilp32f` | `a0`·`a1` | `fa0` | `a2` |
| `ilp32d` | `fa0` | `fa1` | `a0` |
| `lp64` | `a0` | `a1` | `a2` |
| `lp64d` | `fa0` | `fa1` | `a0` |

`mix`의 명령어 수는 `ilp32` 13개, `ilp32f` 12개, `ilp32d` 4개, `lp64` 7개, `lp64d` 4개였다.

`ilp32`에서는 FPU가 있어도 `double`이 정수 레지스터 두 개로 들어온다. 함수는 그것을 스택에 썼다가 `fld`로 다시 읽어 FP 레지스터에 옮기고, 결과도 같은 길로 돌려보낸다. 계산은 `fmadd.d` 하나인데 명령어가 13개가 됐다. `ilp32d`와 `lp64d`는 4개다. 번호는 정수와 FP가 따로 센다. `ilp32d`에서 `z`는 첫 정수 인자라 `a0`다.

ABI는 객체 파일 머리(ELF 헤더의 `e_flags`)에 기록된다. `readelf -h`로 보면 `ilp32`·`lp64`는 `0x0`, `ilp32f`는 `0x2, single-float ABI`, `ilp32d`·`lp64d`는 `0x4, double-float ABI`였다. 링커는 이 값이 다른 파일을 섞지 않는다. `ilp32` 객체와 `ilp32d` 객체를 함께 링크하자 다음 오류로 멈췄다.

```text
can't link double-float modules
with soft-float modules
```

같은 `double`이 한쪽은 `a0`·`a1`, 다른 쪽은 `fa0`에 있다고 믿으면 값이 엉뚱하게 전달되기 때문이다. 리눅스 배포판이 쓰는 표준 조합은 `rv64gc` + `lp64d`다.

## 5. 의사 명령어와 ELF 재배치

### 의사 명령어가 펼쳐지는 모양

어셈블리 파일에 의사 명령어를 모아 쓰고, 별칭을 끈 디스어셈블(`objdump -M no-aliases`)로 진짜 명령어를 봤다.

```text
nop            addi zero,zero,0
li  a0,0x12345678
               lui a0,0x12345 + addi a0,a0,1656
la  a1,buf     auipc a1,0 + addi a1,a1,0  (재배치)
mv  a3,a0      addi a3,a0,0
not a4,a0      xori a4,a0,-1
neg a5,a0      sub a5,zero,a0
seqz a6,a0     sltiu a6,a0,1
beqz a0,L      beq a0,zero,L
bgt a0,a1,L    blt a1,a0,L
ble a0,a1,L    bge a1,a0,L
j   L          jal zero,L
call g         auipc ra,0 + jalr ra,0(ra)
tail g         auipc t1,0 + jalr zero,0(t1)
jr  a0         jalr zero,0(a0)
jalr a0        jalr ra,0(a0)
ret            jalr zero,0(ra)
```

`tail`이 `t1`을 쓰는 것에 주목하자. 꼬리 호출은 `ra`를 건드리면 안 되므로(부른 쪽의 복귀 주소를 그대로 넘겨야 하므로) 임시 레지스터로 주소를 만든다. `la`는 위치 독립 코드(`-fPIC`)가 아니면 `lla`와 같은 `auipc` + `addi`다. PIC에서는 전역 오프셋 표(GOT)에서 주소를 읽는 `auipc` + `lw`가 된다.

### 재배치: 어셈블러가 남긴 빈칸

`auipc a1,0`의 즉시값이 0인 이유는 어셈블러가 `buf`의 최종 주소를 모르기 때문이다. 대신 **재배치(relocation)** 기록을 남겨 "링크할 때 이 칸을 이렇게 채워라"라고 알린다. `abi.c` 객체 파일의 재배치를 종류별로 셌다.

| 재배치 | medlow | medany | 채우는 것 |
|---|---|---|---|
| `R_RISCV_HI20` / `LO12_I` | 14 / 15 | — | 절대 주소의 위·아래 (`lui` + `addi`) |
| `R_RISCV_PCREL_HI20` / `LO12_I` | — | 15 / 15 | pc 기준 거리 (`auipc` + `addi`) |
| `R_RISCV_CALL_PLT` | 19 | 19 | `auipc` + `jalr` 호출 쌍 |
| `R_RISCV_BRANCH` | 4 | 4 | 분기 13비트 거리 |
| `R_RISCV_RELAX` | 48 | 49 | "줄여도 됨" 표시 |

코드 모델(`-mcmodel`)이 주소 만드는 방법을 고른다. `medlow`는 주소 0 근처 ±2 GiB의 절대 주소를 `lui`로, `medany`는 현재 pc에서 ±2 GiB를 `auipc`로 만든다. 3편의 `%hi`·`%lo`와 `%pcrel_hi`·`%pcrel_lo` 계산을 링커가 이 기록을 보고 대신 한다.

`PCREL_LO12_I`에는 함정이 있다. 이 재배치는 `g_small`이 아니라 짝이 되는 `auipc`의 위치(지역 라벨 `.L0`)를 가리킨다. 아래 12비트는 "`auipc`의 pc에서 목적지까지 거리"의 아래 부분이라, `addi`가 아니라 `auipc`의 주소가 기준이어야 하기 때문이다.

### 링커 완화: 닿으면 줄인다

`R_RISCV_RELAX`가 붙은 쌍은 링커가 더 짧게 바꿔도 된다. 실제 거리를 알게 된 링커는 `call`의 목적지가 ±1 MiB 안이면 `jal` 하나로, 주소가 `gp`에서 ±2 KiB 안이면 `auipc`를 지우고 `gp` 기준 접근 하나로 바꾼다.

![객체 파일의 auipc+jalr 호출은 링크 후 jal 하나로, auipc+lw 전역 변수 읽기는 lw a1,-1791(gp) 하나로 줄어 .text가 800바이트에서 712바이트가 됐다|full](/img/01M6K7A6EW8B9APZN6AC6JQ8WK/78af1592417f.webp)

`gp` 완화를 보려면 링커 스크립트에 `__global_pointer$`가 있고 시작 코드가 `gp`를 설정해야 한다. 공용 시작 코드에는 이것이 없어서, 이 편의 폴더에 GNU ld 기본 스크립트와 같은 식으로 `__global_pointer$`를 더한 링커 스크립트와 시작 코드를 따로 만들었다. 시작 코드의 `la gp, __global_pointer$`는 `.option norelax`로 감쌌다. `gp`를 만드는 명령어를 `gp` 기준으로 줄이면 자기 자신을 참조하게 되기 때문이다.

`medany`로 빌드해 `--no-relax`와 기본(완화)을 비교했다. `.text`는 800바이트에서 712바이트로, `auipc`는 37개에서 15개로 줄었고, `jal ra`는 0개에서 20개가 됐다.

호출 20개가 모두 `jal`이 되었고, 작은 전역 변수 `g_small`의 읽기와 주소 계산 두 곳이 `lw a1,-1791(gp)`·`addi a1,gp,-1791`이 되었다. 줄어든 88바이트가 정확히 22개 명령어(4바이트씩)다. `medlow`로 빌드해도 804바이트에서 720바이트로 비슷하게 줄었다. 256바이트짜리 `g_big`은 `gp`에서 2,047바이트 떨어져 숫자로는 범위 안이었지만 `auipc`가 남았다. 링커가 정렬 여유를 두고 판단하는 것으로 보인다(추정).

완화가 하드웨어 설계자에게 중요한 이유가 있다. 소스나 객체 파일의 디스어셈블과 실제로 메모리에 올라간 ELF의 명령어가 다르다. 시뮬레이션 로그와 맞춰 볼 때는 반드시 **링크된 ELF**를 디스어셈블해야 한다.

### 섹션

링크된 ELF의 섹션은 `.text`(코드, `0x80000000`), `.rodata`(문자열 상수), `.data`(초깃값 있는 전역 256바이트), `.sdata`(작은 전역 4바이트), 그리고 메모리에 올라가지 않는 `.riscv.attributes`·`.symtab` 등이었다. `.riscv.attributes`에는 1편에서 본 정식 ISA 이름(`Tag_RISCV_arch`)이 들어 있다. `.sdata`·`.sbss`는 작은 전역 변수를 `gp` 근처에 모아 위의 완화가 잘 되게 하려는 섹션이다.

## 6. 실습: 실행과 디스어셈블 검산

`check.py`가 빌드, QEMU 실행, objdump·readelf 수집을 한 번에 한다. 링크된 ELF의 기계어에서 주소를 직접 복원하는 검사가 핵심이다. `auipc`와 바로 뒤의 `addi`·`lw`를 찾아 `pc + (imm20 ≪ 12) + imm12`를 계산하고, `gp` 기준 접근은 `gp + imm12`를 계산해 심볼 표의 주소와 맞는지 본다.

```python
for (a, w, _), (a2, w2, _) in zip(I, I[1:]):
    # auipc rd + (addi|lw) rd: pc 기준 주소 복원
    if w & 0x7f == 0x17 and a2 == a + 4 \
       and (w2 >> 15) & 31 == (w >> 7) & 31:
        got.append(a + sx(w & 0xfffff000, 32)
                   + sx(w2 >> 20, 12))
    # rs1 == gp(x3): gp 기준 주소 복원
    if w & 0x7f in (0x13, 0x03) and (w >> 15) & 31 == 3:
        got.append(gp + sx(w >> 20, 12))
```

| 시험 | 개수 | 일치 |
|---|---|---|
| 실행 값 (인자 10개, 구조체 반환, `work`) | 5 | 5 |
| `sp`·프레임 16바이트 정렬, `buf` 위치 | 4 | 4 |
| QEMU가 찍은 전역 주소 = 심볼 표 | 2 | 2 |
| 기계어에서 복원한 주소 (완화 켬·끔) | 2 | 2 |
| ABI: 타입 크기, 숨은 포인터, 링크 거부, `fp` | 5 | 5 |

모두 18개, 전부 맞았다. 기계어 복원 검사는 링커가 `%pcrel_hi`·`%pcrel_lo`의 +1 보정(3편)과 `gp` 완화를 제대로 했다는 뜻이기도 하다.

## 7. 설계 체크리스트: 하드웨어 쪽에서 ABI를 쓸 때

ABI는 소프트웨어의 약속이지만, 코어를 검증하고 디버그하는 사람은 매일 이것을 읽는다.

| 항목 | 규칙 |
|---|---|
| 레지스터 파일 | 하드웨어는 `x1`~`x31`을 똑같이 다룬다. ABI 이름은 디스어셈블러의 표기일 뿐 |
| 복귀 주소 | `ra`는 `jal`/`jalr`가 쓴 pc+4. 예측기 RAS 힌트는 `x1`·`x5` (4편) |
| 스택 정렬 | 함수 진입 시 `sp` 16바이트 정렬. 시뮬레이션에서 어긋나면 스택 손상 신호 |
| 트랩 핸들러 | 인터럽트는 아무 곳에서나 들어오므로 보존 안 함 레지스터까지 모두 저장해야 한다 |
| `gp` 설정 | 부팅 코드는 `.option norelax`로 `gp`를 먼저 잡는다. 안 잡으면 완화된 전역 접근이 엉뚱한 주소로 |
| 링크된 ELF | 완화 때문에 `.o`와 명령어가 다르다. 로그와 대조는 링크된 ELF로 |
| ABI 확인 | `readelf -h`의 Flags(`0x0`·`0x2`·`0x4`, RVC면 `0x1` 추가)와 `-A`의 `Tag_RISCV_arch` |
| 별칭 끄기 | `objdump -M no-aliases`로 보면 디코더가 실제로 받는 명령어가 보인다 |
| 환경 호출 | SBI는 `a7`에 확장 ID, `a6`에 함수 ID, `a0`~`a5` 인자, `a0`·`a1` 반환 (36편) |
| `tp`·`gp` | 컴파일러가 일반 값으로 쓰지 않는다. 이 둘이 바뀌는 곳은 런타임 코드뿐 |

트랩 핸들러 줄은 4편의 실습과 이어진다. 4편의 핸들러는 GCC의 `interrupt("machine")` 속성으로 만들었는데, 디스어셈블해 보니 프롤로그에서 `ra`, `t0`~`t6`, `a0`~`a7`, `s0`까지 17개를 저장했다. 일반 함수라면 `t`·`a` 레지스터는 저장하지 않는다. 트랩은 함수 호출 경계가 아닌 곳에서 끼어들어, 끼어든 코드가 그 레지스터에 값을 담고 있을 수 있기 때문이다.

## 정리

- `a0`~`a7`은 인자(`a0`·`a1`은 반환), `s0`~`s11`과 `sp`는 보존, `t`·`a`·`ra`는 보존 안 함이다. 실제 `work()`는 `ra`와 `s0`~`s4`를 저장하는 80바이트 프레임을 잡았고 `sp`는 16의 배수였다.
- 9번째 이후 인자는 부른 쪽의 `0(sp)`부터, 2×XLEN보다 큰 구조체는 부른 쪽이 준 메모리(`a0`)로 돌아온다. 같은 16바이트 구조체가 RV32에서는 메모리, RV64에서는 `a0`·`a1`이었다.
- ABI 이름은 타입 크기(`ilp32`·`lp64`)와 FP 인자 레지스터(`f`·`d`)다. 같은 `mix()`가 `ilp32` 13개, `ilp32d` 4개 명령어였고, 서로 다른 FP ABI는 링크가 거부됐다.
- 객체 파일의 `call`·`la`는 재배치가 붙은 `auipc` 쌍이다. 링커는 거리를 채우고, 닿으면 `jal`이나 `gp` 기준 접근으로 줄인다. `.text`가 800바이트에서 712바이트가 됐다.

**다음 글:** 7편에서는 M 확장을 본다. `mul`·`mulh`·`mulhu`·`mulhsu`와 `div`·`rem` 계열이 무엇을 계산하는지, 0으로 나누면 예외 없이 어떤 값을 돌려주는지, 가장 작은 음수를 −1로 나누는 오버플로를 어떻게 정해 두었는지를 QEMU에서 실행해 확인한다. 1편에서 `utoa10`이 `__udivsi3`를 부르던 것이 M 확장으로 어떻게 바뀌는지도 다시 본다.

## 참고 문헌

- RISC-V ELF psABI Specification (정수·FP 호출 규약, 구조체 전달, 스택 정렬, ELF e_flags, 재배치와 완화). https://github.com/riscv-non-isa/riscv-elf-psabi-doc
- RISC-V Assembly Programmer's Manual (의사 명령어, `.option norelax`, `%pcrel_hi`·`%pcrel_lo`). https://github.com/riscv-non-isa/riscv-asm-manual
- The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture, Version 20240411. 2.1 프로그래머 모델, 2.5 제어 이동(`x1`·`x5` 링크 레지스터)
- RISC-V Supervisor Binary Interface Specification (SBI 호출 규약). https://github.com/riscv-non-isa/riscv-sbi-doc
- D. Patterson, A. Waterman, *The RISC-V Reader: An Open Architecture Atlas*, 2017. 3장 RISC-V 어셈블리 언어
- GCC RISC-V 옵션(`-mabi`, `-mcmodel`, `-mno-relax`). https://gcc.gnu.org/onlinedocs/gcc/RISC-V-Options.html
