# Introduction
This is a quick self-guide / reminder of all performed steps to set up a Raspberry Pi 4 Model B to run several useful packages. This notes are intended to be just general guidelines / side notes to quickly install all required software + fixes to all problems found along these installations. Here I'll post references to the guides used by me (crediting the authors) + some workarounds / fixes I had to apply to solve some of the problems I found along the way. This is not intended to be an exhaustive guide, but a quick reference to fix the problems / log the solutions in case I (or you) want to follow the steps again.

# Initial setup
Just follow [this guide](https://www.flopy.es/instalar-raspbian-server-en-una-raspberry-pi-sin-monitor/). Credits to [Marcos Matas/Flopy.es](https://www.flopy.es/).
Raspbian-buster-lite provided by [DistroWatch](https://distrowatch.com/?newsid=10816) / DD-link [here](https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2020-02-07/2020-02-05-raspbian-buster-lite.zip).

## Force static IP for eth0 and wlan0
``sudo nano /etc/dhcpcd.conf``

Add at the end:
```
interface eth0
static ip_address=192.168.100.169/24
static routers=192.168.100.1
static domain_name_servers=1.1.1.1 8.8.8.8
interface wlan0
static ip_address=192.168.69.1/24 
nohook wpa_supplicant
```
Adapt ``static ip_address`` accordingly for both ``eth0`` and ``wlan0``. We'll leave ``wlan0`` as is for now and configure the DHCP server via Pi-Hole (more on that later).

# PiHole + Wireless AP + DHCP
Credits to [u/spinzthewiz/](https://www.reddit.com/user/spinzthewiz/).

Use steps provided [here](https://www.reddit.com/r/raspberry_pi/comments/eai6i3/tutorial_pihole_wireless_ap_dhcp/).

My ``sudo nano /etc/hostapd/hostapd.conf`` file for reference:
```
country_code=ES
interface=wlan0
ssid=RasPi-WiFi
hw_mode=g
channel=8
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=your-password-goes-here
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

Should you follow the sample network used in this tutorial (192.168.100.0 for ``eth0`` // 192.168.69.0 for ``wlan0``), access to PiHole's DHCP settings should be [here](http://192.168.100.169/admin/settings.php?tab=piholedhcp). I.E. [http://192.168.100.169/admin](http://192.168.100.169/admin) > login > Settings > DHCP tab.

# Plex Server
Use steps provided [here](https://www.flopy.es/instalacion-de-plex-server-en-una-raspberry-pi-todo-un-servidor-multimedia-en-un-equipo-diminuto/). Credits to [Marcos Matas/Flopy.es](https://www.flopy.es/).

## Permanent mounts
This allows us to mount an external hard drive / USB every time the RPi boots. Reference documentation [here](https://www.codingame.com/playgrounds/2135/linux-filesystems-101---block-devices/permanent-mounts) and [here](https://askubuntu.com/questions/154180/how-to-mount-a-new-drive-on-startup).
My ``sudo nano /etc/fstab`` file:
```
proc            /proc           proc    defaults          0       0
PARTUUID=4c8eae3c-01  /boot           vfat    defaults          0       2
PARTUUID=4c8eae3c-02  /               ext4    defaults,noatime  0       1
PARTUUID=3bfb1dcb-13d6-494a-aa9d-33641ed4a306   /mnt/Plex       ntfs    nofail  0       0
/dev/sdb1       /mnt/Torrents   ext4    nofail 0        0
# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
```

Get the ``PARTUUID`` via ``sudo blkid /dev/sda1`` (adapt ``/dev/sdXX`` accordingly).

# Telegram bot
Use steps provided [here](https://www.flopy.es/crea-un-bot-de-telegram-para-tu-raspberry-ordenale-cosas-y-habla-con-ella-a-distancia/). Credits to [Marcos Matas/Flopy.es](https://www.flopy.es/).

The bot / Python script eventually crashes / timeouts for me. Some people fix it with [this](https://github.com/eternnoir/pyTelegramBotAPI/issues/474#issuecomment-401703044), but that didn't work for me. The workaround I eventually found is to add a ``crontab`` action that checks wether the bot is running. If so, nothing happens. If the process has crashed, it starts the bot. This is checked every 15 minutes via ``crontab``, but adapt accordingly.

Command is ``crontab -e``, entry is:

``*/15 * * * *  pgrep python > /dev/null 2>&1 || python /home/pi/apps/pyTelegramBotAPI/Telegram_bot.py &``

(adapt path and .py executable name accordingly)

# VPN - remote connection

Register [here](https://freedns.afraid.org/).

Follow the steps on [this](https://www.flopy.es/pivpn-bloquea-publicidad-en-tu-movil-y-accede-a-los-dispositivos-de-tu-hogar/) guide **starting from step 3**. Credits to [Marcos Matas/Flopy.es](https://www.flopy.es/). Recommended use of ``WireGuard`` instead of ``OpenVPN``. Easier setup and use.

Usage of crontab again recommended for dynamically updating the DNS records. Afraid.org provides example entries in [https://freedns.afraid.org/dynamic/](https://freedns.afraid.org/dynamic/). Go to ``your domain`` > ``quick cron example``. It already provides the corresponding crontab commmand with your API.

# Transmission installation
Straightforward follow [this](https://www.flopy.es/transmission-en-una-raspberry-pi-la-forma-mas-inteligente-de-descargar-torrents/) tutorial. Credits to [Marcos Matas/Flopy.es](https://www.flopy.es/). 

If you want to link it to the previous Telegram bot, edit the ``"script-torrent-done-filename":`` line. In my case, it reads:

``"script-torrent-done-filename": "/home/pi/apps/TransmissionTelegramNotification.sh",``

Contents for the ``TransmissionTelegramNotification.sh`` are:
```
#!/bin/bash
# Shell script: TransmissionTelegramNotification.sh
# Source code: https://github.com/rteixeirax/transmission-telegram-notifications
# Adapted by: Arnau Dalmau
# Bot token
BOT_TOKEN="TOKEN OF YOUR BOT"

# Your chat id
CHAT_ID="CHAT ID YOU WANT TO SEND THE MESSAGE TO"

# Notification message
# If you need a line break, use "%0A" instead of "\n".
MESSAGE="[Transmission] Baixada completada:%0A ${TR_TORRENT_NAME}%0A"

# Prepares the request payload
PAYLOAD="https://api.telegram.org/bot${BOT_TOKEN}/sendMessage?chat_id=${CHAT_ID}&text=${MESSAGE}&parse_mode=HTML"

# Sends the notification to the telegram bot and save the response content into the notificationsLog.txt
curl -S -X POST "${PAYLOAD}" -w "\n\n" | sudo tee -a notificationsLog.txt

# Prints a info message in the console
echo "[${TR_TIME_LOCALTIME}]-[${TR_TORRENT_NAME}] Download completed. Telegram notification sent."
sleep 15
PlexRefresh
```

The ``PlexRefresh`` script is referenced at the bottom. Comment it with a ``#`` if you don't need it / don't want it.

# Sonarr installation
Credits to [Ivan Arjona](https://gist.github.com/IvanArjona/) and his [Sonarr installation instructions](https://gist.github.com/IvanArjona/34cd1ae0b863f5f18cb84e860ab33e74#sonarr).

## Install libmono-cil-dev and mono 3.10
```
sudo apt-get install libmono-cil-dev
wget http://sourceforge.net/projects/bananapi/files/mono_3.10-armhf.deb
sudo dpkg -i mono_3.10-armhf.deb
```

## Install Sonarr
```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 0xA236C58F409091A18ACA53CBEBFF6B99D9B78493
echo "deb http://apt.sonarr.tv/ master main" | sudo tee /etc/apt/sources.list.d/sonarr.list
sudo apt-get update
sudo apt-get install nzbdrone
sudo chown -R pi:pi /opt/NzbDrone
```

## Autostart script
Located in ``/etc/systemd/system/sonarr.service``

Command is:
```
sudo tee /etc/systemd/system/sonarr.service <<EOF
[Unit]
Description=Sonarr Daemon
After=network.target

[Service]
User=pi
Group=pi

Type=simple
ExecStart=/usr/bin/mono /opt/NzbDrone/NzbDrone.exe -nobrowser
TimeoutStopSec=20
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

Then:
```
sudo systemctl daemon-reload
sudo systemctl start sonarr
sudo systemctl enable sonarr
```

Access via [http://192.168.100.169:8989/](http://192.168.100.169:8989/).

# Jackett installation 
Credits to [Ivan Arjona](https://gist.github.com/IvanArjona/) and his [Jackett installation instructions](https://gist.github.com/IvanArjona/34cd1ae0b863f5f18cb84e860ab33e74#jackett).

Download latest [Jackett release](https://github.com/Jackett/Jackett/releases).

```sh
wget -c https://github.com/Jackett/Jackett/releases/download/<VERSION>/Jackett.Binaries.LinuxARM32.tar.gz
tar zxvf Jackett.Binaries.LinuxARM32.tar.gz 
sudo mv Jackett /opt/
```

Startup file `/lib/systemd/system/jackett.service`.

```sh
sudo tee /lib/systemd/system/jackett.service <<EOF
[Unit]
Description=Jackett Daemon
After=network.target

[Service]
SyslogIdentifier=jackett
Restart=always
RestartSec=5
Type=simple
User=pi
Group=pi
WorkingDirectory=/opt/Jackett
ExecStart=/opt/Jackett/jackett --NoRestart
TimeoutStopSec=20

[Install]
WantedBy=multi-user.target
EOF
```

```sh
sudo systemctl daemon-reload
sudo systemctl start jackett
sudo systemctl enable jackett
```

Acces via [192.168.100.169:9191](http://192.168.100.169:9191).

## Jackett uninstallation
Reference [here](https://github.com/Jackett/Jackett/issues/6169#issuecomment-541991563). Credit to user [garfield69](https://github.com/garfield69).

```
systemctl stop jackett.service
cd /lib/systemd/system/
sudo rm jackett.service
cd /opt/Jackett
sudo rm -rf *
sudo rmdir /opt/Jackett
```

# Additional tools and resources
## PlexRefresh script
To increase functionality of the Pi + Transmission, I wanted to force Plex to refresh its library (both when a torrent finishes downloading, check [Transmission installation](https://gist.github.com/arnaudalmau/73c6d17f697380f0fbc3151324bb1e1d#transmission-installation) or manually via the ``PlexRefresh`` command). To do so, a basic script using ``wget`` is used. Contents of this script are:
```
  #!/bin/bash
  # Shell script: PlexRefresh
  # Autor: Arnau Dalmau
  wget http://192.168.100.169:32400/library/sections/all/refresh?X-Plex-Token=YOUR-PLEX-TOKEN &> /dev/null
  rm refresh*
```

(adapt host IP and Plex Token accordingly)

To allow this script to be executed at any time from bash, use:
```
sudo ln -s /home/pi/apps/PlexRefresh /usr/bin
```

(adapt path and name of the script accordingly)

## Show temperatures script
Use steps provided [here](https://www.flopy.es/como-medir-la-temperatura-de-una-raspberry-pi-desde-el-terminal/). Credits to [Marcos Matas/Flopy.es](https://www.flopy.es/).
Code for my script is as follows:
```
 #!/bin/bash
 # Shell script: temps.sh
 # Author: Santiago Crespo - Modified by Marcos Matas, Arnau Dalmau
  clear
  cpu=$(cat /sys/class/thermal/thermal_zone0/temp)
  echo "Hi, I'm $(hostname) :)"
  echo "------------------------------"
  echo "Timestamp:"
  echo "$(date)"
  echo "------------------------------"
  echo "CPU temp=$((cpu/1000)).0'C"
  echo "GPU $(/opt/vc/bin/vcgencmd measure_temp)"
  echo "------------------------------"
  echo "CPU info:"
  echo -n "freq="
  cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq | awk '{print $1/1000,"MHz"}'
  echo "$(vcgencmd measure_volts core)"
  echo "System memory:  $(vcgencmd get_mem arm)"
  echo "Graphics memory: $(vcgencmd get_mem gpu)"
  echo "------------------------------"
  echo "Memory status:"
  echo "$(egrep --color 'Mem|Cache|Swap' /proc/meminfo)"
```
