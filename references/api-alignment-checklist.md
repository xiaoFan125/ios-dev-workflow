# 网络层迁移 API 对齐清单

## 概述

从旧 DomainNetwork 迁移到 ServiceType 时，必须逐个对比每个 API 的 path/method/parameters，不能只看 API 名称。

## 常见差异

### 1. Path 前缀不同
```
旧: /api/v2/user/detail
新: /api/v2/user/detail  ✓ 一致

旧: /api/v1/discovery/list
新: /api/v1/discovery/list  ✓ 一致

旧: /api/v2/firmware/last
新: /api/v1/ota/newest  ✗ 路径完全不同！
```

### 2. 参数名不同
```
旧: uid (Int)
新: userId (String)  ✗ 名称和类型都不同

旧: captcha
新: verifyCode  ✗ 名称不同
```

### 3. 参数类型不同
```
旧: userId: Int = 12345
新: uid: String = "12345"  ✗ 需要转换

旧: type: ServiceLayerAPI.VerifyCodeType = .signIn (Int 枚举)
新: type: AccountTarget.VerifyCodeType = .signIn (Int 枚举)  ✓ 一致
```

### 4. 缺少必填参数
```
旧: api_getOTANewestFirmWareVersion(version:device:username:mac:serialNo:)
新: getLatestFirmwareVersion(deviceType:)  ✗ 缺少 version/username/mac/serialNo
```

### 5. 服务器地址不同
```
旧: 账户 API → identity.test.aivustore.com
旧: 设备/发现 API → api.test.aivustore.com
旧: chat/translate → thirdparty.example.com

新: 需要分别绑定 .bizAPI / .cloudAPI / .thirdParty 服务
```

## 迁移清单

### Account API (BusinessTarget)
- [ ] signInPassword — path/params 对比
- [ ] signInCode — path/params 对比
- [ ] signInApple — path/params 对比
- [ ] requestVerificationCode — path/params 对比
- [ ] checkVerifyCode — path/params 对比
- [ ] register — path/params 对比
- [ ] fetchUserInfo — path/params 对比
- [ ] updatePassword — path/params 对比
- [ ] updatePhoneNumber — path/params 对比
- [ ] updateEmail — path/params 对比
- [ ] updateNickname — path/params 对比
- [ ] updateAvatar — multipart 上传，单独处理
- [ ] deleteAccount — path/params 对比
- [ ] logOut — path/params 对比
- [ ] appVersion — path/params 对比
- [ ] heartbeat — path/params 对比（无参也需要 Sign）

### Device API (CloudTarget)
- [ ] getDeviceIdentifier — path/params 对比
- [ ] deviceBind — path/params 对比
- [ ] deviceUnbind — path/params 对比
- [ ] getDevicesList — path/params 对比
- [ ] getDeviceDetail — path/params 对比
- [ ] deviceRename — path/params 对比
- [ ] resetToFactoryMode — path/params 对比
- [ ] requestDeviceTypeList — path/params 对比
- [ ] requestGuideList — path/params 对比

### Glasses API (CloudTarget)
- [ ] mapSaveAddress — path/params 对比
- [ ] mapGetAddressList — path/params 对比
- [ ] mapChangeAddressList — path/params 对比
- [ ] mapDeleteAddress — path/params 对比
- [ ] discoveryList — path/params 对比（缺 uid/tag？）
- [ ] discoveryDetail — path/params 对比
- [ ] discoveryFavorite — path/params 对比

### Glasses ThirdParty API
- [ ] chatAI — path/params 对比
- [ ] chat — path/params 对比
- [ ] chatgpt — path/params 对比
- [ ] upload — multipart 上传
- [ ] googleTranslate — path/params 对比

### OTA API (CloudTarget)
- [ ] getOTAFirmwareVersion — path/params 对比（缺 uid？）
- [ ] getOTAFirmwareList — path/params 对比
- [ ] getLatestFirmwareVersion — path/params 对比（缺 serialNo？字段名错误？）
- [ ] getAppVersionList — path/params 对比
- [ ] getAppNewestVersion — path/params 对比
- [ ] otaFirmwareDownloading — path/params 对比

### Live API (CloudTarget)
- [ ] createLiveRoom — path/params 对比
- [ ] getRoomList — path/params 对比
- [ ] getLiveRoomDetail — path/params 对比
- [ ] joinLiveRoom — path/params 对比
- [ ] leaveLiveRoom — path/params 对比

### Feedback API (CloudTarget)
- [ ] feedBackUploadImage — multipart 上传
- [ ] feedBackSubmit — path/params 对比
- [ ] feedBackSubmitLog — path/params 对比
- [ ] feedBackCrashOrLogUpload — multipart 上传

### VIP API (CloudTarget)
- [ ] requestVipInfo — path/params 对比
- [ ] requestMemberInfo — path/params 对比
- [ ] createOrder — path/params 对比
- [ ] requestOrders — path/params 对比
- [ ] requestOrderDetail — path/params 对比

### Pay API (CloudTarget)
- [ ] inspectionOrder — path/params 对比

### Meditation API (CloudTarget)
- [ ] fetchMusicList — path/params 对比

## 验证方法

1. 逐个 API 对比旧代码 `ServiceLayerAPI+Xxx.swift` 中的 `referenceURL` 和参数
2. 用 LoggerInterceptor 打印请求日志，对比新旧请求的 URL、Headers、Body
3. 服务器返回 20101 "请求参数异常" = 参数格式错误
4. 服务器返回 20103 "身份验证失败" = Sign 签名不匹配
