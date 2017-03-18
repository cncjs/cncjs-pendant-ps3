# cncjs-pendant-ps3
Dual Shock / PS3 Bluetooth Remote Pendant for CNCjs

Use [Playstation 3 Controller](https://www.playstation.com/en-us/explore/accessories/dualshock-3-ps3/) wirelessly over bluetooth to control CNC from the host device (raspberry pi). [PS3 CNC Control Button Map](https://docs.google.com/drawings/d/1DMzfBk5DSvjJ082FrerrfmpL19-pYAOcvcmTbZJJsvs/edit?usp=sharing)

[Remote Pendant (Playstation 3 Dualshock Controller / SIXAXIS Controller)](https://github.com/cheton/cnc/issues/103)

Using a wireless game controller (like a PS3 controller) seems to be one of the lowest cost & simplest solution method. See related issue [#103](https://github.com/cheton/cnc/issues/103)

## Playstation Controller Setup ( general guide to connect hardware & setup )

Here is what I have figured out so far for PS3 on Raspberry PI 3 w/ integrated bluetooth.
The bellow just shows how to get PS3 controller connected.


## Bluetooth Configuration

### Install
```
# Install & Enable Bluetooth Tools
sudo apt-get install -y bluetooth libbluetooth3 libusb-dev
sudo systemctl enable bluetooth.service

# Add pi user to bluetooth group
sudo usermod -G bluetooth -a pi
```

### Pairing Tools
```
# Get and build the command line pairing tool (sixpair)
wget http://www.pabr.org/sixlinux/sixpair.c
gcc -o sixpair sixpair.c -lusb

### Connect PS3 over USB
# Get PS3 DS 
sudo ./sixpair
```

### [Parting DualShock 3 Controller](https://wiki.gentoo.org/wiki/Sony_DualShock)
```
### Disonnect DualShock 3 over USB

# Start bluetoothctl:
bluetoothctl

# Enable the agent and set it as default:
agent on
default-agent

# Power on the Bluetooth controller, and set it as discoverable and pairable:
power on
discoverable on
pairable on

### Connect DualShock 3 over USB, and press the PlayStation button.

# Discover the DualShock 3 MAC address:
devices

### Disonnect DualShock 3 over USB

#Allow the service authorization request:
#[agent]Authorize service service_uuid (yes/no): yes

#Trust the DualShock 3:
#trust device_mac_address # Replace "MAC" with MAC of "Device 64:D4:BD:B3:9E:66 PLAYSTATION(R)3 Controller"
trust 64:D4:BD:B3:9E:66 

# The DualShock 3 is now paired:
quit

# Turn the DualShock 3 off when it's no longer in use by pressing and holding the PlayStation button for 10 seconds.
# Press the PlayStation button to use the DualShock 3 again.
```

### Test Controller Connectivity
```
### PS3 Controller: press the PS button, the lights on the front of the controller should flash for a couple of seconds then stop, leaving a single light on. If you now look again at the contents of /dev/input you should see a new device, probably called something like ‘js0’:

# List Devices
ls /dev/input
```

### Get Battery Level
`cat "/sys/class/power_supply/sony_controller_battery_64:d4:bd:b3:9e:66/capacity"`


### Joystick Application
```
# Install
sudo apt-get -y install joystick

# Usage / Test
jstest /dev/input/js0
```

----------------------------------------

## Install NodeJS Libraries
 - https://www.npmjs.com/package/node-hid
 - https://www.npmjs.com/package/dualshock-controller

### Node.js DS3 Controller Setup
```
# Install Tools
sudo apt-get install -y libudev-dev libusb-1.0-0 libusb-1.0-0-dev build-essential git
sudo apt-get install -y gcc-4.8 g++-4.8 && export CXX=g++-4.8
#npm install node-gyp node-pre-gyp

# Install node-hid with hidraw support
#npm install node-hid --driver=hidraw --build-from-source --unsafe-perm

# Install Pendant Package
sudo npm install -g cncjs-pendant-ps3 --unsafe-perm  # Install Globally

## If NOT installed globally, Install node-hid with hidraw support (https://github.com/rdepena/node-dualshock-controller)
```

### [Create udev Rules](https://github.com/rdepena/node-dualshock-controller#-create-udev-rules)
```
# Run as Root
sudo su

# You will need to create a udev rule to be able to access the hid stream as a non root user.
sudo touch /etc/udev/rules.d/61-dualshock.rules
sudo cat <<EOT >> /etc/udev/rules.d/61-dualshock.rules
SUBSYSTEM=="input", GROUP="input", MODE="0666"
SUBSYSTEM=="usb", ATTRS{idVendor}=="054c", ATTRS{idProduct}=="0268", MODE:="666", GROUP="plugdev"
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", MODE="0664", GROUP="plugdev"

SUBSYSTEM=="input", GROUP="input", MODE="0666"
SUBSYSTEM=="usb", ATTRS{idVendor}=="054c", ATTRS{idProduct}=="05c4", MODE:="666", GROUP="plugdev"
KERNEL=="hidraw*", SUBSYSTEM=="hidraw", MODE="0664", GROUP="plugdev"
EOT

# Reload the rules, then disconnect/connect the controller.
sudo udevadm control --reload-rules

exit
```

# QUICK FIX for node-hid hidraw... PLEASE HELP IF YOU KNOW FIX
```
# I am having issues with node-hid --driver=hidraw not seeming to work... quickest fix is to reinstall, node-hid --driver=hidraw 

# Reinstall (node-hid --driver=hidraw) on cncjs-pendant-ps3
cd /usr/lib/node_modules/cncjs-pendant-ps3/
sudo npm install node-hid --driver=hidraw --build-from-source --unsafe-perm
```

I recommend rebooting now.
After reboot you can test pendant by running `cncjs-pendant-ps3 -p "/dev/ttyUSB0"`.

----------------------------------------

# Auto Start

### Install [Production Process Manager [PM2]](http://pm2.io)
```
# Install Production Process Manager [PM2]
npm install pm2 -g

# Setup PM2 Startup Script
pm2 startup debian
  #[PM2] You have to run this command as root. Execute the following command:
  sudo su -c "env PATH=$PATH:/home/pi/.nvm/versions/node/v4.5.0/bin pm2 startup debian -u pi --hp /home/pi"

# Start CNC.js (on port 8000) with PM2
pm2 start $(which cncjs-pendant-ps3) -- -p "/dev/ttyUSB0"

# Set current running apps to startup
pm2 save

# Get list of PM2 processes
pm2 list
```

----------------------------------------

# FFMpeg
http://www.jeffreythompson.org/blog/2014/11/13/installing-ffmpeg-for-raspberry-pi/
```
# Run as Sudo
sudo -i

# INSTALL H264 SUPPORT
cd /usr/src
git clone git://git.videolan.org/x264
cd x264
./configure --host=arm-unknown-linux-gnueabi --enable-static --disable-opencl
make
sudo make install

# INSTALL FFMPEG (This may take a REALLY long time, so be patient.)
cd /usr/src
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
sudo ./configure --arch=armel --target-os=linux --enable-gpl --enable-libx264 --enable-nonfree
make
sudo make install
```

# Recored Stream w/ ffmpeg
```
# [Varables]
source_stram="http://xcarve:8080/?action=stream"
destination_directory="/home/pi/Videos"
destination_file="xcarve-recording_$(date +'%Y%m%d_%H%M%S').mpeg"

# Recored Stream w/ ffmpeg
ffmpeg -i "${source_stram}" "${destination_directory}/${destination_file}"
```
