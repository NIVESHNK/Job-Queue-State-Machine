# Exercise 17: NvM Job Queue / State Machine

## Project Documentation

**Tools** - NXP S32 Design Studio - PuTTY (UART monitoring)

**Interfaces** - I²C EEPROM - UART

------------------------------------------------------------------------

## 1. Overview

This project implements a simplified AUTOSAR-style Non-Volatile Memory
(NvM) Manager using an NXP S32K144 microcontroller and an external
M24C04 EEPROM. Multiple write requests are stored in a First-In,
First-Out (FIFO) queue and processed sequentially through a state
machine.

The application verifies stored data through read-back comparison. A
separate power-cycle test checks whether the data remains available
after power is removed and restored.

The main write-state sequence is:

`IDLE → WRITE_PENDING → WRITE_BUSY → DONE → IDLE`

`NvM_MainFunction()` advances the state machine, while UART output is
used to observe job processing and verification results.

> This is a simplified AUTOSAR-style learning exercise, not a claim of a
> complete AUTOSAR-compliant NvM stack.

## 2. Requirements and Scope

  --------------------------------------------------------------------------------
  ID                                  Requirement
  ----------------------------------- --------------------------------------------
  R1                                  Accept EEPROM write requests from
                                      application blocks.

  R2                                  Store requests in a FIFO queue and process
                                      them in acceptance order.

  R3                                  Implement the
                                      `IDLE → WRITE_PENDING → WRITE_BUSY → DONE`
                                      state sequence.

  R4                                  Use I²C/LPI2C to communicate with the
                                      external M24C04 EEPROM.

  R5                                  Verify written data by reading it back and
                                      comparing it with the expected data.

  R6                                  Provide a separate power-cycle retention
                                      test.

  R7                                  Report state transitions and verification
                                      results through UART.
  --------------------------------------------------------------------------------

## 3. How It Works

1.  Power on and initialize the clock, pins, UART, and LPI2C
    peripherals.
2.  Submit EEPROM write requests from the application.
3.  Store each accepted request in the FIFO job queue.
4.  `NvM_MainFunction()` retrieves the next job and starts the EEPROM
    write operation.
5.  Wait for the EEPROM write cycle to complete (approximately 5 ms).
6.  Read the stored data back and compare it with the expected value.
7.  Mark the job complete and process the next request.
8.  Report the result through UART.

A separate power-cycle test removes and restores power, then reads the
stored data again without rewriting it.

## 4. Request Data

Each queued request identifies the EEPROM block or address, the data
buffer, and the data length. The configured queue capacity is eight
requests.

  -----------------------------------------------------------------------
  Request field                       Description
  ----------------------------------- -----------------------------------
  Block ID / address                  Identifies the EEPROM location to
                                      access.

  Data buffer                         Contains the bytes to write or
                                      receives the bytes read.

  Data length                         Number of bytes involved in the
                                      request.
  -----------------------------------------------------------------------

## 5. Test Data

The application submits three write requests for EEPROM blocks 1, 2, and
3. Each block contains four bytes of test data.

  EEPROM block   Expected data         Initial read-back result
  -------------- --------------------- --------------------------
  Block 1        `10, 20, 30, 20`      PASS\*
  Block 2        `50, 60, 70, 80`      PASS\*
  Block 3        `90, 100, 110, 120`   PASS\*

\*The project notes report PASS for the initial read-back comparisons.
Record power-cycle verification separately after performing that test.

## 6. Software Structure

  -----------------------------------------------------------------------
  File                                Function / purpose
  ----------------------------------- -----------------------------------
  `main.c`                            Initializes peripherals, submits
                                      requests, monitors states, and
                                      verifies stored data.

  `nvm.c`                             Implements the NvM state machine
                                      and coordinates write operations.

  `queue.c`                           Implements FIFO enqueue and dequeue
                                      operations.

  `eeprom.c`                          Implements I²C-based EEPROM read
                                      and write operations.

  `nvm.h`                             Defines NvM states, data
                                      structures, constants, and function
                                      declarations.

  `eeprom.h`                          Declares EEPROM functions and
                                      status types.
  -----------------------------------------------------------------------

Adjust this table if the actual file names in your repository differ.

## 7. NvM State Machine

The state machine processes queued write requests using this sequence:

