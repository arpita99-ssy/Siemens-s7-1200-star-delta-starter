# Siemens PLC Star-Delta Starter Control System

## Project Overview

This project implements an automatic star-delta starter for a three-phase squirrel-cage induction motor using a Siemens S7-1200 PLC. The motor is connected in star during start-up, which reduces the phase voltage to roughly 58 percent of line voltage and limits the inrush current to about one third of the direct-on-line value. After 5 seconds in star, the program drops out the star contactor and, following a 500 ms changeover interval, closes the delta contactor so the motor runs at full voltage. Software interlocking prevents the star and delta contactors from being energised at the same time.

The project was developed in Siemens TIA Portal V20 on a CPU 1212C DC/DC/DC.

## Design Requirement

The project comment in TIA Portal states the design requirement as follows:

> Design an automatic Star-Delta starter for a three-phase induction motor using a Siemens S7-1200 PLC. The motor shall start in Star mode, automatically transfer to Delta mode after 5 seconds, and stop immediately when the Stop button is pressed. Electrical and software interlocking shall prevent the Star and Delta contactors from being energized simultaneously.

## System Architecture and Hardware

### Controller

| Item | Detail |
|---|---|
| PLC family | Siemens SIMATIC S7-1200 |
| CPU | CPU 1212C DC/DC/DC, 6ES7 212-1AE40-0XB0, firmware V4.6 |
| Engineering software | Siemens TIA Portal V20 |
| Programming language | Ladder logic (LAD) |
| PLC station name | PLC_1 |

The CPU 1212C DC/DC/DC has 24 V DC digital inputs and 24 V DC transistor outputs. The program uses only standard bit logic, SR flip-flops and IEC TON timers, so it can be moved to another S7-1200 CPU with enough I/O. Moving to an S7-1500 requires a device change in TIA Portal followed by a full recompile.

### Program Structure

| Block | Type | Purpose |
|---|---|---|
| Main | OB1 (cyclic), block title "Main Program Sweep (Cycle)" | Complete starter logic in six networks |
| TON_Star (DB1) | Instance DB, IEC TON | Star-phase timer, preset T#5s |
| TON_Transition (DB2) | Instance DB, IEC TON | Changeover timer, preset T#500ms |

### I/O Mapping

Digital inputs:

| Address | Tag | Field device | Description |
|---|---|---|---|
| I0.0 | Start_PB | Pushbutton, normally open | Start command |
| I0.1 | Stop_PB | Pushbutton, normally closed | Stop command |
| I0.2 | Emergency_Stop | Emergency stop, normally closed | Emergency stop circuit |
| I0.3 | Motor_OL | Overload relay, normally closed contact | Motor overload trip |

Digital outputs:

| Address | Tag | Description |
|---|---|---|
| Q0.0 | Main_Contactor | Main (line) contactor coil |
| Q0.1 | Star_Contactor | Star contactor coil |
| Q0.2 | Delta_Contactor | Delta contactor coil |
| Q0.3 | Run_Lamp | Motor running indication |
| Q0.4 | Fault_Lamp | Fault indication |

Memory bits:

| Address | Tag | Description |
|---|---|---|
| M0.0 | Motor_Run | Latched motor run command |
| M0.1 | Star_Mode | Star sequence active |
| M0.2 | Delta_Mode | Delta sequence active |
| M0.4 | Tag_1 | Star timer elapsed (output of TON_Star) |
| M0.5 | Tag_2 | Changeover pending (starts TON_Transition) |

Supply voltage and output ratings depend on the installation: `<24 V DC>`, `<sinking / sourcing>`, `<current rating per output>`.

## Control Logic Features

All logic is in the cyclic block Main (OB1). The SR flip-flops used in the program give priority to the reset input over the set input, so a stop condition always overrides a start command.

| Network | Title | Function |
|---|---|---|
| 1 | Main_Contactor | Start_PB sets the Motor_Run flip-flop (M0.0). Stop_PB, Emergency_Stop or Motor_OL resets it. The flip-flop output drives Main_Contactor (Q0.0). |
| 2 | Star_Contactor | Main_Contactor sets the Star_Mode flip-flop (M0.1). It is reset when the main contactor drops out or when the star timer has elapsed (Tag_1). Star_Contactor (Q0.1) is energised only while Delta_Contactor is off. |
| 3 | Star_Timer | TON_Star runs for 5 seconds while Star_Contactor is on. Its output sets Tag_1 (M0.4). |
| 4 | Delta_Contactor | Tag_1 sets Tag_2 (M0.5), which starts TON_Transition for 500 ms. When it elapses, the Delta_Mode flip-flop (M0.2) is set. Stop_PB or loss of the main contactor resets Delta_Mode. Delta_Contactor (Q0.2) is energised only while Star_Contactor is off. Tag_2 is reset once the delta contactor is closed. |
| 5 | Run_Lamp | Run_Lamp (Q0.3) is on while the main contactor is closed together with the star or the delta contactor. |
| 6 | Fault_Lamp | Fault_Lamp (Q0.4) is on when the emergency stop is active, when the overload has tripped, or when the main contactor is closed while the star and delta contactors are both open or both closed. The lamp therefore also lights briefly during the 500 ms changeover, when both are open. |

