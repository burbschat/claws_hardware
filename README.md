# Simple SiPM/plastic scintillator based beam loss monitors
## Introduction
This repository contains the hardware design for a kind of SiPM/plastic
scintillator based beam loss monitor.
The design is based on and borrows the name (*CLAWS*) from an earlier design
employed during commissioning of the SuperKEKB collider.
A related thesis can be found [here](https://mediatum.ub.tum.de/1624845).
CLAWS was supposed to be a abbreviation for some longer name but it surely
wasn't a very elegant one, neither can I seem to remember it. Anyhow, as a
short name is quite practical, we kept on calling the system *CLAWS*.

The sensors later have been re-purposed to detect beam loss and trigger a beam
abort if there is too much of it. This application does not require very
precise sensors. Further, opposed to the original design, now a larger dynamic
range was desirable. To also allow for measurement of beam backgrounds
which occur for longer than a few microseconds, DC coupled analog electronics
are required.
The design published here was created to implement the required changes
outlined above and create a system better suited to the current application at
the SuperKEKB collider.

## Disclaimer
The circuits here have been designed by an at the time undergrad student (me)
who had no real experience in electronics or electronics CAD. I am sure there
are many aspects that I would do differently now, be it related to the circuit
itself or how it is described in KiCad.
I am however of course open for suggestions to improve the design (although I
cannot guarantee I will find the time to implement them myself).

## Overview
There are two parts to the detector system: The detectors, which are to be
placed where the beam background is expected to occur and the receiver board,
which should be placed near the digitizer, which is usually positioned outside
the radiation controlled area of the accelerator complex.

The following is an illustration of the use case at SuperKEKB (use for beam
abort), also illustrating the detector structure.
We employ a special trigger to reject short background spikes. The analog
signal however can be used in any other conceivable way depending on the exact
application.

![illustration of use at SuperKEKB](./figures/system_overview_en.svg)


### Detector Board
The detector (sometimes also referred to as the sensor) is based on a plastic
scintillator (BC-408 or similar material), light from which is detected using a
Hamamatsu SiPM (or MPPC). The SiPM is located in a small dimple in the bottom
of the scintillator. The scintillator is wrapped in an appropriately sized
reflective film (M3 DF2000MA or similar) to make sure as much as possible light
reaches the SiPM. This is the same as what was used with the earlier versions
of CLAWS and can be traced back to what is described in [this paper](https://arxiv.org/abs/1512.05900).
The wrapped scintillator is fixed to the PCB using black aluminium tape, which
also serves to shut out any light from the environment, which would overwhelm
(and potentially damage) the SiPM.

The signal obtained from the SiPM is turned into a differential signal using
the LMH6553 fully differential amplifier. The gain of this amplifier may be set
by choosing appropriate resistor values in the feedback loops of the circuit.
This amplifier then drives a differential pair in an ordinary Ethernet cable.
However, generally only CAT6A or CAT7 cables are used, as they usually provide
superior shielding, which even though a differential pair is used, appeared to
significantly lower the noise introduced by the cable.

The required voltages to drive the amplifier circuit as well as the SiPM bias
voltage are obtained from the same Ethernet cable. The amplifier here is
configured with a single 10V power supply. The voltage in the cable should be
slightly above 10V, as it is regulated by a LDO regulator on the detector
board.
10V is the maximum supply voltage supported by the LMH6553 and is chosen to
maximize the dynamic range at the amplifier output (which depends on the supply
voltage).

One twisted pair in the cable remains unused for now.

![detector board V4.1](./figures/sensor.png)


### Receiver Board
The differential signal provided by the detector board is received and properly
terminated by the receiver board. It contains a circuit using the same LMH6553
fully differential amplifier, which provides further amplification of the
received signal, as well as the option to convert it to a single ended signal.
Both inverted and non-inverted signals are supported. The not used side must be
terminated using a 50Ohm resistor (which can however also be mounted directly
on the board while leaving the corresponding output connector unpopulated).
There also is the option to further use both outputs at the same time to obtain
a differential signal.

The receiver amplifier circuit is, opposed to the one on the detector board,
arranged in a symmetric configuration. This is done to avoid a large DC bias at
the output, which poses no problem for in case of the detector board, as it is
removed by the receiver amplification circuit. The symmetric configuration
necessitates +5V and -5V supplies (for both of which there are LDO regulators
on the board).

Presumably due some unavoidable variations in the used components (resistance
values etc.), the circuit is never perfectly symmetric and thus a small DC bias
remains. The same bias appears at both outputs (inverted and non-inverted).
To compensate this, the common mode output voltage control pin of the LMH6553
is used, which is accessible through a potentiometer on the board.
As the bias is present at both outputs, one can only ever tune the circuit to
have one of them centered perfectly around zero. This however should pose no
significant problem, as in most use cases one probably only uses one of the two
outputs, or uses both to get a differential signal, for which a small DC bias
does not matter.

![receiver board](./figures/receiver.png)

The Receiver board further also distributes the voltages required by the
detector boards. These voltages as well as those needed by the receiver board
itself are supplied externally through connectors on the side of the board.
There further is the option to daisy-chain multiple receiver boards and share
the supply voltages between them. Which voltage should be connected to what is
to some degree configurable using a set of jumper pins. As regulators are used
for all but the SiPM bias voltage, the low voltages provided can be chosen
rather freely, as long as they fall within the ranges supported input of the
LDO regulators. However, testing showed that if they are on the high side, the
LDOs get rather hot. Whether this is still within the supported temperature
range was not (yet) verified.

There is also an option to use the positive voltage (before the LDO) for the
receiver board amplifier as the positive voltage for the detector board
amplifier. This turned out to be the most convenient configuration as it allows 
the setup to be powered off a +/-12V double rail supply (next to a supply for
the SiPM bias voltage).

To share voltages between receiver boards, a 10-pin ribbon cable is used. The
boards may be arranged in a stacked configuration, in case of which the
connector on the bottom must be installed and used, or a flat configuration
with a single long ribbon cable with multiple connectors.

A example for the stacked configuration is shown below. There is theoretically
also the option to add further boards of the same form factor which for example
may contain a high voltage power supply to provide the SiPM bias voltage, or
DC/DC converters to provide the other supply voltages. A mockup is shown in the
figure below.

![receiver boards tower configuration](./figures/tower_with_ps_en.svg)

### SiPM Bias Voltage Supply
To avoid having to prepare a adjustable power supply supporting up to ~60V for
every setup, it was attempted to design a simple circuit based on the LT3571
75V DC/DC Converter for APD Bias.
The design appears to function alright, but does not provide a very flat voltage.
This to some extend is reflected in the signal form the SiPM. However, for the
application at SuperKEKB, the signal does not have to be very precise and the
introduced noise can be tolerated.

The board also does indeed emit some EMI, so care must be taken on where to
position it. There likely are possibilities to improve this design. I am open
for suggestions.

The board can be run off the same +12V as used as an input to the receiver
board. There is the option to use a DAC in combination with a microcontroller
to set the voltages and possibly read them back. I did not yet get around to
prepare a proper firmware for that. One can also simply adjust the voltage 
using a potentiometer.

There are four channels on one board, just because four of them appeared to
fit. One can just not populate the unused ones, if less than four are required.
The board does not support the same interconnect headers as the receiver boards.
This is because it mainly was designed to see if the circuit does work/is
usable at all. One could create a new version with only one channel featuring
the interconnect header.

## KiCad
The design is entirely drawn in KiCad. I like to use the newest version. The
files in this repository are for version 9.

## Libraries
There is are custom symbol and footprint library containing all the parts not
present in the standard KiCad libraries. The provided 3D models are managed
with git LFS.

## Simulation
Some of the symbols reference SPICE models in the `${KIPRJMOD}/../lib/spice_models/` directory.
Those are not present in this repository as I am probably not permitted to distribute them.
If simulation is required, please acquire the models directly from the manufacturers.

## KiBot
KiBot can be used to automate creation of gerber files, export circuit diagrams
and assembly documents or similar. The included configs are those which I used
when producing the boards at use for the SuperKEKB abort system. Adjust to taste.
