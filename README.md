# C6 Flight Computer - Complete Design Documentation

## 1. Project Overview

### 1.1 Introduction

The C6 Flight Computer is a custom-designed avionics system developed for experimental model rocketry applications. This documentation captures the complete design process, technical decisions, challenges encountered, and solutions implemented during the development of this flight computer.

### 1.2 Design Philosophy

The C6 was designed with the following priorities:
- **Redundancy**: Dual sensor configurations for critical measurements
- **Compactness**: Optimized component sizing and layout
- **Reliability**: Robust power management and ESD protection
- **Expandability**: Multiple communication interfaces and breakout options
- **Data integrity**: Onboard storage and telemetry capabilities

### 1.3 Key Specifications

| Parameter | Specification |
|-----------|--------------|
| **Microcontroller** | RP2350B (10x10mm, 80 GPIO) |
| **Operating Voltage** | 3.3V logic level |
| **Input Voltage Range** | Up to 12V (regulated) |
| **Sensor Refresh Rate** | Up to 400kHz (I2C) / 25Hz (GPS) |
| **Data Storage** | MicroSD card (SPI) |
| **Pyro Channels** | 4 independent channels |
| **Communication** | I2C, SPI, UART, USB-C |

---

## 2. Microcontroller Selection

### 2.1 Initial Selection: RP2350A

The RP2350A was initially selected based on:
- **Extensive documentation** and community support
- **Dual-core architecture** (Cortex-M33 or RISC-V)
- **Previous experience** with RP2040
- **Cost-effectiveness** and availability

### 2.2 Migration to RP2350B

During pin assignment planning, a critical limitation was discovered:

**Problem**: Insufficient GPIO pins for the planned sensor configuration when attempting to assign dedicated I2C pairs to each sensor.

**Solution**: Upgraded to RP2350B which features:
- **Larger package**: 10x10mm (vs 7x7mm)
- **More GPIO pins**: 80 pins available
- **Same architecture**: Maintains software compatibility
- **Better routing**: More physical space for traces

### 2.3 Justification and Benefits

The migration to RP2350B provided:
1. Adequate GPIO count for all planned peripherals
2. Improved PCB routing possibilities
3. Better heat dissipation due to larger package
4. Future expandability for additional sensors

### 2.4 Technical Specifications

- **Cores**: Dual Cortex-M33 or dual RISC-V Hazard3 (configurable)
- **Clock Speed**: Up to 150MHz
- **Memory**: 520KB SRAM, external flash support
- **ADC**: 12-bit, multiple channels
- **Communication**: 2x I2C, 2x SPI, 2x UART, USB 1.1
- **Package**: QFN-80 (10x10mm)

---

## 3. Power Management System

### 3.1 Power Input Design

The C6 features dual power input options:
1. **USB-C**: For development, programming, and benchtop testing
2. **Terminal connector**: For battery operation during flight

**Power shunt option**: Solder pads allow bridging main VCC and pyro VCC for simplified single-supply operation during testing.

### 3.2 USB-C Implementation

**Connector Selection**:
- **Single pin configuration** (Type-C 16-pin, using only necessary pins)
- **Simplified design**: Reduces PCB complexity
- **Power and data**: Supports both power delivery and USB communication

### 3.3 ESD Protection

**Design Philosophy**: Critical for USB interface reliability

**ESD Diode Selection Criteria** (based on TI guidelines):

1. **Working Voltage (VRWM)**
   - Must be ≥ signal line voltage
   - Also called "Reverse Stand-off Voltage"
   - Ensures normal operation without clamping

2. **Clamping Voltage**
   - Voltage seen during ESD event
   - Should be ≤ TLP (Transmission Line Pulse) rating of device
   - TLP ratings are difficult to find in datasheets
   - NOT the absolute maximum rating
   - **Rule**: Find diode with minimum clamping voltage possible

3. **Capacitance** (Most critical for high-speed interfaces)
   - **Ultra-low**: ≤ 0.5 pF
   - **Low**: 0.5 - 1.5 pF
   - **General purpose**: > 1.5 pF
   - USB 2.0 requires low capacitance

**PCB Layout Guidelines**:
```
Signal Flow: Connector → ESD Diode → Filter → IC
```

### 3.4 Voltage Regulation

**Primary Regulator**: TLV767 (3.3V LDO)

**Selection Rationale**:
- Superior performance compared to AMS1117
- Better line regulation
- Lower dropout voltage
- 1A current capability
- Fixed voltage version eliminates need for resistor divider

**Critical Note**: Purchase the **fixed 3.3V version** to avoid external resistor divider requirement.

**Design Considerations**:
- Input capacitor: Ceramic, 10µF recommended
- Output capacitor: Ceramic, 22µF recommended
- Keep capacitors close to regulator pins
- Pour solid ground plane beneath regulator

**5V Requirement**: Future consideration for camera modules or other peripherals requiring 5V supply.

### 3.5 Reverse Polarity Protection

Implemented to prevent damage from incorrect battery connection:
- Protection MOSFET in series with input
- Ensures correct polarity before power reaches sensitive components

### 3.6 Power Distribution Strategy

**Separate Power Rails**:
1. **Main VCC**: Powers microcontroller and sensors
2. **Pyro VCC**: Dedicated supply for pyrotechnic channels
3. **Isolated by default**, can be shunted for testing

**Power Budget Analysis**:
- TLV767 provides 1A maximum
- Sufficient for:
  - RP2350B: ~100-200mA
  - Sensors: ~50-100mA total
  - GPS: ~30mA
  - Indicators: ~20mA
- **Not sufficient** for powering RF telemetry modules (powered separately)

---

## 4. Sensor Suite

