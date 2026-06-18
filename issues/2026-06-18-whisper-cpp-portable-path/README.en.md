# whisper.cpp Portable Deployment: dylib & venv Path Breakage After Migration

## Summary

After moving a whisper.cpp portable deployment to a different directory (or plugging a USB drive into another Mac), both the compiled binaries and the Python virtual environment fail due to hardcoded absolute paths. A systematic fix is needed for dylib rpath entries, venv configuration, and script shebangs.

## Environment

- **Device**: Apple M1 (MacBook Air), 16 GB RAM
- **OS**: macOS 26 (Darwin 25.5.0, arm64)
- **Project**: whisper-app (based on [whisper.cpp](https://github.com/ggml-org/whisper.cpp))
- **Build flags**: `-DWHISPER_COREML=1 -DCMAKE_BUILD_TYPE=Release`
- **Python**: 3.12.10, virtualenv
- **Key dependencies**: libwhisper, libggml, libggml-metal, libggml-blas

## Reproduction

```bash
# 1. Deploy and verify at original path
cd /Volumes/MyDisk/project/whisper-app
./transcribe.sh -f test.mp3 -l zh   # ✅ works

# 2. Migrate to new location
cp -r /Volumes/MyDisk/project/whisper-app /Volumes/USB/whisper-app

# 3. Attempt to run at new location
cd /Volumes/USB/whisper-app
./whisper.cpp/build/bin/whisper-cli -m whisper.cpp/models/ggml-large-v3-turbo.bin -f test.wav
# ❌ dyld: Library not loaded: @rpath/libwhisper.1.dylib
#   Referenced from: /Volumes/USB/whisper-app/whisper.cpp/build/bin/whisper-cli
#   Reason: tried: '/Volumes/MyDisk/project/whisper-app/whisper.cpp/build/src/...' (no such file)

# 4. Python scripts also fail
venv/bin/python3 scripts/transcribe.py
# ❌ bad interpreter: /Volumes/MyDisk/project/whisper-app/venv/bin/python3: no such file or directory
```

## Investigation

1. **Analyzed error messages** → confirmed dylib loading paths (`@rpath`) and Python interpreter path both reference the pre-migration absolute path
2. **Identified affected files** → Mach-O binaries contain `LC_RPATH` entries pointing to CMake build subdirectories; `pyvenv.cfg` and all venv/bin scripts contain the old absolute path in their shebang
3. **Designed fix** → use `sed` for text-based venv paths, use `install_name_tool` for binary rpath entries, covering all subdirectory variants
4. **Verified** → `whisper-cli --help` and `venv/bin/python3 --version` both function correctly after the fix

## Root Cause Analysis

**Confirmed**: During CMake builds, Mach-O binaries receive `LC_RPATH` entries pointing to the build directory (`build/src`, `build/ggml/src`, `build/ggml/src/ggml-metal`, `build/ggml/src/ggml-blas`). These become invalid after relocation. Python's `virtualenv` writes the creating interpreter's absolute path into `pyvenv.cfg` and all script shebangs.

Both are the trade-off for zero-global-dependency portability — absolute paths enable isolated execution without polluting the host system.

## Solution

Run the following path migration fix after relocating:

```bash
OLD="original/path"
NEW="$(pwd)"

# 1. Fix venv
sed -i '' "s|$OLD|$NEW|g" venv/pyvenv.cfg
find venv/bin -type f -exec sed -i '' "s|$OLD|$NEW|g" {} +

# 2. Fix rpath in whisper.cpp binaries
for f in whisper.cpp/build/bin/*; do
  file "$f" | grep -q "Mach-O" || continue
  for sub in build/src build/ggml/src build/ggml/src/ggml-blas build/ggml/src/ggml-metal; do
    install_name_tool -rpath "${OLD}/whisper.cpp/${sub}" "${NEW}/whisper.cpp/${sub}" "$f" 2>/dev/null || true
  done
done
```

### Notes

- `install_name_tool -rpath` on non-existent old paths produces harmless errors (`|| true` handles this)
- Only Mach-O files are targeted (`file "$f" | grep "Mach-O"` filter)
- All four rpath sub-paths must be replaced — missing any one will cause partial dylib load failures
- No recompilation needed; the deployment works immediately after the fix

## References

- [whisper.cpp project](https://github.com/ggml-org/whisper.cpp)
- [whisper-app deployment repo](https://github.com/victcity/whisper-app)
- [Apple Dynamic Library Programming](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/UsingDynamicLibraries.html)
