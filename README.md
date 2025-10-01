
---

# **AXI Protocol UVM Testbench Repository**

---

## **1. AMBA AXI Protocol Overview**

### **1.1 Specification**

The **AMBA AXI (Advanced eXtensible Interface) protocol** is a burst-based, high-performance, on-chip communication protocol designed for efficient data transfer between master and slave devices.

### **1.2 AXI Architecture**

AXI defines **five independent channels**:

1. **Read Address Channel (AR)** – Carries read address and control info.
2. **Read Data Channel (R)** – Transfers data from slave to master; includes data bus (8–1024 bits), read response, and LAST signal.
3. **Write Address Channel (AW)** – Carries write address and control info.
4. **Write Data Channel (W)** – Transfers data from master to slave; includes data bus, byte lane strobe, and LAST signal.
5. **Write Response Channel (B)** – Slave confirms write completion to master.

**Key Features:**

* Supports **address issued ahead of data**.
* Supports **multiple outstanding transactions**.
* Supports **out-of-order transaction completion**.

### **1.3 Handshake Mechanism**

Each channel uses **VALID** and **READY** signals:

* **VALID:** Sender indicates data is ready.
* **READY:** Receiver indicates it can accept data.
* **Transfer occurs only when both signals are high.**

### **1.4 AXI Transaction Dependencies**

**Read Transaction:**

* Address Phase (ARVALID & ARREADY)
* Data Phase (RVALID & RREADY)

**Write Transaction:**

* Address & Data Phase (AWVALID, WVALID, AWREADY, WREADY)
* Write Response Phase (BVALID & BREADY)

**Key Rules:**

* Write response must follow the last write data.
* Read data must follow its read address.
* Channels are mostly independent, but handshakes must respect VALID & READY rules.

### **1.5 AXI Ordering Model**

* **AXI ID** tracks transaction order.
* Transactions with the same ID must arrive at the peripheral in issued order.
* Ensures data integrity and proper ordering across master, slave, and interconnect.

### **1.6 Typical System Topologies**

1. Shared address & data bus
2. Shared address bus + multiple data buses
3. Multilayer interconnect

### **1.7 Register Slices**

Small registers inserted in AXI channels to break long signal paths. Adds **1 clock cycle latency**, but improves timing and clock frequency.

---

## **2. AXI Interface (Virtual Interface)**

**File:** `src/axi_if.sv`

```systemverilog
interface axi_if #(parameter ADDR_WIDTH = 32,
                   parameter DATA_WIDTH = 32,
                   parameter ID_WIDTH   = 4)
                  (input logic aclk, input logic aresetn);
  // Write Address Channel
  logic [ADDR_WIDTH-1:0] awaddr;
  logic [ID_WIDTH-1:0]   awid;
  logic                   awvalid;
  logic                   awready;

  // Write Data Channel
  logic [DATA_WIDTH-1:0] wdata;
  logic [DATA_WIDTH/8-1:0] wstrb;
  logic                    wvalid;
  logic                    wready;
  logic                    wlast;

  // Write Response Channel
  logic [ID_WIDTH-1:0]   bid;
  logic [1:0]             bresp;
  logic                   bvalid;
  logic                   bready;

  // Read Address Channel
  logic [ADDR_WIDTH-1:0] araddr;
  logic [ID_WIDTH-1:0]   arid;
  logic                   arvalid;
  logic                   arready;

  // Read Data Channel
  logic [DATA_WIDTH-1:0] rdata;
  logic [ID_WIDTH-1:0]   rid;
  logic [1:0]             rresp;
  logic                   rvalid;
  logic                   rready;
  logic                   rlast;
endinterface
```

**Explanation:**

* Defines **all five AXI channels**.
* Parameters allow **customizable address, data, and ID width**.
* This interface will be used in all agents and drivers.

---

## **3. AXI Transaction Classes**

**File:** `src/axi_pkg.sv`

```systemverilog
package axi_pkg;
  import uvm_pkg::*;
  `include "uvm_macros.svh"

  parameter ADDR_WIDTH = 32;
  parameter DATA_WIDTH = 32;
  parameter ID_WIDTH   = 4;

  typedef enum logic [1:0] {FIXED, INCR, WRAP} axi_burst_e;

  class axi_write_trans extends uvm_sequence_item;
    rand logic [ADDR_WIDTH-1:0] addr;
    rand logic [DATA_WIDTH-1:0] data;
    rand logic [ID_WIDTH-1:0] id;
    rand axi_burst_e            burst;
    `uvm_object_utils(axi_write_trans)
    function new(string name = "axi_write_trans");
      super.new(name);
    endfunction
  endclass

  class axi_read_trans extends uvm_sequence_item;
    rand logic [ADDR_WIDTH-1:0] addr;
    rand logic [ID_WIDTH-1:0]   id;
    rand axi_burst_e            burst;
    `uvm_object_utils(axi_read_trans)
    function new(string name = "axi_read_trans");
      super.new(name);
    endfunction
  endclass

