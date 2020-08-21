---
title: Multi-SPI I/O Requests
description: SPBCx contains support for multi-SPI, including both dual and quad SPI transfers.
ms.assetid: C80FE3F2-6659-4DE8-8F77-F77EDA60400F
ms.date: 04/20/2017
ms.localizationpriority: medium
---

# Multi-SPI I/O Requests

SPBCx contains support for multi-SPI, including both dual and quad SPI transfers. These transfers are issued to an SPB controller using the [**IOCTL\_SPB\_MULTI\_SPI\_TRANSFER**](about:blank) I/O control code (IOCTL) included with the simple peripheral bus (SPB) I/O request interface.

Only SPB controller drivers for SPI controllers that support additional multi-SPI modes should support the **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** IOCTL. If an SPB controller driver does not support multi-SPI transfers, the driver fails all **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** requests that it receives and completes them with the error status code STATUS\_NOT\_SUPPORTED.

## Multi-SPI Transfer Structure

An [**IOCTL\_SPB\_MULTI\_SPI\_TRANSFER**](about:blank) request contains a custom input buffer structure, [**SPB_MULTI_SPI_TRANSFER**](about:blank). This structure allows for the following to be specified:
-   Dual or quad SPI transfer modes.
-   One or two transfer phases - a _write_ phase, followed by an optional _read_ phase.
-   A variable number of bytes to be transmitted at the beginning of the _write_ phase in single-SPI mode, before switching to the specified multi-SPI mode.
-   Where a _read_ phase is provided, a variable number of _wait cycles_ between _write_ and _read_ phases - clock cycles where no data is to be transferred.

