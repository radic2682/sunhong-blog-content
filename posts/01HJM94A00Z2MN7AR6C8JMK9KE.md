---
id: "01HJM94A00Z2MN7AR6C8JMK9KE"
slug: "cortex-m0-soc-robot-control"
title: "Cortex-M0 SoC 설계: AHB-Lite 버스로 만든 다기능 로봇 제어 시스템 (Basys3)"
category: "01M651PC00H3J4W7JXTZ32PR45"
tags: ["verilog","vivado"]
summary: "Basys3 보드에 Cortex-M0와 AHB-Lite 주변장치를 묶은 SoC를 설계했다. UART로 작은 로봇을 제어하고 업 카운터, 스톱워치, 난수·곱셈기, 덧셈·뺄셈기를 실행하는 다기능 시스템의 HW·SW 구성과 결과를 담았다."
thumbnail: "/img/01HJM94A00Z2MN7AR6C8JMK9KE/f101db64ce35.webp"
note: false
created: "2023-12-27T09:00:00+09:00"
updated: "2026-09-15T02:10:29+09:00"
---
![Cortex-M0 SOC Design _Multi Function mini Robot|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/f101db64ce35.webp)

## 1. 설계 조건

팀별로 원하는 다양한 주제를 선택하여 Digilent 사의 Basys3 보드를 이용한 시스템을 구현한다. AHB-Lite Verilog 모듈을 결합하여 팀별만의 컴퓨터 시스템을 작성한다. 임베디드 시스템에 어셈블리어 혹은 C 언어로 프로그램 또한 작성하여 컴퓨터 시스템이 수행되어야 한다. 해당 입력은 슬라이딩 스위치, 버튼, 그리고 조도 센서 및 본인이 원하는 모듈을 결합하면 다양한 입력이 가능하며 출력 또한 LED 출력 및 7-segment 출력, VGA 출력, Analog 출력 및 본인이 원하는 모듈을 결합하면 다양한 출력 형태가 가능하다.

## 2. 설계 주제

이번 프로젝트의 주요 주제로는 크게 두 가지로 나뉘며 다기능 카운터, 계산기와 UART 통신을 통한 작은 로봇 제어로 이루어져있다. 우선 아래의 그림 1은 프로젝트에 사용된 Basys3 보드의 핀과 주변장치에 대한 설명이며 보드에 내장되어있는 LED 15개, SW개, KEY 스위치 5개와 Pmod JA, JB를 사용하여 프로젝트를 구성하였다. 또한 Pmod JA는 UART 통신을 하여 외부의 다른 프로세서와 통신을 할 수 있도록 만들어졌다. Pmod JB는 GPIO로 사용하여 JB1-4는 Input, JB7-10은 Output으로 사용할 수 있도록 구성을 했으며, 실제로는 JB1에 LED를 연결하여 JB7 핀만 사용하도록 구성하였다.

![설계 주제|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/c023005a9e58.webp)

<center>그림 1 Basys3에 내장된 장치를 통한 구현 기능 및 Pmod를 사용한 주변 장치 제어</center>

또한 위에서 언급하였듯이 이번 프로젝트에서는 UART 통신을 하여 외부의 프로세서와 통신 및 제어를 하게 되며 32bit, 3.3V 프로세서인 ESP32-WROOM-32D을 사용하였다. ESP32-WROOM-32D에 2개의 서보모터를 부착하여 간단한 로봇을 제작하였으며 이를 Basys3에서 데이터를 받아 동작을 제어하도록 제작하였으며 아래는 최종적으로 Basys3에 ESP32-WROOM-32D 및 다른 주변 장치를 연결한 회로이다.

![설계 주제 (2)|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/307b7d4b4038.webp)

<center>그림 2 Basys3에 Pmod를 통해 연결한 최종 프로젝트 회로도</center>

## 3. 기능

설계된 결과물은 크게 다기능 카운터, 계산기와 UART 통신을 통한 작은 로봇 제어의 두 가지 기능으로 나뉜다. 이에 따라 두 가지 기능으로 나눠서 설명하였다.

### 1) UART 통신을 통한 작은 로봇 제어

ESP32-WROOM-32D에 연결된 2개의 서보모터로 구성된 로봇을 Basys3가 제어하기 위해 UART 통신을 사용하였고 이는 Pmod의 JA2, JA3 Pin을 통해 UART 통신을 구성하였다. 아래의 그림 3의 Basys3의 Interface Specification 참고 매뉴얼을 보면 알 수 있듯이 JA2를 TX, JA3를 RX로 연결하여 UART 통신을 할 수 있었다.

![UART 통신을 통한 작은 로봇 제어|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/29b90d6ac68e.webp)

<center>그림 3 래퍼런스 매뉴얼의 Pmod 자료</center>

ESP32-WROOM-32D는 Basys3에서 로봇의 동작 여부를 전달받으면 이에 따라 ESP32_Servo.h를 통해 연결된 서보 모터를 제어하여 로봇을 동작하게 된다. 이때, Basys3에 내장되어 있는 SW4를 로봇 제어 신호로 사용하여 Switch의 활성화 여부에 따라 로봇의 동작 여부가 결정되게 된다.

![UART 통신을 통한 작은 로봇 제어 (2)|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/2b94f61af679.webp)

그림 4 180도 회전의 서보 모터를 사용하여 구성한 소형 로봇

위의 그림 4를 참고하여 서보 모터 로봇을 구성하였으며 ESP32-WROOM-32D의 전원이 인가되면 앞다리와 뒷다리의 서보 모터가 회전 가능한 각도인 180도 중 절반인 90도로 조절되어 중립 상태를 유지하게 된다. 이후 어느 정도의 안정될 시간을 거친 후, 앞다리 역할을 하는 서보 모터가 왼쪽으로 20도 회전한 후 다시 오른쪽으로 40도 회전하여 양방향으로 총 40도로 움직인다. 뒷다리도 앞다리와 동일하게 동작하며 순차적으로 서보 모터를 동작하여 앞으로 진행할 수 있게 된다.

### 2) 다기능 카운터, 계산기

