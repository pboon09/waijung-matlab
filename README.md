# SerialFrame Library for STM32-MATLAB Communication

This project provides a complete guide to setting up SerialFrame library on an STM32 microcontroller for communication with MATLAB/Simulink using the Waijung toolbox. It walks through the steps required to get started, from understanding the library to implementing your first data exchange application.

## Requirements

Before starting, make sure you have the following:
- STM32CubeIDE installed
- MATLAB with Waijung Blockset installed (File provided)
- An STM32 microcontroller (tested on STM32G4 series)
- A USB cable to connect your STM32 board to your computer

## About SerialFrame

SerialFrame is a lightweight library that simplifies the process of transferring structured data between an STM32 microcontroller and MATLAB. It handles the low-level details of:
- Buffer management
- DMA transfers
- Data serialization and deserialization
- Frame construction and parsing

## Step 1 - Understanding Frame Format

The data frame has a simple structure:
```
[Header][Data Field 1][Data Field 2]...[Data Field N][Terminator]
```

- **Header**: A single byte value (e.g., 37) that marks the beginning of the frame
- **Data Fields**: Variable-sized fields according to their data types
- **Terminator**: A single byte value (e.g., 'N') that marks the end of the frame

## Step 2 - Add SerialFrame to Your Project

1. Add the `serial_frame.c` and `serial_frame.h` files to your project
2. Include the header file in your main application:
   ```c
   #include "serial_frame.h"
   ```

## Step 3 - Configure Your Project

### 1. Configure UART with DMA

In your STM32CubeIDE project:
- Enable a UART or LPUART peripheral
- Enable DMA for both TX and RX
- Configure the baud rate (typically 115200 or higher)
- Enable UART interrupt

For example:
- Set LPUART1 to `Asynchronous` mode
- In DMA Settings tab:
  - Add RX - `Mode: Circular`
  - Add TX
- In NVIC Settings tab, enable `global interrupt`

### 2. Generate Code

Generate the code for your project using STM32CubeIDE.

## Step 4 - Implementing SerialFrame

### 1. Initialize SerialFrame

Initialize the SerialFrame structure with your UART handle and define header and terminator bytes:

```c
SerialFrame serial_frame;
SerialFrame_Init(&serial_frame, &hlpuart1, 37, 'N'); // Header: 37, Terminator: 'N'
```

### 2. Define Transmit Variables

Define the variables you want to transmit and add them to the frame:

```c
float my_float1, my_float2, my_float3;
int16_t my_int1, my_int2, my_int3;

// Add transmit fields
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_FLOAT, &my_float1, "Float1");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_FLOAT, &my_float2, "Float2");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_FLOAT, &my_float3, "Float3");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_INT16, &my_int1, "Int1");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_INT16, &my_int2, "Int2");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_INT16, &my_int3, "Int3");
```

### 3. Define Receive Variables

Define variables to store received data and add them to the frame:

```c
float rx_float1, rx_float2, rx_float3;
int16_t rx_int1, rx_int2, rx_int3;

// Add receive fields
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_FLOAT, &rx_float1, "RxFloat1");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_FLOAT, &rx_float2, "RxFloat2");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_FLOAT, &rx_float3, "RxFloat3");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_INT16, &rx_int1, "RxInt1");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_INT16, &rx_int2, "RxInt2");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_INT16, &rx_int3, "RxInt3");
```

### 4. Start Receiving

Start the reception process:

```c
SerialFrame_StartReceive(&serial_frame);
```

### 5. Set Up Periodic Transmission

For periodic transmission, use a timer interrupt:

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim == &htim2) {
        // Update your data values here
        my_float1 = ...;
        my_float2 = ...;
        my_float3 = ...;
        my_int1 = ...;
        my_int2 = ...;
        my_int3 = ...;
        
        // Build and transmit the frame
        SerialFrame_BuildTxFrame(&serial_frame);
        SerialFrame_Transmit(&serial_frame);
    }
}
```

### 6. Handle Received Data

Implement the UART RX complete callback:

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart == &hlpuart1) {
        // Parse received frame
        SerialFrame_ParseRxFrame(&serial_frame);
        
        // Start receiving again
        SerialFrame_StartReceive(&serial_frame);
        
        // Use received data (rx_float1, rx_int1, etc.)
        // ...
    }
}
```

## Step 5 - Setting Up MATLAB with Waijung

### 1. Create a New Simulink Model

Open MATLAB and create a new Simulink model.

### 2. Configure Serial Setup Block

