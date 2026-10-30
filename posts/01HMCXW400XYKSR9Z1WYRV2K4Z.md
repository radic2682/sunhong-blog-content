---
id: "01HMCXW400XYKSR9Z1WYRV2K4Z"
slug: "cortex-m-nvic-interrupts"
title: "Cortex-M NVIC 인터럽트 처리: 벡터 테이블과 예외 우선순위"
category: "01M651PC003HYJYHCQPHT4KAWF"
tags: []
summary: "Polling vs. Interrupt, Pre-emption, Interrupt Controller"
thumbnail: "/img/01HMCXW400XYKSR9Z1WYRV2K4Z/b004412d3647.webp"
note: true
created: "2024-01-18T09:00:00+09:00"
updated: "2026-04-20T01:45:26+09:00"
---
## 8장 Interrupt Mechanism 개요

**Interrupt**는 프로세서의 정상 실행 흐름을 중단시키고 특정 handler routine을 실행한 뒤 복귀하는 메커니즘. 주변 장치, 타이머, fault 등 비동기 이벤트에 즉각 반응하기 위해 필수적이다.

## Interrupt vs Exception

- **Interrupt**: 비동기 (외부 이벤트, 주변 장치에서 발생)
- **Exception**: 동기 (명령어 실행 중 발생 — SVC, fault)
- ARM에서는 둘 다 **Exception**으로 통일 처리

## NVIC (Nested Vectored Interrupt Controller)

**Cortex-M의 핵심**: NVIC이 interrupt 처리.

### 주요 기능

- **Vectored interrupts**: 각 IRQ마다 전용 vector (자동 분기)
- **Nested interrupts**: 높은 우선순위 IRQ가 낮은 것을 선점 (preemption)
- **Priority levels**: 프로그래머가 설정 가능 (M0: 4 levels, M3: 8~256)
- **Dynamic priority**: runtime에 변경 가능
- **Masking**: PRIMASK, FAULTMASK, BASEPRI

## Vector Table

메모리 0x00000000에 위치. 각 exception의 handler address 저장:

| Offset | Vector | Exception |
|--------|--------|-----------|
| 0x00 | Initial SP | — |
| 0x04 | Reset | Reset |
| 0x08 | NMI | Non-maskable |
| 0x0C | HardFault | Fault escalation |
| ... | ... | ... |
| 0x2C | SVCall | SVC instruction |
| 0x38 | PendSV | Pended Service |
| 0x3C | SysTick | System timer |
| 0x40 ~ | IRQ0 ~ IRQn | External IRQs |

Reset 시 Initial SP 값이 R13에 로드되고, Reset vector로 PC가 점프.

![NVIC 및 Vector Table|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/b004412d3647.webp)

## Interrupt 처리 순서

1. **External event** (예: UART RX) → Peripheral이 IRQ signal assert
2. **NVIC**이 IRQ를 감지, priority 확인
3. 현재 실행 코드보다 우선순위가 높으면 **preempt**
4. **Stacking**: xPSR, PC, LR, R12, R3, R2, R1, R0을 stack에 push (8개 word)
5. **Vector fetch**: Vector table에서 handler address 읽음
6. **Handler 실행** (Handler mode, privileged)
7. **Exit**: `BX LR` 또는 `POP {..., PC}` — EXC_RETURN 값 로드
8. **Unstacking**: Stack frame에서 자동 복원, 원래 코드로 복귀

## Exception Priority

- 고정 우선순위: Reset (-3) > NMI (-2) > HardFault (-1)
- 구성 가능: 나머지 모든 exception
- **낮은 숫자 = 높은 우선순위**

Cortex-M0의 priority field는 2-bit → 4 level. Cortex-M3는 3~8 bit.

## Preemption & Nested

```
Main → IRQ_A handler 실행
      → 더 높은 우선순위 IRQ_B 발생
      → IRQ_A 중단 (preempted), 새 stack frame push
      → IRQ_B handler 실행
      → IRQ_B 완료 → IRQ_A로 복귀
      → IRQ_A 완료 → Main으로 복귀
```

Stack에 frame이 쌓이므로 stack 깊이 충분히 확보 필요.

## Tail-Chaining

Cortex-M의 최적화: 연속된 IRQ 처리 시 **unstack + restack 생략**:

```
Main → IRQ_A → (끝나자마자 IRQ_B pending)
                → Stack pop 생략하고 바로 IRQ_B handler 호출
                → IRQ_B 끝나면 한 번만 unstack → Main 복귀
```

→ Latency 감소, 전력 절약.

## Late Arrival

IRQ_A 처리 중 stacking 도중에 더 높은 IRQ_B 발생 → stacking 완료되자마자 IRQ_B로 분기 (IRQ_A는 pending).

## Interrupt Masking

### PRIMASK

- bit 0: 1이면 configurable exception 모두 mask (NMI, HardFault 제외)
- `CPSID I` / `CPSIE I`로 설정

### FAULTMASK (Cortex-M3 이상)

- 1이면 NMI 제외 모두 mask

### BASEPRI (Cortex-M3 이상)

- 이 값보다 낮은 우선순위 IRQ mask

![Interrupt 처리 및 Masking|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/3d1733422057.webp)

## NVIC Register

- **ISER** (Interrupt Set-Enable Register)
- **ICER** (Interrupt Clear-Enable)
- **ISPR** (Set Pending)
- **ICPR** (Clear Pending)
- **IP** (Interrupt Priority) — 각 IRQ의 priority 설정

예: IRQ 5 활성화:
```c
NVIC->ISER[0] = (1 << 5);
```

## Exception Handler 작성

C 코드로 handler 작성 가능. Compiler가 AAPCS (ARM calling convention) 따라 자동 처리:

```c
void USART1_IRQHandler(void) {
    if (USART1->SR & USART_SR_RXNE) {
        uint8_t data = USART1->DR;
        // 처리
    }
}
```

Vector table의 0x4C (IRQ 3 가정)에 handler address가 등록되어 있어야 함. Startup 코드(linker script)에서 설정.

![Interrupt Handler 예|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/ccbd3f4fb47b.webp)

## 정리

| 개념 | 설명 |
|------|------|
| **NVIC** | Nested Vectored Interrupt Controller |
| **Vector Table** | 0x00000000부터 각 exception handler 주소 |
| **Priority** | 낮은 숫자 = 높은 우선순위 |
| **Preemption** | 높은 우선순위 IRQ가 실행 중인 handler 선점 |
| **Tail-chaining** | 연속 IRQ 처리 시 stacking 생략 |
| **Late-arriving** | Stacking 중 새 IRQ 도착 시 분기 |
| **Masking** | PRIMASK / FAULTMASK / BASEPRI |

NVIC은 **실시간 응답성과 우선순위 기반 선점**을 제공해 Cortex-M이 실시간 임베디드 시스템의 표준이 되게 한 핵심이다.

![Interrupt 종합|full](/img/01HMCXW400XYKSR9Z1WYRV2K4Z/510fdbf101a4.webp)
