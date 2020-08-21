---
title: Handling Multi-SPI (IOCTL_SPB_MULTI_SPI_TRANSFER) Transfer Requests
description: SPBCx contains support for multi-SPI, including both dual and quad SPI transfers.
ms.assetid: B200461F-9F9C-43A7-BA78-0864FD58C64E
ms.date: 04/20/2017
ms.localizationpriority: medium
---

# Handling Multi-SPI (IOCTL\_SPB\_MULTI\_SPI\_TRANSFER) Transfer Requests

SPBCx contains support for multi-SPI, including both dual and quad SPI transfers. These transfers are issued to an SPB controller using the [**IOCTL\_SPB\_MULTI\_SPI\_TRANSFER**](about:blank) I/O control code (IOCTL) included with the simple peripheral bus (SPB) I/O request interface. Only SPB controller drivers for SPI controllers that support additional multi-SPI modes should support the **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** IOCTL.

If an SPB controller driver supports I/O requests for multi-SPI transfers, the driver should only use the **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** IOCTL for these requests, and should follow the implementation guidelines presented in this topic. The purpose of these guidelines is to encourage uniform behavior across all hardware platforms that support **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** requests. Drivers for SPB-connected peripheral devices can then rely on these requests to produce similar results regardless of what hardware platform they are run on.

## Buffer Requirements

An [**IOCTL\_SPB\_MULTI\_SPI\_TRANSFER**](about:blank) request contains a custom input buffer structure, [**SPB_MULTI_SPI_TRANSFER**](about:blank). This structure allows for the following to be specified:
-   Dual or quad SPI transfer modes.
-   One or two transfer phases - a _write_ phase, followed by an optional _read_ phase.
-   A variable number of bytes to be transmitted at the beginning of the _write_ phase in single-SPI mode, before switching to the specified multi-SPI mode.
-   Where a _read_ phase is provided, a variable number of _wait cycles_ between _write_ and _read_ phases - clock cycles where no data is to be transferred.

