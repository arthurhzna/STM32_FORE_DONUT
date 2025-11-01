# STM32 FORE DONUT

This document describes the firmware data pipeline: the flow from input commands through frame generation, brightness modulation, PWM buffer construction, DMA transfer, and synchronization via callbacks. Variable names used in the code are intentionally not referenced here; descriptions use conceptual names (e.g., source buffer, modulated buffer).

Overview
- The firmware maintains a source buffer holding per-entry color components and an auxiliary buffer containing brightness-modulated values.
- A PWM buffer is built from those values and transferred out via DMA driven by a timer peripheral.
- The design separates three stages: generate (produce colors), modulate (apply brightness/transformations), and transmit (build PWM timings and start DMA).

Data structures (conceptual)
- Source buffer (array of entries): holds raw color components per entry and an identifier/index.
- Modulated buffer (array of entries): holds the adjusted color components after brightness or other transforms.
- PWM buffer (numeric array): stores per-bit PWM durations for the entire frame plus reset padding.

Processing steps
1. Frame generation: a generator function writes colors into the source buffer according to the active effect or external command.
2. Brightness modulation: when software brightness is enabled, a modulation function reads the source buffer, applies a brightness transform (the implementation uses a trigonometric scaling), and writes results into the modulated buffer.
3. PWM buffer build: the transmit function iterates entries (reading from the modulated buffer if modulation is enabled, otherwise the source buffer), converts each color component into a 24-bit sequence, and maps each bit to a PWM duty value representing logic '1' or '0'. Those duty values are appended into the PWM buffer.
4. Reset padding: after all bits, a fixed number of zero entries are added to the PWM buffer to generate the required reset period.
5. DMA start: the timer + DMA transfer is started non-blocking to send the PWM buffer out on the configured timer channel.

Transmission & synchronization
- DMA transfer is started using the HAL timer PWM DMA start call (non-blocking). The main loop waits on a transfer-complete flag that is set in the timer PWM pulse-finished callback.
- The pulse-finished callback stops the DMA-driven PWM and sets a flag to indicate the transfer is complete, allowing the pipeline to proceed with the next frame.
- This pattern ensures the transfer itself is non-blocking while retaining a simple synchronization mechanism to avoid starting overlapping transfers.

Timing (timer + PWM values)
- The timer is configured with a specific period so that PWM duty values correspond to the protocol's high/low timing for logic '1' and '0'.
- The implementation maps '1' bits to a higher duty value and '0' bits to a lower duty value; these numeric values should be chosen relative to the configured timer period to meet timing tolerances.

Buffer sizing & constraints
- The PWM buffer size depends on the number of entries and bits per entry. When increasing the number of entries, ensure the PWM buffer can hold all bits plus reset padding.
- Never start a new DMA transfer before the previous one finishes. Use a flag or a frame queue to prevent race conditions.

Triggering & control flow
- External commands (e.g., via UART) set a receive flag. The main loop processes commands in non-ISR context and triggers frame generation + transmit.
- Heavy computation (generation/modulation) should be done in the main context, not inside interrupts, to keep ISRs short and deterministic.

Debugging tips
- Monitor the transfer-complete flag to ensure the callback executes.
- Verify timer period and PWM duty mapping against the expected protocol timing.
- Confirm buffer sizes and watch for overruns when increasing frame length.




