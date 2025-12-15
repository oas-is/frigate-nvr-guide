# Frigate NVR - Local AI Surveillance System

> **Take control of your privacy.** Build a completely local, open-source AI surveillance system that never touches the cloud.

[![NetworkChuck YouTube](https://img.shields.io/badge/YouTube-NetworkChuck-red?style=flat&logo=youtube)](https://youtube.com/@NetworkChuck)
[![Frigate](https://img.shields.io/badge/Frigate-NVR-blue)](https://frigate.video/)

## Why Frigate?

Your surveillance cameras might be watching you - and so might someone else. Cloud-connected cameras have their feeds sold on the dark web. With Frigate:

- **100% Local** - Nothing leaves your network
- **AI-Powered** - Object detection, facial recognition, semantic search
- **Open Source** - You own it, you control it
- **Hardware Flexible** - Runs on a Raspberry Pi to a full server

## What You'll Build

| Feature | Description |
|---------|-------------|
| Live Viewing | Real-time camera feeds in your browser |
| AI Detection | Person, car, pet, and object detection |
| Recording | Motion-based or continuous recording |
| Facial Recognition | Know who's at your door |
| Semantic Search | Search your footage with natural language |
| PTZ Control | Pan, tilt, zoom your cameras from the UI |
| Home Assistant | Full integration for automations |

---

## Shopping List

Here's exactly what I used in the video:

| Component | Link | Price |
|-----------|------|-------|
| **Raspberry Pi 5 (8GB)** | [Buy on Amazon](https://geni.us/q7BXLZN) | ~$80 |
| **Hailo-8L AI HAT** | [Buy on Amazon](https://geni.us/9r4Z) | ~$70 |
| **Reolink E1 Pro Camera** | [Buy on Amazon](https://geni.us/d9dS) | ~$56 |
| **Google Coral USB** | [Buy on Amazon](https://geni.us/BjOg72c) | ~$100 |

---

## Table of Contents

1. [Shopping List](#shopping-list)
2. [Hardware Requirements](#hardware-requirements)
3. [Quick Start (Raspberry Pi)](#quick-start-raspberry-pi)
4. [Desktop/Server Setup](#desktopserver-setup)
5. [Camera Setup (Reolink)](#camera-setup-reolink)
6. [Progressive Configuration Guide](#progressive-configuration-guide)
7. [WiFi Troubleshooting](#wifi-troubleshooting-the-hidden-metric)
8. [AI Accelerator Setup](#ai-accelerator-setup)
9. [Home Assistant Integration](#home-assistant-integration)
10. [Troubleshooting](#troubleshooting)

---

## Hardware Requirements

### Option 1: Raspberry Pi Setup (2-4 cameras)

| Component | Recommendation | Notes |
|-----------|---------------|-------|
| **Computer** | [Raspberry Pi 5 (8GB)](https://geni.us/q7BXLZN) | 4GB works but 8GB recommended |
| **Storage** | 128GB+ microSD or NVMe | NVMe via HAT recommended for recording |
| **AI Accelerator** | [Hailo-8L AI HAT](https://geni.us/9r4Z) (~$70) | Optional but highly recommended |
| **Cameras** | [Reolink E1 Pro](https://geni.us/d9dS) (~$56 each) | PTZ, WiFi, RTSP support |

### Option 2: Desktop/Server Setup (5+ cameras)

| Component | Recommendation | Notes |
|-----------|---------------|-------|
| **Computer** | Any x86 Linux machine | Old gaming PC works great |
| **GPU** | NVIDIA GPU (optional) | For hardware video decoding |
| **AI Accelerator** | [Google Coral USB](https://geni.us/BjOg72c) (~$100) | Handles all AI inference |
| **Storage** | SSD + HDD/NAS | SSD for cache, HDD for recordings |

### Cameras

Any RTSP-compatible camera works. Tested/recommended brands:
- **[Reolink E1 Pro](https://geni.us/d9dS)** - Budget-friendly, PTZ, WiFi (what I used)
- **Reolink RLC-series** - Outdoor, PoE options
- **Amcrest** - Good RTSP support
- **Dahua** - Professional grade
- **UniFi Protect** - Works with RTSP enabled

**Key camera features to look for:**
- RTSP support (required)
- Dual stream (main + sub stream)
- H.264 or H.265 encoding
- 5GHz WiFi support (for wireless cameras)

---

## Quick Start (Raspberry Pi)

### Step 1: Install Raspberry Pi OS

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
2. Flash **Raspberry Pi OS (64-bit)** to your SD card
3. Enable SSH in the imager settings
4. Boot your Pi and connect via SSH

```bash
ssh your-username@your-pi-ip
```

### Step 2: Update System & Enable PCIe Gen 3

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Enable PCIe Gen 3 for Hailo (if using AI HAT)
sudo raspi-config nonint do_pcie_speed 3
```

### Step 3: Install Docker

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add your user to docker group
sudo usermod -aG docker $USER

# Log out and back in, then verify
docker --version
```

### Step 4: Create Frigate Directory Structure

```bash
mkdir -p ~/frigate/{config,storage}
cd ~/frigate
```

### Step 5: Create Docker Compose File

```bash
nano docker-compose.yml
```

**Basic Docker Compose (no AI accelerator):**

```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    shm_size: "64mb"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./config:/config
      - ./storage:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"      # Web UI
      - "8554:8554"      # RTSP feeds
      - "8555:8555/tcp"  # WebRTC
      - "8555:8555/udp"  # WebRTC
    environment:
      - FRIGATE_RTSP_PASSWORD=YourSecurePassword
```

### Step 6: Create Basic Frigate Config

```bash
nano config/config.yml
```

```yaml
mqtt:
  enabled: false

cameras:
  # Empty for now - we'll add cameras next
```

### Step 7: Start Frigate

```bash
docker compose up -d

# Check if it's running
docker ps

# View logs
docker logs frigate --tail 50
```

### Step 8: Access the Web UI

Open your browser and go to: `http://your-pi-ip:5000`

You should see the Frigate dashboard (empty, but working).

---

## Camera Setup (Reolink)

### Enable RTSP on Your Camera

1. Open the **Reolink app** on your phone
2. Add your camera and connect it to WiFi
3. Go to **Settings > Network > Advanced**
4. Enable **RTSP** (default port: 554)
5. Enable **ONVIF** if you want PTZ control

### Find Your Camera's RTSP URL

**Reolink cameras use this format:**

```
# Main stream (high quality - for recording)
rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_main

# Sub stream (low quality - for AI detection)
rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub
```

Replace:
- `PASSWORD` with your camera password
- `CAMERA_IP` with your camera's IP address

### Test Your RTSP Stream

```bash
# Install ffmpeg
sudo apt install ffmpeg -y

# Test the stream (records 5 seconds)
ffmpeg -i "rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub" \
  -t 5 -c copy test.mp4

# Check if file was created
ls -la test.mp4
```

If `test.mp4` exists and plays, your RTSP stream is working!

### Add Camera to Frigate

Edit your config:

```bash
nano ~/frigate/config/config.yml
```

```yaml
mqtt:
  enabled: false

ffmpeg:
  input_args: preset-rtsp-generic

cameras:
  front_door:  # Name your camera
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub
          roles:
            - detect
    detect:
      enabled: false  # Start with detection OFF
      width: 896
      height: 512
      fps: 5
```

Restart Frigate:

```bash
cd ~/frigate && docker compose restart
```

Check the web UI - you should see your camera feed!

---

## Progressive Configuration Guide

Build your config step by step. Each phase adds new capabilities.

### Phase 1: Stream Verification

Just get the camera streaming - no AI yet.

```yaml
mqtt:
  enabled: false

ffmpeg:
  input_args: preset-rtsp-generic

cameras:
  front_door:
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub
          roles:
            - detect
    detect:
      enabled: false
      width: 896
      height: 512
      fps: 5
```

**Verify:** Camera feed appears in UI.

### Phase 2: Enable AI Detection

Turn on object detection (uses CPU without accelerator).

```yaml
mqtt:
  enabled: false

ffmpeg:
  input_args: preset-rtsp-generic

objects:
  track:
    - person
    - car
    - dog
    - cat

cameras:
  front_door:
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub
          roles:
            - detect
    detect:
      enabled: true  # NOW enabled
      width: 896
      height: 512
      fps: 5
```

**Verify:** You see detection boxes around people, cars, pets.

### Phase 3: Add Recording (Dual Stream)

Use main stream for recording, sub stream for detection.

```yaml
mqtt:
  enabled: false

ffmpeg:
  input_args: preset-rtsp-generic

objects:
  track:
    - person
    - car
    - dog
    - cat

record:
  enabled: true
  retain:
    days: 7
    mode: motion

cameras:
  front_door:
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_main
          roles:
            - record
        - path: rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub
          roles:
            - detect
    detect:
      enabled: true
      width: 896
      height: 512
      fps: 5
    record:
      enabled: true
      alerts:
        retain:
          days: 14
        pre_capture: 5
        post_capture: 5
```

**Verify:** Recordings appear in the Review tab.

### Phase 4: Add Second Camera

Simply duplicate the camera config with new name and IP.

```yaml
cameras:
  front_door:
    # ... front door config ...

  backyard:
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://admin:PASSWORD@CAMERA_IP_2:554/h265Preview_01_main
          roles:
            - record
        - path: rtsp://admin:PASSWORD@CAMERA_IP_2:554/h265Preview_01_sub
          roles:
            - detect
    detect:
      enabled: true
      width: 896
      height: 512
      fps: 5
    record:
      enabled: true
```

### Phase 5: Add PTZ Control (ONVIF)

For cameras with pan/tilt/zoom:

```yaml
cameras:
  front_door:
    # ... existing config ...
    onvif:
      host: CAMERA_IP
      port: 8000
      user: admin
      password: PASSWORD
```

**Verify:** PTZ controls appear in camera view.

---

## AI Accelerator Setup

### [Hailo-8L AI HAT](https://geni.us/9r4Z) (Raspberry Pi 5)

> **CRITICAL:** Frigate 0.15.x requires Hailo driver version **4.19**. Do NOT use the latest drivers!

#### Install Hailo Drivers

```bash
# 1. Enable PCIe Gen 3
sudo raspi-config nonint do_pcie_speed 3

# 2. Update system
sudo apt update && sudo apt upgrade -y

# 3. Install SPECIFIC versions (4.19.x)
sudo apt install -y \
  hailo-dkms=4.19.0-1 \
  hailort=4.19.0-3 \
  python3-hailort=4.19.0-2 \
  hailo-tappas-core=3.30.0-1

# 4. LOCK versions to prevent auto-update
sudo apt-mark hold hailo-dkms hailort python3-hailort hailo-tappas-core

# 5. Reboot
sudo reboot
```

#### Verify Installation

```bash
hailortcli fw-control identify
```

You should see:
```
Driver version: 4.19.0
Firmware version: 4.19.0
```

#### Update Docker Compose for Hailo

```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable-h8l  # Note: h8l image!
    shm_size: "128mb"
    devices:
      - /dev/hailo0:/dev/hailo0       # Hailo AI chip
      - /dev/video11:/dev/video11     # RPi hardware decoder
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./config:/config
      - ./storage:/media/frigate
      - type: tmpfs
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "5000:5000"
      - "8554:8554"
      - "8555:8555/tcp"
      - "8555:8555/udp"
    environment:
      - FRIGATE_RTSP_PASSWORD=YourSecurePassword
```

#### Update Frigate Config for Hailo

```yaml
mqtt:
  enabled: false

# Hailo detector configuration
detectors:
  hailo:
    type: hailo8l
    device: PCIe

model:
  width: 300
  height: 300
  input_tensor: nhwc
  input_pixel_format: bgr
  model_type: ssd
  path: /config/model_cache/h8l_cache/ssd_mobilenet_v1.hef

# Use RPi hardware acceleration
ffmpeg:
  hwaccel_args: preset-rpi-64-h264
  input_args: preset-rtsp-generic

objects:
  track:
    - person
    - car
    - dog
    - cat

cameras:
  # ... your cameras ...
```

#### Performance Comparison

| Metric | CPU Only | With Hailo-8L |
|--------|----------|---------------|
| Inference Speed | 60-80ms | 8-10ms |
| CPU Usage | 200%+ | 15-20% |
| Max Cameras | 1-2 | 4-6 |

### [Google Coral USB](https://geni.us/BjOg72c) (Desktop)

#### Install Coral Drivers

```bash
# Add Coral repository
echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | \
  sudo tee /etc/apt/sources.list.d/coral-edgetpu.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | \
  sudo apt-key add -

sudo apt update

# Install runtime (standard speed)
sudo apt install libedgetpu1-std

# OR install max speed (runs hotter)
# sudo apt install libedgetpu1-max
```

#### Update Docker Compose for Coral

```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    privileged: true
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    shm_size: "256mb"
    devices:
      - /dev/bus/usb:/dev/bus/usb  # Coral USB passthrough
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./config:/config
      - ./storage:/media/frigate
    ports:
      - "5000:5000"
      - "8554:8554"
      - "8555:8555/tcp"
      - "8555:8555/udp"
```

#### Update Frigate Config for Coral

```yaml
detectors:
  coral:
    type: edgetpu
    device: usb

# ... rest of config ...
```

---

## WiFi Troubleshooting (The Hidden Metric)

> **This section is gold.** I ran 10 WiFi cameras and my network died after 12 hours. The problem wasn't bandwidth - it was something else entirely.

### The Symptoms

- Network works great for 12-20 hours
- Then everything dies - cameras freeze, internet stops
- Rebooting Frigate fixes it temporarily
- Repeat cycle

### The Diagnosis

**I thought it was bandwidth.** 10 cameras streaming = lots of data, right?

**Wrong.** My bandwidth usage was barely touched. The real culprit was **airtime saturation**.

### The Hidden Metric: TX Retry Rate

In my UniFi controller, I found this metric:
- **Dumbledore (upstairs AP):** 29.1% TX retry rate
- **Hagrid (downstairs AP):** 7.1% TX retry rate

**What does this mean?**

TX retry rate = percentage of packets that failed on first transmit and had to be resent.

At 29.1%, nearly 1 in 3 packets was failing!

### The Root Cause: Airtime Saturation

Think of it like this:

> Imagine 15 people trying to have conversations in a room that only fits 5. Everyone's talking over each other, constantly repeating "What did you say?"

**The math:**
- 10 cameras × 2 streams each = 20 video streams
- Each stream = ~200 packets/second
- Total: ~3,000+ packets/second fighting for airtime
- WiFi is **half-duplex** - only one device can transmit at a time
- Result: Constant collisions, retries, degradation

### The 20-Hour Problem

Why did it take 12-20 hours to fail?

1. Reolink cameras have a known RTSP degradation issue after ~20 hours of continuous streaming
2. As streams degrade, more retries happen
3. More retries = more airtime consumption
4. Eventually hits critical mass and everything collapses

### The Solutions

#### Solution 1: Optimize Camera Settings ($0)

**Enable Constant Bit Rate (CBR):**
- Reolink app > Settings > Display > Encoding
- Set "Fluency First" to **ON**
- Reduces bitrate spikes that cause congestion

**Reduce main stream bitrate:**
- Main stream: 2048 kbps (down from 3072+)
- Sub stream: 512 kbps

**Use TCP instead of UDP:**
- More reliable on congested WiFi
- Frigate handles this automatically with `preset-rtsp-generic`

#### Solution 2: Scheduled Camera Reboots ($0)

Since cameras degrade after ~20 hours, schedule daily reboots:

1. Reolink app > Settings > System > Auto Reboot
2. Enable daily reboot
3. **Stagger times** across cameras:
   - Camera 1: 3:00 AM
   - Camera 2: 3:10 AM
   - Camera 3: 3:20 AM
   - etc.

This prevents all cameras from rebooting simultaneously and resets the RTSP connection.

#### Solution 3: Use go2rtc (Reduces Streams)

Without go2rtc: Each device (Frigate, Home Assistant, your phone) opens its own connection to the camera.

With go2rtc: One connection to the camera, restreamed to all devices.

**Add go2rtc to your config:**

```yaml
go2rtc:
  streams:
    front_door:
      - ffmpeg:rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_main#video=copy#audio=copy
    front_door_sub:
      - ffmpeg:rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub

cameras:
  front_door:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/front_door
          input_args: preset-rtsp-restream
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/front_door_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
```

#### Solution 4: Add More Access Points ($180+)

**The rule: 3-4 cameras per access point maximum.**

I had 5 cameras on each of my 2 APs. After adding a 3rd AP:
- TX retry rate dropped from 29% to 6%
- Network stability restored
- 3-4 cameras per AP = sustainable

**Tips for multiple APs:**
- Use **non-overlapping 5GHz channels** (36, 52, 100+)
- Disable 2.4GHz for camera SSID (5GHz only)
- Lock cameras to specific APs if possible

#### Solution 5: Run Ethernet (Best Solution)

If you can run cable:
- **Eliminates all WiFi issues**
- Zero airtime consumption
- Most reliable long-term solution

Priority order for running cable:
1. Outdoor cameras (weather + distance issues)
2. Cameras with weak signal (<-70 dBm)
3. High-traffic cameras

### Monitoring Your Network

**UniFi Controller metrics to watch:**
- TX Retry Rate: Should be <10%
- Channel Utilization: Should be <50% per AP
- WiFi Experience: Should be >85%

**Frigate metrics:**
- Settings > System Metrics
- Watch inference speed and detector usage

---

## Home Assistant Integration

Frigate integrates beautifully with Home Assistant.

### Install the Integration

1. HACS > Integrations > Search "Frigate"
2. Install and restart Home Assistant
3. Add integration: Settings > Integrations > Add > Frigate
4. Enter your Frigate URL: `http://frigate-ip:5000`

### What You Get

- All cameras as camera entities
- Motion/person/car sensors
- Event snapshots and clips
- Automation triggers

### Example Automation

```yaml
automation:
  - alias: "Notify on Person Detection"
    trigger:
      - platform: state
        entity_id: binary_sensor.front_door_person_occupancy
        to: "on"
    action:
      - service: notify.mobile_app
        data:
          message: "Person detected at front door!"
          data:
            image: "{{ state_attr('camera.front_door', 'entity_picture') }}"
```

---

## Troubleshooting

### Camera Won't Connect

```bash
# Test RTSP stream
ffmpeg -i "rtsp://admin:PASSWORD@CAMERA_IP:554/h265Preview_01_sub" -t 5 test.mp4

# Check Frigate logs
docker logs frigate | grep -i error
```

### High CPU Usage

- Enable AI accelerator (Hailo or Coral)
- Reduce detection FPS (5 → 3)
- Add motion masks to ignore busy areas
- Reduce number of tracked objects

### Cameras Dropping

- Check WiFi signal strength (aim for >-70 dBm)
- Enable scheduled camera reboots
- Use go2rtc to reduce connections
- Consider adding access points

### Hailo Not Detected

```bash
# Check if Hailo is recognized
hailortcli fw-control identify

# Should show driver version 4.19.0
# If not, reinstall drivers with specific versions
```

### Container Won't Start

```bash
# Check logs
docker logs frigate

# Common issues:
# - Config syntax error (validate YAML)
# - Device not passed through
# - shm_size too small (increase it)
```

---

## Quick Reference

### Essential Commands

```bash
# Start Frigate
cd ~/frigate && docker compose up -d

# Stop Frigate
cd ~/frigate && docker compose down

# Restart Frigate
cd ~/frigate && docker compose restart

# View logs
docker logs frigate --tail 100

# Check status
docker ps | grep frigate

# Edit config
nano ~/frigate/config/config.yml
```

### Reolink RTSP URLs

```
Main stream: rtsp://admin:PASSWORD@IP:554/h265Preview_01_main
Sub stream:  rtsp://admin:PASSWORD@IP:554/h265Preview_01_sub
```

### Ports

| Port | Service |
|------|---------|
| 5000 | Web UI |
| 8554 | RTSP restream |
| 8555 | WebRTC |

---

## Resources

- [Frigate Documentation](https://docs.frigate.video/)
- [Frigate GitHub](https://github.com/blakeblackshear/frigate)
- [Frigate Discord](https://discord.gg/frigate-nvr)
- [NetworkChuck YouTube](https://youtube.com/@NetworkChuck)

---

## Support

If this guide helped you, consider:
- Subscribing to [NetworkChuck on YouTube](https://youtube.com/@NetworkChuck)
- Starring this repo
- Sharing with others who want local surveillance

---

**Made with coffee by NetworkChuck**
