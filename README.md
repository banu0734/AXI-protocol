
# AXI Protocol UVM Testbench Repository

## 1. AMBA AXI Protocol Overview

### Specification

The **AMBA AXI (Advanced eXtensible Interface) protocol** is a burst-based, high-performance, on-chip communication protocol designed for efficient data transfer between master and slave devices.

### AXI Architecture

AXI defines **five independent channels**:

1. **Read Address Channel (AR)** – Carries read address and control info.
2. **Read Data Channel (R)** – Transfers data from slave to master. Includes data bus (8–1024 bits), read response, and LAST signal.
3. **Write Address Channel (AW)** – Carries write address and control info.
4. **Write Data Channel (W)** – Transfers data from master to slave. Includes data bus, byte lane strobe, and LAST signal.
5. **Write Response Channel (B)** – Slave confirms write completion to master.

**Key Features:**

* Supports **address issued ahead of data**.
* Supports **multiple outstanding transactions**.
* Supports **out-of-order transaction completion**.

---

### Handshake Mechanism

Each channel uses **VALID** and **READY** signals:

* **VALID**: Sender indicates data is ready.
* **READY**: Receiver indicates it can accept data.
* **Transfer occurs only when both signals are high.**

---

### AXI Transaction Dependencies

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

---

### AXI Ordering Model

* **AXI ID** tracks transaction order.
* Transactions with the same ID must arrive at the peripheral in issued order.
* Ensures data integrity and proper ordering across master, slave, and interconnect.

---

### Typical System Topologies

1. Shared address & data bus
2. Shared address bus + multiple data buses
3. Multilayer interconnect

---

### Register Slices

Small registers inserted in AXI channels to break long signal paths. Adds **1 clock cycle latency**, but improves timing and clock frequency.

---


# **1. AXI Interface (Virtual Interface)**

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

# **2. AXI Transaction Classes**

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

# **3. AXI Driver (Master Side)**

**File:** `src/axi_driver.sv`

```systemverilog
class axi_driver #(type T=axi_pkg::axi_write_trans) extends uvm_driver #(T);
  `uvm_component_utils(axi_driver)

  virtual axi_if vif;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  task run_phase(uvm_phase phase);
    T tr;
    forever begin
      seq_item_port.get_next_item(tr);

      if($cast(tr, axi_pkg::axi_write_trans)) begin
        // Write Transaction
        @(posedge vif.aclk);
        vif.awaddr  <= tr.addr;
        vif.awid    <= tr.id;
        vif.awvalid <= 1;
        wait(vif.awready);
        vif.awvalid <= 0;

        vif.wdata  <= tr.data;
        vif.wstrb  <= '1;
        vif.wvalid <= 1;
        vif.wlast  <= 1;
        wait(vif.wready);
        vif.wvalid <= 0;

        wait(vif.bvalid);
        vif.bready <= 1;
        @(posedge vif.aclk);
        vif.bready <= 0;

      end else if($cast(tr, axi_pkg::axi_read_trans)) begin
        // Read Transaction
        @(posedge vif.aclk);
        vif.araddr  <= tr.addr;
        vif.arid    <= tr.id;
        vif.arvalid <= 1;
        wait(vif.arready);
        vif.arvalid <= 0;

        wait(vif.rvalid);
        vif.rready <= 1;
        @(posedge vif.aclk);
        vif.rready <= 0;
      end

      seq_item_port.item_done();
    end
  endtask
endclass
```

**Explanation:**

* Implements **both write and read transactions**.
* Uses **VALID/READY handshake** properly.
* Supports **dynamic type casting** to handle read/write generically.

---

# **4. AXI Monitor**

**File:** `src/axi_monitor.sv`

```systemverilog
class axi_monitor #(type T=axi_pkg::axi_write_trans) extends uvm_monitor;
  `uvm_component_utils(axi_monitor)

  virtual axi_if vif;
  uvm_analysis_port #(T) ap;

  function new(string name, uvm_component parent);
    super.new(name, parent);
    ap = new("ap", this);
  endfunction

  task run_phase(uvm_phase phase);
    T tr;
    forever begin
      @(posedge vif.aclk);

      if($cast(tr, axi_pkg::axi_write_trans)) begin
        if(vif.awvalid && vif.awready) begin
          tr.addr = vif.awaddr;
          tr.data = vif.wdata;
          tr.id   = vif.awid;
          ap.write(tr);
        end
      end else if($cast(tr, axi_pkg::axi_read_trans)) begin
        if(vif.arvalid && vif.arready) begin
          tr.addr = vif.araddr;
          tr.id   = vif.arid;
          ap.write(tr);
        end
      end
    end
  endtask
endclass
```

