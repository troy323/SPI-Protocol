# SPI Master-Slave RTL & Layered SystemVerilog Verification

![Language](https://img.shields.io/badge/Language-Verilog%20%7C%20SystemVerilog-blue.svg)
![Methodology](https://img.shields.io/badge/Methodology-CRV%20%7C%20OOP-success.svg)
![Simulation](https://img.shields.io/badge/Simulation-Questa%20%7C%20ModelSim%20%7C%20Vivado-orange.svg)

This repository contains the complete RTL implementation and a constrained-random verification (CRV) environment for a Simplex Serial Peripheral Interface (SPI). The project demonstrates industry-standard practices in digital design and functional verification using Object-Oriented SystemVerilog.

## 📖 Project Overview

The objective of this project is to model a robust SPI communication link and verify its data integrity under randomized stimulus. 

The RTL consists of a custom SPI Master that serializes 12-bit parallel data and generates the necessary timing signals, and an SPI Slave that deserializes the incoming bitstream. The testbench is a fully layered, class-based SystemVerilog environment that automates stimulus generation, drives the design under test (DUT), monitors the interface, and self-checks the results.

---

## 📐 RTL Design Architecture (DUT)

The Design Under Test is partitioned into two main modules synchronized by a generated serial clock.

### 1. SPI Master (`spi_master.v`)
* **Clock Divider:** Generates the SPI serial clock (`sclk`) by dividing the system `clk`. The `sclk` toggles every 10 system clock cycles.
* **State Machine:** Uses a 2-state FSM (`idle`, `send`).
  * **Idle:** Waits for the `newd` (new data) flag. Upon assertion, latches the 12-bit parallel `din`, pulls Chip Select (`cs`) low, and transitions to the `send` state.
  * **Send:** Shifts out the 12-bit payload serially on the `mosi` line, **LSB first**. Once 12 bits are transmitted, it deasserts `cs` (pulls high) and returns to `idle`.

### 2. SPI Slave (`spi_slave.v`)
* **State Machine:** Uses a 2-state FSM (`detect_start`, `read_data`) clocked entirely by the incoming `sclk`.
  * **Detect Start:** Polls the `cs` line. Transitions to `read_data` when `cs` is driven low by the Master.
  * **Read Data:** Samples the `mosi` line on the positive edge of `sclk`, shifting the bits into a 12-bit temporary register. After 12 clock cycles, it asserts the `done` flag and outputs the parallel data on `dout`.

---

## 🧩 Verification Environment Architecture

The testbench is built using a layered Object-Oriented Programming (OOP) methodology to ensure high reusability and clean Inter-Process Communication (IPC).

* **Transaction (`transaction.sv`):** The data blueprint. Contains the 12-bit `din` declared as `rand` for automated constraint-driven stimulus generation. Includes a custom `copy()` method for deep copying objects across mailboxes.
* **Generator (`generator.sv`):** Randomizes the Transaction objects and pushes them into a mailbox for the Driver. Uses SystemVerilog `events` to wait for the Scoreboard to finish checking before generating the next packet.
* **Driver (`driver.sv`):** The pin-wiggler. Pulls transactions from the Generator, handles the DUT reset sequence, and toggles the virtual interface (`vif`) signals to inject the parallel data into the Master. It also forwards the injected data to the Scoreboard via a dedicated mailbox (`mbxds`).
* **Monitor (`monitor.sv`):** The passive observer. Watches the virtual interface and triggers when the Slave's `done` signal goes high. It captures the reconstructed 12-bit `dout` and sends it to the Scoreboard via a mailbox (`mbxms`).
* **Scoreboard (`scoreboard.sv`):** The automated checker. Retrieves the "expected" data from the Driver and the "actual" data from the Monitor. It performs a real-time comparison and logs `[SCO] : DATA MATCHED` or `MISMATCHED`.
* **Environment (`environment.sv`):** The container class that instantiates all testbench components, wires up the mailboxes, and handles the `pre_test()`, `test()`, and `post_test()` execution phases.

---

## 📂 File Structure

```text
SPI_Verification_Project/
├── design/
│   ├── spi_master.v       # Master serializer logic
│   ├── spi_slave.v        # Slave deserializer logic
│   └── top.v              # Top-level DUT wrapper
├── testbench/
│   └── tb.sv              # Top-level module and clock generation
└── README.md
