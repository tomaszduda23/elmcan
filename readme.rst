Linux SocketCAN driver for ELM327
==================================

Authors
--------

Max Staudt <max-linux@enpas.org>



Motivation
-----------

This driver aims to lower the initial cost for hackers interested in
working with CAN buses.

CAN adapters are expensive, few, and far between.
ELM327 interfaces are cheap and plentiful.
Let's use ELM327s as CAN adapters.



Introduction
-------------

This driver is an effort to turn abundant ELM327 based OBD interfaces
into full-fledged (as far as possible) CAN interfaces.

Since the ELM327 was never meant to be a stand-alone CAN controller,
the driver has to switch between its modes as quickly as possible in
order to fake full-duplex operation.

As such, elmcan is a best-effort driver. However, this is more than
enough to implement simple request-response protocols (such as OBD II),
and to monitor broadcast messages on a bus (such as in a vehicle).

Most ELM327s come as nondescript serial devices, attached via USB or
Bluetooth. The driver cannot recognize them by itself, and as such it
is up to the user to attach it in form of a TTY line discipline
(similar to PPP, SLIP, slcan, ...).

This driver is meant for ELM327 versions 1.4b and up, see below for
known limitations in older controllers and clones.



Requirements
-------------

This requires Linux 4.11 (for 431af779256c), and has been tested on 4.19.

Also, elmcan depends on ``can-dev``:

    sudo modprobe can-dev

Install
-------------
    cd module/
    
    sudo dkms install `pwd`

Data sheet
-----------

The official data sheets can be found at ELM electronics' home page:

  https://www.elmelectronics.com/



How to check the controller version
------------------------------------

Use a terminal program to attach to the controller.
The default settings are 38400 baud/s, 8 data bits, no parity, 1 stopbit.

After issuing the "``AT WS``" command, the controller will respond with
its version::

    >AT WS


    ELM327 v1.4b

    >

Note that clones may claim to be any version they like.
It is not indicative of their actual feature set.



How to attach the line discipline
----------------------------------

Every ELM327 chip is factory programmed to operate at a serial setting
of 38400 baud/s, 8 data bits, no parity, 1 stopbit.

The line discipline can be attached on a command prompt as follows::

    sudo ldattach \
           --debug \
           --speed 38400 \
           --eightbits \
           --noparity \
           --onestopbit \
           --iflag -ICRNL,INLCR,-IXOFF \
           29 \
           /dev/ttyUSB0

To change the ELM327's serial settings, please refer to its data
sheet. This needs to be done before attaching the line discipline.



Module parameters
------------------

- ``accept_flaky_uart`` - Try to live with a bad controller or UART line

	Some adapters and/or their connection are unreliable. Enable this
	option to try and work around the situation. This is a best-effort
	workaround, so undefined behavior will likely occur sooner or later.
	Without this option, the driver will bail on the first unrecognized
	character it receives from the TTY.

	LOAD TIME::

		module/kernel parameter: accept_flaky_uart=[0|1]



Known limitations of the controller
------------------------------------

- Clone devices ("v1.5" and others)

  Sending RTR frames is not supported and will be dropped silently.

  Receiving RTR with DLC 8 will appear to be a regular frame with
  the last received frame's DLC and payload.

  "``AT CSM``" not supported, thus no ACK-ing frames while listening:
  "``AT MA``" will always be silent. However, immediately after
  sending a frame, the ELM327 will be in "receive reply" mode, in
  which it *does* ACK any received frames. Once the bus goes silent
  or an error occurs (such as BUFFER FULL), the ELM327 will end reply
  reception mode on its own and elmcan will fall back to "``AT MA``"
  in order to keep monitoring the bus.


- All versions

  No full duplex operation is supported. The driver will switch
  between input/output mode as quickly as possible.

  The length of outgoing RTR frames cannot be set. In fact, some
  clones (tested with one identifying as "``v1.5``") are unable to
  send RTR frames at all.

  We don't have a way to get real-time notifications on CAN errors.
  While there is a command (``AT CS``) to retrieve some basic stats,
  we don't poll it as it would force us to interrupt reception mode.


- Versions prior to 1.4b

  These versions do not send CAN ACKs when in monitoring mode (AT MA).
  However, they do send ACKs while waiting for a reply immediately
  after sending a frame. The driver maximizes this time to make the
  controller as useful as possible.

  Starting with version 1.4b, the ELM327 supports the "``AT CSM``"
  command, and the "listen-only" CAN option will take effect.


- Versions prior to 1.4

  These chips do not support the "``AT PB``" command, and thus cannot
  change bitrate or SFF/EFF mode on-the-fly. This will have to be
  programmed by the user before attaching the line discipline. See the
  data sheet for details.


