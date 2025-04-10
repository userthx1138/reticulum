# Set up reticulum RNSD on Linux as a normal user, and run as a service
Adapted from: https://markqvist.github.io/Reticulum/manual/using.html#using-systemd

This is my guide to setting up **R**eticulum **N**ode **S**tack as a daemon service on a Linux based system.

This has now been tested on a RaspberryPI 3b (running Bookworm and fully updated as of Feb. 2025) and my Fedora41 based laptop. Setup on these systems was very easy; all the required modules are available pre-built in the relevant PIP repos.

However; I have now deployed my permanent setup on a MangoPI MQ-PRO running Ubuntu 24.04.2 LTS; this is attached (via USB) to a RNode and serving as a Lora/TCP transport node for my neighbourhood.
- The MQ PRO is a very simple machine with a single risc-v 64bit core running at 1GHz, and 1Gb ram. Setting up was tricky but it is now running `rnsd` very reliably, and autostarts the servie on reboots etc.
- See below (and the appendix) for notes on how I facilitated pip to build the cryptogrophy module needed by RNSD since no prebuilt binary wheel exists for RV64 (yet).

## Install RNSD to a dedicated user account.
### Create a user for the rnsd service
User is `rns`.
```console
user@pi:~ $ sudo adduser --disabled-password rns
Adding user `rns' ...
Adding new group `rns' (1002) ...
Adding new user `rns' (1002) with group `rns (1002)' ...
Creating home directory `/home/rns' ...
Copying files from `/etc/skel' ...
Changing the user information for rns
Enter the new value, or press ENTER for the default
	Full Name []: rns 
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n] y
Adding new user `rns' to supplemental / extra groups `users' ...
Adding user `rns' to group `users' ...

# Add any necessary groups, eg if you want to work with USB RNodes:
user@pi:~ $ sudo usermod -a -G dialout rns

# Install the dependencies needed for rnsd
user@pi:~ $ sudo apt install python3 python3-pip python3-cryptography python3-pyserial

# Now log in as the new user.
user@pi:~ $ sudo su - rns

rns@pi:~ $ id
uid=1002(rns) gid=1002(rns) groups=1002(rns),20(dialout),100(users)
-- The numeric UID/GID's shown may vary from system to system.

rns@pi:~ $ pwd
/home/rns
```
You now have a dedicated `rns` user, if you want you can set up a password or (like me) import keys for ssh login access; this can be useful for maintenance and log checking.
- The goal here is to run rnsd as a low-privilege user, so *do not* add this user to sudo or the `adm` group, or any other group that may compromise your system.

### Install rnsd in a python virtual env
The python virtual environment keeps our python install and it's dependencies seperate from the primary system python install. See: https://docs.python.org/3/library/venv.html
```console
rns@pi:~ $ python3 -m venv reticulum
rns@pi:~ $ . reticulum/bin/activate
```
- To exit the python virtual environment run `deactivate`.

We now install RNS via python; on a x86 (PC/Laptop/Mac) or ARM (Pi) platforms the cryptography module is pre-built and this process is simple and fast.
```console
(reticulum) rns@pi:~ $ pip install rns
Looking in indexes: https://pypi.org/simple, https://www.piwheels.org/simple
Collecting rns
  Downloading https://www.piwheels.org/simple/rns/rns-0.9.1-py3-none-any.whl (399 kB)
Collecting cryptography>=3.4.7 (from rns)
  Downloading https://www.piwheels.org/simple/cryptography/cryptography-44.0.0-cp37-abi3-linux_armv7l.whl (1.6 MB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 1.6/1.6 MB 3.3 MB/s eta 0:00:00
Collecting pyserial>=3.5 (from rns)
  Downloading https://www.piwheels.org/simple/pyserial/pyserial-3.5-py2.py3-none-any.whl (90 kB)
Collecting cffi>=1.12 (from cryptography>=3.4.7->rns)
  Downloading https://www.piwheels.org/simple/cffi/cffi-1.17.1-cp311-cp311-linux_armv7l.whl (384 kB)
Collecting pycparser (from cffi>=1.12->cryptography>=3.4.7->rns)
  Downloading https://www.piwheels.org/simple/pycparser/pycparser-2.22-py3-none-any.whl (117 kB)