1. Add a "Serial_Setup" block from the Waijung Blockset to your model
2. Configure the block parameters:
   - Port: COM port where your STM32 is connected (e.g., COM11)
   - Baud: 2000000 (or match your STM32 UART configuration)
   - Other parameters: 8-None-1 (8 data bits, no parity, 1 stop bit)

### 3. Configure Waijung Serial Receive Block

1. Add a "Serial_Receive" block from the Waijung Blockset to your model
2. Double-click the block to configure its parameters:
   - Port: Same as Serial_Setup (e.g., COM11)
   - Packet: Binary
   - Transfer: Non-Blocking
   - Baud: 2000000 (match your STM32 configuration)
   - Binary header: '25' (in hex, equals 37 in decimal)
   - Binary terminator: '4E' (in hex, equals 'N' in ASCII)
   - Number of data port, type SINGLE: 3 (for your three float values)
   - Number of data port, type INT16: 3 (for your three int16 values)
   - Set all other data type counts to 0

The block will automatically create output ports for each data field:
- First output will be a READY signal
- Next three outputs will be your SINGLE (float) values
- Last three outputs will be your INT16 values

### 4. Configure Waijung Serial Transmit Block

1. Add a "Serial_Transmit" block from the Waijung Blockset to your model
2. Configure the block parameters:
   - Port: Same as Serial_Setup (e.g., COM11)
   - Packet: Binary
   - Baud: 2000000 (match your configuration)
   - Binary header: '25' (in hex, equals 37 in decimal)
   - Binary terminator: '4E' (in hex, equals 'N' in ASCII)
   - Number of data port, type SINGLE: 3 (for three float values)
   - Number of data port, type INT16: 3 (for three int16 values)
   - Set all other data type counts to 0

The block will automatically create input ports for each data field:
- First three inputs will be for your SINGLE (float) values
- Last three inputs will be for your INT16 values

### 5. Connect Blocks and Add Data Sources/Sinks

1. Connect the output ports of the Serial_Receive block to scopes, displays, or other blocks to visualize received data
2. Connect signal sources (constants, signal generators, etc.) to the input ports of the Serial_Transmit block
3. Your completed model should look similar to this:

### 6. Run the Simulation

Click the "Run" button in Simulink to start the simulation. If everything is configured correctly, you should see data being exchanged between MATLAB and your STM32 device.

**Important Notes:**
- The hexadecimal values in Waijung (e.g., '25' for header and '4E' for terminator) correspond to the decimal/ASCII values in your STM32 code (37 and 'N')
- Make sure the data types and order match exactly between your STM32 code and Simulink model
- The sample time (-1) indicates that the blocks will run at the base sample rate of the model

## Example Implementation

A complete example is provided in the source files with the following configuration:

### STM32 Side
```c
// Variables to send to MATLAB
float dum_float1, dum_float2, dum_float3;
int16_t dum_int1, dum_int2, dum_int3;

// Variables to receive from MATLAB
float rev_float1, rev_float2, rev_float3;
int16_t rev_int1, rev_int2, rev_int3;

SerialFrame serial_frame;

// Initialize SerialFrame
SerialFrame_Init(&serial_frame, &hlpuart1, 37, 'N'); // 37 decimal = '25' hex, 'N' = '4E' hex

// Add transmit fields in order
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_FLOAT, &dum_float1, "Float1");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_FLOAT, &dum_float2, "Float2");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_FLOAT, &dum_float3, "Float3");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_INT16, &dum_int1, "Int1");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_INT16, &dum_int2, "Int2");
SerialFrame_AddTxField(&serial_frame, SERIAL_TYPE_INT16, &dum_int3, "Int3");

// Add receive fields in order
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_FLOAT, &rev_float1, "RevFloat1");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_FLOAT, &rev_float2, "RevFloat2");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_FLOAT, &rev_float3, "RevFloat3");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_INT16, &rev_int1, "RevInt1");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_INT16, &rev_int2, "RevInt2");
SerialFrame_AddRxField(&serial_frame, SERIAL_TYPE_INT16, &rev_int3, "RevInt3");
```

### MATLAB Simulink Side

The following Waijung blocks are used:

1. **Serial_Setup Block**:
   - Port: COM11
   - Baud: 2000000
   - Config: 8-None-1

2. **Serial_Receive Block**:
   - Port: COM11
   - Packet mode: Binary
   - Transfer: Non-Blocking
   - Binary header: '25' (hex for 37 decimal)
   - Binary terminator: '4E' (hex for 'N' ASCII)
   - Number of SINGLE data ports: 3
   - Number of INT16 data ports: 3