- Versions prior to 1.3

  These chips cannot be used at all with elmcan. They do not support
  the "``AT D1``", which is necessary to avoid parsing conflicts on
  incoming data, as well as distinction of RTR frame lengths.

  Specifically, this allows for easy distinction of SFF and EFF
  frames, and to check whether frames are complete. While it is possible
  to deduce the type and length from the length of the line the ELM327
  sends us, this method fails when the ELM327's UART output buffer
  overruns. It may abort sending in the middle of the line, which will
  then be mistaken for something else.



Known limitations of the driver
--------------------------------

- No 8/7 timing.

  ELM327 can only set CAN bitrates that are of the form 500000/n, where
  n is an integer divisor.
  However there is an exception: With a separate flag, it may set the
  speed to be 8/7 of the speed indicated by the divisor.
  This mode is not currently implemented.

- No evaluation of command responses.

  The ELM327 will reply with OK when a command is understood, and with ?
  when it is not. The driver does not currently check this, and simply
  assumes that the chip understands every command.
  The driver is built such that functionality degrades gracefully
  nevertheless. See the section on known limitations of the controller.

- No use of hardware CAN ID filtering

  An ELM327's UART sending buffer will easily overflow on heavy CAN bus
  load, resulting in the "``BUFFER FULL``" message. Using the hardware
  filters available through "``AT CF xxx``" and "``AT CM xxx``" would be
  helpful here, however SocketCAN does not currently provide a facility
  to make use of such hardware features.



Communication example
----------------------

This is a short and incomplete introduction on how to talk to an ELM327.


The ELM327 has two modes:

- Command mode
- Reception mode

In command mode, it expects one command per line, terminated by CR.
By default, the prompt is a "``>``", after which a command can be
entered::

    >ATE1
    OK
    >

The init script in the driver switches off several configuration options
that are only meaningful in the original OBD scenario the chip is meant
for, and are actually a hindrance for elmcan.


When a command is not recognized, such as by an older version of the
ELM327, a question mark is printed as a response instead of OK::

    >ATUNKNOWN
    ?
    >

At present, elmcan does not evaluate this response and silently assumes
that all commands are recognized. It is structured such that it will
degrade gracefully when a command is unknown. See the sections above on
known limitations for details.


When a CAN frame is to be sent, the target address is configured, after
which the frame is sent as a command that consists of the data's hex
dump::

    >ATSH123
    OK
    >DEADBEEF12345678
    OK
    >

The above interaction sends the frame "``DE AD BE EF 12 34 56 78``" with
the 11 bit CAN ID ``0x123``.
For this to function, the controller must be configured for 11 bit CAN
ID sending mode (using "``AT PB``", see code or datasheet).


Once a frame has been sent and wait-for-reply mode is on (``ATR1``,
configured on ``listen-only=off``), or when the reply timeout expires and
the driver sets the controller into monitoring mode (``ATMA``), the ELM327
will send one line for each received CAN frame, consisting of CAN ID,
DLC, and data::

    123 8 DEADBEEF12345678

For 29 bit CAN frames, the address format is slightly different, which
elmcan uses to tell the two apart::

    12 34 56 78 8 DEADBEEF12345678

The ELM327 will receive both 11 and 29 bit frames - the current CAN
config (``ATPB``) does not matter.


If the ELM327's internal UART sending buffer runs full, it will abort
the monitoring mode, print "BUFFER FULL" and drop back into command
mode. Note that in this case, unlike with other error messages, the
error message may appear on the same line as the last (usually
incomplete) data frame::

    12 34 56 78 8 DEADBEEF123 BUFFER FULL



Rationale behind the chosen configuration
------------------------------------------

``AT E1``
  Echo on

  We need this to be able to get a prompt reliably.

``AT S1``
  Spaces on

  We need this to distinguish 11/29 bit CAN addresses received.

  Note:
  We can usually do this using the line length (odd/even),
  but this fails if the line is not transmitted fully to
  the host (BUFFER FULL).

``AT D1``
  DLC on

  We need this to tell the "length" of RTR frames.



A note on CAN bus termination
------------------------------

Your adapter may have resistors soldered in which are meant to terminate
the bus. This is correct when it is plugged into a OBD-II socket, but
not helpful when trying to tap into the middle of an existing CAN bus.

If communications don't work with the adapter connected, check for the
termination resistors on its PCB and try removing them.



Thanks
-------

Thanks go out to Oliver Neukum for his early reviews and suggestions.

Several more people have encouraged me to finish this - thank you all.



To Do list for future development
----------------------------------

- None currently