### 4.1 Pressure Sensors

**Dual-Sensor Configuration**: Design includes footprints for two pressure sensors to allow flexibility and redundancy.

#### 4.1.1 MS5611 (High-End Option)

**Specifications**:
- Resolution: 10 cm
- Operating range: 10-1200 mbar
- Interface: I2C
- Price: ~₹800

**I2C Configuration**:
- **CSB pin**: Determines LSB of I2C address
- MS5611 has CSB pulled high internally
- **I2C Address**: `0x77` (1110111)
- **CSB High**: I2C mode
- **CSB Low**: SPI mode (locked at power-on)

**Implementation Notes**:
- Tie CSB to VCC for I2C operation
- High-precision barometric measurements
- Temperature compensation built-in

#### 4.1.2 BMP390 (Cost-Effective Option)

**Specifications**:
- Resolution: 8 cm
- Operating range: 300-1250 hPa
- Interface: I2C / SPI
- Price: ~₹300

**I2C Configuration**:
- **SDO pin**: Determines I2C address
- SDO tied to GND
- **I2C Address**: `0x76` (1110110)
- **Different from MS5611** - allows both sensors on same bus

**Mode Selection**:
- **CSB Low**: SPI mode (locked at power-on)
- **CSB High**: I2C mode
- Tie CSB to VCC for I2C-only operation

**Design Decision**: Include footprints for both sensors, populate based on budget and performance requirements. Different I2C addresses allow simultaneous use if desired.

### 4.2 Inertial Measurement Units

#### 4.2.1 BNO055 (9-Axis Fusion IMU)

**Specifications**:
- 9-axis sensor fusion
- Accelerometer: ±16g maximum
- Gyroscope: Integrated
- Magnetometer: Integrated
- **Onboard sensor fusion**: Quaternion output
- Price: ~₹1000

**I2C Configuration**:
- **Protocol Mode**: HID-I2C (Human Interface Device)
- **COM3 pin**: Tied to GND
- **I2C Address**: `0x28` (01010010)

**Pin Configuration**:
- **nBOOT_LOAD_PIN**: Tied to +3.3V
  - Active-low bootloader entry
  - High = normal operation
  - Firmware updates not expected
- **nRESET**: Tied to +3.3V through pull-up
  - Active-low reset
  - Standard reset configuration

**Advantages**:
- Built-in sensor fusion reduces microcontroller load
- Calibration routines included
- Direct quaternion output
- Magnetometer for orientation reference

**Limitations**:
- Maximum ±16g - insufficient for high-G flight phases

#### 4.2.2 ADXL375 (High-G Accelerometer)

**Specifications**:
- **Range**: ±200g
- 3-axis measurement
- High shock survival: 10,000g
- Interface: I2C / SPI
- Price: ~₹1300

**I2C Configuration**:
- **ALT ADDRESS pin**: Tied to GND
- **Base Address**: `0x53`
- **Read Address**: `0xA7`
- **Write Address**: `0xA6`

**Pull-up Resistors**:
- **Value**: 4.7kΩ (rule of thumb for 100-400kHz I2C)
- Applied to SDA and SCL lines

**Design Rationale**:
- Complements BNO055 for high-G flight phases
- Captures motor burn acceleration
- Detects landing impacts
- Provides redundancy for critical acceleration data

**Combined IMU Strategy**:
- **BNO055**: Primary IMU for normal flight (±16g), sensor fusion, orientation
- **ADXL375**: High-G events (motor burn, landing)
- **Total IMU Cost**: ~₹2300

### 4.3 GPS/GNSS Module

#### 4.3.1 NEO-M9N Selection

**Specifications**:
- **Satellite Systems**: GPS, GLONASS, Galileo, BeiDou (4 constellations)
- **Update Rate**: 25Hz
- **Interface**: I2C (fast mode only)
- **Logic Level**: 3.3V (matches system)
- **Price**: ~₹2500

**Selection Rationale**:
- High accuracy multi-constellation support
- 25Hz update rate suitable for rocketry
- I2C interface simplifies wiring
- Compact module despite not being smallest available
- Preferred over MAX-M10: better accuracy, faster updates (trade-off: higher power consumption, acceptable for application)

**Documentation**:
- Datasheet provides electrical specifications
- **Integration Manual** provides critical implementation guidance
- Both documents essential for successful implementation

**I2C Configuration**:
- **Fast mode only**: 400kHz I2C required
- Must be connected to appropriate I2C bus
- Standard 4.7kΩ pull-ups

**Antenna Configuration**:

**Connector**: SMA (SubMiniature version A)
- Allows external antenna mounting through airframe
- Future-proof for better antenna placement
- Easy to swap antenna types

**Antenna Types**:
1. **Active Antennas**
   - Built-in LNA (Low Noise Amplifier)
   - Built-in filtering
   - Requires power (V_ANT pin)
   - Better for challenging RF environments

2. **Passive Antennas**
   - No power required
   - Simpler implementation
   - Adequate for most applications

**Power Supply for Active Antennas**:
- **V_ANT pin**: Provides antenna power
- 3.3V supply from GPS module
- **External filtered supply**: Only needed if antenna voltage doesn't match GPS module (not required for 3.3V antennas)

**Antenna Options Considered**:
- L1/L2 rubber duck antennas (difficult to source)
- Ceramic patch antennas (via SMA-to-U.FL adapter)
- **Selected**: SMA-compatible active antenna

**Implementation Notes**:
- Reference Sparkfun NEO-M9N breakout schematic
- Keep metal away from antenna
- Applies to both GPS and telemetry antennas
- Antenna placement critical for reception

**UART Backup**:
- 0Ω resistors placed in RX/TX lines
- Allows switching to UART if needed
- I2C primary interface

