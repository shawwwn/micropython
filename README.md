MicroPython Edge
=======================
This fork is created for the reason that [The MicroPython Project](https://github.com/micropython/micropython) has always been reluctant to introduce new features due to corcerns about consistency across platforms.

For those who do not share the same concern and want to **SQUEEZE OUT EVERY LAST BIT OF JUICE** from your microcontroller, you are welcome to try out this fork!

This experimental version of MicroPython aims at bringing the newest and most wholesome vendor-speicific features to your device while mantaining basic(common module level) compatibility with the official MicroPython.


Building
---------------------
Please refer to the standard [MicroPython document](README_mp.md)


Features
-----------------
##### ESP32: More Options in CPU Frequency Setting
```py
import machine
machine.freq(1) # set CPU frequency to 1MHz
```
* Available frequencies: 1MHz, 2MHz, 4MHz , 8MHz, 10MHz, 20MHz, 40MHz, 80MHz, 160MHz, 240MHz.
* WIFI relies on 40MHz XTAL clock. When setting frequency<=40MHz, XTAL clock will supply CPU frequency, wifi will therefore be unuseable.

##### ESP32: Power Management(WIP)
```py
import esp32
esp32.wifi_power_save(esp32.WIFI_PS_MAX_MODEM) # set wifi power save mode to maximum
```

##### ESP32: Support for GPIO Pin Hold/Drive/Lightsleep Wakeup
```py
import machine
p=machine.Pin(10)
p.init(drive=p.DRIVE_CAP_STRONGEST, hold=True, ls_wake=p.WAKE_HIGH)
```
* Setting Pin hold will also enable RTCIO Pin hold(if current pin is initialized as a RTCIO pin).
* Pin drive sets the output current limit of current pin. (Strongest ≈ 80mA) [ref1](https://twitter.com/eMbeddedHome/status/868787870743629825)
* For additional details, please refer to [vendor docs](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/peripherals/gpio.html#_CPPv212gpio_hold_en10gpio_num_t).
* Setting ```ls_wake``` only enables current pin to wake up in lightsleep. To enable global gpio wakeup, you need to sequently call ```esp32.lightsleep_wake_on_gpio()```

##### ESP8266/ESP32: Support for ESP-Now
To get a overview of the ESP-Now protocol, checkout the [official user guide](https://www.espressif.com/sites/default/files/documentation/esp-now_user_guide_en.pdf) first.

Miminal Example:
```py
import network
w = network.WLAN(network.AP_IF) # doesn't matter if AP or STA
w.active(True)
w.config(channel=1) # do make sure all devices are on the same channel
print('self mac addr:', w.config('mac'))

def tx(result):
  mac, sent = result
  print("send: %s - %r" % (mac,sent))
def rx(result):
  mac, msg = result
  print("recv: %s - %s" % (mac,msg))

PEER_MAC1 = b'0\xad\xa3?>\x98'
PEER_MAC2 = b'b\x02\x95)\xda\x8c'

from esp import espnow
espnow.init()
espnow.on_send(tx)
espnow.on_recv(rx)
espnow.pmk('0123456789abcdef')
espnow.add_peer(PEER_MAC1, "1111222233334444")
espnow.add_peer(PEER_MAC2)
espnow.send(PEER_MAC1, 'hello encrypted peer')
espnow.send(PEER_MAC2, 'hello unencrypted peer')
espnow.send(None, "hello all peers")
```

Since ESP-Now library is distributed in binary by Espressif, there isn't much a developer can do to fix issues/bugs within the library. Although I tried implementing serveral workarounds for the existing library bugs, there are still a lot of them.

Some bugs you might have encounter when using ESP-Now:
* Upon changing the primary key(pmk), all local peers' local keys(lmk) have to be re-applied. Otherwise the system will continue to use the old encryption setup by the previous pmk.
* Device who add peer with lmk can still receive unencrypted message from the added peer
* There is no get lmk from peer because of bugs in the SDK.


##### ESP32: Support For More Sleep Options
Light Sleep: [ref](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/sleep_modes.html#entering-light-sleep)
```py
import machine
machine.sleep(ms)
```

New Wakeup Source: [ref](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/sleep_modes.html#ulp-coprocessor-wakeup)
```py
import esp32
esp32.sleep_pd_config(esp32.PD_DOMAIN_RTC_PERIPH, esp32.ESP_PD_OPTION_AUTO)
esp32.wake_on_ulp(True) # enable ULP coprocessor to wake up the device

esp32.lightsleep_wake_on_gpio(True) # wake on gpio, pin(s) are set using machine.Pin.init(ls_wake=True)
esp32.lightsleep_wake_on_uart(True) # wake up when input data is available on uart
```

* ULP wakeup source can not be used when RTC_PERIPH power domain is forced to be powered on (ESP_PD_OPTION_ON) or when ext0 wakeup source is used. [ref](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/sleep_modes.html#_CPPv227esp_sleep_enable_ulp_wakeupv)
* ULP coprocessor can not be disabled for wake_on_ulp() to work.

##### ESP32: Support For RTCIO

RTCIO, short for 'RTC GPIO', is a set of GPIO pins that are routed through the RTC subsystem. An IO MUX is imposed to determine whether these pins should be used as regular GPIO(controlled by the CPU) or RTCIO(controlled by the ULP coprocessor). Once initialized for RTCIO, a pin's state can be carried into deepsleep mode and subsequently be controlled by the ULP coprocessor. Espressif's SDK has included a set of API for CPU-side RTCIO manipulation. This fork aims at exposing these APIs to users.

Added Module:
```py
esp32.RTCPin
```

GPIO and RTCIO use difference pin numbering scheme. Although this module use GPIO numbering for RTCIO, you might want to still check [ESP32 Pin List](http://wiki.ai-thinker.com/_media/esp32/docs/esp32_chip_pin_list_en.pdf).

We will use an example to better describe the functionaliy of this module:
```py
from machine import Pin # GPIO module
from esp32 import RTCPin # RTCIO module

pin = Pin(12) # GPIO 12
rtc_pin = RTCPin(pin) # RTC
print(rtc_pin.pin_num()) # will print (12, 15) as GPIO12 = RTCIO15
RTCPin.force_hold_dis_all() # disable hold for all RTCIO pins
rtc_pin.active(True) # init pin for RTCIO, note that you can not change this pin using the GPIO module from this call
rtc_pin.direction(rtc_pin.OUT) # pin for OUTPUT
rtc_pin.pullup(True) # configure pullup resistor for the pin
rtc_pin.value(1)
print(rtc_pin.value()) # will print '1'
rtc_pin.wake_by(rtc_pin.GPIO_INTR_LOW_LEVEL) # enable wake up on low voltage level(value=0) in lightsleep mode
rtc_pin.hold(True) # holds rtc_pin value/direction, even in deepsleep mode

import machine
machine.deepsleep(1000) # deepsleep for 1 second

###
### ... wake up from deepsleep
###

from machine import Pin # GPIO module
from esp32 import RTCPin # RTCIO module
pin = Pin(12) # GPIO 12
rtc_pin = RTCPin(pin) # RTC
print(rtc_pin.value()) # will print 1
rtc_pin.hold(False) # in order to change rtc_pin's value & direction, need to disable hold first
rtc_pin.value(0) # now you can set pin to low
```
