# jtag2updi

This fork of the repo has been stripped down and modified so that folders don't need to be renamed in order to compile with the Arduino IDE.

This is a firmware, that when uploaded on an atmega328p, or a similar AVR MCU, enables it to interface with avrdude using the jtagice Mk2 protocol via a serial link. In particular, you can use an Arduino to host this firmware.
It provides a bridge to program the new attiny817 family of MCUs, that use the UPDI interface:

avrdude -> HW Serial interface -> Programmer MCU (e.g. Mega328P) -> SW Serial on PD6 -> Target MCU (e.g. tiny817)

Currently, I have not tested this software with a level shifter, however, since the UPDI pin is high voltage tolerant, it's ok to have V_prog > V_target, but not the reverse.

Notice, however, that the logic levels need to be compatible for successful programming: V_target cannot be lower than about 60% of V_prog (60% will likelly work, 70% is guaranteed to work). Therefore, it will not be possible to program a 2.5V target with a 5.0V programmer, because communication errors will surely occur (but no electrical damage), but if V_target is 3.3V (66% of 5.0V) chances are good.

<pre>
                                            V_prog                 V_target
                                              +-+                     +-+
                                               |                       |
 +----------+          +---------------------+ |                       | +--------------------+
 | PC       |          | Programmer          +-+                       +-+  Target            |
 | avrdude  |          |                     |      +----------+         |                    |
 |       TX +----------+ RX              PD6 +------+   4k7    +---------+ UPDI               |
 |          |          |                     |      +----------+         |                    |
 |       RX +----------+ TX                  |                           |                    |
 |          |          |                     |                           |                    |
 |          |          |                     |                           |                    |
 |          |          |                     +--+                     +--+                    |
 +----------+          +---------------------+  |                     |  +--------------------+
             JTAGICE MkII                      +-+     UPDI          +-+
             Protocol                          GND     Protocol      GND

</pre>
Drawing adapted from: https://github.com/mraardvark/pyupdi

If you use an Arduino as host for this program, be sure that, after burning the software, you disable its auto-reset feature, using one of the techniques described here:
https://playground.arduino.cc/Main/DisablingAutoResetOnSerialConnection

Alternatively, you can use an Arduino without integrated USB/serial adapter, like the pro-mini; in that case, just disconecting the DTR wire will disable the auto-reset. Just remember the UPDI chip must be connected to the same supply voltage as the Arduino's MCU!


## Building with Arduino IDE

The program can be built as if it was an Arduino sketch. 

Open jtag2updi.ino in the jtag2updi folder, and upload to your microcontroller. 

The Arduino IDE will automatically set the correct MCU model and F_CPU, but if you want to change the speed of the UPDI link, you will have to edit UPDI_BAUD directly in the source code.


## Using with avrdude

**If using megaTinyCore, you can ignore all this stuff**

You will find a modified avrdude.conf file in the base folder. This is based on the current avrdude.conf file from:
http://svn.savannah.gnu.org/viewvc/*checkout*/avrdude/trunk/avrdude/avrdude.conf.in?revision=1422

It has been modified to work with avrdude 6.3, by removing (actually, commenting out) some incompatible stuff, and adding the "jtag2updi" programmer type.

The definitions for UPDI chips were slightly modified so that avrdude thinks they use the PDI programming interface instead of UPDI (i.e., avrdude thinks they are some kind of XMegas).

This allows the jtagice mk2 protocol to be used for programming UPDI chips, since this protocol predates UPDI and is not formally compatible with it. Originally, I had planed to use the STK500v2 protocol, and emulate the ISP interface, and I actually wrote an ISP version of the programmer software.

However, this would require entirely new definitions for the UPDI chips inside the avrdude.conf file, while using jtagice2 requires only very slight changes to the definions provided by Atmel (now Microchip).

Jtagice mk2 is the most advanced of Atmel's programming protocols that still supports a UART serial connection instead of USB, making it easily compatible with any Arduino you choose to host this software, and any OS you run avrdude on.

It's major limitation is speed; it can't go over 115200 Baud, because the protocol lacks definitions for higher speeds. It's actually inferior to the STK500v2 protocol in this respect, this older standard can run at any speed avrdude instructs it to.

Fortunately, the current UPDI chips do not have very large flash memories, so I think this isn't a major issue.

<b>Example command line (windows):</b>
<pre>
avrdude -c jtag2updi -P com7 -p t1614
</pre>

If all the connections are correct and the target is indeed an *unlocked* tiny1614, the output will be:
<pre>
avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.03s

avrdude: Device signature = 0x1e9422 (probably t1614)

avrdude done.  Thank you.
</pre>

If the chip is locked, the output will be:
<pre>
avrdude: jtagmkII_reset(): bad response to reset command: RSP_ILLEGAL_MCU_STATE
avrdude: initialization failed, rc=-1
         Double check connections and try again, or use -F to override
         this check.

avrdude: jtagmkII_close(): bad response to sign-off command: RSP_ILLEGAL_MCU_STATE

avrdude done.  Thank you.
</pre>

To unlock the chip, you need to erase it. Currently, the "-e" option is not working with jtag2updi, let's call it a known bug, so you need to enter interactive mode, using "-t", and "-F" to override the error:
<pre>
avrdude -c jtag2updi -P com7 -p t1614 -U flash -t -F
</pre>

You will enter the avrdude prompt:
<pre>
avrdude: jtagmkII_reset(): bad response to reset command: RSP_ILLEGAL_MCU_STATE
avrdude: initialization failed, rc=-1
avrdude: AVR device initialized and ready to accept instructions
avrdude: Device signature = 0xffff00
avrdude: Expected signature for ATtiny1614 is 1E 94 22
avrdude: NOTE: Programmer supports page erase for Xmega devices.
         Each page will be erased before programming it, but no chip erase is performed.
         To disable page erases, specify the -D option; for a chip-erase, use the -e option.
avrdude>
</pre>

Enter "erase" then "quit" and the chip will be unlocked (and erased).
<pre>
avrdude> erase
>>> erase
avrdude: erasing chip
avrdude> quit
>>> quit

avrdude done.  Thank you.
</pre>


## Troubleshooting

If you have triple-checked all the connections but still getting errors, the problem might be the speed of the serial links. I have set the jtag2updi entry on the avrdude configuration file to run at 115200 baud by default. This baud rate can cause errors, for example, if your MCU is running at 8MHz.

This can be changed with the avrdude "-b" option. Valid baud rates are 2400, 4800, 9600, 14400, 19200, 38400, 57600 and 115200. You can make the setting permanent by editing the jtag2updi entry on "avrdude.conf".

If the trouble is on the UPDI link, a slower speed can be selected by changing UPDI_BAUD and recompiling.
## Tutorial youtube video

bitluni's lab made a youtube tutorial for this package https://www.youtube.com/watch?v=AL9vK_xMt4E
