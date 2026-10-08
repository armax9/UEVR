# Experimental Wine / CrossOver build

Based on upstream UEVR commit `4ee5c6b6162dee2291fc75f9dfc57667f6d45a2d`.
This is an unofficial experimental backend for macOS using Wine/CrossOver and
D3DMetal. The reported VR test used MoltenVR as the OpenXR runtime. It is not a general Windows build or a guarantee of compatibility with every DX12 game.

## Changes

- Preserve the loaded `D3D12CreateDevice` export instead of temporarily replacing
  its code with bytes from the DLL on disk. Log the device creation result.
- Create the D3D11 hardware device and DXGI swapchain separately. Use a private
  64 x 64 test window instead of the desktop window. Log each initialization stage.
- Apply the reported OpenXR stick correction: negate left X/Y and right X;
  keep right Y unchanged. This correction is specific to the reported setup.
- Enable all three workarounds by default without registry configuration.

## DX12 changes in 0.2

When command-queue offset scanning fails, capture DIRECT queues through
`ExecuteCommandLists`. Match the supported `ID3D12Device` interface to the
presenting swapchain, preferring an unambiguous queue submitted on the present
thread. Do not guess when matching remains ambiguous. Retain queue references
while hooked and release them when unhooking.

The tested D3DMetal device returned `E_NOINTERFACE` for `IID_IUnknown` despite
supporting `ID3D12Device` and `ID3D12Device4`, so the fallback compares the same
supported device interface instead. Diagnostics identify queue selection.

## Installation

1. Close the game and injector. Back up the existing UEVR directory.
2. Use the upstream nightly matching the base commit above.
3. Replace `UEVRBackend.dll` beside `UEVRInjector.exe` with this build.
4. Keep the existing `openvr_api.dll` and `openxr_loader.dll`.
5. An optional matching `UEVRBackend.pdb` helps diagnose crashes.
6. Restart the CrossOver bottle and launch the game and injector in that bottle.
   Select OpenXR and inject after loading into the game.

No registry parameters are required. Optional environment overrides set to `0`
disable individual workarounds (restart the bottle after changing them):

| Variable | Workaround |
| --- | --- |
| `UEVR_D3D12_PRESERVE_EXPORT` | Preserve the D3D12 export |
| `UEVR_D3D11_COMPAT` | Separate hardware D3D11 device / private-window swapchain |
| `UEVR_OPENXR_STICK_FIX` | Left X/Y and right X sign correction |

## Validation and limitations

The user built the initialization patches on Windows with Visual Studio 2026.
Call of the Sea initialized D3D11/OpenXR successfully under CrossOver/D3DMetal;
the log reports `Successfully created OpenXR swapchains for D3D11`.
The user reported that lowering the resolution in MoltenVR restored usable FPS.
This does not establish a performance guarantee for other resolutions or games.

Hogwarts Legacy now works in VR according to the user's test. The latest log
confirms matching device pointers and submission thread, Framework initialization,
D3D12 texture setup and the first worker-thread submit. Engine-scanning errors
were also present, so this is a tested workaround, not complete compatibility.

The source retains the OpenXR stick workaround from 0.1. The user reported working
sticks but later clarified that the standalone stick patch had been skipped;
binary/source equivalence for that optional correction is not established.
The 0.2 user-supplied DLL contains the DX12 step-3 diagnostic markers, but is not a
reproducible build produced by this repository's CI. No PDB was supplied.
Diagnostics currently log device matching frequently; log size and performance
should be monitored. Queue matching can still fail with multiple DIRECT queues,
other device wrappers or runtimes. September 7th is not confirmed working in VR.
The injector's WPF rendering issue is separate and is not fixed by this backend.

## Build

Use Windows, Visual Studio 2026 with Desktop development with C++, Windows SDK,
CMake and Git. The UESDK submodule requires an Epic-authorized GitHub account.
After checking out this branch:

```powershell
git config submodule.dependencies/submodules/UESDK.url https://github.com/praydog/UESDK.git
git submodule update --init --recursive
cmake -S . -B build -G "Visual Studio 18 2026" -A x64 -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release --target uevr
```

Output: `build/bin/uevr/UEVRBackend.dll`.
Existing upstream licenses continue to apply; see `LICENSE` and dependency licenses.