The following restrictions apply to this structure:
-   The [**SPB\_TRANSFER\_LIST**](https://docs.microsoft.com/windows-hardware/drivers/ddi/spb/ns-spb-spb_transfer_list) _TransferPhases_ structure in the request must contain exactly one or two entries. The first entry describes a buffer that contains data to write to the device. The second, optional, entry describes a buffer used to hold data read from the device.
-   Each [**SPB\_TRANSFER\_LIST\_ENTRY**](https://docs.microsoft.com/windows-hardware/drivers/ddi/spb/ns-spb-spb_transfer_list_entry) structure in the transfer list must specify a **DelayInUs** value of zero.

Custom structures & macros are provided in the SpbCx interface to initialize the [**SPB_MULTI_SPI_TRANSFER**](about:blank) structures required to construct a multi-SPI transfer request.

The following code example shows how the driver for an SPB peripheral device builds a transfer list for a quad-SPI read operation, using the **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** request.

```cpp
SPB_MULTI_SPI_READ_TRANSFER readTransfer;

SPB_MULTI_SPI_READ_TRANSFER_INIT(&readTransfer,
                                 SpbMultiSpiTransferModeQuadSpi, // Transfer Mode
                                 1,                // 1 single-SPI byte for opcode
                                 2                 // 2 wait cycle bytes (4 cycles)
);

// Setup the write phase
readTransfer.TransferPhases[0] = SPB_TRANSFER_LIST_ENTRY_INIT_SIMPLE(
                                    SpbTransferDirectionToDevice,
                                    0,
                                    pWriteBuffer,
                                    writeBufferLength);

// Setup the read phase
readTransfer.TransferPhases[1] = SPB_TRANSFER_LIST_ENTRY_INIT_SIMPLE(
                                    SpbTransferDirectionFromDevice,
                                    0,
                                    pReadBuffer,
                                    readBufferLength);
```

The following code example shows a dual-SPI write operation:

```cpp
SPB_MULTI_SPI_WRITE_TRANSFER writeTransfer;

SPB_MULTI_SPI_WRITE_TRANSFER_INIT(&writeTransfer,
                                  SpbMultiSpiTransferModeDualSpi, // Transfer Mode
                                  1,                // 1 single-SPI byte for opcode
                                  0                 // no wait cycles (no read phase)
);

// Setup the write phase
writeTransfer.TransferPhases[0] = SPB_TRANSFER_LIST_ENTRY_INIT_SIMPLE(
                                    SpbTransferDirectionToDevice,
                                    0,
                                    pWriteBuffer,
                                    writeBufferLength);
```

The preceding code examples use the [**SPB\_MULTI\_SPI\_WRITE\_TRANSFER**](about:blank) and [**SPB\_MULTI\_SPI\_READ\_TRANSFER**](about:blank) structs, as well as the corresponding [**SPB\_MULTI\_SPI\_WRITE\_TRANSFER_INIT**](about:blank) and [**SPB\_MULTI\_SPI\_WRITE\_TRANSFER\_INIT**](about:blank) macros to initialize them.

## Multi-SPI Bus Transfers

In response to an **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** request from an SPB peripheral device driver, the SPB controller driver initiates a multi-SPI transfer on the bus.

During a multi-SPI transfer, the controller is expected to execute the following operations:
1.  If _WritePhaseSingleSpiByteCount_ is zero, goto (3)
2.  Transmit in single-SPI mode _WritePhaseSingleSpiByteCount_ bytes of the _write_ phase buffer, provided in the first entry of the _TransferPhases_ transfer list.
3.  Determine the number of remaining bytes to transmit, by subtracting both the _WaitCycleByteCount_, and _WritePhaseSingleSpiByteCount_ from the total size of the _write_ phase buffer.
5.  Transmit remaining bytes in specified multi-SPI (dual or quad) mode.
6.  If no read phase is provided in the _TransferPhases_ list, transfer is completed.
7.  Execute the wait cycles, through either internal controller configuration or by transmitting remaining _WaitCycleByteCount_ from the end of the _write_ phase buffer.
8.  Switch modes and begin receiving bytes from the bus, into the _read_ phase buffer, provided in the second entry of the _TransferPhases_ transfer list.


## Example: WDF Driver

The Windows Driver Foundation (WDF) driver for an SPB peripheral device calls a method such as [**WdfIoTargetSendIoctlSynchronously**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdfiotarget/nf-wdfiotarget-wdfiotargetsendioctlsynchronously) to send an IOCTL request to an SPB controller. This method has *InputBuffer* and *OutputBuffer* parameters. Drivers for some types of devices might use these two parameters to point to the write buffer and the read buffer, respectively, for an IOCTL request. However, to send an IOCTL request to an SPB controller, the SPB peripheral device driver sets the *InputBuffer* parameter to point to a memory descriptor that points to a **SPB_MULTI_SPI_TRANSFER** structure (initialized using the aforementioned macros). For example, this structure describes both the write buffer and the read buffer (in that order) for an **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** request. The driver sets the *OutputBuffer* parameter to NULL.

The following code example shows a **WdfIoTargetSendIoctlSynchronously** call that sends an **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** request to an SPB peripheral device. The `writeTransfer` variable in this example is a transfer list that was defined in the code example in [Transfer List](#multi-spi-transfer-structure).

```cpp
ULONG_PTR BytesTransferred = 0;
NTSTATUS Status;
  
WDF_MEMORY_DESCRIPTOR MemoryDescriptor;
WDF_MEMORY_DESCRIPTOR_INIT_BUFFER(
                            &MemoryDescriptor,  
                            &writeTransfer,  
                            sizeof(writeTransfer));

Status = WdfIoTargetSendIoctlSynchronously(
                            SpbIoTarget,
                            NULL,
                            IOCTL_SPB_FULL_DUPLEX,
                            &MemoryDescriptor,  // InputBuffer
                            NULL,               // OutputBuffer
                            NULL,
                            &BytesTransferred);
```

In the preceding code example, the `MemoryDescriptor` variable is a framework memory descriptor. The [**WDF\_MEMORY\_DESCRIPTOR\_INIT\_BUFFER**](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdfmemory/nf-wdfmemory-wdf_memory_descriptor_init_buffer) macro initializes this descriptor to serve as a container for the structure contained in the `writeTransfer` variable. In the call to the **WdfIoTargetSendIoctlSynchronously** method, the `SpbIoTarget` variable is a previously opened handle to the target peripheral device on the bus. The *InputBuffer* parameter to this method is a pointer to the memory descriptor. The *OutputBuffer* parameter is NULL.

The **WdfIoTargetSendIoctlSynchronously** call in this code example sets the `BytesTransferred` variable to the total number of bytes transferred (bytes written (including wait cycle bytes) plus bytes read).