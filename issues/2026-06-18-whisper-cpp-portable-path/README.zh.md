# whisper.cpp 便携部署：目录迁移后 dylib / venv 路径失效

## 概述

将 whisper.cpp 便携部署从原目录迁移到其他路径（或 U 盘插到另一台 Mac）后，二进制文件和 Python 虚拟环境均因硬编码绝对路径而无法运行。需要系统性地修复 dylib rpath、venv 配置文件及可执行脚本中的路径引用。

## 环境

- **设备**: Apple M1 (MacBook Air), 16 GB RAM
- **系统**: macOS 26 (Darwin 25.5.0, arm64)
- **项目**: [whisper-app](https://github.com/victcity/whisper-app)（基于 [whisper.cpp](https://github.com/ggml-org/whisper.cpp)）
- **编译参数**: `-DWHISPER_COREML=1 -DCMAKE_BUILD_TYPE=Release`
- **Python**: 3.12.10, virtualenv
- **关键依赖**: libwhisper, libggml, libggml-metal, libggml-blas

## 复现步骤

```bash
# 1. 在原目录 /Volumes/MyDisk/project/whisper-app 部署完成
cd /Volumes/MyDisk/project/whisper-app
./transcribe.sh -f test.mp3 -l zh   # ✅ 正常

# 2. 迁移到新路径
cp -r /Volumes/MyDisk/project/whisper-app /Volumes/USB/whisper-app

# 3. 在新路径尝试运行
cd /Volumes/USB/whisper-app
./whisper.cpp/build/bin/whisper-cli -m whisper.cpp/models/ggml-large-v3-turbo.bin -f test.wav
# ❌ dyld: Library not loaded: @rpath/libwhisper.1.dylib
#   Referenced from: /Volumes/USB/whisper-app/whisper.cpp/build/bin/whisper-cli
#   Reason: tried: '/Volumes/MyDisk/project/whisper-app/whisper.cpp/build/src/...' (no such file)

# 4. Python 脚本也无法运行
venv/bin/python3 scripts/transcribe.py
# ❌ bad interpreter: /Volumes/MyDisk/project/whisper-app/venv/bin/python3: no such file or directory
```

## 排查过程

1. **检查错误信息** → 明确是 dylib 加载路径（`@rpath`）和 Python 解释器路径均为迁移前的绝对路径
2. **分析涉及的文件** → 二进制 Mach-O 文件嵌入了来自 CMake 构建目录的 `-rpath` 指令；`pyvenv.cfg` 的 `home` 键和 venv/bin 下所有脚本的 shebang 均指向旧路径
3. **方案设计** → 用 `sed` 批量替换 venv 中的文本路径，用 `install_name_tool` 逐二进制修正 rpath，覆盖所有可能的子路径
4. **测试验证** → 修复后 `whisper-cli --help` 和 `venv/bin/python3 --version` 均正常

## 根因分析

**已确认**: CMake 在构建时会向 Mach-O 二进制嵌入指向构建目录的 `LC_RPATH` 指令（包括 `build/src`、`build/ggml/src`、`build/ggml/src/ggml-metal`、`build/ggml/src/ggml-blas`），这些路径在迁移后失效。Python virtualenv 的 `create` 命令会将创建时的 Python 解释器绝对路径写入 `pyvenv.cfg` 和所有可执行脚本的 shebang 行。

两者均是为实现"零全局依赖"而付出的代价——只有依赖绝对路径才能在不污染系统环境的前提下独立运行。

## 解决方案

迁移后执行以下路径修复脚本：

```bash
OLD="原路径"
NEW="$(pwd)"

# 1. 修复 venv
sed -i '' "s|$OLD|$NEW|g" venv/pyvenv.cfg
find venv/bin -type f -exec sed -i '' "s|$OLD|$NEW|g" {} +

# 2. 修复 whisper.cpp 二进制的 rpath
for f in whisper.cpp/build/bin/*; do
  file "$f" | grep -q "Mach-O" || continue
  for sub in build/src build/ggml/src build/ggml/src/ggml-blas build/ggml/src/ggml-metal; do
    install_name_tool -rpath "${OLD}/whisper.cpp/${sub}" "${NEW}/whisper.cpp/${sub}" "$f" 2>/dev/null || true
  done
done
```

### 注意事项

- `install_name_tool -rpath` 对不存在的旧路径会报错但无害（`|| true` 忽略）
- 仅修复 Mach-O 格式的二进制文件（`file "$f" | grep "Mach-O"` 筛选）
- 四个 rpath 子路径必须**全部替换**，遗漏任何一个都会导致部分 dylib 加载失败
- 此修复无需重新编译，迁移后可立即使用

## 相关链接

- [whisper.cpp 项目](https://github.com/ggml-org/whisper.cpp)
- [whisper-app 部署仓库](https://github.com/victcity/whisper-app)
- [Apple install_name_tool 文档](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/UsingDynamicLibraries.html)
