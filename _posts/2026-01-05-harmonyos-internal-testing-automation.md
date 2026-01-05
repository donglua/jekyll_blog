---
layout: post
title: "HarmonyOS 应用内测分发自动化实践"
date: 2026-01-05 17:30:00 +0800
categories: harmonyos python
tags: harmonyos automation internal-testing python
---

在 HarmonyOS 应用开发过程中，我们经常需要快速将测试包分发给测试人员进行验证。与 Android 的直接安装 APK 不同，HarmonyOS 采用了一套基于 **企业签名 + Manifest 清单** 的内部测试机制。本文将介绍如何构建一个自动化脚本，实现一键发布 HarmonyOS 内测包。

## 核心流程

整个内测分发流程包含以下步骤：

```mermaid
flowchart LR
    A[上传 HAP 包] --> B[计算文件哈希]
    B --> C[更新 Manifest]
    C --> D[签名 Manifest]
    D --> E[上传 Manifest]
    E --> F[生成下载页面]
    F --> G[分发给测试人员]
```

## 技术实现

### 1. 文件上传到云存储

首先需要将 `.hap` 安装包上传到可公开访问的云存储服务（如七牛云、阿里 OSS、腾讯 COS 等）。核心步骤：

1. 从后端接口获取上传凭证（Token）和 URL 前缀
2. 使用 UUID 生成唯一文件名，避免冲突
3. 通过 `multipart/form-data` 上传文件
4. 拼接返回的 key 和 URL 前缀，得到文件的公开访问地址

```python
def upload_file(file_path, token, url_prefix):
    """上传文件到云存储，返回公开访问URL"""
    unique_key = f"{uuid.uuid4().hex}{os.path.splitext(file_path)[1]}"
    with open(file_path, 'rb') as file:
        response = requests.post(upload_url, 
                                 files={'file': (unique_key, file)},
                                 data={'token': token, 'key': unique_key})
    return f"{url_prefix}/{response.json()['key']}"
```

### 2. 计算文件哈希值

HarmonyOS 内测机制要求在 Manifest 中提供安装包的 **SHA256 哈希值**，用于安装时的完整性校验：

```python
import subprocess

def calculate_sha256(file_path):
    """计算文件的SHA256哈希值"""
    try:
        result = subprocess.run(['shasum', '-a', '256', file_path], 
                               capture_output=True, text=True, check=True)
        return result.stdout.split()[0]
    except subprocess.CalledProcessError as e:
        raise Exception(f"计算哈希值失败: {e.stderr}")
```

> 在 Windows 系统上，可以使用 `certutil -hashfile <file> SHA256` 或 Python 的 `hashlib` 模块实现相同功能。

### 3. 更新 Manifest 配置

Manifest 文件是内测分发的核心配置，包含应用信息和下载地址：

```json5
{
  "app": {
    "bundleName": "com.example.app",
    "bundleType": "app",
    "versionCode": 1000000,
    "versionName": "1.0.0",
    "label": "示例应用",
    "deployDomain": "cdn.example.com",
    "icons": {
      "normal": "https://cdn.example.com/icon.png",
      "large": "https://cdn.example.com/icon_large.png"
    },
    "minAPIVersion": "5.0.1(13)",
    "targetAPIVersion": "5.1.0(18)",
    "modules": [
      {
        "name": "示例应用",
        "type": "entry",
        "deviceTypes": ["tablet", "phone"],
        "packageUrl": "https://cdn.example.com/app.hap",
        "packageHash": "sha256_hash_value"
      }
    ]
  }
}
```

更新 Manifest 的关键代码：

```python
import json

def update_manifest_file(manifest_path, package_url, package_hash, version=None):
    """更新manifest文件"""
    with open(manifest_path, 'r', encoding='utf-8') as f:
        manifest_data = json.load(f)
    
    # 可选：更新版本号
    if version:
        manifest_data['app']['versionName'] = version
        # 将版本号转换为版本代码，如 6.142.00 -> 61420200
        major, minor, patch = version.split('.')
        version_code = f"{major}{minor.zfill(2)}{patch.zfill(2)}0"
        manifest_data['app']['versionCode'] = int(version_code)
    
    # 更新下载地址和哈希值
    for module in manifest_data['app']['modules']:
        if module['type'] == 'entry':
            module['packageUrl'] = package_url
            module['packageHash'] = package_hash
    
    with open(manifest_path, 'w', encoding='utf-8') as f:
        json.dump(manifest_data, f, indent=2, ensure_ascii=False)
```

### 4. 签名 Manifest 文件

这是最关键的一步。HarmonyOS 要求 Manifest 必须使用开发者证书进行签名，华为提供了官方的签名工具：

```python
def sign_manifest_file(manifest_path, keystore_path):
    """使用华为官方工具签名manifest文件"""
    command = [
        'java', '-jar', 'manifest-sign-tool-1.0.0.jar',
        '-operation', 'sign',
        '-mode', 'localjks',
        '-inputFile', manifest_path,
        '-outputFile', manifest_path,
        '-keystore', keystore_path,
        '-keystorepasswd', 'your_keystore_password',
        '-keyaliaspasswd', 'your_alias_password',
        '-privatekey', 'your_key_alias'
    ]
    subprocess.run(command, check=True)
```

签名后的 Manifest 会包含 `sign` 字段，格式如下：

```json
{
  "app": { ... },
  "sign": "MEYCIQDC+JmpxzuKrNlH1vu...（Base64编码的签名）"
}
```