3. **Serial_Transmit Block**:
   - Port: COM11
   - Packet mode: Binary
   - Binary header: '25' (hex for 37 decimal)
   - Binary terminator: '4E' (hex for 'N' ASCII)
   - Number of SINGLE data ports: 3
   - Number of INT16 data ports: 3

The data flow in the Simulink model matches the SerialFrame configuration on the STM32:
- STM32 transmits: 3 floats followed by 3 int16s → MATLAB receives these in Serial_Receive
- MATLAB transmits: 3 floats followed by 3 int16s → STM32 receives these in SerialFrame

## Calculating Frame Size and Memory Requirements

### Frame Size Calculation

When setting up your SerialFrame and Waijung blocks, it's important to ensure the frame size matches on both sides:

1. **Header**: 1 byte
2. **Data fields**: Sum of all field sizes
   - UINT8/INT8: 1 byte each
   - UINT16/INT16: 2 bytes each
   - UINT32/INT32/FLOAT: 4 bytes each
   - DOUBLE: 8 bytes each
3. **Terminator**: 1 byte

**Example calculation for our configuration:**
- Header: 1 byte
- 3 × FLOAT (dum_float1, dum_float2, dum_float3): 3 × 4 = 12 bytes
- 3 × INT16 (dum_int1, dum_int2, dum_int3): 3 × 2 = 6 bytes
- Terminator: 1 byte
- **Total**: 1 + 12 + 6 + 1 = 20 bytes

This total frame size should match the sum of data port sizes in your Waijung blocks.

### Memory Usage

The SerialFrame library uses the following memory:
- Two buffer arrays of size `SERIAL_FRAME_MAX_SIZE` (default 255 bytes)
- Two arrays of field definitions (each field is about 8 bytes)
- Minimal stack usage for function calls

For an STM32G4 microcontroller, this is typically not a concern, but for smaller MCUs, you might want to reduce `SERIAL_FRAME_MAX_SIZE` and `SERIAL_FRAME_MAX_FIELDS` in the header file.

## Tips for Reliable Communication

1. **Match Configurations Exactly**: 
   - Use identical baud rates, data formats, and frame structures on both sides
   - Remember that Waijung uses hexadecimal values for header/terminator (e.g., '25' hex = 37 decimal)

2. **Optimize Data Transfer**:
   - Use a high enough baud rate (2000000 works well for most applications)
   - Keep data transmission periodic and consistent
   - Use DMA for UART communication to reduce CPU load

3. **Frame Structure**:
   - Keep frame size under 255 bytes for better reliability
   - Maintain consistent field ordering on both sides
   - Add a checksum field if operating in a noisy environment

4. **Error Handling**:
   - Validate received frames before processing (check header/terminator)
   - Implement timeout handling in case of communication failures
   - Consider adding a sequence number to detect missed frames

## Troubleshooting

If you encounter issues with the communication:

1. **No Communication**:
   - Verify COM port number and settings
   - Check that TX from STM32 is connected to RX on PC and vice versa
   - Ensure baudrate is set correctly and identically on both sides

2. **Corrupted Data**:
   - Verify header and terminator values match (remember: decimal in C, hex in Waijung)
   - Check that the data types and their order match exactly
   - Reduce baud rate to see if reliability improves
   - Examine the raw data with a logic analyzer or serial monitor

3. **Waijung-Specific Issues**:
   - Make sure the READY signal from Serial_Receive is high
   - If using high baudrates, ensure your PC has sufficient processing power
   - Try restarting Simulink if communication becomes unstable

4. **STM32-Specific Issues**:
   - Verify DMA is properly configured for UART
   - Check that interrupts are enabled and prioritized correctly
   - Use HAL_UART_STATE_READY to verify UART is not busy before transmitting

## Limitations

- Maximum frame size is limited to 255 bytes (can be modified in header)
- Maximum number of fields per frame is 32 (can be modified in header)
- No built-in error detection (consider adding a checksum field if needed)
- Waijung requires specific ordering of data types in the blocks

## Advanced Features

- **Named Fields**: Fields can be named for better code readability
- **Multiple Data Types**: Support for various data types (int8, uint8, int16, uint16, int32, uint32, float, double)
- **Dynamic Field Removal**: Use `SerialFrame_RemoveAllFields()` to clear all fields and start fresh
- **Custom Field Positioning**: The library automatically calculates field positions, but they can be manually adjusted if needed
