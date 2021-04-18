# Pihole using Raspberry pi 3b and Ubuntu server 20.04 ARM 64
An example guide to run pihole in a raspberry pi 3b for your local network

## Requirements

* [Raspberry pi](https://www.raspberrypi.org/products/) device
* SD Card
* Host with card reader or an external card reader
* Host machine with the following tools installed:
  * openssh
  * [Raspberry Pi Imager](https://www.raspberrypi.org/software/)
  * nmap

## Setting up your raspberry device

### Installing an OS in your raspberry pi

1. Insert your SD card to your host
2. In your host open *Raspberry Pi Imager* and choose the OS and SD card.

   <img width="680" alt="raspberry pi imager" src="https://user-images.githubusercontent.com/12837326/115128332-7e307780-9fb3-11eb-9295-6a20ad5aa229.png">

3. Follow the required steps to have your OS installed in your SD card.
4. Once the OS is installed, insert your SD card in the raspberry pi and
   connect it to a power supply and in the same network where your host machine
   is connected.

### Config the raspberry name and IP

1. Find the raspberry pi IP using *nmap* from your host

   ```bash
   $ nmap -sp 192.168.100.0/24
   Starting Nmap 7.91 ( https://nmap.org ) at 2021-04-17 19:27 -03
   Nmap scan report for 192.168.100.1
   Host is up (0.035s latency).
   Nmap scan report for 192.168.100.4
   Host is up (0.00034s latency).
   Nmap scan report for 192.168.100.39
   Host is up (0.024s latency).
   Nmap done: 256 IP addresses (3 hosts up) scanned in 10.32 seconds
   ```

   _in my case I know my host IP is `192.168.100.4` and `192.168.100.1`
   is my modem/router adress so my rasberry pi address is `192.168.100.39`_

2. Connect to your rasberry pi using *ssh*
   _(you may have configure the default user/password in the install process).
   In my case I just used the default user `ubuntu` and set `ubuntu` as password_

   ```bash
   $ ssh ubuntu@192.168.100.39
   ubuntu@192.168.100.39's password:
   ```

   * Once you are connected to your device, become sudo

     ```bash
     sudo su
     root@ubuntu:~$ 
     ```

#### Change your raspberry device host name (optional)

   ```bash
   hostnamectl set-hostname rasp3b
   ```

> You must disconect and connect to see the hostname change applied

#### Set an static IP for your rasberry pi

* Create your static ip config

  ```bash
  touch /etc/netplan/99_config.yaml
  vim /etc/netplan/99_config.yaml
  ```

  * Paste the following config and match the values with your network settings

  ```yml
  network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.100.8/24
      gateway4: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
  ```

* Deactivate the cloud network config

  * Create a new file in `/etc/cloud/cloud.cfg.d/`

    ```bash
    vim /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
    ```

  * Add the following contents

    ```yaml
    network: {config: disabled}
    ```

* Remove the cloud ip file

  * remove `/etc/netplan/50-cloud-init.yaml` file

    ```bash
    rm /etc/netplan/50-cloud-init.yaml
    ```

* Apply the changes

  ```bash
  sudo netplan apply
  ```

* Verify your address

  ```bash
  ip a | grep 192.168.100.8
  ```

### Config the access to the raspberry pi

1. in your *host* create a ssh key to connect to the rasberry without using password

   * Open a terminal in *your host* and create the key pair

   ```bash
   $ ssh-keygen -t rsa -b 4096 -c "my host"
   Generating public/private rsa key pair.
   Enter file in which to save the key (/Users/me/.ssh/id_rsa): /Users/me/.ssh/rasp3b
   ```
  
2. Config ssh in your host to connect using private key

   * Add a new host on your `~/.ssh/config` file

   ```bash
   #Pihole rasp3b
   Host pi3b-0
     User rasp3b
     HostName 192.168.100.8
     referredauthentications publickey
     IdentityFile ~/.ssh/rasp3b
   ```

3. Create and config a new user for pihole in your raspberry pi

* Create the user and follow the steps and set an easy password

  ```bash
  $ adduser rasp3b
  Adding user `rasp3b' ...
  Adding new group `rasp3b' (1000) ...
  Adding new user `rasp3b' (1000) with group `rasp3b' ...
  Creating home directory `/home/rasp3b' ...
  Copying files from `/etc/skel' ...
  New password:
  Retype new password:
  passwd: password updated successfully
  Changing the user information for rasp3b
  Enter the new value, or press ENTER for the default
    Full Name []:
    Room Number []:
    Work Phone []:
    Home Phone []:
    Other []:
  Is the information correct? [Y/n] y
  ```

* config private key authentication

  * Create the `.ssh` directory for your user

    ```bash
    sudo su raspb3
    mkdir ~/.ssh
    chmod 700 ~/.ssh
    touch ~/.ssh/authorized_keys
    chmod 644 ~/.ssh/authorized_keys
    ```

  * Copy the public key to the authorized_keys
    * In your *host*

      ```bash
      ssh-copy-id -i ~/.ssh/rasp3b rasp3b@192.168.100.8
      ```

    * follow the steps and login with your password

* test connecting using keys

  * In your *host*

    ```bash
    ssh pi3b-0
    ```

* remove ubuntu user (optional)

  ```bash
  userdel ubuntu
  ```

* remove password for pihole user (optional)

  ```bash
  sudo visudo
  ```

  * add at the end of the file:

  ```bash
  rasp3b  ALL=(ALL) NOPASSWD:ALL
  ```

  * save and exit

### Install the tooling to run pihole

Connected to your raspberry pi as raspb3 user:

* Installing docker and net-tools

  ```bash
  sudo apt update && sudo apt upgrade -y && sudo apt install -yqq \
  docker \
  docker-compose \
  net-tools
  ```

* Giving permission to rasp3b user to use docker

  ```bash
  sudo usermod -aG docker $USER
  ```

> disconnect and connect again to your device to changes take effect

* disable dns service

  ```bash
  sudo systemctl disable systemd-resolved
  sudo systemctl stop systemd-resolved
  ```

### Creating your `docker-compose.yaml` file

* In the `raspb3` user home directory create the following file:

  * docker-compose.yaml

    ```yaml
    version: '3.7'

    services:
      pihole:
        container_name: pihole
        image: pihole/pihole:v5.8
        ports:
          - "53:53/tcp"
          - "53:53/udp"
          - "67:67/udp"
          - "80:80/tcp"
          - "443:443/tcp"
        environment:
          TZ: 'America/Cordoba'
          WEBPASSWORD: 'testpihole'
        volumes:
          - './etc-pihole/:/etc/pihole/'
          - './etc-dnsmasq.d/:/etc/dnsmasq.d/'
        cap_add:
          - NET_ADMIN
        restart: unless-stopped
    ```

### Running pihole

```bash
docker-compose up -d
```

### Testing pihole in your host

* Config your host DNS to use the rasberry pi IP

> this change for every OS network settings

* Open a web broser and load a page with ads
* Enjoy your less ads browsing

### Access the admin page

* In your web browser open the raspberry pi IP `http://192.168.100.8/admin`

* Click login and see the metrics collected

### Config more block rules

* open `http://192.168.100.8/admin/groups-adlists.php`

* Select the community files that blocks the sites/ads you prefer

* add a new entry with the link to the file

* go to `http://pihole/admin/gravity.php` and click the update button
