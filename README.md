# PTGroup Stepper Motor Controller

A controller for the PTGroup stepper motor microwave generator.

## Modifying the project
There is a git repository in this folder so that you can branch/do whatever with it.
Hosted under `ptgroup2@bitbucket.org/ptgroup2/stepper-motor-controller` (private repo)

## Controller usage
The following describes the usage of the various "panes" on the front panel of the controller.

### Communication setup
Here, you must select the COM port of the motor before starting the controller.
Once you select a port, you can press the "run" button of the VI to run the controller.
The `STOP COMMUNICATION` button is used to cleanly shut down the controller;
you should always use this rather than the VI stop button if possible.
After pressing `STOP COMMUNICATION`, please be patient if it doesn't shut down immediately;
since there are multiple processes running in the background, some of which have built-in delays,
it may take a few seconds to exit them cleanly.

### Manual motor control
These controls allow you to move the motor manually when automatic mode is not running.
To make a motion, first set the step size and the velocity, and then press `Move up` or `Move down` to move the motor accordingly.
These motions are controlled in units of complete revolutions; if the frequency is calibrated, you may enter a frequency and use the `Goto` button to go directly to that frequency.
The `STOP MOTOR` button may be used to stop the motor immediately, for example if you started a motion that you wish to cancel.

### Motor information
These indicators display various pieces of information provided by the motor.

The `Motor time` indicator simply measures how much time has elapsed since the controller started running; since this is a value stored within the motor itself, you may use it as a simple communications test (if the number is counting up, then motor communication has been successfully established).

The `Motor position` indicator displays the position of the motor, in revolutions. This position is reset to 0 whenever the motor is physically turned off (this cannot be avoided, unfortunately). The `Frequency` indicator displays the frequency corresponding to the current motor position, once the frequency has been calibrated.

### Motor alarm
If the motor encounters a problem (such as trying to move past one of its boundary positions), it will halt all operations and set an "alarm", which is displayed in this section.
When such an error occurs, the indicator will turn red and the alarm code will be displayed for diagnostic purposes.
The causes of each alarm code are listed in the "Troubleshooting" section of the motor manual.
There is intentionally no way to reset the alarm code without restarting the motor and the VI (although this is possible from a technical standpoint); the best thing to do is to restart the system completely.

### Frequency calibration
The correspondence between motor position and frequency is linear, as has been shown in tests with other controllers.
Since each EIO is different, and since the motor position is reset to 0 on each startup, it is necessary to calculate these fit parameters each time the motor is restarted.

#### Frequency calibration instructions
To calibrate the frequency, follow the following steps:
1. Move the motor to some position (ideally, this would be at one of the extremes of your experimental frequency range, probably 140 GHz)
2. Click the `Read position` button in the first row (corresponding to position 1)
3. Without moving the motor, look at the EIP and enter the frequency displayed, in GHz, as "Frequency 1"
4. Repeat steps 1-3 for Frequency 2 (this should be at the other extreme of your frequency range, e.g. 140.4 GHz for positive polarization of NH_3)
5. Click `Calculate fit parameters` (the VI will warn you if it thinks you made a mistake)

Once all these steps have been completed, the frequency displayed under `Motor information` should be the same as (or at least very close to) the actual frequency displayed on the EIP.

### Automatic control
With a source of data, the VI can control the motor automatically so as to optimize polarization.
The `Data input file` should be chosen to point to the source of this experimental data; the data is assumed to be a CSV file with the first column representing time in seconds and the second column representing polarization.

#### Simulation
To test automatic mode without running a real experiment, click `Launch simulation` to bring up a simulation which will approximately reflect actual experimental behavior.

##### Main simulation controls
To run the simulation, you must select the `Data output file`, where the simulation output will be written.
The indicators display the current time, polarization, etc. of the simulated experiment.

##### Manual controls
The simulation may be run without using the motor controller to provide a frequency input.
In this case, you can set the frequency manually using the `Frequency control` and change the `Delay` between each step.
For example, setting the delay to `500 ms` will output two seconds of simulated data in one second of real time, allowing for faster output.

##### Advanced data and controls
These controls should rarely (if ever) be used, and are mostly for debugging.
You may see the internal parameters calculated for the current frequency, which may be helpful for comparing with fits of real data (the parameters correspond to the expression `P = A + C*exp(-k*t)`).

#### Grapher
When automatic control is enabled, a subVI will appear which contains graphs of the polarization, frequency, etc. versus time.
The grapher will also output this data, in CSV format, to the file specified as output for further analysis (if no file is chosen, the data will be lost when the VI is stopped).

### Advanced configuration
This section contains advanced controls and parameters that you should probably not need to use.
The `Automatic step size` controls the initial step size of the automatic seek algorithm; change it if the initial value is too small/large for a particular situation.
The `Reseek` button will reset the automatic mode step size to the value specified in the `Automatic step size` control; use this if the algorithm gets "lost" and you would like it to try to find a new ideal frequency (for example, you might use this if you've been running the beam for a long time and the automatic mode isn't reacting fast enough to the changes).

## Technical notes
The following are various general notes that may be useful when inspecting the LabVIEW code or output.

### Data file format
All data files consumed or produced by any part of this project will follow the following column format (in CSV format):
1. Eventnum (a Unix timestamp; see below)
2. Polarization
3. Frequency (optional)
4. Rate (optional)
Optional columns may not be present in some output files depending on context.
However, if, for example, column 4 is present, all preceding columns must be present (possibly with no data or N/A specified in the column).
All columns except the first two will typically be different for experimental output files; this is not a problem, as the VI will only process the first two values.

### Time/eventnum format
All timestamps in input/output files are Unix timestamps; they count the number of seconds since January 1, 1970 at midnight UTC.
In VIs where a timestamp is given to a LabVIEW control, it must be converted to the epoch used by LabVIEW, which starts at January 1, 1904 at midnight UTC.
Helper VIs to perform these conversions are located under `Useful subvis`.

### Loop control and exiting
Since many loops are used to perform different tasks throughout the project, there must be a way to ensure all the loops are stopped when the used presses `STOP COMMUNICATION` on the main VI.
This is achieved through the use of `queue` structures; a queue is created which holds boolean values, and which is periodically checked by each loop for a `true` signal waiting in the queue.
When a `true` signal is detected in the queue by any loop, it will stop execution.
Thus, to stop all the loops, all that is needed is to insert `true` into the queue.
A queue is used rather than a notifier because a notifier will block until a specified timeout trying to receive a notification, while a queue can be checked for the presence of a notification without using a timeout.

### Motor communication and commands
Motor communication is done via RS232, at a default baud of `38400`.
There are various commands which can be sent to the motor, and most of these will return some data which can be processed by the VI.
Commands sent to the motor should always be terminated with a carriage return character (`0x0D`, or `\r`), and a carriage return character will always terminate every line of data sent back by the motor.
A subvi to make this process easier (automatically formats input and output) is included under `Useful subvis`.

The commands which can be sent to the motor can be found in the manual, and most are not very useful for our application. Multiple commands can be given at once by separating them by a semicolon, e.g. `dis 0.01; mi` to set the move distance to `0.01 revolutions` and then to move the motor accordingly.
