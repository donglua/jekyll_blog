---
layout: post
title: "HarmonyOS 文件预览工具类：支持远程文件下载与本地预览"
date: 2026-01-08 19:50:00 +0800
categories: [HarmonyOS, ArkTS]
tags: [harmonyos, arkts, filepreview, file-download]
---

在 HarmonyOS 应用开发中，`filePreview.openPreview` 是系统提供的文件预览 API，但它**仅支持本地文件**。当我们需要预览网络上的文件（如 PDF、PPT、Word 文档）时，需要先将文件下载到本地，再调用预览接口。

本文将介绍如何封装一个 `FilePreviewHelper` 工具类，实现：
- 自动判断本地/远程文件
- 远程文件自动下载并缓存
- 正确处理文件 URI 权限
- 自动识别 MIME 类型

## 遇到的问题与解决方案

### 问题 1：filePreview.openPreview 不支持网络 URL

**现象**：直接传入 `https://` 开头的 URL，预览失败。

**解决方案**：检测 URL 类型，如果是网络 URL，先下载到应用沙箱目录。

### 问题 2：文件 URI 权限问题

**现象**：下载完成后，使用 `file://` + 路径的方式构造 URI，预览仍然失败。

**原因**：`filePreview` 在新窗口中打开文件，应用的临时权限无法传递给预览窗口。

**解决方案**：使用 `fileUri.getUriFromPath()` 将沙箱路径转换为正确的 URI 格式：

```typescript
import { fileUri } from '@kit.CoreFileKit';

// ❌ 错误方式
const uri = `file://${targetPath}`;

// ✅ 正确方式
const uri = fileUri.getUriFromPath(targetPath);
// 生成格式：file://bundleName/path/to/file
```

### 问题 3：缓存文件可能不完整

**现象**：之前下载中断，文件大小为 0 或很小，但仍被当作有效缓存使用。

**解决方案**：检查文件大小，如果过小则删除重新下载：

```typescript
const stat = fs.statSync(targetPath);
if (stat.size > 1024) {
  // 有效文件，直接使用
  return fileUri.getUriFromPath(targetPath);
} else {
  // 文件太小，删除重新下载
  fs.unlinkSync(targetPath);
}
```

### 问题 4：MIME 类型处理

**说明**：`PreviewInfo` 的 `mimeType` 字段用于指定文件的媒体资源类型。

> **官方文档**：若无法确定文件格式，该项可直接赋值空字符串（`""`），系统会通过 URI 后缀进行文件格式判断。

**两种处理方式**：

```typescript
// 方式 1：让系统自动判断（推荐）
const fileInfo: filePreview.PreviewInfo = {
  uri: filePath,
  title: fileName,
  mimeType: ""  // 空字符串，系统自动通过后缀判断
};

// 方式 2：手动指定（可选）
const MIME_TYPES: Record<string, string> = {
  'pdf': 'application/pdf',
  'pptx': 'application/vnd.openxmlformats-officedocument.presentationml.presentation',
  // ...
};
```

## 完整代码实现

```typescript
import { filePreview } from '@kit.PreviewKit';
import { common } from '@kit.AbilityKit';
import { fileIo as fs, fileUri } from '@kit.CoreFileKit';
import axios, { AxiosResponse } from '@ohos/axios';

export class FilePreviewHelper {

  /**
   * 打开文件预览。如果是远程 URL，会先下载到本地。
   */
  static async openPreview(
    context: common.UIAbilityContext, 
    url: string, 
    title?: string
  ): Promise<void> {
    
    let filePath = url;
    let fileName = title || "文件预览";

    // 检查是否为远程 URL
    if (url.startsWith("http")) {
      filePath = await FilePreviewHelper.downloadFile(context, url);
    }

    // 构造 PreviewInfo
    const fileInfo: filePreview.PreviewInfo = {
      uri: filePath,
      title: fileName,
      mimeType: ""  // 让系统自动判断
    };

    // 调用系统预览
    filePreview.openPreview(context, fileInfo);
  }

