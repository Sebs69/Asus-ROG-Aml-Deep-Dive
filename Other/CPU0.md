
### **Summary**

> **Hypothesis:** The `acpi.sys` driver explicitly pins its primary interrupt (the SCI) to CPU 0.
>
> **Conclusion:** **Hypothesis is false.** `acpi.sys` connects the SCI using `IoConnectInterruptEx` with `CONNECT_LINE_BASED (Version = 2)` and does **not** pass a CPU affinity. The HAL resolves a **software** affinity (on my VM it ends up as `0x3`, so CPU0+CPU1). For **hardware**, the HAL programs the IOAPIC entry for the SCI (GSI/IRQ 9) to **Lowest-priority delivery** with a **Logical** destination mask that matches that set.
>
> It is **not** physically pinned to CPU 0, but it persistently lands there as a result of the Windows kernel's scheduling and power management policies. The hardware is configured for "lowest priority" delivery, which includes a fair, rotating arbitration mechanism to handle ties. However, the Windows scheduler frequently parks other cores during idle periods, leaving CPU 0 as the only processor in a responsive state. This effectively eliminates any tie-breaking scenario and makes CPU 0 the de facto target for the interrupt.

> This report documents the end-to-end methodology used to trace the behavior from software to hardware and to fulfill my curiosity about this.

---

### 1. My Methodology

I have adopted a high level approach to this, moving from high-level code down to the physical hardware state. Each step builds upon the last to formulate a clear chain of understanding.

1. **Static Analysis (Code Intent):** Reverse engineering `acpi.sys`, `ntoskrnl.exe` in IDA to understand the intended code path and how interrupt affinity *should* be handled.
2. **Live Kernel Debugging (Software Reality):** Intercepting key function calls in WinDbg to prove:

   * What parameters `acpi.sys` actually sends.
   * What affinity policy the kernel/HAL ultimately decides upon.
3. **Hardware State Analysis (Physical Reality):** Dumping the state of the virtual IOAPIC to prove how the interrupt is physically routed, confirming the ultimate outcome of the kernel's decision.

---

### 2. Static Analysis: The Driver's Intent

I've reversed the key functions involved in ACPI interrupt setup.

* **`acpi!OSInterruptVector`:** This is the function responsible for connecting the SCI. My analysis showed it calls `IoConnectInterruptEx` with a `Parameters.Version = 2` structure.
* **`nt!IoConnectInterruptEx`:** The disassembly for this function shows that a Version 2 request is routed to the "line-based" connection path, which does not process a driver-supplied affinity mask.
* **Conclusion:** The `acpi.sys` driver code is designed to be passive. It requests a connection without specifying an affinity, delegating the decision to the OS.

<img width="1666" height="993" alt="Using Line-Based connect, no affinity provided" src="https://github.com/user-attachments/assets/650d5437-2074-44fb-a8c7-1ff2d7f23034" />

---

### 3. Static Analysis: The Kernel's Intent

After receiving the connection request from a driver, the kernel's I/O Manager uses the low-level `nt!IopConnectInterrupt` function to act on the policy decision made by the HAL. By analyzing this function, we can understand the kernel's intent for implementing that policy.

* **Function Purpose:** To take the abstract connection data (including the final affinity mask from the HAL) and create the concrete kernel objects (`_KINTERRUPT`) that are physically bound to specific CPU cores.

* **The Logic:**

  1. **Affinity Unpacking:** The function's first action is to read the Group number and Affinity Mask from the `_IO_CONNECT_ACTIVE_CONTEXT` structure that was populated by the HAL.
  2. **Per-CPU Object Creation:** The code then enters a loop that iterates through each bit set in the affinity mask. For *every valid CPU*, it allocates a `_KINTERRUPT` object and explicitly initializes it to be permanently bound to that specific CPU core using `KeInitializeInterruptEx`.
  3. **Connection:** Finally, it connects this set of CPU-specific interrupt objects to the system's dispatch mechanism.

* **Conclusion:** The kernel's intent is to fully honor the affinity mask provided by the HAL. It doesn't just "prefer" one core, it creates the necessary kernel objects to enable interrupt delivery to **every single processor** specified in the mask.

<img width="1435" height="934" alt="Add a subheading" src="https://github.com/user-attachments/assets/b585c90f-7205-42e3-ab66-7728f4c77d27" />
<img width="1284" height="910" alt="Add a subheading (2)" src="https://github.com/user-attachments/assets/e62a01e6-7633-4a98-a1f3-2ef2ebf71e61" />

---

### 4. Live Kernel Debugging: Tracing the Policy

### Environment 

