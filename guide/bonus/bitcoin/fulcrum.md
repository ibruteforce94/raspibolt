---
layout: default
title: Fulcrum server
parent: + Bitcoin
grand_parent: Bonus Section
nav_exclude: true
has_toc: false
---

## Bonus guide: Fulcrum server

{: .no_toc }

---

[Fulcrum](https://github.com/cculianu/Fulcrum){:target="_blank"} is a fast & nimble SPV server for Bitcoin Cash, Bitcoin BTC, and Litecoin created by Calin Culianu. It can be used as an alternative to Electrs because of its performance, as we can see in Craig Raw's [comparison](https://www.sparrowwallet.com/docs/server-performance.html){:target="_blank"} of servers

Difficulty: Medium
{: .label .label-yellow }

Status: Tested v3
{: .label .label-green }

![Fulcrum](../../../images/fulcrum.png)

---

Table of contents
{: .text-delta }

1. TOC
{:toc}

---

## Requirements

* Bitcoin Core
* Little over 100GB of free storage for database (external backup recommended)

---

Fulcrum is a replacement for an Electrs, these two services cannot be run at the same time (due to the same standard ports used), remember to stop Electrs doing "sudo systemctl stop electrs". Performance issues have been found on Raspberry Pi 4GB, it is recommended to install Fulcrum on 8GB RAM version.

## Preparations

Make sure that you have [reduced the database cache of Bitcoin Core](../../bitcoin/bitcoin-client.md#reduce-dbcache-after-full-sync)

### Install dependencies

* With user "admin", make sure that all necessary software packages are installed

  ```sh
  $ sudo apt install libssl-dev
  ```

### Configure Firewall

* Configure the firewall to allow incoming requests

  ```sh
  $ sudo ufw allow 50002/tcp comment 'allow Fulcrum SSL'
  $ sudo ufw allow 50001/tcp comment 'allow Fulcrum TCP'
  ```

## Installation

### Download and set up Fulcrum

We have our Bitcoin Core configuration file set up and now we can move to next part - installation of Fulcrum

* Download the application, checksums and signature

  ```sh
  $ cd /tmp
  $ wget https://github.com/cculianu/Fulcrum/releases/download/v1.8.2/Fulcrum-1.8.2-arm64-linux.tar.gz
  $ wget https://github.com/cculianu/Fulcrum/releases/download/v1.8.2/Fulcrum-1.8.2-arm64-linux.tar.gz.asc
  $ wget https://github.com/cculianu/Fulcrum/releases/download/v1.8.2/Fulcrum-1.8.2-arm64-linux.tar.gz.sha256sum
  ```

* Get the public key from the Fulcrum developer

  ```sh
  $ curl https://raw.githubusercontent.com/Electron-Cash/keys-n-hashes/master/pubkeys/calinkey.txt | gpg --import
  ```

* Verify the signature of the text file containing the checksums for the application

  ```sh
  $ gpg --verify Fulcrum-1.8.2-arm64-linux.tar.gz.asc
  > gpg: Good signature from "Calin Culianu (NilacTheGrim) <calin.culianu@gmail.com>" [unknown]
  > gpg: WARNING: This key is not certified with a trusted signature!
  > gpg: There is no indication that the signature belongs to the owner.
  > Primary key fingerprint: D465 135F 97D0 047E 18E9  9DC3 2181 0A54 2031 C02C
  ```

* Verify the signed checksum against the actual checksum of your download

  ```sh
  $ sha256sum --check Fulcrum-1.8.2-arm64-linux.tar.gz.sha256sum
  > Fulcrum-1.8.2-arm64-linux.tar.gz: OK
  ```

* Install Fulcrum and check the correct installation requesting the version

  ```sh
  $ tar -xvf Fulcrum-1.8.2-arm64-linux.tar.gz
  $ sudo install -m 0755 -o root -g root -t /usr/local/bin Fulcrum-1.8.2-arm64-linux/Fulcrum Fulcrum-1.8.2-arm64-linux/FulcrumAdmin 
  $ Fulcrum --version
  > Fulcrum 1.8.2 (Release d330248)
  compiled: gcc 8.4.0
  ...
  ```

### Data directory

Now that Fulcrum is installed, we need to configure it to run automatically on startup.

* Create the "fulcrum" service user, and add it to "bitcoin" group

  ```sh
  $ sudo adduser --disabled-password --gecos "" fulcrum
  $ sudo adduser fulcrum bitcoin
  ```

* Create the fulcrum data directory

  ```sh
  $ sudo mkdir -p /data/fulcrum/fulcrum_db
  $ sudo chown -R fulcrum:fulcrum /data/fulcrum/
  ```

* Create a symlink to /home/fulcrum/.fulcrum

  ```sh
  $ sudo ln -s /data/fulcrum /home/fulcrum/.fulcrum
  $ sudo chown -R fulcrum:fulcrum /home/fulcrum/.fulcrum
  ```

* Open a "fulcrum" user session

  ```sh
  $ sudo su - fulcrum
  ```

* Change to fulcrum data folder and generate cert and key files for SSL. When it asks you to put some info, press `Enter` until the prompt is shown again, is not necessary to put any info

  ```sh
  $ cd /data/fulcrum
  $ openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key.pem -out cert.pem
  ```

### Configuration

RaspiBolt uses SSL as default for Fulcrum, but some wallets like [BlueWallet](https://bluewallet.io/) do not support SSL over Tor. Thats why we use TCP in configurations as well to let user choose what he needs. You may as well need to use TCP for other reasons.

* Next, we have to set up our Fulcrum configurations. Troubles could be found without optimizations for Raspberry Pi. Choose either one for Raspberry 4GB or 8GB depending on your hardware. Create the config file with the following content. Save and exit

  ```sh
  $ nano /data/fulcrum/fulcrum.conf
  ```

  ```sh
  # RaspiBolt: fulcrum configuration 
  # /data/fulcrum/fulcrum.conf
  
  # Bitcoin Core settings
  bitcoind = 127.0.0.1:8332
  rpccookie = /home/bitcoin/.bitcoin/.cookie
  
  # Fulcrum server settings
  datadir = /data/fulcrum/fulcrum_db
  cert = /data/fulcrum/cert.pem
  key = /data/fulcrum/key.pem
  ssl = 0.0.0.0:50002
  tcp = 0.0.0.0:50001
  peering = false
  
  fast-sync = 5120
  ```

* Exit "fulcrum" user session to return to "admin" user session

  ```sh
  $ exit
  ```

### Autostart on boot

Fulcrum needs to start automatically on system boot.

* As user "admin", create the Fulcrum systemd unit and copy/paste the following configuration. Save and exit

  ```sh
  $ sudo nano /etc/systemd/system/fulcrum.service
  ```

  ```sh
  # RaspiBolt: systemd unit for Fulcrum
  # /etc/systemd/system/fulcrum.service
  
  [Unit]
  Description=Fulcrum
  PartOf=bitcoind.service
  After=bitcoind.service
  StartLimitBurst=2
  StartLimitIntervalSec=20
  
  [Service]
  ExecStart=/usr/local/bin/Fulcrum /data/fulcrum/fulcrum.conf
  KillSignal=SIGINT
  User=fulcrum
  Type=exec
  LimitNOFILE=8192
  TimeoutStopSec=300
  RestartSec=30
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target
  ```

### Run Fulcrum

* Enable fulcrum service and start

  ```sh
  $ sudo systemctl enable fulcrum.service
  $ sudo systemctl start fulcrum.service
  ```

* We can check if everything goes right using these commands

  ```sh
  $ sudo systemctl status fulcrum
  $ sudo journalctl -f -u fulcrum
  ```

* Expected output:

  ```sh
  -- Journal begins at Mon 2022-04-04 16:41:41 CEST. --
  Jul 28 12:20:13 rasp Fulcrum[181811]: [2022-07-28 12:20:13.063] simdjson: version 0.6.0
  Jul 28 12:20:13 rasp Fulcrum[181811]: [2022-07-28 12:20:13.063] ssl: OpenSSL 1.1.1n  15 Mar 2022
  Jul 28 12:20:13 rasp Fulcrum[181811]: [2022-07-28 12:20:13.063] zmq: libzmq version: 4.3.3, cppzmq version: 4.7.1
  Jul 28 12:20:13 rasp Fulcrum[181811]: [2022-07-28 12:20:13.064] Fulcrum 1.8.2 (Release d330248) - Thu Jul 28, 2022 12:20:13.064 CEST - starting up ...
  Jul 28 12:20:13 rasp Fulcrum[181811]: [2022-07-28 12:20:13.064] Max open files: 524288 (increased from default: 1024)
  Jul 28 12:20:13 rasp Fulcrum[181811]: [2022-07-28 12:20:13.065] Loading database ...
  Jul 28 12:20:14 rasp Fulcrum[181811]: [2022-07-28 12:20:14.489] DB memory: 512.00 MiB
  Jul 28 12:20:14 rasp Fulcrum[181811]: [2022-07-28 12:20:14.491] Coin: BTC
  Jul 28 12:20:14 rasp Fulcrum[181811]: [2022-07-28 12:20:14.492] Chain: main
  Jul 28 12:20:14 rasp Fulcrum[181811]: [2022-07-28 12:20:14.494] Verifying headers ...
  Jul 28 12:20:19 rasp Fulcrum[181811]: [2022-07-28 12:20:19.780] Initializing header merkle cache ...
  Jul 28 12:20:21 rasp Fulcrum[181811]: [2022-07-28 12:20:21.643] Checking tx counts ...
  ...
  ```

Fulcrum will now index the whole Bitcoin blockchain so that it can provide all necessary information to wallets. With this, the wallets you use no longer need to connect to any third-party server to communicate with the Bitcoin peer-to-peer network.

DO NOT REBOOT OR STOP THE SERVICE DURING DB CREATION PROCESS. YOU MAY CORRUPT THE FILES - in case of that happening, start sync from scratch by deleting and recreating `fulcrum_db` folder.

💡 Fulcrum must first fully index the blockchain and compact its database before you can connect to it with your wallets. This can take up to ~3.5 - 4 days. Only proceed with the [Desktop Wallet Section](../../bitcoin/desktop-wallet.md) once Fulcrum is ready.

## Extras

### Remote access over Tor (optional)

To use your Fulcrum server when you're on the go, you can easily create a Tor hidden service.
This way, you can connect the BitBoxApp or Electrum wallet also remotely, or even share the connection details with friends and family. Note that the remote device needs to have Tor installed as well.

* Ensure that you are logged with user "admin" and add the following three lines in the section for "location-hidden services" in the torrc file. Save and exit

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

* Edit torrc

  ```sh
  ############### This section is just for location-hidden services ###
  # Hidden Service Fulcrum SSL
  HiddenServiceDir /var/lib/tor/hidden_service_fulcrum_ssl/
  HiddenServiceVersion 3
  HiddenServicePort 50002 127.0.0.1:50002
  
  # Hidden Service Fulcrum TCP
  HiddenServiceDir /var/lib/tor/hidden_service_fulcrum_tcp/
  HiddenServiceVersion 3
  HiddenServicePort 50001 127.0.0.1:50001
  ```

* Reload Tor configuration and get your connection address

  ```sh
  $ sudo systemctl reload tor
  ```
  ```sh
  $ sudo cat /var/lib/tor/hidden_service_fulcrum_ssl/hostname
  > abcdefg..............xyz.onion
  ```
  ```sh
  $ sudo cat /var/lib/tor/hidden_service_fulcrum_tcp/hostname
  > abcdefg..............xyz.onion
  ```

* You should now be able to connect to your Fulcrum server remotely via Tor using SSL or TCP.

### Add banner to Fulcrum server (For fun!)

You can get creative when making your server banner, for example creating your own [ASCII art](https://patorjk.com/software/taag/#p=display&f=Slant&t=Fulcrum). In [Fulcrum docs](https://github.com/cculianu/Fulcrum/blob/master/doc/fulcrum-example-config.conf) you can find additional info about making a banner in the "# Server banner text file - 'banner'" section.

* Create and open `banner.txt` file inside Fulcrum directory

  ```sh
  $ sudo nano /data/fulcrum/banner.txt
  ```

* Paste your own creation into `banner.txt`. Save and exit

  ```sh
      ____      __
     / __/_  __/ /___________  ______ ___
    / /_/ / / / / ___/ ___/ / / / __ `__ \
   / __/ /_/ / / /__/ /  / /_/ / / / / / /
  /_/  \__,_/_/\___/_/   \__,_/_/ /_/ /_/
  
  server version: $SERVER_VERSION
  bitcoind version: $DAEMON_VERSION
  ```

* Open `fulcrum.conf`

  ```sh
  $ sudo nano /data/fulcrum/fulcrum.conf
  ```

* Specify path to banner at the end of your configuration file. Save and exit

  ```sh
  # Banner path
  banner = /data/fulcrum/banner.txt
  ```

* Restart Fulcrum

  ```sh
  $ sudo systemctl restart fulcrum.service
  ```

Now you should see your banner when connecting to Fulcrum with supported wallet (ex. Sparrow)

![Banner](../../../images/Fulcrum_Banner.png)

### Configure BTC RPC Explorer to Fulcrum API connection and modify the service

To get address balances, either an Electrum server or an external service is necessary. Your local Fulcrum server can provide address transaction lists, balances, and more.

* Change to `btcrpcexplorer` user, enter to `btc-rpc-explorer` folder and open `.env` file

  ```sh
  $ sudo su - btcrpcexplorer
  $ cd btc-rpc-explorer
  $ nano .env
  ```

* Add or modify the following line. Save and exit

  ```sh
  BTCEXP_ELECTRUM_SERVERS=tls://127.0.0.1:50002
  ```

* Return to `admin` user by exiting and open `btcrpcexplorer` service

  ```sh
  $ exit
  $ sudo nano /etc/systemd/system/btcrpcexplorer.service
  ```

* Replace `"After=electrs.service"` to `"After=fulcrum.service"` parameter. Save and exit

  ```sh
  After=fulcrum.service
  ```

* Restart BTC RPC Explorer service to apply the changes

  ```sh
  $ sudo systemctl restart btcrpcexplorer
  ```

### Backup the database

If the database gets corrupted and you don't have a backup, you will have to resync it from scratch, which takes several days. This is why we recommend to make backups of the database once in a while, on an external drive. Like this, if something happens, you'll only have to resync since the date of your latest backup. Before doing the backup, remember to stop Fulcrum doing `"sudo systemctl stop fulcrum"`.

## For the future: Fulcrum upgrade

* As “admin” user, stop the Fulcrum service

  ```sh
  $ sudo systemctl stop fulcrum
  ```

* Download, verify and install the latest Fulcrum binaries as described in the [Fulcrum section](fulcrum.md#download-and-set-up-fulcrum) of this guide

## Uninstall

### Uninstall Fulcrum

Ensure you are logged with user "admin"

* Stop, disable and delete the service

  ```sh
  $ sudo systemctl stop fulcrum
  $ sudo systemctl disable fulcrum
  $ sudo rm /etc/systemd/system/fulcrum.service
  ```

* Delete "fulcrum" user

  ```sh
  $ sudo userdel -r fulcrum
  ```

* Delete fulcrum directory

  ```sh
  $ sudo rm -rf /data/fulcrum/
  ```

### Uninstall Tor hidden service

* Comment or remove fulcrum hidden service in torrc. Save and exit

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```sh
  ############### This section is just for location-hidden services ###
  # Hidden Service Fulcrum SSL
  #HiddenServiceDir /var/lib/tor/hidden_service_fulcrum/
  #HiddenServiceVersion 3
  #HiddenServicePort 50002 127.0.0.1:50002
  
  # Hidden Service Fulcrum TCP
  #HiddenServiceDir /var/lib/tor/hidden_service_fulcrum_tcp/
  #HiddenServiceVersion 3
  #HiddenServicePort 50001 127.0.0.1:50001
  ```

* Reload torrc config

  ```sh
  $ sudo systemctl reload tor
  ```

### Uninstall FW configuration

* Display the UFW firewall rules and notes the numbers of the rules for Fulcrum (e.g., X and Y below)

  ```sh
  $ sudo ufw status numbered
  > [...]
  > [X] 50002                   ALLOW IN    Anywhere                   # allow Fulcrum SSL
  > [...]
  > [X] 50002 (v6)              ALLOW IN    Anywhere (v6)              # allow Fulcrum SSL
  > [...]
  > [Y] 50001                   ALLOW IN    Anywhere                   # allow Fulcrum TCP
  > [...]
  > [Y] 50001 (v6)              ALLOW IN    Anywhere (v6)              # allow Fulcrum TCP
  ```

* Delete the rule with the correct number and confirm with "yes"

  ```sh
  $ sudo ufw delete X
  ```

<br /><br />

---

<< Back: [+ Bitcoin](index.md)
