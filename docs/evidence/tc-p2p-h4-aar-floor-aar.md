# TC-P2P-H4-AAR-FLOOR-AAR evidence

Card: **TC-P2P-H4-AAR-FLOOR-AAR** (ACK P4).  
Goal: F1 Java/JNI for `setBaseMinimumPlayoutDelayMs`, rebuild the AAR on this Cursor Cloud VM, record sha256 and symbols.

This note is build evidence only. It is not CLEARED, not Soft-close, and it does not pin Host.

## Build host

| Item | Value |
|---|---|
| Host | Cursor Cloud VM (`hostname=cursor`). Not the Lab machine. |
| `uname` | Linux cursor 6.12.113+ #1 SMP PREEMPT_DYNAMIC x86_64 |
| Built at | 2026-11-19T17:29:06Z (evidence collected) |
| Toolchain | depot_tools + `tools_webrtc/android/build_aar.py` + ninja, then `./gradlew :stream-webrtc-android:assembleRelease` |
| Android SDK for the Gradle AAR | `/home/ubuntu/android-sdk` platforms;android-35, build-tools 35.0.0 |

## Source alignment

| Tree | Commit |
|---|---|
| `stream-webrtc-android` base (this fork `main`) | `c3e85c7f1fcbae2d43db876edb84b1588789cbe4` |
| Upstream tag `1.3.10` | `5d388e533d87f857e7de4a729630200cf9766369` (ancestor of `main`; same `libjingle_peerconnection_so.so` blob before this rebuild) |
| Native tree rebuilt | `GetStream/webrtc` `4655e557d1206a42899b6b03fbf017fc64c654cc` (`patch/m125`, `[AND-179] Refactor external audio processing`, 2024-12-17). Same day as the wrapper commit that landed the m125.4 SOs. |

`versionName` stays **1.3.10**. The artifact identity is the sha256 below.

## F1 path (not F3)

`webrtc::Call` in this m125 tree does not declare `SetBaseMinimumPlayoutDelayMs`. The log strings already in the previous SO (`SetBaseMinimumPlayoutDelayMs: no recv stream`, `Failed to SetBaseMinimumPlayoutDelayMs on AudioReceiveStreamInterface`) come from `WebRtcVoiceReceiveChannel::SetBaseMinimumPlayoutDelayMs(uint32_t ssrc, int delay_ms)`. That is the peer-connection equivalent, and it is linkable. F1 was not falsified, so a from-scratch Chromium tree was not used.

JNI `Java_org_webrtc_RtpReceiver_nativeSetBaseMinimumPlayoutDelayMs` calls `AudioRtpReceiver::SetBaseMinimumPlayoutDelayMs`, which calls `media_channel_->SetBaseMinimumPlayoutDelayMs(ssrc, delay_ms)`. That channel method calls `SetBaseMinimumDelayMs` on the audio receive stream / ACM. It does not call `SetMinimumDelay`.

`false` is returned for a non-audio receiver, a delay outside 0..10000 ms, no media channel, no SSRC, or a native `false`. A valid delay with no stream yet is cached for `RestartMediaChannel_w` and still returns `false` for that call.

Patch applied to the native tree: `patches/m125.4-set-base-minimum-playout-delay.patch`.

FlexFEC stays in the same SOs (`Call::CreateFlexfecReceiveStream` and the existing FlexFEC strings). No separate FlexFEC change.

## AAR sha256

File: `stream-webrtc-android/build/outputs/aar/stream-webrtc-android-release.aar` (Gradle release AAR; not committed).

```
46d601b2803ee52eaca76eb90f7d2177050e72262661fb7b2ac0dbe051539da2  stream-webrtc-android-release.aar
```

Inner hashes:

```
1ab3deb8dd75d9b5645884b478cf3c3e634a23f3d7028a3f6b9e35f4e0b02fb2  classes.jar
1c3d684ddaee3ebc29560a0814bec3d587407a9120aa55da2adcc5317e6b0303  jni/arm64-v8a/libjingle_peerconnection_so.so
e2764d344a5a0903f6843ca0f577c2ba34b6d217c9f17da9b7e22f2f35168327  jni/armeabi-v7a/libjingle_peerconnection_so.so
1b0f86f0eaf5f6a002159a995b78d42e4e455a57da92ab1d9f1af5fad3a2895f  jni/x86/libjingle_peerconnection_so.so
701114d7241fb711ecf79f45097068074960b998ab9cd0f0314d6ecabc536e35  jni/x86_64/libjingle_peerconnection_so.so
```

## javap

`javap -public -classpath classes.jar org.webrtc.RtpReceiver`

```
public class org.webrtc.RtpReceiver {
  public org.webrtc.RtpReceiver(long);
  public org.webrtc.MediaStreamTrack track();
  public org.webrtc.RtpParameters getParameters();
  public java.lang.String id();
  public void dispose();
  public void SetObserver(org.webrtc.RtpReceiver$Observer);
  public void setFrameDecryptor(org.webrtc.FrameDecryptor);
  public boolean setBaseMinimumPlayoutDelayMs(int);
  public int getBaseMinimumPlayoutDelayMs();
}
```

`javap -private` also shows:

```
private static native boolean nativeSetBaseMinimumPlayoutDelayMs(long, int);
private static native int nativeGetBaseMinimumPlayoutDelayMs(long);
```

No `setMinimumDelay` method on `RtpReceiver`.

## nm -D (from the AAR `jni/` libraries)

```
arm64-v8a
000000000093cd88 T Java_org_webrtc_RtpReceiver_nativeGetBaseMinimumPlayoutDelayMs
000000000093ce8c T Java_org_webrtc_RtpReceiver_nativeSetBaseMinimumPlayoutDelayMs

armeabi-v7a
00546a01 T Java_org_webrtc_RtpReceiver_nativeGetBaseMinimumPlayoutDelayMs
00546a71 T Java_org_webrtc_RtpReceiver_nativeSetBaseMinimumPlayoutDelayMs

x86
00a0bab1 T Java_org_webrtc_RtpReceiver_nativeGetBaseMinimumPlayoutDelayMs
00a1e965 T Java_org_webrtc_RtpReceiver_nativeSetBaseMinimumPlayoutDelayMs

x86_64
0000000000ace392 T Java_org_webrtc_RtpReceiver_nativeGetBaseMinimumPlayoutDelayMs
0000000000ace446 T Java_org_webrtc_RtpReceiver_nativeSetBaseMinimumPlayoutDelayMs
```

Unstripped arm64 also contains `webrtc::AudioRtpReceiver::SetBaseMinimumPlayoutDelayMs(int)`.

There is no `Java_org_webrtc_RtpReceiver_nativeSetMinimumDelay` symbol.
