# RISC-V Processor

## Introduction
RISC-V (Reduced Instruction Set Computing - V) is an open-source instruction set architecture (ISA) designed to be simple, efficient, and scalable for various applications. Unlike proprietary ISAs such as x86 or ARM, RISC-V is open and free, making it ideal for academic research, embedded systems, and high-performance computing.

This project is an implementation of a basic RISC-V processor that supports a subset of the RISC-V instruction set.

## Features
- Implements a **32-bit RISC-V processor** (RV32I instruction set)
- Supports fundamental instructions like **arithmetic, logical, control flow, and memory access**
- Single-cycle or multi-cycle execution (based on configuration)
- Simple **pipeline structure** (optional)
- **Load/store architecture**
- Can be simulated using **Verilog/VHDL**

## Architecture Overview
The processor follows a standard **von Neumann architecture** with the following components:

### 1. Instruction Fetch (IF)
- Retrieves instructions from memory using the **Program Counter (PC)**
- Increments the PC after each fetch

### 2. Instruction Decode (ID)
- Decodes the fetched instruction
- Identifies the operation and required registers
- Prepares operands for execution

### 3. Execute (EX)
- Performs arithmetic and logical operations using the **ALU (Arithmetic Logic Unit)**
- Computes memory addresses for **load/store operations**
- Evaluates conditions for **branch instructions**

### 4. Memory Access (MEM)
- Accesses **data memory** for load/store instructions
- Fetches values from memory or writes data to memory

### 5. Write Back (WB)
- Writes computed results back to registers

## ALU (Arithmetic Logic Unit)
The ALU is responsible for performing arithmetic and logical operations in the **Execute (EX)** stage of the pipeline. Below is the Verilog implementation of the ALU:

```verilog
module alu (alu_control,inp1,inp2,alu_result,zeroflag);
    input [31:0] inp1,inp2;
    input [3:0] alu_control;
    output reg [31:0] alu_result;
    output reg zeroflag;
    always @(alu_control, inp1, inp2) begin
        case(alu_control)
            4'b0011: alu_result = inp1 & inp2;  // AND operation
            4'b0001: alu_result = inp1 | inp2;  // OR operation
            4'b0010: alu_result = inp1 + inp2;  // Addition
            4'b0100: alu_result = inp1 - inp2;  // Subtraction
            default: alu_result = 32'bx;        // Undefined operation
        endcase
        
        zeroflag = (alu_result == 0) ? 1 : 0;
    end
endmodule
```

## ALU Source MUX (ALUSrcMUX2)
The **ALUSrcMUX2** module is responsible for selecting the second operand input for the ALU. It determines whether the ALU should receive a **register value (d0)** or an **immediate value (d1)** based on a selection signal (s).

```verilog
module alusrcmux2(s, d0, d1, y);
    input logic [31:0] d0, d1;
    input logic s;
    output logic [31:0] y;
    
    always_comb begin
        y = s ? d1 : d0;
    end
endmodule
```

### How ALUSrcMUX2 fits into the pipeline
- The **s signal** is controlled by the instruction decoder and determines whether the second ALU input comes from a register (**d0**) or an immediate value (**d1**).
- If `s = 0`, the ALU receives a value from a **register**.
- If `s = 1`, the ALU receives an **immediate value**, useful for operations like **ADDI (Add Immediate)**.
- The output `y` is then forwarded to the ALU for execution.

## Control Unit
The **Control Unit** is responsible for generating control signals based on the opcode and function fields of the instruction. These signals dictate the behavior of different components in the pipeline.

```verilog
module control(
    input [6:0] funct7,
    input [2:0] funct3,
    input [6:0] opcode,
    output reg [3:0] alu_opera,
    output reg regwrite, memread, memwrite, branch, alusrc, memtoreg
);
logic [1:0] aluop;
    always @(funct3 or funct7 or opcode) begin
        case(opcode)
            51: begin regwrite=1; memread=0; memwrite=0; branch=0; alusrc=0; memtoreg=0; aluop=2'b10; alu_opera=4'b0010; end // add
            19: begin regwrite=1; memread=0; memwrite=0; branch=0; alusrc=1; memtoreg=0; alu_opera=4'b0010; end // addi
            3: begin regwrite=1; memread=1; memwrite=0; branch=0; alusrc=1; memtoreg=1; aluop=2'b00; alu_opera=4'b0010; end // load
            35: begin regwrite=0; memread=0; memwrite=1; branch=0; alusrc=1; memtoreg=1; aluop=2'b00; alu_opera=4'b0010; end // store
            103: begin regwrite=0; memread=0; memwrite=0; branch=1; alusrc=0; memtoreg=0; alu_opera=4'b0100; end // beq
            default: begin regwrite=1'bx; memread=1'bx; memwrite=1'bx; branch=1'bx; alusrc=1'bx; memtoreg=1'bx; alu_opera=4'bx; end
        endcase
    end
endmodule
```