SW[0]~SW[4]의 입력은 시스템의 mode를 결정한다.

SW[0]이 선택되면 업카운터로 동작을 하며 HW Timer를 통해 값이 하나씩 증가한다. 이때, up-key를 사용하면 Timer를 초기화할 수 있다.

SW[1]이 선택되면 스탑워치로 동작을 하며 HW Timer를 통해 값이 하나씩 감소한다. left-key를 통해 값을 십의 자리 단위로 증가시킬 수 있고, right-key를 통해 값을 십의 자리 단위로 감소시킬 수 있다. 또한, down-key를 통해 스탑워치를 시작하고 일시정지할 수 있다. 이때, up-key를 사용하면 스탑워치를 초기화할 수 있다. 스탑워치가 카운트다운을 해서 0에 도달하면 JB7핀을 통해 LED가 점멸하게 된다.

SW[2]이 선택되면 랜덤 숫자 발생기/곱셈기로 동작을 하며 4자리의 난수를 받아와 디스플레이시켜주거나 SW[9:5]와 SW[15:10]을 곱해서 디스플레이 해준다. 이때, 랜덤 숫자 발생기, 곱셈기 모드를 변경하는 방법은 middle-key를 누를 때마다 변경된다.

SW[3]이 선택되면 덧셈기/뺄셈기로 동작을 하며 SW[9:5]와 SW[15:10]을 더하거나 빼서 디스플레이 해준다. 이때, 덧셈과 뺄셈 모드를 변경하는 방법은 middle-key를 누를 때마다 변경된다.

SW[4]가 선택되면 로봇컨트롤로 사용되며 스위치가 ON이 되면 로봇이 전진한다.

## 4. 설계 과정

### 1) H/W 구성

#### Basys_Master.xdc

```
# Switches
set_property PACKAGE_PIN V17 [get_ports {sw[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {sw[0]}]
                          ː
set_property PACKAGE_PIN T1 [get_ports {sw[14]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {sw[14]}]
set_property PACKAGE_PIN R2 [get_ports RESET]					
	set_property IOSTANDARD LVCMOS33 [get_ports RESET]
```

```
# LEDs
set_property PACKAGE_PIN U16 [get_ports {LED[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {LED[0]}]
                          ː
set_property PACKAGE_PIN L1 [get_ports {LED[15]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {LED[15]}]
```

```
#7 segment display
set_property PACKAGE_PIN W7 [get_ports {seg[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {seg[0]}]
                          ː
set_property PACKAGE_PIN U7 [get_ports {seg[6]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {seg[6]}]

set_property PACKAGE_PIN V7 [get_ports dp]							
	set_property IOSTANDARD LVCMOS33 [get_ports dp]

set_property PACKAGE_PIN U2 [get_ports {an[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {an[0]}]
                          ː
set_property PACKAGE_PIN W4 [get_ports {an[3]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {an[3]}]
```

```
#Buttons
set_property PACKAGE_PIN T18 [get_ports KEY[0]]						
	set_property IOSTANDARD LVCMOS33 [get_ports KEY[0]]
                          ː
 set_property PACKAGE_PIN U18 [get_ports KEY[4]]						
	set_property IOSTANDARD LVCMOS33 [get_ports KEY[4]]
```

```
#Pmod Header JA
set_property PACKAGE_PIN L2 [get_ports JATx]					
	set_property IOSTANDARD LVCMOS33 [get_ports JATx]
set_property PACKAGE_PIN J2 [get_ports JARx]					
	set_property IOSTANDARD LVCMOS33 [get_ports JARx]
```

```
#Pmod Header JB
set_property PACKAGE_PIN A14 [get_ports {JBI[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {JBI[0]}]
                          ː
set_property PACKAGE_PIN B16 [get_ports {JBI[3]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {JBI[3]}]

set_property PACKAGE_PIN A15 [get_ports {JBO[0]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {JBO[0]}]
                          ː
set_property PACKAGE_PIN C16 [get_ports {JBO[3]}]					
	set_property IOSTANDARD LVCMOS33 [get_ports {JBO[3]}]
```

프로젝트에서 사용할 Switch와 LED, 7Segment, KEY, JA Pmod, JB Pmod를 보드의 핀과 연결해주기 위해 xdc파일을 다음과 같이 수정하였다. 주석 처리가 되어 있는 핀을 활성화함으로써 핀을 사용한다. 또한 JA Pmod에서 JA2, JA3 핀은 각각 JATx, JARx로 수정하여 UART 통신에 사용한다.

#### 사용 모듈

![사용 모듈|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/6afc70f6ffaf.webp)

<center>그림 5 사용 모듈</center>

#### 버스 모듈 정의

```verilog
module AHBLITE_SYS(    
    //CLOCKS & RESET
    input  wire CLK,
    input  wire RESET,
    
    //TO BOARD LEDs
    output 	wire	[15:0] LED,
    
    // SW
    input   wire    [14:0]  sw,
    
    // KEY
    input   wire  [4:0]  KEY,
    
    // SEG
    output  wire  [6:0]  seg,
    output  wire  [3:0]  an,
    output  wire         dp,
    
    // UART
    input wire           JARx,
    output wire          JATx,

    // GPIO JB
    input wire  [3:0]   JBI,
    output wire  [3:0]   JBO
);
```

Basys3 보드의 UART 통신을 구성하기 위해 버스 모듈에서 UART의 input, output을 위와 같이 정의해준다. 이는 xdc파일에서 수정한 Pmod 포트들과 연결이 되고 서로 이름이 같아야 한다.

#### 내부 신호 정의

```verilog
//SELECT SIGNALS

wire [3:0] MUX_SEL;
wire HSEL_MEM;
wire HSEL_GPIO;
wire HSEL_KEY;
wire HSEL_SEG;
wire HSEL_TIMER;
wire HSEL_UART;
wire HSEL_GPIO_JB;
```

위는 SELECT SIGNALS을 정의한 코드이며 위와 같이 wire를 SLAVE READ DATA, SLAVE HREADYOUT 또한 정의하였고 DCD, MUX에 연결해주었다. 이는 항상 하던 과정으로 생략한다.

