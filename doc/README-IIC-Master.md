# IIC Master 

## Overview

This Software Building Block demonstrates use of the Renesas FSP IIC Master HAL APIs to communicate with a Digilent Pmod ACL board containing an ADXL345 accelerometer. The reference application opens the IIC master, verifies the sensor device ID, enables measurement, periodically reads the six X/Y/Z data bytes, converts the byte pairs to signed 16-bit axis values, and prints the results to a serial terminal. Communication and initialization errors are also reported on the terminal.

This documentation applies only to these reference boards:

- FPB-RA0L1
- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

## 1. What use case does this SBB demonstrate?

The SBB demonstrates a basic MCU-to-sensor I2C/IIC master use case. A Renesas RA MCU acts as the IIC master and communicates with the ADXL345 accelerometer on the Pmod ACL board. After initialization, the application repeatedly reads acceleration data for the X, Y, and Z axes and displays it through the UART-backed console.

The example is intended to show the typical sequence for using an FSP IIC Master HAL module:

1. Initialize the console used by the reference application.
2. Open the configured IIC Master instance.
3. Read the ADXL345 device ID and verify the expected signature (`0xE5`).
4. Write the ADXL345 `POWER_CTL` register (`0x2D`) to enable measurement.
5. Select the axis-data register (`0x32`) using an IIC write with repeated start.
6. Read six bytes containing the X, Y, and Z axis data.
7. Convert each byte pair into a signed 16-bit axis value.
8. Print the values on the serial terminal.
9. Repeat after the configured reference-application delay.

## 2. What does the SBB core code do?

The reusable sensor-facing code is implemented in `iic_sensor.c` and `iic_sensor.h`.

The core code:

- Provides a small public interface for sensor initialization, data acquisition, and deinitialization.
- Opens the selected FSP IIC Master driver instance through compatibility macros.
- Reads and validates the ADXL345 device ID.
- Enables ADXL345 measurement mode.
- Performs IIC writes and reads.
- Uses the IIC Master callback to capture transfer-completion events.
- Waits for the asynchronous transfer event and returns an error when a transfer aborts or the local polling timeout expires.
- Reads the six raw accelerometer data bytes into a caller-provided buffer.
- Closes the IIC Master module during deinitialization.

The implementation selects the FSP master API at compile time. It can map the SBB operations to `r_iica_master`, `r_iic_b_master`, or `r_iic_master`, depending on the MCU/board configuration.

## 3. Demo/reference application code versus reusable SBB code

### Reusable SBB code

The primary reusable interface is contained in:

- `iic_sensor.c`
- `iic_sensor.h`

These files contain the sensor and IIC transaction logic. An application can call this interface without reproducing the reference application's terminal formatting and continuous demonstration loop.

### Demo/reference application code

The following files form the demonstration application layer:

- `iic_master_example_app.c`
- `iic_master_example_app.h`

The reference application:

- Initializes the UART console.
- Retrieves and prints the FSP version.
- Prints the SBB information banner.
- Calls the reusable sensor API.
- Converts raw axis bytes to signed values for display.
- Prints X/Y/Z values.
- Delays three seconds between reads.
- Prints an error, deinitializes the sensor, and enters a breakpoint trap when an SBB operation fails.

`iic_master_example_app_run()` is the reference application's single entry point intended to be called by `hal_entry.c`.

## 4. Which public functions should I look at first?

Start with these functions from `iic_sensor.h`:

### `fsp_err_t init_sensor(void)`

Initializes the configured IIC Master module, reads the ADXL345 device ID, verifies that it equals `DEVICE_SIGNATURE` (`0xE5`), and enables measurement through the ADXL345 `POWER_CTL` register.

### `fsp_err_t read_sensor_data(uint8_t * xyz_data)`

Reads the six raw bytes containing X, Y, and Z acceleration data. The caller must provide a valid buffer large enough for `SENSOR_DATA_SIZE` bytes.

### `void deinit_sensor(void)`

Closes the IIC Master module when it is open.

For understanding the complete reference flow, also review:

### `void iic_master_example_app_run(void)`

This is the demo entry point. It shows the intended order of console initialization, SBB initialization, repeated sensor reads, data conversion, output, and error handling.

The IIC callback, `i2c_master_callback()`, is also important when adapting the configuration because it updates the event used by the core transfer-completion logic.

## 5. What configuration assumptions were made for the reference boards?

### Supported reference boards

Only the following boards are covered by this SBB documentation.

| Board | SDA connection | SCL connection | Power | Ground | Additional reference-board requirement |
|---|---|---|---|---|---|
| FPB-RA0L1 | SDAA0 P913, J1:15 | SCLA0 P914, J1:14 | J1:9 | J1:6 | Place J9 on pins 1-2 to use on-board debug functionality. |
| FPB-RA2E3 | SDA0 P401, J3:2 | SCL0 P400, J3:1 | J3:9 | J3:6 | Close E25, E28, E46, and E47 to use J-Link OB VCOM for the serial terminal. |
| FPB-RA4E2 | SDA P101, J4:15 | SCL P100, J4:16 | J4:7 | J4:8 | No additional board-specific requirement is stated in the source README. |
| FPB-RA6E2 | SDA P101, J4:15 | SCL P100, J4:16 | J4:7 | J4:8 | No additional board-specific requirement is stated in the source README. |
| FPB-RA8E1 | SDA1 P511, J4:4 | SCL1 P512, J4:3 | J4:1 | J4:2 | Open J6 and J9 to use on-board debug functionality. |

