# MAME 编译为 WebAssembly 指南

> 适用系统：CentOS 8 / Rocky Linux  
> 目标：将 MAME 编译为可在浏览器中运行的 WebAssembly

---

## 环境准备

### 系统依赖

```bash
dnf install -y git python3 cmake make
```

### 安装 Emscripten (emsdk)

```bash
git clone https://github.com/emscripten-core/emsdk.git /opt/work/emsdk
cd /opt/work/emsdk
./emsdk install 3.1.50
./emsdk activate 3.1.50
source ./emsdk_env.sh
```

### 克隆 MAME 源码

```bash
git clone https://github.com/mamedev/mame.git /opt/work/mame
cd /opt/work/mame
```

---

## 编译命令

> ⚠️ **不能编译完整 MAME**，产物太大浏览器无法加载，必须用 `SOURCES` 指定特定驱动。

```bash
cd /opt/work/mame

emmake make \
  SUBTARGET=psikyo \
  SOURCES=src/mame/psikyo/psikyo.cpp \
  REGENIE=1 \
  -j$(nproc)
```

编译成功后生成：
- `psikyo.js` — Emscripten 胶水代码
- `psikyo.wasm` — WebAssembly 二进制

---

## 离线处理 Emscripten Ports（重点）

Emscripten 编译时会自动下载第三方依赖（ports），在无法访问外网的环境中会卡住。  
需要手动下载并放置到正确位置。

### 原理说明

Emscripten ports 的缓存目录由 `config.PORTS` 决定：

```bash
# 查看实际路径
python3 -c "
import sys
sys.path.insert(0, '/opt/work/emsdk/upstream/emscripten')
from tools import config
print('PORTS dir:', config.PORTS)
"
# 输出：/opt/work/emsdk/upstream/emscripten/cache/ports
```

`fetch_project` 函数的逻辑：
1. 检查 `cache/ports/<name>/.emscripten_url` 文件是否存在
2. 文件内容与下载 URL 一致 → 跳过下载，直接使用本地文件
3. 不一致或不存在 → 去网络下载

**因此只需：解压 zip + 写入 `.emscripten_url` 文件，即可跳过下载。**

---

### MAME 所需 Ports 清单

| Port | 下载地址 | 本地文件名 |
|------|---------|-----------|
| freetype | `https://github.com/emscripten-ports/FreeType/archive/version_1.zip` | `freetype.zip` |
| sdl2 | `https://github.com/libsdl-org/SDL/archive/release-2.24.2.zip` | `sdl2.zip` |
| harfbuzz | `https://github.com/harfbuzz/harfbuzz/releases/download/3.2.0/harfbuzz-3.2.0.tar.xz` | `harfbuzz.tar.xz` |
| sdl2_ttf | `https://github.com/libsdl-org/SDL_ttf/archive/release-2.20.2.zip` | `sdl2_ttf.zip` |

### 下载文件（在能联网的机器上执行）

```bash
wget https://github.com/emscripten-ports/FreeType/archive/version_1.zip -O freetype.zip
wget https://github.com/libsdl-org/SDL/archive/release-2.24.2.zip -O sdl2.zip
wget https://github.com/harfbuzz/harfbuzz/releases/download/3.2.0/harfbuzz-3.2.0.tar.xz -O harfbuzz.tar.xz
wget https://github.com/libsdl-org/SDL_ttf/archive/release-2.20.2.zip -O sdl2_ttf.zip
```

### 验证 Hash（可选但推荐）

```bash
sha512sum freetype.zip
# 期望：0d0b1280ba0501ad0a23cf1daa1f86821c722218b59432734d3087a89acd22aa...

sha512sum sdl2.zip
# 期望：b178bdc8f7c40271e09a72f639649d1d61953dda4dc12b77437259667b63b961...
```

> Hash 值可在对应的 `.py` 文件中查看：
> ```bash
> grep "HASH" /opt/work/emsdk/upstream/emscripten/tools/ports/sdl2.py
> ```

### 部署脚本

将下载好的文件传到 VM 后，执行以下脚本一次性部署所有 ports：

