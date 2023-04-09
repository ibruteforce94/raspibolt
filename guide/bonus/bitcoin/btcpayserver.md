---
layout: default
title: BTCPay Server
parent: + Bitcoin
grand_parent: Bonus Section
nav_exclude: true
has_toc: false
---

# Bonus guide: BTCPay Server
{: .no_toc }

[BTCPay Server](https://btcpayserver.org/){:target="_blank"} is a self-hosted, open-source cryptocurrency payment processor.

Difficulty: Intermediate
{: .label .label-yellow }

Status: Tested v3
{: .label .label-green }

![btcpay](../../../images/btcpay.jpeg)

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Preparations

### Install PostgreSQL as admin

PostgreSQL is used as data storage. You can install it with:

* Install PostgreSQL

  ```sh
  $ sudo apt -y install postgresql postgresql-contrib
  ```

* Create PostgreSQL database for NBXplorer (change the default password and write it down, you will need it later)

  ```sh
  $ sudo -u postgres psql
  
  postgres=# CREATE DATABASE nbxplorer TEMPLATE 'template0' LC_CTYPE 'C' LC_COLLATE 'C' ENCODING 'UTF8';
  CREATE USER nbxplorer WITH ENCRYPTED PASSWORD 'urpassword';
  GRANT ALL PRIVILEGES ON DATABASE nbxplorer TO nbxplorer;
  
  postgres=# \q
  ```
  
* Create PostgreSQL database for BTCPayServer (change the default password and write it down, you will need it later)

  ```sh
  $ sudo -u postgres psql
  
  postgres=# CREATE DATABASE btcpay TEMPLATE 'template0' LC_CTYPE 'C' LC_COLLATE 'C' ENCODING 'UTF8';
  CREATE USER btcpay WITH ENCRYPTED PASSWORD 'urpassword';
  GRANT ALL PRIVILEGES ON DATABASE btcpay TO btcpay;
  
  postgres=# \q
  ```

### Install .NET 6.0 SDK as admin

This software is used to build and execute BTCPay Server.

* Create /etc/apt/preferences.d/99microsoft-dotnet.pref

  ```sh
  $ sudo nano /etc/apt/preferences.d/99microsoft-dotnet.pref
  ```
  
* Inside the new created file (99microsoft-dotnet.pref) add below text

  ```sh
  Package: *
  Pin: origin "packages.microsoft.com"
  Pin-Priority: 1001
  ```
  
* Then, regular update & install

  ```sh
  $ sudo apt update
  $ sudo apt install dotnet-sdk-6.0
  ```

* Check .NET SDK 6.0 is correctly installed

  ```sh
  $ dotnet --version
  > 6.0.407
  ```

### Create a new user

We do not want to run BTCPay Server alongside bitcoind and lightningd because of security reasons. For that we will create a separate user and we will be running the code as the new user.

* Create a new user , add it to bitcoin and lightningd (or lnd) group and open a new session

  ```sh
  $ sudo adduser --disabled-password --gecos "" btcpay
  $ sudo adduser btcpay bitcoin
  $ sudo adduser btcpay lightningd
  $ sudo su - btcpay
  ```

## Install NBXplorer

[NBXplorer](https://github.com/dgarage/NBXplorer) is a minimalist UTXO tracker for HD Wallets, exploited by BTCPay Server.

Still with user "btcpay", we execute the following commands:

* Make src directory

  ```sh
  $ mkdir src
  $ cd src
  ```

* Build NBXplorer

  ```sh
  $ git clone https://github.com/dgarage/NBXplorer
  $ cd NBXplorer
  $ ./build.sh
  ```

### NBXplorer configuration

* Create the data folder and a new config file

  ```sh
  $ mkdir -p ~/.nbxplorer/Main
  $ cd ~/.nbxplorer/Main
  $ nano settings.config
  ```

* Insert the following lines in the configuration file (instead of the default password, insert the one you written down when making nbxplorer postgres db)

  ```sh
  btc.rpc.cookiefile=/home/bitcoin/.bitcoin/.cookie
  port=24445
  mainnet=1
  
  ### Database ###
  postgres=User ID=nbxplorer;Password=urpassword;Application Name=nbxplorer;MaxPoolSize=20;Host=localhost;Port=5432;Database=nbxplorer;
  ```

### Autostart NBXplorer on boot

* First, we go back to user "admin"

  ```sh
  $ exit
  ```

Then, we use systemd to execute NBXplorer on boot

* Create the configuration file in the Nano text editor and copy the following paragraph.
  Save and exit.

  ```sh
  $ sudo nano /etc/systemd/system/nbxplorer.service
  ```

  ```sh
  [Unit]
  Description=NBXplorer daemon
  Requires=bitcoind.service
  After=bitcoind.service
  
  [Service]
  ExecStart=/usr/bin/dotnet "/home/btcpay/src/NBXplorer/NBXplorer/bin/Release/net6.0/NBXplorer.dll" -c /home/btcpay/.nbxplorer/Main/settings.config
  User=btcpay
  Group=btcpay
  Type=simple
  PIDFile=/run/nbxplorer/nbxplorer.pid
  Restart=on-failure
  
  PrivateTmp=true
  ProtectSystem=full
  NoNewPrivileges=true
  PrivateDevices=true
  
  [Install]
  WantedBy=multi-user.target
  ```

* Enable the service, start it and check logging output.

  ```sh
  $ sudo systemctl enable nbxplorer
  $ sudo systemctl start nbxplorer
  $ sudo journalctl -f -u nbxplorer
  ```

## Install BTCPay Server

With user "btcpay", we execute the following commands:

* Build BTCPay

  ```sh
  $ cd src
  $ git clone https://github.com/btcpayserver/btcpayserver
  $ cd btcpayserver
  $ ./build.sh
  ```

* Create the data folder and a new config file

  ```sh
  $ mkdir -p ~/.btcpayserver/Main
  $ cd ~/.btcpayserver/Main
  $ nano settings.config
  ```

### BTCPay Server configuration

* Insert the following lines in the the configuration file (instead of the default passwords, insert the ones you written down when making nbxplorer and btcpay postgres db)

  ```sh
  network=mainnet
  port=23001
  bind=0.0.0.0
  chains=btc
  BTC.explorer.url=http://127.0.0.1:24445
  ### CLN setup string ###
  BTC.lightning=type=clightning;server=unix://home/lightningd/.lightning/bitcoin/lightning-rpc
  ### LND setup string will be different, lnd-rest for example ###
  ### BTC.lightning=type=lnd-rest;server=https://127.0.0.1:8080/;macaroonfilepath=~/.lnd/data/chain/bitcoin/mainnet/admin.macaroon;certthumbprint=<fingerprint> ###
  
  ### Database ###
  postgres=User ID=btcpay;Password=urpassword;Application Name=btcpayserver;Host=localhost;Port=5432;Database=btcpay;
  explorer.postgres=User ID=nbxplorer;Password=urpassword;Application Name=nbxplorer;MaxPoolSize=20;Host=localhost;Port=5432;Database=nbxplorer;
  ```

### Autostart BTCPay Server on boot

* First, we go back to user "admin":

  ```sh
  $ exit
  ```

Then, we use systemd to execute BTCPayServer on boot:

* Create the configuration file in the Nano text editor and copy the following paragraph.
  Save and exit.

  ```sh
  $ sudo nano /etc/systemd/system/btcpay.service
  ```

  ```sh
  [Unit]
  Description=BtcPayServer daemon
  Requires=nbxplorer.service
  After=nbxplorer.service
  
  [Service]
  ExecStart=/usr/bin/dotnet run --no-launch-profile --no-build -c Release -p "/home/btcpay/src/btcpayserver/BTCPayServer/BTCPayServer.csproj" -- $@
  User=btcpay
  Group=btcpay
  Type=simple
  PIDFile=/run/btcpay/btcpay.pid
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target
  ```

* Enable the service, start it and check logging output.

  ```sh
  $ sudo systemctl enable btcpay
  $ sudo systemctl start btcpay
  $ sudo journalctl -f -u btcpay
  ```


## Remote access over Tor

You can easily add a Tor hidden service on the RaspiBolt and access the BTCPay Server interface with the Tor browser from any device.

* Add the following three lines in the section for "location-hidden services" in the `torrc` file. Save and exit

  ```sh
  $ sudo nano /etc/tor/torrc
  ```

  ```ini
  ############### This section is just for location-hidden services ###
  # Hidden Service BTCPay
  HiddenServiceDir /var/lib/tor/hidden_service_btcpay/
  HiddenServiceVersion 3
  HiddenServicePort 80 127.0.0.1:23001
  ```

  Update Tor configuration changes and get your connection address.

  ```sh
  $ sudo systemctl reload tor
  $ sudo cat /var/lib/tor/hidden_service_btcpay/hostname
  > abcefg...................zyz.onion
  ```

With the Tor browser (link this), you can access this onion address from any device.

**Congratulations!**
You now have BTCPay Server running to manage Bitcoin and Lightning payments on your own node.

**This guide is in no way made by me, I just added some finishing touches, all thanks go to:**

https://github.com/raspibolt/raspibolt/blob/e5624bcf5d1c5fcc6fe3932dfdedd4d3159026f7/guide/bonus/lightning/btcpayserver.md

https://freedomnode.com/blog/how-to-setup-btc-and-lightning-payment-gateway-with-btcpayserver-on-linux-manual-install/

https://docs.btcpayserver.org/Deployment/ManualDeploymentExtended/
