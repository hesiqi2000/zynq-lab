# Software Writing for Timer and Debugging

## Objectives
After completing this lab, you will be able to:
*	Utilize the CPU’s private timer in polled mode
*	Use Vitis Debugger to set break points and view the content of variables and memory

## Steps

### Open the project in Vivado

1.	Open the lab4 project from the previous lab, and Save it as lab5 to the **{labs}** directory. Make sure that the **Create Project Subdirectory** and **Include run results** option is checked.

     Since we will be using the private timer of the CPU, which is always present, we don’t need to modify the hardware design.

2.	Open the Block Design. You may notice that the status changes to synthesis and implementation out-of-date as the project was saved as.   
     Since the bitstream is already generated and will be in the exported directory, we can safely ignore any warning about this.

3.	In Vivado, select **Tools > Launch Vitis IDE**

     A warning pop-up window indicating that the design is out of date. Since we have not made any changes, we can ignore this.

4.	Click Yes.

### Create an Application Project

1.	In the Explorer, right click on _lab4_system_ and select **Close System Project**
1.	Select **File > New > Application Project**.
1.  Select the platform created in lab4. Click Next
1.	Name the project **lab5**, click Next
1.  Select **standalone on ps7_cortexa9_0** as the domain. Click Next.
1.	Select **Empty Application(C)** and click Finish.
1.	Select **lab5 > src** in the Explorer, right-click, and select **Import Sources**.
1.	Browse to **{sources}\lab5**, click Select Folder.
1.  Select **lab5.c** and click finish.
1.  Build the project by clicking the hammer button.

    You will notice that there are multiple compilation errors. This is expected as the code is incomplete. You will complete the code in this lab.

###	Refer to the Scutimer API documentation

1.	Open **platform.spr** from the Explorer and click **Board Support Package**.
2.	Click on **Documentation link** corresponding to _scutimer_ (ps7_scutimer) driver under the Drivers section to open the documentation in a default browser window.  
3.	Click on the Files link to see available files related to the private timer API.
4.	Browse to the source directory, **{Xilinx installation}\Vitis\2021.2\data\embeddedsw\XilinxProcessorIPLib\drivers\scutimer_v2_3\src** and open xscutimer.h link to see various functions available.
Look at the *XScuTimer_LookupConfig( )* and *XScuTimer_CfgInitialize( )* API functions which must be called before the timer functionality can be accessed.

    Look at various functions available to interact with the timer hardware, including:

  <p align="center">
  <img src ="pics/lab5/1_usefulfn.jpg" width="70%" height="80%"/>
  </p>
  <p align="center">
  <img src ="pics/lab5/2_usefulfn.jpg" width="70%" height="80%"/>
  </p>
  <p align = "center">
  <i>Useful Functions</i>
  </p>

###	Correct the errors

1.	In Vitis IDE, in the **Problems** tab, double-click on the unknown type name x for the parse error. This will open the source file and bring you around to the error place.  

    <p align="center">
    <img src ="pics/lab5/3_fsterr.jpg" width="80%" height="80%"/>
    </p>
    <p align = "center">
    <i>First error</i>
    </p>

2.	Add the include statement at the top of the file for the **XScuTimer.h**. Re-build the project and the errors should disappear.

    ```C
    #include “xscutimer.h”
    ```

3.	Scroll down the file and notice that there are few lines intentionally left blank with some guiding comments.

     <p align="center">
     <img src ="pics/lab5/4_fillin.jpg" width="80%" height="80%"/>
     </p>
     <p align = "center">
     <i>Fill in Missing Code</i>
     </p>

    The timer needs to be initialized, the timer needs to be loaded with the value ONE_TENTH*dip_check_prev, AutoLoad needs to be enabled, and the timer needs to be started.

4.	Using the _API functions list_, fill those lines.  Save the file and correct errors if any. (Use the completed code further on in this workbook as a guide if necessary.)

    Functions needed:
    ```C
    XScuTimer_LookupConfig( )
    XScuTimer_CfgInitialize( )
    XScuTimer_LoadTimer( )
    XScuTimer_EnableAutoReload( )
    XScuTimer_Start( )
    ```
    
