# README-RPi

This README covers details of deploying `rsked` on a Raspberry Pi.
`rsked` has been tested with a Pi 3B+. Other models may work; your
choice will depend on your network requirements (if any) and whether
you indend to use the SDR player (radio). A minimal application 
*might* run on a Pi Zero W. Note that less capable models may have
difficulty with the `gqrx` software defined radio application.

## Operating System

`rsked` has been tested with Raspbian Linux **10** (Buster). It might
work with other versions. 


## Time

`rsked` needs to know the time of day to know when to play the
programs in its schedule.  Without a battery, each power cycling of
the Pi (intended or unintended) will leave the system clueless as to
the correct time.  The default Raspbian approach to time
synchronization is to use  the network time protocol. This works well
*except* if the Pi will not always connected to a network with a working
time server.

The best solution is a battery-backed real-time clock (RTC).
The *Adafruit PiRTC-PCF8523* is an inexpensive option, and is easy to
interface.  For systems that will normally never be connected to an
internet time source, consider the more stable 
*Adafruit DS3231 Precision RTC*.

After installing one of these modules, it is necessary to change
`/boot/config.txt` to enable i2c, and to load the right kernel module
for the RTC.  See further instructions at
[Raspberry Pi RTC](https://pimylifeup.com/raspberry-pi-rtc/).


## Audio Output

Native RPi audio output is "a compromise" ill suited for high
fidelity.  In addition to obtaining reasonable sounding powered
speakers, attach a USB audio adapter to the RPi that is known to be Linux
compatible. Even inexpensive devices can deliver high quality stereo
output,  e.g. the Antlion Audio USB Sound Card. Then:

- Set the USB audio device to be the Linux default
- Disable the built-in audio.

In the file `/boot/config.txt` one can disable the unused built-in
audio with the line `dtparam=audio=off`, replacing the `audio=on`
setting. A reboot will be necessary.

The utility command `aplay -L` will list the audio hardware and the
current default. With a source playing, adjust the master volume
control to achieve a level suitable for your hardware. Levels that are
too high will cause audible distortion. Your choice of powered speakers will
affect this setting. For example to set playback to 50%:

```
amixer set Master playback 50%
```

You may wish to execute this command at the start of each session.

## Networking

If you intend to use internet streaming, consider whether you
need wired or wireless access. Select a model with integral Wi-Fi
in latter case.

In an embedded solution, the network and WEP credentials to use
Wi-Fi must be added to the device prior to installation. Add
stanzas to the file `/etc/wpa_supplicant/wpa_supplicant.conf`
e.g. 

```
network={
	ssid="FrobozGuest"
	psk=e8657d35e525a0ad.....4f0732a8a8032a9cacce93fbf6a2
}
```

The preshared key (psk) hash may be generated with `wpa_passphrase`. See
[Wireless-CLI](https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md)
for more details.

Consider adding a *firewall* package like `ufw`. Configure it to
open only ports needed for the installation.

## FM Radio

For FM demodulation with `gqrx`, one needs at least a Pi 3.  Less
capable models will be unsatisfactory.  Consult the `gqrx` development
site for progress with other models such as the Pi 4.  The `gqrx`
configuration should be tailored to minimize the display features,
e.g. the spectrum waterfall--none of these are needed here.


## Storage

Use a MicroSD card with sufficient capacity to store all of the
recorded music you intend to deploy.   It should also have
room for the development tools if you intend to compile all of the
software on the target system.
A 16GB card should be sufficient for most applications.


## Cooling

Even with passive heat sinks, the RPi may run hot when doing FM
reception via `gqrx` SDR.  An inexpensive 5-volt cooling fan can make
a huge difference.  Applications that are *not using SDR* can get by
fine with passive cooling.  Heat sinks should be used even if active
cooling is installed.

There are various cooling fan selection guides
in the RPi community. In this embedded application, the noise
generated by the fan is a real consideration.  The
[Noctua](https://noctua.at/en/products/fan/nf-a4x10-5v)
`NF-A4x10 5V` is a notably quiet one.

Another way to limit noise is to only run the fan when it is needed.
The `cooling` application in this repository can turn on the fan when 
the core temperature is high, and turn it off when it is cool enough.
It does this by toggling a configurable GPIO line.
Because GPIO provides only very low current, that line should never be directly
connected to the fan motor. Instead, use a driver transistor
as shown in the accompanying schematic...


## Autostart

In an embedded application, `rsked` should be started as soon as the
system boots.  An easy way to achieve this is to configure the default
user (pi) to login automatically on boot, and to start things with via
the desktop mechanism. (However read the section [Screen](#Screen) below!)
To add a user custom autostart, use:

-  `/home/pi/.config/lxsession/LXDE-pi/autostart`

Add a line with the program name prefixed by '@' and this program will
be automatically started, and will be restarted if it crashes. (It
will not be restarted if it exits without error.)  Included in
`scripts/` is a shell script `startup_rsked` that will initialize the
environment and start the `cooling` application.

```
@/home/pi/bin/startup_rsked
```

Other startup mechanisms are possible--feel free to experiment.

## Screen

While `rsked` is designed to run "headless" (no monitor), the `gqrx`
application, written with a Qt5 interface, expects a display. 
Also, the LXDE autostart mechanism depends on the presence of a display.
Thus you will need to provide some sort of screen.
Possible solutions here include:

1. attach an small HDMI display
2. attach an HDMI "pacifier" (plug that fakes a monitor)
3. create a virtual screen via **Xvfb**

Option 1 is useful for debugging, but option 3 is cheap and easy.
Use Option 3 for embedded deployment.

```
sudo apt install xvfb
```

Change `/etc/lightdm/lightdm.conf` to use this xserver-command:
```
[SeatDefaults]
...
xserver-command=/etc/X11/xinit/xserverrc
```

In `/etc/X11/xinit/xserverrc` change the server to start
`Xvfb` like this:


```
 #!/bin/bash
 exec Xvfb :0 -screen 0 1280x1024x24
```


## Power

Use an approved power supply with a tightly mating connector.
For an Pi-3B+, it should provide at least 2.5A.

If you attach too many external devices to an RPi, it risks under-voltage
operation.  A message to this effect may appear on an HDMI attached screen,
but the RPi also might simply reboot.


## GPIO

The GPIO pins of the RPi may be wired to control a cooling fan
and status LEDs, or to monitor buttons.
Pins to use are configurable in the `cooling.json` file.
The pin mapping used in the `rsked` prototypes is:

- 4 : OUT fan motor 
- 17 : OUT green LED
- 18 : IN snooze pushbutton
- 27 : OUT red LED

**WARNING**: GPIO pins can provide only minimal drive current (OUT) and can
tolerate only a limited range of voltages (IN). Use suitable buffer
circuitry to protect the RPi. An example schematic is shown below:

![hardware]

[hardware]: hardware.png


**Permissions** The operating user for cooling must have permissions
to read and write to the GPIO devices. One way to achieve this is to
add `udev` rules.  An example suitable for RPi is given in the file
`cooling/99-gpio.rules`.  You must manually install this file (as root)
in `/etc/udev/rules.d/`; correct permissions should be applied
automatically on the next boot. The rules assume the target user, e.g.
`pi`, is a member of the Linux group `gpio`; add the user to the
group if necessary.

**Initialization** The GPIO configuration is not always as desired on
power-up.  To normalize this, an included Python script,
`gpiopost.py`, will force the relevant lines to a known state. It will
also strobe the outputs, which provides a quick power-on test of the
hardware. This is invoked automatically by included shell script
`startup_rsked`.  If you use a different GPIO mapping, be sure to
adjust `gpiopost.py`.
