# Batch API Migration Pattern (ServiceLayerAPI → ServiceType)

## Context

Migrating 49+ files from old ObjC network stack (DomainNetwork/ServiceLayer/DomainNetworkLayer) to new Swift stack (ServiceType).

## Migration Steps

### Step 1: Import Replacement (Batch)

```bash
# Replace imports in all files
find SmartGlasses -name "*.swift" -exec grep -l "import ServiceLayer\|import DomainNetwork" {} \; | while read f; do
  sed -i '' 's/import ServiceLayer/import ServiceType/' "$f"
  sed -i '' 's/import DomainNetwork//' "$f"
  sed -i '' 's/import DomainNetworkLayer//' "$f"
done
```

### Step 2: API Call Replacement (Per-File)

**Old pattern:**
```swift
ServiceLayerAPI.defaultService()?.someMethod(params) { (success, dict, message) in
    // handle
}
```

**New pattern:**
```swift
NetworkCore.shared.sendRequest(SomeTarget(.someMethod(params))) { (success, dict, message) in
    // handle (same callback signature)
}
```

### Step 3: Type Migration

When removing ObjC modules, copy dependent types:
```bash
cp Modules/DomainNetwork/Classes/User.swift SmartGlasses/AppGroup/
cp Modules/DomainNetwork/Classes/UserManager.swift SmartGlasses/AppGroup/
# Then add to Xcode target manually
```

### Step 4: Delegate Cleanup

Old ObjC delegate pattern (`ServiceLayerDelegate`) must be fully removed:
- `ServiceLayerAPI.defaultDelegate = self` → delete
- `extension AppDelegate: ServiceLayerDelegate { ... }` → delete entirely
- Keep `initNetwork()` logic (environment config, NetworkCoreBootstrap.configure())

## API Mapping Reference

| Old Method | New Target |
|------------|-----------|
| `requestVerificationCode(type:username:countryCode:)` | `AccountTarget(.requestVerificationCode(type:username:countryCode:))` |
| `signIn(code:username:countryCode:)` | `AccountTarget(.signInCode(captcha:username:countryCode:))` |
| `signIn(password:username:countryCode:)` | `AccountTarget(.signInPassword(username:password:countryCode:))` |
| `signIn(userIdentifier:fullName:email:)` | `AccountTarget(.signInApple(providerId:nickname:username:))` |
| `api_deviceUnbind(tid:)` | `DeviceTarget(.deviceUnbind(deviceId:))` |
| `discoverySeriviceList(start:length:favorite:)` | `GlassesTarget(.discoveryList(start:length:favorite:))` |

## Pitfalls

1. **`ServiceType.shared` doesn't exist** — Use `NetworkCore.shared` (the shared instance is on the generic library, not the business layer)
2. **Missing API parameters** — New Targets require all parameters explicitly (no defaults from old ObjC methods)
3. **`ServiceLayerAPI.VerifyCodeType`** — Replace with `AccountTarget.VerifyCodeType`
4. **delegate_task timeout** — 49+ files will timeout in Claude Code delegation. Split into batches of 10-15, or use `execute_code` for batch import replacement first
5. **DomainNetworkCore direct usage** — Old code using `DomainNetworkCore.multipartFormRequest` (e.g., crash reporting) should be replaced with `URLSession` directly, not the new network layer (crash reporting should be independent of app network layer)
6. **ServiceLayerAPI extension methods** — Old code calling `ServiceLayerAPI.defaultService()?.checkErroList()` should be replaced with local static methods in the consuming class
7. **Missing Target cases** — If old API has a method not in new Target enum, add the case to the API enum + implement apiPath/apiMethod/apiParameters
8. **Type parameter mismatch** — Old `Gender` enum may need `.rawValue` conversion: `gender: gender.rawValue` if new Target expects `Int` not `enum`