1. Scroll down the file further and notice that there are few more lines intentionally left blank with some guiding comments.


   <p align="center">
   <img src ="pics/lab5/5_morecd.jpg" width="80%"  height="80%"/>
   </p>
   <p align = "center">
   <i>More Code to be completed</i>
   </p>

   The value of **ONE_TENTH*dip_check** needs to be written to the timer to update the timer, the *InterruptStatus* needs to be cleared, and the LED needs to be written to, and the count variable incremented.

   Functions needed: 	
   ```C
   XScuTimer_LoadTimer( )
   XScuTimer_ClearInterruptStatus ( )
   LED_IP_mWriteReg ( )
   ```

6.Using the API functions list, complete those lines.  Save the file and correct errors if necessary.

```C
#include "xparameters.h"
#include "xgpio.h"
#include "led_ip.h"
// Include scutimer header file
#include "xscutimer.h"
//====================================================
XScuTimer Timer;		/* Cortex A9 SCU Private Timer Instance */

#define ONE_TENTH 32500000 // half of the CPU clock speed/10

int main (void) 
{

   XGpio dip, push;
   int psb_check, dip_check, dip_check_prev, count, Status;

   // PS Timer related definitions
   XScuTimer_Config *ConfigPtr;
   XScuTimer *TimerInstancePtr = &Timer;

   xil_printf("-- Start of the Program --\r\n");
 
   XGpio_Initialize(&dip, XPAR_SWITCHES_DEVICE_ID);
   XGpio_SetDataDirection(&dip, 1, 0xffffffff);
	
   XGpio_Initialize(&push, XPAR_BUTTONS_DEVICE_ID);
   XGpio_SetDataDirection(&push, 1, 0xffffffff);

   count = 0;
	
   // Initialize the timer
   ConfigPtr = XScuTimer_LookupConfig (XPAR_PS7_SCUTIMER_0_DEVICE_ID);
   Status = XScuTimer_CfgInitialize	(TimerInstancePtr, ConfigPtr, ConfigPtr->BaseAddr);
   if(Status != XST_SUCCESS){
	  xil_printf("Timer init() failed\r\n");
	  return XST_FAILURE;
   }
   // Read dip switch values
   dip_check_prev = XGpio_DiscreteRead(&dip, 1);
   // Load timer with delay in multiple of ONE_TENTH
   XScuTimer_LoadTimer(TimerInstancePtr, ONE_TENTH*dip_check_prev);
   // Set AutoLoad mode
   XScuTimer_EnableAutoReload(TimerInstancePtr);
   // Start the timer
   XScuTimer_Start (TimerInstancePtr);
   while (1)
   {
	  // Read push buttons and break the loop if Center button pressed
	  psb_check = XGpio_DiscreteRead(&push, 1);
	  if(psb_check > 0)
	  {
		  xil_printf("Push button pressed: Exiting\r\n");
		  XScuTimer_Stop(TimerInstancePtr);
		  break;
	  }
	  dip_check = XGpio_DiscreteRead(&dip, 1);
	  if (dip_check != dip_check_prev) {
		  xil_printf("DIP Switch Status %x, %x\r\n", dip_check_prev, dip_check);
		  dip_check_prev = dip_check;
	  	  // load timer with the new switch settings
		  XScuTimer_LoadTimer(TimerInstancePtr, ONE_TENTH*dip_check);
		  count = 0;
	  }
	  if(XScuTimer_IsExpired(TimerInstancePtr)) {
			  // clear status bit
		  	  XScuTimer_ClearInterruptStatus(TimerInstancePtr);
		  	  // output the count to LED and increment the count
		  	  LED_IP_mWriteReg(XPAR_LED_IP_S_AXI_BASEADDR, 0, count);
		  	  count++;
	  }
   }
   return 0;
}
```

### Verify Operation in Hardware

1.	Make sure that micro-USB cable(s) is(are) connected between the board and the PC. Change the boot mode to JTAG. Turn ON the power of the board.
1. Open the **Vitis Serial Terminal** and add a connection to the corresponding port.
2. Right-click on the **lab5** project in the Explorer and select **Run as > 1 Launch Hardware (Single Application Debug)**
3. Depending on the switch settings you will see LEDs implementing a binary counter with corresponding delay.
    <p align="center">
    <img src ="pics/lab5/7_termop.jpg" width="35%" height="80%"/>
    </p>
    <p align = "center">
    <i> Terminal window output </i>
    </p>

    Note: Setting the DIP switches and push buttons will change the results displayed.

    Flip the DIP switches and verify that the LEDs light with corresponding delay according to the switch settings. Also notice in the Terminal window, the previous and current switch settings are displayed whenever you flip switches.

### Launch Debugger


