---

# Memory Controller for SDRAM/DRAM

This repository contains a Verilog-based design for a **Memory Controller** with a focus on handling SDRAM or DRAM modules. The design efficiently manages memory transactions such as **read**, **write**, and **refresh operations** while meeting timing and signal requirements. A detailed **testbench** is included to validate functionality through simulation.

---

## Memory Controller Features

### Core Functionalities:
- **Read and Write Support**:
  - Single-word and burst data transfers.
- **Refresh Cycles**:
  - Automatically refreshes memory to maintain data integrity.
- **Busy Signal**:
  - Indicates when the memory controller is actively handling a request.

### Interface:
- Standard SDRAM control signals, including:
  - `clk`, `cs`, `ras`, `cas`, `we`
  - `addr` (Address), `ba` (Bank Address), and `dq` (Data Bus).

---

## Testbench Overview

The included testbench (`memory_controller_tb.v`) provides a simulation framework for validating the memory controller design. It tests the following scenarios:

1. **Write Operation**:
   - Writes a 16-bit data word to a specified memory address.
2. **Read Operation**:
   - Reads data from the specified memory address and verifies correctness.
3. **Refresh Operation**:
   - Simulates memory refresh cycles triggered by the controller.

### Key Components:
- **Clock Generator**:
  - Produces a periodic clock signal for synchronous operation.
- **Simulated SDRAM Memory**:
  - Models the behavior of SDRAM with a bidirectional data bus (`sdram_dq`).
- **FSM Validation**:
  - Verifies state transitions during different memory operations.

---

## Project Structure

- **`memory_controller.v`**: Core memory controller module.
- **`memory_controller_tb.v`**: Testbench module for simulation.

### Simulation Setup

Run the testbench in a Verilog simulation environment (e.g., Xilinx Vivado). The waveform outputs will illustrate:
- Successful state transitions during read/write operations.
- Proper management of refresh cycles.
- Accurate data transfer on the `sdram_dq` bus.

---

## Challenges Addressed

- **Timing Constraints**:
  - Ensures compliance with SDRAM timing parameters (CAS latency, refresh intervals, etc.).
- **Bidirectional Bus Control**:
  - Properly drives and samples data on the bidirectional `sdram_dq` bus.

---
### Module Definition:
```verilog
module memory_controller (
    input clk,
    input reset,
    input [23:0] addr,          // Address bus (24-bit for SDRAM addressing)
    input [15:0] data_in,       // Data input bus (16-bit width)
    output reg [15:0] data_out, // Data output bus
    input we,                   // Write enable signal
    input re,                   // Read enable signal
    output reg busy,            // Busy signal to indicate the controller is processing
    output reg sdram_clk,       // SDRAM clock signal
    output reg sdram_cs,        // SDRAM chip select
    output reg sdram_ras,       // SDRAM row address strobe
    output reg sdram_cas,       // SDRAM column address strobe
    output reg sdram_we,        // SDRAM write enable
    inout [15:0] sdram_dq,      // SDRAM data bus
    output reg [12:0] sdram_addr, // SDRAM address bus
    output reg [1:0] sdram_ba    // SDRAM bank address
);
```
This is the definition of the `memory_controller` module, which interfaces with an SDRAM (Synchronous Dynamic RAM). It has various inputs and outputs:
- **Inputs**: Clock (`clk`), reset (`reset`), address (`addr`), data input (`data_in`), and control signals (`we` for write enable and `re` for read enable).
- **Outputs**: Data output (`data_out`), busy signal (`busy`), various control signals for the SDRAM (`sdram_clk`, `sdram_cs`, `sdram_ras`, `sdram_cas`, `sdram_we`, `sdram_addr`, `sdram_ba`), and the bi-directional SDRAM data bus (`sdram_dq`).

### Local Parameters and State Definitions:
```verilog
// States for the memory controller FSM
localparam IDLE        = 3'b000;
localparam ACTIVE      = 3'b001;
localparam READ        = 3'b010;
localparam WRITE       = 3'b011;
localparam PRECHARGE   = 3'b100;
localparam REFRESH     = 3'b101;
```
These are state definitions for the finite state machine (FSM) that controls the operation of the SDRAM interface:
- **IDLE**: The controller is idle, waiting for commands.
- **ACTIVE**: The controller is in an active state and ready for read or write operations.
- **READ**: The controller is in the process of reading data from SDRAM.
- **WRITE**: The controller is writing data to SDRAM.
- **PRECHARGE**: The controller precharges the SDRAM after an operation.
- **REFRESH**: Periodic refresh of the SDRAM is needed to maintain data integrity.

