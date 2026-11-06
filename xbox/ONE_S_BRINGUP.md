# Xbox One / One S bring-up

## Status

This branch is an experimental compatibility bring-up. It does **not** yet establish that the game runs on Xbox One S, and it must not be described as a supported release until the hardware tests below pass.

The starting point is the existing Series X|S implementation. The first One-generation-specific change is the CoreWindow presentation path; renderer features, texture handling, memory pressure, CPU instructions, and sustained gameplay still need independent validation on real hardware.

## Hardware identification

The graphics probe calls `GetGamingDeviceModelInformation` and uses the official device IDs:

| Device | ID |
|---|---:|
| Xbox One | `0x768BAE26` |
| Xbox One S | `0x2A7361D9` |
| Xbox One X | `0x5AD617C7` |

The API is documented in [Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/gamingdeviceinformation/nf-gamingdeviceinformation-getgamingdevicemodelinformation); the IDs are listed in the [GAMING_DEVICE_DEVICE_ID enumeration](https://learn.microsoft.com/windows/win32/api/gamingdeviceinformation/ne-gamingdeviceinformation-gaming_device_device_id).

## CoreWindow presentation paths

The Xbox One generation uses the documented Xbox One convention:

- Pass the `ID3D12Device*` to `IDXGIFactory2::CreateSwapChainForCoreWindow`, not the command queue.
- Use `DXGI_SWAP_EFFECT_FLIP_SEQUENTIAL`.
- Keep the D3D12 command queue for command submission, copies, and fences.

The Series X|S path remains unchanged: it passes the command queue and uses `DXGI_SWAP_EFFECT_FLIP_DISCARD`.

Microsoft's reference implementation is [Xbox-ATG-Samples / SimpleCompute12 / DeviceResources.cpp](https://github.com/microsoft/Xbox-ATG-Samples/blob/main/XDKSamples/IntroGraphics/SimpleCompute12/DeviceResources.cpp), which explicitly comments that Xbox One uses the device rather than the command queue.

## First hardware test: graphics bridge

### Build the test package

The workflow is named **Xbox build**. Open the repository's **Actions** tab, select **Xbox build**, choose **Run workflow**, and select the `xbox-one-s/bring-up` branch. Download the generated package artifact from that run. The workflow also listens for pushes to this branch. Its release step is explicitly disabled for this branch, so it cannot publish an experimental One S package as a release.

1. Install the package in Developer Mode and keep its application type set to **Game**.
2. Put `graphics` on the first line of `LocalState/run.txt`.
3. Launch the app. This diagnostic mode does not require a game dump.
4. Inspect `LocalState/loader.log`, `LocalState/errors.log`, and `LocalState/dozen.log`.

The log should identify the device as `0x2a7361d9` and report the `Xbox One generation` presentation path. Before creating the CoreWindow swap chain, the bridge also records the actual D3D12 values reported by the console:

- Feature level and highest recognized/supported Shader Model.
- Resource binding tier and tiled-resource tier.
- `D3D12_OPTIONS4`: shared-resource compatibility tier, native 16-bit shader operations, and 64 KiB-aligned MSAA support.
- `R8G8B8A8_UNORM` format support flags, used by the Vulkan/D3D12 shared presentation image.
- Vulkan's external-image query for the exact RGBA8 usage and `VK_EXTERNAL_MEMORY_HANDLE_TYPE_D3D12_RESOURCE_BIT`, including exportability and compatible handle types. The bridge stops with an explicit error if Vulkan does not report the image as exportable.

Treat these values as runtime evidence, not assumptions based on the console family. In particular, do not enable native 16-bit shader operations solely because the shader model query succeeds.

A shared-resource tier of `0` is not the same as "no sharing": Microsoft's tier-0 list explicitly includes `DXGI_FORMAT_R8G8B8A8_UNORM`, the bridge's current format. The definitive test for this path remains successful Vulkan export plus D3D12 `OpenSharedHandle`. See [Microsoft's tier definitions](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_shared_resource_compatibility_tier).

The new `Graphics probe stage:` records show the last completed Vulkan/DXGI stage. Within the presentation bridge, `Graphics bridge stage:` records distinguish adapter lookup, D3D12 device creation, CoreWindow swap-chain creation, external Vulkan image allocation, memory export, and D3D12 handle opening. A successful first milestone ends with:

- `CoreWindow first frame presented`
- `Graphics probe: Vulkan clear -> shared D3D12 image -> CoreWindow PRESENTED`
- `Graphics probe stage: completed successfully; blue test frame should be visible`

A blue frame only validates the narrow presentation/interoperability path. It does not validate the game's renderer, shared-image formats used by gameplay, native 16-bit shader operations, or texture detiling.

## Remaining One S validation gates

Do not move the branch's support status until these are tested on an actual Xbox One S:

1. D3D12 feature queries, shader-model ceiling, `D3D12_OPTIONS4`, native 16-bit shader operations, and shared-resource compatibility.
2. Vulkan device creation and required extension/feature fallbacks in Dozen.
3. External image sharing, barriers, synchronization, presentation, and resize/suspend/resume.
4. Texture probe formats, mutable views, detiling, readbacks, depth/stencil, and real game texture correctness.
5. Jaguar CPU feature checks and a build that does not emit unsupported AVX2/FMA3/BMI2 instructions.
6. USB and internal dump file access, controller input, XAudio2, video decode, save/load, and app lifecycle.
7. Long gameplay and memory-pressure testing against the One S app memory budget. Do not infer performance from Series X results or promise a frame rate before measurement.

Every test result should retain the exact device ID, OS build, app package version, Dozen build, and the three diagnostic logs.
