# Bootstrap Raspberry PI

## Prerequisites

1. MicroSD Card
2. USB to MicroSD Card adapter
3. Raspberry PI (tested on v3)
4. Mac or other computer which can run ssh and the imaging software

## Stage 1

Grab the imaging software from the raspberry pi website, relevant to your OS.

[Raspberry Pi Imager](https://downloads.raspberrypi.org/imager/imager_1.4.dmg) 

Install the software and execute.
When executing it will ask you to select an OS to install, pick other; then Debian (without desktop).


Once the image complete you need to remove the USB from the computer and then reinsert.

Open a terminal window and navigate to /Volumes/boot

```bash
cd /Volumes/boot
```

Next, run this command: touch ssh. This creates an empty file named “ssh” in the root directory of your SD card. Each time Raspbian boots it will enable sshd if it sees the “ssh” file and deletes the file immediately afterwards. This means that until you enable sshd permanently, you will need to add this file each time you boot your Pi.

```bash
touch ssh
```

Add WiFi network information

```bash
cat <<'EOF' >> wpa_supplicant.conf 
country=UK
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="NETWORK-NAME"
    psk="NETWORK-PASSWORD"
}
EOF
cd ~
```

Eject the SD Card
Right-click on boot (on your desktop or File Explorer) and select the Eject option
This is a “logical” eject - meaning it closes files and preps the SD card for removal - you still have to pull the card out yourself


Boot the Raspberry Pi
Remove the mini-SD card from the adapter and plug it into the Raspberry Pi
Plug a Micro-USB power cable into the power port
Give the Pi plenty of time to boot up (it can take as much as 90 seconds – or more)

Login over Wifi
Open up a terminal window
Run the following commands:

```bash
ssh-keygen -R raspberrypi.local
ssh pi@raspberrypi.local
```

Don’t worry if you get a host not found error for the first command - the idea is to clear out any previous references to raspberrypi.local
If the pi won’t respond, press Ctrl-C and try the last command again
If prompted with a warning just hit enter to accept the default (Yes)
Type in the password – by default this is *raspberry*


Configure the PI

```bash
 sudo raspi-config
```

* Ensure you change the password
* Set the hostname to "*vault*"
* Under interface options disable everything - but enable SSH
* Under advanced select expand disk to fill volume

Once complete reboot - wait about 90 seconds then reconnect via ssh

If you set the hostname above to "vault" use the following command

```bash
ssh pi@vault.local
```

Update the OS

```bash
sudo apt-get update -y
sudo apt-get upgrade -y
sudo reboot
```


### References

https://desertbot.io/blog/headless-raspberry-pi-3-bplus-ssh-wifi-setup