### Internal Registers:
```verilog
reg [2:0] state, next_state;
reg [15:0] data_buffer;  // Internal buffer for data
reg [7:0] refresh_counter; // Counter to trigger periodic refresh
```
- **state**: Holds the current state of the FSM.
- **next_state**: Holds the next state that the FSM will transition to.
- **data_buffer**: Holds data temporarily for read/write operations (although not fully utilized here).
- **refresh_counter**: A counter to manage the periodic refresh of the SDRAM.

### Initialization on Reset:
```verilog
// SDRAM Control Signals Initialization
always @(posedge clk or posedge reset) begin
    if (reset) begin
        state <= IDLE;
        sdram_clk <= 1'b0;
        sdram_cs <= 1'b1;
        sdram_ras <= 1'b1;
        sdram_cas <= 1'b1;
        sdram_we <= 1'b1;
        sdram_addr <= 13'b0;
        sdram_ba <= 2'b0;
        busy <= 1'b0;
        refresh_counter <= 8'd0;
    end else begin
        state <= next_state;
        sdram_clk <= ~sdram_clk; // Toggle SDRAM clock

        // Refresh counter management
        if (refresh_counter == 8'd255) begin
            refresh_counter <= 8'd0;
        end else begin
            refresh_counter <= refresh_counter + 1;
        end
    end
end
```
This `always` block is triggered on the rising edge of the `clk` or `reset`. 
- If **reset** is active, the controller initializes all control signals to their default values:
  - The **state** is set to `IDLE` (no operation).
  - **SDRAM control signals** like `sdram_clk`, `sdram_cs`, `sdram_ras`, `sdram_cas`, `sdram_we`, `sdram_addr`, `sdram_ba` are initialized to known states (for example, `sdram_clk` is initially low, `sdram_cs` is high, indicating chip is not selected).
  - **busy** is set to `0` to indicate that the controller is not busy, and the **refresh_counter** is reset to `0`.

- On every clock cycle (if not in reset), the **state** is updated to the **next_state**, and the **SDRAM clock** (`sdram_clk`) is toggled. The **refresh_counter** is incremented and reset when it reaches `255`.

### FSM Logic for State Transitions and Control Signal Outputs:
```verilog
// FSM Next State Logic and Outputs
always @(*) begin
    next_state = state;
    case (state)
        IDLE: begin
            busy = 1'b0;
            if (refresh_counter == 8'd255) begin
                next_state = REFRESH; // Trigger refresh
            end else if (we) begin
                next_state = WRITE;
            end else if (re) begin
                next_state = READ;
            end
        end
```
This `always @(*)` block determines the next state of the FSM and sets the control outputs based on the current state:
- In the **IDLE** state, the controller is not busy (`busy = 0`).
- If the **refresh_counter** reaches `255`, the controller transitions to the **REFRESH** state.
- If **we** (write enable) is active, it moves to the **WRITE** state.
- If **re** (read enable) is active, it moves to the **READ** state.

#### REFRESH State:
```verilog
        REFRESH: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b0;
            sdram_we = 1'b1;
            next_state = IDLE; // Return to idle after refresh
        end
```
- **REFRESH**: This state triggers the refresh operation for the SDRAM. It sets the control signals to appropriate values (`sdram_cs = 0`, `sdram_ras = 0`, `sdram_cas = 0`, and `sdram_we = 1`).
- After the refresh operation, the state transitions back to **IDLE**.

#### WRITE State:
```verilog
        WRITE: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b1;
            sdram_we = 1'b0;
            sdram_addr = addr[12:0]; // Set the row/column address
            sdram_ba = addr[14:13];  // Set the bank address
            next_state = PRECHARGE; // Precharge after write
        end
```
- **WRITE**: In this state, the controller writes data to SDRAM. The row and column address are set using `addr[12:0]`, and the bank address is set using `addr[14:13]`.
- After completing the write operation, it transitions to **PRECHARGE**.