**Backup Power** (V_BCKP):
- Considered but deemed overkill
- Provides RTC battery backup
- Not implemented in this design

### 4.4 Sensor Interface Architecture

**I2C Bus Strategy**:

Initially attempted to assign separate I2C pin pairs to each sensor - **this was a critical mistake**. The RP2350 has only 2 I2C peripherals (I2C0 and I2C1), each with one SDA and one SCL pin pair.

**Final Configuration**:

**I2C0 Bus** (GPIO 40 SDA, GPIO 41 SCL):
- ADXL375 (High-G Accelerometer) - 0x53
- BMP390 (Pressure Sensor) - 0x76
- NEO-M9N (GPS Module) - 0x42 (default)

**I2C1 Bus** (GPIO 10 SDA, GPIO 11 SCL):
- MS5611 (Pressure Sensor) - 0x77
- BNO055 (9-Axis IMU) - 0x28

**Design Benefits**:
1. **Redundancy**: Each bus has one pressure sensor and one IMU type
2. **Load distribution**: Splits high-bandwidth sensors across buses
3. **Fault isolation**: Failure of one bus doesn't disable all sensors
4. **Different sensor priorities**: Better IMU paired with better pressure sensor, and vice versa

**Pull-up Resistors**:
- **Value**: 4.7kΩ
- Applied to both SDA and SCL on each bus
- Suitable for 100-400kHz operation

**Address Conflict Prevention**:
All sensors on each bus have unique I2C addresses:
- I2C0: 0x53, 0x76, 0x42 ✓
- I2C1: 0x77, 0x28 ✓

---

## 5. Peripheral Systems

### 5.1 Camera System

#### 5.1.1 Camera Selection Process

**Requirements**:
- Small package size
- Onboard video storage (SD card)
- Controllable by flight computer
- Reasonable quality
- Compact integration

**Challenge**: Most small cameras are analog FPV (First Person View) cameras designed for drones:
- Output analog video signal
- Designed for real-time radio transmission
- **No onboard storage**
- Require separate DVR (Digital Video Recorder) for storage

**DVR Problem**:
- Converting analog to digital requires complex circuitry
- Designing custom DVR is time and resource intensive
- Most high-quality cameras require expensive dedicated DVRs

**Alternatives Considered**:
- Raspberry Pi Camera modules (too large, need separate processor)
- RunCam modules (analog output, need DVR)
- Custom camera + DVR solution (too complex)

#### 5.1.2 Final Selection: Seeed Studio XIAO ESP32-S3 Sense

**Specifications**:
- **Processor**: ESP32-S3 with WiFi and BLE 5.0
- **Camera**: OV2640 sensor
- **Storage**: MicroSD card slot
- **Additional**: Digital microphone, battery charging
- **Size**: Compact XIAO form factor
- **Price**: ~₹2500 (estimated)

**Advantages**:
1. **Self-contained**: Camera, processor, and storage in one module
2. **Direct recording**: Stores video directly to SD card
3. **Controllable**: Can be powered on/off by flight computer
4. **Compact**: Small package suitable for rocket integration
5. **Reasonable quality**: OV2640 adequate for flight documentation

**Implementation Notes**:
- Power controlled via transistor from flight computer
- Can be powered independently
- Multiple cameras can be used
- **Important**: Use transistor for power switching (not direct GPIO)

**Future Consideration**: Design separate camera control/storage board that can be manufactured alongside main flight computer - potential cost savings through shared manufacturing.

### 5.2 Data Storage (SD Card)

#### 5.2.1 SD Card Socket Selection

**Connector Type**: Flip-open type microSD socket

**Selection Rationale**:
- **Vibration protection**: Flip-open cover secures card during flight
- **Ease of access**: Simple to swap cards between flights
- **Reliability**: Mechanical security prevents ejection

**Supplier**: Sunrom Electronics

#### 5.2.2 Interface Configuration

**Protocol**: SPI (Serial Peripheral Interface)

**Pin Configuration**:
- **CS (Chip Select)**: Active LOW
- **SCK (Clock)**: SPI clock
- **MOSI (Master Out Slave In)**: Data to SD card
- **MISO (Master In Slave Out)**: Data from SD card

**Implementation Status**: 
- Socket selected and integrated in design
- SPI pins assigned on microcontroller
- Final interfacing code to be developed

### 5.3 State Indicators (LEDs and Buzzer)

#### 5.3.1 LED Indicators

**Power LED**:
- **Type**: Red SMD LED
- **Package**: 0603
- **Function**: Power-on indication
- **Supplier**: Sunrom Electronics

**RGB Status LEDs**:
- **Type**: SMD RGB LED
- **Package**: 5050
- **Configuration**: Common cathode
- **Quantity**: 2 units
- **Function**: Flight state indication, error codes, system status
- **Supplier**: Sunrom Electronics

**RGB LED Characteristics**:
- 3 LEDs in one package (Red, Green, Blue)
- Independent control of each color
- Allows complex status indication through color mixing
- Common cathode simplifies driver design

#### 5.3.2 Buzzer

**Model**: MLT-8530

**Specifications**:
- **Size**: Compact SMD package
- **Availability**: Sunrom Electronics

**Implementation**:
- **Driver Circuit**: Required (buzzer does not have built-in driver)
- **Components**: Transistor + diode
- **Dummy Pads**: 2 of 4 pads are mechanical only

**Circuit Design**:
- Transistor for switching
- Flyback diode for inductive spike protection
- GPIO-controlled activation

**Status**: Selected and integrated, driver circuit designed, pending final connection to microcontroller.

### 5.4 Pyrotechnic Channels

