# Multipart 上传 Target 模式

## 问题

旧代码使用 `DomainNetworkCore.multipartFormRequest` 上传文件（如头像），新网络层需要等价实现。

## 解决方案

创建同时遵循 `MultipartTarget` 和 `BusinessTarget` 的 Target：

```swift
/// 头像上传 Multipart Target
struct AvatarUploadTarget: MultipartTarget, BusinessTarget {
    typealias Response = JSONResponse
    
    let uid: String
    let fileData: Data
    let fileName: String
    let mimeType: String
    
    // BusinessTarget 协议
    var apiPath: String { "/api/v2/user/img" }
    var apiMethod: HTTPMethod { .post }
    var apiParameters: [String: Any]? { ["uid": uid] }
    var apiEncoding: NKParameterEncoding { .json }
    var apiTimeout: TimeInterval { 60 }
    var needsSign: Bool { true }
    
    // MultipartTarget 协议
    func buildMultipartFormData(_ builder: MultipartFormDataBuilder) {
        builder.append(value: uid, name: "uid")
        builder.append(fileData: fileData, name: "file", fileName: fileName, mimeType: mimeType)
    }
}
```

## 调用方式

```swift
let target = AvatarUploadTarget(uid: "\(userId)", fileData: imageData, fileName: "avatar.jpeg", mimeType: "image/jpeg")

_ = NetworkCore.shared.upload(target) { result in
    switch result {
    case .success(let response):
        let dict = response.json
        // 处理响应
    case .failure(let error):
        // 处理错误
    }
}
```

## 关键点

1. **同时遵循两个协议** — `MultipartTarget`（构建表单数据）+ `BusinessTarget`（签名 headers）
2. **buildMultipartFormData** — 使用 `MultipartFormDataBuilder` 添加字段和文件
3. **参数放在 apiParameters** — 非文件参数仍需在 apiParameters 中声明
4. **文件数据在 buildMultipartFormData 中添加** — 不是通过 apiParameters

## Crash Reporting 不适用此模式

Crash reporting 应独立于主网络层，使用 `URLSession.uploadTask` 直接实现，不走 NetworkCore。
