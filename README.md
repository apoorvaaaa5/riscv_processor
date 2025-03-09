RISC-V Processor

Introduction

RISC-V (Reduced Instruction Set Computing - V) is an open-source instruction set architecture (ISA) designed to be simple, efficient, and scalable for various applications. Unlike proprietary ISAs such as x86 or ARM, RISC-V is open and free, making it ideal for academic research, embedded systems, and high-performance computing.

This project is an implementation of a basic RISC-V processor that supports a subset of the RISC-V instruction set.

Features

Implements a 32-bit RISC-V processor (RV32I instruction set)

Supports fundamental instructions like arithmetic, logical, control flow, and memory access

Single-cycle or multi-cycle execution (based on configuration)

Simple pipeline structure (optional)

Load/store architecture

Can be simulated using Verilog/VHDL

Architecture Overview

The processor follows a standard von Neumann architecture with the following components:

1. Instruction Fetch (IF)

Retrieves instructions from memory using the Program Counter (PC)

Increments the PC after each fetch

2. Instruction Decode (ID)

Decodes the fetched instruction

Identifies the operation and required registers

Prepares operands for execution

3. Execute (EX)

Performs arithmetic and logical operations using the ALU (Arithmetic Logic Unit)

Computes memory addresses for load/store operations

Evaluates conditions for branch instructions

4. Memory Access (MEM)

Accesses data memory for load/store instructions

Fetches values from memory or writes data to memory

5. Write Back (WB)

Writes computed results back to registers

ALU (Arithmetic Logic Unit)

The ALU is responsible for performing arithmetic and logical operations in the Execute (EX) stage of the pipeline. Below is the Verilog implementation of the ALU:

module alu (alu_control,inp1,inp2,alu_result,zeroflag);
    input [31:0] inp1,inp2;
    input[3:0] alu_control;
    output reg [31:0] alu_result;
    output reg zeroflag;
    always@(alu_control,inp1,inp2)
    begin
        case(alu_control)
        4'b0011: alu_result = inp1 & inp2;  // AND operation
        4'b0001: alu_result = inp1 | inp2;  // OR operation
        4'b0010: alu_result = inp1 + inp2;  // Addition
        4'b0100: alu_result = inp1 - inp2;  // Subtraction
        default: alu_result= 32'bx;         // Undefined operation
        endcase

        if (alu_result == 0)
            zeroflag = 1; // Set zero flag if result is zero
        else
            zeroflag = 0;
    end
endmodule

ALU Source MUX (ALUSrcMUX2)

The ALUSrcMUX2 module is responsible for selecting the second operand input for the ALU. It determines whether the ALU should receive a register value (d0) or an immediate value (d1) based on a selection signal (s).

module alusrcmux2(s,d0,d1,y);
	input logic [31:0] d0, d1;
	input logic s;
	output logic [31:0] y;

always_comb
begin
	y = s ? d1 : d0;
end
endmodule

How ALUSrcMUX2 fits into the pipeline

The s signal is controlled by the instruction decoder and determines whether the second ALU input comes from a register (d0) or an immediate value (d1).

If s = 0, the ALU receives a value from a register.

If s = 1, the ALU receives an immediate value, which is useful for operations like ADDI (Add Immediate).

The output y is then forwarded to the ALU for execution.

Control Unit

The Control Unit is responsible for generating control signals based on the opcode and function fields of the instruction. These signals dictate the behavior of different components in the pipeline.

module control(
    input [6:0] funct7,
    input [2:0] funct3,
    input [6:0] opcode,
    output reg [3:0] alu_opera,
    output reg regwrite,memread,memwrite,branch,alusrc,memtoreg
);
logic [1:0] aluop;
    always @(funct3 or funct7 or opcode)
    begin
	 case(opcode)
	 51:begin regwrite=1; memread=0; memwrite=0; branch=0; alusrc=0; memtoreg=0; aluop=2'b10; alu_opera=4'b0010;end //add
    19:begin regwrite=1; memread=0; memwrite=0; branch=0; alusrc=1; memtoreg=0; alu_opera=4'b0010;end //addi
    3:begin  regwrite=1; memread=1; memwrite=0; branch=0; alusrc=1; memtoreg=1; aluop=2'b00; alu_opera=4'b0010;end //load
    35:begin regwrite=0; memread=0; memwrite=1; branch=0; alusrc=1; memtoreg=1; aluop=2'b00; alu_opera=4'b0010;end //store
	 103:begin regwrite=0; memread=0; memwrite=0; branch=1; alusrc=0; memtoreg=0; alu_opera=4'b0100;end //beq
    default:begin regwrite=1'bx; memread=1'bx; memwrite=1'bx; branch=1'bx; alusrc=1'bx; memtoreg=1'bx; alu_opera=4'bx;end
endcase
end
endmodule

How the Control Unit Works

It takes opcode, funct3, and funct7 fields as inputs from the instruction.

Based on these values, it sets control signals that dictate processor behavior:

regwrite: Enables writing to registers.

memread: Signals a memory read operation.

memwrite: Signals a memory write operation.

branch: Indicates a branch instruction.

alusrc: Determines whether the ALU's second operand is a register or immediate value.

memtoreg: Controls data flow from memory to registers.

alu_opera: Specifies the ALU operation.

Supported Instructions

Our implementation currently supports:

Arithmetic Operations: ADD, SUB, MUL, DIV, etc.

Logical Operations: AND, OR, XOR, NOT

Memory Operations: LOAD, STORE

Control Flow: JUMP, BRANCH

Future Enhancements

Implement pipelining for improved performance

Add support for floating-point operations

Implement a cache system

Extend to 64-bit architecture (RV64I)

# riscv_processor
RTL VIEW OF PROCESSOR: 

<img width="791" alt="image" src="https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/1e74a3cd-802c-480f-933b-15328556b143">

<img width="809" alt="Screenshot 2024-01-02 123207" src="https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/0aa5e93e-2025-4f1c-b08e-4db571e00305">

<img width="802" alt="image" src="https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/5093e11e-97af-4132-b65b-fa665213f7d0">

WAVEFORMS OF INDIVIDUAL MODULES:
1) alu
   ![46ea2808-7f23-4bac-9b7c-a9b53ef1198c](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/8faf2dce-a75b-4fe0-8a10-bcd0e43dfbc7)

2) decoder
  ![ceffd19f-5e79-464f-b5d1-478e52998889](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/aaf4d3a9-0201-436b-b769-19247926da3f)

3) control
   ![a3ecb824-069c-4c8a-8410-c6d91ffc0750](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/30b448c2-db94-4c6c-8bde-cfd4c0d5a0b3)

4) instr memory
   ![13507247-3f98-4907-893b-073bb46932bd](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/882f9283-1838-4eeb-b1a7-94532afe5396)

5) reg file
   ![c5f78d1d-76f6-40bb-bcc0-1406b1911caa](https://github.com/apoorvaaaa5/riscv_processor/assets/117642634/74d72bc4-533b-4bc3-93f6-2d0b230fb32f)
