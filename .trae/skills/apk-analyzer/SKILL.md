---
name: "apk-analyzer"
description: "Analyzes APK files using mt_mcp service and generates analysis reports. Invoke when user wants to analyze APK for payment features, sensitive strings, class structures, resources, or any APK analysis task."
---

# APK Analyzer

APK 分析技能，使用 mt_mcp 服务对 APK 文件进行深度分析并生成结构化报告。

## 核心原则

**本技能不预设固定分析内容，而是根据用户的具体分析需求动态执行分析。**

用户可以指定：
- APK 文件（支持模糊名称匹配）
- 分析关键词（单个或多个）
- 搜索范围（dex_string、dex_class、dex_method、smali、axml、resource_table_* 等）
- 分析深度（简单搜索、详细分析、完整报告）
- 特定包名或类名过滤

## 工作流程

### 步骤1: 选择 APK 文件

**支持模糊名称匹配，无需精确输入完整文件名。**

#### 匹配规则

| 用户输入 | 匹配方式 | 说明 |
|---------|---------|------|
| `mt://current-apk` | 当前APK | 直接使用当前打开的 APK |
| `微信` | 模糊匹配 | 匹配包含"微信"的 APK 文件名或包名 |
| `com.tencent.mm` | 包名匹配 | 匹配包名包含该字符串的 APK |
| `a.apk` | 精确匹配 | 直接使用指定文件 |
| `kwyy` | 模糊匹配 | 匹配文件名或包名包含该字符串的 APK |

#### 智能选择流程

1. 用户输入 APK 名称（可以是部分名称）
2. 调用 `mt_apk_open` 尝试打开
3. 如果当前 APK 不可用，服务会返回可用 APK 列表
4. 从列表中模糊匹配用户输入
5. 找到匹配后自动打开对应 APK

#### 示例

```
用户: "分析 微信 的支付功能"
→ 自动搜索匹配 "微信" 的 APK
→ 找到 "com.tencent.mm.apk" 或包含 "wechat" 的 APK
→ 打开并分析

用户: "分析 kwyy 的敏感字符串"
→ 自动搜索匹配 "kwyy" 的 APK
→ 找到 "kwyy.apk"
→ 打开并分析
```

#### 实现方式

```json
// 1. 先尝试打开当前 APK 或指定 APK
mt_apk_open({ "path": "mt://current-apk" })

// 2. 如果返回 CURRENT_APK_NOT_AVAILABLE 错误
// 错误响应中包含 availableApkFiles 列表
{
  "errorCode": "CURRENT_APK_NOT_AVAILABLE",
  "availableApkFiles": ["a.apk", "kwyy.apk", "wechat.apk"],
  "availableApkFileCount": 3
}

// 3. 根据用户输入模糊匹配列表中的文件
// 4. 打开匹配的 APK
mt_apk_open({ "path": "匹配到的文件名.apk" })
```

### 步骤2: 理解分析需求

根据用户描述提取分析意图：

| 用户描述示例 | 分析类型 | 推荐搜索范围 |
|-------------|---------|-------------|
| "分析支付功能" | 支付相关 | dex_string, dex_class, dex_method |
| "查找敏感字符串" | 安全相关 | dex_string, smali |
| "分析 XXX 类" | 类结构 | dex_class_outline |
| "查找网络请求" | 网络相关 | dex_string, smali |
| "分析权限使用" | 权限相关 | axml (AndroidManifest) |
| "查找资源引用" | 资源相关 | resource_table_* |
| "分析 XXX 包" | 包结构 | dex_classes |
| "搜索关键词 YYY" | 自定义 | 用户指定 |

### 步骤3: 执行分析

根据需求选择合适的分析方法：

#### 字符串搜索

```json
mt_apk_search({
  "workspaceId": "xxx",
  "query": "关键词",
  "queryType": "literal|regex",
  "caseSensitive": false,
  "scopes": ["dex_string", "smali"],
  "classPrefix": "可选-类前缀过滤"
})
```

#### 类结构分析

```json
mt_apk_list({
  "workspaceId": "xxx",
  "view": "dex_class_outline",
  "className": "Lcom/example/MainActivity;"
})
```

#### 类列表

```json
mt_apk_list({
  "workspaceId": "xxx",
  "view": "dex_classes",
  "prefix": "com/example"
})
```

#### 读取代码

```json
mt_apk_read({
  "workspaceId": "xxx",
  "locator": {
    "kind": "dex_method",
    "className": "Lcom/example/MainActivity;",
    "methodSig": "方法签名"
  }
})
```

