[Light Controller v1.1](https://github.com/brain1011/light-controller)
======================

Monitor and control your Light from the web via a Raspberry Pi.

![Screenshot from the controller app][1] &nbsp; ![Screenshot from the controller app][2]

Overview
---------

This project provides software and hardware installation instructions for monitoring and controlling your Light remotely (via the web or a smart phone). The software is designed to run on a [Raspberry Pi](www.raspberrypi.org), which is an inexpensive ARM-based computer, and supports:

* Monitoring of the state of the garage doors (indicating whether they are open, closed, opening, or closing)
* Remote control of the garage doors
* Timestamp of last state change for each door
* Logging of all garage door activity

Requirements
-----

**Hardware**

* Raspberry Pi
* Relay Module, 8 channel
* 8-Koppelrelais

**Software**

* [Raspbian](http://www.raspbian.org/)
* Python >2.7 (installed with Raspbian)
* Raspberry Pi GPIO Python libs (installed with Raspbian)
* Python Twisted web module

Hardware Setup
------

*Step 1: Install the magnetic contact switches:*

The contact switches are the sensors that the raspberry pi will use to recognize whether the garage doors are open or shut.  You need to install one on each door so that the switch is *closed* when the garage doors are closed.  Attach the end without wire hookups to the door itself, and the other end (the one that wires get attached to) to the frame of the door in such a way that they are next to each other when the garage door is shut.  There can be some space between them, but they should be close and aligned properly, like this:

![Sample closed contact switch][3]

*Step 2: Install the relays:*

The relays are used to mimic a push button being pressed which will in turn cause your garage doors to open and shut.  Each relay channel is wired to the garage door opener identically to and in parallel with the existing push button wiring.  You'll want to consult your model's manual, or experiment with paper clips, but it should be wired somewhere around here:

![!\[Wiring the garage door opener\]][4]

*Step 3: Wiring it all together:*

The following diagram illustrates how to wire up a two-door controller.  The program can accommodate fewer or additional garage doors (available GPIO pins permitting).

![enter image description here][5]

Note: User [@lamping7](https://github.com/lamping7) has kindly informed me that my wiring schematic is not good.  He warns that the relay should not be powered directly off of the Raspberry Pi.  See his explanation and proposed solution [here](https://github.com/andrewshilliday/light-controller/issues/16).  That being said, I've been running my Raspberry Pi according to the above schematic for years now and I haven't yet fried anything or set fire to my house.  Your milage may vary.

Software Installation
-----

1. **Install the python twisted module** (used to stand up the web server):

    `sudo apt-get install python-twisted`

2. **Install the controller application**

    I just install it to ~/$USER/light-controller.  You can install it anywhere you want but make sure to adapt these instructions accordingly. You can obtain the code via Git by executing the following:

    `sudo apt-get install git`

    `git clone https://github.com/brain1011/light-controller.git`

    That's it; you don't need to build anything.

3. **Create SSL Certificates** (if desired).

    If you plan on using SSL by setting the **use_https** option to *true* in the `config.json` file, you will need to complete this step or provide your own private keys and certificate for secure communication.

    In order for secure communication to be allowed, the controller application needs SSL certificates.  To quickly generate SSL certificates, do the following:

    `mkdir -p /home/pi/light-controller-cert`

    `openssl req -new -x509 -days 3650 -nodes -out /home/$USER/light-controller-cert/localhost.crt -newkey rsa:4096 -sha256 -keyout /home/pi/light-controller-cert/localhost.key -subj "/CN=localhost"`

    `chmod 700 /home/$USER/light-controller-cert`

    `chmod 600 /home/$USER/light-controller-cert/*`

4. **Edit `config.json`**

    You'll need one configuration entry for each light.  The settings are fairly obvious, but are defined as follows:
    * **name**: The name for the garage door as it will appear on the controller app.
    * **relay_pin**: The GPIO pin connecting the RPi to the relay for that door.
    * **state_pin**: The GPIO pin conneting to the contact switch.
    * **state_pin_closed_value**: The GPIO pin value (0 or 1) that indicates the door is closed. Defaults to 0.
    * **approx_time_to_close**: How long the garage door typically takes to close.
    * **approx_time_to_open**: How long the garage door typically takes to open.

    The **approx_time_to_XXX** options are not particularly crucial.  They tell the program when to shift from the opening or closing state to the "open" or "closed" state.  You don't need to be out there with a stopwatch and you wont break anything if they are off.  In the worst case, you may end up with a slightly odd behavior when closing the garage door whereby it goes from "closing" to "open" (briefly) and then to "closed" when the sensor detects that the door is actually closed.

5. **Set to launch at startup**

    For Raspbian, a service script has been created. You can install it using the following:

    `sudo cp /home/$USER/light-controller/extra/lightcontrollerd.service /lib/systemd/system`

    `sudo chmod 644 /lib/systemd/system/lightcontrollerd.service`

    `sudo systemctl daemon-reload`

    `sudo systemctl enable lightcontrollerd.service`

    `sudo systemctl start lightcontrollerd.service`

    For other distributions; simply add the following line to your /etc/rc.local file, just above the call to `exit 0`:

    `(cd ~pi/light-controller; python controller.py)&`

6. **Using the Controller Web Service**

    The garage door controller application runs directly from the Raspberry Pi as a web service running on port **8081**, or port **8444** when HTTPS is enabled.  It can be used by directing a web browser (on a PC or mobile device) to **http://[hostname-or-ip-address]:8081/**, or **https://[hostname-or-ip-address]:8444/** when HTTPS is enabled.  If you want to connect to the raspberry pi from outside your home network, you will need to establish port forwarding in your cable modem.  

    When the app is open in your web browser, it should display one entry for each garage door configured in your `config.json` file, along with the current status and timestamp from the time the status was last changed.  Click on any entry to open or close the door (each click will behave as if you pressed the garage button once).

Using IFTTT and Basic API
------------

IFTTT has been implemented using a combination of sending 'alerts' to the maker channel on IFTTT and using the webhooks channel to send commands to the controller.

Set key under IFTTT to your maker key found under the maker channel.
Set ifttt_event_open and ifttt_event_close to the event code you want to trigger for each door.

API can be used via IFTTT webhooks (or anything else that can make get requests) to send a command to the controller (eg `http://public ip or dns/api?key=<key you set>&command=<open, close, or toggle>&id=<door id or all_doors>`)
Key is a string you set in the config.json, I recommend using something like <https://www.allkeysgenerator.com/Random/Security-Encryption-Key-Generator.aspx> to generate a random 256bit or higher key.

Close All
------------

Close all button will close all doors in the open state, all other states are ignored.
![Screenshot_20200612-162450](https://user-images.githubusercontent.com/5156472/84547697-ad0dfb00-acc9-11ea-8175-d8ec2e5f38ba.png)

TODO
----------  

This section contains the features I would like to add to the application, but do not currently have time for.  If someone would like to contribute changes or patches, I would be all to happy to incorporate them.

* *Security*: Impose a configurable password on the web service.  Would need to discuss the best strategy (i.e., should we require the pw every time, or can the session persist on any given device which has authenticated).
* ~~*New Feature*: Add a "close all" button to the bottom of the page to close all doors that have a state other than "closed" or "closing"~~ (Only checks for open state due to the way many garage door openers handle a button push while opening, would rather reduce the chance of pulling the door in a state not intended, ie partly open when user thought is was closing)
* *Occupancy sensors*: Add proximity sensors to check if car port is in use
* ~~*IFTTT Integration*: make a smooth secure way to call the door and get information online~~

  [1]: http://i.imgur.com/rDx9YIt.png
  [2]: http://i.imgur.com/bfjx9oy.png
  [3]: http://i.imgur.com/vPHx7kF.png
  [4]: http://i.imgur.com/AkNl6FI.jpg
  [5]: http://i.imgur.com/48bpyG0.png