### Sequence of Operation

1. Start_PB is pressed. Motor_Run is latched and Main_Contactor closes.
2. The closed main contactor sets Star_Mode and Star_Contactor closes. The motor runs in star and TON_Star starts.
3. After 5 seconds Tag_1 is set. Star_Mode is reset and Star_Contactor opens.
4. Tag_2 is set and TON_Transition runs for 500 ms with both star and delta open.
5. Delta_Mode is set and Delta_Contactor closes. The motor runs in delta at full voltage.
6. Pressing Stop_PB, operating the emergency stop or an overload trip resets Motor_Run. The main contactor opens, Star_Mode and Delta_Mode are reset and all contactors drop out.

Delta mode is reached about 5.5 seconds after the start command, which is the 5 second star period plus the 500 ms changeover interval.

### Interlocking

The software interlock is in the output rungs. The star coil includes a normally closed contact of Delta_Contactor, and the delta coil includes a normally closed contact of Star_Contactor, so neither coil can be energised while the other contactor is closed. The same interlock must also be built into the wiring (see the wiring note below).

### Protection

Stop_PB, Emergency_Stop and Motor_OL all act on the Motor_Run flip-flop. After any of them operates, the starter does not restart by itself. A new start command is required once the condition has cleared.

## Deployment and How to Import

The project is stored as a standard TIA Portal project folder. The file to open is `Star_Delta_Starter_V20.ap20`.

1. Install Siemens TIA Portal V20 or a later version. A project saved in V20 cannot be opened in V19 or earlier.
2. Clone or download this repository and keep the folder structure unchanged. The `.ap20` file depends on the `System` and `AdditionalFiles` folders next to it.
3. Start TIA Portal and select Project > Open.
4. Browse to `Star_Delta_Starter_V20.ap20` and open it. If TIA Portal offers to upgrade the project to a newer version, accept only if you do not need to return to V20.
5. In the project tree, expand PLC_1 > Program blocks and open Main (OB1) to view the logic. The tag table is under PLC_1 > PLC tags.
6. To run on hardware, set the PG/PC interface, assign the CPU IP address, then compile and download. Check the I/O addresses against the wiring first.

If the project is delivered as an archive (`.zap20`), use Project > Retrieve and select the archive instead of opening the `.ap20` file directly.

## Simulation Notes

The program can be tested with S7-PLCSIM for the S7-1200. Compile the project, start the simulation, download the program and use a simulation table to set the inputs.

Stop_PB, Emergency_Stop and Motor_OL are programmed with normally closed contacts for wiring in the healthy state. Set I0.1, I0.2 and I0.3 to TRUE before starting, otherwise the Motor_Run flip-flop is held in reset and the motor will not start. Pulse I0.0 to start. Setting any of I0.1, I0.2 or I0.3 to FALSE stops the motor.

## Wiring and Electrical Schematics Note

This repository contains the PLC program only. Place the electrical drawings, if available, in a `docs/` folder and refer to them here.

Reading the connection requirements:

- The power circuit uses three contactors. The main contactor connects the supply to terminals U1, V1 and W1 of the motor. The star contactor shorts terminals U2, V2 and W2 together. The delta contactor connects U2 to V1, V2 to W1 and W2 to U1.
- The star and delta contactors must never be closed together. Fit a mechanical interlock between them if the contactor type supports one, and wire a normally closed auxiliary contact of each contactor in series with the coil of the other.
- The stop pushbutton, the emergency stop and the overload relay contact must be wired as normally closed devices. The input is high in the healthy state and goes low when the button is pressed or the relay trips, which also means a broken wire is treated as a stop. The start pushbutton is normally open.
- The overload relay is set according to its position. In series with the motor winding it is set to the winding current (line current divided by the square root of three). In the supply line it is set to the line current.
- Check that the motor is rated for the supply voltage when connected in delta, and that all six winding terminals are brought out to the terminal box.
- The transistor outputs of the CPU 1212C DC/DC/DC are 24 V DC and cannot drive AC contactor coils directly. Use interposing relays or DC-coil contactors.

## Repository Layout

```
Star_Delta_Starter_V20/
    Star_Delta_Starter_V20.ap20
    System/
    AdditionalFiles/
    UserFiles/
README.md
LICENSE
.gitignore
```

Folders that TIA Portal regenerates (`IM`, `TMP`, `Logs`, `XRef`, `Vci`) are excluded through `.gitignore`.

## Author

Arpita Priyadarshi Sahoo

The program was developed and tested in Siemens TIA Portal V20 for an S7-1200 CPU 1212C DC/DC/DC.

## License

Released under the MIT License. Copyright (c) 2026 Arpita Priyadarshi Sahoo. See the `LICENSE` file for details.
