# BI_M5_QwenSoftPrefix

LLM推論フレームワーク [StackFlow](https://github.com/DakeQQ/StackFlow) をベースとした、Qwen2.5モデルのSoft Prefix実装です。

## 目次

- [環境要件](#環境要件)
- [セットアップ](#セットアップ)
- [ビルド](#ビルド)
- [トラブルシューティング](#トラブルシューティング)

## 環境要件

- Python 3.10+
- GCC (aarch64対応)
- 十分なメモリ容量（ビルド時に大量のメモリを使用）

## セットアップ

### 依存関係のインストール

```bash
pip install parse scons
```

### サブモジュールの初期化

```bash
git submodule update --init --recursive
```

## ビルド

メモリ不足を回避するため、並列ビルドを無効化（`-j1`）することを推奨します。

```bash
scons -j1
```

## トラブルシューティング

### エラー1: `KeyError: 'GCC_DUMPMACHINE'`

以下のようなエラーが発生した場合:
```
KeyError: 'GCC_DUMPMACHINE':
  File ".../StackFlow/SDK/components/simdjson_component/SConstruct", line 24:
    gcc_dumpmachine = env["GCC_DUMPMACHINE"].split("-")
```

**解決方法**: 以下のパッチスクリプトを実行して、`GCC_DUMPMACHINE`環境変数を自動設定します。

```bash
python3 - <<'PY'
from pathlib import Path
import re

# Adjust path to your actual StackFlow location
p = Path("StackFlow/SDK/components/simdjson_component/SConstruct")
txt = p.read_text()

needle = 'gcc_dumpmachine = env["GCC_DUMPMACHINE"].split("-")'
marker = "ensure GCC_DUMPMACHINE exists even if SCons GCC tool could not populate it"

if marker in txt:
    print("Already patched.")
    raise SystemExit(0)

lines = txt.splitlines(True)

# Find the gcc_dumpmachine assignment line and capture its indentation
idx = None
indent = ""
for i, line in enumerate(lines):
    if line.lstrip().startswith(needle):
        idx = i
        indent = re.match(r'^(\s*)', line).group(1)
        break
if idx is None:
    raise SystemExit("Pattern not found: " + needle)

patch_rel = [
    f"# {marker}",
    "import subprocess",
    'if "GCC_DUMPMACHINE" not in env:',
    "    try:",
    '        cc = env.get("CC", "gcc")',
    '        dm = subprocess.check_output([cc, "-dumpmachine"]).decode().strip()',
    "    except Exception:",
    '        dm = "aarch64-linux-gnu"',
    '    env["GCC_DUMPMACHINE"] = dm',
    "",
]

patch_text = "".join(((indent + s) if s else indent) + "\n" for s in patch_rel)
lines.insert(idx, patch_text)
p.write_text("".join(lines))
print("Patched:", p)
PY
```

### エラー2: クロスコンパイラが見つからない

以下のようなエラーが発生した場合:

```
sh: /opt/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-g++: not found
scons: *** [build/Backward_cpp/empty_src_file.cpp.o] Error 127
```

**解決方法**: ネイティブコンパイラへのラッパースクリプトを作成します。

```bash
set -e

TC=/opt/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin
PREFIX=aarch64-none-linux-gnu

# Verify native compilers exist on the device
command -v gcc >/dev/null
command -v g++ >/dev/null

mkdir -p "$TC"
cd "$TC"

cat > ${PREFIX}-gcc << "EOF"
#!/bin/sh
exec gcc "$@"
EOF

cat > ${PREFIX}-g++ << "EOF"
#!/bin/sh
exec g++ "$@"
EOF

chmod +x ${PREFIX}-gcc ${PREFIX}-g++

# Create symlinks for other build tools
for t in ar ranlib strip nm ld objcopy objdump; do
  if command -v "$t" >/dev/null 2>&1; then
    ln -sf "$(command -v "$t")" "${PREFIX}-${t}"
  fi
done

echo "Wrappers installed in $TC"
```

**動作確認**:

```bash
/opt/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-gcc --version
/opt/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-g++ --version
```

### エラー3: 位置独立コード（PIC）エラー

以下のようなリンクエラーが発生した場合:

```
/bin/ld: relocation R_AARCH64_ADR_PREL_PG_HI21 ... can not be used when making a shared object; recompile with -fPIC
scons: *** [build/llm_kws/llm_kws] Error 1
```

**原因**: 静的ライブラリが `-fPIC` フラグなしでコンパイルされています。

**解決方法**: ビルド設定を確認し、必要に応じて依存ライブラリを `-fPIC` フラグ付きで再ビルドしてください。