```bash
cat << 'EOF' > /tmp/setup_ports.sh
#!/bin/bash
set -e
PORTS_DIR=/opt/work/emsdk/upstream/emscripten/cache/ports

setup_port() {
  local name=$1
  local url=$2
  local file=$3

  echo "=== Setting up $name ==="
  mkdir -p $PORTS_DIR/$name

  # 写入 marker 文件（让 emscripten 认为已缓存）
  echo "$url" > $PORTS_DIR/$name/.emscripten_url

  # 解压到对应目录
  if [[ "$file" == *.zip ]]; then
    unzip -q $PORTS_DIR/$file -d $PORTS_DIR/$name
  elif [[ "$file" == *.tar.xz ]]; then
    tar xf $PORTS_DIR/$file -C $PORTS_DIR/$name
  elif [[ "$file" == *.tar.gz ]]; then
    tar xzf $PORTS_DIR/$file -C $PORTS_DIR/$name
  fi

  echo "=== Done: $name ==="
}

setup_port "freetype" \
  "https://github.com/emscripten-ports/FreeType/archive/version_1.zip" \
  "freetype.zip"

setup_port "sdl2" \
  "https://github.com/libsdl-org/SDL/archive/release-2.24.2.zip" \
  "sdl2.zip"

setup_port "harfbuzz" \
  "https://github.com/harfbuzz/harfbuzz/releases/download/3.2.0/harfbuzz-3.2.0.tar.xz" \
  "harfbuzz.tar.xz"

setup_port "sdl2_ttf" \
  "https://github.com/libsdl-org/SDL_ttf/archive/release-2.20.2.zip" \
  "sdl2_ttf.zip"

echo ""
echo "All ports deployed successfully!"
echo "Verifying marker files:"
cat $PORTS_DIR/freetype/.emscripten_url
cat $PORTS_DIR/sdl2/.emscripten_url
cat $PORTS_DIR/harfbuzz/.emscripten_url
cat $PORTS_DIR/sdl2_ttf/.emscripten_url
EOF

chmod +x /tmp/setup_ports.sh
bash /tmp/setup_ports.sh
```

---

## 部署到 Web 服务器

编译完成后，需要配合 [Emularity](https://github.com/db48x/emularity) 加载器来运行。

### 所需文件

```
web/
├── psikyo.js          # emmake 编译输出
├── psikyo.wasm        # emmake 编译输出
├── es6-promise.js     # Emularity
├── browserfs.min.js   # Emularity
├── loader.js          # Emularity
├── pacman.zip         # ROM 文件
└── index.html         # 加载页
```

### index.html 示例

```html
<!DOCTYPE html>
<html>
<head>
  <title>MAME WASM</title>
</head>
<body>
<canvas id="canvas"></canvas>
<script src="es6-promise.js"></script>
<script src="browserfs.min.js"></script>
<script src="loader.js"></script>
<script>
  start({
    "mame": "psikyo.js",
    "driver": "samuraia",   // 替换为实际驱动名
    "rom": "samuraia.zip",
    "width": 224,
    "height": 288
  });
</script>
</body>
</html>
```

### 启动本地服务器

```bash
# Python 简易服务器
python3 -m http.server 8080

# 或者 nginx
```

> ⚠️ **必须通过 HTTP 服务器访问**，不能直接打开本地 HTML 文件，否则浏览器安全策略会阻止 WASM 加载。

---

## 常见问题排查

### Port 一直重新下载

检查 `.emscripten_url` 文件内容是否与 URL 完全一致（注意末尾换行）：

```bash
cat /opt/work/emsdk/upstream/emscripten/cache/ports/sdl2/.emscripten_url
```

### Hash 校验失败

```bash
# 对比 sha512
sha512sum /opt/work/emsdk/upstream/emscripten/cache/ports/sdl2.zip
grep "HASH" /opt/work/emsdk/upstream/emscripten/tools/ports/sdl2.py
```

### 查看某个 port 的期望文件名和 URL

```bash
cat /opt/work/emsdk/upstream/emscripten/tools/ports/<name>.py | head -15
```

### git 所有权警告

```
fatal: detected dubious ownership in repository at '/opt/work/mame'
```

```bash
git config --global --add safe.directory /opt/work/mame
```

---

## 附：Emscripten fetch_project 工作原理

```
emmake make
  └── fetch_project(name, url, sha512hash)
        ├── fullpath = cache/ports/<name>.<ext>     # zip 文件路径
        ├── marker  = cache/ports/<name>/.emscripten_url
        │
        ├── [检查] marker 存在 且 内容 == url ?
        │     ├── YES → 直接跳过，使用已解压的目录
        │     └── NO  → 下载 zip → 解压 → 写入 marker
        │
        └── 解压目录结构：
              cache/ports/<name>/
                ├── .emscripten_url   ← marker 文件
                └── <解压内容>/
```

**手动部署就是模拟这个流程：提前解压 + 写好 marker。**
