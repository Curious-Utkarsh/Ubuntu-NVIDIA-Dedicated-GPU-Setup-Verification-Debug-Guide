# Complete NVIDIA GPU Setup Guide for Robotics on Ubuntu 22.04
## The Ultimate Beginner's Guide

**Target Audience:** Brand new laptop owners with NVIDIA GPU wanting to set up for robotics (ROS 2, Gazebo, Isaac Sim, RViz)

**Goal:** Configure your system to ALWAYS use the dedicated NVIDIA GPU for maximum performance

---

## Table of Contents

1. [Initial System Setup](#1-initial-system-setup)
2. [Check Hardware Detection](#2-check-hardware-detection)
3. [Install NVIDIA Drivers](#3-install-nvidia-drivers)
4. [Configure PRIME for Dedicated GPU](#4-configure-prime-for-dedicated-gpu)
5. [Switch to Xorg Display Server](#5-switch-to-xorg-display-server)
6. [Verify OpenGL Configuration](#6-verify-opengl-configuration)
7. [Configure Vulkan for NVIDIA](#7-configure-vulkan-for-nvidia)
8. [Optimize NVIDIA Settings](#8-optimize-nvidia-settings)
9. [Install Robotics Tools](#9-install-robotics-tools)
10. [Verify Everything Works](#10-verify-everything-works)
11. [Docker GPU Setup](#11-docker-gpu-setup)
12. [Troubleshooting Guide](#12-troubleshooting-guide)
13. [Maintenance & Updates](#13-maintenance--updates)

---

## 1. Initial System Setup

### Step 1.1: Update System

First, ensure your Ubuntu 22.04 is fully updated:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt dist-upgrade -y
reboot
```

**What this does:** Gets the latest packages and security updates

**After reboot, continue:**

---

## 2. Check Hardware Detection

### Step 2.1: Check if Ubuntu Sees Your GPU

```bash
lspci | grep -i nvidia
```

**Expected output (example):**
```
01:00.0 VGA compatible controller: NVIDIA Corporation GA107M [GeForce RTX 3050 Mobile] (rev a1)
```

**✅ If you see this:** Your GPU is detected by the system. Continue to driver installation.

**❌ If you see nothing:**

**Problem:** Hardware not detected

**Solutions:**

1. **Check BIOS settings:**
   - Reboot and enter BIOS (usually F2, F10, or Del key during boot)
   - Look for "Discrete Graphics" or "Switchable Graphics"
   - Enable the dedicated GPU
   - Save and exit

2. **Check secure boot:**
   ```bash
   mokutil --sb-state
   ```
   If enabled, you may need to disable it for NVIDIA drivers:
   - Reboot → BIOS → Disable Secure Boot

3. **Hardware issue:** If still not detected, contact laptop manufacturer

---

## 3. Install NVIDIA Drivers

### Step 3.1: Remove Any Existing Drivers (Clean Slate)

```bash
sudo apt remove --purge '^nvidia-.*' -y
sudo apt autoremove -y
sudo apt autoclean
```

### Step 3.2: Check Recommended Driver Version

```bash
ubuntu-drivers devices
```

**Expected output:**
```
== /sys/devices/pci0000:00/0000:00:01.0/0000:01:00.0 ==
modalias : pci:v000010DEd00002520sv00001028sd00000A99bc03sc00i00
vendor   : NVIDIA Corporation
model    : GA107M [GeForce RTX 3050 Mobile]
driver   : nvidia-driver-535 - distro non-free recommended
driver   : nvidia-driver-525 - distro non-free
driver   : nvidia-driver-515 - distro non-free
```

**Look for the line with "recommended"**

### Step 3.3: Install the Recommended Driver

**Method 1: Automatic (Recommended for beginners)**

```bash
sudo ubuntu-drivers autoinstall
```

**Method 2: Manual (if you want specific version)**

Replace `535` with your recommended version:

```bash
sudo apt install nvidia-driver-535 -y
```

**What this does:** Downloads and installs the NVIDIA proprietary driver

**Installation time:** 5-10 minutes depending on internet speed

### Step 3.4: Reboot

```bash
reboot
```

**IMPORTANT:** You MUST reboot for drivers to load

### Step 3.5: Verify Driver Installation

After reboot:

```bash
nvidia-smi
```

**✅ Expected output:**
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.154.05   Driver Version: 535.154.05   CUDA Version: 12.2   |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA GeForce ...  Off  | 00000000:01:00.0 Off |                  N/A |
| N/A   45C    P8     3W /  N/A |      5MiB /  4096MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
```

**❌ If you get "command not found" or "Failed to initialize NVML":**

### Troubleshooting Driver Installation

**Problem 1: nvidia-smi not found**

```bash
# Check if driver packages are installed
dpkg -l | grep nvidia-driver

# If empty, reinstall
sudo ubuntu-drivers autoinstall
reboot
```

**Problem 2: NVML initialization failed**

```bash
# Check kernel modules
lsmod | grep nvidia
```

If empty, modules aren't loading:

```bash
# Try loading manually
sudo modprobe nvidia
sudo modprobe nvidia_drm
sudo modprobe nvidia_modeset

# If this fails, check dmesg for errors
dmesg | grep -i nvidia
```

**Problem 3: Secure Boot blocking driver**

```bash
mokutil --sb-state
```

If enabled:
- Reboot → Enter BIOS → Disable Secure Boot → Save and Exit

Then reinstall driver:
```bash
sudo ubuntu-drivers autoinstall
reboot
```

**Problem 4: Conflicting nouveau driver**

```bash
# Check if nouveau is loaded
lsmod | grep nouveau
```

If yes, blacklist it:

```bash
sudo bash -c "echo blacklist nouveau > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
sudo bash -c "echo options nouveau modeset=0 >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
sudo update-initramfs -u
reboot
```

---

## 4. Configure PRIME for Dedicated GPU

### Step 4.1: Check Current Mode

```bash
prime-select query
```

**Possible outputs:**

- `intel` → Using Intel integrated GPU only (BAD for robotics)
- `on-demand` → Hybrid mode, apps choose GPU (INCONSISTENT)
- `nvidia` → Always use NVIDIA (WHAT WE WANT)

### Step 4.2: Switch to NVIDIA Mode

```bash
sudo prime-select nvidia
```

**Expected output:**
```
Info: selecting the nvidia profile
```

**If command not found:**

```bash
sudo apt install nvidia-prime -y
sudo prime-select nvidia
```

### Step 4.3: Reboot (CRITICAL)

```bash
reboot
```

### Step 4.4: Verify PRIME Setting

```bash
prime-select query
```

**Expected output:**
```
nvidia
```

**✅ If you see "nvidia":** Success! Continue.

**❌ If still shows "intel" or "on-demand":**

```bash
# Force it again
sudo prime-select nvidia
sudo systemctl restart display-manager
# Or just reboot again
reboot
```

---

## 5. Switch to Xorg Display Server

**Why:** Wayland (default in Ubuntu 22.04) has poor NVIDIA support and causes issues with Gazebo, RViz, and OpenGL apps.

### Step 5.1: Check Current Display Server

```bash
echo $XDG_SESSION_TYPE
```

**Output:**
- `wayland` → Need to switch (most common on fresh install)
- `x11` → Already on Xorg, skip to next section

### Step 5.2: Switch to Xorg

**Method 1: GUI (Easiest)**

1. **Logout** (don't reboot)
2. On login screen, click your username
3. **BEFORE entering password**, click the ⚙️ (gear icon) in bottom right
4. Select **"Ubuntu on Xorg"**
5. Enter password and login

**Method 2: Make Xorg Default**

```bash
sudo nano /etc/gdm3/custom.conf
```

Find this line:
```
#WaylandEnable=false
```

Uncomment it (remove the #):
```
WaylandEnable=false
```

Save (Ctrl+O, Enter) and exit (Ctrl+X)

```bash
reboot
```

### Step 5.3: Verify Xorg is Active

```bash
echo $XDG_SESSION_TYPE
```

**Expected output:**
```
x11
```

**✅ If "x11":** Perfect, continue!

---

## 6. Verify OpenGL Configuration

### Step 6.1: Install OpenGL Utilities

```bash
sudo apt install mesa-utils -y
```

### Step 6.2: Check OpenGL Renderer

```bash
glxinfo | grep "OpenGL renderer"
```

**✅ Expected (GOOD):**
```
OpenGL renderer string: NVIDIA GeForce RTX 3050 Laptop GPU/PCIe/SSE2
```

**❌ BAD outputs and fixes:**

**Problem 1: Shows "Intel" or "UHD Graphics"**
```
OpenGL renderer string: Mesa Intel(R) UHD Graphics
```

**Fix:**
```bash
# Verify PRIME is set to nvidia
prime-select query

# If not nvidia:
sudo prime-select nvidia
reboot

# Verify again
glxinfo | grep "OpenGL renderer"
```

**Problem 2: Shows "llvmpipe" (software renderer)**
```
OpenGL renderer string: llvmpipe (LLVM 12.0.0, 256 bits)
```

**This means NO GPU is being used!**

**Fix:**
```bash
# Check if nvidia modules are loaded
lsmod | grep nvidia

# If empty, driver issue:
sudo ubuntu-drivers autoinstall
reboot
```

**Problem 3: "glxinfo: command not found"**
```bash
sudo apt install mesa-utils -y
```

### Step 6.3: Full OpenGL Details

```bash
glxinfo -B
```

**Expected output (key parts):**
```
name of display: :0
display: :0  screen: 0
direct rendering: Yes
Extended renderer info (GLX_MESA_query_renderer):
    Vendor: NVIDIA Corporation (0x10de)
    Device: NVIDIA GeForce RTX 3050 Laptop GPU (0x2520)
    Version: 535.154.5
OpenGL vendor string: NVIDIA Corporation
OpenGL renderer string: NVIDIA GeForce RTX 3050 Laptop GPU/PCIe/SSE2
```

**Key checks:**
- ✅ Direct rendering: Yes
- ✅ Vendor: NVIDIA Corporation
- ✅ Renderer: Your NVIDIA GPU name

---

## 7. Configure Vulkan for NVIDIA

**Why:** Gazebo Fortress, Isaac Sim, and modern simulators use Vulkan. By default, Ubuntu may try to use Intel or CPU software renderer.

### Step 7.1: Install Vulkan Tools

```bash
sudo apt install vulkan-tools -y
```

### Step 7.2: Check Available Vulkan Devices

```bash
vulkaninfo | grep deviceName
```

**Typical output (PROBLEM):**
```
        deviceName     = Intel(R) UHD Graphics
        deviceName     = llvmpipe (LLVM 12.0.0, 256 bits)
        deviceName     = NVIDIA GeForce RTX 3050 Laptop GPU
```

**Issue:** Multiple devices listed. Apps may choose the wrong one!

### Step 7.3: List Vulkan ICD Files

```bash
ls /usr/share/vulkan/icd.d/
```

**Output:**
```
intel_icd.x86_64.json  
lvp_icd.x86_64.json  
nvidia_icd.json
```

**What these are:**
- `intel_icd*.json` → Intel GPU
- `lvp_icd*.json` → CPU software renderer (llvmpipe)
- `nvidia_icd.json` → NVIDIA GPU (WHAT WE WANT)

### Step 7.4: Force System to Use NVIDIA Vulkan Only

**Edit system environment file:**

```bash
sudo nano /etc/environment
```

**You'll see something like:**
```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
```

**Add this line below PATH:**
```
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
```

**Final file should look like:**
```
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
```

**Save and exit:**
- Press `Ctrl+O` (write out)
- Press `Enter` (confirm)
- Press `Ctrl+X` (exit)

### Step 7.5: Reboot to Apply

```bash
reboot
```

### Step 7.6: Verify Vulkan is NVIDIA-Only

```bash
vulkaninfo | grep deviceName
```

**✅ Expected (GOOD):**
```
        deviceName     = NVIDIA GeForce RTX 3050 Laptop GPU
```

**Only ONE device, and it's NVIDIA!**

**❌ If still showing Intel/llvmpipe:**

```bash
# Check if environment variable is set
echo $VK_ICD_FILENAMES
```

Should output:
```
/usr/share/vulkan/icd.d/nvidia_icd.json
```

If empty:
```bash
# Edit file again
sudo nano /etc/environment

# Make sure line is exactly:
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json

# No quotes, no spaces, no typos
```

Reboot again.

---

## 8. Optimize NVIDIA Settings

### Step 8.1: Open NVIDIA Settings

```bash
nvidia-settings
```

A GUI window opens.

### Step 8.2: Recommended Settings

**Navigate to each section and configure:**

#### **X Server Display Configuration**
- Select your display
- Resolution: Native (highest available)
- Refresh Rate: Highest available

#### **OpenGL Settings**
- **Image Settings:** High Performance
- **Sync to VBlank:** OFF (reduces input lag)
- **Allow Flipping:** ON
- **Allow G-SYNC:** ON (if your display supports it)

#### **PowerMizer**
- **Preferred Mode:** Prefer Maximum Performance
- (This keeps GPU at full speed during robotics work)

**Click "Save to X Configuration File"** if prompted

### Step 8.3: Enable Persistence Mode

This keeps the NVIDIA driver loaded even when no apps are using GPU:

```bash
sudo nvidia-smi -pm 1
```

**Expected output:**
```
Enabled persistence mode for GPU 00000000:01:00.0.
All done.
```

**Make it permanent:**

```bash
sudo systemctl enable nvidia-persistenced
sudo systemctl start nvidia-persistenced
```

### Step 8.4: Disable Auto-Suspend (Laptops)

Prevents GPU from sleeping during work:

```bash
sudo systemctl mask nvidia-suspend.service
sudo systemctl mask nvidia-hibernate.service
sudo systemctl mask nvidia-resume.service
```

---

## 9. Install Robotics Tools

### Step 9.1: Install ROS 2 Humble

```bash
# Add ROS 2 repository
sudo apt install software-properties-common -y
sudo add-apt-repository universe -y
sudo apt update

sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt upgrade -y

# Install ROS 2 Humble Desktop
sudo apt install ros-humble-desktop -y

# Install development tools
sudo apt install ros-dev-tools -y
```

**Installation time:** 10-15 minutes

### Step 9.2: Source ROS 2

Add to your bash profile:

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### Step 9.3: Install Gazebo

**For Gazebo Classic (11):**
```bash
sudo apt install gazebo -y
```

**For Gazebo Fortress (newer):**
```bash
sudo apt install gazebo-fortress -y
```

### Step 9.4: Install RViz2

Already included in `ros-humble-desktop`, verify:

```bash
which rviz2
```

Should output:
```
/opt/ros/humble/bin/rviz2
```

---

## 10. Verify Everything Works

### Step 10.1: Test OpenGL with glxgears

```bash
glxgears
```

**Expected:** Smooth rotating gears window

**Check GPU usage:**

Open new terminal:
```bash
watch -n 1 nvidia-smi
```

You should see `glxgears` in the **Processes** section.

### Step 10.2: Test RViz2

```bash
rviz2
```

**Expected:** RViz window opens smoothly

**Check GPU in other terminal:**
```bash
nvidia-smi
```

Look for `rviz2` in processes.

**If RViz is slow or glitchy:**
- Check OpenGL renderer: `glxinfo | grep "OpenGL renderer"`
- Should show NVIDIA, not Intel

### Step 10.3: Test Gazebo

**Gazebo Classic:**
```bash
gazebo
```

**Gazebo Fortress:**
```bash
gz sim
```

**Expected:** World loads, you can orbit camera smoothly

**Check GPU usage:**
```bash
nvidia-smi
```

Should show `gzserver` or `gz` process.

**Common Gazebo issues:**

**Problem: Black screen or crash**

**Fix 1: Check Vulkan**
```bash
vulkaninfo | grep deviceName
```
Must show NVIDIA only.

**Fix 2: Disable VSYNC**
```bash
export GAZEBO_DISABLE_VSYNC=1
gazebo
```

**Fix 3: Force Vulkan device (add to ~/.bashrc)**
```bash
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
```

### Step 10.4: Monitor GPU Usage Live

**Best practice: Keep this running in a terminal while working**

```bash
watch -n 1 nvidia-smi
```

**What to look for:**
- GPU Utilization: Should spike when running sims
- Memory Usage: Should increase with complex scenes
- Processes: Should list your apps

**Example during work:**
```
|  0  NVIDIA GeForce RTX...  Off  |   30%   2000MiB / 4096MiB |      45%      |
...
Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                          Usage      |
|    0      2341      G   /usr/bin/gnome-shell                   200MiB    |
|    0      3456      G   rviz2                                  800MiB    |
|    0      4567      G   gzserver                              1000MiB    |
```

---

## 11. Docker GPU Setup

**Why:** Many robotics tools (Isaac Sim, custom simulators) run in Docker.

### Step 11.1: Install Docker

```bash
# Remove old versions
sudo apt-get remove docker docker-engine docker.io containerd runc

# Install dependencies
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release -y

# Add Docker GPG key
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

**Logout and login again** for group changes to take effect.

### Step 11.2: Install NVIDIA Container Toolkit

```bash
# Add NVIDIA Docker repository
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install toolkit
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Configure Docker
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

### Step 11.3: Test Docker GPU Access

```bash
docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
```

**Expected:** Same `nvidia-smi` output as your host system.

### Step 11.4: Run Robotics Containers with GPU

**Template:**
```bash
docker run --rm -it \
  --gpus all \
  -e VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  --network host \
  <your-robotics-image>
```

**Example: ROS 2 with GPU**
```bash
# Allow X server access
xhost +local:docker

docker run --rm -it \
  --gpus all \
  -e VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  --network host \
  osrf/ros:humble-desktop
```

Inside container:
```bash
rviz2
```

Should use GPU!

---

## 12. Troubleshooting Guide

### Issue 1: "nvidia-smi" shows no processes but app is running

**Cause:** App is not GPU accelerated

**Fix:**
```bash
# Check OpenGL renderer
glxinfo | grep "OpenGL renderer"

# Must show NVIDIA
# If shows Intel:
sudo prime-select nvidia
reboot
```

### Issue 2: RViz/Gazebo very slow/laggy

**Diagnosis:**
```bash
nvidia-smi
```

If app NOT in process list → not using GPU.

**Fix:**
```bash
# Check display server
echo $XDG_SESSION_TYPE

# Must be x11, not wayland
# If wayland, logout and select "Ubuntu on Xorg"
```

### Issue 3: Gazebo black screen or crash on startup

**Fix 1: Vulkan device**
```bash
vulkaninfo | grep deviceName
# Must show NVIDIA only

# If shows Intel/llvmpipe:
sudo nano /etc/environment
# Add: VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
reboot
```

**Fix 2: Force environment variable**
```bash
echo "export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json" >> ~/.bashrc
source ~/.bashrc
```

### Issue 4: Driver updates break system

**Recovery:**

Boot into recovery mode:
1. Reboot
2. Hold Shift during boot
3. Select "Advanced options for Ubuntu"
4. Select "Recovery mode"
5. Select "root - Drop to root shell prompt"

```bash
# Mount filesystem as read-write
mount -o remount,rw /

# Remove broken drivers
apt remove --purge '^nvidia-.*'
apt autoremove

# Reinstall
ubuntu-drivers autoinstall
reboot
```

### Issue 5: Screen tearing

**Fix:**
```bash
nvidia-settings
# OpenGL Settings → Sync to VBlank → ON
# Or Force Full Composition Pipeline in X Server settings
```

### Issue 6: High GPU temperature/fan noise

**Check temperature:**
```bash
nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader
```

**Fix:**
```bash
nvidia-settings
# PowerMizer → Adaptive (instead of Max Performance)
```

### Issue 7: External display not working

```bash
# Force detect
sudo nvidia-settings

# X Server Display Configuration → Detect Displays
```

---

## 13. Maintenance & Updates

### Update NVIDIA Drivers

**Check for updates:**
```bash
ubuntu-drivers devices
```

**Install newer driver if available:**
```bash
sudo ubuntu-drivers autoinstall
reboot
```

**Or specific version:**
```bash
sudo apt install nvidia-driver-545
reboot
```

### Monthly Maintenance Routine

```bash
# Update system
sudo apt update
sudo apt upgrade -y

# Clean old kernels
sudo apt autoremove -y

# Verify GPU still works
nvidia-smi
glxinfo | grep "OpenGL renderer"
vulkaninfo | grep deviceName
```

### Backup Critical Configs

```bash
# Backup environment file
sudo cp /etc/environment /etc/environment.backup

# Backup NVIDIA settings
nvidia-settings --save-config

# Backup modprobe configs
sudo cp /etc/modprobe.d/blacklist-nvidia-nouveau.conf ~/backups/
```

---

## Final Checklist

After completing this guide, verify:

- [ ] `nvidia-smi` shows your GPU
- [ ] `prime-select query` returns "nvidia"
- [ ] `echo $XDG_SESSION_TYPE` returns "x11"
- [ ] `glxinfo | grep "OpenGL renderer"` shows NVIDIA
- [ ] `vulkaninfo | grep deviceName` shows NVIDIA only
- [ ] `lsmod | grep nvidia` shows kernel modules loaded
- [ ] RViz2 appears in `nvidia-smi` process list
- [ ] Gazebo runs smoothly without black screen
- [ ] Docker containers can access GPU

---

## Quick Reference Commands

```bash
# Check GPU detection
lspci | grep -i nvidia

# Check driver
nvidia-smi

# Check PRIME mode
prime-select query

# Check display server
echo $XDG_SESSION_TYPE

# Check OpenGL
glxinfo | grep "OpenGL renderer"

# Check Vulkan
vulkaninfo | grep deviceName

# Monitor GPU usage
watch -n 1 nvidia-smi

# Switch to NVIDIA mode
sudo prime-select nvidia

# Force Xorg
# Edit /etc/gdm3/custom.conf → WaylandEnable=false
```

---

## Conclusion

Your Ubuntu 22.04 system is now a **dedicated NVIDIA-powered robotics workstation**!

**What you've achieved:**
- ✅ NVIDIA driver installed and verified
- ✅ System always uses dedicated GPU
- ✅ OpenGL and Vulkan properly configured
- ✅ ROS 2, Gazebo, RViz GPU-accelerated
- ✅ Docker containers can access GPU
- ✅ Optimized for robotics performance

**Your system is ready for:**
- ROS 2 development
- Gazebo Classic & Fortress simulations
- RViz 3D visualization
- Isaac Sim
- CUDA development
- Machine learning workloads
- Point cloud processing
- Computer vision

**Keep this guide bookmarked for future troubleshooting!**

---

## Additional Resources

- NVIDIA Driver Documentation: https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/
- ROS 2 Documentation: https://docs.ros.org/en/humble/
- Gazebo Fortress: https://gazebosim.org/docs/fortress
- Ubuntu NVIDIA Setup: https://ubuntu.com/server/docs/nvidia-drivers-installation

---

**Document Version:** 1.0  
**Last Updated:** December 2024  
**Tested On:** Ubuntu 22.04 LTS with NVIDIA RTX 30/40 series laptop GPUs
