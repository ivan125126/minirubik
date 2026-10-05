# Homework 1: Optimizations and RISC-V Assembly

## Project Information

- Author: 蘇凡誠 (P76154642)
- GitHub repository: [ivan125126/minirubik](https://github.com/ivan125126/minirubik)
- Upstream commit: [Commit hash]
- Submission tag: [Tag]
- Ripes release type: Pre-release
- Ripes source commit (reported): [`5b8a616`](https://github.com/mortbopet/Ripes/commit/5b8a616edcb6f0a2ddb07e78951348b72497f1e1) (`5b8a616edcb6f0a2ddb07e78951348b72497f1e1`)
- Processor model used for the memory experiment: ISA simulator
- Instruction set architecture: RV32I
- Development period: September 30, 2026–ongoing


## 1. Baseline Characterization

### 1.1 How the Baseline Works

The baseline solver follows these steps:

1. Fix one corner to eliminate whole-cube rotational symmetry.
2. Encode the remaining corner permutation and six independent orientations as a dense integer rank.
3. Perform breadth-first search from the solved state using `R`, `B`, and `D`, including inverse and half turns.
4. Store one move toward the solved state for each discovered state. Following these stored moves gives a shortest solution of at most 11 moves in the half-turn metric (HTM).

Source: [minirubik README](https://github.com/ivan125126/minirubik/blob/231796cc48868f4ea276f652139b6bebbad0cd02/README.md).

#### Cube Representation and State Count

The 2×2×2 cube consists of eight corner cubies. The baseline fixes the position and orientation of the front-upper-left corner to eliminate whole-cube rotational symmetry. The remaining seven cubies can occupy the seven remaining positions in $7!$ different permutations. In the input representation, the digits `1` through `7` identify which cubie occupies each position.

Each corner cubie has three possible orientations, represented internally by `0`, `1`, and `2`. Under the baseline's orientation convention, every legal face turn preserves the orientation sum modulo 3:

$$
\sum_{i=1}^{7} o_i \equiv 0 \pmod{3}.
$$

Therefore, only six orientations are independent: once those six are specified, the seventh is determined by the constraint. This gives $3^6$ orientation configurations and a total state count of:

$$
7! \times 3^6
= 5{,}040 \times 729
= 3{,}674{,}160.
$$

The internal orientation values `0`, `1`, and `2` correspond to the input digits `1`, `2`, and `3`, respectively.
### 1.2 State Encoding

The baseline represents 5,040 × 729 = 3,674,160 states. The function `rank_state()` encodes each valid state as an integer in the range 0 to 3,674,159, while `unrank_state()` reconstructs a state from a rank in that range.

The encoding combines a permutation rank with an orientation rank. If `p` is the permutation rank and `o` is the orientation rank, the combined rank is:

$$
\operatorname{rank}(s) = p \times 729 + o.
$$

Source: [`rank_state()` and `unrank_state()`](https://github.com/ivan125126/minirubik/blob/231796cc48868f4ea276f652139b6bebbad0cd02/solver.c#L87).

### 1.3 Baseline Solver

The solver first builds separate quarter-turn transition tables for permutations and orientations. It then initializes the `toward_solved` table to 255 (`UINT8_MAX`), which marks unvisited states.

The BFS begins at rank 0, corresponding to the solved input `12345671111111`. For each state removed from the FIFO queue, the solver examines all nine HTM moves. If the resulting state has not been visited, it stores the inverse of the discovery move in `toward_solved` and appends the new state's rank to the queue. The process continues until the queue is empty.

Source: [`build_table()`](https://github.com/ivan125126/minirubik/blob/231796cc48868f4ea276f652139b6bebbad0cd02/solver.c#L191).

### 1.4 Memory Requirements

The following table summarizes the baseline's three dominant allocations. Their combined size is 18,405,414 bytes (approximately 17.553 MiB); this is not the total memory usage of the host process.

| Component | Bytes | Storage |
| --- | ---: | --- |
| Move table (`toward_solved`) | 3,674,160 | Heap |
| BFS queue | 14,696,640 | Heap |
| Factored quarter-turn transition tables | 34,614 | Automatic storage |
| Combined size of the three dominant allocations | 18,405,414 | — |

Source: [`build_table()`](https://github.com/ivan125126/minirubik/blob/231796cc48868f4ea276f652139b6bebbad0cd02/solver.c#L191).

### 1.5 Measurements on My Installation

### Host Environment

The following host information was obtained using the Windows `systeminfo` command.

| Item | Reported value |
| --- | --- |
| Operating system | Microsoft Windows 11 Pro |
| OS version | 10.0.26200 (Build 26200) |
| System manufacturer / model | ASUSTeK COMPUTER INC. / E500 G9 |
| Architecture | x64-based PC |
| Processor identification | Intel64 Family 6 Model 183 Stepping 1, GenuineIntel |
| Reported processor frequency | Approximately 2,100 MHz |
| Total physical memory | 24,178 MB, as reported by `systeminfo` |
| BIOS | American Megatrends Inc. 4505, November 29, 2025 |

At the time of the `systeminfo` snapshot, available physical memory was 10,939 MB and the reported maximum virtual memory size was 25,714 MB. These values describe that snapshot and were not recorded separately for each experimental run. The processor identification above is the value reported by Windows; the exact commercial CPU model remains to be recorded.

#### Host Memory per Guest Byte

##### Test Program and Experimental Sizes

I used the following memory-write loop, changing only the initial value of `a0` between the three experimental sizes:

```asm
li a0, 1024  # Use 131072 for 512 KiB or 262144 for 1 MiB.

loop:
    addi sp, sp, -4
    sw   a0, 0(sp)
    addi a0, a0, -1
    bne  a0, zero, loop
```

Each iteration moves the stack pointer down by four bytes and writes one 32-bit word to the new address. The intended write sizes are therefore:

| Initial `a0` | Iterations | Guest bytes written | Size |
| ---: | ---: | ---: | --- |
| 1,024 | 1,024 | 4,096 | 4 KiB (control) |
| 131,072 | 131,072 | 524,288 | 512 KiB |
| 262,144 | 262,144 | 1,048,576 | 1 MiB |

Test source / repository link: [Add the link to the exact test program]

##### Measurement Procedure

All memory experiments used the ISA simulator with the RV32I instruction set on a pre-release build of Ripes. For each size, I repeated the following procedure three times:

1. Open a fresh instance of Ripes and load the corresponding test program.
2. Run the program using **Execute simulator without updating UI**.
3. Record the Ripes process ID, `WorkingSet64`, and `PrivateMemorySize64` using the PowerShell command below.
4. Close Ripes before starting the next run.

```powershell
Get-Process *Ripes* |
    Select-Object Id, ProcessName, WorkingSet64, PrivateMemorySize64
```

Both memory columns are recorded in bytes. `WorkingSet64` reports the process working set, whereas `PrivateMemorySize64` reports private memory allocation. The two metrics are reported separately.

Measurement endpoint: [Describe how completion of all writes was confirmed before reading the process counters.]

##### Raw Measurements

| Guest size | Run | Process ID | WorkingSet64 (B) | PrivateMemorySize64 (B) |
| --- | ---: | ---: | ---: | ---: |
| 4 KiB | 1 | 26448 | 69,263,360 | 29,593,600 |
| 4 KiB | 2 | 29964 | 69,361,664 | 29,720,576 |
| 4 KiB | 3 | 8788 | 69,455,872 | 29,855,744 |
| 512 KiB | 1 | 34652 | 111,206,400 | 73,080,832 |
| 512 KiB | 2 | 18984 | 111,210,496 | 73,142,272 |
| 512 KiB | 3 | 14520 | 111,267,840 | 73,240,576 |
| 1 MiB | 1 | 11992 | 152,899,584 | 117,747,712 |
| 1 MiB | 2 | 26876 | 153,321,472 | 116,744,192 |
| 1 MiB | 3 | 2592 | 153,403,392 | 116,752,384 |

##### Results and Interpretation

For each guest write size, I calculated the arithmetic mean of the three recorded runs. To estimate the incremental host-memory cost per additional guest byte, I used:

$$
r = \frac{H_l - H_s}{G_l - G_s},
$$

where $H_l$ and $H_s$ are the mean host-memory measurements for the larger and smaller write sizes, respectively, and $G_l$ and $G_s$ are the corresponding guest write sizes in bytes. I calculated this ratio separately for Working Set and Private Bytes.

| Guest bytes written | Mean Working Set (B) | Mean Private Bytes (B) |
| --- | ---: | ---: |
| 4 KiB | 69,360,298.67 | 29,723,306.67 |
| 512 KiB | 111,228,245.33 | 73,154,560.00 |
| 1 MiB | 153,208,149.33 | 117,081,429.33 |

| Write-size interval | Private Bytes per additional guest byte | Working Set per additional guest byte |
| --- | ---: | ---: |
| 4 KiB → 512 KiB | 83.49 | 80.49 |
| 512 KiB → 1 MiB | 83.78 | 80.07 |
| 4 KiB → 1 MiB | 83.64 | 80.28 |

The two adjacent-interval Private Bytes slopes differ by approximately 0.35%, relative to the first interval's slope.

#### Simulation Rate

##### Experimental Method

I executed the same nested-loop program on the ISA simulator (RV32_ISS) and the 5-stage processor, using the RV32I instruction set and Ripes pre-release commit `5b8a616`.

Both loop bounds were set to 8,192. The inner loop consisted of an `addi` instruction and a conditional branch. After execution, I recorded the cycle count, retired instruction count, and displayed clock rate from the Execution info panel.

##### Recorded Results

| Processor model | Cycles | Retired instructions | Calculated CPI | Displayed clock rate |
| --- | ---: | ---: | ---: | ---: |
| RV32_ISS | 134,242,306 | 134,242,306 | 1.00000 | 11.00 MHz |
| 5-stage processor | 268,460,036 | 134,242,306 | 1.99982 | 337.72 kHz |

CPI was calculated as total cycles divided by retired instructions.
The GUI displayed a rounded CPI of 2 for the 5-stage processor.

##### Throughput Estimates

The following estimates were calculated from the displayed
clock rate using:

IRPS = displayed cycles per second / calculated CPI

| Processor model | Estimated IRPS from the GUI rate |
| --- | ---: |
| RV32_ISS | Approximately 11.00 million instructions/s |
| 5-stage processor | Approximately 168.88 thousand instructions/s |

These values are estimates derived from the displayed GUI rate.
They are not whole-run average throughput measurements based
on independently measured elapsed time.

### 1.6 Why the Baseline Does Not Fit Ripes

The original baseline is impractical for this Ripes assignment because :



## 2. Representation and Search Design

### 2.1 Target Constraints

static data < 128KiB
No heap, no recursion, no floating point. 
Use only RV32I instructions. No extensions are permitted, M included.
No distance-11 state exceeding 5*10^7 retired instructions on RV32_ISS

### 2.2 Chosen State Representation

[Describe your representation and justify the choice.]

### 2.3 Search Algorithm

[Explain the search procedure, termination, and optimality.]

### 2.4 Heuristic Design and Admissibility

[If heuristics are used, explain their construction and
why they never overestimate the exact distance.]

### 2.5 Data Layout and Memory Budget

| Data structure | Entries | Bytes/entry | Total bytes |
| --- | ---: | ---: | ---: |
| [ ] | [ ] | [ ] | [ ] |

- Total static data: [Bytes]
- Peak working set: [Bytes and calculation]

### 2.6 Alternatives and Trade-offs

[Explain alternatives you evaluated and why you accepted
or rejected them. Distinguish measurements from estimates.]

## 3. C Implementation and Optimization

### 3.1 Initial Implementation

[Link to the implementation and explain its structure.]

### 3.2 Optimization Steps

| Change | Reason | Operation-count effect | Evidence |
| --- | --- | --- | --- |
| [ ] | [ ] | [ ] | [ ] |

[Discuss branches, memory traffic, and costly arithmetic.]

### 3.3 Host Correctness Gates

| Gate | Verification method | Result | Evidence |
| --- | --- | --- | --- |
| H1: Heuristic admissibility over all states | [ ] | [ ] | [ ] |
| H2: Table completeness, maxima, solved entries | [ ] | [ ] | [ ] |
| H3: Optimal solution length for every state | [ ] | [ ] | [ ] |
| H4: Packed accessors match unpacked references | [ ] | [ ] | [ ] |

- H3 wall-clock time: [Seconds]
- Exact BFS oracle: [Implementation / version]

[If a gate is inapplicable, explain why.]

## 4. Hand-written RV32I Implementation

### 4.1 Program Structure

[Explain input handling, search, path storage,
and in-program validation.]

### 4.2 Register and Storage Allocation

[Explain important register assignments and storage choices.]

### 4.3 RV32I-specific Instruction Sequences

[Discuss relevant arithmetic, indexing, and control flow.
Quote only short code fragments needed for the explanation.]

### 4.4 Iterative Refinement

Measurements use the pinned Ripes build with the renderer
disabled. Code size is the byte size of linked `.text`.

| Version / commit | Input | Retired instructions | `.text` bytes | Change |
| --- | --- | ---: | ---: | --- |
| [ ] | [ ] | [ ] | [ ] | [ ] |

### 4.5 Comparison with GCC

Reference options:
`-O2 -march=rv32i -mabi=ilp32`

[Record compiler version, build commands, and comparable
measurement conditions.]

| Input | GCC instructions | Hand-written instructions | GCC `.text` | Hand-written `.text` |
| --- | ---: | ---: | ---: | ---: |
| [ ] | [ ] | [ ] | [ ] | [ ] |

[Explain any case where the hand-written version does not win.]

### 4.6 Worst-case Instruction Budget

- Number of distance-11 states tested: [ ]
- Maximum retired instruction count: [ ]
- State attaining the maximum: [ ]
- Budget satisfied for every distance-11 state: [ ]
- Instruction count for `21345671111111`: [ ]
- Test scripts and raw results: [Links]

### 4.7 Target Correctness Gates

| Gate | Method | Result | Evidence |
| --- | --- | --- | --- |
| T5: Returned paths reach solved | [ ] | [ ] | [ ] |
| T6: `21345671111111` has an optimal 11-move solution | [ ] | [ ] | [ ] |
| T7: Tests reproduce across required models | [ ] | [ ] | [ ] |

## 5. LED Matrix Visualization

### 5.1 Matrix Configuration and Addressing

- Width: 35
- Height: 25
- Assembler symbols: [Symbols used]

[Explain row-major addressing and the unfolded net layout.]

### 5.2 Cubie Orientation and Color Mapping

[Explain how state data determines facelet colors.]

### 5.3 Animation from Solver Output

[Explain how each actual solution move updates the display.]

### 5.4 GUI and CLI Builds

[Explain the assemble-time renderer switch.
Confirm that the builds differ only in the renderer.]

- Source and build instructions: [Link]
- Demonstration: [Screenshots or link]

## 6. Instruction-level Walkthrough

- Selected instructions / input: [ ]
- Ripes processor model: [ ]

| Stage | Behavior for the selected instructions |
| --- | --- |
| IF | [ ] |
| ID | [ ] |
| EX | [ ] |
| MEM | [ ] |
| WB | [ ] |

[Explain register write enable, multiplexer selection,
memory updates, and why the observed result is correct.]

## 7. Development Record

| Date | Commit / HackMD revision | Substantive progress |
| --- | --- | --- |
| [ ] | [ ] | [ ] |
| [ ] | [ ] | [ ] |
| [ ] | [ ] | [ ] |

## 8. AI Usage Disclosure


I used ChatGPT to prepare an initial report outline, obtain explanations of the baseline code, assist with arithmetic checks on the measurement data, and polish the English wording and Markdown formatting of the host-information and memory-measurement sections.

## 9. Conclusions and Remaining Limitations

[Summarize verified results, unresolved limitations,
and possible improvements supported by evidence.]

## References

1. [Source and link]
2. [Source and link]

## Appendix: Reproduction Instructions

[Provide exact build commands, test commands, inputs,
and links to raw measurement outputs.]