# Changes in this fork

The changes this fork makes on top of Bitcraze's crazyflie-firmware.

Based on: https://github.com/bitcraze/crazyflie-firmware at `9cf9d86c7efe6a94e30b917c413477560deee5b4`
(release `2026.08` plus 58 commits).

## Bug fixes

**A healthy radio link could read as lost, deleting every log block.**
Under heavy radio traffic, all log blocks sometimes stopped at once while the link stayed up.
`radiolinkIsConnected()` read the tick before `lastPacketTick`; a packet stamped in between made
`now - last` wrap to about 2^32, so `logRunBlock()` treated the link as lost and called `logReset()`.
`commanderGetInactivityTime()` had the same race, which can shut the system down when
`CONFIG_PM_AUTO_SHUTDOWN` is enabled. Both now read the stamp (made `volatile`) before the tick. (ea748cb2b)

**Resetting the ESP32 into its bootloader rebooted the STM32 while the deck was talking.**
While the AI-deck sent CPX traffic, the ESP32 reset-to-bootloader command was never acknowledged, so
ESP32 flashing failed. `cpxUARTTransportDeinit()` sets `shutdownTransport`, but the external router keeps
calling `cpxUARTTransportReceive()` and `cpxUARTTransportSend()`, and both asserted that the flag was clear.
Now receive keeps draining, send drops the packet, and the TX task also wakes on `DEINIT_EVENT`, so
deinit cannot hang waiting for a CTS that never comes. (3a06d0f53)

**A corrupt CPX frame from the deck rebooted the STM32.**
GAP8 flashes and cold boots sometimes stalled at 0%: noise on the UART while the deck boots reads as a
frame, and `CPX_UART_RX()` asserted on the bad CRC. A frame with a bad CRC, or a length outside 2..MTU,
is now dropped and the receiver resyncs on the next start byte; a length above the MTU no longer
overruns `uartRxp`, and the ESP32 still gets its clear-to-receive so the link keeps flowing.
(9dbb05db8)

**A GAP8 flash rebooted the STM32 after the image was written.**
After the last chunk, `gap8DeckFlasherWrite()` waited a fixed 5 s for the GAP8 bootloader's MD5 reply and
then asserted. The bootloader hashes about 270 kB/s, so every image over about 1.3 MB, and any lost reply,
rebooted the STM32 at the very end of the flash. The wait is now 5 s plus 5 µs per byte, and a missing
reply is printed (`GAP8 md5: no reply within <N> ms`) instead of asserted. (8ed8c3a25)

**`CONFIG_RADIO_ASSERT_ON_COM_PROBLEM` had no effect in `radiolink.c`.**
`radiolinkSyslinkDispatch()` tested `CONFIG_RADIO_ASSERT_ON_QUEUE_FULL`, which no Kconfig option defines,
so a full CRTP receive queue always dropped the radio packet silently. It now tests
`CONFIG_RADIO_ASSERT_ON_COM_PROBLEM`, as `uart_syslink.c` does: with the default (`y`) a full queue
asserts, and with `n` the packet is dropped. (7356491e1)

## Features

**The GAP8 bootloader's MD5 is printed at the end of a GAP8 flash.**
After the last chunk the console shows `GAP8 md5: <32 hex digits> len=<reply length>`, the MD5 the GAP8
bootloader computed over the written region, so a host can compare it with the MD5 of the image it sent.
The digits are uppercase (the firmware printf prints `%x` that way); `len=17` is a complete reply, and a
shorter one leaves the digest as zeros. Always on. (a249ef73e, 8ed8c3a25)

**Kconfig options for the CRTP transmit queue and the log limits.**
`CONFIG_CRTP_TX_QUEUE_SIZE` (menu "Communication", default 200) sets how many outgoing CRTP packets can
wait for the link; the queue now lives in static RAM instead of the FreeRTOS heap, so its size shows in
the build's RAM report. `CONFIG_LOG_MAX_BLOCKS` (default 16) and `CONFIG_LOG_MAX_OPS` (default 128), in the
new "Log subsystem" menu and capped at 255, size the log subsystem; both are reported in the
`CMD_GET_INFO_V2` reply. (f9d4e7011)
