---
layout: post
title: "鸿蒙 Next 实战：集成阿里云 HTTPDNS 优化网络请求"
date: 2025-12-31 10:00:00 +0800
categories: harmonyos network
tags: harmonyos httpdns network optimization
---

在移动应用开发中，域名劫持和解析延迟是常见的网络问题。为了提升网络连接的稳定性和速度，引入 HTTPDNS 是一个非常有效的方案。本文将分享如何在鸿蒙 HarmonyOS Next 项目中集成阿里云 HTTPDNS（`@aliyun/httpdns`）并结合 Axios 实现自定义 DNS 解析。

## 1. 引入依赖

首先，我们需要在项目的 `oh-package.json5` 中引入阿里云 HTTPDNS 的 SDK 和 Axios。

```json
"dependencies": {
  "@aliyun/httpdns": "^1.0.1",
  "@ohos/axios": "^2.2.0"
}
```

执行 `ohpm install` 安装依赖。

## 2. 封装 HttpDnsManager

为了方便管理和调用，我们封装一个单例类 `HttpDnsManager`。这个类主要负责 SDK 的初始化和提供 DNS 解析方法。

创建 `HttpDnsManager.ets`：

```typescript
import { httpdns, IHttpDnsService, HttpDnsResult } from '@aliyun/httpdns';
import { connection } from '@kit.NetworkKit';
import common from '@ohos.app.ability.common';

export class HttpDnsManager {
  private static instance: HttpDnsManager;
  private httpDnsService: IHttpDnsService | undefined;
  
  // 替换为你自己的阿里云 Account ID
  private static readonly ALIYUN_ACCOUNT_ID = "105310";

  private constructor() {
  }

  public static getInstance(): HttpDnsManager {
    if (!HttpDnsManager.instance) {
      HttpDnsManager.instance = new HttpDnsManager();
    }
    return HttpDnsManager.instance;
  }

  /**
   * 初始化 HTTPDNS 服务
   * @param context UIAbilityContext
   * @param accountId 阿里云 Account ID
   */
  public init(context: common.UIAbilityContext, accountId: string = HttpDnsManager.ALIYUN_ACCOUNT_ID) {
    // 1. 配置服务上下文
    httpdns.configService(accountId, {
      context: context
    });
    
    // 2. 异步获取服务实例
    httpdns.getService(accountId).then((service) => {
      this.httpDnsService = service;
    }).catch((err: Error) => {
      console.error("HttpDns init failed: " + JSON.stringify(err));
    });
  }

  /**
   * 自定义 DNS 解析方法 (供 Axios 调用)
   */
  public async lookup(hostname: string): Promise<Array<connection.NetAddress>> {
    if (!this.httpDnsService) {
      // 服务未初始化，返回空让 Axios 走默认系统 DNS
      return []; 
    }
    
    try {
      // 异步获取解析结果，内部包含了缓存策略：有缓存用缓存，无缓存请求网络
      let result: HttpDnsResult = await this.httpDnsService.getHttpDnsResultAsync(hostname);
      
      let netAddresses: Array<connection.NetAddress> = [];
      
      // 处理 IPv4 结果
      if (result.ipv4s && result.ipv4s.length > 0) {
        for (let ip of result.ipv4s) {
          netAddresses.push({
            address: ip,
            family: 1, // 1 代表 IPv4 (AF_INET)
            port: 0
          });
        }
      }
      
      // 处理 IPv6 结果
      if (result.ipv6s && result.ipv6s.length > 0) {
        for (let ip of result.ipv6s) {
          netAddresses.push({
            address: ip,
            family: 2, // 2 代表 IPv6 (AF_INET6)
            port: 0
          });
        }
      }
      
      return netAddresses;
    } catch (e) {
      console.error("HttpDns lookup failed: " + JSON.stringify(e));
      return [];
    }
  }
}
```

## 3. 在 Axios 中应用

Axios 自 `2.2.0` 版本起支持自定义 `dns.lookup` 配置。我们可以将 `HttpDnsManager` 的 `lookup` 方法注入到 Axios 中。

### 全局配置

如果你希望项目中所有的 Axios 请求都走 HTTPDNS：

```typescript
import axios from '@ohos/axios';
import { HttpDnsManager } from './HttpDnsManager'; // 假设路径

// 配置 Axios 默认的 DNS 解析
axios.defaults.dns = {
  lookup: async (hostname: string) => {
    // 获取 HTTPDNS 解析结果
    return await HttpDnsManager.getInstance().lookup(hostname);
  }
};
```

### 实例配置

如果你只想为特定的请求实例开启 HTTPDNS：

```typescript
import axios from '@ohos/axios';
import { HttpDnsManager } from './HttpDnsManager';

const request = axios.create({
  baseURL: 'https://api.yourdomain.com',
  timeout: 10000,
  dns: {
    lookup: async (hostname: string) => {
      return await HttpDnsManager.getInstance().lookup(hostname);
    }
  }
});
```

## 4. 全局初始化

最后，别忘了在应用的入口 `EntryAbility` 中初始化 `HttpDnsManager`：

```typescript
import { HttpDnsManager } from '@jzdy/common'; // 你的模块路径

export default class EntryAbility extends UIAbility {
  onCreate(want, launchParam) {
    // ...
    // 初始化 HTTPDNS
    HttpDnsManager.getInstance().init(this.context);
    // ...
  }
}
```

## 总结

通过以上方式，我们成功将阿里云 HTTPDNS 集成到了鸿蒙应用的 Axios 网络栈中。
1.  **HttpDnsManager**：负责与阿里云 SDK 交互，提供统一的解析接口。
2.  **Axios DNS Config**：利用 `config.dns.lookup` 钩子，拦截 DNS 解析过程，替换为 HTTPDNS 的结果。

这种方案侵入性小，且充分利用了 Axios 的扩展能力，是鸿蒙网络优化的最佳实践之一。

## 5. 参考资料

*   [阿里云 HTTPDNS 产品文档](https://help.aliyun.com/product/37963.html) - 获取 Account ID 及控制台配置
*   [OpenHarmony Axios 组件](https://ohpm.openharmony.cn/#/cn/detail/@ohos%2Faxios) - Axios for HarmonyOS
*   [阿里云 HTTPDNS HarmonyOS SDK (OHPM)](https://ohpm.openharmony.cn/#/cn/detail/@aliyun%2Fhttpdns) - SDK 下载及更新日志
*   [自定义 DNS 解析配置参考](https://gitee.com/openharmony-sig/ohos_axios#%E8%87%AA%E5%AE%9A%E4%B9%89dns) - Axios 自定义 DNS 文档