#### 读取 AndroidManifest

```json
mt_apk_read({
  "workspaceId": "xxx",
  "locator": { "kind": "axml", "path": "AndroidManifest.xml" }
})
```

#### 资源搜索

```json
mt_apk_search({
  "workspaceId": "xxx",
  "query": "资源名或ID",
  "queryType": "literal",
  "caseSensitive": false,
  "scopes": ["resource_table_name", "resource_table_id", "resource_table_value"]
})
```

### 步骤4: 生成报告

报告文件命名：`{apkName}_{分析类型}报告.md`

## APK 智能匹配详解

### 匹配优先级

1. **精确匹配** - 用户输入与 APK 文件名完全一致
2. **包名匹配** - 用户输入匹配 APK 的包名
3. **应用名匹配** - 用户输入匹配应用显示名称
4. **模糊匹配** - 用户输入是文件名或包名的子串

### 匹配示例

| 可用 APK 列表 | 用户输入 | 匹配结果 |
|--------------|---------|---------|
| a.apk, kwyy.apk, wechat.apk | `kwyy` | kwyy.apk |
| a.apk, kwyy.apk, wechat.apk | `微信` | wechat.apk (如果有对应映射) |
| a.apk, kwyy.apk, wechat.apk | `a` | a.apk |
| com.tencent.mm.apk, com.taobao.app.apk | `tencent` | com.tencent.mm.apk |
| com.tencent.mm.apk, com.taobao.app.apk | `taobao` | com.taobao.app.apk |

### 常见应用名称映射

| 用户可能输入 | 可能匹配的 APK/包名 |
|-------------|-------------------|
| 微信 | wechat, com.tencent.mm |
| 支付宝 | alipay, com.eg.android.AlipayGphone |
| 淘宝 | taobao, com.taobao.taobao |
| 抖音 | douyin, com.ss.android.ugc.aweme |
| 快手 | kuaishou, com.smile.gifmaker |
| QQ | qq, com.tencent.mobileqq |
| 百度 | baidu, com.baidu.searchbox |
| 京东 | jd, com.jingdong.app.mall |

### 多匹配处理

如果用户输入匹配到多个 APK：

```
用户输入: "com.tencent"
匹配结果: 
  - com.tencent.mm.apk (微信)
  - com.tencent.mobileqq.apk (QQ)
  - com.tencent.qqlive.apk (腾讯视频)

处理方式: 列出匹配结果，让用户选择具体 APK
```

## 分析场景参考

以下是常见的分析场景，用户可参考或自定义：

### 功能分析类

| 场景 | 推荐关键词 | 推荐范围 |
|------|-----------|---------|
| 支付功能 | payment, pay, purchase, order, checkout | dex_string, dex_class |
| 会员/VIP | vip, premium, member, subscribe, subscription | dex_string, dex_class |
| 登录认证 | login, auth, token, session, credential | dex_string, smali |
| 社交分享 | share, social, wechat, weibo, facebook | dex_string |
| 推送通知 | push, notification, fcm, jpush, xiaomi | dex_string |
| 广告SDK | ad, ads, banner, interstitial, reward | dex_string, dex_class |
| 地图定位 | map, location, gps, baidu, amap | dex_string |
| 网络请求 | http, request, api, retrofit, okhttp | dex_string, smali |
| 数据存储 | database, sqlite, room, sharedpreferences | dex_string, dex_class |
| 文件操作 | file, download, upload, storage | dex_string, smali |

### 安全分析类

| 场景 | 推荐关键词 | 推荐范围 |
|------|-----------|---------|
| 敏感字符串 | password, secret, key, token, api_key | dex_string |
| 加密相关 | encrypt, decrypt, cipher, aes, rsa, md5 | dex_string, smali |
| 硬编码数据 | http://, https://, api., .com, .net | dex_string |
| 权限分析 | permission, uses-permission | axml |
| WebView | webview, javascript, loadurl | dex_string, smali |
| 组件暴露 | exported=true, intent-filter | axml |

### 结构分析类

| 场景 | 分析方法 | 说明 |
|------|---------|------|
| 入口分析 | 分析 AndroidManifest | Activity、Service、Receiver |
| 类继承 | dex_class_outline | 查看父类和接口 |
| 方法调用 | smali 搜索 | 查找 invoke 指令 |
| 资源引用 | resource_table_* | 查找资源使用 |
| 第三方库 | dex_classes + prefix | 分析第三方包结构 |

### 自定义分析

