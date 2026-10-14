# AUTOSAR NvM Manager – Job Queue and State Machine

## Exercise 17: AUTOSAR NvM Manager – Job Queue and State Machine

## 1. Project Overview

This project implements a simplified AUTOSAR-style Non-Volatile Memory (NvM) Manager using an external M24C04 EEPROM and an NXP S32K144 microcontroller.

The application demonstrates how multiple memory-write requests can be queued, processed sequentially through a state machine, and verified by reading the stored data back from the EEPROM.

The project is developed in Embedded C using NXP S32 Design Studio and the S32K144 SDK.

## 2. Objectives

- Implement a FIFO queue for EEPROM write requests.
- Develop an NvM state machine to manage write operations.
- Interface with an external M24C04 EEPROM using I2C.
- Process multiple write requests sequentially.
- Monitor state transitions and write results through UART.
- Verify EEPROM data using read-back comparison.
- Verify non-volatile data retention across a power cycle.

## 3. Hardware Requirements

- NXP S32K144 Evaluation Board
- M24C04 external EEPROM
- I2C communication interface
- UART connection for serial monitoring
- USB connection for programming and debugging

## 4. Software Requirements

- NXP S32 Design Studio
- S32K1xx SDK
- Embedded C
- PuTTY or another serial terminal

## 5. System Architecture

The application is organized into the following modules:

| File | Responsibility |
|---|---|
| `main.c` | Initializes peripherals, submits write requests, monitors states, and verifies stored data |
| `nvm.c` | Implements the NvM state machine and coordinates write operations |
| `queue.c` | Implements FIFO enqueue and dequeue operations |
| `eeprom.c` | Implements I2C-based EEPROM read and write operations |
| `nvm.h` | Defines NvM states, data structures, constants, and function declarations |
| `eeprom.h` | Declares EEPROM functions and status types |

### System Workflow

`Main Application → FIFO Job Queue → NvM State Machine → EEPROM Driver → M24C04 EEPROM`

The UART interface reports job processing, state transitions, write results, and data verification results.

### Architecture Diagram

<img width="1600" height="730" alt="image" src="https://github.com/user-attachments/assets/82dce10f-20c8-456b-96fc-dd9306ce14b0" />


## 6. NvM State Machine

The state machine processes each queued write request through the following sequence:

`IDLE → WRITE_PENDING → WRITE_BUSY → DONE → IDLE`

- **IDLE:** Checks for a queued request.
- **WRITE_PENDING:** Starts the EEPROM write operation.
- **WRITE_BUSY:** Waits for the configured EEPROM write-cycle delay.
- **DONE:** Records the job result and returns to IDLE for the next request.

The state machine is advanced by repeated calls to `NvM_MainFunction()`.

This design separates job scheduling from EEPROM hardware access and supports sequential processing of multiple write requests.

## 7. FIFO Job Queue

The FIFO queue stores write requests until the NvM Manager processes them.

Each request contains:

- EEPROM block ID
- Data buffer
- Data length

The queue uses `queueHead`, `queueTail`, and `queueCount` to manage insertion, removal, and the number of queued requests.

The configured queue capacity is eight requests.

The FIFO mechanism ensures that requests are processed in the order in which they are accepted into the queue.

## 8. EEPROM Interface

The EEPROM driver communicates with the external M24C04 using the LPI2C peripheral.

The implementation includes:

- EEPROM device-address selection based on the memory address.
- Address and data validation.
- Page-boundary checks for write operations.
- I2C-based write and read operations.
- Write-cycle delay handling.
- Read-back data verification.

The current write-cycle handling uses a fixed delay rather than I2C ACK polling.

The EEPROM provides non-volatile storage, allowing successfully written data to remain available when power is removed, provided the write cycle has completed and the device is operated within its specifications.

## 9. Testing and Verification

The application submits three write requests for EEPROM blocks 1, 2, and 3. Each block stores four bytes of test data.

After processing the queued requests, the application reads the data back from the EEPROM and compares each byte with the original expected test data.

UART output reports:

- Successful or failed queue requests.
- NvM state transitions.
- EEPROM write results.
- Read-back values.
- PASS or FAIL verification results.

### Initial Verification Procedure

1. Initialize the microcontroller peripherals.
2. Submit the three EEPROM write requests.
3. Allow the NvM state machine to process the requests sequentially.
4. Wait for the write operations to complete.
5. Read back the contents of each EEPROM block.
6. Compare the retrieved data against the expected values.
7. Confirm the verification result through UART.

