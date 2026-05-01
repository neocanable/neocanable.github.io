## Binary static analysis problem: null pointer check

I was working on the IR of a rosemary and encountered a very interesting problem. After several rounds of troubleshooting, I discovered that this type of problem has a name: null pointer check.

#### 1. The Beginning of the Problem

I was trying to identify jumptables and vtables. The corresponding IR should look like this:

```c
call:blr(load([(load([X1]) + 0x48)]))
```

Here, `X1` should be `this` or some other pointer. However, during debugging, I found a lot of **call:blr(load([(load([0x0]) + 0x48)]))**

The pointer here is a constant 0x0. If it were a constant in `.data` or `.bss`, this would be easy to understand, but it's 0, which is very frustrating.

#### 2. Debugging

The function code:

```assembly
.text:00000000004C1FD0 ; __unwind { // sub_4DAAD8
.text:00000000004C1FD0                 STP             X28, X27, [SP,#-0x10+var_50]!
.text:00000000004C1FD4                 STP             X26, X25, [SP,#0x50+var_40]
.text:00000000004C1FD8                 STP             X24, X23, [SP,#0x50+var_30]
.text:00000000004C1FDC                 STP             X22, X21, [SP,#0x50+var_20]
.text:00000000004C1FE0                 STP             X20, X19, [SP,#0x50+var_10]
.text:00000000004C1FE4                 STP             X29, X30, [SP,#0x50+var_s0]
.text:00000000004C1FE8                 ADD             X29, SP, #0x50
.text:00000000004C1FEC                 SUB             SP, SP, #0x180
.text:00000000004C1FF0                 ADRP            X25, #__stack_chk_guard_ptr@PAGE
.text:00000000004C1FF4                 LDR             X25, [X25,#__stack_chk_guard_ptr@PAGEOFF]
.text:00000000004C1FF8                 MOV             X22, X5
.text:00000000004C1FFC                 MOV             X19, X4
.text:00000000004C2000                 MOV             X20, X2
.text:00000000004C2004                 LDR             X25, [X25]
.text:00000000004C2008                 MOV             X21, X1
.text:00000000004C200C                 ADD             X8, SP, #0x1D0+var_1B8
.text:00000000004C2010                 STR             X25, [X8]
.text:00000000004C2014                 STP             XZR, XZR, [SP,#0x1D0+var_170]
.text:00000000004C2018                 STR             XZR, [SP,#0x1D0+var_178]
.text:00000000004C201C                 ADD             X8, SP, #0x1D0+var_180
.text:00000000004C2020                 MOV             X0, X3
.text:00000000004C2024                 BL              sub_4BB978
.text:00000000004C2028                 ADRP            X1, #off_6C84B0@PAGE
.text:00000000004C202C                 LDR             X1, [X1,#off_6C84B0@PAGEOFF]
.text:00000000004C2030                 ADD             X0, SP, #0x1D0+var_180
.text:00000000004C2034                 BL              sub_4D12A4
.text:00000000004C2038                 LDR             X8, [X0]
.text:00000000004C203C                 LDR             X8, [X8,#0x60]
.text:00000000004C2040                 ADRP            X1, #off_6C84A8@PAGE ; "0123456789abcdefABCDEFxX+-pPiInN"
.text:00000000004C2044                 LDR             X1, [X1,#off_6C84A8@PAGEOFF] ; "0123456789abcdefABCDEFxX+-pPiInN"
.text:00000000004C2048                 SUB             X3, X29, #-var_C0
.text:00000000004C204C                 ADD             X2, X1, #0x1A
.text:00000000004C2050                 BLR             X8
.text:00000000004C2054                 LDR             X0, [SP,#0x1D0+var_180]
.text:00000000004C2058                 BL              sub_4D87DC
.text:00000000004C205C                 STP             XZR, XZR, [SP,#0x1D0+var_190]
.text:00000000004C2060                 STR             XZR, [SP,#0x1D0+var_198]
.text:00000000004C2064                 ADD             X0, SP, #0x1D0+var_198
.text:00000000004C2068                 MOV             W1, #0x16
.text:00000000004C206C                 MOV             W2, WZR
.text:00000000004C2070                 ADD             X23, SP, #0x1D0+var_198
.text:00000000004C2074                 BL              sub_7C2E4
.text:00000000004C2078                 LDRB            W8, [SP,#0x1D0+var_198]
.text:00000000004C207C                 LDR             X9, [SP,#0x1D0+var_188]
.text:00000000004C2080                 ORR             X26, X23, #1
.text:00000000004C2084                 STR             WZR, [SP,#0x1D0+var_1AC]
.text:00000000004C2088                 TST             W8, #1
.text:00000000004C208C                 ADD             X10, SP, #0x1D0+var_160
.text:00000000004C2090                 SUB             X27, X29, #-var_C0
.text:00000000004C2094                 CSEL            X23, X26, X9, EQ
.text:00000000004C2098                 ADD             X28, SP, #0x1D0+var_1A8
.text:00000000004C209C                 STP             X10, X23, [SP,#0x1D0+var_1A8]
.text:00000000004C20A0                 B               loc_4C20AC
.text:00000000004C20A4 ; ---------------------------------------------------------------------------
.text:00000000004C20A4
.text:00000000004C20A4 loc_4C20A4                              ; CODE XREF: sub_4C1FD0+220↓j
.text:00000000004C20A4                 ADD             X8, X8, #4
.text:00000000004C20A8                 STR             X8, [X21,#0x18]
.text:00000000004C20AC
.text:00000000004C20AC loc_4C20AC                              ; CODE XREF: sub_4C1FD0+D0↑j
.text:00000000004C20AC                                         ; sub_4C1FD0+234↓j
.text:00000000004C20AC                 CBZ             X21, loc_4C20C4
.text:00000000004C20B0                 LDP             X8, X9, [X21,#0x18]
.text:00000000004C20B4                 CMP             X8, X9
.text:00000000004C20B8                 B.EQ            loc_4C20D0
.text:00000000004C20BC                 LDR             W0, [X8]
.text:00000000004C20C0                 B               loc_4C20E0
.text:00000000004C20C4 ; ---------------------------------------------------------------------------
.text:00000000004C20C4
.text:00000000004C20C4 loc_4C20C4                              ; CODE XREF: sub_4C1FD0:loc_4C20AC↑j
.text:00000000004C20C4                 MOV             X21, XZR
.text:00000000004C20C8                 MOV             W24, #1
.text:00000000004C20CC                 B               loc_4C20EC
.text:00000000004C20D0 ; ---------------------------------------------------------------------------
.text:00000000004C20D0
.text:00000000004C20D0 loc_4C20D0                              ; CODE XREF: sub_4C1FD0+E8↑j
.text:00000000004C20D0                 LDR             X8, [X21]
.text:00000000004C20D4                 LDR             X8, [X8,#0x48]
.text:00000000004C20D8                 MOV             X0, X21
.text:00000000004C20DC                 BLR             X8
.text:00000000004C20E0
.text:00000000004C20E0 loc_4C20E0                              ; CODE XREF: sub_4C1FD0+F0↑j
.text:00000000004C20E0                 CMN             W0, #1
.text:00000000004C20E4                 CSET            W24, EQ
.text:00000000004C20E8                 CSEL            X21, XZR, X21, EQ
.text:00000000004C20EC
.text:00000000004C20EC loc_4C20EC                              ; CODE XREF: sub_4C1FD0+FC↑j
.text:00000000004C20EC                 CBZ             X20, loc_4C2124
.text:00000000004C20EC                 CBZ             X20, loc_4C2124
.text:00000000004C20F0                 LDP             X8, X9, [X20,#0x18]
.text:00000000004C20F4                 CMP             X8, X9
.text:00000000004C20F8                 B.EQ            loc_4C2104
.text:00000000004C20FC                 LDR             W0, [X8]
.text:00000000004C2100                 B               loc_4C2114
.text:00000000004C2104 ; ---------------------------------------------------------------------------
.text:00000000004C2104
.text:00000000004C2104 loc_4C2104                              ; CODE XREF: sub_4C1FD0+128↑j
.text:00000000004C2104                 LDR             X8, [X20]
.text:00000000004C2108                 LDR             X8, [X8,#0x48]
.text:00000000004C210C                 MOV             X0, X20
.text:00000000004C2110                 BLR             X8
.text:00000000004C2114
.text:00000000004C2114 loc_4C2114                              ; CODE XREF: sub_4C1FD0+130↑j
.text:00000000004C2114                 CMN             W0, #1
.text:00000000004C2118                 B.EQ            loc_4C2124
.text:00000000004C211C                 CBNZ            W24, loc_4C212C
.text:00000000004C2120                 B               loc_4C2224
.text:00000000004C2124 ; ---------------------------------------------------------------------------
.text:00000000004C2124
.text:00000000004C2124 loc_4C2124                              ; CODE XREF: sub_4C1FD0:loc_4C20EC↑j
.text:00000000004C2124                                         ; sub_4C1FD0+148↑j
.text:00000000004C2124                 MOV             X20, XZR
.text:00000000004C2128                 TBNZ            W24, #0, loc_4C2224
.text:00000000004C212C
.text:00000000004C212C loc_4C212C                              ; CODE XREF: sub_4C1FD0+14C↑j
.text:00000000004C212C                 LDRB            W8, [SP,#0x1D0+var_198]
.text:00000000004C2130                 LDR             X9, [SP,#0x1D0+var_190]
.text:00000000004C2134                 LDR             X10, [SP,#0x1D0+var_1A0]
.text:00000000004C2138                 LSR             X11, X8, #1
.text:00000000004C213C                 TST             W8, #1
.text:00000000004C2140                 CSEL            X24, X11, X9, EQ
.text:00000000004C2144                 ADD             X8, X23, X24
.text:00000000004C2148                 CMP             X10, X8
.text:00000000004C214C                 B.NE            loc_4C219C
.text:00000000004C2150                 LSL             X1, X24, #1
.text:00000000004C2154                 MOV             W2, WZR
.text:00000000004C2158                 ADD             X0, SP, #0x1D0+var_198
.text:00000000004C215C                 BL              sub_7C2E4
.text:00000000004C2160                 LDRB            W8, [SP,#0x1D0+var_198]
.text:00000000004C2164                 MOV             W1, #0x16
.text:00000000004C2168                 TBZ             W8, #0, loc_4C2178
.text:00000000004C216C                 LDR             X8, [SP,#0x1D0+var_198]
.text:00000000004C2170                 AND             X8, X8, #0xFFFFFFFFFFFFFFFE
.text:00000000004C2174                 SUB             X1, X8, #1
.text:00000000004C2178
.text:00000000004C2178 loc_4C2178                              ; CODE XREF: sub_4C1FD0+198↑j
.text:00000000004C2178                 MOV             W2, WZR
.text:00000000004C217C                 ADD             X0, SP, #0x1D0+var_198
.text:00000000004C2180                 BL              sub_7C2E4
.text:00000000004C2184                 LDRB            W8, [SP,#0x1D0+var_198]
.text:00000000004C2188                 LDR             X9, [SP,#0x1D0+var_188]
.text:00000000004C218C                 TST             W8, #1
.text:00000000004C2190                 CSEL            X23, X26, X9, EQ
.text:00000000004C2194                 ADD             X8, X23, X24
.text:00000000004C2198                 STR             X8, [SP,#0x1D0+var_1A0]
.text:00000000004C219C
.text:00000000004C219C loc_4C219C                              ; CODE XREF: sub_4C1FD0+17C↑j
.text:00000000004C219C                 LDP             X8, X9, [X21,#0x18]
.text:00000000004C21A0                 CMP             X8, X9
.text:00000000004C21A4                 B.EQ            loc_4C21B0
.text:00000000004C21A8                 LDR             W0, [X8]
.text:00000000004C21AC                 B               loc_4C21C0
.text:00000000004C21B0 ; ---------------------------------------------------------------------------
.text:00000000004C21B0
.text:00000000004C21B0 loc_4C21B0                              ; CODE XREF: sub_4C1FD0+1D4↑j
.text:00000000004C21B0                 LDR             X8, [X21]
.text:00000000004C21B4                 LDR             X8, [X8,#0x48]
.text:00000000004C21B8                 MOV             X0, X21
.text:00000000004C21BC                 BLR             X8
.text:00000000004C21C0
.text:00000000004C21C0 loc_4C21C0                              ; CODE XREF: sub_4C1FD0+1DC↑j
.text:00000000004C21C0                 MOV             W1, #0x10
.text:00000000004C21C4                 ADD             X3, SP, #0x1D0+var_1A0
.text:00000000004C21C8                 ADD             X4, SP, #0x1D0+var_1AC
.text:00000000004C21CC                 ADD             X6, SP, #0x1D0+var_178
.text:00000000004C21D0                 ADD             X7, SP, #0x1D0+var_160
.text:00000000004C21D4                 MOV             X2, X23
.text:00000000004C21D8                 MOV             W5, WZR
.text:00000000004C21DC                 STP             X28, X27, [SP,#0x1D0+var_1D0]
.text:00000000004C21E0                 BL              sub_4C23C8
.text:00000000004C21E4                 CBNZ            W0, loc_4C2224
.text:00000000004C21E8                 LDP             X8, X9, [X21,#0x18]
.text:00000000004C21EC                 CMP             X8, X9
.text:00000000004C21F0                 B.NE            loc_4C20A4
.text:00000000004C21F4                 LDR             X8, [X21]
.text:00000000004C21F8                 LDR             X8, [X8,#0x50]
.text:00000000004C21FC                 MOV             X0, X21
.text:00000000004C2200                 BLR             X8
.text:00000000004C2204                 B               loc_4C20AC
.text:00000000004C2208 ; ---------------------------------------------------------------------------
.text:00000000004C2208
.text:00000000004C2208 loc_4C2208                              ; CODE XREF: sub_4C1FD0+3C0↓j
.text:00000000004C2208                 MOV             X19, X0
.text:00000000004C220C
.text:00000000004C220C loc_4C220C                              ; CODE XREF: sub_4C23B4+10↓j
.text:00000000004C220C                 ADD             X0, SP, #0x1D0+var_198
.text:00000000004C2210                 BL              sub_75B48
.text:00000000004C2214
.text:00000000004C2214 loc_4C2214                              ; CODE XREF: sub_4C1FD0+3D4↓j
.text:00000000004C2214                                         ; sub_4C1FD0+3DC↓j
.text:00000000004C2214                 ADD             X0, SP, #0x1D0+var_178
.text:00000000004C2218                 BL              sub_75B48
.text:00000000004C221C                 MOV             X0, X19
.text:00000000004C2220                 BL              sub_505080
.text:00000000004C2224
.text:00000000004C2224 loc_4C2224                              ; CODE XREF: sub_4C1FD0+150↑j
.text:00000000004C2224                                         ; sub_4C1FD0+158↑j ...
.text:00000000004C2224                 LDR             X8, [SP,#0x1D0+var_1A0]
.text:00000000004C2228                 SUB             X1, X8, X23
.text:00000000004C222C                 MOV             W2, WZR
.text:00000000004C2230                 ADD             X0, SP, #0x1D0+var_198
.text:00000000004C2234                 BL              sub_7C2E4
.text:00000000004C2238                 ADD             X24, SP, #0x1D0+var_1B8
.text:00000000004C223C                 LDRB            W8, [SP,#0x1D0+var_198]
.text:00000000004C2240                 ADRP            X10, #byte_6D2CB8@PAGE
.text:00000000004C2244                 LDR             X9, [SP,#0x1D0+var_188]
.text:00000000004C2248                 ADD             X10, X10, #byte_6D2CB8@PAGEOFF
.text:00000000004C224C                 LDARB           W10, [X10]
.text:00000000004C2250                 TST             W8, #1
.text:00000000004C2254                 CSEL            X23, X26, X9, EQ
.text:00000000004C2258                 AND             W8, W10, #1
.text:00000000004C225C                 TBNZ            W8, #0, loc_4C2298
.text:00000000004C2260                 ADRL            X0, byte_6D2CB8
.text:00000000004C2268                 BL              sub_4DA76C
.text:00000000004C226C                 CBZ             W0, loc_4C2298
.text:00000000004C2270                 ADRP            X1, #aC@PAGE ; "C"
.text:00000000004C2274                 MOV             X2, XZR
.text:00000000004C2278                 ADD             X1, X1, #aC@PAGEOFF ; "C"
.text:00000000004C227C                 MOV             W0, #0x1FBF
.text:00000000004C2280                 BL              .newlocale
.text:00000000004C2284                 ADRP            X8, #qword_6D2CB0@PAGE
.text:00000000004C2288                 STR             X0, [X8,#qword_6D2CB0@PAGEOFF]
.text:00000000004C228C                 ADRL            X0, byte_6D2CB8
.text:00000000004C2294                 BL              sub_4DA82C
.text:00000000004C2298
.text:00000000004C2298 loc_4C2298                              ; CODE XREF: sub_4C1FD0+28C↑j
.text:00000000004C2298                                         ; sub_4C1FD0+29C↑j
.text:00000000004C2298                 ADRP            X8, #qword_6D2CB0@PAGE
.text:00000000004C229C                 LDR             X1, [X8,#qword_6D2CB0@PAGEOFF]
.text:00000000004C22A0                 ADRL            X2, aP_1 ; "%p"
.text:00000000004C22A8                 MOV             X0, X23
.text:00000000004C22AC                 MOV             X3, X22
.text:00000000004C22B0                 BL              sub_4BFE04
.text:00000000004C22B4                 CMP             W0, #1
.text:00000000004C22B8                 B.EQ            loc_4C22C4
.text:00000000004C22BC                 MOV             W8, #4
.text:00000000004C22C0                 STR             W8, [X19]
.text:00000000004C22C4
.text:00000000004C22C4 loc_4C22C4                              ; CODE XREF: sub_4C1FD0+2E8↑j
.text:00000000004C22C4                 CBZ             X21, loc_4C22DC
.text:00000000004C22C8                 LDP             X8, X9, [X21,#0x18]
.text:00000000004C22CC                 CMP             X8, X9
.text:00000000004C22D0                 B.EQ            loc_4C22E8
.text:00000000004C22D4                 LDR             W0, [X8]
.text:00000000004C22D8                 B               loc_4C22F8
.text:00000000004C22DC ; ---------------------------------------------------------------------------
.text:00000000004C22DC
.text:00000000004C22DC loc_4C22DC                              ; CODE XREF: sub_4C1FD0:loc_4C22C4↑j
.text:00000000004C22DC                 MOV             X21, XZR
.text:00000000004C22E0                 MOV             W22, #1
.text:00000000004C22E4                 B               loc_4C2304
.text:00000000004C22E8 ; ---------------------------------------------------------------------------
.text:00000000004C22E8
.text:00000000004C22E8 loc_4C22E8                              ; CODE XREF: sub_4C1FD0+300↑j
.text:00000000004C22E8                 LDR             X8, [X21]
.text:00000000004C22EC                 LDR             X8, [X8,#0x48]
.text:00000000004C22F0                 MOV             X0, X21
.text:00000000004C22F4                 BLR             X8
```



At location **0x4C22F4** is a call function. To identify x8, I need to find the expression corresponding to x8. 

Therefore, I should find the definition of x21 in **LDR X8, [X21]**. Finally found its definition at location 0x4C20C4: **MOV X21, XZR**. Here, it's confirmed that the predecessor of the 0x4C22F4 basic block does indeed set X21 to NULL. 

Then, I continued tracing its assembly code and discovered the final problem: The actual place where x21 is set to 0 is at 4C22DC, which is the taken target of CBZ X21. Then, it directly calls B loc_4C2304, without reaching 4C22E8/4C22F4. Therefore, although the definition X21 = XZR exists, it is unreachable from the path 4C22F4.

#### 3. Solution: 

When using a lifter, the basic semantics of conditional statements such as cbz/cbnz need to be expanded. This allows for the direct pruning of many unreachable branches during control flow backtracking. 

This is called null pointer check.