# Ubuntu 22.04 – NVIDIA Dedicated GPU Setup, Verification & Debug Guide

This README documents **every step, check, command, and fix** required to:

* Verify whether Ubuntu apps (Gazebo, RViz, Isaac Sim, OpenGL, Vulkan) are using the **dedicated NVIDIA GPU**
* Force **system-wide NVIDIA usage** (no Intel / CPU fallback)
* Fix OpenGL and Vulkan device selection issues
* Debug common failures on Ubuntu 22.04

This guide is written for **robotics, simulation, ROS 2, Gazebo, RViz, Isaac Sim users**.

---

## System Context (Target Setup)

* OS: Ubuntu 22.04 LTS
* GPU: NVIDIA RTX Laptop GPU (example: RTX 3050)
* Use case: ROS 2, Gazebo, RViz, Isaac Sim, Docker
* Goal: **Always use NVIDIA GPU** for everything

---

## 1. Check if NVIDIA GPU is detected

```bash
nvidia-smi
```

### Expected:

* Shows **Driver Version**
* Shows **GPU name**
* Does NOT error

❌ If this fails → NVIDIA driver is not installed correctly.

---

## 2. Check PRIME GPU mode

```bash
prime-select query
```

### Possible outputs:

* `intel` → Using Intel iGPU only
* `on-demand` → Hybrid mode (default on laptops)
* `nvidia` → **Dedicated NVIDIA only (what we want)**

---

## 3. Force system-wide NVIDIA GPU (IMPORTANT)

```bash
sudo prime-select nvidia
reboot
```

After reboot:

```bash
prime-select query
```

Expected:

```
nvidia
```

---

## 4. IMPORTANT: Use Xorg (NOT Wayland)

Wayland causes PRIME, OpenGL and Vulkan issues with NVIDIA.

### Check current session:

```bash
echo $XDG_SESSION_TYPE
```

If output is:

```
wayland
```

### Switch to Xorg:

1. Logout
2. Login screen → ⚙️ icon
3. Select **Ubuntu on Xorg**
4. Login

---

## 5. Verify OpenGL Renderer (CRITICAL)

Install tool if missing:

```bash
sudo apt install mesa-utils
```

Check renderer:

```bash
glxinfo | grep "OpenGL renderer"
```

### Expected (GOOD):

```
OpenGL renderer string: NVIDIA GeForce ...
```

### BAD outputs:

* `Mesa Intel UHD Graphics`
* `llvmpipe`

These mean GPU is NOT being used.

---

## 6. Live GPU usage check

```bash
watch -n 1 nvidia-smi
```

Launch:

* `rviz2`
* `gazebo` / `gzclient`

You should see them listed under **Processes**.

---

## 7. Verify kernel NVIDIA modules

```bash
lsmod | grep nvidia
```

Expected modules:

* `nvidia`
* `nvidia_drm`
* `nvidia_modeset`
* `nvidia_uvm`

❌ If missing → driver install issue.

---

## 8. Check OpenGL details (full)

```bash
glxinfo -B
```

Verify:

* Vendor: NVIDIA Corporation
* Renderer: NVIDIA GPU
* Direct Rendering: Yes

---

## 9. Vulkan setup & verification

### Install Vulkan tools

```bash
sudo apt install vulkan-tools
```

### Check Vulkan devices

```bash
vulkaninfo | grep deviceName
```

### Typical BAD output (multiple devices):

* Intel UHD Graphics
* llvmpipe
* NVIDIA GPU

This causes:

* Gazebo Fortress issues
* Isaac Sim crashes / black screen

---

## 10. List Vulkan ICD files

```bash
ls /usr/share/vulkan/icd.d/
```

Typical files:

* `intel_icd*.json`
* `lvp_icd.json` (CPU)
* `nvidia_icd.json` (WHAT WE WANT)

⚠️ **DO NOT DELETE ANY FILES**

---

## 11. Force NVIDIA Vulkan (CORRECT FIX)

### Permanent system-wide fix

Edit environment file:

```bash
sudo nano /etc/environment
```

Add this line:

```text
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
```

Ensure file looks like:

```text
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json
```

Reboot:

```bash
reboot
```

---

## 12. Verify Vulkan is NVIDIA-only

```bash
vulkaninfo | grep deviceName
```

Expected:

```
deviceName = NVIDIA GeForce RTX ...
```

No Intel. No llvmpipe.

---

## 13. NVIDIA Settings (Performance tuning)

```bash
nvidia-settings
```

### Recommended:

* **OpenGL Image Settings** → High Performance
* Sync to VBlank → OFF
* Allow Flipping → ON
* PowerMizer → Prefer Maximum Performance

---

## 14. Enable persistence mode

```bash
sudo nvidia-smi -pm 1
```

---

## 15. Sanity tests

### OpenGL test

```bash
glxgears
nvidia-smi
```

### ROS tools

```bash
rviz2
nvidia-smi
```

```bash
gzclient
nvidia-smi
```

They must appear in GPU process list.

---

## 16. Docker users (IMPORTANT)

When running containers:

```bash
docker run --gpus all \
  -e VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json \
  --network host \
  <image>
```

---

## 17. Common Problems & Fixes

### OpenGL shows Intel

* PRIME not set to NVIDIA
* Logged into Wayland

### Vulkan shows Intel / llvmpipe

* `VK_ICD_FILENAMES` not set
* Missing reboot

### Gazebo slow / black screen

* Vulkan wrong device
* VSYNC enabled

### `nvidia-smi` empty

* App not GPU accelerated
* Running inside Docker without `--gpus all`

---

## 18. Recovery (If something breaks)

Boot into recovery shell:

```bash
sudo nano /etc/environment
```

Comment Vulkan line:

```text
# VK_ICD_FILENAMES=...
```

Reboot.

---

## Final Status Checklist

* [x] PRIME = nvidia
* [x] OpenGL = NVIDIA
* [x] Vulkan = NVIDIA only
* [x] Kernel modules loaded
* [x] Gazebo / RViz GPU accelerated

---

## Conclusion

Your Ubuntu 22.04 system is now configured as a **full NVIDIA-only robotics workstation**, suitable for:

* ROS 2
* Gazebo Classic & Fortress
* RViz2
* Isaac Sim
* CUDA + Vulkan workloads

No further driver changes are required.
