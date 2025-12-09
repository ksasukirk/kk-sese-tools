# V3加密文件读取示例

本文档介绍如何使用示例脚本读取和解析V3格式的加密图像文件。

## 📁 示例脚本列表

### 1. `example_read_v3_encrypted.py` - 完整读取示例

完整读取V3加密文件的所有内容，包括元数据、缩略图和原始图像。

**功能特点：**
- ✅ 读取并解密JSON元数据
- ✅ 读取并解密WebP缩略图
- ✅ 读取并解密原始图像
- ✅ 显示详细的文件结构分析
- ✅ 自动保存所有解密内容到文件

**使用方法：**
```bash
# 基本用法
python example_read_v3_encrypted.py <加密文件> <密钥文件>

# 指定输出目录
python example_read_v3_encrypted.py <加密文件> <密钥文件> <输出目录>
```

**示例：**
```bash
python example_read_v3_encrypted.py test.klock klock_secret.key
python example_read_v3_encrypted.py image.klock klock_secret.key output_folder
```

**输出文件：**
- `<文件名>_metadata.json` - JSON元数据
- `<文件名>_thumbnail.webp` - 缩略图
- `<文件名>_original.<ext>` - 原始图像（保持原格式）

---

### 2. `example_quick_read_v3.py` - 快速读取示例

快速读取元数据和缩略图，**不解密完整图像**，适合快速预览和批量扫描。

**功能特点：**
- ✅ 快速读取元数据（无需解密完整图像）
- ✅ 快速读取缩略图
- ✅ 节省内存和时间
- ✅ 支持批量读取

**使用方法：**
```bash
python example_quick_read_v3.py <加密文件> <密钥文件>
```

**示例：**
```bash
python example_quick_read_v3.py test.klock klock_secret.key
```

**输出：**
- 在控制台显示元数据信息
- 保存缩略图预览为 `thumbnail_preview.webp`

---

## 📊 V3文件格式说明

```
┌─────────────────────────────────────────────────────────┐
│ Header (19 bytes)                                       │
│ "KK_ENCRYPTED_IMG_V3"                                  │
├─────────────────────────────────────────────────────────┤
│ Meta Length (4 bytes)                                   │
│ Unsigned Int, Big-Endian                               │
├─────────────────────────────────────────────────────────┤
│ Encrypted Metadata (variable)                          │
│ AES-256 encrypted JSON                                 │
│ {width, height, phash, format, version}                │
├─────────────────────────────────────────────────────────┤
│ Thumbnail Length (4 bytes)                             │
│ Unsigned Int, Big-Endian                               │
├─────────────────────────────────────────────────────────┤
│ Encrypted Thumbnail (variable)                         │
│ AES-256 encrypted WebP image                           │
├─────────────────────────────────────────────────────────┤
│ Encrypted Image (remaining)                            │
│ AES-256 encrypted original image                       │
└─────────────────────────────────────────────────────────┘
```

## 🔧 代码示例

### 仅读取元数据

```python
import struct
import json
from cryptography.fernet import Fernet

def read_metadata_only(file_path, key_file):
    """只读取元数据，不解密图像"""
    # 加载密钥
    with open(key_file, 'rb') as f:
        cipher = Fernet(f.read())
    
    # 读取文件
    with open(file_path, 'rb') as f:
        f.read(19)  # 跳过文件头
        meta_len = struct.unpack('>I', f.read(4))[0]
        encrypted_meta = f.read(meta_len)
        
        # 解密元数据
        metadata = json.loads(cipher.decrypt(encrypted_meta))
        return metadata

# 使用
metadata = read_metadata_only('image.klock', 'klock_secret.key')
print(f"图像尺寸: {metadata['width']}x{metadata['height']}")
print(f"pHash: {metadata['phash']}")
```

### 仅读取缩略图

```python
import struct
from cryptography.fernet import Fernet
from PIL import Image
import io

def read_thumbnail_only(file_path, key_file):
    """只读取缩略图，不解密完整图像"""
    # 加载密钥
    with open(key_file, 'rb') as f:
        cipher = Fernet(f.read())
    
    # 读取文件
    with open(file_path, 'rb') as f:
        f.read(19)  # 跳过文件头
        
        # 跳过元数据
        meta_len = struct.unpack('>I', f.read(4))[0]
        f.seek(meta_len, 1)
        
        # 读取缩略图
        thumb_len = struct.unpack('>I', f.read(4))[0]
        if thumb_len == 0:
            return None
            
        encrypted_thumb = f.read(thumb_len)
        thumb_data = cipher.decrypt(encrypted_thumb)
        return Image.open(io.BytesIO(thumb_data))

# 使用
thumbnail = read_thumbnail_only('image.klock', 'klock_secret.key')
thumbnail.show()  # 显示缩略图
```

### 批量读取目录中的元数据

```python
import os

def scan_directory_metadata(directory, key_file):
    """扫描目录中所有V3文件的元数据"""
    results = []
    
    for root, _, files in os.walk(directory):
        for file in files:
            if file.endswith('.klock'):
                file_path = os.path.join(root, file)
                try:
                    metadata = read_metadata_only(file_path, key_file)
                    results.append({
                        'path': file_path,
                        'width': metadata['width'],
                        'height': metadata['height'],
                        'format': metadata['format'],
                        'phash': metadata['phash']
                    })
                except Exception as e:
                    print(f"跳过 {file}: {e}")
    
    return results

# 使用
files_info = scan_directory_metadata('D:/Images', 'klock_secret.key')
for info in files_info:
    print(f"{info['path']}: {info['width']}x{info['height']} ({info['format']})")
```

## 🔐 安全注意事项

1. **密钥文件保护**
   - 密钥文件包含敏感信息，请妥善保管
   - 不要将密钥文件提交到版本控制系统
   - 建议设置文件只读权限

2. **错误处理**
   - 所有示例都包含基本的错误处理
   - 生产环境中应添加更完善的异常处理
   - 验证文件完整性后再进行操作

3. **性能考虑**
   - 完整解密大图像会消耗较多内存
   - 批量处理时建议使用快速读取模式
   - 缩略图适合用于预览和索引

## 📚 依赖库

```bash
pip install cryptography pillow
```

## 🆘 常见问题

### Q: 提示"不是V3格式文件"？
A: 该文件可能是V1或V2格式。V3格式文件头为 `KK_ENCRYPTED_IMG_V3`。

### Q: 密钥错误或文件损坏？
A: 确保使用正确的密钥文件，且加密文件未被修改。

### Q: 如何转换V1/V2文件到V3？
A: 使用主程序先解密，然后重新加密即可自动升级到V3格式。

### Q: 缩略图为空？
A: 某些旧版本加密的文件可能不包含缩略图，这是正常的。

## 🔗 相关工具

- **主程序**: `main.py` - 完整的加密/解密工具
- **查看器**: 内置加密图像查看器
- **pHash工具**: `kklibs/kklib_phash.py` - 图像哈希比对

## 📞 技术支持

如有问题或建议，请查看项目README或联系开发者。