Installing collected packages: pyserial, pycparser, cffi, cryptography, rns
Successfully installed cffi-1.17.1 cryptography-44.0.0 pycparser-2.22 pyserial-3.5 rns-0.9.1
```

ON OTHER PLATFORMS (eg risc-v at this time of writing) the cryptography module needs to be built. This requres installing Rust and building some extra dependencies and can be slow.
- You see this as the install failing with ` Installing build dependencies ... error: subprocess-exited-with-error`
- See [the appendix](#appendix-building-cryptography-module) for an example of the errors and steps to install rust and build dependencies so that python can compile the module on demand.

### Run `rnsd` to generate a default config in `~/.reticulum`
```console
(reticulum) rns@lilly:~ $ rnsd
[2025-02-09 17:36:42] [Notice] Could not load config file, creating default configuration file...
[2025-02-09 17:36:42] [Notice] Default config file created. Make any necessary changes in /home/rns/.reticulum/config and restart Reticulum if needed.
[2025-02-09 17:36:45] [Notice] Started rnsd version 0.9.1
^C
```

You can now edit the newly-created `.reticulum/config` as needed, run `rnsd --exampleconfig` for a full example, also see: https://markqvist.github.io/Reticulum/manual/interfaces.html

## Test config, comission rnodes
Test your config by running `rnsd -v` in the virtual environment.

## Checking status 
When `rnsd` is running you can check it's status by logging in as the reticulem user and running `~/reticulum/bin/rnstatus`; this will run the command in the virtual environment context.

Alternatively; log in as the rns user and run `. reticulum/bin/activate` (as above) to enter the virtual environment, the `reticulum/bin` folder is added to your path when you do this, and you can now use reticulum commands such as `rnstatus`, `rnodeconf` etc. easily.

Now is also a good time to set up rnodes etc. 

# Run as a service
Once you have rnsd running correctly you can set it up as a service to run in the background.

The reticulum documentation uses `systemd`'s new(ish) 'user' service file mechanism for this. However; the steps below use a more conventional approach and start the service as part of the main systemd startup mechanism; switching to the rns user only when the service starts.

## As `root` create a (user) service file:
`user@pi:~ $ sudo vi /etc/systemd/system/rnsd.service`
(OR use Nano, emacs, or whatever editor you use..)

This file should contain:
```console
[Unit]
Description=Reticulum Network Stack Daemon
After=default.target

[Service]
# If you run Reticulum on WiFi devices,
# or other devices that need some extra
# time to initialise, you might want to
# add a short delay before Reticulum is
# started by systemd:
# ExecStartPre=/bin/sleep 10
User=rns
Type=simple
Restart=always
RestartSec=3
ExecStart=/home/rns/reticulum/bin/rnsd --service

