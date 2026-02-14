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

### OS/os.c

Contains:

### OS/root.s

Contains:

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