### IIC electrical assumptions

- SDA and SCL must be configured for n-channel open-drain output.
- The source README states that external 3.9 kΩ or 4.7 kΩ I2C pull-up resistors are required for supported boards except a listed set. Among the five boards documented here, FPB-RA2E3, FPB-RA4E2, FPB-RA6E2, and FPB-RA8E1 are in the exception list. FPB-RA0L1 is not in that exception list, so the reference information requires external 3.9 kΩ or 4.7 kΩ pull-ups for FPB-RA0L1.

### IIC/Sensor assumptions

The reference implementation assumes:

- A Pmod ACL board with an ADXL345 accelerometer.
- ADXL345 device signature `0xE5` at register `0x00`.
- `POWER_CTL` register at `0x2D`.
- Measurement-enable value `0x08`.
- Axis data beginning at register `0x32`.
- Six bytes of X/Y/Z sensor data per sample.
- The FSP-generated IIC instance symbols `g_i2c_master_ctrl` and `g_i2c_master_cfg` are available through `hal_data.h`.
- The IIC configuration supplies `i2c_master_callback()` as the callback used to signal transaction completion.

### Serial terminal assumptions

The reference README specifies these terminal settings:

- Speed: 115200 baud
- Data: 8 bits
- Parity: none
- Stop bits: 1
- Flow control: none
- Receive newline: CR
- Transmit newline: CR
- Local echo: enabled

The serial port is provided by J-Link On-Board in the reference setup.

For the GCC Arm Embedded Toolchain, the README also requires **Use float with nano printf** when floating-point values are printed to the terminal.

## 6. What must I change for my board, pins, clocks, timing, memory, task priorities, or middleware setup?

When porting the SBB, review the items below against the new target. The supplied files provide concrete reference settings for pins and application timing, but they do not specify custom-board clock frequencies, memory budgets, RTOS task priorities, or middleware configuration values. Those items therefore must be established from the target project's requirements and FSP configuration rather than copied from this documentation.

### Board and pins

Update the FSP pin configuration and physical connections so the selected IIC peripheral is routed to the target SDA and SCL pins. Ensure both signals use the required open-drain configuration. Provide suitable I2C pull-ups when they are not already present on the target board or attached sensor board.

### FSP IIC instance

The SBB expects the generated objects:

- `g_i2c_master_ctrl`
- `g_i2c_master_cfg`

If the target project uses different generated instance names, adapt the SBB references or align the FSP configuration with these names.

The header chooses among IICA, IIC_B, and IIC Master APIs using MCU/board build definitions. Verify that the selected target resolves to the intended FSP driver.

### Sensor configuration

If the target is not the same ADXL345/Pmod ACL setup, the following definitions and transaction sequence are sensor-specific and must be replaced as appropriate:

- `DEVICE_ID_REG`
- `DEVICE_SIGNATURE`
- `POWER_CTL_REG`
- `ENABLE_BIT`
- `AXIS_DATA`
- Data length and byte interpretation

### Timing

The demo uses `SENSOR_READ_DELAY` with `R_BSP_SoftwareDelay()` and the current constant is three seconds. This delay belongs to the reference application behavior rather than the low-level IIC transfer interface. Change it to match the sampling requirements of the target application.

The internal transfer-completion check uses a decrementing `uint16_t` polling counter. Its duration is not expressed as a time interval in the supplied source. If deterministic timeout behavior is required on another target, replace or redesign this mechanism using timing appropriate to that application.

### Clocks, memory, RTOS priorities, and middleware

The supplied files do not document specific clock-tree values, memory allocations, RTOS task priorities, or middleware setup. Validate these items in the target FSP project when integrating the SBB. The supplied SBB code itself uses a callback plus foreground polling for IIC completion and does not contain RTOS task-management code.

### Console

The reusable sensor layer does not perform application logging. The demo layer depends on `console/uart_console.h` and standard `printf()` routing. If the target application uses a different console or no console, replace or remove the reference application's console-specific code while retaining the SBB sensor interface as needed.

## 7. What dependencies does the SBB have?

### FSP dependencies

The source directly depends on Renesas FSP types and generated configuration through `hal_data.h`. Depending on the target MCU/board build, the IIC wrapper macros call one of:

- `R_IICA_MASTER_Open/Write/Read/Close`
- `R_IIC_B_MASTER_Open/Write/Read/Close`
- `R_IIC_MASTER_Open/Write/Read/Close`

The reference application additionally uses:

- `R_FSP_VersionGet()`
- `R_BSP_SoftwareDelay()`