#### 인터럽트 신호 정의

```verilog
wire          KEY_IRQ;
wire          TIMER_IRQ;
wire          UART_IRQ;

//CM0-DS INTERRUPT SIGNALS  
assign 			  IRQ = {13'b0000_0000_0000_000, KEY_IRQ, UART_IRQ, TIMER_IRQ};
assign 			  LED[15] = LOCKUP;

assign 	      HRESETn = ~RESET;   
```

인터럽트 신호는 Key, Timer, UART로부터 발생하며 데이터를 처리하기 때문에 이를 16비트 IRQ로 정의하여 인터럽트 신호를 받을 수 있도록 하였다.

#### 인터럽트 추가된 모듈 설명

```verilog
AHBGPIO uAHBGPIO(
	//AHBLITE Signals
	.HSEL(HSEL_GPIO),
                          ː
	
	.HRDATA(HRDATA_GPIO), 
	.HREADYOUT(HREADYOUT_GPIO),
	
    .GPIOIN({1'b0, sw[14:0]}),
    .GPIOOUT(LED[14:0])
);
```

Basys3에 내장되어 있는 SW와 LED를 사용하기 위해 GPIO 모듈을 정의하였다.

```verilog
AHBGPIO uAHBGPIOJB(
	//AHBLITE Signals
	.HSEL(HSEL_GPIO_JB),
                          ː
	
	.HRDATA(HRDATA_GPIO_JB), 
	.HREADYOUT(HREADYOUT_GPIO_JB),
	
    .GPIOIN({12'b0, JBI[3:0]}),
    .GPIOOUT(JBO[3:0])
);
```

JB Pmod의 핀을 GPIO핀으로 사용하여 외부 장치에 입력을 받고 출력을 보낼 수 있도록 설정하였다. 각 입력과 출력은 4bit 신호로 사용하였으며, 이번 프로젝트에서는 출력 1bit만 사용한다.

```verilog
AHB2KEY2 uAHB2KEY2 (
	//AHBLITE Signals
	.HSEL(HSEL_KEY),
                          ː
	
	.HRDATA(HRDATA_KEY),   
	.HREADYOUT(HREADYOUT_KEY),
	//Sideband Signals
	.KEY_IRQ(KEY_IRQ),
	.KEY(KEY)
);
```

KEY를 사용하기 위해서 모듈을 버스에 붙였다. KEY에서 발생한 인터럽트 신호를 받기 위해서 인터럽트 신호도 연결되어 있다.

```verilog
//AHBLite Slave

AHBTIMER uAHBTIMER (

//AHBLITE Signals

.HSEL(HSEL_TIMER),

ː

.HRDATA(HRDATA_TIMER),

.HREADYOUT(HREADYOUT_TIMER),

//Sideband Signals

.timer_irq(TIMER_IRQ)

);
```

시스템에서 사용하는 업카운터와 스탑워치를 사용하기 위해서 타이머 모듈을 버스에 붙였다. 또한, 인터럽트 처리를 위한 신호가 연결되어 있다.

```verilog
AHBUART uAHBUART (
	//AHBLITE Signals
	.HSEL(HSEL_UART),
                          ː
	
	.HRDATA(HRDATA_UART),   
	.HREADYOUT(HREADYOUT_UART),
	//Sideband Signals
	.JARx(JARx),
	.JATx(JATx),
	.uart_irq(UART_IRQ)
);
```

UART를 통해 외부의 로봇과 통신을 하기 위해서 버스에 UART 모듈을 붙였다. 신호는 Rx, Tx를 사용하며 이는 JA Pmod와 연결이 되어 있다. 또한, 인터럽트 처리를 위한 신호가 연결되어 있다.

```verilog
module BAUDGEN
(
  input wire clk,
  input wire resetn,
  output wire baudtick
);


reg [21:0] count_reg;
wire [21:0] count_next;

//Counter
always @ (posedge clk, negedge resetn)
  begin
    if(!resetn)
      count_reg <= 0;
    else
      count_reg <= count_next;
end


//Baudrate  = 19200 = 50Mhz/(163*16)
assign count_next = ((count_reg == 162) ? 0 : count_reg + 1'b1);

assign baudtick = ((count_reg == 162) ? 1'b1 : 1'b0);

endmodule
```

또한 AHBUART 하위 파일 중 BAUDGEN 모듈을 살펴보면 UART 통신에 있어 중요한 통신 속도가 기재되어 있는 것을 확인할 수 있다. 코드 중간에 Baudrate에 주석이 존재하며 여기서 통신 속도를 19200으로 설정하였다는 안내가 있어 이를 바탕으로 ESP32-WROOM-32D과의 원활한 UART 통신이 가능하다.

#### KEY 모듈 수정

```verilog
module AHB2KEY2(
                          ː
  input wire [4:0] KEY
    );

                          ː

  reg  [4:0] keyout;        //key output

                          ː

  // sequential part
  always @(posedge HCLK or negedge HRESETn)
  begin
	 if(!HRESETn)
   begin
     keyout <= 5'b0;
     key_pushed <= 1'b0;
   end
   else if ((key_pushed == 1'b0) && (KEY != 5'b0))
   begin
     keyout <= KEY;
     key_pushed <= 1'b1; 
   end
   else if (KEY == 5'b0) 
   begin
     keyout <= 5'b0;
     key_pushed <= 1'b0; 
   end
  end

  // sequential part
  always @(posedge HCLK or negedge HRESETn)
  begin
	 if(!HRESETn)
     KEY_IRQ <= 1'b0;
   else if ((key_pushed == 1'b0) && (KEY != 5'b0))
     KEY_IRQ <= 1'b1;
   else if (rd)
     KEY_IRQ <= 1'b0;
  end

                          ː
endmodule
```

기존 KEY 모듈이 4개의 key만 사용할 수 있었기에, 5개를 사용하기 위해 모듈을 위와 같이 수정하였다.

### 2) S/W 구성

#### cm0dsasm.s

```
 ː      

;Reset Handler
Reset_Handler   PROC
                GLOBAL Reset_Handler
                ENTRY
        IMPORT  __main
                LDR     R0, =__main               
                BX      R0                        ;Branch to __main
                ENDP
                      ː  
```