#### 5.4.1 Overview

**Quantity**: 4 independent channels

**Function**: Control of:
- Drogue parachute deployment
- Main parachute deployment
- Staging separation (if applicable)
- Additional deployment events

#### 5.4.2 MOSFET Switching

**Primary Option**: SSM6N43FU
- **Type**: Dual N-Channel MOSFET
- **Package**: SOT-363
- **Configuration**: 2 channels per IC (2 ICs total for 4 channels)

**Backup Option**: SX1308
- **Package**: SOT23-6
- **Availability**: Robu.in

**Switching Configuration**:
- Logic-level gate drive from microcontroller GPIO
- Source to ground
- Drain to pyrotechnic device
- Gate resistor for controlled switching

#### 5.4.3 Power Supply

**Dedicated Pyro Supply**:
- Separate terminal connector
- Isolated from main VCC
- Prevents voltage drop during firing from affecting sensors

**Shunt Option**:
- Resistor footprint between VCC and Pyro VCC
- Solder bridge for testing (single supply)
- Remove/don't populate for flight (isolated supplies)

#### 5.4.4 Safety Features

**Arming**:
- External switch connected via terminal
- Breaks pyro power supply
- Mechanical safety interlock

**Continuity Testing**:
- Software-based continuity check capability
- Pre-flight verification of pyro circuits
- Non-destructive testing

#### 5.4.5 Implementation Notes

- 4 dedicated GPIO pins for MOSFET control
- Current sensing not implemented (future enhancement)
- Protection diodes recommended across pyro loads

---

## 6. Communication Interfaces

### 6.1 I2C Bus Configuration

Detailed in Section 4.4 (Sensor Interface Architecture)

**Summary**:
- **Two I2C buses**: I2C0 and I2C1
- **Pull-ups**: 4.7kΩ on all SDA/SCL lines
- **Speed**: 100-400kHz standard, 400kHz for GPS
- **Address management**: Carefully planned to avoid conflicts

### 6.2 UART Telemetry

#### 6.2.1 Purpose

**Function**: Communication with external telemetry radio module

**Data Transmitted**:
- Real-time sensor data
- Flight state information
- Error conditions
- GPS coordinates

#### 6.2.2 Interface Design

**Protocol Selection**: UART chosen over I2C/SPI because:
- **Simplicity**: Point-to-point communication
- **Noise immunity**: Differential signaling option available
- **Standard**: Widely supported by radio modules
- **Reduced complexity**: No addressing or complex protocol

**Connector Configuration**:
- **Pin count**: 4 pins (RX, TX, VCC, GND)
- **Connector type**: Header pins (initially)
  - Easy for testing and debugging
  - Can be changed to JST or other connector for flight
  
**Power Considerations**:
- Telemetry module powered separately (not from flight computer)
- VCC pin provided for convenience/testing only
- TLV767 (1A) insufficient for most RF modules during transmission

**Data Integrity**:
- **TODO**: Implement error detection/correction at software level
- Checksum or CRC recommended
- Acknowledgment protocol for critical data

### 6.3 SPI Interface

**Primary Use**: SD card communication

**Secondary Use**: Available for breakout/expansion

**Implementation**:
- Standard SPI pins from RP2350B
- SD card uses dedicated CS line
- Additional CS lines available for expansion

### 6.4 Programming Interfaces

#### 6.4.1 Primary: USB-C

**Function**:
- Programming and debugging
- Serial communication
- Power supply during development

**Implementation**:
- Full USB 2.0 support via RP2350B
- UF2 bootloader support
- Native USB in firmware

#### 6.4.2 Backup: SWD/UART

**Purpose**: Failsafe programming if USB fails

**Breakout Connections**:
- RX/TX for UART programming
- SWD (Serial Wire Debug) for debugging
- Brought out to header or test points

**Use Case**: Recovery from failed USB firmware or hardware issues

---

## 7. PCB Design Process

### 7.1 Component Sizing Strategy

#### 7.1.1 Passive Component Selection

**Challenge**: Balancing size vs. solderability

**0402 vs 0603 Decision**:

**0402 (1.0mm × 0.5mm)**:
- **Pros**: 
  - Smaller footprint
  - Allows denser routing
  - Required near microcontroller (limited space)
- **Cons**: 
  - Difficult hand soldering
  - Requires good soldering equipment
  - Easy to lose

**0603 (1.6mm × 0.8mm)**:
- **Pros**: 
  - Easier hand assembly
  - More forgiving for rework
  - Standard for many applications
- **Cons**: 
  - Larger footprint
  - May not fit in tight spaces
  - "Quite large" in compact designs

**Final Strategy**:
- **0402**: Used for passives near microcontroller (space-constrained areas)
- **0603**: Considered for sensors and less-dense areas
- **Trade-off**: Assembly difficulty vs. board size

#### 7.1.2 Crystal Oscillator

**Critical Error Identified**:
- Initial design specified 15µF capacitors
- **Correct value**: 15pF (picofarads, not microfarads)
- Note added later after catching mistake

**Lesson**: Always double-check units, especially for timing components

### 7.2 Layout Considerations

#### 7.2.1 Microcontroller Decoupling

**Power Pins** (RP2350B):
- **IOVDD** (I/O voltage): Pins 5, 15, 24, 29, 41, 50, 60, 76
- **DVDD** (Core voltage): Pins 10, 32, 51

**Decoupling Strategy**:
- 0.1µF ceramic capacitor at each power pin
- As close as possible to pin
- Short, wide traces to ground plane
- Larger bulk capacitor (10µF) for each power domain

#### 7.2.2 ESD Protection Layout

**Signal Path**: Connector → ESD Diode → Filter → IC