* **Guest OS:** Windows 11 Pro 24H2 (26100.1742)
* **Hypervisor:** VMware Workstation
* **vCPUs:** 4
* **Debugger:** WinDbg (Preview) over **named pipe** serial KD
* **Scope & applicability.** The results here are from a VMware guest using xAPIC/IOAPIC. On bare metal with **x2APIC** and/or **Interrupt Remapping (VT-d/AMD-IOMMU)**, the **programming path** (IOAPIC vs. remapping hardware) may differ, but the **policy** remains: the HAL resolves a **software eligible set**, and delivery uses **lowest-priority** semantics to one CPU in that set. 

#### 4.1. Proof Point #1: The ACPI Driver's Call

I've set a breakpoint on `nt!IoConnectInterruptEx` and rebooted.

* **Breakpoint Hit:** The debugger stopped on a call originating from `acpi!OSInterruptVector`.
* **Parameter Analysis:** I've dumped the first ULONG of the parameters structure pointed to by `rcx`:

```
kd> dd @rcx L1
ffff....  00000002
```

* **Conclusion:** This proves that ``acpi.sys`` is using ```CONNECT_LINE_BASED (Version = 2)```, providing no affinity.

<img width="1870" height="1009" alt="image" src="https://github.com/user-attachments/assets/985d0bf2-b822-433d-af94-7d61ca75405f" />

#### 4.2. Proof Point #2: The HAL's Permissive Software Policy

I've set a second breakpoint on `nt!IopConnectInterrupt`, the function that acts on the HAL's decision.

* **Breakpoint Hit:** The debugger stopped inside this function.
* **Parameter Analysis:** The parameters are passed via an internal, undocumented structure `_IO_CONNECT_ACTIVE_CONTEXT`. By analyzing its memory layout in the debugger and cross-referencing against the values of known adjacent fields, the resolved GROUP_AFFINITY was identified at an offset of +0x20:

```
kd> dq [addr]+20 L2
ffff....  00000000 00000003 00000000 00000000
```

* **Conclusion:** The HAL's default software policy for this interrupt is a mask of `0x3` (CPUs 0 and 1). This proves the affinity is not hardcoded to Core 0 at the software policy level. It creates a set of eligible processors.

<img width="919" height="62" alt="image" src="https://github.com/user-attachments/assets/ff72bbf0-2ad6-4702-b012-68e117a541c3" />

#### 4.3. Identifying the Hardware Interrupt Line (GSI)

Now we have to conclusively determine the interrupt request (IRQ) line assigned to the ACPI System Control Interrupt (SCI) on the target system. The hypothesis is that the SCI is assigned to IRQ 9 (as per default). To prove this, we will use the Windows Kernel Debugger (WinDbg) to inspect the ACPI tables loaded in physical memory. The two key tables for this verification are:

1. **FADT (Fixed ACPI Description Table):** This table contains a field, `SCI_Interrupt`, which explicitly defines the system interrupt vector for the SCI.
2. **MADT (Multiple APIC Description Table):** This table describes the system's interrupt controller configuration. It is used to confirm that the SCI's IRQ is not being re-mapped or overridden.

#### Step 1: Locating the ACPI Root Tables

The first step is to locate the root of the ACPI table structure. The Extended System Description Table (XSDT) contains pointers to all other ACPI description tables. We can dump this table using the `!rsdt` debugger extension.

**Command:**

```
0: kd> !rsdt
```

**Output:**

<img width="455" height="275" alt="{06C0F980-C237-4880-8335-A0EC1E0E7942}" src="https://github.com/user-attachments/assets/b8320965-678f-4d89-bfba-7074be7943ea" />

From this output, we successfully identified the physical memory addresses for our tables of interest:

* **FADT (listed as FACP):** `0xfbfad0b`
* **MADT (listed as APIC):** `0xfbfae73`

#### Step 2: Analysis of the Fixed ACPI Description Table (FADT)

I've dumped the raw physical memory of the table using the `!db` (display bytes physical) command.

**Command:**

```
0: kd> !db 0xfbfad0b L50
```

<img width="1261" height="95" alt="{8578FC50-973A-486F-A91C-46E55DA72537}" src="https://github.com/user-attachments/assets/58652a59-7467-4e41-ab1a-82001cf81098" />

**Analysis:**
According to the ACPI specification, the `SCI_Interrupt` field is located at offset `+0x2E` from the start of the FADT [1]. We can locate this value in the memory dump:

* Start Address: `0xfbfad0b`
* Offset: `+0x2E`
* Target Address: `0xfbfad39`

Examining the third line of the output, which starts at `...ad2b`, we can count forward to find the byte at `...ad39`.

<img width="546" height="62" alt="89C05E4B-3414-4164-8562-CE1A5E13AFC1" src="https://github.com/user-attachments/assets/fc00cd61-35be-46c5-b492-59ce97cfc9fb" />

