## Raspberry PI Setup

Clone this project and edit the playbooks before running. Follow this _README_ for additional details.

### Network Configuration

I prefer to setup my Raspberry Pi headless, without a monitor and preferably, without a mouse and keyboard. For those Pis with wireless connectivity, We can have them connect to a wireless network on its first boot by adding a _wpa_supplicant.conf_ to the SD card before. For those Pis with wireless connectivity, We can have them connect to a wireless network on its first boot by adding a _wpa_supplicant.conf_ to the SD card before.

Make a copy of [_wpa_supplicant.conf_](https://github.com/leonelgalan/ansible-pi/blob/master/wpa_supplicant.conf.example) and fill in your network's name (ssid) and pre-shared key (psk), configure other properties as needed (e.g. country):

```sh
cp wpa_supplicant.conf.example wpa_supplicant.conf
code wpa_supplicant.conf
```

```conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US

network={
  ssid=""
  psk=""
  key_mgmt=WPA-PSK
}
```

### Prepare your SD Card

All the code in this section should be run on the same terminal session. Just make sure you edit the `IMAGE_NANE` and `DISK` variables before.

#### Download Raspian

Choose the Raspian image of your choice and download it from [https://www.raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/). Modify `IMAGE_NAME` before running.

```sh
# Options: raspbian (Desktop), raspbian_full (Desktop with recommended software), raspbian_lite (Lite)
IMAGE_NANE=raspbian_lite
curl --location --remote-name "https://downloads.raspberrypi.org/${IMAGE_NANE}_latest"
unzip *.zip
rm *.zip
IMAGE=$(ls *.img)
```

### Copy Raspbian and Setup the Network

Insert your card and find the disk's name: `diskN`, where `N` is a number. Identify the disk (not the partition) of your SD card (`disk2`, not `disk2s1`) by looking at its size  (for example, a 16GB SD card might show as *15.5 GB

```sh
diskutil list
```

**Make sure you modify `DISK` before.** Copy raspbian to the card and setup the network to:

1. Connect to a wireless network.
2. Accept incoming SSH connections.

```sh
DISK=disk2
diskutil unmountDisk /dev/$DISK
sudo dd bs=1m if=$IMAGE of=/dev/r$DISK conv=sync
# Between few seconds and a couple of minutes, Ctrl+T to view Progress

# Copy the wpa_supplicant.conf you created above
cp wpa_supplicant.conf /Volumes/boot/
# Enable incoming SSH connections by creating an empty ssh file.
touch /Volumes/boot/ssh

# Eject the SD card properly
sudo diskutil eject /dev/r$DISK
```

### Find your Raspberry Pi's IP Address

Insert the SD card on the and turn on your Pi; Wait about 1 minute for it to boot.

Find your Raspberry Pi's IP address by searching for part of its MAC Address:

```sh
$ sudo nmap -sP 192.168.1.0/24 | awk '/^Nmap/{ip=$NF}/(B8:27:EB|DC:A6:32)/{print ip}
192.168.1.11
```

You might need to install _nmap_ before. See additional details in this [Stack Exchange's post](https://raspberrypi.stackexchange.com/a/13937). Copy this IP address to the [_hosts_](https://github.com/leonelgalan/ansible-pi/blob/master/hosts) file under `[pi]`:

```
[pi]
192.168.1.11
```

### Install Ansible's requirements and Run _step01.yml_

#### A note about Ansible

> Ansible is an open-source software provisioning, configuration management, and application-deployment tool. - [Wikipedia: Ansible (software)](https://en.wikipedia.org/wiki/Ansible_(software))

##### What You Need to Know?

* _hosts_ define the list of hosts, playbooks say in which hosts they run. We are saying there is host _pi_ and its IP Address.
* The playbooks, defines in which hosts are going to run or "all", as who (what user) and what roles to run and their configuration.
* Roles are groupings of functionality, which facilitates sharing and there are plenty hosted in the Ansible Galaxy. This "project" has one local role and uses two from the Ansible Galaxy, listed in the [_requirements.yml_](https://github.com/leonelgalan/ansible-pi/blob/master/requirements.yml) and installed by calling the `ansible-galaxy` command below.
* A project might have local roles, in this case [_pip3_](https://github.com/leonelgalan/ansible-pi/tree/master/pip3). Inside of it the folders are named appropriately, you should explore its contents to peek inside of how Ansible works.
* Each role's documentation should tell you what the role can do and how to set it up.

#### [_step01.yml_](https://github.com/leonelgalan/ansible-pi/blob/master/step01.yml)

This is the minimum configuration I do on my Raspberry Pi's before using them. It adds a single role: _raspi_config_ which does some configuration by default, but allows me to override those to better suit my needs.

##### Implicit (default)

* Update and Upgrade
* Expand filesystem to fill the SD card
* Setup the Locale: `en_US.UTF-8`

##### Explicit

* Setup my timezone to `America/New_York`, default is `UTC`
* Enable the camera, if that's my intended use for this particular Pi
* Replace the default **pi** user with myself (`whoami`) and copy my public ssh key, so I can ssh in without specifying a user or typing a password.

Find the defaults and additional settings on the [role's README](https://github.com/mikolak-net/ansible-raspi-config). **Edit _step01.yml_ as needed before running** the following lines:

```sh
ansible-galaxy install -r requirements.yml
ansible-playbook _initial.yml -i hosts --ask-pass
```

### Run [_step02.yml_](https://github.com/leonelgalan/ansible-pi/blob/master/step02.yml)

Now connected as the user you just created (`remote_user: leonelgalan`, **REMEMBER** to change this), install the packages you might need, **edit step02.yml_** based on your desire setup:

* Packages:
  * `python3-pip`
  * `sense-hat`
  * `python-smbus`
  * `i2c-tools`
  * `python-setuptools`
* Python Packages
  * Upgrade `setuptools`
  * `RPI.GPIO`
  * `adafruit-circuitpython-htu21d`

```sh
ansible-playbook step02.yml -i hosts --ask-pass
```