**Guidelines**:
- Keep ESD diode very close to connector
- Short trace from connector to diode
- Ground connection must be low impedance
- Filter components between ESD and IC

#### 7.2.3 Antenna Considerations

**Critical Requirements**:
- Keep metal (ground plane, components) away from antenna
- Applies to both GPS and telemetry antennas
- Maintain clearance for proper directionality
- Consider antenna radiation pattern

**Learning Point**: Need deeper understanding of antenna theory, directionality, and ground plane interactions.

### 7.3 Signal Routing

#### 7.3.1 I2C Routing

**Guidelines**:
- Keep SDA and SCL traces together
- Equal length not critical at these speeds (100-400kHz)
- Avoid routing near high-speed signals
- Pull-ups close to termination points (near sensors)

#### 7.3.2 Power Distribution

**Strategy**:
- Wide traces for power distribution
- Solid ground plane (top priority)
- Separate analog and digital ground (single-point star connection)
- Kelvin connections for current sensing

### 7.4 Design Decisions and Trade-offs

#### 7.4.1 Solder Mask Expansion

**Decision**: Left at default settings

**Rationale**:
- First iteration of design
- Focus on functionality over optimization
- Can adjust in future revisions based on manufacturing feedback

#### 7.4.2 Terminal vs. Connector Selection

**Initial Thought Process**:
- Considered JST connectors for professional look
- Considered standard header pins for simplicity

**Decision**: Header pins initially
- **Advantages**:
  - Easier to test and debug
  - More flexible during development
  - Readily available
  - Low cost
- **Future**: Can change to JST or other connectors in production version

#### 7.4.3 Dual Pressure Sensor Footprints

**Decision**: Include pads for both MS5611 and BMP390

**Benefits**:
- Budget flexibility (use cheaper BMP390 for testing)
- Performance flexibility (use MS5611 for critical flights)
- Can populate both for redundancy
- Different I2C addresses enable both simultaneously

---

## 8. Critical Design Decisions

### 8.1 Component Selection Rationale

#### 8.1.1 Sensor Availability and Lead Time

**Key Consideration**: Component availability is critical

**Strategy**:
- Check availability before finalizing design
- Identify multiple suppliers
- Consider lead times in project planning
- Have backup component options

**Supplier Research**:
- LCSC (primary for ICs and SMD components)
- SemiKart.com (India-based)
- DigiKey.in (international, reliable)
- Sunrom Electronics (connectors, LEDs, buzzers)

#### 8.1.2 Logic Level Matching

**Requirement**: All sensors must match microcontroller logic level

**System Standard**: 3.3V

**Verification**:
- RP2350B: 3.3V I/O
- All sensors: 3.3V compatible
- GPS module: 3.3V
- No level shifters required

### 8.2 I2C Address Management

**Address Conflict Prevention**: Critical for reliable communication

**Complete I2C Address Map**:

| Device | Bus | Address (7-bit) | Config Pin |
|--------|-----|----------------|------------|
| MS5611 | I2C1 | 0x77 | CSB = HIGH |
| BMP390 | I2C0 | 0x76 | SDO = LOW |
| BNO055 | I2C1 | 0x28 | COM3 = LOW |
| ADXL375 | I2C0 | 0x53 | ALT ADDRESS = LOW |
| NEO-M9N | I2C0 | 0x42 | Default |

**Verification**: No address conflicts on either bus ✓

### 8.3 Pin Assignment Strategy

#### 8.3.1 Initial Mistake

**Error**: Attempted to assign separate GPIO pairs to each sensor
- Assumed each sensor could have dedicated SDA/SCL pins
- Did not account for peripheral limitations

**Problem Discovered**: 
> "Ohh wait, did I do something massively wrong by connecting to different pins of the same I2C?"
> "Yash You're a fuckin retardddddd boy"

**Root Cause**: Misunderstanding of I2C peripheral architecture
- RP2350B has 2 I2C controllers (I2C0, I2C1)
- Each controller has ONE SDA and ONE SCL pin
- Cannot use different GPIO pairs for same peripheral

#### 8.3.2 Corrected Configuration

**Final Pin Assignments**:

**I2C0** (Fast mode, 400kHz capable):
- **GPIO 40**: SDA0
- **GPIO 41**: SCL0
- **Devices**: ADXL375, BMP390, NEO-M9N

**I2C1** (Standard mode, 100kHz):
- **GPIO 10**: SDA1
- **GPIO 11**: SCL1
- **Devices**: MS5611, BNO055

**ADC**:
- **GPIO 47** (ADC3): Voltage monitoring

**SPI** (SD Card):
- MOSI, MISO, SCK, CS (pins to be finalized)

**UART** (Telemetry):
- TX, RX (pins to be finalized)

**Pyro Channels**:
- 4 GPIO pins (to be assigned)

**Indicators**:
- RGB LEDs: 6 GPIO (2 LEDs × 3 colors)
- Buzzer: 1 GPIO
- Power LED: Always on (not GPIO controlled)

### 8.4 Connector Selection

#### 8.4.1 External Interfaces

**Breakout Connections**:
- Initially: Terminals
- Reconsidering: Change to pads
- **Rationale**: Pads more compact, can solder wires or use pogo pins

**Interfaces to Break Out**:
- I2C buses (SDA, SCL, VCC, GND)
- SPI (MOSI, MISO, SCK, CS, VCC, GND)
- 3.3V power
- GND

---

## 9. Lessons Learned

### 9.1 Design Mistakes and Corrections

#### 9.1.1 Crystal Capacitor Value

**Error**: Specified 15µF instead of 15pF
**Impact**: Would prevent oscillator from functioning
**Detection**: Caught during design review
**Correction**: Changed to 15pF before manufacturing

