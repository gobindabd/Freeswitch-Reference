# FreeSWITCH Installation on Debian 11/12 (with mod_spandsp and Admin Token Setup)

This README provides a complete, step-by-step guide for building and installing FreeSWITCH from source on **Debian 11 or Debian 12**—including fax/TDD support via `mod_spandsp` (with a patch), and instructions for admin token (CLI authentication) collection and usage.

---

## Table of Contents

- [About](#about)
- [Features](#features)
- [Included Services](#included-services)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation Steps](#installation-steps)
  - [Patching mod_spandsp](#patching-mod_spandsp)
  - [Admin Token Collection](#admin-token-collection)
- [Usage](#usage)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)
- [Acknowledgments](#acknowledgments)

---

## About

This guide is built to help you compile and run FreeSWITCH (v1.10 or compatible) on modern Debian servers with full fax/TDD (mod_spandsp) support and secure CLI access.

---

## Features

- Complete FreeSWITCH build for Debian 11/12
- mod_spandsp fax/TDD support (patched for current Debian spandsp libraries)
- Admin token setup and usage for secure CLI
- Easy, reproducible steps

---

## Included Services

This build includes:

- Core FreeSWITCH engine
- SIP endpoint (`mod_sofia`)
- Conference bridge (`mod_conference`)
- Voicemail (`mod_voicemail`)
- Fax/TDD (`mod_spandsp`)
- IVR/dialplan tools (`mod_dptools`, `mod_expr`)
- Call recording/playback (`mod_recording`, `mod_playback`)
- Music on Hold (`mod_moh`)
- And more—customizable via `modules.conf`

---

## Getting Started

### Prerequisites

- Debian 11 or 12 (fresh is best)
- Root or sudo privileges
- Internet access

---

### Installation Steps

#### 1. Update System and Install Dependencies

```bash
apt-get update
apt-get upgrade -y
apt-get install -y git build-essential autoconf automake libtool pkg-config libncurses5-dev libssl-dev libxml2-dev libsqlite3-dev libcurl4-openssl-dev libpcre3-dev uuid-dev zlib1g-dev libjpeg-dev libspandsp-dev libedit-dev yasm
```

---

#### 2. Download FreeSWITCH Source

```bash
cd /usr/src
git clone https://github.com/signalwire/freeswitch.git -b v1.10 --depth 1
cd freeswitch
```

---

#### 3. Bootstrap and Configure

```bash
./bootstrap.sh
./configure
```

---

### Patching mod_spandsp

The Debian spandsp package is not API-compatible with upstream FreeSWITCH.  
Patch the module as below **before building**:

1. **Edit the module source:**

   ```bash
   vim /usr/src/freeswitch/src/mod/applications/mod_spandsp/mod_spandsp_dsp.c
   ```

2. **Replace missing constants and fix v18_init arguments:**

   - Change any line like:
     ```c
     int r = V18_MODE_5BIT_4545;
     ```
     to:
     ```c
     int r = 0;
     ```
   - Change any line like:
     ```c
     r = V18_MODE_5BIT_50;
     ```
     to:
     ```c
     r = 0;
     ```
   - For each call to `v18_init`, add two `NULL` arguments at the end (make 8 arguments):
     ```c
     v18_init(..., ..., ..., ..., ..., ..., NULL, NULL);
     ```
     For example, change:
     ```c
     v18_init(NULL, TRUE, get_v18_mode(session), V18_AUTOMODING_GLOBAL, put_text_msg, NULL);
     ```
     to:
     ```c
     v18_init(NULL, TRUE, get_v18_mode(session), V18_AUTOMODING_GLOBAL, put_text_msg, NULL, NULL, NULL);
     ```
   - Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X` for nano).

---

#### 4. Build and Install FreeSWITCH

```bash
cd /usr/src/freeswitch
make clean
./bootstrap.sh
./configure
make
make install
make sounds-install moh-install
```

---

### Admin Token Collection

FreeSWITCH (1.10+) uses an **admin API token** for fs_cli and remote access.

**How to get your admin token:**

1. Find your `vars.xml` file (usually `/usr/local/freeswitch/conf/vars.xml`).

2. Look for the line:

   ```xml
   <X-PRE-PROCESS cmd="set" data="api_auth_token=YOURTOKENHERE"/>
   ```

   Or after first startup, check `/usr/local/freeswitch/log/freeswitch.log` for:
   ```
   Admin API auth token: YOURTOKENHERE
   ```

3. If not set, generate a random token and add a line in `vars.xml`:

   ```xml
   <X-PRE-PROCESS cmd="set" data="api_auth_token=your_very_secure_token"/>
   ```

4. Restart FreeSWITCH after changing the token.

---

## Usage

**Starting FreeSWITCH:**

```bash
/usr/local/freeswitch/bin/freeswitch -nc
```

**Connecting with fs_cli:**

```bash
fs_cli -a your_very_secure_token
```

**To load mod_spandsp (if not loaded automatically):**

```bash
fs_cli -x "load mod_spandsp"
```

**Check for the module:**

```bash
ls /usr/local/freeswitch/mod/ | grep spandsp
```

---

## Troubleshooting

- If build fails at mod_spandsp, re-check your edits for `mod_spandsp_dsp.c`.
- If `fs_cli` rejects your token, check for spaces/typos or restart FreeSWITCH after changing the token.
- If you don’t need fax/TDD, comment out `applications/mod_spandsp` in `modules.conf`.

---
## Contact

Maintained by: **Gobinda Paul**  
Email: gobinda@live.com

---
