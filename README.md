# Zero BTC Screen

Bitcoin (or any other currency) stock price for your RPi Zero

![display](docs/display.jpg)

## Hardware

### Platform

* Raspberry Pi Zero W
* Raspberry Pi 3b+
* Raspberry Pi 4
* Any other modern RPi

### Supported displays

* Waveshare eInk types:
  * epd2in13v2
  * epd2in13v3
  * epd2in13bv3
  * epd2in7
  * epd3in7
* inkyWhat (Red, Black, White)
* Virtual (picture)

## Installation

### Pi Zero 1 notes

The Pi Zero 1 uses an ARMv6 CPU with only 512 MB RAM. Many Python packages do
not ship pre-built wheels for ARMv6, so you **must** install the heavy
dependencies through `apt` before using `uv` or `pip`:

```bash
sudo apt update
sudo apt install python3-numpy python3-pil python3-rpi.gpio python3-spidev
```

Skipping this step means `numpy` and `Pillow` will attempt to compile from
source, which can take an hour or more and may run out of memory.

---

### Steps

1. Turn on SPI via `sudo raspi-config`
    ```
    Interfacing Options -> SPI
    ```

2. Install system-level dependencies (required on Pi Zero 1)
    ```bash
    sudo apt update
    sudo apt install python3-numpy python3-pil python3-rpi.gpio python3-spidev
    ```

3. Install [uv](https://docs.astral.sh/uv/getting-started/installation/)
    ```bash
    curl -LsSf https://astral.sh/uv/install.sh | sh
    ```

4. Clone the repo
    ```bash
    git clone https://github.com/dr-mod/zero-btc-screen.git ~/zero-btc-screen
    cd ~/zero-btc-screen
    ```

5. Create a virtualenv that can see the system packages (numpy, Pillow, RPi.GPIO)
    ```bash
    uv venv --system-site-packages
    ```

6. Install project dependencies
    - For an **Inky pHAT / Inky wHAT** display:
      ```bash
      uv pip install -e ".[inky]"
      ```
    - For a **Waveshare** display, install the vendor driver first, then:
      ```bash
      git clone https://github.com/waveshare/e-Paper.git ~/e-Paper
      uv pip install ~/e-Paper/RaspberryPi_JetsonNano/python/
      uv pip install -e .
      ```
      For more information refer to: https://www.waveshare.com/wiki/2.13inch_e-Paper_HAT
    - For a **virtual / picture** output only:
      ```bash
      uv pip install -e .
      ```

7. Edit `configuration.cfg` to select your screen (see **Screen configuration** below)

8. Run it
    ```bash
    uv run python main.py
    ```


## Screen configuration

The application supports multiple types of e-ink screens, and an additional "picture" screen.

To configure which display(s) to use, configuration.cfg should be modified. In the following example an e-ink epd2in13v2
and "picture" screens are select:

```cfg
[base]
console_logs             : false
#logs_file               : /tmp/zero-btc-screen.log
dummy_data               : false
refresh_interval_minutes : 15
# Price pair from Coinbase e.g. BTC-EUR or ADA-GBP
currency                 : BTC-USD

# Enabled screens or devices
screens : [
    epd2in13v2
#    epd2in13v3
#    epd2in13bv3
#    epd2in7
#    epd3in7
    picture
#    inkyWhatRBW
  ]

# Configuration per screen
# This doesn't make any effect if screens are not enabled above
[epd2in13v2]
mode : candle

[epd2in13v3]
mode : candle

[epd2in13bv3]
mode : line

[epd2in7]
mode : candle

[epd3in7]
mode : candle

[picture]
filename : /home/pi/output.png

[inkyWhatRBW]
mode : candle
```

## Troubleshooting

### `Building numpy==x.x.x` hangs on Pi Zero 1

The Pi Zero 1 (ARMv6) has no pre-built numpy wheel on PyPI, so pip will
attempt to compile it from source — which takes over an hour and often runs
out of memory.

Fix: install numpy via apt *before* running `uv pip install`, then create the
venv with `--system-site-packages` so uv can see it:

```bash
sudo apt install python3-numpy python3-pil python3-rpi.gpio python3-spidev
uv venv --system-site-packages
uv pip install -e ".[inky]"
```

piwheels (pre-built ARM wheels) is already configured in `pyproject.toml` for
any packages that do have ARMv6 builds there.

### `"some pins we need are in use!"` on startup

Newer versions of the inky library use `gpiodevice` (libgpiod) to manage GPIO
pins, which can conflict with the kernel SPI driver. This project pins
`inky==2.3.0` which avoids this issue. If you see it anyway, add the following
to `/boot/config.txt` and reboot:

```
dtoverlay=spi0-0cs
```

### Autostart

To make it run on startup you can choose from 2 options:

1. Using the rc.local file
    1. `sudo nano /etc/rc.local`
    2. Add one the following before `exit 0`
   ```
   su - pi -c "cd /home/pi/zero-btc-screen && /home/pi/.local/bin/uv run python main.py" &
   ```
2. Using the system's services daemon (recommended)
    1. Create a new service configuration file
       ```
        sudo nano /etc/systemd/system/btc-screen.service
        ```
    2. Copy and paste the following into the service configuration file and change any settings to match your
       environment
       ```
        [Unit]
        Description=zero-btc-screen
        After=network.target
 
        [Service]
        ExecStart=/home/pi/.local/bin/uv run python main.py
        WorkingDirectory=/home/pi/zero-btc-screen
        StandardOutput=inherit
        StandardError=inherit
        Restart=always
        User=pi
 
        [Install]
        WantedBy=multi-user.target
        ```
    3. Enable the service so that it starts whenever the RPi is rebooted
       ```
        sudo systemctl enable btc-screen.service
       ```
    4. Start the service and enjoy!
       ```
        sudo systemctl start btc-screen.service
       ```

       If you need to troubleshoot you can use the logging configurations of this program (mentioned below).
       Alternatively, you can check to see if there is any output in the system service logging.
       ```
        sudo journalctl -f -u btc-screen.service
       ```

### Support the project
If you would like to support this project and and keep me caffeinated, you can do it here:

[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://www.buymeacoffee.com/drmod)
