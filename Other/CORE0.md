### Deconstructing Windows’ ACPI Interrupt Affinity

**A deeper look at core 0 pinning for ACPI SCI interrupts**

---

### Summary

**Hypothesis:** The `acpi.sys` driver explicitly pins its primary interrupt (the SCI) to CPU 0.

**Conclusion:** **Hypothesis is false.** Through this deeper look, I will statically and dynamically prove that the `acpi.sys` driver is innocent, it does **not** specify a CPU affinity. Instead, the affinity is assigned by the Windows HAL's default policy for legacy interrupts, which then programs the hardware to physically route the SCI to the lowest-numbered available processor, which is typically **CPU 0**.

**The Chain:**

1.  **Static Analysis (`acpi.sys`):** The driver uses a `Version = 2` (line-based) `IoConnectInterruptEx` call, which structurally has no mechanism to provide a CPU affinity mask.
2.  **Kernel Debugging (Proof Point #1):** I've intercepted this call live and confirmed `acpi.sys` passes `Version = 2` and provides **no affinity parameters**.
3.  **Kernel Debugging (Proof Point #2):** I've intercepted the subsequent low-level connection in `nt!IopConnectInterrupt` and captured the final, resolved affinity mask chosen by the HAL. On a multi-core system, this mask was **`0x3`** (allowing CPU 0 and CPU 1).
4.  **Hardware Analysis (`!ioapic`):** I've identified the ACPI SCI's hardware interrupt line (IRQ 9) and dumped the I/O APIC's hardware state. The redirection entry for IRQ 9 was programmed with a physical destination APIC ID of **`0`**, proving the hardware was hard-wired to route the interrupt to **CPU 0**, despite the more permissive software mask.
5.  **Static Analysis (`HAL`):** I've reverse-engineered the HAL to find the exact code that enforces this policy, converting the `0x3` software mask into a `CPU 0 only` hardware rule.

This report documents the end-to-end methodology used to trace the behavior from software to hardware and to fullfill my curiosity about this.

---

### 1. My Methodology

I have adopted a high level approach to this, moving from high-level code down to the physical hardware state. Each step builds upon the last to formulate a clear chain of understanding.

1.  **Static Analysis (Code Intent):** Reverse engineering `acpi.sys`, `ntoskrnl.exe` in IDA to understand the intended code path and how interrupt affinity *should* be handled.
2.  **Live Kernel Debugging (Software Reality):** Intercepting key function calls in WinDbg to prove:
    *   What parameters `acpi.sys` actually sends.
    *   What affinity policy the kernel/HAL ultimately decides upon.
3.  **Hardware State Analysis (Physical Reality):** Dumping the state of the virtual I/O APIC to prove how the interrupt is physically routed, confirming the ultimate outcome of the kernel's decision.
---

### 2. Static Analysis: The Driver's Intent

I've reversed the key functions involved in ACPI interrupt setup.

*   **`acpi!OSInterruptVector`:** This is the function responsible for connecting the SCI. My analysis showed it calls `IoConnectInterruptEx` with a `Parameters.Version = 2` structure.
*   **`nt!IoConnectInterruptEx`:** The disassembly for this function shows that a Version 2 request is routed to the "line-based" connection path, which does not process a driver-supplied affinity mask.
*   **Conclusion:** The `acpi.sys` driver code is designed to be passive. It requests a connection without specifying an affinity, delegating the decision to the OS.

<img width="1666" height="993" alt="Using Line-Based connect, no affinity provided" src="https://github.com/user-attachments/assets/650d5437-2074-44fb-a8c7-1ff2d7f23034" />

---

### 3. Static Analysis: The Kernel's Intent

After receiving the connection request from a driver, the kernel's I/O Manager uses the low-level `nt!IopConnectInterrupt` function to act on the policy decision made by the HAL. By analyzing this function, we can understand the kernel's intent for implementing that policy.

*   **Function Purpose:** To take the abstract connection data (including the final affinity mask from the HAL) and create the concrete kernel objects (`_KINTERRUPT`) that are physically bound to specific CPU cores.
*   **The Logic:**
    1.  **Affinity Unpacking:** The function's first action is to read the Group number and Affinity Mask from the `_IO_CONNECT_ACTIVE_CONTEXT` structure that was populated by the HAL.
    2.  **Per-CPU Object Creation:** The code then enters a loop that iterates through each bit set in the affinity mask. For *every valid CPU*, it allocates a `_KINTERRUPT` object and explicitly initializes it to be permanently bound to that specific CPU core using `KeInitializeInterruptEx`.
    3.  **Connection:** Finally, it connects this set of CPU-specific interrupt objects to the system's dispatch mechanism.

*   **Conclusion:** The kernel's intent is to fully honor the affinity mask provided by the HAL. It doesn't just "prefer" one core, it creates the necessary kernel objects to enable interrupt delivery to **every single processor** specified in the mask.

<img width="1435" height="934" alt="Add a subheading" src="https://github.com/user-attachments/assets/b585c90f-7205-42e3-ab66-7728f4c77d27" />
<img width="1284" height="910" alt="Add a subheading (2)" src="https://github.com/user-attachments/assets/e62a01e6-7633-4a98-a1f3-2ef2ebf71e61" />

---

### 4. Live Kernel Debugging: Tracing the Policy

### Environment

*   **Guest OS:** Windows 10 x64 (26100.1 *ge_release.240331-1435*)
*   **Hypervisor:** VMware Workstation/Player
*   **vCPUs:** 4
*   **Debugger:** WinDbg (Preview) over **named pipe** serial KD

#### 4.1. Proof Point #1: The ACPI Driver's Call

I've set a breakpoint on `nt!IoConnectInterruptEx` and rebooted.

*   **Breakpoint Hit:** The debugger stopped on a call originating from `acpi!OSInterruptVector`.
*   **Parameter Analysis:** I've dumped the first ULONG of the parameters structure pointed to by `rcx`:
    `kd> dd @rcx L1`
    `ffff....  00000002`
*   **Conclusion:** This proves that `acpi.sys` is making a `Version = 2` call, providing no affinity.

<img width="1870" height="1009" alt="image" src="https://github.com/user-attachments/assets/985d0bf2-b822-433d-af94-7d61ca75405f" />

#### 4.2. Proof Point #2: The HAL's Permissive Software Policy

I've set a second breakpoint on `nt!IopConnectInterrupt`, the function that acts on the HAL's decision.

*   **Breakpoint Hit:** The debugger stopped inside this function.
*   **Parameter Analysis:** I've located the `_IO_CONNECT_ACTIVE_CONTEXT` structure on the stack and dumped its memory. At an offset of `+0x20`, I found the resolved `GROUP_AFFINITY`:
    `kd> dq [addr]+20 L2`
    `ffff....  00000000 00000003 00000000 00000000`
*   **Conclusion:** The HAL's default software policy for this interrupt is a mask of `0x3` (CPUs 0 and 1). This proves the affinity is not hardcoded to Core 0 at the software policy level. It creates a set of eligible processors.

<img width="919" height="62" alt="image" src="https://github.com/user-attachments/assets/ff72bbf0-2ad6-4702-b012-68e117a541c3" />

#### 4.3. Identifying the Hardware Interrupt Line (GSI)

Before examining the hardware state, I wanted to first see if i can identify which physical interrupt line the ACPI SCI is using. This information, the Global System Interrupt (GSI), is contained within the connection parameters passed to `nt!IopConnectInterrupt`.

*   **Method:** While broken in at the `nt!IopConnectInterrupt` breakpoint, we can locate this information within the `_IO_CONNECT_ACTIVE_CONTEXT` structure. Since the public symbols for this structure are incomplete, we read the GSI directly from the raw memory of the structure.
    1.  First, I've retrieved the pointer to the context structure from the stack:
        `kd> dq @rsp+58 L1`
    2.  Next, dump the memory of this structure. Through analysis of the structure's layout (comparing known values like the affinity mask), I've determined the GSI is located at an offset of **`+0x40`**.
        `kd> dd [addr]+40 L1`
*   **Conclusion:** We have identified the physical hardware line for the ACPI SCI as **GSI 9**. We can now use this number to query the I/O APIC's programming for this specific interrupt.

<img width="946" height="67" alt="{93E47C47-ED25-442A-A49B-F38D9E05ECE1}" src="https://github.com/user-attachments/assets/916312c4-643e-46f0-a73f-85594e5bd0d6" />

---

### 5. Hardware State Analysis: The Physical Reality

My live debugging proved that the software *policy* allows for the SCI to be handled by either CPU 0 or CPU 1. However, this is not the end of the story. We have to inspect the hardware itself to see the final result of this policy.

1.  **Dump the I/O APIC:** I've used the `!ioapic` extension to view the hardware redirection table.
2.  **Analyze the SCI Entry:** I've looked at the entry for the GSI I've identified in the previous step, **IRQ 9**. The output showed:
    `Inti09.: ... Ph:00000000 ...`
3.  **Conclusion:** The `Ph:00000000` field indicates the interrupt is set to **Physical Delivery (`Ph`)** to the processor with an **APIC ID of 0 (`00000000`)**. Despite the permissive software mask of `0x3`, the HAL's algorithm chose the loI'vest-numbered processor and programmed the I/O APIC to physically route **all SCI signals exclusively to CPU 0**. The hardware routing table has the final say.

<img width="953" height="294" alt="{B39802C3-1B0E-4B1B-95DA-D188A07BE41B}" src="https://github.com/user-attachments/assets/f75cb011-8548-4b33-b229-308d6c45af56" />

---

### 6. Static Analysis: The HAL's Policy and Execution

The hardware state analysis proves a restrictive policy is being enforced. To complete the evidence chain, I've statically reverse-engineered HAL in the kernel to find the exact code responsible for converting the permissive software mask into a restrictive hardware rule.

<img width="1019" height="696" alt="Add a little bit of body text" src="https://github.com/user-attachments/assets/34cd8de3-95d4-4f6e-94ee-809324569cae" />

*   **Function Purpose:** To act as the "policy engine" for interrupt routing. It receives the affinity mask from the kernel and, based on the type of interrupt, decides on the final physical routing.
*   **The Logic:** For a legacy line-based interrupt like the ACPI SCI, the HAL contains a deterministic policy: **it will always choose the lowest-numbered processor from the eligible set defined by the affinity mask.** It then changes the routing parameters in memory from a "Logical" destination with a mask of `0x3` to a "Physical" destination with an APIC ID of `0`.

*   **Conclusion:** This is where the software decision is finalized. The HAL's conservative policy for legacy interrupts overrides the flexibility offered by the kernel, making a hard choice to target only CPU 0.
---

<img width="997" height="582" alt="The code reads the chosen APIC ID (0) and shifts it into the destination field (bits 56-63) of the 64-bit hardware command" src="https://github.com/user-attachments/assets/244e7456-58a7-4a40-b9fa-8a2a37e7b8da" />

*   **Function Purpose:** This is the "command encoder" it's sole job is to take the final routing decision from the policy engine and convert it into the raw 64-bit value that the I/O APIC hardware understands.
*   **The Logic:** The function checks the delivery mode. Since the policy engine chose "Physical," it executes a specific code path:
    1.  It reads the target APIC ID (which is now `0`) from its input parameters.
    2.  It takes this APIC ID (`0`) and **shifts it left by 24 bits** (`<< 0x18`), placing it into the "Destination Field" (bits 56-63) of the 64-bit Redirection Table Entry (RTE).

*   **Conclusion:** The code explicitly encodes the "CPU 0 only" policy into a hardware command, creating the 64-bit value that will be written to the I/O APIC.

---
<img width="852" height="557" alt="Add a heading" src="https://github.com/user-attachments/assets/52a2a001-8d60-4271-9fc9-4c2e26d7278b" />

*   **Function Purpose:** Finally this function takes the fully-formed 64-bit RTE from `HalpApicConvertToRte` and performs the physical write to the hardware.
*   **The Logic:** It performs the classic two-step I/O APIC write:
    1.  It gets the memory-mapped base address of the I/O APIC hardware.
    2.  It writes to the I/O APIC's window register to select the upper 32-bits of the target RTE, then writes the data containing the APIC ID (`0`).
    3.  It writes to the window register again to select the lower 32-bits, then writes the rest of the configuration data.

*   **Conclusion:** This code is the final step, taking the command that was hard-coded for CPU 0 and physically programming the I/O APIC. After this function executes, the hardware has no choice but to route all SCI signals to CPU 0.