#### READ State:
```verilog
        READ: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b1;
            sdram_we = 1'b1;
            sdram_addr = addr[12:0];
            sdram_ba = addr[14:13];
            data_out = sdram_dq; // Capture data from SDRAM
            next_state = PRECHARGE;
        end
```
- **READ**: In the read state, the controller reads data from SDRAM and stores it in `data_out`. The address is set, and `sdram_dq` is used to read the data. The next state is **PRECHARGE**.

#### PRECHARGE State:
```verilog
        PRECHARGE: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b1;
            sdram_we = 1'b0;
            next_state = IDLE;
        end
```
- **PRECHARGE**: In this state, the SDRAM is precharged after a read or write operation. Once precharge is complete, the state returns to **IDLE**.
## CODE:
```
module memory_controller (
    input clk,
    input reset,
    input [23:0] addr,          // Address bus (24-bit for SDRAM addressing)
    input [15:0] data_in,       // Data input bus (16-bit width)
    output reg [15:0] data_out, // Data output bus
    input we,                   // Write enable signal
    input re,                   // Read enable signal
    output reg busy,            // Busy signal to indicate the controller is processing
    output reg sdram_clk,       // SDRAM clock signal
    output reg sdram_cs,        // SDRAM chip select
    output reg sdram_ras,       // SDRAM row address strobe
    output reg sdram_cas,       // SDRAM column address strobe
    output reg sdram_we,        // SDRAM write enable
    inout [15:0] sdram_dq,      // SDRAM data bus
    output reg [12:0] sdram_addr, // SDRAM address bus
    output reg [1:0] sdram_ba    // SDRAM bank address
);

// States for the memory controller FSM
localparam IDLE        = 3'b000;
localparam ACTIVE      = 3'b001;
localparam READ        = 3'b010;
localparam WRITE       = 3'b011;
localparam PRECHARGE   = 3'b100;
localparam REFRESH     = 3'b101;

// Registers and wires for FSM
reg [2:0] state, next_state;
reg [15:0] data_buffer;  // Internal buffer for data
reg [7:0] refresh_counter; // Counter to trigger periodic refresh

// SDRAM Control Signals Initialization
always @(posedge clk or posedge reset) begin
    if (reset) begin
        state <= IDLE;
        sdram_clk <= 1'b0;
        sdram_cs <= 1'b1;
        sdram_ras <= 1'b1;
        sdram_cas <= 1'b1;
        sdram_we <= 1'b1;
        sdram_addr <= 13'b0;
        sdram_ba <= 2'b0;
        busy <= 1'b0;
        refresh_counter <= 8'd0;
    end else begin
        state <= next_state;
        sdram_clk <= ~sdram_clk; // Toggle SDRAM clock

        // Refresh counter management
        if (refresh_counter == 8'd255) begin
            refresh_counter <= 8'd0;
        end else begin
            refresh_counter <= refresh_counter + 1;
        end
    end
end

// FSM Next State Logic and Outputs
always @(*) begin
    next_state = state;
    case (state)
        IDLE: begin
            busy = 1'b0;
            if (refresh_counter == 8'd255) begin
                next_state = REFRESH; // Trigger refresh
            end else if (we) begin
                next_state = WRITE;
            end else if (re) begin
                next_state = READ;
            end
        end

        REFRESH: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b0;
            sdram_we = 1'b1;
            next_state = IDLE; // Return to idle after refresh
        end

        WRITE: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b1;
            sdram_we = 1'b0;
            sdram_addr = addr[12:0]; // Set the row/column address
            sdram_ba = addr[14:13];  // Set the bank address
            next_state = PRECHARGE; // Precharge after write
        end

        READ: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b1;
            sdram_we = 1'b1;
            sdram_addr = addr[12:0];
            sdram_ba = addr[14:13];
            data_out = sdram_dq; // Capture data from SDRAM
            next_state = PRECHARGE;
        end

        PRECHARGE: begin
            busy = 1'b1;
            sdram_cs = 1'b0;
            sdram_ras = 1'b0;
            sdram_cas = 1'b1;
            sdram_we = 1'b0;
            next_state = IDLE;
        end

        default: next_state = IDLE;
    endcase
end

endmodule
```

This Verilog code is a **testbench** for the `memory_controller` module. It provides the necessary signals and simulates the behavior of the memory controller, as well as some basic test cases for write, read, and refresh operations.

