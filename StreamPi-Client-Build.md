# StreamPi Client Build

This document is provided as an example for building and setting up the StreamPi client.

> StreamPi does not require a great deal of hardware resources.  
> A Raspberry Pi 3 has penty of resources, especially when using this build guide.

Recommended Hardware List $125-$150

- [ ] Raspberry Pi 3 Model B/B+
- [ ] MicroSD memory card 8GB or larger
- [ ] MicroSD compatible card reader (used to apply initial OS image)
- [ ] USB Keyboard
- [ ] Ethernet Cable** and Internet access from the Raspberry Pi
- [ ] 5V 2.5A Raspberry Pi Power Adapter
- [ ] Raspberry Pi 7" Touch Screen Display [https://www.amazon.com/gp/product/B0153R2A9I]
- [ ] Raspberry Pi 7-Inch LCD Touch Screen Case, Black [https://www.amazon.com/gp/product/B01GQFUWIC]

\*** Wireless configuration is also possible but not covered in these instructions.

Required Software:

- [ ] Raspberry Pi Imager [https://www.raspberrypi.org/software/]
- [ ] An SSH Client. For example. PuTTY [https://www.putty.org/]

## Install Raspbian

> **NOTE** Installation of Raspbian is performed through a base disk image. Raspberry Pi provides a tool called *Raspberry Pi Imager* that facilitates selection, download, and write of the image to the memory card from your computer.

Complete the following steps to apply the Raspbian OS onto the MicroSD card:

1. Connect card ready to computer.
1. Connect MicroSD memory card into your card reader.
1. Download and install Raspberry Pi Imager [https://www.raspberrypi.org/software/]
1. Start/launch Raspberry Pi Imager
    1. Click **CHOOSE OS**
        1. Select **Rasberry Pi OS (other)**
        1. Select **Raspberry Pi OS Lite (32-bit)**
    1. Click **CHOOSE SD CARD** 
        1. Select your SD Card.
    1. Click **WRITE**

When write is complete, disconnect the SD card.

## Configure /boot/config.txt

> **NOTE** The display is inverted (upside-down) in the case when installed.  A setting is required to flip the orientation.

Complete the following steps to set boot configuration options:

1. Connect card ready to computer.
1. Connect MicroSD memory card into your card reader.
1. Access the SD card and open config.txt in a text editor
    1. Append to the bottom of the file 'lcd_rotate=2' (example below)
1. Save file.
1. Diconnect and remove memory card.

For example:
```text
[all]
#dtoverlay=vc4-fkms-v3d
lcd_rotate=2
```

## Hardware Assembly

Complete the following steps to assembly the hardware components:

1. Unpack and assembly the touch screen display.  Leave display face down.
1. Attach video ribbon cable to display adapter board.
1. Attach Raspberry Pi to display adapter board.
1. Insert memory card into Rawpberry Pi.
1. Attach video ribbon cable to Raspberry Pi.
1. Attach power cables between adapter board and Raspberry Pi.
    1. Connect display adapter board pin 5 (5v) to Raspberry Pi pin 2 (5v).
    1. Connect display adapter board pin 1 (ground) to Raspberry Pi pin 6 (ground).
1. Attach hardware to case and secure with provided screws (4).
1. Connect power adapter.

> **NOTE** With power applied the display will show a square color test and boot to a TTY Console.

## Create 'KIOSK'

> **NOTE** StreamPi works best as an applliance-type build.  This kiosk approach provides a minimal environment to meet the needs of StreamPi.

Complete the following steps to enable SSH to assist in setup steps:

1. Connect the Raspberry Pi to a network that provides DHCP and has Internet access.
1. Login to the Raspberry Pi:
    - Username: **pi**
    - Password: **raspberry**
1. Verify the system has an IP address:
    ```bash
    hostname -I
    ```
    For example:
    ```bash
    pi@raspberrypi:~ $ hostname -I
    192.168.10.84
    ```
1. Start SSH:
    ```bash
    sudo systemctl start ssh.service
    systemctl is-active ssh.service
    ```
    For example,
    ```bash
    pi@raspberrypi:~ $ sudo systemctl start ssh.service
    pi@raspberrypi:~ $ systemctl is-active ssh.service
    active
    ```

> **NOTE** The SSH service is now active but not enabled on boot.  If the Raspberry Pi is rebooted, these steps will need to be repeated.  SSH is **not** required post setup.

> **NOTE** Through an SSH session, you will be able to cut and paste the commands provided below. :sunglasses:

Complete the following steps to configure kiosk running under user 'pi':

1. SSH to Raspberry Pi and login with credentials provided above.
1. Update Raspbian:
    ```bash
    apt-get update
    apt-get -y upgrade
    ```
    > **NOTE** If prompted, NODM does not need to be enabled at this time.
1. Install application to support kiosk:
    ```bash
    apt-get install feh matchbox-window-manager nodm unclutter xorg
    ```
1. Add group 'input' to device /sys/class/input ('pi' user is already a member of group 'input')
    ```bash
    cat <<-EOF >> /etc/udev/rules.d/99-com.rules
    SUBSYSTEM=="input*", PROGRAM="/bin/sh -c '\
    chown -R root:input /sys/class/input/*/ && chmod -R 770 /sys/class/input/*/;\
    '"
    EOF
    ```
    > **IMPORTANT** This will remove the need to run StreamPi as root.
1. Enable NODM and configure to run as 'pi' user.
    ```bash
    sudo sed -i \
    -e 's/^NODM_ENABLE=/NODM_ENABLE=true/' \
    -e 's/^NODM_USER=/NODM_USER=pi/' \
    /etc/default/nodm
    ```
1. Create startup script:
    ```bash
    cat <<-EOF > /home/pi/.xsession
    #!/usr/bin/env bash

    function f_startStreamPiClient
    {
            # Make sure we have an IP address
            while [[ -z "$(hostname -I)" ]]; do
                    echo "Waiting for a network address ..."
                    sleep 15s
            done

            # Start StreamPi Client
            if [[ -f ~/StreamPi/StreamPiClient-0.6.jar && -x ~/StreamPi/jre/bin/java ]]; then
                    cd ~/StreamPi && {
                    ~/StreamPi/jre/bin/java \
                    -Dcom.sun.javafx.isEmbedded=true \
                    -Dcom.sun.javafx.touch=true \
                    -Dcom.sun.javafx.virtualKeyboard=javafx \
                    -jar StreamPiClient-0.6.jar &
                    }
            fi
    }

    # Disable Screen Sleep/Blank
    xset s off
    xset -dpms
    xset s noblank

    # Set background (optional)
    if [ -f /home/pi/wallpaper/bg-image ]; then
            feh --bg-center /home/pi/wallpaper/bg-image
    fi

    # Set brightness
    sudo /bin/bash -c '{ echo 255 > /sys/class/backlight/rpi_backlight/brightness; }'

    # Pause before starting (allow disable auto-start)
    xmessage -buttons Disable-StreamPi-Client:101,Start-StreamPi-Client:102 -center -default Start-StreamPi-Client -timeout 15 "Ready to start StreamPi Client. Waiting 15 seconds."
    case "$?" in
            101)    touch ~/noApp
                    ;;
            102)    # Do Nothing
                    ;;
    esac

    # Start StreamPii Client or show message if disabled.
    if [ ! -f ~/noApp ]; then

            # StreamPi Client enabled
            f_startStreamPiClient

    else
            # StreamPi Client disabled.
            while true; do
                    xmessage -buttons Enable-StreamPi-Client:103,Reboot-RaspberryPi:104 -center "StreamPi Client is currently disabled. Press Ctrl+Alt+F1 for TTY login."
                    case "$?" in
                            103)    rm ~/noApp
                                    f_startStreamPiClient
                                    ;;
                            104)    sudo reboot
                                    ;;
                    esac
                    sleep 15s
            done
    fi

    # Hide cursor ??
    unclutter -idle 0 &

    # Force full screen and hold X session open.
    while true; do
            exec matchbox-window-manager -use_titlebar no
            sleep 15s
    done

    EOF

    ```
    > **NOTE** Assumes *StreamPiClient-0.6.jar*
1. Update shared-object links to support hardware accelaration.
    ```bash
    sudo ln -sf libbrcmEGL.so /opt/vc/lib/libEGL.so
    sudo ln -sf libbrcmGLESv2.so /opt/vc/lib/libGLESv2.so
    ```

## Install StreamPi Client

> **NOTE** This installation bypasses the install and startup script include in the released zip file.  This guide and the .xsession script servers to replace those items.

Complete the following steps to install StreamPi Client:

1. SSH to Raspberry Pi and login with credentials provided above.
1. Download client from GIT:
    ```bash
    wget https://github.com/dubbadhar/streampi_client/releases/download/0.0.6/linux_armv7.zip -O /tmp/linux_armv7.zip
    ```
1. Extract and place required components:
    ```bash
    [ -d ~/StreamPi ] || mkdir ~/StreamPi
    unzip /tmp/linux_armv7.zip -d /tmp/
    find /tmp/linux_arm* -type f -name 'StreamPiClient*.jar' -exec mv {} ~/StreamPi/ \; 2>/dev/null
    find /tmp/linux_arm* -type d -name 'jre' -exec mv {} ~/StreamPi/ \; 2>/dev/null
    rm -Rf /tmp/linux_arm*
    chmod 550 ~/StreamPi/jre/bin/*
    ```
1. Create initial configuration:
    ```bash
     declare -a screen0=($(xrandr -q -d :0 2> /dev/null | sed -n -r 's/.*connected\s([0-9]+)x([0-9]+).*$/\1 \2/p'))
     echo "${screen0[0]}::${screen0[1]}::192.168.0.106::63::test1::1::1::100::15::" > ~/StreamPi/config
    ```
    > **NOTE** IP address (192.168.0.106) and port (63) above can be updated to reflect StreamPi server ip address and port.
1. Reboot System
    ```bash
    sudo reboot
    ```
