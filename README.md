# Automated Raspberry Pi Wardriving Rig Setup Guide

This guide walks through the setup of an automated wardriving system using a Raspberry Pi.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation and Configuration](#installation-and-configuration)
  - [Updating Raspberry Pi OS](#updating-raspberry-pi-os)
  - [Install Latest Kismet Release](#install-latest-kismet-release)
- [GPS Configuration](#gps-configuration)
  - [Identify your USB GPS device's port](#identify-your-usb-gps-devices-port)
  - [Configure Kismet to use the USB GPS device](#configure-kismet-to-use-the-usb-gps-device)
- [GPS via Serial Connection](#gps-via-serial-connection)
  - [Hardware Connection](#hardware-connection)
  - [Serial Console Configuration](#serial-console-configuration)
  - [Configure gpsd](#configure-gpsd)
- [USB GPS & Kismet Configuration](#usb-gps--kismet-configuration)
- [Setting Up RaspAP](#setting-up-raspap)
- [Automating Kismet, GPS Detection, and Starting the Python HTTP Server](#automating-kismet-gps-detection-and-starting-the-python-http-server)
- [Automating Script Execution at Boot](#automating-script-execution-at-boot)
- [Usage](#usage)
  - [Accessing Kismet and Data](#accessing-kismet-and-data)

## Prerequisites

- Raspberry Pi (3/4/5 recommended) running Raspberry Pi OS Lite (64-bit, Bullseye or Bookworm).
- USB WLAN adapter(s) compatible with Linux.
- GPS module or compatible USB device.

## Installation and Configuration

### Updating Raspberry Pi OS

- Connect via ssh: `ssh user@<PIDEVICEIP>` or use an external monitor and keyboard
- Update: `sudo apt update && sudo apt upgrade -y`

### Install Latest Kismet Release

1. Remove any previous kismet from source
    ```bash
    sudo rm -rfv /usr/local/bin/kismet* /usr/local/share/kismet* /usr/local/etc/kismet*
    ```

2. Add Kismet to apt sources & install
    
> [!NOTE]
> Enter yes when asked for suid-root helpers during installation.
    
 - **Debian Bullseye** ( i386 amd64 armhf arm64 )
```bash
wget -O - https://www.kismetwireless.net/repos/kismet-release.gpg.key --quiet | gpg --dearmor | sudo tee /usr/share/keyrings/kismet-archive-keyring.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/kismet-archive-keyring.gpg] https://www.kismetwireless.net/repos/apt/git/bullseye bullseye main' | sudo tee /etc/apt/sources.list.d/kismet.list >/dev/null
sudo apt update
sudo apt install kismet
```
        
 - **Debian Bookworm** ( amd64 arm64 )
 ```bash
wget -O - https://www.kismetwireless.net/repos/kismet-release.gpg.key --quiet | gpg --dearmor | sudo tee /usr/share/keyrings/kismet-archive-keyring.gpg >/dev/null
echo 'deb [signed-by=/usr/share/keyrings/kismet-archive-keyring.gpg] https://www.kismetwireless.net/repos/apt/git/bookworm bookworm main' | sudo tee /etc/apt/sources.list.d/kismet.list >/dev/null
sudo apt update
sudo apt install kismet       
 ```

3. Add your username to the Kismet group: `sudo usermod -aG kismet your-user-here`

4. Reload the Groups: Either log back out and log in, or in some cases, reboot.

5. Check that you are in the Kismet group with the `groups` command output.

_For help visit the Kismet [Docs](https://www.kismetwireless.net/docs/readme/installing/linux/)_

## GPS via Serial Connection

>[!IMPORTANT]
> If using a USB GPS unit, skip this section.
>

**BEGIN GPIO SERIAL GPS INSTRUCTIONS**

- If using a GPS module instead of USB device, Pins 14 & 15 are used GPS data. 
    - Most any will work, [view the Pin Out here](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html)

### Hardware Connection

1. Connect your GPS module to the Raspberry Pi Zero as follows:

- **GPS Module TX** to **Raspberry Pi RX** on GPIO 15 (physical pin 10)
- **GPS Module RX** to **Raspberry Pi TX** on GPIO 14 (physical pin 8)
- **GPS Module Ground** to **Raspberry Pi Ground**
- **GPS Module Power** to **Raspberry Pi 3.3V or 5V** (depending on your GPS module requirements)

2. Install `gpsd`
    ```bash
    sudo apt-get install gpsd gpsd-clients
    ```

### Serial Console Configuration

Disable the serial console that might be using the serial pins:

3. Open the `raspi-config` tool:
    ```bash
    sudo raspi-config
    ```

4. Navigate to **Interface Options > Serial Port**.

5. Answer 'No' to the login shell over serial and 'Yes' to the serial port hardware being enabled.

### Configure `gpsd`

Edit the `gpsd` configuration file to adjust settings:

6. Open the configuration file:
    ```bash
    sudo nano /etc/default/gpsd
    ```

7. Ensure the `DEVICES` line points to the serial device your GPS is connected to (usually `/dev/serial0` or `/dev/ttyS0`):
    ```bash
    START_DAEMON="true"
    DEVICES="/dev/ttyS0"
    GPSD_OPTIONS="-n"
    ```

8. Save and exit the editor.

9. Restart `gpsd` to apply the changes:
    ```bash
    sudo systemctl restart gpsd
    ```

10. Test if the GPS module is working properly with gpsd: `cgps -s`

*END GPIO SERIAL GPS INSTRUCTIONS*

## USB GPS Configuration

### Identify your USB GPS device's port:

```bash
dmesg | grep tty
```

Look for lines that indicate your GPS device, typically `/dev/ttyUSB0`, or using the RX & TX, `ttyS0`.

## Configure Kismet to use the USB GPS device:

```bash
sudo nano /etc/kismet/kismet.conf
```

1. Add or modify the line for your GPS device:
    ```plaintext
    gpstype=serial:device=/dev/ttyUSB0
    ```

2. Replace `/dev/ttyUSB0` with the correct port for your USB GPS device

## Setting Up RaspAP 
*(Optional, enables an access point on the pi)* Install RaspAP for easy system management via your own access point.
Visit the [RaspAP Docs](https://raspap.com/#quick) for more info.

1. One-line quick installer:
    ```bash
    curl -sL https://install.raspap.com | bash
    ```

2. Follow the installation prompts, connect to the default AP then access the RaspAP web interface at 
`http://raspberry_pi_ip/` to configure your network settings.

- SSID: `raspi-webgui`
- Password: `ChangeMe`
- Default login to WebUI: username `admin` password `secret`.
- **Change these!**.

## Automating Kismet, GPS Detection, and Starting the Python HTTP Server

### Option 1: Complex Script:

1. Install `jq` for the script parsing
    ```bash
    sudo apt-get install jq
    ```

2. Create the startup script named start_kismet.sh:

This can be done using a text editor. For simplicity, we'll use nano:

- Enter the following command to create and edit the script:
    ```bash
    sudo nano ~/start_kismet.sh
    ```

3. Add the Script Content:

*Script adapted from [here](https://felixkohlhas.com/projects/wardriving/)
**Note: Change the /yourUsernameHere to your username**

```bash
#!/bin/bash

# Wait for 30 seconds to give the system time to detect and bring up interfaces
sleep 30

# Define the user's home directory and the directory for Kismet data
USER_HOME="/home/yourUsernameHere"
KISMET_DIR="${USER_HOME}/kismet"

# Create the Kismet directory if it doesn't exist
mkdir -p "${KISMET_DIR}"
cd "${KISMET_DIR}"

# Find network interfaces that are not wlan0 and bring them up
interfaces=$(iw dev | grep Interface | cut -d ' ' -f2 | grep -v wlan0)
kismet_sources=""
source_count=0

for interface in $interfaces; do
    sudo ip link set "$interface" up
    let "source_count+=1"
    kismet_sources+="-c ${interface}:name=Wifi${source_count} "
done

# Start Kismet with the detected interfaces
kismet $kismet_sources --override wardrive > kismet.log &
``` 
### Option 2: Simplified Script with heartbeat monitor:

Or simplified, with a check to restart Kismet and run as daemon:
_(Sources auto-detected or specified as above in the kismet.conf file)_

```bash
#!/bin/bash

# Wait for 30 seconds to give the system time to detect and bring up interfaces
sleep 30

# Define the user's home directory and the directory for Kismet data
USER_HOME="/home/yourUsernameHere"
KISMET_DIR="${USER_HOME}/kismet"

# Create the Kismet directory if it doesn't exist
mkdir -p "${KISMET_DIR}"
cd "${KISMET_DIR}"

# Command to start Kismet with WebUI enabled, running in quiet mode
KISMET_COMMAND="kismet -t PiDriver --override wardrive -q -s"

# Function to check if Kismet is running by pinging the web interface
check_kismet() {
    if ! curl -Is http://localhost:2501 | grep -q "200 OK"; then
        $KISMET_COMMAND
    fi
}

# Initial Kismet start
$KISMET_COMMAND

# Loop to check and restart Kismet if not running
while true; do
    sleep 60  # Check every 60 seconds
    check_kismet
done

``` 
 
4. Save and exit the editor by pressing `CTRL + X`, then `Y`, then `ENTER`.

5. Make the script executable:
    ```bash
    sudo chmod +x ~/start_kismet.sh
    ```

## Automating Script Execution at Boot

To ensure the script runs at startup, add it to the crontab:

1. Open the crontab editor:
    ```bash
    crontab -e
    ```

2. Add the following line at the end of the file:
    ```bash
    @reboot /path/to/your/script.sh
    ```

3. Save and exit the editor.
## Optimizing Networks Seen
Though it’s best practice to scan all available channels, some users may want to see more APs for max yield. By using the most popular (wigle.net stats) channels we can do that:

1. Set identifiable network interfaces to yes  using raspi-config
2. Use ifconfig or airmon-ng to show the device names
3. Edit /etc/kismet/kismet.conf and modify the source= lines as so, changing device names to match your own. 

```
source=wlx00c0cab46dh4:name=Device1,channels="1,6,11,36,149,9,4,10,153,7,52,13,108,132,116,124"
source=wlx00c0cab563vb:name=Device2,channels="1,6,11,44,157,48,3,2,100,8,161,56,112,60,136,128"
source=wlx00c0cab565jg:name=Device3,channels="1,6,11,40,149,5,36,100,153,12,52,64,104,116,132,120"
```


> [!NOTE]
> This is an example. Please use the channels that you see fit.  		

From [Kismet Docs](https://www.kismetwireless.net/docs/readme/datasources/wifi-linux/):
> By default, Kismet enables all channels it discovers on all bands. By specifying a specific band, Kismet will only enable channels on the selected bands.
> 
> Example:
> 
> ```
> # Source0 enables 2.4ghz channels only.
> source=wlan0:name=Source0:band24ghz=true 
> # Source1 enables 5ghz and 6ghz channels only.
> source=wlan1:Name=Source1:band5ghz=true,band6ghz=true```


## Usage

### Accessing Kismet and Data

**Kismet**

- Once the Raspberry Pi boots up, Kismet should start automatically
- Access the Kismet web interface by navigating to `http://<raspberry_pi_ip>:2501` in your web browser.

**Filesystem**
- If using script #1, connect to the pi & access the python http.server: 
  - Files hosted at: `http://<raspberry_pi_ip>:8080`

Alternatively, connect via ssh, sftp, or other methods of your choice. 
  - Kismet data is stored in `/home/<YOUR_USERNAME>/kismet`
