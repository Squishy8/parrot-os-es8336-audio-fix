# parrot-os-es8336-audio-fix
Fix for missing headphone audio on Parrot OS using Intel Alder Lake and ES8336 codec, enabled via SOF (Sound Open Firmware) support.
## 🧩 The Problem

On laptops using **Intel Alder Lake processors** and the **ES8336 audio codec**, Parrot OS and other Debian-based distributions often fail to detect analog audio output.

Common symptoms:
- Only HDMI outputs are visible (devices 3, 7, 8, 9)
- No sound from built-in speakers or headphones
- The system loads `snd-hda-intel` but ignores the analog codec

---

## ✅ Confirmed Working Fix

This guide enables **audio through wired headphones** using **Sound Open Firmware (SOF)**.

### ✔️ Applies to:
- **Intel Alder Lake**
- **ES8336 Codec**
- Linux Kernel 6.1+ (tested on Parrot OS)
- Detected with:

```bash
lspci | grep -i audio
```
Expected output:
```
00:1f.3 Multimedia audio controller: Intel Corporation Alder Lake PCH-P High Definition Audio Controller (rev 01)
```

---

## 🛠️ Step-by-Step Solution

### 1. Check that SOF firmware is available

```bash
ls /lib/firmware/intel/sof/
```

You should see files like `sof-adl.ri`, `sof-essx8336`, etc.

---

### 2. Install necessary firmware and tools

```bash
sudo apt update
sudo apt install firmware-sof-signed firmware-linux firmware-linux-nonfree alsa-utils
```

> Ensure that `contrib non-free` is enabled in your `/etc/apt/sources.list`

---

### 2.5 Disable legacy HDA override (critical for Alder Lake)

If you've ever attempted to fix your audio manually in the past *(I have, and that’s why I’m including this section)*, you may have created a custom modprobe configuration at:
```bash
/etc/modprobe.d/alsa-base.conf
```
This file is **not present by default** in Parrot OS or other modern Debian-based systems. However, if it exists, and contains options like:
```bash
options snd-hda-intel model=generic
options snd-intel-dspcfg dsp_driver=1
```
...then your system is being forced to load the **legacy HDA driver** (`snd-hda-intel`), which is not compatible with **Intel Alder Lake + ES8336**.

This will prevent the correct driver from loading and cause analog outputs (like headphone jacks) to remain hidden.

#### 🧼 Fix it:

Check if the file exists:
```bash
sudo nano /etc/modprobe.d/alsa-base.conf
```

If it contains those lines, comment them out or delete them:
```bash
# options snd-hda-intel model=generic
# options snd-intel-dspcfg dsp_driver=1
```
 
Then save and close:
```plaintext
Ctrl + O, Enter, Ctrl + X
```

This allows the system to fall back to the correct SOF driver:
```bash
snd_sof_pci_intel_tgl
```

Which should expose the device:
```bash
card 0: sofessx8336 [sof-essx8336], device 0: ES8336
```

### 3. Reboot the system

```bash
sudo reboot
```

---

### 4. Confirm that the codec is now detected

```bash
aplay -l
```

```
card 0: sofessx8336 [sof-essx8336], device 0: ES8336
```

---

### 5. Enable headphone output in ALSA

```bash
alsamixer
```

- Press `F6`, select `sof-essx8336`
- Navigate using ← and → keys
- Press `M` to unmute `Headphone`, `Speaker`, and `Playback`
- Use ↑ to raise volume to 100%

---

### 6. Test audio

Plug in your wired headphones and run:

```bash
aplay -D plughw:0,0 /usr/share/sounds/alsa/Front_Center.wav
```

Or:
```bash
speaker-test -c 2 -t wav -D plughw:0,0
```

---

### 7. If device is busy, release it

```bash
sudo fuser -v /dev/snd/*
sudo killall pulseaudio
sudo killall pipewire
 ```

---

## 🎯 Result

- ✅ **Headphone audio works** via 3.5mm jack
- ❌ Internal speakers are still detected but produce no sound
- ✅ This fix restores functional analog audio for daily use

---

## 🙌 Credits

This guide is shared by a fellow Parrot OS user who went through several hours of trial and error. If it helps you — give it a star, share it, and help someone else too!