The value at this location is **`0x09`**. This confirms that the FADT specifies interrupt vector 9 for the SCI.

#### Step 3: Analysis of the Multiple APIC Description Table (MADT)

To be certain that IRQ 9 isn't being redirected, we need to inspect the MADT. We're looking for a special entry called an "Interrupt Source Override" that would remap the IRQ.

**Command:**

```
0: kd> !db 0xfbfae73 L7a
```

**Output Data:**

<img width="1211" height="229" alt="image" src="https://github.com/user-attachments/assets/db900062-7d7a-43c9-9d57-be8d52dfa6ba" />

**Analysis:**
The memory dump shows the entire MADT, which contains a list of entries describing the system's interrupt controllers. By walking through this list entry by entry, we can account for the entire table and see exactly how interrupts are configured.

If you parse it you will see these entries for the system's processor cores (Type 0), their NMIs (Type 4), and the main IOAPIC (Type 1). Most importantly, there is only **one** "Interrupt Source Override" (Type 2) in the entire table. As seen on the last line of the dump, this entry (`02 0a 00...`) is for **ISA IRQ 0**, remapping it to global interrupt 2.

After accounting for every entry in the table, our complete scan confirms there is **no override entry for IRQ 9**. Because no override exists, IRQ 9 maintains its default 1-to-1 mapping, connecting directly to Global System Interrupt 9.

#### 4.4 OS-Level Verification: Confirming the GSI in Action

Our analysis of the FADT and MADT tables established the hardware's blueprint: the ACPI SCI is configured to use IRQ 9. To complete the proof, we will now verify that the Windows kernel itself follows this blueprint by connecting its SCI handler to the correct Global System Interrupt (GSI).

* **Method:** The interrupt parameters, including the GSI, are passed in the internal `_IO_CONNECT_ACTIVE_CONTEXT` structure. As its definition is not in public symbols, its layout must be determined empirically. This is a standard reverse engineering practice for analyzing kernel behavior. We dump the memory of this structure. Based on this live analysis, the GSI was consistently found at the offset of `+0x40`.

  1. First, we retrieve the pointer to the context structure from the stack.
  2. Next, we dump the memory of this structure. Through analysis of its layout, the GSI was determined to be at an offset of **`+0x40`**.

* **Results:** The commands below execute this process, revealing the GSI value used by the kernel.

  ```
  0: kd> dq @rsp+58 L1
  fffff806`8d73a830  fffff806`8d73a970

  0: kd> dd fffff806`8d73a970+40 L1
  
  ```

  <img width="946" height="67" alt="{93E47C47-ED25-442A-A49B-F38D9E05ECE1}" src="https://github.com/user-attachments/assets/916312c4-643e-46f0-a73f-85594e5bd0d6" />

* **Conclusion:** The memory dump at the `+0x40` offset clearly shows the value `0x00000009`. This confirms that the Windows kernel is requesting to connect its ACPI SCI handler to **GSI 9**. This OS-level action perfectly corroborates our hardware-level analysis, providing end-to-end proof that the ACPI SCI is operating on IRQ 9.

#### 4.5. Final IOAPIC state

Results show: 

<img width="810" height="466" alt="DCFC7AEF-F15E-4045-BDA2-6996D29C6F07" src="https://github.com/user-attachments/assets/45485fa6-de9d-4323-a819-c6cae04b69aa" />

```
Inti09.: 03000000`0000a9b0  Vec:B0  LowestDl  Lg:03000000  lvl low
```

This line is a parsed representation of the 64-bit value that configures GSI 9. Let's break down what each component means, with reference to Intel's official architecture:

*   **`Vec:B0` (Vector):** This sets the interrupt vector to `0xB0`. When a CPU receives this interrupt, it will use this number to look up the correct handler (the ISR) in its Interrupt Descriptor Table (IDT). This corresponds to bits 0-7 of the RTE.

*   **`lvl low` (Trigger Mode & Polarity):** This confirms the interrupt is configured as **level-triggered** (bit 15) and **active-low** (bit 13). This is a critical setting that matches the electrical specification of the ACPI SCI, which remains asserted at a low voltage level until the OS services it.

*   **`LowestDl` (Delivery Mode):** This is the most important setting for our analysis. It sets the **Delivery Mode** (bits 8-10 of the RTE) to `001b`, which Intel defines as **"Lowest Priority"**. This instructs the I/O APIC *not* to send the interrupt to a fixed, predetermined CPU. Instead, it targets a group of CPUs, and the interrupt is delivered to whichever processor in that group is currently running at the lowest interrupt priority. This hardware feature is the foundation for interrupt load balancing.