## Supported Instructions
Our implementation currently supports:
- **Arithmetic Operations:** ADD, SUB, MUL, DIV, etc.
- **Logical Operations:** AND, OR, XOR, NOT
- **Memory Operations:** LOAD, STORE
- **Control Flow:** JUMP, BRANCH

## Instruction Memory

The **Instruction Memory** module stores machine code instructions and outputs the instruction corresponding to the current program counter (PC). It is implemented using a 256-word memory array.

```verilog
module instrmemory #(parameter width=32)(clk,addr,inst_out);
	input logic clk;
	input logic [31:0] addr;
	output logic [width-1:0] inst_out;
	reg [width-1:0] imem [255:0];

	always_ff @ (posedge clk)
	begin
		imem[0]=32'b00000000000100000000000110010011;//addi x3,x0,1
		imem[4]=32'b00000000010000101010000100000011;//lw x2,4(x5)
		imem[8]=32'b00000000000100010000001000110011;//add x4,x2,x1
		imem[12]=32'b00000000010000101010010000100011;//sw x4,8(x5)
		imem[16]=32'b00000000000000011000111001100111;//beq x3,x0,3
		imem[20]=32'b00000000001100000000001100010011;//addi x6,x0,3
		imem[24]=32'b00000000011000101010011000100011;//sw x6,12(x5)
		imem[28]=32'b0;
	end

	always_ff@(posedge clk)
	inst_out<=imem[addr];

endmodule
```

## Data Memory

The **Data Memory** module handles memory read and write operations, interfacing with the control unit and ALU.

```verilog
module datamemory(clk,memread,memwrite,a,wd,rd);
    input logic clk;
	input logic memread;
    input logic memwrite;
    input logic [31:0] a;
    input logic [31:0] wd;
    output logic [31:0] rd;
    
    logic [31:0] mem [255:0];
    
    initial
    begin
       if(memread)
            rd <= mem[a];
	 end
    
    always @(posedge clk) begin
       if (memwrite)
            mem[a] = wd;
    end
    
endmodule
```

## Register File

Implements general-purpose registers. A small snippet of the code below. Detailed code is in files section
```verilog
module regfile(
    input clock,
    input reset,
	 input regwrite,
    input [4:0] read_reg_num1,
    input [4:0] read_reg_num2,
    input [4:0] write_reg,
    input [31:0] write_data,
    output [31:0] read_data1,
    output [31:0] read_data2
    
);
```

## Next PC Calculation

Handles branch operations and PC updates.
```verilog 
module nextpc (pc,imm,pcsrc,pc_next);
	input logic pcsrc;
	input logic [31:0] imm,pc;
	output logic [31:0] pc_next;

always_comb
begin
	pc_next=pcsrc?(pc+imm):(pc+32'h00000004);
end
endmodule
```

## Sign Extension

The **Sign Extension** module is used to extend immediate values from a smaller bit-width to a 32-bit representation while preserving the sign.

```verilog
module signextension #(
  parameter InputWidth  = 12,
  parameter OutputWidth = 32
) (
  input  logic signed [InputWidth-1:0]  in_put,
  output logic signed [OutputWidth-1:0] out_put
);

  assign out_put = { {OutputWidth-InputWidth{in_put[InputWidth-1]}}, in_put };

endmodule
```


## Future Enhancements
- Implement **pipelining** for improved performance
- Add support for **floating-point operations**
- Implement a **cache system**
- Extend to **64-bit architecture (RV64I)**

---

## RTL View of Processor
![RTL View](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/1e74a3cd-802c-480f-933b-15328556b143)

## Waveforms of Individual Modules
### 1) ALU
![ALU Waveform](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/8faf2dce-a75b-4fe0-8a10-bcd0e43dfbc7)

### 2) Decoder
![Decoder Waveform](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/aaf4d3a9-0201-436b-b769-19247926da3f)

### 3) Control Unit
![Control Unit Waveform](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/30b448c2-db94-4c6c-8bde-cfd4c0d5a0b3)

### 4) Instruction Memory
![Instruction Memory](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/882f9283-1838-4eeb-b1a7-94532afe5396)

### 5) Register File
![Register File](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/74d72bc4-533b-4bc3-93f6-2d0b230fb32f)