프로젝트에서 S/W 구성할 때, 어셈블리 언어만 사용하여 S/W를 구성하는 것이 아닌 C 언어도 함께 사용하기 위해 위와 같이 어셈블리어를 수정하였다. Reset_Handler가 작동하면 C언어의 main으로 이동한다.

```
                      ː      
        				; External Interrupts
						        				
        				DCD		Timer_Handler
        				DCD		UART_Handler
        				DCD		Key_Handler 
                      ː  
```

인터럽트가 발생했을 때, 처리해줄 ISR의 주소를 exception vector에 정의해줬다.

```
                      ː      
Key_Handler     PROC
                EXPORT Key_Handler
				IMPORT  Key_ISR
                PUSH    {R0, R1, R2, R3, LR}
				        BL Key_ISR
                POP     {R0, R1, R2, R3, PC}                    ;return
                ENDP
                      ː  
```

위 함수는 KEY 인터럽트 핸들러이며, 인터럽트가 발생했을 때 인터럽트를 처리하는 함수를 담고 있다. 여기서는 c로 작성한 코드를 사용하기 위해 context switching을 해주었다. 나머지 Timer_Handler, UART_Handler도 동일하게 작성하였으며 여기서는 생략하였다.

#### edk.driver.h

```c
                      ː      
#define AHB_GPIO_JB_BASE		0x50000000
#define AHB_UART_BASE			0x51000000
#define AHB_TIMER_BASE			0x52000000
#define AHB_GPIO_BASE			0x53000000
#define AHB_7SEG_BASE			0x54000000
#define AHB_KEY_BASE			0x55000000
#define NVIC_INT_ENABLE		0xE000E100
                      ː  
```

각각의 주변 장치를 컨트롤하기 위해 주솟값을 정의해줬다. 이를 구조체를 사용하여 접근할 수 있도록 코드를 작성하였으며 이 부분은 생략하였다.

```c
                      ː      
void seven_seg_write(char dig1, char dig2,char dig3,char dig4);		//Write to 7-segment display
void timer_init(int load_value, int prescale, int mode);			//Initialize the timer	
void timer_enable(void);				//Enable the timer	
void timer_irq_clear(void);			//Clear interrupt request from timer
int GPIO_read(void);						//GPIO read (from switches)
void GPIO_write(int data);			//GPIO write (to LEDs)
int GPIO_JB_read(void);
void GPIO_JB_write(int data);
                      ː  
```

edk.driver.c에는 각 driver를 컨트롤하기 위한 함수를 정의하였으며 함수들은 위와 같다. 7Segment에 값을 쓸 수 있는 함수, timer 관련 초기화, 활성화 인터럽트 클리어 함수가 있다. GPIO는 4개의 함수가 있으며 읽고 쓸 수 있게 하는 함수이다. 그 중 GPIO_JB는 JB Pmod를 GPIO로 사용하여 읽고 쓸 수 있는 함수이다.

#### main.c [헤더부분]

```c
#include "EDK_CM0.h" 
#include "core_cm0.h"
#include "edk_driver.h"
#include <stdlib.h>
#include <time.h>
```

사용하는 파일을 include해줬다. “”는 파일이며, <>는 라이브러리이다. <stdlib.h>, <time.h>는 시스템에서 랜덤값을 시간을 seed로 만들어주기 위해서 import하였다.

```c
#define Timer_Interrput_Frequency 			1
#define System_Tick_Frequency 					50000000
#define Timer_Prescaler 					1					
#define Timer_Load_Value 								(System_Tick_Frequency/Timer_Interrput_Frequency/Timer_Prescaler)
```

TImer를 50000000 주기로 사용하기 위해서 위와 같이 정의해줬다.

```c
// MODE (SW)
#define MODE_UPCOUNTER					0x01
#define MODE_STOPWATCH					0x02
#define MODE_RANDOM_MULTIPLIER	                        0x04
#define MODE_ADDER_SUB					0x08
#define MODE_ROBOT						0x10
```

SW[4:0]은 모드를 나타낸다. 이를 위와 같이 정의해줬다. 이는 스위치에서 활성화된 스위치를 기반으로 작성하였다. 아래 그림은 스위치에 따른 모드이다.

![main.c [헤더부분]|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/541651aa21da.webp)

<center>그림 6 스위치의 모드</center>

```c
// Function
void delay(unsigned int);
unsigned int knowBit (unsigned int, unsigned int);
unsigned int checkMoreThanTwoSW (unsigned int);
unsigned int cut0to4(unsigned int);
unsigned int cut5to9(unsigned int);
unsigned int cut10to15(unsigned int);
void eachSegmentDivider(unsigned int, unsigned int);
void eachSegmentDivider_all(unsigned int);
void digReset(void);
void diggReset(void);
void inverseBit(unsigned int*);
void increase_dig_4bit(void);
void decrease_dig_4bit(void);
void increase_dig_3bit(void);
void decrease_dig_3bit(void);
```

시스템에서 사용한 함수들을 헤더 부분에 작성해주었다. 이 함수들의 역할은 다음과 같다.

`delay(unsigned int period)`: period 변수 하나를 받아 반복문을 사용해 그만큼 지연시키는 함수
`knowBit (unsigned int temp, unsigned int where)`: temp 변수에서 \[where] bit에 있는 bit를 추출하는 함수
`checkMoreThanTwoSW (unsigned int temp)`: 스위치가 0개나 2개 이상 켜져 있음을 감지하는 함수
`cut0to4(unsigned int temp)`: temp의 \[4:0]를 추출하는 함수
`cut5to9(unsigned int temp)`: temp의 \[9:5]를 추출하는 함수
`cut10to15(unsigned int temp)`: temp의 \[15:10]를 추출하는 함수
`eachSegmentDivider(unsigned int result, unsigned int thousand)`: result 변수를 세그먼트에 넣을 수 있도록 각 자리로 나눠서 넣어주는 함수, 1000의 자리는 임의의 값으로 선택할 수 있다.
eachSegmentDivider_all(unsigned int result): result 변수를 7세그먼트에 넣을 수 있도록 각 자리로 나눠서 넣어주는 함수
`digReset(void)`: dig 변수를 초기화해주는 함수
`diggReset(void)`: digg 변수를 초기화해주는 함수
`inverseBit(unsigned int* temp)`: temp가 1이면 0으로 0이면 1로 바꿔주는 함수
`increase_dig_4bit(void)`: 4자리의 dig 변수를 하나씩 증가하는 함수
`decrease_dig_4bit(void)`: 4자리의 dig 변수를 하나씩 감소하는 함수
`increase_dig_3bit(void)`: 3자리의 dig 변수를 하나씩 증가하는 함수
`decrease_dig_3bit(void)`: 3자리의 dig 변수를 하나씩 감소하는 함수

