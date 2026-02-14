---
layout: post
title:  "Getting Hardware Transcoding Working in Jellyfin on Proxmox LXC"
date:   2026-02-14
categories: homelab
tags: proxmox lxc jellyfin amd vaapi transcoding gpu
---

I recently set up [Jellyfin][jellyfin] in an LXC container on Proxmox to manage my media library. It worked great out of the box - until I tried playing certain videos. Some would load fine, others would spin for a second and then just... close. No error message, no fallback to a lower quality. Just nothing.

The culprit turned out to be hardware transcoding - or rather, a mismatch between what Jellyfin *thought* my GPU could do and what it *actually* could do. This post covers the full setup for GPU passthrough to an LXC container, and more importantly, the subtle configuration gotcha that had me chasing my tail.

**A quick note:** My setup uses an AMD Renoir APU (Ryzen 4000 series integrated graphics). The general approach applies to other AMD GPUs, but the specific codec support will vary. Intel users will want QSV instead of VA-API.

## The Setup

Here's what I'm working with:

| Component | Value |
|-----------|-------|
| Proxmox Host | AMD Renoir APU |
| Container OS | Ubuntu 22.04 LTS |
| Container Type | Unprivileged LXC |
| Jellyfin | Latest (via community script) |
| Media Storage | NFS mounts from a Synology NAS |

The media library is a mix of everything - old DVDs ripped years ago (MPEG4/XviD), Blu-ray rips (H.264, HEVC), and some newer stuff. The older files are the ones that consistently need transcoding, since most clients don't natively support MPEG4 ASP.

## Step 1: Verify the GPU on the Proxmox Host

First things first - make sure the GPU devices exist on the host:

```bash
ls -la /dev/dri/
```

You should see something like:

```
crw-rw---- 1 root video  226,   0 Feb  4 18:53 card0
crw-rw---- 1 root render 226, 128 Feb  4 18:53 renderD128
```

Note the group names and their GIDs - you'll need these later:

```bash
stat -c '%g' /dev/dri/card0        # video GID (e.g. 44)
stat -c '%g' /dev/dri/renderD128   # render GID (e.g. 104)
```

And confirm what GPU you're actually working with:

```bash
lspci | grep -iE 'vga|display|3d'
```

## Step 2: Pass the GPU Through to the Container

Edit your container's config file on the Proxmox host (replace `104` with your container ID):

```bash
nano /etc/pve/lxc/104.conf
```

Add these lines for GPU passthrough:

```
dev0: /dev/dri/card0,gid=44
dev1: /dev/dri/renderD128,gid=104
```

This is the Proxmox 7+ method. The `gid` values tell Proxmox which group should own the device inside the container. Make sure they match your host - run the `stat` commands above if you're not sure.

There's an older method using raw LXC mount entries that you'll see in a lot of guides:

```
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file 0 0
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file 0 0
```

Both work. The `dev0`/`dev1` syntax is cleaner and handles permissions more reliably.

## Step 3: Handle the Boot Race Condition

There's a subtle issue with `onboot` containers - the GPU device nodes might not exist yet when the container tries to start after a host reboot. The container starts, can't find `/dev/dri/renderD128`, and GPU passthrough silently fails.

The fix is a hookscript that waits for the device before starting the container:

```bash
mkdir -p /var/lib/vz/snippets

cat > /var/lib/vz/snippets/gpu-wait.sh << 'EOF'
#!/bin/bash
if [ "$2" == "pre-start" ]; then
    for i in {1..30}; do
        if [ -e /dev/dri/renderD128 ]; then
            exit 0
        fi
        sleep 1
    done
fi
exit 0
EOF

chmod +x /var/lib/vz/snippets/gpu-wait.sh
```

Attach it to the container:

```bash
pct set 104 --hookscript local:snippets/gpu-wait.sh
```

This gives the GPU driver up to 30 seconds to initialise before the container starts. In practice it's usually ready within a few seconds, but better safe than sorry.

## Step 4: Fix Permissions Inside the Container

Restart the container and add the Jellyfin user to the required groups:

```bash
pct restart 104
pct exec 104 -- usermod -aG video jellyfin
pct exec 104 -- usermod -aG render jellyfin
pct exec 104 -- systemctl restart jellyfin
```

Verify it took:

```bash
pct exec 104 -- id jellyfin
```

You should see both `video` and `render` in the groups list:

```
uid=110(jellyfin) gid=118(jellyfin) groups=118(jellyfin),44(video),104(render)
```

## Step 5: Verify VA-API Works

Check that the GPU is accessible and VA-API is functioning:

```bash
pct exec 104 -- vainfo
```

If `vainfo` isn't installed, grab it first:

```bash
pct exec 104 -- apt install -y vainfo mesa-va-drivers
```

You want to see it successfully load the driver and list supported profiles:

```
libva info: VA-API version 1.14.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/radeonsi_drv_video.so
libva info: Found init function __vaDriverInit_1_14
libva info: va_openDriver() returns 0
vainfo: Driver version: Mesa Gallium driver ... for RENOIR
vainfo: Supported profile and entrypoints
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointEncSlice
      VAProfileVP9Profile0            : VAEntrypointVLD
      ...
```

You'll probably see a "can't connect to X server" warning - ignore it, it's harmless.

The important thing to note here is what's listed. `VAEntrypointVLD` means hardware decode, `VAEntrypointEncSlice` means hardware encode. **Write down which codecs your GPU actually supports.** This is crucial for the next step.

## Step 6: Configure Jellyfin (The Part That Actually Matters)

Open the Jellyfin web UI and go to **Dashboard > Playback > Transcoding**.

Set these:

