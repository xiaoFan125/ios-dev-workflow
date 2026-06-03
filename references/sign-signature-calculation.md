# Sign 签名计算详解

## 背景

BrandA/BrandC 业务 API 使用 Sign 签名验证请求合法性。签名基于 headers + 请求参数计算 MD5。

## 旧代码签名逻辑（ServiceLayerAPI）

```swift
// 1. 参数大写
let parametersDicUpperCase = ["Username": username, "CountryCode": countryCode]

// 2. 合并 headers
let mergedDic = headers.merging(parametersDicUpperCase) { (headers, _) -> String in headers }

// 3. 排序 + 格式化
let allParameter = dic.sorted(by: {$0.key < $1.key})
var signMutStr = ""
var tokenStr = "Token="
for (_, key) in allParameter.enumerated() {
    if key.key == "Token" {
        tokenStr = String(format: "%@=%@", key.key, key.value)
    } else {
        let keyStr = String(format: "%@=%@&", key.key, key.value)
        if !key.value.isEmpty {
            signMutStr = signMutStr + keyStr
        }
    }
}
return signMutStr + tokenStr
// 结果: "CountryCode=+86&Customer=BrandC&Language=zh&Timestamp=xxx&Username=xxx&Token=xxx"
```

## 新代码签名逻辑（RequestSigner）

```swift
enum RequestSigner {
    static func sign(headers: [String: String], parameters: [String: Any]?) -> String {
        // 1. 参数 key 首字母大写
        var upperParams: [String: String] = [:]
        if let parameters {
            for (key, value) in parameters {
                let upperKey = key.prefix(1).uppercased() + key.dropFirst()
                let strValue = "\(value)"
                if !strValue.isEmpty {
                    upperParams[upperKey] = strValue
                }
            }
        }

        // 2. 合并 headers + 大写参数
        let merged = headers.merging(upperParams) { headers, _ in headers }
        let sortedKeys = merged.keys.sorted()

        // 3. 格式化为 key=value 对，Token 放最后
        var signParts: [String] = []
        var tokenPart = ""
        for key in sortedKeys {
            guard let value = merged[key], !value.isEmpty else { continue }
            let part = "\(key)=\(value)"
            if key == "Token" {
                tokenPart = part
            } else {
                signParts.append(part)
            }
        }

        // 4. 拼接 + MD5
        let signString = signParts.joined(separator: "&")
        if !tokenPart.isEmpty {
            return (signString + "&" + tokenPart).md5
        }
        return signString.md5
    }
}
```

## 关键 Pitfall

**✗ 错误格式**（直接拼接 keyvalue）:
```
CountryCode+86CustomerTOTimestampxxxUsernamexxxTokenxxx
```
→ 服务器返回 "身份验证失败" (code 20103)

**✓ 正确格式**（key=value& 分隔）:
```
CountryCode=+86&Customer=BrandC&Language=zh&Timestamp=xxx&Username=xxx&Token=xxx
```
→ 服务器正常响应

## Headers 必须字段

| 字段 | 值 | 来源 |
|------|-----|------|
| Timestamp | Unix 时间戳（秒） | `Int(Date().timeIntervalSince1970)` |
| Token | 用户 token | `UserManager.shared.user?.token` |
| Customer | 固定 "BrandC" | 硬编码 |
| Language | 语言代码 | `Locale.current.language.languageCode?.identifier` |

## 注意事项

1. **BusinessTarget 不能直接引用 UserManager** — ServiceType 模块无法访问 App 层的 UserManager 类。应从 `NetworkCoreConfiguration.shared.mergedHeaders(for: .bizAPI)` 获取 token
2. **Language 使用系统 API** — 不要依赖 App 层的 LanguageManager 类，使用 `Locale.current` 获取
3. **空值过滤** — 签名计算时跳过空值的 key-value 对
4. **Token 特殊处理** — Token 放在签名字符串最后，前面加 `&`