  /**
   * 下载文件到缓存目录
   */
  private static async downloadFile(
    context: common.UIAbilityContext, 
    url: string
  ): Promise<string> {
    
    const cacheDir = context.cacheDir;
    const fileName = FilePreviewHelper.getFileNameFromUrl(url);
    const targetPath = `${cacheDir}/${fileName}`;

    // 检查文件是否已存在且有效
    try {
      const stat = fs.statSync(targetPath);
      if (stat.size > 1024) {
        return fileUri.getUriFromPath(targetPath);
      } else {
        fs.unlinkSync(targetPath);
      }
    } catch (e) {
      // 文件不存在
    }

    // 下载文件
    const response: AxiosResponse<ArrayBuffer> = await axios.get<ArrayBuffer>(url, {
      responseType: 'array_buffer'
    });

    // 写入文件
    const file = fs.openSync(targetPath, 
      fs.OpenMode.READ_WRITE | fs.OpenMode.CREATE | fs.OpenMode.TRUNC);
    fs.writeSync(file.fd, response.data);
    fs.closeSync(file);

    // 返回正确格式的 URI
    return fileUri.getUriFromPath(targetPath);
  }

  /**
   * 从 URL 中提取文件名
   */
  private static getFileNameFromUrl(url: string): string {
    const split = url.split('/');
    let fileName = split[split.length - 1] || "temp_file";

    // 去除查询参数
    const queryIndex = fileName.indexOf('?');
    if (queryIndex !== -1) {
      fileName = fileName.substring(0, queryIndex);
    }

    // 解码 & 清理文件名
    try {
      fileName = decodeURIComponent(fileName);
    } catch (e) {}

    const lastDot = fileName.lastIndexOf('.');
    let baseName = lastDot > 0 ? fileName.substring(0, lastDot) : fileName;
    let ext = lastDot > 0 ? fileName.substring(lastDot) : '';

    baseName = baseName.replace(/[^a-zA-Z0-9_-]/g, "_");
    if (baseName.trim().length === 0) {
      baseName = `download_${Date.now()}`;
    }

    return baseName + ext;
  }
}
```

## 使用示例

```typescript
// 预览网络 PPT 文件
FilePreviewHelper.openPreview(
  context, 
  'https://example.com/presentation.pptx', 
  '演示文稿'
);

// 预览本地文件
FilePreviewHelper.openPreview(
  context, 
  'file://bundleName/data/storage/.../document.pdf', 
  'PDF文档'
);
```

## 注意事项

1. **Office 文档预览**：HarmonyOS 预览 Office 文档（如 PPTX、DOCX）依赖系统中安装的 WPS 应用
2. **文件权限**：使用 `fileUri.getUriFromPath()` 确保权限能正确传递给预览窗口
3. **缓存策略**：文件下载到 `context.cacheDir`，系统在存储空间不足时会自动清理
4. **备选方案**：如果本地预览不可用，可考虑使用微软在线预览：
   ```typescript
   const onlineUrl = `https://view.officeapps.live.com/op/view.aspx?src=${encodeURIComponent(url)}`;
   ```

## 总结

通过封装 `FilePreviewHelper`，我们解决了 HarmonyOS 中预览网络文件的痛点。关键点包括：

- 使用 `axios` 下载文件到应用沙箱
- 使用 `fileUri.getUriFromPath()` 生成正确的 URI
- 系统自动识别 MIME 类型
- 实现文件缓存机制

希望这篇文章对你有所帮助！

## 参考资料

- [openPreview - PreviewKit API 参考](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/preview-arkts#section144826162913)
- [fileUri.getUriFromPath - CoreFileKit API](https://developer.huawei.com/consumer/cn/doc/harmonyos-references-V5/js-apis-file-fileuri-V5#fileurigeturifrompath)
- [应用沙箱路径详解 - HarmonyOS 开发指南](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides-V5/app-sandbox-directory-V5)
- [@ohos/axios - OpenHarmony 三方库](https://ohpm.openharmony.cn/#/cn/detail/@ohos%2Faxios)