[Install]
WantedBy=default.target
```
Note that this differs from the 'user' service file given in the RNode documentation, this service file is started by the root systemd process, and has additional options to make it run as the `rns` user.

Tell systemd about the service file, enable and activate with:
```console
user@pi:~ $ sudo systemctl daemon-reload
user@pi:~ $ sudo systemctl enable --now rnsd.service
Created symlink /etc/systemd/system/default.target.wants/rnsd.service → /etc/systemd/system/rnsd.service.
user@pi:~ $ systemctl status rnsd.service
● rnsd.service - Reticulum Network Stack Daemon
     Loaded: loaded (/etc/systemd/system/rnsd.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-02-09 18:44:50 CET; 8s ago
   Main PID: 7024 (rnsd)
      Tasks: 12 (limit: 2040)
        CPU: 767ms
     CGroup: /system.slice/rnsd.service
             └─7024 /home/rns/reticulum/bin/python3 /home/rns/reticulum/bin/rnsd --service

Feb 09 18:44:50 lilly.easytarget.org systemd[1]: Started rnsd.service - Reticulum Network Stack Daemon.
```
The service will use `~rns/.reticulum/` for it's config, working data and logfiles.

For for more about using, configuring and running the service see:
https://markqvist.github.io/Reticulum/manual/using.html

# Maintain
When you want to maintain, upgrade or work on the config you must log in as the `rns` user and activate the virtual environment with `$ source reticulum/bin/activate`

To check the current status try:
```console
user@pi:~ $ sudo su - rns
rns@pi:~ $ source reticulum/bin/activate
(reticulum) rns@pi:~/.reticulum $ rnstatus
```

You must stop and start the service as needed when reconfiguring or maintaining it
- use `systemctl` as root for this..

To upgrade try
```console
user@pi:~ $ sudo systemctl stop rnsd.service
user@pi:~ $ sudo su - rns
rns@pi:~ $ source reticulum/bin/activate
(reticulum) rns@lilly:~ $ pip install --upgrade rns
Looking in indexes: https://pypi.org/simple, https://www.piwheels.org/simple
Requirement already satisfied: rns in ./reticulum/lib/python3.11/site-packages (0.9.1)
Requirement already satisfied: cryptography>=3.4.7 in ./reticulum/lib/python3.11/site-packages (from rns) (44.0.0)
Requirement already satisfied: pyserial>=3.5 in ./reticulum/lib/python3.11/site-packages (from rns) (3.5)
Requirement already satisfied: cffi>=1.12 in ./reticulum/lib/python3.11/site-packages (from cryptography>=3.4.7->rns) (1.17.1)
Requirement already satisfied: pycparser in ./reticulum/lib/python3.11/site-packages (from cffi>=1.12->cryptography>=3.4.7->rns) (2.22)
```
You can now do any other maintenance, check logs in .reticulum/logfile, etc

Test your changes by running `rnsd -v` on the commandline.

When finished, exit and restart the service.
```console
(reticulum) rns@lilly:~ $ exit
user@pi:~ $ sudo systemctl start rnsd.service
```
[TODO: work out correct /etc/sudoers systax to allow the `rns` user to stop/status/start the service]

# APPENDIX: BUILDING CRYPTOGRAPHY MODULE

If you see the `pip install rns` step fail like this:
```console
Collecting cryptography>=3.4.7 (from rns)
  Downloading cryptography-44.0.1.tar.gz (710 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 710.8/710.8 kB 774.4 kB/s eta 0:00:00
  Installing build dependencies ... error
  error: subprocess-exited-with-error
```
Look at the error reported after that, it is probably a message that the rust compiler is not available.
- There might also be missing dependencies; look at the error and search on that for solutions.
- If you let me know below I can help debug and add the error and resolution here too.

## Additional dependencies needed:
As root install libusb and libtffi headers:
```console
user@pi:~ $ sudo apt install libffi-dev libusb-dev python3-dev libudev-dev
```
- *Note: this list was done from memory+searching apt logs, and may be too broad, or may miss something, I'll update it if I become aware of errors.*

### Error that Rust is not available
Go to: https://www.rust-lang.org/tools/install

Run the `Rustup` command given there *as the rns user*
- This will install a local copy of rust to your user homedir, do not install this as 'root'.
- Accept the defaults when asked by the rustup installer.
- If rust is **not** available for your architecture I'm afraid you will not be able to run `rnsd`, sorry.

```console
(reticulum) rns@iris:~$ <command given on rustup site, eg: curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh>
info: downloading installer

Welcome to Rust!

This will download and install the official compiler for the Rust
programming language, and its package manager, Cargo.

Rustup metadata and toolchains will be installed into the Rustup
home directory, located at:

  /home/rns/.rustup

This can be modified with the RUSTUP_HOME environment variable.

The Cargo home directory is located at:

  /home/rns/.cargo

This can be modified with the CARGO_HOME environment variable.

The cargo, rustc, rustup and other commands will be added to
Cargo's bin directory, located at:

  /home/rns/.cargo/bin

This path will then be added to your PATH environment variable by
modifying the profile files located at:

  /home/rns/.profile
  /home/rns/.bashrc
You can uninstall at any time with rustup self uninstall and
these changes will be reverted.

Current installation options:


   default host triple: riscv64gc-unknown-linux-gnu
     default toolchain: stable (default)
               profile: default
  modify PATH variable: yes

1) Proceed with standard installation (default - just press enter)
2) Customize installation
3) Cancel installation
>

info: profile set to 'default'
info: default host triple is riscv64gc-unknown-linux-gnu
info: syncing channel updates for 'stable-riscv64gc-unknown-linux-gnu'
info: latest update on 2025-02-20, rust version 1.85.0 (4d91de4e4 2025-02-17)
info: downloading component 'cargo'
  8.8 MiB /   8.8 MiB (100 %)   1.5 MiB/s in  9s ETA:  0s
info: downloading component 'clippy'
  3.2 MiB /   3.2 MiB (100 %)   1.0 MiB/s in  4s ETA:  0s
info: downloading component 'rust-docs'
 18.2 MiB /  18.2 MiB (100 %)   1.4 MiB/s in 14s ETA:  0s
info: downloading component 'rust-std'
 25.4 MiB /  25.4 MiB (100 %)   1.7 MiB/s in 19s ETA:  0s
info: downloading component 'rustc'
 89.2 MiB /  89.2 MiB (100 %)   1.3 MiB/s in  1m  7s ETA:  0s
info: downloading component 'rustfmt'
  2.5 MiB /   2.5 MiB (100 %) 996.1 KiB/s in  3s ETA:  0s
info: installing component 'cargo'
  8.8 MiB /   8.8 MiB (100 %)   1.1 MiB/s in  8s ETA:  0s
info: installing component 'clippy'
  3.2 MiB /   3.2 MiB (100 %)   1.4 MiB/s in  3s ETA:  0s
info: installing component 'rust-docs'
 18.2 MiB /  18.2 MiB (100 %) 436.8 KiB/s in  2m 12s ETA:  0s
info: installing component 'rust-std'
 25.4 MiB /  25.4 MiB (100 %)   1.4 MiB/s in 27s ETA:  0s
