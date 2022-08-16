# I2C Timing Registers for the iMXRT 1062

## References
### iMXRT 1062
The Teensy 4 and Teensy 4.1 use the iMXRT 1062 processor.

Information in here is based on the **IMXRT1060RM_rev2.pdf** datasheet;
**i.MX RT1060 Processor Reference Manual, Rev. 2, 12/2019**. References
to this datasheet are given like this `47.4.1.24`.

All results have been confirmed by experiment unless stated otherwise.

Relevant sections:
* Chapter 47 - Low Power Inter-Integrated Circuit (LPI2C)

## Symbols
All symbols are taken from the I2C specification except for:
* t<sub>r0</sub> - time for signal to rise from 0 to 0.3 V<sub>dd</sub>
  - for an RC curve this = 0.4 x t<sub>r</sub> 
* t<sub>f1</sub> - time for signal to fall from V<sub>dd</sub> to 0.7 V<sub>dd</sub>
  - for an RC curve this = 0.4 x t<sub>f</sub> 
* T<sub>LPI2C</sub> - period of the LPI2C functional clock 
* T<sub>SCL</sub> - period of the SCL clock
* scale - scaling factor applied to I2C register values

## Units
All durations are given in nanoseconds (ns).

# I2C Master Registers
* MCCR0 `47.4.1.13`
  - CLKLO - sets low period of the SCL clock pulse (t<sub>low</sub>)
  - CLKHI - sets high period of the SCL clock pulse (t<sub>high</sub>)
  - DATAVD
  - SETHOLD

# Calculations
## scale
**scale = (2 ^ PRESCALE) x T<sub>LPI2C</sub>**

## t<sub>low</sub>
**t<sub>low</sub> = ((CLKLO + 1) x scale) + t<sub>r0</sub>**
* probably affected by t<sub>f</sub> but I haven't tested it yet
* NOT affected by FILTSCL in any appreciable way at 100kHz 
* confirmed that it's not affected by:
  - FILTSCL
  - FILTSDA
  - CLKHI
  - DATAVD
  - SETHOLD
  - IOMUXC_PAD_HYS

## t<sub>high</sub>
I've found it impossible to relate the master config directly to t<sub>high</sub>.
Testing with the built in 100kΩ, 47kΩ and 22kΩ pull up resistors shows:-
* t<sub>high</sub> is affected by:
  * CLKHI - as expected
  * PRESCALE - as expected
  * FILTSCL
  * SCL rise time
  * IOMUXC_PAD_HYS
* WARNING: t<sub>high</sub> is probably affected by t<sub>f</sub> but I haven't tested it yet
* t<sub>high</sub> is not affected by:
  - FILTSDA
  - CLKLO
  - DATAVD
  - SETHOLD
  - BUSIDLE
  - Slave settings

It appears that the CLKHI period begins after SCL reaches 0.3 V<sub>dd</sub>
and before it reaches 0.7 V<sub>dd</sub>. If so, then the exact trigger point
depends heavily on the SCL rise time and anything that affects it.

I've therefore given the minimum and maximum values for t<sub>high</sub> instead
of trying to specify an exact value. 

Tests with the Teensy loopback are fairly
close to ((CLKHI + 1 + SCL_LATENCY) x scale) - (t<sub>r</sub> * 0.5) but I
suspect this isn't useful as it relies on the exact shape of the rise time curve.

### t<sub>high</sub> Minimum Value
**t<sub>high</sub> <= ((CLKHI + 1 + SCL_LATENCY) x scale) - t<sub>r</sub>**

### t<sub>high</sub> Max Value
**t<sub>high</sub> <= ((CLKHI + 1 + SCL_LATENCY) x scale)</sub>**

## t<sub>fall</sub>
According to the spec, t<sub>fall</sub> must have a minimum value of 12 ns.
The Teensy in loopback is much faster than that. It seems to vary between
4 and 8 ns. It's a hard to be sure as the actual value is approximately
the same as my oscilloscope's resolution. These figures are similar to
the ones given in section "7.2 Slew rate" of `AN5078.pdf`.

I searched the web and concluded that:
* fall times below 10ns are common
* the I2C spec figure is probably designed to reduce electrical noise such as undershoot spikes

SPEED, DSE and SRE are intended to control the "strength" of the pin.
I tried a lot of combinations of SPEED and DSE. If DSE is set to 1 then
the fall time is approx 8 ns. Otherwise it's 4 or 5 ns. SPEED seems to
have little effect although values 0 and 1 may be marginally slower.

I changed the default values to DSE(2) and SPEED(0). This still doesn't
meet the spec but it seems to reduce noise spikes. (Compared to the previous
values of DES(4) and SPEED(1)).

This table shows:
* DSE value
* size of falling edges undershoot spike with 22k pullup
* LOW voltage (V<sub>OL</sub>) with 1k external pullup

| DSE | t<sub>fall</sub> | Undershoot | V<sub>OL</sub> |
|-----|------------------|------------|----------------|
| 1   | 8 ns             | -180 mV    | 290 mV         |
| 2   | 5 ns             | -150 mV    | 130 mV         |
| 3   | 5 ns             | -160 mV    | 90 mV          |
| 4   | 5 ns             | -270 mV    | 70 mV          |

Adding a series resistor increases t<sub>fall</sub> but the effect is
very minor. For example 100Ω, increases fall time by a couple of
nanoseconds but increases V<sub>OL1</sub> to 500 mV with a 1k pull up
which exceeds the maximum allowed in the spec.