### Driver/configuration dependencies

The SBB expects:

- A configured FSP IIC Master instance.
- `g_i2c_master_ctrl`.
- `g_i2c_master_cfg`.
- `i2c_master_callback()` connected to the IIC Master callback configuration.

### Demo-only service dependency

The reference application includes `console/uart_console.h` and uses:

- `uart_console_init()`
- `uart_console_flush()`
- `printf()`

These are demonstration/output dependencies rather than sensor transaction APIs.

### Hardware dependencies

The demonstrated setup requires:

- One of the five FPB boards listed in this document.
- A Pmod ACL/ADXL345 sensor board.
- SDA, SCL, power, and ground connections.
- A USB connection for programming/debugging.
- Board-specific jumper or VCOM settings where listed above.
- Appropriate I2C pull-ups where required.

## 8. What behavior and performance were demonstrated or measured?

### Demonstrated behavior

The supplied implementation demonstrates:

- Opening the IIC Master interface.
- Confirming communication by reading the ADXL345 device ID.
- Enabling accelerometer measurement.
- Reading six bytes of axis data.
- Reconstructing signed X/Y/Z values from three little-endian byte pairs.
- Printing X/Y/Z values to the UART-backed terminal.
- Repeating the read after a three-second software delay.
- Reporting FSP errors and stopping at a breakpoint when an initialization or read operation fails.

### Performance data

The supplied source and README do not provide measured IIC throughput, CPU utilization, interrupt latency, transaction latency, power consumption, memory footprint, or maximum sustainable sample rate. The three-second read interval is a reference-application delay and should not be interpreted as a measured performance limit of the IIC peripheral, FSP driver, ADXL345, or SBB.

## 9. What files are safe to modify, and what files are generated, preset, or demo-only?

Based on the supplied files, the intended separation is:

### Reusable SBB source

- `iic_sensor.c`
- `iic_sensor.h`

These contain the core behavior that should be carried into an application when the same sensor/IIC abstraction is desired. Sensor register definitions and target-selection wrappers are located here, so changes required for another sensor or different generated IIC instance may affect these files.

### Demo/reference application

- `iic_master_example_app.c`
- `iic_master_example_app.h`

These files are appropriate places to change demonstration behavior such as terminal messages, sample presentation, read-loop behavior, and the reference application's sampling delay.

### Generated/preset project content

The supplied source references FSP-generated objects through `hal_data.h`, but the attached file set does not include or identify the complete set of generated FSP project files. Treat FSP-generated configuration output as generated content and make peripheral/pin configuration changes through the project's FSP configuration workflow rather than relying on manual edits to generated code.

## 10. What happens if I modify the source and later adopt a newer SBB release?

Local changes to reusable SBB files can conflict with changes made in a newer SBB release. To reduce upgrade effort:

- Keep board/application behavior in the reference/application layer when it does not require a change to the reusable sensor implementation.
- Keep target hardware configuration in the FSP project configuration where possible.
- Minimize changes to the public SBB interface (`init_sensor()`, `read_sensor_data()`, and `deinit_sensor()`) when application compatibility across releases is important.
- Track intentional changes to `iic_sensor.c` and `iic_sensor.h` in source control so they can be reviewed and reapplied or merged when adopting a newer SBB version.
- Compare a newer SBB release against the locally modified reusable files before replacing them. Do not assume modified files can be overwritten without losing application-specific changes.

The supplied README and source do not define an automated migration or backward-compatibility mechanism between releases, so source-level differences must be reviewed when integrating a newer release.

## Reference Application Flow

```text
hal_entry()
    |
    +--> iic_master_example_app_run()
             |
             +--> uart_console_init()
             |
             +--> init_sensor()
             |      |
             |      +--> Open configured FSP IIC Master
             |      +--> Read and verify ADXL345 device ID
             |      +--> Enable measurement mode
             |
             +--> Loop
                    |
                    +--> read_sensor_data()
                    |      +--> Select axis-data register
                    |      +--> Read six X/Y/Z bytes
                    |
                    +--> Convert byte pairs to signed axis values
                    +--> Print X/Y/Z values
                    +--> Delay
```

## Integration Checklist

Before reusing this SBB on another board or in another application, verify:

- [ ] The required FSP IIC Master driver is available for the target MCU.
- [ ] `g_i2c_master_ctrl` and `g_i2c_master_cfg` match the intended generated instance, or the source is adapted accordingly.
- [ ] `i2c_master_callback()` is connected to the configured IIC Master callback.
- [ ] SDA/SCL pins are routed correctly and configured as n-channel open drain.
- [ ] Required I2C pull-up resistors are present.
- [ ] The connected device uses the ADXL345 register map assumed by the SBB, or the sensor-specific definitions/transactions have been updated.
- [ ] The application's sampling interval is appropriate rather than copied blindly from the three-second demo delay.
- [ ] UART/console dependencies are retained only if terminal output is required.
- [ ] Floating-point `printf` support is enabled when using the GCC Arm Embedded Toolchain and keeping the demo's floating-point terminal output.
