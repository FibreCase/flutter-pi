# DRM/KMS hardware color calibration

`--gamma R,G,B` accepts three finite positive exponents; `--rgb-gain R,G,B`
accepts three finite nonnegative gains. Both default to `1,1,1`.
Options must precede the application bundle path.

```sh
flutter-pi --rgb-gain 1,1,0.95 /path/to/app
flutter-pi --gamma 1,1,1.05 --rgb-gain 1,1,0.95 /path/to/app
# Explicitly restore a linear LUT:
flutter-pi --gamma 1,1,1 --rgb-gain 1,1,1 /path/to/app
```

Without either option, flutter-pi leaves the existing CRTC gamma state untouched,
exactly as before. This does not reset calibration left by an earlier process.
Explicit identity parameters install a linear LUT. Other color transforms or
panel calibration are outside this feature.

## Build and test on Armbian/Linux

Install the normal dependencies described in README.md, including libdrm-dev.

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j2
ctest --test-dir build --output-on-failure
```

The connected display must support atomic KMS and expose both GAMMA_LUT and
GAMMA_LUT_SIZE on the selected CRTC. Explicit calibration fails startup with a
clear error if these are unavailable, if the LUT size is below two, or if blob
allocation fails. The feature does not use the legacy gamma API.

## Verify on the device

While flutter-pi is running, inspect the selected CRTC using `modetest -p`
(or `modetest -M <driver> -p` for the appropriate DRM driver). Its GAMMA_LUT
property should reference a nonzero blob ID and GAMMA_LUT_SIZE should match the
driver's reported size. Do not run a competing modeset from modetest.
Compare grayscale ramps and white/color patches with the blue gain example and
with explicit identity parameters. Property inspection confirms assignment;
visual comparison confirms the panel/driver actually uses the LUT.

## Implementation and ownership

- flutter-pi.c / flutter-pi.h: parse, validate and pass RGB parameters; reject
  calibration on a dummy display; document options in command-line help.
- modesetting.c / modesetting.h: retain the enumerated LUT size and expose a
  request-builder setter. Track the last successfully committed gamma blob so
  queued initialization frames do not upload it repeatedly.
- window.c / window.h: allocate one LUT using the selected CRTC size, calculate
  `round(clamp(pow(x, gamma) * gain, 0, 1) * 65535)`, create one DRM blob,
  and add it alongside the initial modeset. Clear the modeset flag after a
  successful calibrated frame. Uncalibrated startup keeps its existing behavior.
- test/flutterpi_test.c: cover defaults, valid combined parameters and malformed,
  nonfinite, overflowing and out-of-range input.

The KMS window owns the blob. Temporary LUT memory is freed immediately after
blob creation. Initialization failures destroy the blob; window teardown destroys
its userspace blob reference before releasing the DRM device. Scheduled frames
retain the window until presentation or cancellation, keeping its blob alive.
After assignment, the kernel retains its own reference to the active blob;
userspace destruction does not reset the display. Explicit identity parameters
restore a linear curve. No per-frame pixel processing or LUT creation occurs.

The existing modeset flag was never cleared on successful presentation. For
calibrated windows this is now cleared after successful commit, and pending
initial frames are deduplicated at commit time. Failed commits remain retryable.

Validation on the development Mac is limited: CMake stops at missing libdrm,
so a complete Linux build and actual DRM hardware verification remain required.