**Explanation:**

* Monitors **all AXI channels**.
* Sends observed transactions to **scoreboard** for verification.

---

# **5. Scoreboard**

**File:** `src/axi_scoreboard.sv`

```systemverilog
class axi_scoreboard #(type T=axi_pkg::axi_write_trans) extends uvm_scoreboard;
  `uvm_component_utils(axi_scoreboard)

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  function void write(T tr);
    if($cast(tr, axi_pkg::axi_write_trans)) begin
      `uvm_info("SCOREBOARD", $sformatf("Write observed: Addr=%h, Data=%h, ID=%0d", tr.addr, tr.data, tr.id), UVM_LOW)
    end else if($cast(tr, axi_pkg::axi_read_trans)) begin
      `uvm_info("SCOREBOARD", $sformatf("Read observed: Addr=%h, ID=%0d", tr.addr, tr.id), UVM_LOW)
    end
  endfunction
endclass
```

**Explanation:**

* Logs transactions to **console or log files**.
* Can be extended for **functional checking** with expected data comparison.

---

# **6. Environment and Test Setup**

**File:** `src/axi_env.sv`

```systemverilog
class axi_env extends uvm_env;
  `uvm_component_utils(axi_env)

  axi_master_agent #(axi_pkg::axi_write_trans) master0_agent, master1_agent;
  axi_scoreboard #(axi_pkg::axi_write_trans) sb;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  function void build_phase(uvm_phase phase);
    master0_agent = axi_master_agent::type_id::create("master0_agent", this);
    master1_agent = axi_master_agent::type_id::create("master1_agent", this);
    sb = axi_scoreboard::type_id::create("scoreboard", this);
  endfunction

  function void connect_phase(uvm_phase phase);
    master0_agent.monitor.ap.connect(sb.analysis_export);
    master1_agent.monitor.ap.connect(sb.analysis_export);
  endfunction
endclass
```

**Explanation:**

* Multiple masters and one scoreboard.
* Connects **monitors to scoreboard** for checking all transactions.

---

# **7. Test Sequences**

**File:** `src/axi_test.sv`

```systemverilog
class axi_test extends uvm_test;
  `uvm_component_utils(axi_test)

  axi_env env;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  function void build_phase(uvm_phase phase);
    env = axi_env::type_id::create("env", this);
  endfunction

  task run_phase(uvm_phase phase);
    axi_pkg::axi_write_trans w_tr;
    axi_pkg::axi_read_trans  r_tr;

    // Master 0 -> Slave 0 Write
    w_tr = axi_pkg::axi_write_trans::type_id::create("w_tr");
    w_tr.addr = 32'h1000_0000;
    w_tr.data = 32'hDEAD_BEEF;
    w_tr.id   = 4'b0000;
    env.master0_agent.driver.seq_item_port.put(w_tr);

    // Master 1 -> Slave 1 Read
    r_tr = axi_pkg::axi_read_trans::type_id::create("r_tr");
    r_tr.addr = 32'h2000_0000;
    r_tr.id   = 4'b0001;
    env.master1_agent.driver.seq_item_port.put(r_tr);

    // Additional transactions for all master-slave combinations can be added here
  endtask