> **注意**：签名工具需要依赖以下 JAR 包：
> - `bcprov-jdk18on-1.75.jar` (Bouncy Castle 加密库)
> - `commons-codec-1.15.jar`
> - `gson-2.9.1.jar`
> - `log4j-api-2.23.1.jar` / `log4j-core-2.23.1.jar`

### 5. 生成下载页面

为了方便测试人员安装，我们生成一个包含 **DeepLink** 和 **二维码** 的 HTML 下载页面：

```python
import qrcode
import io
import base64
from datetime import datetime

def generate_download_page(manifest_url, output_path, version=None, app_icon_url=None):
    """生成包含下载按钮和二维码的HTML页面"""
    deep_link = f'store://enterprise/manifest?url={manifest_url}'
    html_url = f'https://your-server.com/downloads/{os.path.basename(output_path)}'
    
    # 生成二维码
    qr = qrcode.QRCode(version=1, box_size=10, border=4)
    qr.add_data(html_url)
    qr.make(fit=True)
    qr_img = qr.make_image(fill_color="black", back_color="white")
    
    # 转换为 Base64 内嵌到 HTML
    buffered = io.BytesIO()
    qr_img.save(buffered, format="PNG")
    qr_base64 = base64.b64encode(buffered.getvalue()).decode('utf-8')
    qr_data_url = f'data:image/png;base64,{qr_base64}'
    
    # 生成 HTML（省略样式代码）
    html_content = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <title>下载应用 v{version}</title>
    </head>
    <body>
        <h1>应用名称</h1>
        <p>版本: {version}</p>
        <button onclick="window.open('{deep_link}', '_parent')">立即安装</button>
        <img src="{qr_data_url}" alt="扫码下载">
    </body>
    </html>
    """
    
    with open(output_path, 'w', encoding='utf-8') as f:
        f.write(html_content)
```

**DeepLink 说明**：

`store://enterprise/manifest?url=<manifest_url>` 是 HarmonyOS 的内测安装协议，可以在华为应用市场 App 中解析并触发安装流程。

## 完整工作流

将上述步骤整合，形成一键发布脚本：

```python
def main():
    parser = argparse.ArgumentParser(description='发布HarmonyOS内测包')
    parser.add_argument('hap_file', type=str, help='.hap文件路径')
    parser.add_argument('-v', '--version', type=str, help='版本号 (如: 1.0.0)')
    args = parser.parse_args()

    # 步骤1: 上传 HAP 文件
    print("步骤1: 上传.hap文件...")
    token, url_prefix = get_upload_token()
    hap_url = upload_file(args.hap_file, token, url_prefix)
    
    # 步骤2: 计算哈希值
    print("步骤2: 计算SHA256哈希值...")
    hash_value = calculate_sha256(args.hap_file)
    
    # 步骤3: 更新 Manifest
    print("步骤3: 更新manifest配置...")
    update_manifest_file("manifest_sign.json5", hap_url, hash_value, args.version)
    
    # 步骤4: 签名 Manifest
    print("步骤4: 签名manifest文件...")
    sign_manifest_file("manifest_sign.json5", "your_keystore.p12")
    
    # 步骤5: 上传 Manifest
    print("步骤5: 上传manifest文件...")
    manifest_url = upload_file("manifest_sign.json5", token, url_prefix)
    
    # 步骤6: 生成下载页面
    print("步骤6: 生成下载页面...")
    html_filename = f"app_v{args.version}_{datetime.now().strftime('%m%d_%H%M')}.html"
    generate_download_page(manifest_url, html_filename, args.version)
    
    print(f"\n✅ 发布完成！下载链接: https://your-server.com/downloads/{html_filename}")

if __name__ == "__main__":
    main()
```

## 使用方式

```bash
# 安装依赖
pip3 install requests pillow qrcode

# 发布内测包
python3 upload_version.py app-release.hap -v 1.2.0
```

执行后输出：
```
步骤1: 上传.hap文件...
.hap文件上传成功，URL: https://cdn.xxx.com/abc123.hap

步骤2: 计算SHA256哈希值...
哈希值: 9734d7dac55a4a8aa23a241f9de289773eafa27e...

步骤3: 更新manifest配置...
manifest文件更新成功

步骤4: 签名manifest文件...
manifest文件签名成功

步骤5: 上传manifest文件...
manifest文件上传成功

步骤6: 生成下载页面...
HTML文件生成成功

✅ 发布完成！下载链接: https://your-server.com/downloads/app_v1.2.0_0105_1730.html
```

## 下载页面效果

生成的下载页面支持：

- 📱 **一键安装**：点击按钮通过 DeepLink 唤起应用市场安装
- 📷 **扫码下载**：其他测试人员扫码访问下载页
- 🌙 **暗黑模式**：自动适配系统深色主题
- 📐 **响应式布局**：适配手机和平板等不同屏幕

## 安全注意事项

1. **密钥保护**：证书文件 (`.p12`) 和密码不应提交到代码仓库，建议使用环境变量或密钥管理服务
2. **内网分发**：下载页面和 Manifest 文件建议部署在内网服务器，避免公网泄露

## 总结

通过这套自动化脚本，我们将 HarmonyOS 内测包的发布流程从手动操作简化为一条命令，大大提高了开发效率。核心技术点包括：

- 云存储文件上传
- SHA256 文件完整性校验
- Manifest 配置与企业签名
- DeepLink 协议触发安装
- QR Code 二维码生成

希望本文对正在开发 HarmonyOS 应用的团队有所帮助！

---

**参考资料**：
- [HarmonyOS 应用内部测试官方文档](https://developer.huawei.com/consumer/cn/doc/app/agc-help-harmonyos-internaltest-0000001937800101)
- [内部测试验签工具](https://gitee.com/arkin-internal-testing/internal-testing)
