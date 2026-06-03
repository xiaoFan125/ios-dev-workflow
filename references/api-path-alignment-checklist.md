# API Path 对齐清单（旧代码 vs 新代码）

## 原则

迁移时必须逐个对照旧代码的 `referenceURL`，不能只看 API 名称推断路径。

## AccountAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| requestVerificationCode | /api/tozo/captcha/send | /api/tozo/captcha/send |
| checkVerifyCode | /api/v2/captcha/check | /api/v2/captcha/check |
| signInCode | /api/tozo/captcha/login | /api/tozo/captcha/login |
| signInApple | /api/v2/oauth/iOS | /api/v2/oauth/iOS |
| signInPassword | /api/tozo/login | /api/tozo/login |
| checkUsername | /api/v2/user/check | /api/v2/user/check |
| register | /api/tozo/user/register | /api/tozo/user/register |
| uploadGoogle | /api/v2/oauth/google | /api/v2/oauth/google |
| fetchUserInfo | /api/v2/user/detail | /api/v2/user/detail |
| updateAvatar | /api/v2/user/img | /api/v2/user/img |
| updatePassword | /api/tozo/user/password | /api/tozo/user/password |
| updatePhoneNumber | /api/v2/user/mobile | /api/v2/user/mobile |
| updateEmail | /api/v2/user/email | /api/v2/user/email |
| bindThirdParty | /api/v2/oauth/bind | /api/v2/oauth/bind |
| updateNickname | /api/v2/user/nickname | /api/v2/user/nickname |
| deleteAccount | /api/v2/user/delete | /api/v2/user/delete |
| logOut | /api/v2/user/logout | /api/v2/user/logout |
| cancelAccount | /api/tozo/user/logout | /api/tozo/user/logout |
| forgotPassword | /api/tozo/captcha/password | /api/tozo/captcha/password |
| bindAccount | /api/v2/oauth/bind | /api/v2/oauth/bind |
| bindSocial | /api/v2/user/bind | /api/v2/user/bind |
| unbindSocial | /api/v2/user/unbind | /api/v2/user/unbind |
| getUserKey | /api/v2/user/key | /api/v2/user/key |
| heartbeat | /api/v2/system/health | /api/v2/system/health |
| updateUserInfo | /api/v2/user/info | /api/v2/user/info |

## DeviceAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| getDeviceIdentifier | /api/v2/terminals/check | /api/v2/terminals/check |
| deviceBind | /api/v2/terminals/bind | /api/v2/terminals/bind |
| deviceUnbind | /api/v2/terminals/unbind | /api/v2/terminals/unbind |
| getDevicesList | /api/v2/terminals/list | /api/v2/terminals/list |
| getDeviceDetail | /api/v2/terminals/detail | /api/v2/terminals/detail |
| deviceRename | /api/v2/terminals/name | /api/v2/terminals/name |
| resetToFactoryMode | /api/v2/terminals/reset | /api/v2/terminals/reset |
| requestDeviceTypeList | /api/v2/terminals/typeList | /api/v2/terminals/typeList |
| requestGuideList | /api/v2/operation/list | /api/v2/operation/list |

## GlassesAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| mapSaveAddress | /api/v2/userSetup/save | /api/v2/userSetup/save |
| mapGetAddressList | /api/v2/userSetup/list | /api/v2/userSetup/list |
| mapChangeAddressList | /api/v2/userSetup/update | /api/v2/userSetup/update |
| mapDeleteAddress | /api/v2/userSetup/delete | /api/v2/userSetup/delete |
| discoveryList | /api/v2/discover/List | /api/v2/discover/List |
| discoveryDetail | /api/v2/discover/detail | /api/v2/discover/detail |
| discoveryFavorite | /api/v2/discover/favorite | /api/v2/discover/favorite |

## GlassesThirdPartyAPI

| API | 旧 path | 服务器 |
|-----|---------|--------|
| chatAI | /api/v1/chat/completions | tparty |
| chat | /api/v1/chat/completions | tparty |
| chatgpt | /api/v1/chat/chatgpt | tparty |
| upload | /api/v1/utils/upload | tparty |
| googleTranslate | /api/v1/google/translate | tparty |

## OTAAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| getOTAFirmwareVersion | /api/v2/firmware/detail | /api/v2/firmware/detail |
| getOTAFirmwareList | /api/v2/firmware/list | /api/v2/firmware/list |
| getLatestFirmwareVersion | /api/v2/firmware/last | /api/v2/firmware/last |
| getAppVersionList | /api/v2/app/list | /api/v2/app/list |
| getAppNewestVersion | /api/v2/file/last | /api/v2/file/last |

## LiveAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| createLiveRoom | /api/v2/live/create | /api/v2/live/create |
| getRoomList | /api/v2/live/list | /api/v2/live/list |
| joinLiveRoom | /api/v2/live/enter | /api/v2/live/enter |
| leaveLiveRoom | /api/v2/live/exit | /api/v2/live/exit |

## VIPAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| requestVipInfo | /api/v2/plan/list | /api/v2/plan/list |
| requestMemberInfo | /api/v2/record/last | /api/v2/record/last |
| createOrder | /api/v2/record/create | /api/v2/record/create |
| requestOrders | /api/v2/record/list | /api/v2/record/list |
| requestOrderDetail | /api/v2/record/detail | /api/v2/record/detail |

## PayAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| inspectionOrder | /api/v2/record/check | /api/v2/record/check |

## FeedbackAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| feedBackUploadImage | /api/v2/feedback/upload | /api/v2/feedback/upload |
| feedBackSubmit | /api/v2/feedback/create | /api/v2/feedback/create |
| feedBackSubmitLog | /api/v2/system/upload | /api/v2/system/upload |
| feedBackCrashOrLogUpload | /api/v2/system/upload | /api/v2/system/upload |

## MeditationAPI

| API | 旧 path | 新 path |
|-----|---------|---------|
| fetchMusicList | /api/v2/meditate/list | /api/v2/meditate/list |

## 服务器地址

| 服务 | 测试环境 | 生产环境 |
|------|----------|----------|
| bizAPI (账户) | identity.test.aivustore.com | identity.tozostore.com |
| cloudAPI (云) | api.test.aivustore.com | app-api.tozostore.com |
| thirdParty | thirdparty.example.com | tparty.tozostore.com |
| voiceChat | api.test.aivustore.com | vcapi.tozo.tozostore |
