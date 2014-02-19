====================================
The pyenergy and platformrun modules
====================================

Pyenergy
========

This python module can be used in two ways, to script the energy monitor, and
to configure the energy monitoring boards. This module is used by platformrun
to communicate with the energy monitor, and implements commands such as
setting up the energy monitoring, as well as pin triggering.

The energytool is also exposed by this module, allowing commandline access to
the energy monitors.

::

    $ energytool -h

    Energy tools

    Usage:
        energytool (-m MPOINT)... [options] read SERIAL PIN
        energytool (-m MPOINT)... [options] continuous SERIAL
        energytool (-m MPOINT)... [options] debug SERIAL
        energytool list
        energytool changeserial SERIAL NEWSERIAL

    Commands:
        read            This sets up a trigger on the specified PIN and waits for
                        an energy measurement.

        continuous      Continuously read measurements from the specified energy
                        monitor.

        debug           Output some debug data about the instantaneous voltages
                        seen on the ADCs, along with current and voltage.

        list            Show the serial numbers and API version of each connected
                        energy monitor.

        changeserial    Connect to the device specified by SERIAL, and change the
                        serial to NEWSERIAL


Scripting
---------

The module exposes one class, EnergyMonitor. This class can be instantiated as such::

    em = pyenergy.EnergyMonitor("EE00")

Up to three of the four measurement points can then be enabled::

    em.enableMeasurementPoint(1)

Measurement can be started and stopped in a similar way::

    em.start(1)
    em.stop(1)

This allows the measurement to be manually started and stopped. A pin to automatically start and stop measurements can be set up too. Note the ``start`` call is not needed for this, as the device will automatically start. The device can then be polled for a completed measurement::

    em.setTrigger("PA0", 1)

    while not em.measurementCompleted(1):
        sleep(0.1)

    m = em.getMeasurement(1)

The above snippet sets a trigger on PA0. The returned measurement has several properties with metrics about the run::

    m.energy        # Energy, joules
    m.time          # Time between start and stop
    m.avg_power     # Average power throughout the run
    m.avg_current   # Average current throughout the run
    m.avg_voltage   # Average voltage throughout the run

Platform run
============

These scripts form the basis to easily run binaries on a range of platforms.
For example, to run a binary on the STM32F0DISCOVERY board, this is simply the
following command line invocation::

    $ platformrun stm32f0discovery program.elf
    Energy:            27.386 mJ
    Time:             626.119 ms
    Average current:   14.382 mA
    Average voltage:    3.041 V

The compilation of programs to run on the target architecture is out of scope
for this document. The only particular feature a program must have, is that it
asserts a digital 1 on a pin when it wants the measurement to start, and
returns to 0V when the measurement should end.

Two configuration files are needed by platformrun, one describing the set of
up the energy monitors, and one describing where platformrun can find all the
tools necessary to run on a specific platform. The first file must be created
manually - it is specific to the way you connect the boards, and the serial
numbers given to the energy monitors.

A detection script is provided to find locations of the tools necessary to run
binaries on a platform. The configuration of the location of tools, and
enabled platforms is kept in ``~/.platformrunrc``.

The configuration file is set up as follows::

    $ platformrun-detect

    Summary

    pic32mx250f128b           disabled
    stm32vldiscovery          enabled
    atmega328p                enabled
    stm32f0discovery          enabled
    msp-exp430fr5739          enabled
    msp-exp430f5529           enabled

This will try to automatically detect where the required executables are.

List of platforms
-----------------

Currently, several different platforms are set up to easily have programs run
via platform run.

STM32F0DISCOVERY
~~~~~~~~~~~~~~~~

This board contains a cortex-m0. Stlink is used as a gdb-server and an arm
debugger is used to flash the code to the board.

Platform name: ``stm32f0discovery``

Prerequisites for running:
 - st-util (st-link package)
 - arm-none-eabi-gdb (or other arm gdb-compatible debugger)


ATMEGA328P
~~~~~~~~~~

This is an ATMEGA328P chip, loaded with an arduino compatible bootloader. This
is programmed with a USB to serial converter and avrdude. This was tested on a
breadboarded ATMEGA328P chip, with a FTDI USB to serial converter.

Platform name: ``atmega328p``

Prerequisites for running:
 - avrdude
 - avr-objcopy


PIC32MX250F128B
~~~~~~~~~~~~~~~

This is a 32-bit pic chip, tested on a breadboard and hooked up to a pickit2.
This requires pic32prog (so that no bootloader is needed on the PIC chip) and
the pic32 tools to convert the elf to a hex file.

Platform name: ``pic32mx250f128b``

Prerequisites for running:
 - pic32prog
 - pic32-objcopy


Measurement configuration
-------------------------

The measurement configuration file tells platformrun about the energy monitors
connectted to the platforms, and some additional details about the platforms.
For example, to measure the stm32f0discovery, the application needs to know the
serial number of the energy measurement device, as well as the measurement
point and the shunt resistor value. Other platforms have additional details,
such as atmega328p, which needs to know the ID of the USB-serial adaptor.

The measurement config is a standard JSON format file, and by default is
loaded from ``~/.measurementrc``. An example of the measurement configuration
for the stm32f0discovery platform is given below.

::

    {
        "stm32f0discovery" : {
            "energy-monitor" : "CXM0",
            "trigger-pin" : "PA0",
            "measurement-point" : 1,
            "resistor" : 1
        }
    }


Basic configuration
~~~~~~~~~~~~~~~~~~~

Keys:
 - energy-monitor. This specifies the serial number of the energy monitor that
   is used for this platform.
 - trigger-pin.  This specifies the pin on which the platform will trigger the
   energy monitor.
 - measurement-point. Which measurement point is connected to the platform.
 - resistor. The value of the shunt resistor that intercept's the platform's
   power supply.

Platform specific keys
~~~~~~~~~~~~~~~~~~~~~~

Keys for ``atmega328p``:
 - serial-dev. This specifies the ID of the USB to serial device. This ID comes
   directly from the link found in /dev/serial/by-id/ when the adapter is
   plugged in. By selecting the ID this way, multiple similar USB-serial
   adapters can be uniquely specified.

Keys for ``pic32mx250f128b``:
 - serial-number. This specifies the serial number of the pickit2 connected to
   the platform (not currently used).