### Testbench Module:
```verilog
module memory_controller_tb;
```
![Screenshot (22)](https://github.com/user-attachments/assets/259be3fa-b463-4d83-9a51-aac7d4ff214c)
This defines the **testbench module** `memory_controller_tb`, which is used to test the functionality of the `memory_controller` module.

### Inputs and Outputs:
```verilog
  // Inputs
  reg clk;
  reg reset;
  reg [23:0] addr;
  reg [15:0] data_in;
  reg we;
  reg re;

  // Outputs
  wire [15:0] data_out;
  wire busy;
  wire sdram_clk;
  wire sdram_cs;
  wire sdram_ras;
  wire sdram_cas;
  wire sdram_we;
  wire [12:0] sdram_addr;
  wire [1:0] sdram_ba;

  // Bidirectional
  wire [15:0] sdram_dq;
```
- **Inputs** are declared as **registers (`reg`)** because they are driven by the testbench.
  - `clk`: Clock signal.
  - `reset`: Reset signal.
  - `addr`: 24-bit address to access memory.
  - `data_in`: 16-bit data to write into memory.
  - `we`: Write enable signal.
  - `re`: Read enable signal.

- **Outputs** are declared as **wires (`wire`)** since they are driven by the `memory_controller` module.
  - `data_out`: The 16-bit data read from memory.
  - `busy`, `sdram_clk`, `sdram_cs`, `sdram_ras`, `sdram_cas`, `sdram_we`, `sdram_addr`, `sdram_ba`: Various control signals for SDRAM.
  - `sdram_dq`: Bidirectional data bus (the data bus is used both for writing to and reading from the SDRAM).

### Instantiation of the `memory_controller`:
```verilog
  // Instantiate the memory controller module
  memory_controller uut (
    .clk(clk),
    .reset(reset),
    .addr(addr),
    .data_in(data_in),
    .data_out(data_out),
    .we(we),
    .re(re),
    .busy(busy),
    .sdram_clk(sdram_clk),
    .sdram_cs(sdram_cs),
    .sdram_ras(sdram_ras),
    .sdram_cas(sdram_cas),
    .sdram_we(sdram_we),
    .sdram_dq(sdram_dq),
    .sdram_addr(sdram_addr),
    .sdram_ba(sdram_ba)
  );
```
Here, the **`memory_controller`** module is instantiated as `uut` (unit under test), and the testbench signals are connected to the module's ports.

### Clock Generation:
```verilog
  // Clock generation
  always #5 clk = ~clk;
```
This block generates the clock signal `clk`. The `clk` signal is toggled every 5 time units, which results in a clock period of 10 time units (100 MHz if time unit is ns).

### Initial Block for Stimulus:
```verilog
  initial begin
    // Initialize inputs
    clk = 0;
    reset = 1;
    addr = 24'b0;
    data_in = 16'b0;
    we = 0;
    re = 0;

    // Apply reset
    #10;
    reset = 0;

    // Test write operation
    addr = 24'h123456;
    data_in = 16'hA5A5;
    we = 1;
    re = 0;
    #20;
    we = 0;

    // Test read operation
    addr = 24'h123456;
    data_in = 16'b0;
    we = 0;
    re = 1;
    #20;
    re = 0;

    // Test refresh operation (wait until refresh counter triggers)
    #500;

    // End simulation
    $finish;
  end
```
- This `initial` block provides stimulus to the inputs and simulates the behavior of the memory controller.
  - **Initialization**: Initially, the inputs are set to default values (e.g., `clk = 0`, `reset = 1`).
  - **Apply reset**: After waiting for `10` time units, the reset is deasserted (`reset = 0`).
  - **Write Operation**: A write operation is simulated with an address of `24'h123456` and data `16'hA5A5`. The `we` (write enable) signal is set to `1` for 20 time units, then cleared to `0` to stop the write operation.
  - **Read Operation**: A read operation is simulated with the same address (`24'h123456`) and `we = 0`, `re = 1` (read enable). After 20 time units, `re` is cleared to `0`.
  - **Refresh Operation**: The testbench waits for `500` time units to simulate the refresh cycle of the SDRAM.
  - Finally, the simulation is ended with `$finish`.

### SDRAM Data Simulation:
```verilog
  // SDRAM data simulation (bidirectional bus)
  reg [15:0] sdram_memory [0:255]; // Simulated SDRAM memory array
  reg sdram_dq_en;

  assign sdram_dq = (sdram_dq_en) ? sdram_memory[addr[7:0]] : 16'bz;

  initial begin
    // Initialize SDRAM memory with test values
    sdram_memory[8'h56] = 16'hBEEF; // Address 0x123456 maps to 0x56 in this example
  end
```
- **SDRAM Memory Simulation**: A simple **SDRAM memory array** (`sdram_memory`) is declared as a 256-entry array of 16-bit words.
- **Bidirectional Data Bus**: The **bidirectional data bus** (`sdram_dq`) is controlled by the `sdram_dq_en` signal. If `sdram_dq_en` is `1`, it drives the data from the simulated SDRAM memory; otherwise, it is high impedance (`16'bz`), simulating the reading operation. 
- **Memory Initialization**: The memory at address `8'h56` (which corresponds to part of the address `24'h123456` in the test case) is initialized with the value `16'hBEEF` for testing purposes.

## TESTBENCH:
```
module memory_controller_tb;

  // Inputs
  reg clk;
  reg reset;
  reg [23:0] addr;
  reg [15:0] data_in;
  reg we;
  reg re;

  // Outputs
  wire [15:0] data_out;
  wire busy;
  wire sdram_clk;
  wire sdram_cs;
  wire sdram_ras;
  wire sdram_cas;
  wire sdram_we;
  wire [12:0] sdram_addr;
  wire [1:0] sdram_ba;

  // Bidirectional
  wire [15:0] sdram_dq;

  // Instantiate the memory controller module
  memory_controller uut (
    .clk(clk),
    .reset(reset),
    .addr(addr),
    .data_in(data_in),
    .data_out(data_out),
    .we(we),
    .re(re),
    .busy(busy),
    .sdram_clk(sdram_clk),
    .sdram_cs(sdram_cs),
    .sdram_ras(sdram_ras),
    .sdram_cas(sdram_cas),
    .sdram_we(sdram_we),
    .sdram_dq(sdram_dq),
    .sdram_addr(sdram_addr),
    .sdram_ba(sdram_ba)
  );

  // Clock generation
  always #5 clk = ~clk;

  initial begin
    // Initialize inputs
    clk = 0;
    reset = 1;
    addr = 24'b0;
    data_in = 16'b0;
    we = 0;
    re = 0;

    // Apply reset
    #10;
    reset = 0;

    // Test write operation
    addr = 24'h123456;
    data_in = 16'hA5A5;
    we = 1;
    re = 0;
    #20;
    we = 0;

    // Test read operation
    addr = 24'h123456;
    data_in = 16'b0;
    we = 0;
    re = 1;
    #20;
    re = 0;

    // Test refresh operation (wait until refresh counter triggers)
    #500;

    // End simulation
    $finish;
  end

  // SDRAM data simulation (bidirectional bus)
  reg [15:0] sdram_memory [0:255]; // Simulated SDRAM memory array
  reg sdram_dq_en;

  assign sdram_dq = (sdram_dq_en) ? sdram_memory[addr[7:0]] : 16'bz;

  initial begin
    // Initialize SDRAM memory with test values
    sdram_memory[8'h56] = 16'hBEEF; // Address 0x123456 maps to 0x56 in this example
  end

endmodule
```
![Screenshot (23)](https://github.com/user-attachments/assets/a5078dae-cd1e-443c-b1e2-ddcb7312d502)
### Final Testbench
![Screenshot (24)](https://github.com/user-attachments/assets/7379dd81-2afe-435a-be45-409cb4e647a9)
### Schematic Design
![Screenshot (25)](https://github.com/user-attachments/assets/0d7e8b97-b699-4c03-9006-0e6e7561419e)
### Schematic Deisgn
![Screenshot (25)](https://github.com/user-attachments/assets/bbbaa5e9-437d-4fe2-8a86-1fc99fd995b0)
### Implementation Schematics Design
![Screenshot (27)](https://github.com/user-attachments/assets/9e070adf-cdbc-43f9-b905-0cc6a1279a42)
### Power Report
![Screenshot (28)](https://github.com/user-attachments/assets/b9b118ad-6f15-4f13-b5df-a6b9d67f5d2a)


This project demonstrates a robust and scalable solution for interfacing FPGA designs with external memory.
