---
id: "01GP261500EH9QX3MHJNVE5XXA"
title_en: "Kogge-Stone Adder in Verilog: Fewer Logic Levels with Higher-Valency Cells"
summary_en: "I designed a Kogge-Stone adder, the tree adder with the fewest logic levels, in Verilog using higher-valency cells. Covers the bitwise PG, group PG, and SUM logic code, testbench waveform verification, and the performance optimizations and analysis."
---
![Kogge-Stone Adder using higher-valency Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/c502f1c01bc2.webp)

## 1. Kogge-Stone Adder
A fast adder can be achieved by implementing the adder with the fewest logic levels in its group PG network. Among tree adders, the one with the fewest logic levels is the Kogge-Stone adder, which has 4 logic levels as shown below.

![Kogge-Stone Adder|medium](/img/01GP261500EH9QX3MHJNVE5XXA/b9d8f35c5636.webp)

<center>Figure 1 16-bit Kogge-Stone Adder (no Carry_in)</center>

<u>Source: CMOS VLSI Design: A Circuits and Systems Perspective, 4th Edition</u>

This Kogge-Stone adder needs 8 wiring tracks, or 9 if Carry_in is included. It trades off this wiring-track cost for fast performance with few logic levels.

Below is a Kogge-Stone adder that includes Carry_in.

