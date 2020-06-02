# rasp_kiosk
Instructions to build raspberry kiosk

# Headless raspbian lite
Use the raspbian lite version

## Image the device (OSX):
1. Identify the disk: 
`diskutil list`
2. Eject: 
`diskutil unmountDisk /dev/disk4`
3. Image the device with raspbien lite: 
`sudo dd if=2018-11-13-raspbian-stretch-lite.img of=/dev/rdisk4 bs=1m`
3. Enable ssh, create the ssh config: 
`sudo touch /Volumes/OS/ssh`
4. Eject the disk: 
`diskutil unmountDisk /dev/disk4`

## Boot the pi
1. Login with the default credentials (this will need looked at):
```
    Username: pi
    Password: raspberry
```
2. Start ssh: 
`sudo service ssh start`
3. Set hostname: 
```
    sudo nano /etc/hostname
    sudo nano /etc/hosts
    sudo hostname [name]
```
4. Make a folder for ssh: `Sudo mkdir ~/.ssh/authorised/`

## Start flow
- Update the device:
```
    sudo apt-get update
    sudo apt-get upgrade --yes
    sudo apt-get dist-upgrade --yes
```
- Install chromium:
``` 
    sudo apt-get install --no-install-recommends chromium-browser --yes
    sudo apt-get install xinit
```
- For xset (keep display alive)
`sudo apt-get install x11-xserver-utils`
		
- Create script: `sudo nano /etc/startWeb.sh`
    - Script content:
```
    #!/bin/sh
    # Disable any form of screen saver / screen blanking / power management
    xset -dpms     # disable DPMS (Energy Star) features.
    xset s off     # disable screen saver
    xset s noblank # don't blank the video device
    
    # Allow quitting the X server with CTRL-ATL-Backspace
    # setxkbmap -option terminate:ctrl_alt_bksp
    
    # Start Chromium in kiosk mode
    DISPLAY=:0 chromium-browser  --enable-native-gpu-memory-buffers --kiosk --start-maximized --incognito --disable-infobars 'https://www.example.com/' 
```
   - Script content (updated):
```
    # Set this to your URL - it must return a 200 OK when called, not a redirect.
    export URL=https://www.example.com/
		
    # setxkbmap -option terminate:ctrl_alt_bksp
    # Sit and wait until you can hit the URL you'll be showing in the kiosk
    while ! curl -s -o /dev/null -w "%{http_code}" ${URL} | grep -q "200"; do
      sleep 1
    done
		
    # get screen resolution
    WIDTH=$(sudo fbset --show | grep "geometry" | cut -d " " -f6)
    HEIGHT=$(sudo fbset -s | grep "geometry" | cut -d " " -f7)
		
		
    # Open chrome in incognito mode + kiosk mode
    chromium-browser --window-size=${WIDTH},${HEIGHT} --window-position=0,0 --incognito --kiosk ${URL}
    # chromium-browser --window-size=${WIDTH},${HEIGHT} --window-position=0,0  ${URL}
```

- Change permissions on the script: `sudo chmod +x /etc/startWeb.sh`

- Auto start the script: `sudo nano ~/.bash_profile`
Script content:
```
    if [[ -z $DISPLAY ]] && [[ $(tty) = /dev/tty1 ]]; then
        xinit /etc/startWeb.sh 
    fi
```			
- Reduce the mouse lag: `sudo nano /boot/cmdline.txt`
add to the end of the line: `usbhid.mousepoll=0`

- Network issue:
    - Eth0 stops on reboot, This is a temp workaround:
    - Created a script in /etc/netStart.sh
    - Script content:
```
    #!/bin/bash
    sudo service dhcpcd restart
    sudo ifconfig eth0 down
    sudo ifconfig eth0 up 
```
- Set permission: `sudo chmod +x /etc/netStart.sh`

- Then set a crontab job: `sudo crontab -e` 
    - Sselect option 2
    - Add the line: `@reboot /etc/netStart.sh`


## Issues Summary

- Keys (keys that can be pressed in the kiosk)
    - F1 help and links to various other options
    - ALT+F4: kills X (desktop utils) and returns to the terminal
        - Remove Alt F4: https://www.linuxquestions.org/questions/debian-26/disable-ctrl-alt-switching-of-vt-screens-4175607708/
        - Command: `sudo nano `/etc/xdg/openbox/rc.xml``
            - comment out the lines: with <!-- -->
```
            <keybind key="A-F4">
            <action name="Close"/>
            </keybind>
```
- Automation

