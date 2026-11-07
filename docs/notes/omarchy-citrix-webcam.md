# Webcam in Citrix Workspace on Omarchy (Intel IPU6 laptop)

Not managed by Nix: this is an Arch/Omarchy machine. Recorded here so the fix
can be redone after a reinstall or an `intel-ipu7-camera` update.

Symptom: Citrix HDX webcam redirection shows no camera in the remote app.

## Root causes (in the order they were found)

1. **Resolution mismatch.** The remote app asked for 640x480. The v4l2loopback
   relay (`/dev/video50`) only offers 1920x1080 NV12, so `gst_read` exits with
   `Resolution 640x480 is not supported in selected Webcam`.
2. **No H.264 encoder.** Citrix looks for `x264enc`, which is in
   `gst-plugins-ugly`, not installed by default.
3. **Wrong camera stack.** The controller is IPU6 (`8086:7d19`, driver
   `intel-ipu6`) but the packaged `intel-ipu7-camera` HAL fails with
   `failed to open library: .../ipu6epmtl.so`. The relay ran but had no live
   source.
4. **Relay sandbox blocked libcamera buffers.** `v4l2-relayd@.service` has
   `DevicePolicy=closed`; libcamera's software ISP needs `/dev/dma_heap/system`
   and `/dev/udmabuf`.
5. **NVIDIA EGL fails to debayer.** libcamera's GPU path on the NVIDIA EGL
   vendor errors with `glFrameBufferTexture2D error 36054`; Mesa (Intel) works.
6. **Privacy shutter closed.** The sensor returned fully saturated raw data
   (every pixel 1023) until the hardware shutter was opened.

## Fix

Client config, `~/.ICAClient/wfclient.ini`, under `[WFClient]`:

```ini
HDXWebCamEnabled=True
HDXWebCamEnablePnp=False       ; otherwise the server's requested size wins
HDXWebCamDevice=/dev/video50
HDXWebCamWidth=1920
HDXWebCamHeight=1080
HDXH264InputEnabled=True
```

Packages:

```sh
omarchy pkg add gst-plugins-ugly libcamera libcamera-tools gst-plugin-libcamera
```

`/etc/v4l2-relayd.d/ipu7.conf`, replacing the `icamerasrc` `VIDEOSRC`:

```sh
VIDEOSRC="libcamerasrc ! video/x-raw,width=1920,height=1080,framerate=30/1 ! videoconvert"
```

`/etc/systemd/system/v4l2-relayd@ipu7.service.d/90-libcamera.conf`:

```ini
[Service]
Environment=__EGL_VENDOR_LIBRARY_FILENAMES=/usr/share/glvnd/egl_vendor.d/50_mesa.json
DeviceAllow=/dev/dma_heap/system rw
DeviceAllow=/dev/udmabuf rw
```

Then `sudo systemctl daemon-reload && sudo systemctl restart v4l2-relayd@ipu7`
and reconnect the Citrix session.

## Verifying

- `cam -l` lists the sensor.
- Read raw frames from the loopback and check they vary and are not white:
  `gst-launch-1.0 v4l2src device=/dev/video50 io-mode=mmap ! ...`.
- Raw sensor check: `cam -c 1 -s role=raw --capture=1 --file=...`; all values
  at 1023 means the shutter is closed or the sensor is overexposed.
- `gst_read` run standalone without `-b` emits a test pattern; its output does
  not prove the real camera works.
- Session log: `journalctl --user -t wfica -t citrix-gst_read`.

## Known quirks

- The libcamera software auto-exposure restarts at minimum on each open, so
  video takes about 12 s to reach normal brightness.
- Opening the relay again immediately after a close can return a single stale
  frame.
- `intel-ipu7-camera` updates may rewrite `ipu7.conf`; the `.bak-*` copies and
  this note have the previous values.