```c
// value
static char dig1,dig2,dig3,dig4;
static char digg1,digg2,digg3,digg4;
static char seven1000, seven100, seven10, seven1;
static unsigned int result;
static unsigned int stopwatchStart;
static unsigned int add_sub_select; // 0: add mode, 1: sub mode
static unsigned int stop_multi_select; // 0: stopwatch mode, 1: multiplier mode
```

전역 변수의 값들로 역할은 다음과 같다.

`dig1,dig2,dig3,dig4`: 업 카운터의 카운팅 값
`digg1,digg2,digg3,digg4`: 스탑워치의 카운팅 값
`seven1000, seven100, seven10, seven1`: 시스템에서 계산된 값을 7segment에 적기 전에 저장하는 변수
`result`: 시스템에서 계산된 값을 저장하는 변수
`stopwatchStart`: 스탑워치를 작동할지 멈출지를 결정하는 변수
`add_sub_select`: 덧셈과 뺄셈 모드를 결정하는 변수 (0: add mode, 1: sub mode)
`stop_multi_select`: 스탑워치와 곱셈 모드를 결정하는 변수 (0: stopwatch mode, 1: multiplier mode)

#### main.c \[UART_ISR()]

```c
void UART_ISR(void){
	*(unsigned char*) AHB_UART_BASE = knowBit(GPIO_read(),4);
}
```

UART_ISR 함수이다. UART의 인터럽트를 처리하기 위해서 정의했다. 여기서는 SW[4]의 값을 UART로 보내주는 역할을 한다.

#### main.c \[Timer_ISR()]

```c
void Timer_ISR(void){
	increase_dig_4bit();
	
	// MODE_UPCOUNTER
	if(cut0to4(GPIO_read()) == MODE_UPCOUNTER){ 
		seven_seg_write(dig1, dig2, dig3, dig4);
	}
	

	// MODE_STOPWATCH
	if(cut0to4(GPIO_read()) == MODE_STOPWATCH){
		decrease_dig_4bit();
		seven_seg_write(digg1, digg2, digg3, digg4);
	}
	
	// clear JB when not digg:0000
	if(((digg4==0) & (digg3==0) & (digg2==0) & (digg1==0)) != 1){ // 1 is digg all 0000
		GPIO_JB_write(0x00);
	}
	
	timer_irq_clear();
}
```

Timer_ISR 함수이다. Timer의 인터럽트를 처리하기 위해서 정의했다. 타이머 인터럽트가 발생하면 우선 dig가 하나 증가하고 UPCOUNTER 모드하면 이를 7세그먼트로 출력한다. 만약 STOPWATCH 모드라면 반대로 숫자를 하나씩 감소해주는 기능을 수행한다.

STOPWATCH 모드에서 0이 되면 GPIO_JB 핀을 통해 외부 LED가 점등을 하게 된다. 만약 digg가 0000이 아니라면 GPIO_JB 핀의 출력을 0으로 초기화해 LED를 꺼준다.

그 후 timer의 인터럽트 신호를 clear해준다.

#### main.c \[Key_ISR()]

```c
void Key_ISR(){
  unsigned int temp;
  temp = *(unsigned int*) AHB_KEY_BASE;

  if (temp == 0x1){ // top key:  up-counter reset, stopwatch reset
		digReset();
		diggReset();
                seven_seg_write(0,0,0,0);
		stopwatchStart = 0;
  }
  else if (temp == 0x2){ // left key
		increase_dig_3bit();
		seven_seg_write(digg1, digg2, digg3, digg4);
  }
  else if (temp == 0x4){ // right key
		decrease_dig_3bit();
		seven_seg_write(digg1, digg2, digg3, digg4);
  }
  else if (temp == 0x8){ // bottom key
		inverseBit(&stopwatchStart);
  }
  else if (temp == 0x10){ // middle key
		inverseBit(&add_sub_select);
		inverseBit(&stop_multi_select);
  }
}
```

Key_ISR 함수이다. Key의 인터럽트를 처리하기 위해서 정의했다. Key 인터럽트가 발생하면 5개의 물리적 Key 중에서 어느 Key를 눌렀는지 확인해야 한다. 그렇기 때문에 if문을 사용하여 각 5개의 물리적 Key가 눌렸을 때의 기능을 구현하였다. 아래는 이에 대한 설명이다.

`top key`: 업카운터와 스탑워치를 초기화하는 버튼으로 이에 대한 기능을 수행한다. dig와 digg변수를 초기화해주고 7세그먼트를 0000으로 초기화해준다. 또한, 스탑워치 작동 변수 stopwatchStart 를 0으로 초기화 시켜줌으로써 스탑워치를 정지시킨다.

`left key`: 스탑워치의 시간을 세팅하는데, 그 중 10의 자리를 증가시킨다. digg3를 하나 증가시켜준다.

`right key`: 스탑워치의 시간을 세팅하는데, 그 중 10의 자리를 감소시킨다. digg3를 하나 감소시켜준다.

`bottom key`: 스탑워치의 작동을 시작하거나, 일시중지한다. 이는 비트를 뒤집어주는 함수를 통해 수행된다. stop_multi_select가 0이면 작동을 하지 않는 상태이고, 1이면 작동을 하는 상태를 나타낸다.

