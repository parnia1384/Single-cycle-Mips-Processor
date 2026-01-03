# Single-Cycle MIPS Processor

## Overview
This is a Verilog implementation of a single-cycle MIPS processor based on the design described in "Digital Design and Computer Architecture" by David Harris and Sarah Harris. The processor implements a subset of the MIPS instruction set architecture and executes one instruction per clock cycle.

## Features
- **32-bit architecture** with 32 general-purpose registers
- **Single-cycle execution**: Each instruction completes in one clock cycle
- **Supported instruction types**:
  - R-type: ADD, SUB, AND, OR, SLT
  - I-type: ADDI, ANDI, ORI, SLTI, LW, SW, BEQ, BNE
- **Memory-mapped I/O** support
- **Five-stage pipeline conceptually implemented in single cycle**:
  1. Instruction Fetch (IF)
  2. Instruction Decode (ID)
  3. Execute (EX)
  4. Memory Access (MEM)
  5. Write Back (WB)

## Block Diagram
```mermaid
flowchart TD
    subgraph "Data Path"
        PC["PC<br/>Program Counter"]
        ADD4["Adder<br/>PC + 4"]
        IM["IM<br/>Instruction Memory"]
        RF["RF<br/>Register File<br/>(32 x 32-bit)"]
        ALU["ALU<br/>Arithmetic Logic Unit"]
        DM["DM<br/>Data Memory"]
        SE["Sign Extend<br/>16→32 bits"]
        SHL2["Shift Left 2"]
    end
    
    subgraph "Control Path"
        CU["Control Unit"]
        ALU_Ctrl["ALU Control"]
        PC_Src["PC Source Logic"]
    end
    
    subgraph "Multiplexers"
        MUX_ALUSrc["MUX<br/>ALUSrc"]
        MUX_RegDst["MUX<br/>RegDst"]
        MUX_MemtoReg["MUX<br/>MemtoReg"]
        MUX_PCSrc["MUX<br/>PCSrc"]
    end
    
    %% Main Flow
    PC --> IM
    IM --> CU
    IM --> RF
    IM --> SE
    IM --> MUX_RegDst
    
    %% Register File Flow
    RF --> ALU
    RF --> MUX_ALUSrc
    
    %% ALU Flow
    SE --> MUX_ALUSrc
    CU --> MUX_ALUSrc
    MUX_ALUSrc --> ALU
    CU --> ALU_Ctrl
    ALU_Ctrl --> ALU
    ALU --> DM
    ALU --> MUX_MemtoReg
    
    %% Memory Flow
    CU --> DM
    DM --> MUX_MemtoReg
    
    %% Write Back Flow
    CU --> MUX_MemtoReg
    MUX_MemtoReg --> RF
    CU --> MUX_RegDst
    MUX_RegDst --> RF
    CU -- RegWrite --> RF
    
    %% PC Update Flow
    PC --> ADD4
    ADD4 --> MUX_PCSrc
    SE --> SHL2
    SHL2 --> ADD_Branch["Adder<br/>PC+4 + offset"]
    ADD4 --> ADD_Branch
    ADD_Branch --> MUX_PCSrc
    IM --> Jump_Calc["Jump Address<br/>{PC[31:28], addr, 00}"]
    Jump_Calc --> MUX_PCSrc
    CU --> PC_Src
    ALU -- Zero Flag --> PC_Src
    PC_Src --> MUX_PCSrc
    MUX_PCSrc --> PC
    
    %% Styling
    classDef dataPath fill:#e1f5fe,stroke:#01579b
    classDef controlPath fill:#f3e5f5,stroke:#4a148c
    classDef mux fill:#fff3e0,stroke:#e65100
    
    class PC,IM,RF,ALU,DM,SE,SHL2,ADD4,ADD_Branch,Jump_Calc dataPath
    class CU,ALU_Ctrl,PC_Src controlPath
    class MUX_ALUSrc,MUX_RegDst,MUX_MemtoReg,MUX_PCSrc mux
```
## Implementation Details

### 1. Instruction Format Support

