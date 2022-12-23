[![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/IndrekHaav/nightscout-debian/link-check.yml?branc=main&label=link-check)](https://github.com/IndrekHaav/nightscout-debian/actions/workflows/link-check.yml)

# Installing Nightscout on Debian

## Background

### What is Nightscout?

Nightscout is a self-hosted, web-based application that allows storing, visualising and sharing blood glucose and insulin treatment data from a variety of CGM (continuous glucose monitoring) systems. For more information, see <https://nightscout.github.io/>.

### Why this guide?

The recommended way to install Nightscout is to make use of free cloud services like [Heroku](https://www.heroku.com/) and [MongoDB Atlas](https://www.mongodb.com/atlas/database). However, this leaves your data and the availability of the application itself at the hands of those providers. In the spirit of decentralisation and owning your data, this guide aims to provide instructions to install Nightscout on any server running Linux (specifically, [Debian](https://www.debian.org/)).

Existing guides for running Nightscout on self-hosted platforms include:

 - [Nightscout on Docker](https://github.com/nightscout/nightscout-docker)
 - [Nightscout on Windows Server](https://github.com/jaylagorio/Nightscout-on-Windows-Server)
 - [Nightscout on Arch Linux](https://github.com/schmitzn/howto-nightscout-linux)

## Before you start

### Requirements

The following things are needed to set up Nightscout using this guide:

 - General Linux knowledge and the ability to manage a Debian-based system using a terminal. Although the guide contains commands you can copy&paste, it is advised that you understand what those commands do in principle.
 - A Debian-based server. This can in principle be pretty much any computer, a Raspberry Pi, a virtual machine, a VPS on a cloud provider, etc. The guide has been written for and tested on Debian 11 "Bullseye", the latest release as of the writing of this, but should also work (possibly with minor modifications) on older versions like Debian 10 "Buster", or Debian derivatives like Ubuntu or Raspbian.
 - The machine should have at least 1 GB of RAM, and potentially 2 GB is required to actually install Nightscout. This might rule out some options like Raspberry Pis and cheap VPS plans. However, with a VPS it should be possible to temporarily add more RAM when running the installation steps, and then reduce it afterwards.

### What this guide doesn't cover

This guide covers the steps needed to get Nightscout up and running, to the point of being able to load the web interface. It does **not** cover the following:

 - General Linux setup. It is assumed that you have SSH access to a functional Debian-based system, with a user account that has sudo rights. There are many guides for achieving this; [here is one example](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-debian-11).
 - Configuring Nightscout, including getting it to work with your particular CGM system. For that, see the official documentation at <https://nightscout.github.io/>.
 - Properly securing your Nightscout installation. Particularly when making Nightscout accessible to the entire internet (as opposed to running it on your home network only), you'll probably want to use TLS and add an authentication layer in front of Nightscout.

### DISCLAIMER

This guide is provided in good faith and for informational purposes only. No claims are made or guarantees given that it will work on any particular combination of hardware and software, or that it will be kept up-to-date with new releases of Nightscout. You will assume all responsibility for managing your Nightscout server, including the prevention of unauthorised access and safeguards against data loss.

## Installing Nightscout

### Prerequisites

A few packages need to be installed first:

```shell
$ sudo apt update
$ sudo apt install -y git gnupg
```

[Git](https://git-scm.com/) is a version control system, and is used to download the Nightscout source code from Github. [GnuPG](https://gnupg.org/) is used to manage cryptographic keys, and is needed for the following steps.

The remaining dependencies need to be installed separately, they should not be installed from the operating system's own package repository as the versions there are probably out-of-date.

#### Node.js

[Node.js](https://nodejs.org/) is a Javascript-based platform for building web applications. There are several ways to install Node.js on a Linux server, but the easiest way is to use pre-built packages from [Nodesource](https://github.com/nodesource/distributions#deb):

```shell
$ wget https://deb.nodesource.com/setup_14.x -O node_setup.sh
$ chmod +x node_setup.sh
$ sudo ./node_setup.sh
$ sudo apt update
$ sudo apt install -y nodejs
```

You can check that the correct version was installed by running `node -v`, it should report the version of Node.js installed on your system.

#### MongoDB

[MongoDB](https://www.mongodb.com/) is the database engine that Nightscout uses to store data. Once again there are several ways to install it, and once again the easiest is to use pre-built packages from [MongoDB's own repository](https://docs.mongodb.com/v4.4/tutorial/install-mongodb-on-debian/):

```shell
$ wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
$ echo "deb http://repo.mongodb.org/apt/debian buster/mongodb-org/4.4 main" | sudo tee /etc/apt/sources.list.d/mongodb.list
$ sudo apt update
$ sudo apt install -y mongodb-org
```

> **Note:** MongoDB doesn't, as of the writing of this, have a separate repository for Debian 11. If and when it is added, "buster" should be replaced with "bullseye" in the above commands. For Ubuntu, the commands might be a little different; consult the [official docs](https://docs.mongodb.com/v4.4/tutorial/install-mongodb-on-ubuntu/).

Then run the following commands to start the MongoDB service and to enable it to start automatically on boot:

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl start mongod
$ sudo systemctl enable mongod
```

Finally, connect to MongoDB and create a new database and user for Nightscout:

```shell
$ mongo
> use nightscout
> db.createUser({user: "nightscout", pwd: "password", roles:["readWrite"]})
> quit()
```

Replace "password" with a strong password, ideally randomly generated and stored in a password manager. In any case, make a note of the password, you will need it later when configuring Nightscout.

### System setup

Instead of running Nightscout as root or your own user, it is advisable to create a separate user account for it:

```shell
$ sudo useradd --system -m -d /opt/nightscout -s /bin/bash nightscout
```

This will assume that all Nightscout-related files go to `/opt/nightscout`. If you would like to use another location, change it in the above and following commands.

### Download and install Nightscout

Now switch to the newly-added user account and download the Nightscout source code:

```shell
$ sudo -u nightscout -i
$ git clone https://github.com/nightscout/cgm-remote-monitor.git
```

If you have forked the Nightscout repo on Github, then replace the URL above with that of your own repository.

Next, install Nightscout:

```shell
$ cd cgm-remote-monitor
$ npm install
```

> **Note:** It is this step that might fail on systems with less than 2 GB of RAM.

### Configure Nightscout

Still in the `cgm-remote-monitor` directory, create a file for Nightscout configuration parameters:

```shell
$ nano .env
```

This opens the file in the [Nano](https://www.nano-editor.org/) text editor. Feel free to use another editor if you have a preference, but this guide will assume Nano.

Add the following contents:

```
BASE_URL="http://YOUR-IP-HERE:1337"
INSECURE_USE_HTTP=true
API_SECRET="API-SECRET-HERE"
MONGODB_URI="mongodb://nightscout:MONGODB-PASSWORD-HERE@localhost:27017/nightscout"
DISPLAY_UNITS="mmol/L"
```

Replace the following:
 - "YOUR-IP-HERE" with the IP address of the server you're installing Nightscout on.
 - "API-SECRET-HERE" with a passphrase. Make it a good one and store it securely, as knowing this will give full access to your Nightscout data.
 - "MONGODB-PASSWORD-HERE" with the password you created above for the MongoDB user.
 - "mmol/L" with "mg/dl" if that's your preferred unit for glucose measurements.

> **Note:** The above is just the bare minimum, consult the [official docs](https://github.com/nightscout/cgm-remote-monitor#environment) for a full list of configuration parameters.

Press `Ctrl+O` and `Enter` to save, then `Ctrl+X` to exit Nano. Now enter the following command:

```shell
$ chmod 640 .env
```

This ensures that the file cannot be read by other users on the system, as it contains sensitive details.

Finally, enter the following command to start Nightscout:

```shell
$ export $(cat .env | xargs) && node server.js
```

Try opening `http://YOUR-IP-HERE:1337` in a web browser. If all went well, you should see the Nightscout interface load. Congratulations! Just a few more steps to go.

Back in the terminal, press `Ctrl+C` to stop the server. We will now set up a way to have Nightscout run automatically in the background.

### System service

There's nothing else to do as the Nightscout user, so type `exit` to switch back to your own user account. Then create a file for the service definition:

```shell
$ sudo nano /etc/systemd/system/nightscout.service
```

Add the following contents:

```
[Unit]
Description=Nightscout CGM service
After=network.target
After=mongod.service

[Service]
Type=simple
User=nightscout
Group=nightscout
WorkingDirectory=/opt/nightscout/cgm-remote-monitor
EnvironmentFile=/opt/nightscout/cgm-remote-monitor/.env
ExecStart=/usr/bin/node server.js

[Install]
WantedBy=multi-user.target
```

> **Note:** If you used a directory other than `/opt/nightscout`, change the above accordingly.

Now run the following commands to start the service and to have it start automatically on every boot:

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl start nightscout
$ sudo systemctl enable nightscout
```

If all went well, you should be able to open the web UI again at `http://YOUR-IP-HERE:1337`. You can also check that the service is running with the following command:

```shell
$ systemctl status nightscout
```

That's it! You can now start configuring Nightscout to work with your CGM system.

## Updating Nightscout

New versions of Nightscout are published to their Github repository: <https://github.com/nightscout/cgm-remote-monitor/releases>

It is a good idea to keep an eye on that page, in order to keep your Nightscout installation up-to-date. If a new version is published, the following steps need to be done.

First, stop the Nightscout service:

```shell
$ sudo systemctl stop nightscout
```

Then, switch to the Nightscout user, pull the latest version from Github and reinstall:

```shell
$ sudo -u nightscout -i
$ cd cgm-remote-monitor
$ git pull --force
$ npm install
```

Finally, type `exit` to switch back to your own user account, and start the Nightscout service:

```shell
$ sudo systemctl start nightscout
```

## Backing up Nightscout

While a comprehensive overview of backing up a Linux server is outside the scope of this guide, it should be noted that the following locations are important to Nightscout and should be backed up:

 - `/opt/nightscout/cgm-remote-monitor/.env` (contains the configuration parameters)
 - `/var/lib/mongodb/` (contains all data stored by MongoDB)