`middle key`: 덧셈기/뺄셈기 모드, 랜덤 숫자 발생기/곱셈기 모드 일 때, 모드를 변경시켜준다. 이는 비트를 뒤집어주는 함수를 통해 수행된다. 예를 들어 0이면 덧셈기 모드, 1이면 뺄셈기 모드이다.

#### main.c \[main()]

```c
int main(void){

	//Initialise timer (load value, prescaler value, mode value)
	timer_init(Timer_Load_Value,Timer_Prescaler,1);
	timer_enable();
	seven_seg_write(0,0,0,0);
	GPIO_JB_write(0x00); //Initialise GPIO_JB
	stopwatchStart = 0;
	add_sub_select=0;
	stop_multi_select=0;
	
	NVIC_SetPriority (Timer_IRQn, 0x00);
	NVIC_SetPriority (UART_IRQn, 0x80);
	NVIC_SetPriority (KEY_IRQn, 0xC0);
	NVIC_EnableIRQ(Timer_IRQn);			//enable timer interrupt
	NVIC_EnableIRQ(UART_IRQn);			//enable UART interrupt
	NVIC_EnableIRQ(KEY_IRQn);				//enable UART interrupt
                          ː
```

main함수가 실행되면 기본적으로 초기화를 해준다. Timer의 Timer_Load_Value, Timer_Prescaler를 초기화해주고 타이머를 할성화해준다. 또한, 7세그먼트를 0으로 초기화, JB Pmod의 출력신호를 0으로 초기화, 변수 초기화 작업을 해준다.

그 후 인터럽트 신호의 우선순위를 설정하고 활성화하여 인터럽트를 받을 수 있는 상태로 만들어준다.

```c
                         ː
	while(1){
		GPIO_write(GPIO_read());
		
		if (checkMoreThanTwoSW(GPIO_read()) == 0)
			goto nothingHappen;

                          ː

		nothingHappen:
		delay(1000000); // ******
	}
}
```

main 내부에는 while문이 존재해 프로그램이 종료되지 않고 계속 Polling을 통해 서비스를 처리할 수 있도록 하였다. GPIO_write(GPIO_read());는 basys3 보드 내부의 sw를 읽어 내부 led에 적도록 하기 위해 사용되었다.

checkMoreThanTwoSW(GPIO_read()) 함수는 sw[4:0] 모드 스위치가 만약 0개 켜있거나 2이상 켜있을 경우 0을 반환하는데, 0을 반환한 경우 nothingHappen: 라벨로 건너뛰어서 다른 기능을 수행하지 못하게 하였다. 만약 sw가 만약 0이거나 2이상일 경우 시스템이 작동을 하게 되면 프로그램이 꼬이기 때문이다.

만약 모드 스위치가 정상적으로 1개만 켜있다면 goto문은 실행되지 않고 아래의 코드를 계속 수행한다.

```c

                          ː
		// MODE_RANDOM_MULTIPLIER	
		if (cut0to4(GPIO_read()) == MODE_RANDOM_MULTIPLIER	){
			if(stop_multi_select==0){ // RANDOM MODE
				result = rand() % 10000;
				eachSegmentDivider_all(result);
				seven_seg_write(seven1000, seven100, seven10, seven1);
			}
			else{ // MULTIPLIER MODE
				result = cut5to9(GPIO_read()) * cut10to15(GPIO_read());
				eachSegmentDivider(result, 0x00);
				seven_seg_write(seven1000, seven100, seven10, seven1);
			}
		}
                          ː

```

sw[2]가 켜진 경우, 랜덤 숫자를 출력하거나 곱셈기로 동작한다. stop_multi_select 변수에 따라 랜덤 숫자를 출력하거나 곱셈기로 동작하는데, stop_multi_select 변수는 middle-key를 누를 때마다 바뀐다.

우선 stop_multi_select가 0이면 랜덤 숫자를 출력한다. c함수 stdlib.h 라이브러리 내부의 rand()함수를 이용해 난수 값을 만들어 줬다. 이때 seed가 시간이므로 time.h도 include 해줘야 한다. 이렇게 만든 변수값을 % 10000을 통해 4자리의 값으로 만들어 주었고 7세그먼트에 이를 출력한다.

stop_multi_select가 1이면 곱셈기로 동작한다. cut5to9(GPIO_read())를 통해 sw[9:5]의 값을 추출하고 cut10to15(GPIO_read())를 통해 sw[15:10] 값을 추출하여 곱해준다. 곱해준 값을 7세그먼트에 출력한다.

```c
 ː
		// MODE_ADDER_SUB
		else if (cut0to4(GPIO_read()) == MODE_ADDER_SUB){
			if(add_sub_select==0){ // ADDER MODE
				result = cut5to9(GPIO_read()) + cut10to15(GPIO_read());
				eachSegmentDivider(result, 0x00);
				seven_seg_write(seven1000, seven100, seven10, seven1);
			}
			else{ // SUB MODE
				if(cut10to15(GPIO_read())> cut5to9(GPIO_read())){
					result = cut10to15(GPIO_read()) - cut5to9(GPIO_read());
					eachSegmentDivider(result, 0x00);
					seven_seg_write(seven1000, seven100, seven10, seven1);
				}
				else if(cut10to15(GPIO_read())< cut5to9(GPIO_read())){
					result = cut5to9(GPIO_read()) - cut10to15(GPIO_read());
					eachSegmentDivider(result, 0x11); // 0x11 is -
					seven_seg_write(seven1000, seven100, seven10, seven1);
				}
				else{
					seven_seg_write(0,0,0,0);
				}
			}
		}
                          ː
```

sw[3]이 켜진 경우, 덧셈기나 뺄셈기로 동작한다. add_sub_select 변수에 따라 덧셈기나 뺄셈기로 동작하는데, add_sub_select 변수는 middle-key를 누를 때마다 바뀐다.

우선 add_sub_select 0이면 덧셈기로 동작한다. cut5to9(GPIO_read())를 통해 sw[9:5]의 값을 추출하고 cut10to15(GPIO_read())를 통해 sw[15:10] 값을 추출하여 더해준다. 더해준 값을 7세그먼트에 출력한다.