1. **Hardware acceleration**: VA-API
2. **VA-API Device**: `/dev/dri/renderD128`
3. **Enable hardware encoding**: Yes
4. **Allow encoding in HEVC format**: Yes (if your GPU supports it)

For the hardware decoding codecs, **only enable what `vainfo` confirmed your GPU supports**. For my Renoir APU, that's:

- H.264
- HEVC
- HEVC 10-bit
- MPEG2
- VC1
- VP9

And critically: **disable "Allow encoding in AV1 format"**.

## The AV1 Gotcha

This is what was actually breaking my setup, and it's easy to miss.

Jellyfin's default configuration enables AV1 encoding. The Renoir APU (and most AMD GPUs older than RDNA 3) does not support AV1 encoding in hardware. When a video needs transcoding, Jellyfin asks FFmpeg to encode to AV1 via VA-API, FFmpeg can't find a usable encoding profile, and the whole transcode fails silently. The client just sees... nothing.

Here's what the FFmpeg log looks like when this happens:

```
Stream mapping:
  Stream #0:1 -> #0:0 (mpeg4 (native) -> av1 (av1_vaapi))

[av1_vaapi @ 0x732236ae7280] No usable encoding profile found.
[vost#0:0/av1_vaapi @ 0x732236adb700] Error while opening encoder
Conversion failed!
```

No fallback. No retry with a different codec. Just failure.

The fix is straightforward. Either disable it in the web UI as described above, or edit the config file directly:

```bash
nano /etc/jellyfin/encoding.xml
```

Make sure `AllowAv1Encoding` is set to `false`:

```xml
<AllowAv1Encoding>false</AllowAv1Encoding>
```

And check the `HardwareDecodingCodecs` list doesn't include codecs your GPU can't handle:

```xml
<HardwareDecodingCodecs>
  <string>h264</string>
  <string>hevc</string>
  <string>mpeg2video</string>
  <string>vc1</string>
  <string>vp9</string>
</HardwareDecodingCodecs>
```

No `av1`. No `vp8`. Only what `vainfo` says your GPU supports.

Restart Jellyfin after making changes:

```bash
systemctl restart jellyfin
```

## Troubleshooting

### Videos spin briefly then close

This is the symptom I had. Check the most recent FFmpeg transcode log:

```bash
ls -lt /var/log/jellyfin/FFmpeg.Transcode*.log | head -5
```

Read the latest one and look for:

- `No usable encoding profile found` - You've enabled a codec your GPU doesn't support
- `Error opening input: No such file or directory` - Media file path issue (check your NAS mounts)
- `Conversion failed!` - Generic failure, check the lines above it for the real error

### vainfo shows no profiles

Make sure the VA-API drivers are installed:

```bash
apt install mesa-va-drivers
```

Newer AMD GPUs (RDNA 2+) may need a more recent Mesa than what Ubuntu 22.04 ships. The [Kisak PPA][kisak-ppa] is a good option if you're stuck on 22.04.

### Permission denied on /dev/dri

Verify the Jellyfin user is in both groups:

```bash
id jellyfin
```

If the GIDs don't match between host and container, the devices will appear to exist but be inaccessible. This fails silently - Jellyfin just falls back to software transcoding (or fails entirely). Double-check with:

```bash
ls -la /dev/dri/
```

The group ownership inside the container should match what Jellyfin's user belongs to.

## Hardware Support Reference

What your GPU can actually do matters. Here's what the Renoir APU supports:

| Codec | Decode | Encode |
|-------|--------|--------|
| H.264 | Yes | Yes |
| HEVC/H.265 | Yes | Yes |
| HEVC 10-bit | Yes | Yes |
| VP9 | Yes | No |
| MPEG2 | Yes | No |
| VC1 | Yes | No |
| AV1 | No | No |
| VP8 | No | No |

This will be different for other AMD GPUs. RDNA 3 (RX 7000 series) adds AV1 encode support, for instance. Always check `vainfo` on your specific hardware rather than assuming.

## Complete LXC Config

For reference, here's my full container config:

```
arch: amd64
cores: 2
dev0: /dev/dri/card0,gid=44
dev1: /dev/dri/renderD128,gid=104
features: keyctl=1,nesting=1
hookscript: local:snippets/gpu-wait.sh
hostname: jellyfin
memory: 2048
mp0: /mnt/movies,mp=/movies
mp1: /mnt/music,mp=/music
mp2: /mnt/shows,mp=/shows
net0: name=eth0,bridge=vmbr0,hwaddr=XX:XX:XX:XX:XX:XX,ip=dhcp,type=veth
onboot: 1
ostype: ubuntu
rootfs: local-lvm:vm-104-disk-0,size=18G
swap: 512
unprivileged: 1
```

The `mp0`/`mp1`/`mp2` entries are NFS mounts from my Synology NAS, passed through as mount points so Jellyfin can access the media directly.

## Was It Worth the Effort?

For me, yes. Hardware transcoding makes a real difference - my Renoir APU handles multiple concurrent transcode streams without breaking a sweat, whereas software transcoding would peg the CPU and struggle with more than one stream.

The actual GPU passthrough setup is straightforward. It's the Jellyfin codec configuration that's the trap. The symptom ("videos just don't play") gives you almost nothing to go on, and the fix is a single checkbox buried in the transcoding settings. I spent more time staring at logs than I did on the entire LXC configuration.

If you're hitting similar issues, check your FFmpeg logs. The answer is almost always in there.

---

If you have questions or spot something I've got wrong, drop me an email at [mattcree@proton.me](mailto:mattcree@proton.me).

[jellyfin]: https://jellyfin.org/
[kisak-ppa]: https://launchpad.net/~kisak/+archive/ubuntu/kisak-mesa
