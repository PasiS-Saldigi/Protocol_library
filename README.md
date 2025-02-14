# Protocol_library
LabVIEW Protocol library: Boundary Scan JTAG + I2C + SPI + Others

**Operating system requirements:**

Microsoft Windows. Developed on 11, most propably works on previous versions.

**Software requirements:**

LabVIEW development environment 2017 or later.

**Compatible hardware (HAL):** 

	DAQmx V1:	6535/6536/6537 (5-25MHz bus clock) 6321/6361/6363 and so on
	DAQmx V2:	6321/6361/6363 and so on (PCIe 0.5-5MHz bus clock, USB 0.5MHz)
			NI/Emerson hardware requires "Digital Waveform" and DAQmx support, multifunction DAQs must be X-series
			NI/Emerson hardware requires NI-DAQmx drivers, version used during development: ni-daqmx_20.7.0

	FTn232H: 	FTDI devices FT2232H or FT232H (3k7 - 10MHz bus clock)

	J-LINK BSCAN: 	Segger J-Link debug probes (max 15MHz bus clock, but slow on BSCAN V2)

	
**Protocol library description:**
--------------------------------------------------------------------------------------
This protocol library is distributed as packed library (lvlibp). The block diagrams are only partially visible to help to understand functionality and debugging. Without a paid license the physical bus update rate is locked to 10000 which correlates to 5kHz bus clock speed. This enables full use of the library to test it and use it for applications where speed is not a concern.

IDE (Integrated Development Environment) for this library is LabVIEW. You can create your test sequencing 100% in native labview. With BSCAN V1 / I2C / SPI1 the target MCU/chip state is synchronous to LabVIEW execution, e.g. breakpoints. Physical bus clock stops after a VI is executed, then you can alter your code and execute it straight away again.

This protocol library is on beta-testing phase. It is coded 100% on native LabVIEW. It includes multiple protocols with multiple bus master possibilities. See table below:

        	       | FTDI | DAQmx | J-Link | BSCAN V1 | BSCAN V2 | Note
	I2C            |  x   |   x   |        |    x     |    x     | (SCK, SDA-out, SDA-in)
	SPI1           |  x   |   x   |        |    x     |    x     | (MISO/MOSI,-CS, CLK)
 	BSCAN V1       |  x   |   x   |   x    |          |          | (V1 = Sequential code execution)
	BSCAN V2       |  x   |   x   |   x    |          |          | (V2 = Parallel code execution)
 	GPIO           |      |       |        |    x     |    x     |

From the table we can see that I2C and SPI1 can be driven directly to physical bus OR via BitBanging over BSCAN (Boundary Scan).
I2C requires hardware conditioning when driven directly to physical bus, since SDA-out/in are separate physical pins.

DAQmx V1 also requires PFI5 channel for all protocols input data clocking. PFI5 also enables longer cables between DAQmx-board and target MCU.

GETTING STARTED:
----------------
	- Your target MCU must be bootstrapped or configured so it is in JTAG Boundary Scan mode.
 	
 	- Obtain FT232H hardware with 3V3 logic compatible target is most simple
	- Obtain FT232H serial number with "FTDI_Get_Device_Serial_Number_By_Index.vi"
 	- Segger J-Link is also very easy to start use but tested only with V6.98e jlinkarm.dll
 
	- Download correct BSDL file(s) for your target MCU
	- Open example VI and check the parameters and their help-texts: "BSCAN_DEMO00_init_xc7z020_clg400.vi"
 	- Modify the example VI to match your target. Also modify the block diagram UserConfig for safe states!
	  Usually BSDL files configure all pins as inputs.
  	- Run the example VI and check that the "ALL TAPs IDCODE OK" boolean is true.
   
   	- Next you can open and modify "BSCAN_DEMO01_GPIO_writeRead.vi" to match your target and toggle GPIO output states.


**Boundary Scan BSCAN V1 and V2:**
--------------------------------------------------------------------------------------
There exists two versions of execution for the Boundary Scan (BSCAN): V1 and V2.

V1 uses sequential principle where one bitstream (VI code) is generated and then driven to BSCAN physical bus. If multiple VI are run same time the BSCAN state is erroneous. Sequential V1 ensures physical bus clock 50/50 duty cycle, no "clock stretching".

In this context bitstream stands for predefined binary states of pins that are sequentially written/read to/from physical bus. After the whole bitstream has made the physical bus write-read round-trip, the read bistream payload/protocol frame is encoded out.

V2 uses parallel principle where multiple bitstreams can be generated and driven in parallel to physical pins. V2 server also achieves faster write-read turn-around times on longer bitstreams. V2 bitstream execution consists of following states:

 	1: Client VI generates bitstream (GPIO, I2C, SPI, DDR3 or similar)
	2: Client VI writes bitstream to V2 Server queue
	3: V2 server encodes the data to BSCAN Chain
	4: V2 server writes the BSCAN Chain to physical bus
	5: V2 server reads the BSCAN Chain from physical bus
	6: V2 server decodes Client VI bitstream and returns
	7: Client VI decodes out payload/protocol frame from the returned bitstream

	States 3,4,5 and 6 are executed in parallel threads.

V2 does not ensure clock duty-cycle because physical bus is written in buffered bursts. If Operating System does not give enough processor time for state 3: encode, then writing to physical bus has "clock stretching". This is not a problem with BSCAN but e.g. some I2C peripherals might be affected.

Example of a stretched clock signal:

	/⎺⎺\_/⎺⎺\_/⎺⎺\_/⎺⎺\_____________________/⎺⎺\_/⎺⎺\_/⎺⎺\_/⎺⎺\_
				strecthing		 


V1/V2 execution exceptions to note:
-----------------------------------
Do not mix these two execution methods. Use one of following methods:

	1: INIT					:OK	(INIT loads BSDL safe-values and initializes target MCU, run this to clear any state)
 	2: INIT > V1 > V1 ...   		:OK	(V1 always stops automatically and previous BSCAN state is known)
	3: INIT > V2 > STOP			:OK


Below are examples of erroneous/ok executions:

	V2 > STOP > V1 > 			:ERROR 	(must INIT in-between mode change)
	V1 > V2		 			:ERROR
	INIT > V2 > STOP > INIT > V1 		:OK
	INIT > V1 > INIT > V2			:OK	(V1 always stops automatically)
	INIT > V2 > STOP > V2 			:ERROR 	(V2 exit state is not always known due to parallel threads exit at different times)
	INIT > V2 > STOP > INIT > V2 		:OK



Hardware handle (INIT) open/close exceptions to note:
-----------------------------------------------------

	VI open > FTDI handle open > VI close 	:ERROR 	(When all VIs are closed then also FTDI Global is unloaded and handle lost)
	VI open > DAQmx handle open > VI close 	:OK
	VI open > J-Link handle open > VI close :OK 




   :END 