add_sub_select 1이면 뺄셈기로 동작한다. cut5to9(GPIO_read())를 통해 sw[9:5]의 값을 추출하고 cut10to15(GPIO_read())를 통해 sw[15:10] 값을 추출하여 빼준다. 빼준 값을 7세그먼트에 출력한다.

뺄셈기일 경우 if문을 사용하여 처리를 해주었는데, A-B에서 A가 B보다 크면 양수가 나와 문제가 없지만 B가 A보다 클 경우 (-)를 디스플레이해줘야 하기에 if문으로 처리해주었다.

```c
                          ː
		// MODE_ROBOT
		else if (cut0to4(GPIO_read()) == MODE_ROBOT){
			seven_seg_write(0x11,0x11,0x11,0x11);
		}
                          ː
```

sw\[4]가 켜진 경우, ROBOT mode로 동작한다. 이때는 기본적으로 UART의 인터럽트로 처리를 하며, 위 코드는 ROBOT mode로 동작함을 알려주기 위해 7세그먼트에 (- - - -)를 표시하는 코드이다.

#### main.c \[Functions]

```c
                          ː
void delay(unsigned int period){
	unsigned int counter=0;
	unsigned int i=0;
  for (i=0;i<period;i++){
		counter++;
	}
}
                          ː
```

period 변수 하나를 받아 반복문을 사용해 그만큼 지연시키는 함수이다.

```c
  ː
// A function that extracts a specific bit
unsigned int knowBit (unsigned int temp, unsigned int where){
	return (temp & (1 << where)) >> where;
}

// A function that sets a specific bit
unsigned int SetBit (unsigned int temp, unsigned int where){
	return (temp | (1 << where));
}
                          ː
```

temp 변수에서 [where] bit에 있는 bit를 추출하는 함수이다. temp와 원하는 비트가 1인 값을 AND 연산하여 추출한다. 반대로 원하는 비트에 1을 OR 연산하면 값을 쓸 수 있다.

```c
 ː
// Check if more than two mode SW on => 1: only one, 0: 0 or more than two
unsigned int checkMoreThanTwoSW (unsigned int temp){
	unsigned int i = 0;
	unsigned int check_v = 0;
	
	while (i < 5){
		if (knowBit(temp, i) == 1)
			check_v ++;
		i ++;
	}
	
	if (check_v == 1)
		return check_v;
	else
		return 0;
}
                          ː
```

모드 스위치가 0개나 2개 이상 켜져 있음을 감지하는 함수이다. temp값을 받아서 각 자리에 1이 있는지 없는지 검사를 Loop로 수행한다. 만약 check_v가 1이라면 하나면 켜진 것이고 그 이외의 값이라면 0이나 2이상 켜진 것으로 0을 반환한다.

```c
                          ː
unsigned int cut0to4(unsigned int temp){
	temp = 0x001F & temp; // 0x001F: 0000 0000 0001 1111
	return temp;
}

unsigned int cut5to9(unsigned int temp){
	temp = 0x03E0 & temp; // 0x03E0: 0000 0011 1110 0000
	return temp >> 5;
}

unsigned int cut10to15(unsigned int temp){
	temp = 0x7C00 & temp; // 0x7C00: 0111 1100 0000 0000
	return temp >> 10;
}
                          ː
```

각각 temp의 [4:0]를 추출하는 함수, temp의 [9:5]를 추출하는 함수, temp의 [15:10]를 추출하는 함수이다. 원하는 비트에 1이 써진 값을 &연산함으로써 원하는 값을 추출할 수 있다.

```c
                          ː
// SegmentDivider -> 3bit + special sign
void eachSegmentDivider(unsigned int result, unsigned int thousand){
	seven1		=	(result) % 10;
	seven10		= (result % 100) /10 ;    
	seven100 	= (result %1000) / 100;
	seven1000 =	thousand;
}

// SegmentDivider -> 3bit + special sign
void eachSegmentDivider_all(unsigned int result){
	seven1		=	(result) % 10;
	seven10		= (result % 100) /10 ;    
	seven100 	= (result %1000) / 100;
	seven1000 =	(result) / 1000;
}
                          ː
```

result 변수를 세그먼트에 넣을 수 있도록 각 자리로 나눠서 넣어주는 함수이다. 1000의 자리는 임의의 값으로 선택할 수 있도록 따로 분리하였다. 1000의 자리를 따로 설정할 수 있는 함수가 필요한 이유는 연산 결과 (-)를 7세그먼트에 써야할 일이 있기 때문이다.

```c
                          ː
void inverseBit(unsigned int* temp){
	if(*temp==0)
		*temp = 1;
	else
		*temp = 0;
}
                          ː
```

temp가 1이면 0으로 0이면 1로 바꿔주는 함수이다. 모드를 바꿀 때 사용할 수 있다.

```c
                          ː
void digReset(){
	dig1 = 0;
	dig2 = 0;
	dig3 = 0;
	dig4 = 0;
}

void diggReset(){
	digg1 = 0;
	digg2 = 0;
	digg3 = 0;
	digg4 = 0;
}
                          ː
```

dig, digg 변수를 초기화해주는 함수이다.

```c
                          ː
void increase_dig_4bit(){
	dig4++;
	if(dig4==10){
		dig4=0;
		dig3++;
		if (dig3==10){
			dig3=0;
			dig2++;
			if (dig2==10){
				dig2=0;
				dig1++;
			}
		}
	}
}

void decrease_dig_4bit(){
	if(stopwatchStart==1){
			if(digg4==0){
				if(digg3==0){
					if(digg2==0){
						if(digg1==0){
							GPIO_JB_write(0x01);
						}
						else{
							digg1--;
							digg2 = 9;
						}
					}
					else{
						digg2--;
						digg3 = 9;
					}
				}
				else{
					digg3--;
					digg4 = 9;
				}
			}
			else
				digg4--;
		}
}
                          ː
```

각각 4자리의 dig 변수를 하나씩 증가하는 함수, 4자리의 dig 변수를 하나씩 감소하는 함수이다. GPIO_JB_write(0x01);는 스탑워치에서 감소하다가 0000이 되면 외부 LED를 밝혀줘야 할 때 사용된다.