This test verifies the initial write operations and read-back comparisons. Power-cycle retention is tested separately in Section 10.

## 10. Power-Cycle Verification (EEPROM Data Retention)

This test verifies whether the data stored in the external M24C04 EEPROM remains available after power is removed and restored.

### Verification Procedure

1. **Write the test data:** Run the application and allow the three EEPROM write requests to complete.
2. **Verify the initial data:** Confirm that the initial read-back comparisons report PASS.
3. **Remove power:** Disconnect the power supply to the microcontroller and EEPROM. Ensure the EEPROM write cycle has completed before removing power.
4. **Restore power:** Reconnect the power supply and restart the microcontroller.
5. **Initialize the peripherals:** Initialize the required LPI2C and UART peripherals.
6. **Read the EEPROM:** Read blocks 1, 2, and 3 without rewriting the test data.
7. **Compare the data:** Compare the retrieved bytes with the original expected values.
8. **Report the result:** Display the retrieved data and PASS or FAIL status through UART.

### Important Implementation Requirement

The firmware must support a post-restart, read-only verification mode. If the application automatically rewrites the test data at startup, the startup flow must be modified to prevent overwriting the stored data before the verification read.

The expected test data must be available independently of the EEPROM contents, for example through constant arrays in firmware.

### Verification Flow

`Power ON → Initialize Peripherals → Write Data → Initial Read-Back → Power OFF → Power ON → Initialize Peripherals → Read EEPROM → Compare Data → PASS / FAIL`

**Expected outcome:** If the EEPROM data was written successfully and retained, the post-restart read operation should retrieve the original values and report PASS for matching blocks.

Actual power-cycle retention must be confirmed by performing this procedure on the hardware and observing the UART output.

## 11. Execution Output and Expected Results

### UART Execution Output

The following output was observed during the initial execution of the application:
<img width="756" height="795" alt="image" src="https://github.com/user-attachments/assets/a9e631eb-6859-4564-befc-96a1e0f67974" />



### Observed Results

- NvM initialization completed successfully.
- All three write requests were queued.
- The state machine processed the requests sequentially.
- All three EEPROM write operations completed.
- The queue became empty after processing the requests.
- The initial read-back comparison reported PASS for all three blocks.

### Initial Read-Back Results

| EEPROM Block | Expected Data | Initial Verification |
|---|---|---|
| Block 1 | `10, 20, 30, 20` | PASS |
| Block 2 | `50, 60, 70, 80` | PASS |
| Block 3 | `90, 100, 110, 120` | PASS |

The observed UART output demonstrates successful queue processing and initial EEPROM data verification for this execution.

### Expected Results After a Power Cycle

After removing and restoring power, the application should retrieve the previously stored EEPROM data without rewriting it. If the retrieved bytes match the expected values, the post-restart verification should report PASS.

The initial UART output above does not establish power-cycle retention. That result must be documented separately after the power-cycle test has been performed.

## 12. Build, Execution and Conclusion

### Build and Execution

1. Import the project into NXP S32 Design Studio.
2. Verify the clock configuration, pin multiplexing, LPUART, and LPI2C settings.
3. Confirm the EEPROM wiring and I2C device-address configuration.
4. Build the project and resolve any compilation errors.
5. Flash the firmware onto the S32K144 evaluation board.
6. Open the configured UART COM port using PuTTY or another serial terminal.
7. Observe the queue processing, state transitions, EEPROM write results, and initial read-back verification.
8. Confirm that the initial verification reports PASS.
9. Perform the power-cycle procedure described in Section 10.
10. Run the post-restart read-only verification and confirm whether the stored data is retained.

### Conclusion

This project demonstrates a simplified AUTOSAR-style NvM Manager using a FIFO job queue, a state machine, and an external M24C04 EEPROM interfaced through I2C.

The implementation demonstrates sequential job processing, EEPROM read/write operations, UART-based monitoring, and initial read-back data comparison.

The additional power-cycle verification checks whether the stored data remains available after power is removed and restored. Successful verification requires the application to retrieve the previously stored values without rewriting them and compare them against the original expected data.

Together, these tests demonstrate queue management, state-machine operation, I2C communication, and non-volatile data retention in an embedded microcontroller application.
