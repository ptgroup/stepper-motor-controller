# PTGroup Stepper Motor Controller

A controller for the PTGroup stepper motor microwave generator.

## Table of contents

* [How to read this documentation](#how-to-read)
* [Modifying the project](#modifying)
* [Project structure](#structure)
* [Motor controller](#motor-controller)
  * [Communication setup](#communication-setup)
    * [Debug mode](#debug-mode)
  * [Manual motor control](#manual-control)
  * [Motor information](#motor-info)
  * [Motor alarm](#motor-alarm)
  * [Frequency calibration](#calibration)
    * [Frequency calibration instructions](#calibration-steps)
  * [Automatic control](#automatic-mode)
    * [Grapher](#grapher)
  * [Configuration backup/restore](#config-backup)
  * [Power data](#power-data)
  * [Advanced configuration](#advanced-config)
* [Simulation](#sim)
  * [Main simulation controls](#sim-controls)
  * [Advanced data and controls](#advanced-sim-controls)
* [Technical notes](#technical-notes)
  * [Data file format](#csv-format)
  * [Time/eventnum format](#time-format)
  * [Motor communication and commands](#motor-communication)

## How to read this documentation <a name="how-to-read"></a>

This file is written in Markdown, which can be processed with a
Markdown renderer like
[cmark](https://github.com/commonmark/cmark). Once you have `cmark`
installed, run the command `cmark README.md >README.html`.
Alternatively, you can read it as a plain text file, and it should
still be fairly usable. Hopefully, somebody remembered to run this
command the last time the README was updated, so that `README.html` is
already generated; if you're reading `README.html` and something seems
out of date, it's probably because whoever changed `README.md` forgot
to re-render it.

## Modifying the project <a name="modifying"></a>

There is a git repository in this folder so that you can branch/do
whatever with it; it is hosted remotely under
`https://github.com/ptgroup/stepper-motor-controller`. When working
with the project, it is best to open the file `Motor
controller.lvproj` and then do stuff, since some features (such as
classes) are only supported within the LabVIEW project interface.

## Project structure <a name="structure"></a>

The motor controller project is divided into two main components, the
motor controller itself and the simulation, each of which can be run
independently and which are documented in their respective sections
below. In a real experiment, only the motor controller will be used,
while the simulation can be used to test behavior outside an
experiment.

## Motor controller <a name="motor-controller"></a>

The following describes the usage of the various "panes" on the front
panel of the controller.

### Communication setup <a name="communication-setup"></a>

Here, you must select the COM port of the motor before starting the
controller. Once you select a port, you can press the "run" button of
the VI to run the controller. The `STOP COMMUNICATION` button is used
to cleanly shut down the controller; you should always use this rather
than the VI stop button if possible. After pressing `STOP
COMMUNICATION`, please be patient if it doesn't shut down immediately;
since there are multiple processes running in the background, some of
which have built-in delays, it may take a few seconds to exit them
cleanly.

Note that after pressing the `STOP COMMUNICATION` button, the motor
will move back to a known position (motor position 0) before the VI
shuts down. This is because the motor position is only a relative
measure: if the motor is restarted, it will treat its initial position
as 0 regardless of what the position value may have been before the
restart. Since this causes problems with saved configuration files, it
was thought best to make this "reset" the default behavior. To avoid
this, use the `RESET EVERYTHING AND STOP` button under the `Advanced
configuration` pane, which will not move the motor before stopping the
VI.

#### Debug mode <a name="debug-mode"></a>

If you don't have access to a physical motor, you can enable "Debug
mode" using the switch in the `Communication setup` pane. This will
"simulate" the motor (really, it just keeps track of a fake position)
so that the other features can be used.

### Manual motor control <a name="manual-control"></a>

These controls allow you to move the motor manually when automatic
mode is not running. To make a motion, first set the step size and
the velocity, and then press `Move up` or `Move down` to move the
motor accordingly. These motions are controlled in units of complete
revolutions; if the frequency is calibrated, you may enter a frequency
and use the `Goto` button to go directly to that frequency. The `STOP
MOTOR` button may be used to stop the motor immediately, for example
if you started a motion that you wish to cancel.

### Motor information <a name="motor-info"></a>

These indicators display various pieces of information provided by the motor.

The `Motor time` indicator simply measures how much time has elapsed
since the controller started running; since this is a value stored
within the motor itself, you may use it as a simple communications
test (if the number is counting up, then motor communication has been
successfully established).

The `Motor position` indicator displays the position of the motor, in
revolutions. This position is reset to 0 whenever the motor is
physically turned off (this cannot be avoided, unfortunately). The
`Frequency` indicator displays the frequency corresponding to the
current motor position, once the frequency has been calibrated.

### Motor alarm <a name="motor-alarm"></a>

If the motor encounters a problem (such as trying to move past one of
its boundary positions), it will halt all operations and set an
"alarm", which is displayed in this section. When such an error
occurs, the indicator will turn red and the alarm code will be
displayed for diagnostic purposes. The causes of each alarm code are
listed in the "Troubleshooting" section of the motor manual. There is
intentionally no way to reset the alarm code without restarting the
motor and the VI (although this is possible from a technical
standpoint); the best thing to do is to restart the system completely
to avoid damage to equipment.

### Frequency calibration <a name="calibration"></a>

The correspondence between motor position and frequency is linear, as
has been shown in tests with other controllers. Usually, after
calibrating the frequency once, it is sufficient to save the current
configuration in order to use it in future experiments. The exception
to this is if the motor position is not reset to 0 before shutting off
the motor (see the section on [communication
setup](#communication-setup)); in this case, the calibration will be
invalidated and you will need to calibrate it again.

#### Frequency calibration instructions <a name="calibration-steps"></a>

To calibrate the frequency, follow the following steps:

1. Move the motor to some position (ideally, this would be at one of
   the extremes of your experimental frequency range, probably 140 GHz)
2. Click the `Read position` button in the first row (corresponding to
   position 1)
3. Without moving the motor, look at the EIP and enter the frequency
   displayed, in GHz, as "Frequency 1"
4. Repeat steps 1-3 for Frequency 2 (this should be at the other
   extreme of your frequency range, e.g. 140.4 GHz for positive
   polarization of NH_3)
5. Click `Calculate fit parameters` (the VI will warn you if it thinks
   you made a mistake)

Once all these steps have been completed, the frequency displayed
under `Motor information` should be the same as (or at least very
close to) the actual frequency displayed on the EIP. If this is not
the case, you can repeat the above steps, pressing the `Refine fit
parameters` button instead of `Calculate fit parameters` to
(hopefully) get a better fit.

### Automatic control <a name="automatic-mode"></a>

With a source of data, the VI can control the motor automatically so
as to optimize polarization. The `Data input file` should be chosen to
point to the source of this experimental data; the data is assumed to
be a CSV file with the first column representing time in seconds and
the second column representing polarization (see [the CSV format
description](#csv-format) for a more detailed description of the
format used by this VI).

The values displayed above the `Data input file` control (e.g. the
`Samples taken` indicator) are only really useful for debugging the
algorithm and providing the [grapher](#grapher) with data, so you
don't need to worry about them.

The `Launch simulation` button is used to launch the simulation, for
testing the control algorithm outside of a real experiment. It is
important to launch the simulation using this button rather than
launching it separately by opening its VI from the folder; if you do
the latter, the simulation won't be able to keep track of the motor
position, making it pretty useless. You should only launch the
simulation by itself if you want to use it by itself.

The three boxes at the bottom of this pane are used for providing the
seek algorithm with some additional data to help it in its quest for
the ideal frequency. The positive (resp. negative) seek bounds provide
the ranges in which the optimal frequency for positive
(resp. negative) polarization is to be found. The seek algorithm will
never move the motor outside the applicable range, so these can be
useful to prevent it from getting lost. The `Home position` is the
frequency at which the algorithm will start: you should set this as
close to the optimal frequency as possible.

After a certain time (configurable in the [advanced configuration
pane](#advanced-config)), the automatic seek will switch from a rate-based
search to a polarization-based search. The countdown timer shows how much
time is left before this switch occurs. If desired, the countdown can be
reset, returning to a rate-based search and restarting the timer. The button
next to the countdown timer can also be used to disable the timer entirely
and control the choice of algorithm manually.

#### Grapher <a name="grapher"></a>

When automatic control is enabled, a subVI will appear which contains
graphs of the polarization, frequency, etc. versus time. The grapher
will also output this data, in CSV format, to the file specified as
output for further analysis (if no file is chosen, the data will be
lost when the VI is stopped).

### Configuration backup/restore <a name="config-backup"></a>

These controls allow you to save some commonly used settings
(enumerated in the description on the pane) to a configuration file
which you can restore after restarting the VI. In particular, this
means that you can save the [frequency calibration](#calibration) so
that you don't have to recalibrate every time you start the VI.

### Power data <a name="power-data"></a>

The power generated by the microwave can vary between different positions
(frequencies). To record an unusual power level at the current motor
position, enter the power in the `New power` input along with an optional
`Width` and press `Add power reading`. The `Width` measures the width of the
frequency band that is affected by the power discrepancy.

When more and more power readings are added, the `Microwave power` indicator
will more closely reflect the actual microwave power at the current
frequency.

### Advanced configuration <a name="advanced-config"></a>

This section contains advanced controls and parameters that you should
probably not need to use. The `Automatic step size` controls the
initial step size of the automatic seek algorithm; change it if the
initial value is too small/large for a particular situation. The
`Reseek` button will reset the automatic mode step size to the value
specified in the `Automatic step size` control; use this if the
algorithm gets "lost" and you would like it to try to find a new ideal
frequency (for example, you might use this if you've been running the
beam for a long time and the automatic mode isn't reacting fast enough
to the changes).

The `"Good" rate threshold` controls an internal detail of the rate-based
automatic seek algorithm, namely, the minimum ratio of new rate to old rate
that should be considered "acceptable" by the algorithm. If the ratio is less
than this threshold, the algorithm will begin trying frequencies in the
direction opposite to wherever it was searching before (a more detailed
description of the algorithm is outside the scope of this document). In
practice, this means that the higher the threshold, the more "picky" the
algorithm is in finding a good frequency, with the possible negative effect
that it will be easily confused by naturally decreasing rates over time.

The `Algorithm switch time` controls the point at which the automatic seek
algorithm should switch from a rate-based search to a polarization-based
search. The intent of this setting is to make the algorithm more effective
when changes in rate are very small and more focus should be put on improving
the raw polarization value.

The `Samples per step` field controls the number of samples that should be
taken at each frequency encountered in automatic mode. The value must be at
least 3; higher values will take longer to process, but may provide more
precision due to random fluctuations in the data.

When automatic mode is seeking by polarization, encountering a series of
significant drops in polarization can be an indicator that something has
changed in the physical system, and that the current stable position should
be re-evaluated. This re-evaluation could take place manually using the
`Reseek` button, but it will occur automatically after the automatic mode
algorithm notices a series of consistent polarization drops. The number of
drops that triggers the automatic reseek is controlled by the `Reseek after #
bad` input.

The `RESET EVERYTHING AND STOP` button will completely shut down the
VI, without trying to do any of the helpful stuff that the `STOP
COMMUNICATION` button does to make the shut-down a bit more graceful
(like resetting the motor position). It will also reset **all** the
controls on the front panel to their default values, which means you
lose the frequency calibration, seek bounds, home position (and
everything else).

## Simulation <a name="sim"></a>

To test automatic mode without running a real experiment, click
`Launch simulation` to bring up a simulation which will approximately
reflect actual experimental behavior. The simulation can also be
launched on its own (from the LabVIEW project, `Motor
controller.lvproj`) if you just want to see how it behaves without
testing the motor controller as well.

The simulation actually simulates both the physical environment and
the PDP data collection, so it should be a reasonably accurate
reflection of the data you would get during a real experiment.

### Main simulation controls <a name="sim-controls"></a>

You can choose an output file to save the data generated by the
simulation. Notably, the [format of the output file](#csv-format)
generated by the simulation is mostly compatible with the format
generated by the real PDP software (except it contains far fewer
columns), so that the motor controller can accept both formats in the
same way. The other controls on this main pane are designed to be
self-explanatory.

There is a small pane directly to the right of the one just described
which shows some additional information about the PDP simulation. The
`Seconds until next data` counter is exactly what it sounds like (it's
identical in function to the one in the real PDP software), and below
it is a control to configure the number of sweeps performed to take a
single data point. As in the real PDP, more sweeps are slower but lead
to more precise polarization values.

The only confusing control on this pane is the `Time per second (s)`
control: this specifies the number of real-world seconds that are
taken during each simulated second. For example, setting this control
to 0.1 will (at least in theory) make the `Current time (simulated)`
counter increase by ten seconds in the time that it takes your
computer's clock to advance by one second. In practice, this will
depend heavily on the speed of your computer, and it might not be
possible to run the simulation very fast. To allow for faster speeds,
you can decrease the `Precision` value under `Advanced data and
controls`, but be warned that this may lead to other unexpected
behavior, especially if the value of `A` listed under `Fit parameters`
is larger than the `Precision` value.

### Advanced data and controls <a name="advanced-sim-controls"></a>

It shouldn't be necessary to interact with these controls very much,
with the possible exception of the `System parameters` and the `Fit
parameters`.

The names of the `Internal parameters` are self-explanatory, as are
the names of the `System parameters` (the `System temperature` can be
thought of as the temperature of the fridge, as opposed to the
`Temperature` which is the temperature of the sample).

The `Fit parameters` are dependent on the type of material you want to
simulate, and clicking on `Launch fit parameters chooser` will launch
a subVI that describes the meaning of each parameter and allows you to
preview how changing the parameters will impact the distributions of
alpha and beta with respect to frequency (for more information on
alpha and beta, see the paper by Jeffries, which uses the same names
for these parameters).

The other controls on this pane are only for debugging, or because I
didn't want to put too many "magic numbers" directly into the block
diagram. In particular, decreasing the `Precision` value can lead to
some very unexpected behavior when used with certain fit parameters,
and increasing it can slow down the simulation, so it is recommended
to leave it alone unless you're having problems.

## Technical notes <a name="technical-notes"></a>

The following are various general notes that may be useful when
inspecting the LabVIEW code or output.

### Data file format <a name="csv-format"></a>

All data files consumed or produced by any part of this project will
follow the following column format (in CSV format):

1. Eventnum (a Unix timestamp; see below)
2. Polarization
3. Frequency (GHz) (optional)
4. Rate (polarization/s) (optional)
5. Dose (e-/cm^2) (optional)
6. Optimal frequency for positive polarization (GHz) (optional)
7. Optimal frequency for negative polarization (GHz) (optional)
8. Steady state at optimal positive frequency (optional)
8. Steady state at optimal negative frequency (optional)

Optional columns may not be present in some output
files depending on context. However, if, for example, column 4 is
present, all preceding columns must be present (possibly with no data
or N/A specified in the column). All columns except the first two
will typically be different for experimental output files; this is not
a problem, as the VI will only process the first two values.

### Time/eventnum format <a name="time-format"></a>

All timestamps in input/output files are Unix timestamps; they count
the number of seconds since January 1, 1970 at midnight UTC. In VIs
where a timestamp is given to a LabVIEW control, it must be converted
to the epoch used by LabVIEW, which starts at January 1, 1904 at
midnight UTC. Helper VIs to perform these conversions are located
under `Useful subvis`.

### Motor communication and commands <a name="motor-communication"></a>

Motor communication is done via RS232, at a default baud of 38400.
There are various commands which can be sent to the motor, and most of
these will return some data which can be processed by the VI.
Commands sent to the motor should always be terminated with a carriage
return character (`0x0D`, or `\r`), and a carriage return character
will always terminate every line of data sent back by the motor. A
subvi to make this process easier (automatically formats input and
output) is included under `Useful subvis`.

The commands which can be sent to the motor can be found in the
manual, and most are not very useful for our application. Multiple
commands can be given at once by separating them by a semicolon,
e.g. `dis 0.01; mi` to set the move distance to 0.01 revolutions and
then to move the motor accordingly.