```c
                          ː
void increase_dig_3bit(){
	digg3++;
	if (digg3==10){
		digg3=0;
		digg2++;
		if (digg2==10){
			digg2=0;
			digg1++;
		}
	}
}

void decrease_dig_3bit(){
	if(digg3==0){
		if(digg2==0){
			if(digg1==0){
				// digg: 0000
			}
			else{
				digg1--;
				digg2 = 9;
			}
		}
		else{
			digg2--;
			digg3 = 9;
		}
	}
	else{
		digg3--;
	}
}
                          ː
```

각각 10의 자리의 dig 변수를 증가하는 함수, 10의 자리의 dig 변수를 감소하는 함수이다. key의 왼쪽과 오른쪽을 눌렀을 때, 각각 스탑워치의 설정 시간이 증가, 감소하는 역할을 한다.

#### ESP32_Servo_UART.ino

```c
#include <ESP32_Servo.h>

Servo frontLegs, backLegs;

int FRONT_Servo = 2;
int BACK_Servo = 4;

void setup() {
  Serial.begin(19200);
  Serial2.begin(19200, SERIAL_8N1, 16, 17);

  frontLegs.attach(FRONT_Servo);
  backLegs.attach(BACK_Servo);
  frontLegs.write(90);  //Center the servo
  backLegs.write(90);   //Center the servo

  delay(5000);  //Wait 5 sec to fix bug legs
}

void loop() {
  if (Serial2.available()) {
    Serial.println(Serial2.read());

    if (Serial2.read() == 1)
      Servo_Work();
    else
      Servo_Wait();
  }
}

void Servo_Work() {
  delay(200);
  frontLegs.write(70);  //move front leg a little
  delay(200);
  backLegs.write(70);  //move back leg a little
  delay(200);
  frontLegs.write(110);  //move front leg a little
  delay(200);
  backLegs.write(110);  //move back leg a little
}


void Servo_Wait() {
  frontLegs.write(90);  //Center the servo
  backLegs.write(90);   //Center the servo
}
```

Basys3와 UART 통신을 위하여 ESP32에서도 동일하게 UART 통신을 위한 구성을 하였다. Serial2.begin(19200, SERIAL_8N1, 16, 17);을 통해 16번의 RX 핀과 17번의 TX 핀을 Serial2 통신으로 구성하였으며 앞서 Basys3에서 설정한 통신 속도(Baudrate)가 19200이였기에 ESP32에서도 동일하게 설정하였다. 또한 Basys3의 UART 통신에 관한 래퍼런스 매뉴얼을 참고하면 페러티 bit이 존재하지 않는 것을 알 수 있기에 ESP32의 통신 설정에서도 SERIAL_8N1 모드를 사용하여 통신을 진행하였다.

이후, Serial2.read()를 통해 UART 통신을 하여 Basys3에서 값을 읽어와 모터를 제어하였다.

## 5. 결과

### 기능1) 업 카운터

![기능1) 업 카운터|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/b067ae9fc847.webp)

### 기능2) 스탑워치

![기능2) 스탑워치|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/4b5ad35e34f7.webp)

### 기능3) 랜덤숫자/곱셈기

![기능3) 랜덤숫자/곱셈기|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/2bba5fd19156.webp)

### 기능4) 덧셈기/뺄셈기

![기능4) 덧셈기/뺄셈기|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/6d0759c29369.webp)

### 기능5) 로봇 컨트롤

ESP32의 코드를 구현한 Arduino IDE에서 아래의 코드를 통해 Basys3와의 UART 통신 결과값을 받아볼 수 있었으며 Basys3의 SW4가 활성화됨에 따라 들어오는 값이 0 또는 1로 변경된다는 것을 알 수 있었다.

![기능5) 로봇 컨트롤|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/db4cc3a24915.webp)

<center>그림 7 UART 통신 함수 및 Serial 통신 함수를 사용한 UART 통신 확인</center>

![기능5) 로봇 컨트롤 (2)|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/4dc03ee4dcd5.webp)

<center>그림 8 Basys3 SW4의 활성화 여부에 따른 UART 통신 결과값 확인</center>

이를 이용하여 서보 모터를 0 또는 1의 값에 의해 제어를 하였으며 이 조건문을 통해 아래 그림의 로봇을 제어할 수 있게 되었다.

![기능5) 로봇 컨트롤 (3)|medium](/img/01HJM94A00Z2MN7AR6C8JMK9KE/57f0e37709b3.webp)

<center>그림 9 Basys3와 ESP32를 통해 제어하는 소형 로봇</center>

## 6. 고찰

다기능 카운터, 계산기와 UART 통신을 통한 작은 로봇 제어를 설계하였다. 모든 결과가 설계대로 잘 작동함을 확인할 수 있었다.

프로젝트를 수행하며 ARM HW에서 모듈과 버스의 관계에 대한 이해가 향상되었으며, 각 모듈에 대한 사용법을 확실히 익힐 수 있었다. 또한, memory mapped I/O를 이해해 외부 장치 컨트롤에 대한 이해가 높아졌고 외부 통신 방법에 대한 이해가 높아졌다.

프로젝트를 수행하며 각자의 역할은 다음과 같이 나누어 수행을 했다.

Me : Verilog ARM Cortex-M0 HW 구성, 시스템의 SW 구현

황태욱 : Verilog ARM Cortex-M0 HW 구성, ESP32-WROOM-32D 회로 구성 및 코드 작성, 로봇 구현

[조원 블로그로 이동하기](https://velog.io/@tae_uk/%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%98%A8-%EC%B9%A9SoC-%EA%B3%BC%EC%A0%9C-...-2-Basys3%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-%EB%8B%A4%EA%B8%B0%EB%8A%A5-%EC%B9%B4%EC%9A%B4%ED%84%B0-%EA%B3%84%EC%82%B0%EA%B8%B0%EC%99%80-UART-%ED%86%B5%EC%8B%A0%EC%9D%84-%ED%86%B5%ED%95%9C-%EB%A1%9C%EB%B4%87-%EC%A1%B0%EC%A0%95)