**Lesson**: Always verify units, especially for frequency-determining components.

#### 9.1.2 I2C Pin Assignment Misunderstanding

**Error**: Attempted to assign separate pin pairs to each I2C device
**Impact**: Design wouldn't work; required complete pin reassignment
**Detection**: During detailed pin mapping phase
**Correction**: Moved to proper I2C bus architecture

**Lesson**: Thoroughly understand microcontroller peripheral architecture before starting schematic.

#### 9.1.3 GPIO Count Insufficiency

**Error**: Selected RP2350A without counting total GPIO requirements
**Impact**: Insufficient pins for all planned features
**Detection**: During final pin assignment
**Correction**: Upgraded to RP2350B (80 GPIO)

**Lesson**: Create complete pin assignment table BEFORE selecting microcontroller.

#### 9.1.4 Quad-Pin Switch Internal Connections

**Caution**: "MAKE SUREEE THAT THEY ARE CONNECTED PROPERLY (UNDERSTAND WHICH PINS ARE CONNECTED INTERNALLY)"

**Issue**: Quad-pin tactile switches have pins connected in pairs
- Not all 4 pins are independent
- Typically: pins 1-2 connected, pins 3-4 connected
- Switch bridges the two pairs

**Lesson**: Always check datasheet for internal connections on multi-pin components.

### 9.2 Technical Insights

#### 9.2.1 ESD Protection Complexity

**Learning**: ESD protection is more nuanced than initially thought
- Three critical parameters must be balanced
- TLP ratings are difficult to find
- Capacitance matters significantly for USB
- Layout is as important as component selection

**Resource**: TI video on USB-C ESD protection was invaluable

#### 9.2.2 Camera System Complexity

**Learning**: Camera integration more complex than anticipated
- Most small cameras are analog (FPV drones)
- DVR design is non-trivial
- Integrated solutions (like ESP32-S3 Sense) are valuable
- Cost vs. complexity trade-off

**Time Spent**: "I think I've spent too much time on this decision"

**Outcome**: Integrated module approach (ESP32-S3) justified despite cost

#### 9.2.3 Antenna Design Knowledge Gap

**Self-Assessment**: "Need to learn more in depth about antennas"

**Areas Identified**:
- Directionality and radiation patterns
- Ground plane effects
- Metal clearance requirements
- Active vs. passive antenna trade-offs

**Application**: Critical for both GPS and telemetry performance

### 9.3 Best Practices Identified

#### 9.3.1 Documentation During Design

**Practice**: Creating detailed notes during design process

**Benefits**:
- Reduces re-questioning previous decisions
- Captures rationale for component selection
- Documents mistakes and corrections
- Aids future designs

**Quote**: "I really question all my previous research all the time and I dont understand why, I've started making notes this time in order to reduce that recurrence."

#### 9.3.2 Address Management for I2C

**Practice**: Create address map early in design

**Process**:
1. List all I2C devices
2. Determine default addresses
3. Identify address selection pins
4. Configure pins to avoid conflicts
5. Document in table

**Benefit**: Prevents conflicts discovered late in design or during testing

#### 9.3.3 Redundancy Through Bus Distribution

**Practice**: Distribute sensors across multiple buses

**Benefits**:
- Fault isolation
- Bandwidth distribution
- Partial system operation on bus failure

**Implementation**: Each I2C bus has one pressure sensor and one IMU type

#### 9.3.4 Flexible Power Options

**Practice**: Include shunt option between power domains

**Benefits**:
- Simplified testing with single supply
- Production configuration with isolated supplies
- Easy to switch between modes

### 9.4 Future Improvements

#### 9.4.1 Voltage Monitoring Enhancement

**Current Implementation**: Basic voltage divider for battery monitoring
**Future**: Consider current sensing for power budget analysis

#### 9.4.2 Pyro Continuity Testing

**Current**: Software-based continuity (to be implemented)
**Future**: Hardware continuity sense resistors for more reliable detection

#### 9.4.3 Camera Control Board

**Concept**: Separate camera control/storage board
**Benefits**: 
- Manufactured with flight computer (cost savings)
- Modular design
- Easier to test independently

#### 9.4.4 Connector Standardization

**Current**: Mix of terminals and headers
**Future**: Standardize on JST or other connector family for production

#### 9.4.5 Data Integrity

**Current**: Basic UART communication
**To Implement**: Error detection/correction protocol
**Options**: Checksum, CRC, acknowledgment system

---

## 10. Bill of Materials

### 10.1 Microcontroller and Supporting Components

| Component | Part Number | Package | Quantity | Est. Cost (₹) |
|-----------|-------------|---------|----------|---------------|
| Microcontroller | RP2350B | QFN-80 | 1 | ~500 |
| Crystal | To be specified | SMD | 1 | ~50 |
| Crystal Load Caps | 15pF | 0402 | 2 | ~5 |
| Flash Memory | To be specified | SOIC-8 | 1 | ~100 |
| Decoupling Caps | 0.1µF | 0402 | 10+ | ~20 |
| Bulk Caps | 10µF | 0603 | 3 | ~15 |

### 10.2 Sensors and Modules

| Component | Part Number | Quantity | Est. Cost (₹) | Notes |
|-----------|-------------|----------|---------------|-------|
| Pressure Sensor (High-end) | MS5611 | 1 | 800 | Optional |
| Pressure Sensor (Budget) | BMP390 | 1 | 300 | Optional |
| 9-Axis IMU | BNO055 | 1 | 1000 | Primary IMU |
| High-G Accelerometer | ADXL375 | 1 | 1300 | Secondary IMU |
| GPS Module | NEO-M9N | 1 | 2500 | Include integration manual |
| Camera Module | ESP32-S3 Sense | 1-2 | 2500 each | Seeed Studio XIAO |