endpackage
```

**Explanation:**

* Defines **write and read transactions**.
* `rand` allows **randomization for constrained-random testing**.
* Used by **drivers and sequences** in UVM.

---

## **4. AXI Driver (Master Side)**

**File:** `src/axi_driver.sv`

*Code omitted for brevity; see your original.*

**Explanation:**

* Implements **both write and read transactions**.
* Uses **VALID/READY handshake** properly.
* Supports **dynamic type casting** to handle read/write generically.

---

## **5. AXI Monitor**

*Monitors all channels and sends transactions to scoreboard.*

---

## **6. Scoreboard**

*Logs transactions to console or files; can be extended for functional checking.*

---

## **7. Environment and Test Setup**

*Multiple masters connected to scoreboard; monitors connected to analysis ports.*

---

## **8. Test Sequences**

*Demonstrates write and read transactions; extendable to all master-slave combinations.*

---

## **9. AXI Transaction Outputs**

### **9.1 AXI Write Transactions**

**Master 0 → Slave 0:** This transaction tests the **write operation from Master 0 to Slave 0**, ensuring that data is correctly sent and acknowledged via the write response channel.
![AXI-Waveforms\_page-0004](https://github.com/user-attachments/assets/7ef62b18-eead-4625-bc00-534839255fdc)

**Master 0 → Slave 1:** This transaction verifies **write communication from Master 0 to Slave 1**, checking for proper handshakes and data integrity.
![AXI-Waveforms\_page-0002](https://github.com/user-attachments/assets/64c51a02-9690-452c-8d82-5b12dbbc62b5)

**Master 1 → Slave 0:** Validates **write from Master 1 to Slave 0**, ensuring proper AXI handshake.
![AXI-Waveforms\_page-0003](https://github.com/user-attachments/assets/96a3acf6-fdab-4cd3-a144-ad40fe53c4a4)

**Master 1 → Slave 1:** Tests **write from Master 1 to Slave 1**, confirming correct data transfer.
![AXI-Waveforms\_page-0001](https://github.com/user-attachments/assets/9716e0cf-1d90-4dc0-a3ae-aa42d71916a5)

---

### **9.2 AXI Read Transactions**

**Master 0 → Slave 0:** Verifies **read from Slave 0 by Master 0**, checking correct data and read response signals.
![AXI-Waveforms\_page-0008](https://github.com/user-attachments/assets/101a44bf-ddf6-4ad6-be7c-6b103633979b)

**Master 0 → Slave 1:** Ensures **Master 0 can read from Slave 1** with correct data.
![AXI-Waveforms\_page-0006](https://github.com/user-attachments/assets/dc5f244e-ed0d-47e1-a2b9-4aa82136d3ae)

**Master 1 → Slave 0:** Validates **read from Slave 0 by Master 1**.
![AXI-Waveforms\_page-0007](https://github.com/user-attachments/assets/1f65f9e4-45ec-4347-b8dc-6b2092dcfd9f)

**Master 1 → Slave 1:** Tests **Master 1 reading from Slave 1**.
![AXI-Waveforms\_page-0005](https://github.com/user-attachments/assets/7ec8573f-0c9a-47a3-ad50-3e29d3b7768b)

---

### **9.3 AXI Write-Read Transactions**

**Master 0 → Slave 0:** Combined **write and read** to ensure full AXI functionality.
![AXI-Waveforms\_page-0015](https://github.com/user-attachments/assets/b52f1cb7-628e-4b95-be12-7b106ac75249)

**Master 0 → Slave 1:** Write followed by read from Slave 1.
![AXI-Waveforms\_page-0011](https://github.com/user-attachments/assets/021194cf-ead3-4b45-a2fe-f1f3baf1ec57)

**Master 1 → Slave 0:** Write-read from Master 1 to Slave 0.
![AXI-Waveforms\_page-0013](https://github.com/user-attachments/assets/22d67cbb-a387-4a8f-8332-5e3e6b19c4c7)

**Master 1 → Slave 1:** Write-read from Master 1 to Slave 1.
![AXI-Waveforms\_page-0009](https://github.com/user-attachments/assets/a6b3f261-d1d9-4ac3-9142-0b0c6bcfc421)

---

## **10. AXI Protocol: Applications, Advantages, and Limitations**

### **10.1 Applications**

* System-on-Chip (SoC) designs
* High-performance embedded systems
* FPGA designs
* Memory-mapped peripherals
* Multicore and multi-master architectures

### **10.2 Advantages**

* High throughput and burst transfers
* Flexible VALID/READY handshake
* Out-of-order transaction completion
* Supports multiple masters and slaves
* Separate read/write channels
* ID-based ordering
* Industry-wide support

### **10.3 Limitations**

* Complexity with five channels
* Higher resource usage
* Timing constraints for high-speed designs
* Steep learning curve
* Not ideal for low-speed simple peripherals

### **10.4 Summary**

The AXI protocol is a high-performance, burst-based interface suitable for SoCs, FPGAs, and multicore systems. It enables **parallel read/write operations, high throughput, and flexible transaction ordering**, but is **more complex and resource-intensive** compared to simpler protocols.

---