用户可完全自定义：

```
用户: "分析 微信 中所有包含 'alipay' 的代码"
→ 匹配 APK: wechat.apk 或 com.tencent.mm.apk
→ 搜索关键词: alipay
→ 范围: dex_string, dex_class, dex_method, smali
→ 生成报告: 微信_alipay分析报告.md

用户: "分析 kwyy 的敏感字符串"
→ 匹配 APK: kwyy.apk
→ 搜索关键词: password|token|secret|key
→ 范围: dex_string
→ 生成报告: kwyy_敏感字符串分析报告.md

用户: "分析 淘蓝 com.example.app 包下的所有类"
→ 匹配 APK: 淡蓝.apk 或包含 "deepblue" 的 APK
→ 列出类: prefix=com/example/app
→ 生成报告: 淡蓝_包结构分析报告.md
```

## 报告模板

```markdown
# {APK名称} - {分析类型}分析报告

## 基本信息

| 项目 | 值 |
|------|------|
| APK名称 | {apkFileName} |
| 包名 | {packageName} |
| 版本 | {versionName} ({versionCode}) |
| 最低SDK | {minSdk} |
| 目标SDK | {targetSdk} |
| 文件大小 | {文件大小} |

## 分析配置

| 配置项 | 值 |
|--------|-----|
| 分析类型 | {分析类型} |
| 搜索关键词 | {关键词列表} |
| 搜索范围 | {scopes列表} |
| 分析时间 | {时间戳} |

## 分析结果

### 1. {结果分类1}

{详细内容，包含代码片段、定位器信息等}

### 2. {结果分类2}

{详细内容}

...

## 详细代码

{可选：关键代码片段}

## 总结

{分析总结和建议}

---

*报告生成时间: {时间}*
*分析工具: mt_mcp APK Analyzer*
```

## 输出要求

1. **报告格式**: Markdown (.md)
2. **文件命名**: `{apkName}_{分析类型}报告.md`
3. **编码**: UTF-8
4. **内容结构**: 基本信息 → 分析配置 → 分析结果 → 总结 → 解决方案建议
5. **格式化**: 使用表格、代码块、列表提高可读性

## 解决方案建议

**重要原则：分析报告必须包含解决方案建议部分，指导用户如何处理分析结果。**

### 工具选择原则

| 场景 | 推荐工具 | 说明 |
|------|---------|------|
| 常规修改 | MT管理器 | 首选，手机端直接操作 |
| 字符串修改 | MT管理器 | 字符串常量替换 |
| 跳过逻辑 | MT管理器 | 方法修改、条件跳转 |
| 资源修改 | MT管理器 | XML、图片等资源 |
| 复杂重构 | 电脑端工具 | 需要重新编译的情况 |
| 大规模修改 | 电脑端工具 | 批量处理、自动化 |

### MT管理器操作指南

#### 基础操作

| 操作 | MT管理器方法 |
|------|-------------|
| 打开APK | 长按APK → 查看详情 → 打开方式 → APK查看 |
| 查看DEX | APK查看 → classes.dex → Dex编辑器++ |
| 搜索字符串 | Dex编辑器++ → 搜索 → 字符串搜索 |
| 搜索类名 | Dex编辑器++ → 搜索 → 类名搜索 |
| 搜索方法 | Dex编辑器++ → 搜索 → 方法搜索 |

#### 字符串修改

```
场景: 修改字符串常量
步骤:
1. 打开 APK → Dex编辑器++
2. 搜索 → 字符串搜索 → 输入目标字符串
3. 找到后点击 → 转到定义
4. 长按字符串 → 编辑
5. 输入新字符串 → 保存
6. 保存并退出 → 自动签名
```

#### 方法修改（跳过验证）

```
场景: 跳过条件判断（如VIP验证）
步骤:
1. 打开 APK → Dex编辑器++
2. 搜索方法 → 找到目标方法
3. 进入方法 → 查看Smali代码
4. 找到条件判断指令（if-eq, if-ne等）
5. 修改跳转逻辑:
   - if-eq 改为 if-ne（条件取反）
   - 或直接修改为 goto 跳转
6. 保存 → 保存并退出 → 自动签名
```

#### 常见修改模式

| 需求 | Smali修改方法 |
|------|--------------|
| 跳过VIP验证 | if-eq → if-ne 或 return-void |
| 解锁功能 | const/4 v0, 0x0 → const/4 v0, 0x1 |
| 移除广告 | return-void 提前返回 |
| 绕过登录 | 修改登录状态检查 |
| 禁用更新 | 修改版本比较逻辑 |