info: installing component 'rustc'
 89.2 MiB /  89.2 MiB (100 %)   1.1 MiB/s in  1m 30s ETA:  0s
info: installing component 'rustfmt'
  2.5 MiB /   2.5 MiB (100 %)   1.5 MiB/s in  4s ETA:  0s
info: default toolchain set to 'stable-riscv64gc-unknown-linux-gnu'

  stable-riscv64gc-unknown-linux-gnu installed - rustc 1.85.0 (4d91de4e4 2025-02-17)


Rust is installed now. Great!

To get started you may need to restart your current shell.
This would reload your PATH environment variable to include
Cargo's bin directory ($HOME/.cargo/bin).

To configure your current shell, you need to source
the corresponding env file under $HOME/.cargo.

This is usually done by running one of the following (note the leading DOT):
. "$HOME/.cargo/reticulum"            # For sh/bash/zsh/ash/dash/pdksh
source "$HOME/.cargo/reticulum.fish"  # For fish

(reticulum) rns@iris:~$ . "$HOME/.cargo/env"
(reticulum) rns@iris:~$ rustc --version
rustc 1.85.0 (4d91de4e4 2025-02-17)
```
This example was done on a Allwinner D1 powered risc-v system running Ubuntu 24.04. The subsequent install of RNS looks like:
```console
(reticulum) rns@mqpro:~$ pip install rns
Collecting rns
  Downloading rns-0.9.2-py3-none-any.whl.metadata (21 kB)
Collecting cryptography>=3.4.7 (from rns)
  Downloading cryptography-44.0.1.tar.gz (710 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 710.8/710.8 kB 985.8 kB/s eta 0:00:00
  Installing build dependencies ... done
  Getting requirements to build wheel ... done
  Preparing metadata (pyproject.toml) ... done
Collecting pyserial>=3.5 (from rns)
  Downloading pyserial-3.5-py2.py3-none-any.whl.metadata (1.6 kB)
Collecting cffi>=1.12 (from cryptography>=3.4.7->rns)
  Using cached cffi-1.17.1-cp312-cp312-linux_riscv64.whl
Collecting pycparser (from cffi>=1.12->cryptography>=3.4.7->rns)
  Using cached pycparser-2.22-py3-none-any.whl.metadata (943 bytes)
Downloading rns-0.9.2-py3-none-any.whl (400 kB)
Downloading pyserial-3.5-py2.py3-none-any.whl (90 kB)
Using cached pycparser-2.22-py3-none-any.whl (117 kB)
Building wheels for collected packages: cryptography
  Building wheel for cryptography (pyproject.toml) ... done
  Created wheel for cryptography: filename=cryptography-44.0.1-cp37-abi3-linux_riscv64.whl size=1661987 sha256=46987985840b79d0d822e2f18db4191de4b04f3893a3494b1a47eb10b509e50d
  Stored in directory: /home/rns/.cache/pip/wheels/26/4a/06/7c6ef086a56fcd8fcb74368e21438bb87f02a03fad39255377
Successfully built cryptography
Installing collected packages: pyserial, pycparser, cffi, cryptography, rns
Successfully installed cffi-1.17.1 cryptography-44.0.1 pycparser-2.22 pyserial-3.5 rns-0.9.2
(reticulum) rns@mqpro:~$ rnsd -v
[2025-02-25 20:54:51] [Notice]   Could not load config file, creating default configuration file...
[2025-02-25 20:54:51] [Notice]   Default config file created. Make any necessary changes in /home/rns/.reticulum/config and restart Reticulum if needed.
[2025-02-25 20:54:52] [Verbose]  Bringing up system interfaces...
[2025-02-25 20:54:52] [Verbose]  AutoInterface[Default Interface] discovering peers for 1.5 seconds...
[2025-02-25 20:54:54] [Verbose]  System interfaces are ready
[2025-02-25 20:54:54] [Verbose]  Configuration loaded from /home/rns/.reticulum/config
[2025-02-25 20:54:54] [Verbose]  Destinations file does not exist, no known destinations loaded
[2025-02-25 20:54:54] [Verbose]  No valid Transport Identity in storage, creating...
[2025-02-25 20:54:54] [Verbose]  Identity keys created for <SNIPSNIPSNIPSNIPSNIPSNIPSNIPSNIP>
[2025-02-25 20:54:54] [Notice]   Started rnsd version 0.9.2
```
The cryptography dependencies are all compiled with Rust, and will take some time.
- Unfortunately no progress is shown; you can use `top` to check that the cargo (rust compiler tool) processes are running.
- This is *reasonably* fast when compiled on a multi core system with plenty of RAM and a fully optomised compiler.
- But.. it took 5 hours to complete using a 1Ghz single core risc-v cpu with just 1Gb of ram (ManoPi MQ Pro).
- YMMV
