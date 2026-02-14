# 003 Interrupt and Exception Handling

## 1. Code comments
### How does the timer interruption work?

The operation occurs in several stages:

1. Starting the timer:

    We turn on the timer value. Then interrupt 68 is unmasked in the interrupt controller (INTC), since that is the line associated with the timer
    
    After that:

    The timer is stopped (TCLR = 0).
    - Pending interrupts are cleared (TISR).
    - The TLDR value is loaded with the desired time (2 seconds).
    - The counter is initialized (TCRR).
    - The overflow interrupt is enabled (TIER).
    - The timer is started in auto-reload mode (TCLR = 0x3)


2. Enable interrupts.
    
    By default, the CPU has IRQs disabled.
    The enable_irq() function clears that bit to allow interrupts to reach the processor.

3. Wait for two seconds to pass (overflow)

    The timer starts counting down to 0xFFFFFFFF.
    When overflow occurs (after 2 seconds),
    the hardware generates IRQ 68.
    The controller detects the request and sends it to the CPU.


4. Enter IRQ mode

    - Automatically switches to IRQ mode
    - Jumps to address 0x18 of the vector table

5. Execute the handler

    irq_handler:

    - Saves the registers to restore the state later
    - Calls timer_irq_handler()

    timer_irq_handler:
    - Clears the timer interrupt flag (TISR)
    - Notifies the controller of the interrupt
    - Prints the tick

6. Return from the interrupt

    The registers are restored and this returns execution exactly to the point where the program was interrupted.

The process continues indefinitely, until the Beagle, in order to avoid getting “stuck,” takes us out of its operating system.

---

## 2. Function documentation

The functions that were added/edited here are as follows:

### OS/os.c

- `timer_init()`

    - `PUT32(CM_PER_TIMER2_CLKCTRL, 0x2);`:

        We enable the timer clock. If it is not activated, the timer does not receive a clock signal and cannot count.
        The important values are:

        - 0x0 → Disabled
        - 0x2 → Enabled

    - `PUT32(INTC_MIR_CLEAR2, (1 << (68 - 64)));`:

        Unmask interrupt 68 on the controller.

        - Bit 4 is activated
        - This allows the interrupt to reach the CPU.

    - `PUT32(INTC_ILR68, 0x0);`

        Set the interrupt priority 68.  
        The value `0x0` indicates maximum priority.  
        Values up to `0x7F` can be used, where lower numbers represent higher priority.  

    - `PUT32(TCLR, 0x0);`

        We stop the timer before setting it. 

    - `PUT32(TISR, 0x7);`

        Clears any pending interrupts from the timer.  
        If not cleared, it could trigger immediately upon activation.

    - `PUT32(TLDR, 0xFD115F00);`

        The initial value of the timer is loaded.  
        This value determines how much time must pass before the overflow occurs.  
        In this case, it corresponds to approximately 2 seconds.


    - `PUT32(TCRR, 0xFD115F00);`

        The counter is initialized with the same value as `TLDR`.  
        This ensures that the first cycle has the correct duration.

    - `PUT32(TIER, 0x2);`

        Enables the timer overflow interrupt.  
        The value `0x2` specifically activates the overflow event.


    - `PUT32(TCLR, 0x3);`

        Resets the timer in auto-reload mode.  
        The value `0x3` means:
        - Bit 0 → Start the timer  
        - Bit 1 → Activate auto-reload mode  
        This allows the timer to reset automatically after each overflow.

- `timer_irq_handler()`

    - `PUT32(TISR, 0x2);`:

        Clears the timer overflow interrupt flag.

        Here, `0x2` is used because we only want to clear the bit corresponding to the overflow event.

        In `timer_init()`, `0x7` was used to clear all possible pending flags before starting the timer. 

        However, within the handler, it is only necessary to clear the specific event that generated the interrupt.

    - `PUT32(INTC_CONTROL, 0x1);`:

        Notify the interrupt controller that the IRQ has already been handled.
        
        This allows future interrupts to be processed.

        If this is not done, the controller may become blocked
        and not allow new interrupts.

    - ` os_write("Tick\n");`:

        We print the ticket.

- `main()`

    - `os_write("initialization timer\n");`:

        We print that the timer is going to initialize.

    - `timer_init();`:

        We start the timer

    - `enable_irq();`:

        We enable interruptions, as they are disabled by default.

    - `os_write("interrupts are enabled\n");`:

        We only print that interrupts are already enabled.

    - `while()`

        This is the main cycle of the program.
        Here, random numbers are printed continuously,
        demonstrating that the main program continues to run
        while interrupts occur.


### OS/root.s

- `irq_handler`

    This is the interrupt handler at the assembly level.
    It runs automatically when the CPU gets an IRQ
    and jumps to address 0x18 of the vector table.

    - `push {r0-r12, lr}`:

        Save all general registers (r0–r12)
        and the link register (lr) to the stack.

    - `bl timer_irq_handler`

        Call the function in C that handles the interrupt logic.

    - `pop {r0-r12, lr}`:

        Restores the records that were previously saved.

    - `subs pc, lr, #4`:

        Return from the interrupt.

        This instruction:
        - Subtracts 4 from the value stored in `lr`
        - Places the result in `pc` (program counter)
        - Automatically restores the previous state of the processor

        This causes the program to continue exactly at the instruction
        where it was interrupted.

- `enable_irq`

    This function globally enables IRQ interrupts on the CPU. By default, IRQs are disabled when the system starts up.

    The ARM processor controls this via the CPSR
    (Current Program Status Register).

    - `mrs r0, cpsr`

        Copy the current value of the CPSR register to register `r0`.

        The CPSR contains important information such as:
        - Execution mode (User, IRQ, Supervisor, etc.)
        - Condition flags
        - Interrupt enable bits

    - `bic r0, r0, #0x80`

        Clears bit 7 of the CPSR (I-bit).

        The I-bit controls whether IRQs are enabled or disabled:

        - Bit 7 = 1 → IRQs disabled
        - Bit 7 = 0 → IRQs enabled

        `bic` means “Bit Clear,” so it clears (sets to 0) the bit corresponding to `0x80`.

    - `msr cpsr_c, r0`

        Write the new value to the CPSR.

        The suffix `_c` indicates that only the control bits are updated,
        not all fields in the register.

        At this point, IRQ interrupts are enabled.

    - `bx lr`

        Returns to the function that called `enable_irq`.

To solve this lab, it was simply a matter of following the instructions.

---
## 3. Architecture explanation
### How does the system manage IRQs from the hardware to the controller?

The interruption is handled as follows.

1. The timer counts until overflow. When this occurs, the hardware generates the IRQ 68 interrupt.

2. The signal reaches the interrupt controller. If the interrupt is unmasked, the INTC sends it to the CPU.

3. If IRQs are enabled on the CPU, it switches to IRQ mode and jumps to address 0x18 in the vector table.

4. In that position is the jump to `irq_handler`, where the registers are saved and `timer_irq_handle(` is called.

5. The `timer_irq_handler()` function clears the interrupt notifies the controller that it has been handled, and prints “Tick.”

6. Finally, the records are restored and the program continues from the point where it was interrupted.