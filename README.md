# PX4-Autopilot Custom Modifications (MicoAir H743-v2)

This README comprehensively documents all custom modifications made to the `PX4-Autopilot` (v1.17.0) source code to properly configure and enhance the hardware safety button behavior, as well as the board configuration for the **MicoAir H743-v2** flight controller.

## 1. Board Configuration Changes (`default.px4board`)

Several modules were disabled to save firmware space and processing power for unused features, while required sensors (like TFmini) and the safety button driver were enabled.

**Disabled Modules/Drivers:**
* `CONFIG_DRIVERS_CAMERA_CAPTURE`
* `CONFIG_DRIVERS_CAMERA_TRIGGER`
* `CONFIG_DRIVERS_DSHOT`
* `CONFIG_COMMON_OPTICAL_FLOW`
* `CONFIG_COMMON_UWB`
* `CONFIG_MODULES_GIMBAL`
* `CONFIG_MODULES_UXRCE_DDS_CLIENT`

**Enabled Modules/Drivers:**
* `CONFIG_DRIVERS_DISTANCE_SENSOR_TFMINI=y`
* `CONFIG_DRIVERS_SAFETY_BUTTON=y`
* `CONFIG_SYSTEMCMDS_UORB=y`

## 2. Hardware Pin Conflict Resolution (PB5)

The hardware safety switch was connected to the `PB5` pin, but this pin was natively configured for `UART5_RX` by default. 

**Changes Made:**
* **`boards/micoair/h743-v2/nuttx-config/nsh/defconfig`**: Commented out `CONFIG_STM32H7_UART5=y`, `CONFIG_UART5_RXBUFSIZE`, and `CONFIG_UART5_TXBUFSIZE` to disable UART5 at the OS level.
* **`boards/micoair/h743-v2/nuttx-config/include/board.h`**: Commented out the definitions mapping `GPIO_UART5_RX` and `GPIO_UART5_TX` to `PB5` and `PB6` to free the GPIO for general use.

## 3. Safety Button Pin Configuration (Active-Low / GND Toggle)

The safety button was previously set as a floating input (`GPIO_FLOAT`). To allow safe manual triggering by touching the wire to Ground (GND), the pin was reconfigured with an internal pull-up resistor and the driver logic was inverted.

**Changes Made:**
* **`boards/micoair/h743-v2/src/board_config.h`**: 
  * Replaced `GPIO_FLOAT` with `GPIO_PULLUP` for `GPIO_BTN_SAFETY`.
* **`src/drivers/safety_button/SafetyButton.cpp`**:
  * Inverted the pin read logic (`const bool button_pressed = !px4_arch_gpioread(GPIO_BTN_SAFETY);`).
  * The button now reliably triggers when safely shorted to **GND**.

## 4. Enabling the Safety Button Driver at Startup

The `safety_button` module was not enabled to compile or run by default on the MicoAir H743-v2 board.

**Changes Made:**
* **`boards/micoair/h743-v2/init/rc.board_defaults`**: Appended `safety_button start` to the board's startup script to initialize the driver upon boot.

## 5. Custom Hold-Time Logic (3s Disable / 1s Enable)

The standard PX4 safety button driver requires a fixed 1-second hold to disable the safety, and does not natively support different hold times based on the current safety state. The driver was modified to check the `vehicle_status` and apply dynamic timings.

**Changes Made:**
* **`src/drivers/safety_button/SafetyButton.hpp`**: 
  * Included `<uORB/topics/vehicle_status.h>`.
  * Added a `uORB::Subscription` for `_vehicle_status_sub` to read the live safety state.
* **`src/drivers/safety_button/SafetyButton.cpp`**: 
  * Replaced the static `CYCLE_COUNT` constant with a dynamic `target_cycle_count`.
  * Added logic to read `status.safety_off`.
  * **Timing**: Requires 90 cycles (3 seconds) to disable safety, and 30 cycles (1 second) to re-enable safety.

## 6. Toggle Capability Enhancement

By default, the PX4 Commander module uses the hardware safety button as a one-way switch (it can only unlock the safety; it cannot lock it again). This was modified to act as a two-way toggle switch.

**Changes Made:**
* **`src/modules/commander/Safety.cpp`**: 
  * Replaced the one-way `_safety_off |= button_event.triggered;` logic with a true toggle: `_safety_off = !_safety_off;`.

> **⚠️ WARNING:** Because the safety button is now a true toggle switch, it is active at all times. **Pressing it while the drone is in flight will immediately cut the motors.** Ensure the switch is safely mounted where it cannot be accidentally triggered!