![Kogge-Stone Adder (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/5cfd968106c2.webp)

<center>Figure 2 16-bit Kogge-Stone Adder (Carry_in)</center>

The logic levels of the Kogge-Stone adder in Figure 2 can be reduced with higher-valency cells.

![Kogge-Stone Adder (3)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/fcd85a60d31c.webp)

<center>Figure 3 4-bit Higher Valency cell</center>

## 2. Kogge-Stone Adder using Higher Valency Cell
Figure 4 below shows the Kogge-Stone adder designed with higher-valency cells.

![Kogge-Stone Adder using Higher Valency Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/c502f1c01bc2.webp)

<center>Figure 4 16-bit Kogge-Stone Adder using Higher Valency Cell</center>

The `Bitwise PG Logic` performs a bitwise operation on each input to produce `Propagate` and `Generate`. These Propagate and Generate results go into the `Group PG Logic`, pass through the `higer valency cell`s, and each <u>outputs the Generate result from n down to C_in</u>.

The black higher-valency cells in stage 2 take at most 4 inputs, so they can only output results down to 15:0. This was handled later in the SUM logic by combining G(15:0) and C_in in a gray cell to produce C_out.

Finally, the SUM logic computes the result by XORing the propagate output of the `Bitwise PG Logic` with the group generate output of the group PG logic.

With this design, the 4 logic levels of a regular Kogge-Stone adder were reduced to 2 logic levels.

## 3. Verilog Code Review

### Basic Gate/Cell Define

#### 1) XOR GATE

![XOR GATE|medium](/img/01GP261500EH9QX3MHJNVE5XXA/075824875611.webp){: width="300px" height=""}

![XOR GATE (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/a60cab7ab6a1.webp){: width="500px" height=""}

<center>Figure 5 XOR GATE</center>

I built the XOR used in the SUM logic. Only NAND and NOR were allowed, so I designed it with NANDs as shown above.

#### 2) Generate & Propagate Cell

![Generate & Propagate Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/9c013aecc9d2.webp){: width="300px" height=""}

![Generate & Propagate Cell (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/dcd78733803b.webp){: width="500px" height=""}

<center>Figure 6 Generate & Propagate Cell</center>

I defined a Generate & Propagate cell that produces the generate and propagate signals for each bit in the bitwise logic. Generate comes from an AND operation, and propagate comes from an XOR operation. I implemented it as shown above.

#### 3) Gray Cell

![Gray Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/734a4d4acbd8.webp){: width="300px" height=""}

![Gray Cell (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/c792f7b61f18.webp){: width="500px" height=""}

<center>Figure 7 Gray Cell</center>

I defined a 2-bit gray cell. It performs an AND-OR operation, which I implemented with NANDs. It computes the following logic.

`Ga:b = Pa Gb + Ga`

#### 4) 3-higher valency Gray Cell

![-higher valency Gray Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/c6356768db9a.webp){: width="300px" height=""}

![-higher valency Gray Cell (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/328dc93dcf45.webp){: width="500px" height=""}

<center>Figure 8 3-higher valency Gray Cell</center>

I built a 3-input higher-valency gray cell that takes P and G of a, P and G of b, and G of c, and produces Generate. It computes the following logic.

`Ga:c = Pa Pb Gc + Pa Gb + Ga`

#### 5) 3-higher valency Black Cell

![-higher valency Black Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/61066aedb6d8.webp){: width="300px" height=""}

![-higher valency Black Cell (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/b7e1c3ef9759.webp){: width="500px" height=""}

<center>Figure 9 3-higher valency Black Cell</center>

I built a 3-input higher-valency black cell that takes P and G of a, P and G of b, and P and G of c, and produces Generate and Propagate. It computes the following logic.

`Ga:c = Pa Pb Gc + Pa Gb + Ga`

`Pa:c = Pa Pb Pc`

It was not used in this design, but I built it for consistency.

#### 6) 4-higher valency Gray Cell

![-higher valency Gray Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/bfbb181e9e15.webp){: width="300px" height=""}

![-higher valency Gray Cell (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/562bf4274397.webp){: width="500px" height=""}

<center>Figure 10 4-higher valency Gray Cell</center>

I built a 4-input higher-valency gray cell that takes P and G of a, P and G of b, P and G of c, and G of d, and produces Generate. It computes the following logic.

**Ga:d = Pa Pb Pc Gd + Pa Pb Gc + Pa Gb + Ga**

#### 7) 4-higher valency Black Cell

![-higher valency Black Cell|medium](/img/01GP261500EH9QX3MHJNVE5XXA/77b08e362b22.webp){: width="300px" height=""}

![-higher valency Black Cell (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/d432b4141244.webp){: width="500px" height=""}

<center>Figure 11 4-higher valency Black Cell</center>

I built a 4-input higher-valency black cell that takes P and G of a, P and G of b, P and G of c, and P and G of d, and produces Generate and Propagate. It computes the following logic.

`Ga:d = Pa Pb Pc Gd + Pa Pb Gc + Pa Gb + Ga`

`Pa:d = Pa Pb Pc Pd`

### Bitwise PG Logic

![Bitwise PG Logic|medium](/img/01GP261500EH9QX3MHJNVE5XXA/7a23acc3eb9b.webp)


![Bitwise PG Logic (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/5d0277d1af0d.webp)

<center>Figure 12 Bitwise PG Logic</center>

This is the bitwise PG logic, which produces generate and propagate for each input bit. The logic for each of bits 0 to 15 was implemented with a generate statement.

### Group PG Logic

![Group PG Logic|medium](/img/01GP261500EH9QX3MHJNVE5XXA/a3c4226619d9.webp)

![Group PG Logic (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/54d596398dff.webp)

<center>Figure 13 Group PG Logic</center>

This logic performs the group PG operation on the P and G produced by the bitwise PG logic.

- 1 gray cell
- 1 3-higher valency gray cell
- 1 4-higher valency gray cell
- 13 4-higher valency black cells
- 4 gray cells
- 4 3-higher valency gray cells
- 4 4-higher valency gray cells
- 1 4-higher valency black cell

G15:0, which does not receive Cin here, is handled in the SUM logic with a gray cell.

### SUM Logic

![SUM Logic|medium](/img/01GP261500EH9QX3MHJNVE5XXA/a3df1742614a.webp)

![SUM Logic (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/4e7c0210945f.webp)

<center>Figure 14 SUM Logic</center>

This logic performs the final SUM operation on the P from the bitwise PG logic and the GG from the group PG logic. It performs XOR operations, and a gray cell combines G15:0 and P15:0 from the group PG logic with C_in to form G15:C_in, which becomes the final C_out.

The XOR's critical path goes through 3 NANDs, while the gray cell's critical path goes through 2 NANDs. Because of this, the critical path of the SUM logic does not pass through C_out.

## 4. Result Waveforms / Accuracy Verification

### Test Bench

![Test Bench|medium](/img/01GP261500EH9QX3MHJNVE5XXA/5678a4ebdf79.webp){: width="500px" height=""}

<center>Figure 15 Test Bench</center>

The testbench was designed to run in two modes. By default, it runs code that tests 10,000 random values. If you comment out the \`define statement declared at the very top, you can simulate with values you enter manually. Below are the results for manually entered values with the \`define statement commented out.

#### 1) Result Waveforms

![Result waveform|medium](/img/01GP261500EH9QX3MHJNVE5XXA/2c487f9389f1.webp)

<center>Figure 16 Result waveform in hex</center>

![Result waveform (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/b6650d31f6ab.webp)

<center>Figure 17 Result waveform in decimal</center>

#### 2) Accuracy Verification

To verify the accuracy of the module, I fed values into the inputs and checked whether the outputs were correct. Ideally I would verify every operation possible with 16-bit x 16-bit inputs, but that would mean 65536 x 65536 = 4,294,967,296 operations, including 0. Comparing all of them one by one in Vivado would also take a long time. So I verified with 10,000 randomly generated numbers instead.

These are the results of running the testbench code above with the \`define statement declared. The code compares the output against the result of adding the random values and flags an error if they differ.

- $urandom%65536 // Generates a random value from 0 to 65535. 16 bits can represent values from 0 to 65535.
- ((o_c<<16) + o_s) // The carry is worth 65536, so shifting it and adding o_s gives the final value.

![Accuracy verification|medium](/img/01GP261500EH9QX3MHJNVE5XXA/d0bfd14dce91.webp)

![Accuracy verification (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/24201cf8cb92.webp)

![Accuracy verification (3)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/f3f3bd3178b0.webp)

<center>Figure 18 Results of 10,000 checks (partially omitted)</center>

The verification confirmed that the adder works without problems. The error count is 0, so it operates correctly.

## 5. Performance Optimization / Performance Analysis

### 1) Performance Optimization

- Using higher-valency cells reduces logic levels, which cuts delay compared with chaining several gates
- In the Generate & Propagate cell, the AND gate is not used directly; it is taken from the middle of the XOR gate, saving gates
- The group PG logic computes 15:0 instead of 15:C_in and leaves the rest to the SUM logic, reducing logic levels

### 2) Performance Analysis

An adder's delay comes from the delay of each gate. The figure shows the critical path of the designed adder. Running a simulation to check it gives the results below.

![Performance analysis|medium](/img/01GP261500EH9QX3MHJNVE5XXA/b165d77e62d4.webp)

<center>Figure 19 Critical Path</center>

**Kogge-Stone Adder using Higher Valency Cell**

![Performance analysis (2)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/7830700ff568.webp)

**Original Kogge stone**

![Performance analysis (3)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/8296e9f59c87.webp)

<center>Figure 20 Result of 16'h0 + 16'h0</center>

For 16'h0 + 16'h0 (C_in: 0), the original Kogge-Stone adder has a delay of 1700ns. The Kogge-Stone adder using higher-valency cells has a delay of 1400ns, confirming that the delay decreased.

**Kogge-Stone Adder using Higher Valency Cell**

![Performance analysis (4)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/ffecaca0109c.webp)

**Original Kogge stone**

![Performance analysis (5)|medium](/img/01GP261500EH9QX3MHJNVE5XXA/2ad092a1c505.webp)

<center>Figure 21 Result of 16'hffff + 16'h1</center>

For 16'hffff + 16'h1 (C_in: 0), the original Kogge-Stone adder has a delay of 2000ns. The Kogge-Stone adder using higher-valency cells has a delay of 1800ns, confirming that the delay decreased.

## 6. An Alternative Design

This is an approach I came up with but did not actually implement. It is a sparse-tree adder in which the red blocks in the SUM logic are carry-lookahead adders. The red blocks compute at the same time as the group PG logic. Their outputs are then selected by the previous carry.

This design has the advantage of using less logic than the Kogge-Stone adder built only with higher-valency cells. In terms of performance, the red blocks are computed in parallel with the group PG logic, so they drop out, leaving only the XOR-based selection. That gives performance similar to the original Kogge-Stone adder built only with higher-valency cells.

Although it uses less logic than the Kogge-Stone adder built only with higher-valency cells, it could not improve performance, which was the goal of this project, so I only sketched it and did not implement it.

![Alternative design|medium](/img/01GP261500EH9QX3MHJNVE5XXA/29b76948cffd.webp)

<center>Figure 22 kogge stone sparse-tree adder using Higher Valency Cell</center>
