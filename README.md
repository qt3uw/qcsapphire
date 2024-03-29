# Quantum Composer Sapphire 9200 Pulser Control

Helper code to communicate with [Quantum Composer's
Sapphire 9200 TTL pulse generator](https://www.quantumcomposers.com/_files/ugd/fe3f06_357ff95b25534660b8390c0305582a3f.pdf).

This code facilitates connections to the device, communication, error handling and system status.

## Installation

```
pip install qcsapphire
```

## Usage

### Determine the port


```python
import qcsapphire
qcsapphire.discover_devices()
```

Will return a list of ports and information about devices connected to those ports.
For example, on *nix platforms, you may see

```python
[['/dev/cu.BLTH', 'n/a', 'n/a'],
 ['/dev/cu.Bluetooth-Incoming-Port', 'n/a', 'n/a'],
 ['/dev/cu.usbmodem141101',
  'QC-Pulse Generator',
  'USB VID:PID=04D8:000A LOCATION=20-1.1']]
```

The device here is connected to `\dev\cu.usbmodem141101`.

On Windows you may see

```python
[['COM3',
  'Intel(R) Active Management Technology - SOL (COM3)',
  'PCI\\VEN_8086&DEV_43E3&SUBSYS_0A541028&REV_11\\3&11583659&1&B3'],
 ['COM5',
  'USB Serial Device (COM5)',
  'USB VID:PID=0483:A3E5 SER=206A36705430 LOCATION=1-2:x.0'],
 ['COM6',
  'USB Serial Device (COM6)',
  'USB VID:PID=04D8:000A SER= LOCATION=1-8:x.0'],
 ['COM7',
  'USB Serial Device (COM7)',
  'USB VID:PID=239A:8014 SER=3B0D07C25831555020312E341A3214FF LOCATION=1-6:x.0']]
  ```

It is certainly not obvious to which USB port the QC Sapphire is connected. However,
using the Windows Task Manager, as well as trial and error, should eventually
reveal the correct serial port to use. In this case, `COM6`.

### Connection to Pulser

```python
pulser = qcsapphire.Pulser('COM6')
```

### Communication

For normal usage, all commands sent to the device should use the `query()` method or
with property-like calls (see section below).

The `query()` method will write a command, read the response from the device,
check for errors (raising an Exception when an error is found) and return the string
response. For example,

```python
ret_val = pulser.query(':PULSE0:STATE?')
print(ret_val)
'0'
```

```python
ret_val = pulser.query(':PULSE1:WIDTH 0.000025')
print(ret_val)
'ok'

ret_val = pulser.query(':PULSE1:WIDTH?')
print(ret_val)
'0.000025000'
```

#### Property-Like Calls

It's possible to make the same calls to the pulser using a property-like call.
Instead of calling `pulser.query("command1:subcommand:subsubcommand value")`,
one can call `pulser.command1.subcommand.subsubcommand(value)`, which is more readable.

For example,

```python
ret_val = pulser.pulse1.width(0.000025) #sets the width of channel A
print(ret_val) # 'ok'

width = pulser.pulse1.width() #asks for the width of channel A
print(width) # '0.000025000'
```

All of the SCPI commands can be built this way.

In either case, the user is responsible
for sending the correct command strings by following
[the API](https://www.quantumcomposers.com/_files/ugd/fe3f06_357ff95b25534660b8390c0305582a3f.pdf).
It should be pointed out there is no need to worry about string encoding and carriage returns / line feeds,
as that is taken care of by the code.

#### Programming Channels

Instead of using 'pulseN' to access a particular QCSapphire channel, methods have
been added to facilitate more programmatic ways of control.

```

```


### Global and Channel States

Two methods exist to report on global and channel settings

##### Global Settings

```python
import pprint
pp = pprint.PrettyPrinter(indent=4)

pp.pprint(pulser.report_global_settings())
```

##### Channel Settings

```python
for channel in range(1,5):
    pp.pprint(f'channel {channel}')
    pp.pprint(pulser.report_channel_settings(channel))
```

### Examples

The following examples demonstrate both simple and advanced usage.

#### Resets

The QCSapphire can be unstable at times. We have found that forcing the
system to reset, multiple times, is necessary to program the pulser the
the ways described below. You may or may-not have better stability
if you reset the pulser before programming.


```python
for i in range(2):
      pulser.set_all_state_off()
      time.sleep(1)
pulser.query('*RST')
pulser.system.mode('normal')
```

#### CWODMR

In CWODMR our spin system is continuously illuminated with an optical pump,
while an oscillating (RF) magnetic field drives magnetic transitions between
spin states. While the optical pump is continuously on, we can use the QCSapphire
to control the RF magnetic signal timing. The RF signal hardware will control
the frequency of the field (for NV centers, between 2.6 and 3.2 GHz, depending on
the level of Zeeman splitting). The QCSapphire's TTL output may operate a
switch that blocks the RF signal at known times. This allows the experimenter
to acquire photo luminescence (PL) signal for periods when no RF signal is applied
in order to measure a background, and to then measure PL when the RF is applied.

In this case, we wish to program the QCSapphire to output a TTL signal of fixed
duration and period. The following example shows how to generate a 5 microsecond on/off TTL signal.

##### Simple Example: Single Channel

In the simple case, we just have a single channel ('B') that we wish to use to
generate a square wave. The documentation is straight-forward here. NB: channel
'B' is demonstrated here because we use channel 'A' to control the optical
pumping in other experiments (pulsed-ODMR, Rabi, etc.).

```python
total_width = 10e-6 #in seconds
pulser.system.period(total_width)

#use channel B for RF switch and use 'normal' mode
channel = pulser.channel('B')
channel.mode('normal')

#create a 5 microsecond pulse
#round to nearest 10ns bc QCSapphire does not behave well with machine errors
channel.width(np.round(total_width/2, 8))
channel.delay(0)

pulser.system.state(1) #start the pulser
```

##### Realistic Example: Adding Clock and Trigger Channels

In order to take robust data, however, we need to control our data acquisition
system (DAQ). Supplying an external clock and trigger signal from the QCSapphire ensures
that the DAQ and pulser do not drift from each other and data is acquired at the
exact moment the system is ready.

However, this changes how we must program the RF on/off to be 5 microseconds in
width. The following steps through the necessary calls to produce a

  * 200 ns clock period
  * 5 microsecond on/off RF pulse
  * single trigger pulse at the start of every 10 microsecond period

###### First the Clock

In this example, channel ('C') is programmed to act as the clock. The
pulser system clock period is set to the smallest allowed value in order to
set our clock tick leading edge period to 200 ns.
This channel is programmed exactly as the simple case above with a smaller width.

```python
clock_period = 200e-9
pulser.system.period(clock_period)
channel = pulser.channel('C')
channel.mode('normal')
channel.width(np.round(clock_period/2, 8)) #100 ns wide square wave
channel.delay(0)
```

###### Add RF switch

A 5 microsecond wide pulse (positive), followed by
5 microseconds off on channel B is implemented using the duty-cycle mode.
In the duty-cycle mode, we specify the number of `clock_periods` ON we wish for the
channel to generate a signal we describe. We then specify how many `clock_periods`
OFF until the pattern repeats. In this case channel B is programmed to output
a 5 microsecond wide pulse ONE time and then wait the appropriate number of
`clock_periods` such that the start of the next 5 microsecond wide pulse occurs
10 microseconds later. Thus, the OFF duty cycle will be `10 mu*s / clock_period  - 1`


```python
rf_pulse_width = 5e-6
full_pulse_sequence_period = 2 * rf_pulse_width
n_on_cycles = 1
n_off_cycles = np.round(full_cycle_width/clock_period).astype(int) - n_on_cycles
channel = pulser.channel('B')
channel.mode('dcycle')
channel.width(np.round(rf_pulse_width, 8)) #5 mu*s wide square wave
channel.delay(0)
channel.pcounter(n_on_cycles)
channel.ocounter(n_off_cycles)
```

When data are acquired, the first 5 microseconds will be the 'signal' where
we expect a dip in PL, and the second 5 microseconds will be the 'background'
where no RF signal is present and we expect full PL intensity.

###### Add Trigger

We now want to produce a single square wave output on channel D at the beginning of
each 10 microsecond period. This will be used to trigger our DAQ. Again, we
use the duty cycle mode.

```python
trigger_width = 100e-9
n_on_cycles = 1
n_off_cycles = np.round(full_pulse_sequence_period/clock_period).astype(int) - n_on_cycles
channel = pulser.channel('D')
channel.mode('dcycle')
channel.width(np.round(trigger_width, 8)) #100 ns wide square wave
channel.delay(0)
channel.pcounter(n_on_cycles)
channel.ocounter(n_off_cycles)
```

###### Set the Channel States

We finally set the states of the channels and system to start the sequence.

```python
pulser.channel('B').state(1)
pulser.channel('C').state(1)
pulser.channel('D').state(1)
pulser.system.state(1)
```

#### Rabi Oscillation / pulsed-ODRM Programming

In pulsed-ODMR, one separates the optical pumping from the application of
the RF magnetic field signal. In a Rabi oscillation experiment, the
width of the RF signal is varied as the PL contrast to background is measured.
Both of these pulse sequences are similar to each other and both are more complicated than the example
above.

The main difference between the CWODMR pulses above and Rabi/pulse-ODMR sequences is
that an appropriate delay to RF pulse is added and channel A is used to
control the optical pump/readout. The RF pulse must come after the initial optical pump.

The full sequence is an optical pump signal (~5 microseconds), followed by an RF
signal of some duration, followed by an optical readout (~5 microseconds),
followed by some period with no RF or optical pump. A description of this sequence, though done
with a lock-in amplifier, is found [here](https://aapt.scitation.org/doi/full/10.1119/10.0001905).

However, the duty-cycle mode is used. The RF channel (B) is programmed to generate
a delayed signal, of some width, ONE time, and then wait M clock cycles before
the next signal, where M = `full_pulse_sequence_period / clock_period - 1`.

An example of this sequence is found in the [QT3 lab experimental code](https://github.com/gadamc/qt3-utils/blob/b03050d86c5116a21986278734817be39df2da8a/src/qt3utils/experiments/rabi.py#L164).

Additionally, as can be seen in the code above, some extra delay after the optical
pump was added because of a finite hardware response time of ~700-900 ns.


#### Trigger Pulses on Channel A via Software Trigger

Here's an example of using an external trigger to generate an output pulse from
the QCSapphire. This may be very useful when used in combination with a
device that has more output channels and more sophisticated pulse sequeunce
programming, such as the Spin Core Pulse Blaster. One of the Pulse Blaster's
output channels could be used to trigger the external trigger channel of the QCSapphire.

One reason to do this would be to utilize QCSapphire's superior pulse width
resolution. The smallest square wave from the Pulse Blaster is 50 ns, while
the QCSapphire can produce a 10 ns wide pulse (according to its documentation)

```python

### TODO -- check this code! It doesn't look right.
pulser.channel('A').mode('single')
#pulser.channel('A').cmode('single') #do we need cmode instead of mode?

pulser.channel('A').period(0.2) #200 ms system pulse
pulser.channel('A').external.mode('trigger')
pulser.channel('A').width(10e-9) #10 ns wide pulse

#pulser.channel('B').cmode('single')
#pulser.channel('B').polarity('normal')
#pulser.channel('B').width(10e-9) #10 ns wide pulse
#pulser.channel('B').state(0)
pulser.channel('A').state(0)


#trigger loop example

#wait N seconds between triggers
wait_N = 5.0
N_pulses = 50

#activate pulser and output channel
pulser.channel('A').state(1)
#pulser.channel('B').state(1)

for i in range(N_pulses):
    pulser.software_trigger() #trigger the pulser
    time.sleep(wait_N)

#deactivate
pulser.channel('A').state(0)
#pulser.channel('B').state(0)
```


### Debugging

If you hit an error, especially when trying to use the property-like calls,
the last string written to the Serial port is found in the
`.last_write_command` attribute of the pulser object.

```python
pulser.channle('B').width(25e-6)
print(pulser.last_write_command)
# ':PULSE1:WIDTH 2.5e-05'
```

Additionally, you can see the recent command history of the object (last 1000 commands)

```python
for command in pulser.command_history():
  print(command)
```

# LICENSE

[LICENCE](LICENSE)

##### Acknowledgments

The `Property` class of this code was taken from `easy-scpi`: https://github.com/bicarlsen/easy-scpi
and modified.