1.	Right-click on the **lab5** project in the Explorer and select **Debug as > 1 Launch Hardware (Single Application Debug)**. Click OK to relaunch the session if prompted.

3.	Double-click in the left margin to set a breakpoint on various lines in *lab5.c* shown below. A breakpoint has been set when a “tick” and blue circle appear in the left margin beside the line when the breakpoint was set. (The line numbers may be slightly different in your file.)

     The _first_ breakpoint is where count is initialized to 0.  The _second_ breakpoint is to catch if the timer initialization fails. The _third_ breakpoint is when the program is about to read the dip switch settings.  The _fourth_ breakpoint is when the program is about to terminate due to pressing of center push button. The _fifth_ breakpoint is when the timer has expired and about to write to LED.

    <p align="center">
    <img src ="pics/lab5/8_bp.jpg" width="80%"  height="80%"/>
    </p>
    <p align="center">
    <img src ="pics/lab5/9_bp.jpg" width="80%"  height="80%"/>
    </p>
    <p align = "center">
    <i>Setting breakpoints</i>
    </p>

4.	Click on the **Resume** button or press **F8** to continue executing the program up until the first breakpoint is reached.

    In the _Variables_ tab you will notice that the count variable may have value other than 0.
5.	Click on the **Step Over** button or press **F6** to execute one statement. As you do step over, you will notice that the count variable value changed to 0.
6.	Click on the **Resume** button again and you will see that several lines of the code are executed and the execution is suspended at the third breakpoint. The second breakpoint is skipped.  This is due to successful timer initialization.
7.	Click on the **Step Over (F6)** button to execute one statement. As you do step over, you will notice that the *dip_check_prev* variable value changed to a value depending on the switch settings on your board.
8.	Click on the memory tab.  If you do not see it, go to **Window > Show View > Memory**.
9.	Click the plus sign to add a **Memory Monitor**

    <p align="center">
    <img src ="/pics/lab5/cmemlocn.jpg" width="50%"  height="80%"/>
    </p>
    <p align = "center">
    <i>Monitor memory location</i>
    </p>

10.	Enter the address for the private counter load register (_0xF8F00600_), and click OK.

    <p align="center">
    <img src ="/pics/lab5/dmonitormem.jpg" width="30%"  height="80%"/>
    </p>
    <p align = "center">
    <i>Monitoring a Memory Address</i>
    </p>

    You can find the address by looking at the _xparameters.h_ file entry to get the base address (```# XPAR_PS7XPAR_PS7_SCUTIMER_0_BASEADDR1``` ), and find the load offset double-clicking on the xscutimer.h in the outline window followed by double-clicking on the *xscutimer_hw.h* and then selecting XSCUTIMER_LOAD_OFFSET.


11.	Make sure the DIP Switches are not set to “0000” and click on the **Step Over** button to execute one statement which will load the timer register.

    Notice that the address 0xF8F00604 has become red colored as the content has changed. Verify that the content is same as the value: dip_check_prev*32500000. You will see hexadecimal equivalent (displaying bytes in the order 0 -> 3).

    E.g. for dip_check_prev = 1; the value is 0x01EFE920; (reversed: 0x20E9EF01)
12.	Click on the **Resume** button to continue execution of the program. The program will stop at the writing to the LED port (skipping fourth breakpoint as center push button as has not occurred).

    Notice that the value of the counter register is changed from the previous one as the timer was started and the countdown had begun.
13.	Click on the **Step Over** button to execute one statement which will write to the LED port and which should turn OFF the LEDs as the count=0.
14.	Double-click on the _fifth_ breakpoint, the one that writes to the LED port, so the program can execute freely.
15.	Click on the **Resume** button to continue execution of the program. This time it will continuously run the program changing LED lit pattern at the switch setting rate.
16.	Flip the switches to change the delay and observe the effect.
17.	Press a push button and observe that the program suspends at the fourth breakpoint.  The timer register content as well as the **control register** (offset 0x08) is red as the counter value had changed and the control register value changed due to timer stop function call. (In the Memory monitor, you may need to right click on the address that is being monitored and click Reset to refresh the memory view.)
18.	Terminate the session by clicking on the **Terminate** button.
19.	Exit the Vitis and Vivado.
20.	Power **OFF** the board.

## Conclusion

This lab led you through developing software that utilized CPU’s private timer.  You studied the API documentation, used the appropriate function calls and achieved the desired functionality.  You verified the functionality in hardware. Additionally, you used the Vitis debugger to view the content of variables and memory, and stepped through various part of the code.