#### XML修改

```
场景: 修改AndroidManifest或布局XML
步骤:
1. 打开 APK → 查看详情
2. 找到目标XML文件
3. 长按 → 编辑（文本编辑器）
4. 修改内容 → 保存
5. 返回 → 自动重新打包签名
```

#### 资源修改

```
场景: 修改图片、字符串资源
步骤:
1. 打开 APK → 查看详情
2. 导航到 res/ 目录
3. 找到目标资源文件
4. 替换/编辑资源
5. 返回 → 自动重新打包签名
```

### 电脑端工具推荐

当MT管理器无法处理时，推荐以下电脑端工具：

| 工具 | 用途 | 说明 |
|------|------|------|
| jadx | 反编译查看 | 查看Java源码，理解逻辑 |
| apktool | 完整反编译 | 反编译Smali和资源 |
| Android Studio | 调试分析 | 动态调试、内存分析 |
| Frida | 动态Hook | 运行时修改、绕过检测 |
| Xposed | 框架Hook | 系统级Hook |
| np管理器 | 类似MT | 另一个手机端选择 |

#### 电脑端工作流程

```
1. 使用 jadx 查看并理解代码逻辑
2. 使用 apktool 反编译 APK
3. 修改 Smali 代码或资源
4. 使用 apktool 重新打包
5. 使用签名工具签名
6. 安装测试
```

### 解决方案模板

在报告中添加以下部分：

```markdown
## 解决方案建议

### 推荐工具

本分析结果推荐使用 **MT管理器** 进行修改。

### 修改步骤

1. **定位目标**
   - 类名: `com.example.xxx`
   - 方法名: `methodName`
   - 关键字符串: `xxx`

2. **MT管理器操作**
   - 打开APK → Dex编辑器++
   - 搜索 [类名/方法名/字符串]
   - 进入目标位置

3. **具体修改**
   - 找到指令: `if-eq v0, v1, :cond_x`
   - 修改为: `if-ne v0, v1, :cond_x`
   - 保存并退出

4. **验证**
   - 安装修改后的APK
   - 测试功能是否正常

### 注意事项

- 修改后需要重新签名
- 部分应用有签名校验，需要额外处理
- 建议备份原APK

### 备选方案

如果MT管理器无法处理，推荐使用:
- jadx + apktool（电脑端）
- Frida（动态Hook）
```

### 特殊情况处理

| 情况 | MT管理器方案 | 电脑端方案 |
|------|-------------|-----------|
| 签名校验 | 搜索签名验证代码并移除 | Lucky Patcher / 核心破解 |
| 加壳应用 | 无法处理 | 脱壳工具（BlackDex等） |
| 混淆代码 | 需要分析映射关系 | jadx + proguard mapping |
| Native库 | 无法直接修改 | IDA Pro / Ghidra |
| 资源加密 | 需要解密脚本 | Python脚本处理 |

## 高级用法

### 多关键词组合搜索

```json
mt_apk_search({
  "workspaceId": "xxx",
  "query": "payment|pay|purchase|order",
  "queryType": "regex",
  "caseSensitive": false,
  "scopes": ["dex_string", "dex_class"]
})
```

### 限定包范围搜索

```json
mt_apk_search({
  "workspaceId": "xxx",
  "query": "config",
  "queryType": "literal",
  "caseSensitive": false,
  "scopes": "dex_string",
  "classPrefix": "Lcom/example/"
})
```

### 精确匹配类名

```json
mt_apk_search({
  "workspaceId": "xxx",
  "query": "MainActivity",
  "queryType": "literal",
  "caseSensitive": true,
  "matchMode": "exact",
  "scopes": "dex_class"
})
```

### 带位置信息搜索

```json
mt_apk_search({
  "workspaceId": "xxx",
  "query": "关键词",
  "queryType": "literal",
  "caseSensitive": false,
  "scopes": "smali",
  "includeMatchOffsets": true
})
```

## 注意事项

1. **APK 选择**: 支持模糊名称匹配，无需精确输入完整文件名
2. **多匹配处理**: 如果匹配到多个 APK，列出结果让用户选择
3. **大型 APK**: 使用分页处理（limit + nextCursor）
4. **性能优化**: 使用 classPrefix、zipEntryPrefix 缩小范围
5. **敏感信息**: 在报告中适当脱敏
6. **资源释放**: 分析完成后可关闭工作区
7. **报告保存**: 保存在当前工作目录