*   **`Lg:03000000` (Destination Mode & Destination):** This defines the group of CPUs eligible to receive the interrupt.
    *   The `Lg` signifies that the **Destination Mode** (bit 11) is set to **Logical**. In this mode, the `Destination` field (bits 56-63) is interpreted as a bitmap, where each bit represents a CPU.
    *   The value `0x03` (`0b00000011`) is a mask where bits 0 and 1 are set. This explicitly defines the target group as **{CPU 0, CPU 1}**, perfectly matching the software affinity (`0x3`) that the HAL decided upon in section 4.2.

**In summary, these hardware settings provide the definitive proof.** The I/O APIC is explicitly commanded to take any active-low, level-triggered signal on GSI 9 and deliver it as interrupt vector `0xB0` using the "Lowest Priority" delivery scheme to the logical set of processors containing CPU 0 and CPU 1.

This immediately raises the next logical question: if both CPU 0 and CPU 1 are eligible targets and running at the same priority, how does the hardware decide which one gets the interrupt? This is where the processor's arbitration mechanism comes into play. [2]

---

### 5. The Question: Why Core 0 Almost Always Wins

A logical question comes from the evidence: if the hardware uses a fair, rotating arbitration for tie-breaking, why does the ACPI SCI land on Core 0 nearly 100% of the time on a typical Windows system?

The answer is that the hardware's tie-breaking is rarely needed because **the Windows kernel's software policies prevent a tie from ever occurring.**

The IOAPIC's rule is to deliver an interrupt to the lowest-priority processor *that is able to service it*. An advanced OS like Windows heavily manages processor power states to improve efficiency.

1.  **Core Parking and Idle States (C-States):** On a lightly loaded or idle system, the Windows scheduler actively works to consolidate tasks onto a minimal number of cores (usually starting with Core 0). Other cores are placed into a "parked" state, allowing them to enter deep sleep (C-states).
2.  **Eliminating the Competition:** A processor in a deep C-state is not considered a ready target for an interrupt. When the SCI fires, the IOAPIC looks at its destination set (`{CPU0, CPU1}`) and finds that CPU 1 is in a deep sleep state. CPU 0, which the kernel deliberately keeps more active for system housekeeping, is the only viable target.
3.  **No Tie, No Arbitration:** Since there is only one ready processor, the interrupt is sent directly to CPU 0. The hardware's fair arbitration logic is never invoked because there is no tie to break. The OS's power management strategy has created a scenario where CPU 0 is the winner by default.

Therefore, the persistent targeting of CPU 0 is not a sign of hardware pinning but rather a predictable outcome of the synergy between the IOAPIC's delivery rules and the Windows scheduler's power-saving heuristics.

---

### **6. Conclusion: The Truth About the SCI and CPU 0**

After a full-stack investigation from driver code down to hardware registers, the answer is definitive: **the `acpi.sys` SCI is not, by any mechanism, hard-wired or pinned to CPU 0.**

The persistent observation that it lands on CPU 0 is an emergent behavior resulting from three layers of policy:

1.  **The Driver is Hands-Off (`acpi.sys`):** The ACPI driver does not request a specific CPU affinity, delegating the decision to the operating system.
2.  **The HAL Forms a Team (Windows HAL):** The Hardware Abstraction Layer establishes a software policy, creating a set of eligible processors for the SCI (CPUs 0 and 1 in this case). It then programs the hardware to match this policy.
3.  **The OS Makes CPU 0 the Only Viable Player (Windows Scheduler):** The kernel’s power manager aggressively parks idle cores. This ensures that when an SCI occurs on a quiet system, CPU 0 is the only processor in the eligible "team" that is actually awake and ready. The IOAPIC, following its rules, delivers the interrupt to the sole available target.


#### References:

**[1]** UEFI ACPI Spec 6.5a, *Advanced Configuration and Power Interface (ACPI) Specification*, Version 6.5a, §5.2.9 "Fixed ACPI Description Table (FADT)" **SCI_INT** (2 bytes at offset 46). Available at: [https://uefi.org/sites/default/files/resources/ACPI_Spec_6.5a_Final.pdf](https://uefi.org/sites/default/files/resources/ACPI_Spec_6.5a_Final.pdf)

**[2]** Intel Corporation, *Intel® 82093AA I/O Advanced Programmable Interrupt Controller (IOAPIC) ; Datasheet*, §§3.2.1 “IOAPICID - IOAPIC Identification Register” & 3.2.3 "IOAPICARB  IOAPIC Arbitration Register," pp. 2, 9. Available at: [https://pdos.csail.mit.edu/6.828/2016/readings/ia32/ioapic.pdf](https://pdos.csail.mit.edu/6.828/2016/readings/ia32/ioapic.pdf).
