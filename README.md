# FreeSWITCH Installation on Debian 11/12 (SignalWire Repo, mod_spandsp Patch)

This README describes how to build and install FreeSWITCH on Debian 11/12 using the official SignalWire repository and manual mod_spandsp patching, as well as all necessary dependencies, build steps, and secure admin setup.

---

## Table of Contents

- [About](#about)
- [Included Services](#included-services)
- [Installation Steps](#installation-steps)
  - [Set Up the Token](#set-up-the-token)
  - [Install Dependencies](#install-dependencies)
  - [Configure SignalWire Repository](#configure-signalwire-repository)
  - [Install Build Dependencies](#install-build-dependencies)
  - [Download and Build FreeSWITCH](#download-and-build-freeswitch)
  - [Patch mod_spandsp](#patch-mod_spandsp)
- [Systemd Service Example](#systemd-service-example)
- [Admin Token Setup](#admin-token-setup)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [License](#license)
- [Contact](#contact)

---

## About

This guide automates most of the modern FreeSWITCH installation process, combining the official SignalWire apt repository for binaries and dependencies with a manual source build and patch for the mod_spandsp fax module (required on Debian 11/12).

---

## Included Services

- Core FreeSWITCH switching engine
- SIP endpoint (`mod_sofia`)
- Conference bridge (`mod_conference`)
- Voicemail (`mod_voicemail`)
- Fax/TDD (`mod_spandsp`)
- IVR/dialplan tools (`mod_dptools`, `mod_expr`)
- Call recording/playback (`mod_recording`, `mod_playback`)
- Music on Hold (`mod_moh`)
- Customizable via `modules.conf` when building from source

---

## Installation Steps

### Set Up the Token

Set your SignalWire token as an environment variable (**replace with your own for security!**):

```bash
TOKEN=pat_LENXXXXXXXXXXXXXXXXXXXXXXX
```

### Install Dependencies

```bash
apt-get update
apt-get upgrade -y
apt-get install -y -yq gnupg2 wget lsb-release git build-essential autoconf automake libtool pkg-config \
libncurses5-dev libssl-dev libxml2-dev libsqlite3-dev libcurl4-openssl-dev \
libpcre3-dev uuid-dev zlib1g-dev libjpeg-dev libspandsp-dev libedit-dev yasm
```

### Configure SignalWire Repository

```bash
wget --http-user=signalwire --http-password=$TOKEN -O /usr/share/keyrings/signalwire-freeswitch-repo.gpg https://freeswitch.signalwire.com/repo/deb/debian-release/signalwire-freeswitch-repo.gpg

echo "machine freeswitch.signalwire.com login signalwire password $TOKEN" > /etc/apt/auth.conf
chmod 600 /etc/apt/auth.conf

echo "deb [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list

apt-get -yq update
```

### Install Build Dependencies

```bash
apt-get -yq build-dep freeswitch
```

### Download and Build FreeSWITCH

```bash
cd /usr/src/
git clone https://github.com/signalwire/freeswitch.git -bv1.10 freeswitch
cd freeswitch
git config pull.rebase true
./bootstrap.sh -j
./configure
make
make install
```

---

### Patch mod_spandsp

```bash
cd /usr/src/freeswitch/src/mod/applications/mod_spandsp/
wget https://raw.githubusercontent.com/zenthangplus/ansible-role-fsmrf/9a73a47bfa19a485ddfc10f496bfc2041594f552/files/mod_spandsp_dsp.c.patch

patch -p0 < mod_spandsp_dsp.c.patch
# When prompted:
# File to patch: mod_spandsp_dsp.c

cd /usr/src/freeswitch
make clean && ./bootstrap.sh && ./configure
make && make install
```

---

## Systemd Service Example

Create `/etc/systemd/system/freeswitch.service`:

```ini
[Service]
Type=forking
PIDFile=/usr/local/freeswitch/run/freeswitch.pid
PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /usr/local/freeswitch/run
ExecStartPre=/bin/chown freeswitch:daemon /usr/local/freeswitch/run
ExecStart=/usr/local/freeswitch/bin/freeswitch -ncwait -nonat
TimeoutSec=45s
Restart=always
WorkingDirectory=/usr/local/freeswitch/run
User=root
Group=root
LimitCORE=infinity
LimitNOFILE=100000
LimitNPROC=60000
LimitRTPRIO=infinity
LimitRTTIME=7000000
IOSchedulingClass=realtime
IOSchedulingPriority=2
CPUSchedulingPolicy=rr
CPUSchedulingPriority=89
UMask=0007

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
systemctl daemon-reload
systemctl start freeswitch
systemctl enable freeswitch
```

---

## Admin Token Setup

**After first start**, check `/usr/local/freeswitch/conf/vars.xml` or `/usr/local/freeswitch/log/freeswitch.log` for your admin API token, or set it manually in `vars.xml` like:

```xml
<X-PRE-PROCESS cmd="set" data="api_auth_token=YOURTOKEN"/>
```

Restart FreeSWITCH after changing.

---

## Usage

- Start FreeSWITCH (if not using systemd):  
  `/usr/local/freeswitch/bin/freeswitch -nc`
- CLI Access:  
  `fs_cli -a YOURTOKEN`
- Load mod_spandsp:  
  `fs_cli -x "load mod_spandsp"`

---

## Troubleshooting

- If patching `mod_spandsp` fails, ensure you specify `mod_spandsp_dsp.c` when prompted.
- If `fs_cli` authentication fails, check/restart after updating your token.
- For missing dependencies, rerun `apt-get build-dep freeswitch`.

---

## Contact

Maintained by: **Gobinda Paul**
Email: gobinda@live.com