`IDLE → WRITE_PENDING → WRITE_BUSY → DONE → IDLE`

  -----------------------------------------------------------------------
  State                               Purpose
  ----------------------------------- -----------------------------------
  `IDLE`                              Checks whether the queue contains a
                                      request.

  `WRITE_PENDING`                     Starts the EEPROM write operation.

  `WRITE_BUSY`                        Waits for the EEPROM's internal
                                      write cycle, approximately 5 ms.

  `DONE`                              Records the result, removes the
                                      completed job, and returns to
                                      `IDLE`.
  -----------------------------------------------------------------------

`NvM_MainFunction()` is called repeatedly to advance the state machine.
The write-cycle wait should not block the main loop. Read requests can
use the same queueing approach but do not normally need the EEPROM's
internal write-cycle delay.

## 8. EEPROM Interface

  -----------------------------------------------------------------------
  Item                                Description
  ----------------------------------- -----------------------------------
  Interface                           I²C using the S32K144 LPI2C
                                      peripheral

  EEPROM device                       M24C04 external EEPROM

  Write-cycle delay                   Approximately 5 ms after a write

  Read verification                   Read back stored bytes and compare
                                      them with expected data

  Power removal                       Only after the write cycle has
                                      completed
  -----------------------------------------------------------------------

The project uses a fixed write-cycle delay rather than I²C ACK polling.
EEPROM communication errors should be reported as failed jobs rather
than treated as successful completion.

## 9. Testing and Verification

### 9.1 Initial Read-Back Verification

1.  Initialize the microcontroller peripherals.
2.  Submit the three EEPROM write requests.
3.  Allow the state machine to process each request sequentially.
4.  Wait for the write operations to complete.
5.  Read back blocks 1, 2, and 3.
6.  Compare the retrieved bytes with the expected data.
7.  Confirm the PASS or FAIL result through UART.

### 9.2 Power-Cycle Verification

1.  Complete the initial write and read-back test.
2.  Ensure the EEPROM write cycle has completed.
3.  Remove power from the EEPROM and the board as required by the
    hardware setup.
4.  Restore power and restart the microcontroller.
5.  Initialize LPI2C and UART.
6.  Read the EEPROM blocks without rewriting the test data.
7.  Compare the retrieved data with expected values stored independently
    in firmware.
8.  Report PASS if the values match; otherwise report FAIL.

**Important:** If startup code automatically rewrites the test pattern,
disable that behavior during the power-cycle test. Report successful
power-cycle retention only after performing the test and recording the
UART result.

## 10. Algorithm Decision Table

  -----------------------------------------------------------------------
  Condition                           Expected result
  ----------------------------------- -----------------------------------
  Queue is empty                      Remain in `IDLE` and wait for a
                                      request.

  A write request is available        Start the write and enter the
                                      write-cycle wait state.

  Write cycle completes               Mark the job complete and remove it
                                      from the queue.

  Read-back data matches              Report verification PASS.

  Read-back data differs              Report verification FAIL.

  I²C transaction fails               Report a failed job and handle the
                                      error.
  -----------------------------------------------------------------------

## 11. Build and Execution

1.  Open the project in NXP S32 Design Studio.
2.  Check clock configuration, pin multiplexing, LPUART, and LPI2C
    settings.
3.  Confirm EEPROM wiring and I²C address configuration.
4.  Build the project and resolve compilation errors.
5.  Flash the firmware to the S32K144 evaluation board.
6.  Open the configured UART COM port using PuTTY.
7.  Observe queue processing, state transitions, write results, and
    read-back verification.
8.  Perform the power-cycle test and run read-only verification after
    restart.

## 12. Expected Results and UART Monitoring

UART monitoring should report queue activity, state transitions, EEPROM
operation status, and data verification.

Expected behavior: - Requests are processed sequentially in FIFO
order. - The queue becomes empty after all requests complete. -
Read-back data matches the expected values. - After a power cycle, the
stored values remain available if the writes completed successfully and
the EEPROM retained the data.

The power-cycle result must be recorded separately from initial
verification.

## 13. Conclusion

This exercise demonstrates a simplified AUTOSAR-style NvM Manager using
a FIFO job queue, a state machine, and an external M24C04 EEPROM over
I²C. It covers sequential request handling, EEPROM write-cycle
management, UART monitoring, read-back verification, and a separate
procedure for checking non-volatile data retention after power removal
and restoration.