The following restrictions apply to this structure:
-   The [**SPB\_TRANSFER\_LIST**](https://docs.microsoft.com/windows-hardware/drivers/ddi/spb/ns-spb-spb_transfer_list) _TransferPhases_ structure in the request must contain exactly one or two entries. The first entry describes a buffer that contains data to write to the device. The second, optional, entry describes a buffer used to hold data read from the device.
-   Each [**SPB\_TRANSFER\_LIST\_ENTRY**](https://docs.microsoft.com/windows-hardware/drivers/ddi/spb/ns-spb-spb_transfer_list_entry) structure in the transfer list must specify a **DelayInUs** value of zero.


## Transfer Flow
During a multi-SPI transfer, the controller is expected to execute the following operations:
1.  If _WritePhaseSingleSpiByteCount_ is zero, goto (3)
2.  Transmit in single-SPI mode _WritePhaseSingleSpiByteCount_ bytes of the _write_ phase buffer, provided in the first entry of the _TransferPhases_ transfer list.
3.  Determine the number of remaining bytes to transmit, by subtracting both the _WaitCycleByteCount_, and _WritePhaseSingleSpiByteCount_ from the total size of the _write_ phase buffer.
5.  Transmit remaining bytes in specified multi-SPI (dual or quad) mode.
6.  If no read phase is provided in the _TransferPhases_ list, transfer is completed.
7.  Execute the wait cycles, through either internal controller configuration or by transmitting remaining _WaitCycleByteCount_ from the end of the _write_ phase buffer.
8.  Switch modes and begin receiving bytes from the bus, into the _read_ phase buffer, provided in the second entry of the _TransferPhases_ transfer list.


## Request Handling
Although the [**IOCTL\_SPB\_MULTI\_SPI\_TRANSFER**](about:blank) uses a transfer list for the transfer stages, the request uses a custom parameter structure [**SPB_MULTI_SPI_TRANSFER**](about:blank), and parameters will not be automatically validated or buffers captured by SpbCx.

As with **IOCTL\_SPB\_FULL\_DUPLEX**, SpbCx treats the **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** request as a custom, driver-defined IOCTL request. SpbCx passes **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** requests to the SPB controller driver through the driver's [*EvtSpbControllerIoOther*](https://docs.microsoft.com/windows-hardware/drivers/ddi/spbcx/nc-spbcx-evt_spb_controller_other) callback function, which also handles any custom IOCTL requests that the driver supports. SpbCx does no initial parameter checking or buffer capture for these requests. The driver is responsible for calling into SpbCx or performing any parameter checking or buffer capture that might be required for the IOCTL requests that the driver receives through its *EvtSpbControllerIoOther* function.

To enable buffer capture, the driver must supply an [*EvtIoInCallerContext*](https://docs.microsoft.com/windows-hardware/drivers/ddi/wdfdevice/nc-wdfdevice-evt_wdf_io_in_caller_context) callback function when the driver registers its *EvtSpbControllerIoOther* function. The SPB controller driver must call the [**SpbRequestCaptureIoOtherTransferList**](about:blank) method to validate the request and capture these buffers in the process context of the request originator. The driver may then call the [**SpbRequestGetTransferParameters**](https://docs.microsoft.com/windows-hardware/drivers/ddi/spbcx/nf-spbcx-spbrequestgettransferparameters) method to access these buffers, as it would with an **IOCTL\_SPB\_FULL\_DUPLEX** request.

The following code example shows an *EvtIoInCallerContext* function that is implemented by an SPI controller driver supporting both regular single-SPI (through **IOCTL\_SPB\_FULL\_DUPLEX**), and multi-SPI (through **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER**) requests, in order to capture the transfer lists for these SPI requests.

```cpp
VOID
EvtIoInCallerContext(
    _In_  WDFDEVICE   SpbController,
    _In_  WDFREQUEST  FxRequest
    ) 
{
    NTSTATUS status = STATUS_SUCCESS;
    WDF_REQUEST_PARAMETERS fxParams;
  
    WDF_REQUEST_PARAMETERS_INIT(&fxParams);
    WdfRequestGetParameters(FxRequest, &fxParams);

    if ((fxParams.Type != WdfRequestTypeDeviceControl) &&
        (fxParams.Type != WdfRequestTypeDeviceControlInternal))
    {
        status = STATUS_NOT_SUPPORTED;
        goto exit;
    }

    //
    // The driver should check for custom IOCTLs that it handles.
    // If the IOCTL is not recognized, complete the request with a
    // status of STATUS_NOT_SUPPORTED.
    //

    switch (fxParams.Parameters.DeviceIoControl.IoControlCode)
    {
    case IOCTL_SPB_FULL_DUPLEX:
    case IOCTL_SPB_MULTI_SPI_TRANSFER:
        break;
    default:
        status = STATUS_NOT_SUPPORTED;
        goto exit;
    }

    //
    // The IOCTL is recognized. Capture the buffers in the request.
    //
    status = SpbRequestCaptureIoOtherTransferList((SPBREQUEST)FxRequest);

    //
    // If the capture fails, the driver must complete the request instead
    // of placing it in the SPB controller's request queue.
    //

    if (!NT_SUCCESS(status))
    {
        goto exit;
    }

    status = WdfDeviceEnqueueRequest(SpbController, FxRequest);

    if (!NT_SUCCESS(status))
    {
        goto exit;
    }

exit:

    if (!NT_SUCCESS(status))
    {
        WdfRequestComplete(FxRequest, status);
    }
}
```

In the preceding code example, the **switch** statement verifies that the request contains an IOCTL that the SPI SPB controller driver recognizes.  Next, the call to either the **SpbRequestCaptureIoOtherTransferList** method captures the buffers in the transfer list of the request. If this call succeeds, the request is added to the SPB controller's I/O queue. Otherwise, the request is completed with an error status code.

Typically, the SPB controller driver validates, and retrieves additional transfer parameter values of an **IOCTL\_SPB\_MULTI\_SPI\_TRANSFER** request in the *EvtSpbControllerIoOther* function instead of in the *EvtIoInCallerContext* function. The provided **SpbRequestCaptureIoOtherTransferList** method validates the following:
-   The **SPB_MULTI_SPI_TRANSFER** input parameter structure is valid and the correct size.
-   There are 1 or 2 transfer phases provided.
-   The first entry in the transfer list is for a write buffer, and if present, the second is for a read buffer.
-   The transfer mode is valid (dual or quad-SPI).
-   The wait cycle count is zero for single-phase transfers.
-   The **DelayInUs** value for both entries is zero.

The following code example shows how a quad-SPI controller driver might implement the remainder of required checks. In this example, the driver verifies that the following parameter requirements are satisfied:
-   The write phase buffer is sized correctly with respect to *WritePhaseSingleSpiByteCount* and *WaitCycleByteCount*
-   The controller supports the requested transfer modes (in this example, controller only supports quad-SPI with a single opcode byte).

```cpp
//
// Retrieve the SPB_MULTI_SPI_TRANSFER_HEADER parameter structure.
// This structure should always be requested instead of the whole SPB_MULTI_SPI_TRANSFER,
// as accessing the transfer list from the SPB_MULTI_SPI_TRANSFER is not safe in this undefined context.
//
PSPB_MULTI_SPI_TRANSFER_HEADER transferParameters;

status = WdfRequestRetrieveInputBuffer(SpbRequest,
                                       sizeof(SPB_MULTI_SPI_TRANSFER_HEADER),
                                       (PVOID*) &transferParameters,
                                       NULL);

//
// Validate controller supports transfer mode
// This example controller only supports quad SPI transfers, with a single
// single-SPI byte for an opcode.
//
if (transferParameters->Mode != SpbMultiSpiTransferModeQuadSpi) {
    status = STATUS_NOT_SUPPORTED;
    goto exit;
}

if (transferParameters->WritePhaseSingleSpiByteCount != 1) {
    status = STATUS_NOT_SUPPORTED;
    goto exit;
}

//
// Validate the transfer count.
// SpbRequestCaptureIoOtherTransferList has already validated we have either 1 or 2 phases.
//
SPB_REQUEST_PARAMETERS params;
SPB_REQUEST_PARAMETERS_INIT(&params);
SpbRequestGetParameters(SpbRequest, &params);
BOOLEAN hasReadPhase = FALSE;

if (params.SequenceTransferCount == 2) {
    hasReadPhase = TRUE;
}

//
// Retrieve the write and read transfer descriptors.
//

const ULONG writePhaseTransferIndex = 0;
const ULONG readPhaseTransferIndex = 1;

SPB_TRANSFER_DESCRIPTOR writeDescriptor;
SPB_TRANSFER_DESCRIPTOR readDescriptor;
PMDL pWriteMdl;
PMDL pReadMdl;

SPB_TRANSFER_DESCRIPTOR_INIT(&writeDescriptor);
SPB_TRANSFER_DESCRIPTOR_INIT(&readDescriptor);

SpbRequestGetTransferParameters(
    SpbRequest, 
    writePhaseTransferIndex, 
    &writeDescriptor,
    &pWriteMdl);

if (hasReadPhase) {
    SpbRequestGetTransferParameters(
        SpbRequest, 
        readPhaseTransferIndex, 
        &readDescriptor,
        &pReadMdl);
}

//
// Validate the write phase buffer is sufficient to hold the wait cycle bytes
// and the single-SPI bytes (in this case, opcode byte).
// Here, we allow for case where wait cycles immediately follow the opcode.
//
ULONG requiredWritePhaseSize = 0;

status = RtlULongAdd(transferParameters->WritePhaseSingleSpiByteCount, transferParameters->WaitCycleByteCount, &requiredWritePhaseSize);
if (!NT_SUCCESS(status)) {
    goto exit;
}

if (writeDescriptor.TransferLength < requiredWritePhaseSize) {
    status = STATUS_INVALID_PARAMETER;
    goto exit;
}

MyDriverPerformQuadSpiTransfer(
    pDevice, 
    pRequest,
    transferParameters->WaitCycleByteCount,
    writeDescriptor,
    pWriteMdl,
    hasReadPhase ? readDescriptor : NULL,
    hasReadPhase ? pReadMdl : NULL);
```

After checking the parameter values, the preceding code example calls a driver-internal routine, named `MyDriverPerformQuadSpiTransfer`, to initiate the actual quad-SPI I/O transfer.