#### R-Type Instructions (Register)
```mermaid
flowchart TD
    subgraph "R-Type Instruction Format"
        direction LR
        subgraph bit31_26["31:26 (6 bits)"]
            op["Opcode<br/>000000"]
        end
        
        subgraph bit25_21["25:21 (5 bits)"]
            rs["rs<br/>Source Register"]
        end
        
        subgraph bit20_16["20:16 (5 bits)"]
            rt["rt<br/>Target Register"]
        end
        
        subgraph bit15_11["15:11 (5 bits)"]
            rd["rd<br/>Destination Register"]
        end
        
        subgraph bit10_6["10:6 (5 bits)"]
            shamt["shamt<br/>Shift Amount<br/>(00000 for ALU)"]
        end
        
        subgraph bit5_0["5:0 (6 bits)"]
            funct["funct<br/>Function Code"]
        end
        
        op --> rs --> rt --> rd --> shamt --> funct
    end
    
    subgraph "R-Type Examples"
        direction TB
        ADD["add $rd, $rs, $rt<br/>op=000000, funct=100000"]
        SUB["sub $rd, $rs, $rt<br/>op=000000, funct=100010"]
        AND["and $rd, $rs, $rt<br/>op=000000, funct=100100"]
        OR["or $rd, $rs, $rt<br/>op=000000, funct=100101"]
        SLT["slt $rd, $rs, $rt<br/>op=000000, funct=101010"]
    end
    
    subgraph "Execution Flow"
        RF["Register File"] -->|Read $rs| A
        RF -->|Read $rt| B
        A --> ALU["ALU<br/>Performs operation"]
        B --> ALU
        funct --> ALU_CTRL["ALU Control"]
        ALU_CTRL --> ALU
        ALU -->|Result| RF_Write["Write to $rd"]
        rd --> RF_Write
    end
    
    %% Styling
    classDef rtype fill:#e8f5e8,stroke:#2e7d32
    classDef example fill:#f3e5f5,stroke:#7b1fa2
    classDef exec fill:#e3f2fd,stroke:#1565c0
    
    class op,rs,rt,rd,shamt,funct rtype
    class ADD,SUB,AND,OR,SLT example
    class RF,ALU,ALU_CTRL,RF_Write,A,B exec
```
#### I-Type Instructions (Immediate)
31-26 25-21 20-16 15-0
┌───────┬───────┬───────┬──────────────┐
│ op │ rs │ rt │ immediate │
└───────┴───────┴───────┴──────────────┘
6-bit 5-bit 5-bit 16-bit

### 2. Instruction Set

| Instruction | Format | Opcode | Funct | Operation |
|-------------|--------|--------|-------|-----------|
| ADD         | R      | 0x00   | 0x20  | rd = rs + rt |
| SUB         | R      | 0x00   | 0x22  | rd = rs - rt |
| AND         | R      | 0x00   | 0x24  | rd = rs & rt |
| OR          | R      | 0x00   | 0x25  | rd = rs \| rt |
| SLT         | R      | 0x00   | 0x2A  | rd = (rs < rt) ? 1 : 0 |
| ADDI        | I      | 0x08   | -     | rt = rs + imm |
| ANDI        | I      | 0x0C   | -     | rt = rs & imm |
| ORI         | I      | I      | 0x0D  | rt = rs \| imm |
| SLTI        | I      | 0x0A   | -     | rt = (rs < imm) ? 1 : 0 |
| LW          | I      | 0x23   | -     | rt = MEM[rs + imm] |
| SW          | I      | 0x2B   | -     | MEM[rs + imm] = rt |
| BEQ         | I      | 0x04   | -     | if (rs == rt) PC = PC + 4 + (imm << 2) |
| BNE         | I      | 0x05   | -     | if (rs != rt) PC = PC + 4 + (imm << 2) |
| J           | J      | 0x02   | -     | PC = {PC[31:28], address, 2'b00} |

### 3. Control Signals

| Signal | Width | Description |
|--------|-------|-------------|
| RegDst | 1-bit | Register destination (0=rt, 1=rd) |
| RegWrite | 1-bit | Register file write enable |
| ALUSrc | 1-bit | ALU source (0=rt, 1=immediate) |
| MemWrite | 1-bit | Data memory write enable |
| MemRead | 1-bit | Data memory read enable |
| MemtoReg | 1-bit | Write back source (0=ALU, 1=Memory) |
| Branch | 1-bit | Branch instruction |
| Jump | 1-bit | Jump instruction |
| ALUOp | 2-bit | ALU operation control |

### 4. ALU Operations

| ALU Control | Function |
|-------------|----------|
| 0000        | AND      |
| 0001        | OR       |
| 0010        | ADD      |
| 0110        | SUB      |
| 0111        | SLT      |
| 1100        | NOR      |