**Sensor Subtotal**: ₹5600 - ₹8900 (depending on configuration)

### 10.3 Power Components

| Component | Part Number | Package | Quantity | Est. Cost (₹) |
|-----------|-------------|---------|----------|---------------|
| 3.3V LDO Regulator | TLV767 (3.3V fixed) | SOT23-5 | 1 | ~50 |
| Input Cap (LDO) | 10µF | 0805 | 1 | ~5 |
| Output Cap (LDO) | 22µF | 0805 | 1 | ~5 |
| ESD Diode | To be specified | SOT23 | 2+ | ~20 |
| Reverse Protection | P-Channel MOSFET | SOT23 | 1 | ~15 |

### 10.4 Passive Components

| Component | Value | Package | Quantity | Est. Cost (₹) |
|-----------|-------|---------|----------|---------------|
| Pull-up Resistors (I2C) | 4.7kΩ | 0402 | 8 | ~10 |
| Pull-up (Reset) | 10kΩ | 0402 | 1 | ~2 |
| Reset Cap | 1µF | 0402 | 1 | ~2 |
| Voltage Divider R1 | 12kΩ | 0402 | 1 | ~2 |
| Voltage Divider R2 | 4.7kΩ | 0402 | 1 | ~2 |
| Gate Resistors (MOSFETs) | To be determined | 0402 | 4 | ~10 |
| LED Current Limit | To be determined | 0402 | 7 | ~15 |

### 10.5 Connectors and Indicators

| Component | Part Number | Quantity | Est. Cost (₹) | Source |
|-----------|-------------|----------|---------------|--------|
| USB-C Connector | 16-pin | 1 | ~50 | Generic |
| MicroSD Socket | Flip-open type | 1 | ~100 | Sunrom |
| Power Terminal | 2-pin | 2 | ~20 | Generic |
| Pyro Terminal | 2-pin | 1 | ~10 | Generic |
| SMA Connector (GPS) | SMA | 1 | ~100 | Generic |
| Header Pins | 2.54mm pitch | As needed | ~50 | Generic |
| Power LED | Red 0603 | 1 | ~5 | Sunrom |
| RGB LED | 5050 Common Cathode | 2 | ~50 | Sunrom |
| Buzzer | MLT-8530 | 1 | ~50 | Sunrom |
| Tactile Switches | Quad-pin | 2 | ~10 | Generic |

### 10.6 Pyrotechnic Components

| Component | Part Number | Package | Quantity | Est. Cost (₹) |
|-----------|-------------|---------|----------|---------------|
| Dual N-Channel MOSFET | SSM6N43FU | SOT-363 | 2 | ~40 |
| Backup MOSFET | SX1308 | SOT23-6 | 2 | ~30 |
| Flyback Diodes | To be specified | SOD-123 | 4 | ~20 |

### 10.7 Cost Summary

| Category | Estimated Cost (₹) |
|----------|-------------------|
| Microcontroller & Support | 700 |
| Sensors & Modules (max config) | 8900 |
| Power Components | 95 |
| Passive Components | 45 |
| Connectors & Indicators | 445 |
| Pyrotechnic Components | 90 |
| **Subtotal (Components)** | **~10,300** |
| PCB Manufacturing | ~1000-2000 |
| **Total (Estimated)** | **~11,500 - 12,500** |

**Notes**:
- Costs are estimates based on notes (circa 2025)
- Actual costs vary by supplier and order quantity
- GPS antenna not included
- SD card not included
- Pricing in Indian Rupees (₹)

---

## 11. Appendices

### Appendix A: Component Datasheets Reference

**Microcontroller**:
- RP2350B: Raspberry Pi official documentation

**Sensors**:
- MS5611: [LCSC Link in notes]
- BMP390: [LCSC Link in notes]
- BNO055: [LCSC Link in notes]
- ADXL375: [LCSC Link in notes]
- NEO-M9N: Datasheet + Integration Manual (both essential)

**Power**:
- TLV767: Texas Instruments
- ESD Protection: TI Video on USB-C ESD (Link in notes)

**Connectors**:
- MicroSD Socket: Sunrom [Link in notes]
- Buzzer MLT-8530: [Sunrom PDF in notes]

**Reference Designs**:
- Sparkfun NEO-M9N SMA Breakout: [Link in notes]

### Appendix B: I2C Address Map

#### Complete Address Allocation

**I2C0 Bus (GPIO 40 SDA, GPIO 41 SCL)**:
| Device | 7-bit Address | Read Address | Write Address | Config |
|--------|---------------|--------------|---------------|--------|
| ADXL375 | 0x53 | 0xA7 | 0xA6 | ALT ADDRESS = GND |
| BMP390 | 0x76 | - | - | SDO = GND, CSB = VCC |
| NEO-M9N | 0x42 | - | - | Default |

**I2C1 Bus (GPIO 10 SDA, GPIO 11 SCL)**:
| Device | 7-bit Address | Config |
|--------|---------------|--------|
| MS5611 | 0x77 | CSB = VCC (internal pull-up) |
| BNO055 | 0x28 | COM3 = GND, HID-I2C mode |

**Pull-up Resistors**: 4.7kΩ on all SDA and SCL lines

### Appendix C: Pin Assignment Table

#### RP2350B Final Pin Mapping