endclass
```

**Explanation:**

* Demonstrates **Write and Read** transactions.
* Extendable for **all combinations and write-read sequences**.
* Can randomize addresses/data for **constrained-random testing**.

---

# **AXI Transaction Outputs**

## **1. AXI Write Transactions**

### **Master 0 → Slave 0:** This transaction tests the **write operation from Master 0 to Slave 0**, ensuring that data is correctly sent and acknowledged via the write response channel.
![AXI-Waveforms_page-0004](https://github.com/user-attachments/assets/7ef62b18-eead-4625-bc00-534839255fdc)


### **Master 0 → Slave 1:** This transaction verifies **write communication from Master 0 to Slave 1**, checking for proper handshakes and data integrity.
![AXI-Waveforms_page-0002](https://github.com/user-attachments/assets/64c51a02-9690-452c-8d82-5b12dbbc62b5)


### **Master 1 → Slave 0:** This transaction validates **write from Master 1 to Slave 0**, ensuring the AXI handshake protocol is properly followed.
![AXI-Waveforms_page-0003](https://github.com/user-attachments/assets/96a3acf6-fdab-4cd3-a144-ad40fe53c4a4)


### **Master 1 → Slave 1:** This transaction tests **write from Master 1 to Slave 1**, confirming correct data transfer and write response.
![AXI-Waveforms_page-0001](https://github.com/user-attachments/assets/9716e0cf-1d90-4dc0-a3ae-aa42d71916a5)

---

## **2. AXI Read Transactions**

**Master 0 → Slave 0:** This transaction verifies **read from Slave 0 by Master 0**, checking that data is correctly returned and read response signals are valid.
![AXI-Waveforms_page-0008](https://github.com/user-attachments/assets/101a44bf-ddf6-4ad6-be7c-6b103633979b)


**Master 0 → Slave 1:** This transaction ensures **Master 0 can read from Slave 1** with correct data and timing.
![AXI-Waveforms_page-0006](https://github.com/user-attachments/assets/dc5f244e-ed0d-47e1-a2b9-4aa82136d3ae)


**Master 1 → Slave 0:** This transaction validates **read from Slave 0 by Master 1**, confirming correct data and adherence to read handshake rules.
![AXI-Waveforms_page-0007](https://github.com/user-attachments/assets/1f65f9e4-45ec-4347-b8dc-6b2092dcfd9f)


**Master 1 → Slave 1:** This transaction tests **Master 1 reading from Slave 1**, ensuring proper read response and data integrity.
![AXI-Waveforms_page-0005](https://github.com/user-attachments/assets/7ec8573f-0c9a-47a3-ad50-3e29d3b7768b)


---

### **3. AXI Write-Read Transactions**

**Master 0 → Slave 0:** This combined transaction writes data to Slave 0 and immediately reads it back to ensure **full AXI functionality and ordering**.
![AXI-Waveforms_page-0015](https://github.com/user-attachments/assets/b52f1cb7-628e-4b95-be12-7b106ac75249)

---

![AXI-Waveforms_page-0016](https://github.com/user-attachments/assets/9ee9b58d-6b65-4368-bfa1-6f9bbae155e6)


**Master 0 → Slave 1:** This transaction performs **write followed by read from Slave 1 by Master 0**, verifying complete write-read handshake flow.
![AXI-Waveforms_page-0011](https://github.com/user-attachments/assets/021194cf-ead3-4b45-a2fe-f1f3baf1ec57)

---

![AXI-Waveforms_page-0012](https://github.com/user-attachments/assets/6ee55941-261a-4c36-9971-2339783710fc)


**Master 1 → Slave 0:** This transaction tests **write-read from Master 1 to Slave 0**, checking data consistency and AXI protocol compliance.
![AXI-Waveforms_page-0013](https://github.com/user-attachments/assets/22d67cbb-a387-4a8f-8332-5e3e6b19c4c7)

---

![AXI-Waveforms_page-0014](https://github.com/user-attachments/assets/4d259f00-ca3f-464b-a817-550a3c446f6e)


**Master 1 → Slave 1:** This transaction ensures **write-read from Master 1 to Slave 1** is correctly executed with proper response ordering.
![AXI-Waveforms_page-0009](https://github.com/user-attachments/assets/a6b3f261-d1d9-4ac3-9142-0b0c6bcfc421)

---

![AXI-Waveforms_page-0010](https://github.com/user-attachments/assets/4f5d28e8-2e1d-4f0a-a743-33d4f6479caf)



---

