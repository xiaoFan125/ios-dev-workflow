# Agora SDK `didAudioSubscribeStateChange` Investigation

## Summary

Agora SDK 4.6.2 (`from: "4.6.2"` in Package.swift) does NOT trigger `didAudioSubscribeStateChange` callback under any tested configuration.

## Tested Fixes (all failed)

### Fix 1: `enableAudioVolumeIndication` before joinChannel

```swift
// Added in AgoraVoiceEngine init, BEFORE joinChannel
agoraKit?.enableAudioVolumeIndication(500, smooth: 3, reportVad: true)
```

**Result**: Callback still not triggered.

### Fix 2: Force `enableAudioRecordingOrPlayout = true`

```swift
// Before (conditional):
if policy.usesSDKPlayback || policy.usesExternalAudioSink {
    mediaOptions.enableAudioRecordingOrPlayout = true
}

// After (unconditional):
mediaOptions.enableAudioRecordingOrPlayout = true
```

**Result**: Callback still not triggered.

### Fix 3: Both fixes combined

**Result**: Callback still not triggered. Tested multiple times, different sessions.

## Confirmed Working Alternative

Use `remoteUserDidJoin` as the handshake trigger:

```swift
public func agoraVoiceSession(_ session: AgoraVoiceSession, remoteUserDidJoin uid: UInt) {
    guard !isTearingDown, session === voiceSession else { return }
    guard uid != localRtcUid else { return }
    adoptBotRemoteUid(uid)
    session.ensureRemoteAudioSubscribed(uid: uid)
    // Backup: trigger handshake here since didAudioSubscribeStateChange may not fire
    _ = handshakeGate.markAudioSubscribed()
    performHandshakeIfReady()
}
```

**Result**: `client.ready` and `metric.client` sent successfully every time.

## SDK Configuration (for reference)

```swift
// AgoraVoiceDeviceProfile.swift
case .glassV2:
    usesExternalAudioSink = false
    usesSDKPlayback = true
    needsAudioSessionRestriction = true
    handshakePrefersBotSubscribe = true  // ← This flag exists but callback never fires

// Channel media options
mediaOptions.autoSubscribeAudio = true
mediaOptions.enableAudioRecordingOrPlayout = true
mediaOptions.clientRoleType = .broadcaster
mediaOptions.channelProfile = .liveBroadcasting
```

## Root Cause Hypothesis

The `didAudioSubscribeStateChange` callback may only fire in specific Agora SDK configurations (e.g., when using `AgoraRtcEngineDelegate` directly vs through the wrapper, or when the SDK internally manages audio routing differently). In our setup with custom audio tracks and SDK playback mode, the callback is simply never invoked.

## Recommendation

- Keep `didAudioSubscribeStateChange` implementation as a future optimization point
- Use `remoteUserDidJoin` as the primary/backup handshake trigger
- `markAudioSubscribed()` has internal `guard !audioSubscribed` protection, so calling it from both places is safe