| GPIO | Function | Device/Purpose | Notes |
|------|----------|----------------|-------|
| 10 | I2C1 SDA | MS5611, BNO055 | |
| 11 | I2C1 SCL | MS5611, BNO055 | |
| 40 | I2C0 SDA | ADXL375, BMP390, GPS | Fast mode (400kHz) |
| 41 | I2C0 SCL | ADXL375, BMP390, GPS | Fast mode (400kHz) |
| 47 | ADC3 | Voltage Monitoring | Voltage divider input |
| TBD | SPI MOSI | SD Card | |
| TBD | SPI MISO | SD Card | |
| TBD | SPI SCK | SD Card | |
| TBD | SPI CS | SD Card | Chip select |
| TBD | UART TX | Telemetry | |
| TBD | UART RX | Telemetry | |
| TBD | GPIO | Pyro Channel 1 | MOSFET gate |
| TBD | GPIO | Pyro Channel 2 | MOSFET gate |
| TBD | GPIO | Pyro Channel 3 | MOSFET gate |
| TBD | GPIO | Pyro Channel 4 | MOSFET gate |
| TBD | GPIO | RGB LED 1 - Red | |
| TBD | GPIO | RGB LED 1 - Green | |
| TBD | GPIO | RGB LED 1 - Blue | |
| TBD | GPIO | RGB LED 2 - Red | |
| TBD | GPIO | RGB LED 2 - Green | |
| TBD | GPIO | RGB LED 2 - Blue | |
| TBD | GPIO | Buzzer | Through driver circuit |
| TBD | USB D+ | USB-C | Built-in |
| TBD | USB D- | USB-C | Built-in |

**Power Pins**:
- IOVDD (I/O Supply): Pins 5, 15, 24, 29, 41, 50, 60, 76
- DVDD (Core Supply): Pins 10, 32, 51

**Note**: "TBD" pins to be assigned in final schematic phase

### Appendix D: Supplier Information

#### Primary Suppliers

**LCSC (China)**:
- Electronics components
- ICs, sensors, passives
- PCB manufacturing (JLCPCB)
- International shipping to India
- Consider customs duties

**SemiKart.com (India)**:
- Electronics components
- Reduced shipping time
- No customs concerns
- Potentially higher prices

**DigiKey.in (International)**:
- Comprehensive catalog
- Reliable shipping
- Technical support
- Higher costs, longer lead times

**Sunrom Electronics (India)**:
- Connectors (SD card socket)
- LEDs (power, RGB)
- Buzzer (MLT-8530)
- Local inventory

**Robu.in (India)**:
- Backup MOSFET (SX1308)
- Development modules
- Camera modules
- Local shipping

#### Sourcing Strategy

**Component Selection Process**:
1. Check availability at multiple suppliers
2. Compare lead times
3. Consider minimum order quantities
4. Factor shipping and customs costs
5. Identify backup components

**Procurement Notes**:
- Order long-lead items early (GPS, specialty sensors)
- Order extra passives (0402 components easily lost)
- Keep backup component options
- Consider supply chain disruptions in planning

---

## 12. Personal Reflections

### 12.1 On the Design Process

**Self-Awareness**: The notes reveal introspection about the design process:

*"I really question all my previous research all the time and I dont understand why, I've started making notes this time in order to reduce that recurrence. And I still feel I do it often."*

**Analysis**: 
- Recognizes tendency to second-guess decisions
- Attributes to "underconfident nature"
- Links to perfectionism
- Documentation as a solution

**Value**: This self-awareness is a strength - thorough review catches mistakes (like the crystal capacitor value).

### 12.2 On Decision-Making

**Camera Selection**:
*"I think I've spent too much time on this decision, but now Finally have decided to use the ESP32S3 sense by seeed"*

**Observation**: Complex decisions require time - the camera system investigation was thorough and resulted in an optimal solution.

### 12.3 On Learning

**Antenna Knowledge**:
*"Need to learn more in depth about antennas, should know in depth how they work"*

**Growth Mindset**: Identifying knowledge gaps is the first step to addressing them. The design works around current limitations while noting areas for future learning.

### 12.4 On Making Mistakes

**I2C Pin Assignment**:
*"Youre an absolute fuckin retard, you cant use all these pins for just scl and sda"*

**Healthy Approach**: While self-critical, the mistake was caught and corrected. These errors are normal in complex designs and are valuable learning experiences.

---

## 13. Conclusion

The C6 Flight Computer represents a comprehensive approach to rocket avionics design. Through careful component selection, redundant sensor architecture, and robust power management, the design achieves a balance between capability, reliability, and practical implementation.

**Key Achievements**:
- ✅ Redundant sensor suite for critical measurements
- ✅ Flexible power options for testing and flight
- ✅ Multiple communication interfaces
- ✅ Integrated data storage
- ✅ Safety-conscious pyrotechnic control
- ✅ Comprehensive documentation of design process

**Challenges Overcome**:
- Microcontroller GPIO limitations → Upgraded to RP2350B
- I2C peripheral misunderstanding → Corrected bus architecture
- Camera system complexity → Integrated module solution
- Component availability → Multi-supplier strategy

**Next Steps**:
1. Complete final pin assignments (TBD items)
2. Finish PCB layout
3. Design review with focus on ESD and power
4. Order components (considering lead times)
5. PCB manufacturing
6. Assembly and bring-up testing
7. Sensor calibration
8. Firmware development
9. Ground testing
10. Flight testing

This documentation serves as both a record of the design process and a guide for future iterations. The lessons learned, particularly around I2C architecture and GPIO planning, will inform subsequent projects and help avoid similar mistakes.

The combination of detailed technical specifications and personal reflections makes this a complete record - not just *what* was designed, but *why* decisions were made and *how* problems were solved.

---

**Document Version**: 1.0  
**Based on**: Mark1.docx design notes  
**Flight Computer**: C6  
**Design Date**: 2025  
**Documentation Created**: October 2025