# ResQMesh: Build Plan and Phase 1 Spec

Offline phone-to-phone emergency mesh for disasters. No internet, SIM, or router needed. Any phone that finds internet later acts as a gateway.

**How to use this file:** Put it in the project root. Give your VS Code coding agent the prompt in section 0, then follow the phases in order. Phase 1 is the only thing to build first. Everything else is a later phase.

-----

## 0. Prompt to paste into the VS Code agent

> Read RESQMESH_PLAN.md fully. Build **Phase 1 only** as a native Android app (Kotlin, Jetpack Compose, Google Nearby Connections). Create the whole Gradle project from scratch in this folder using the structure in section 4. Make it build with `./gradlew assembleDebug` and produce an installable APK. Follow the acceptance criteria in section 6. Do not add features from later phases. If a library version conflicts, pick the nearest compatible version and tell me what you changed. When done, give me the exact commands to build and install on a phone.

---

## 1. Goal of Phase 1

An installable Android app where phones **discover and connect to each other automatically with no network**, and users can **send chat messages that hop across phones**.

Phase 1 is done when 3 phones, with Wi-Fi not connected to anything and mobile data off, all show each other as peers and a message sent from one appears on all others.

---

## 2. Tech stack (locked, do not substitute)

| Concern | Choice | Reason |
|---|---|---|
| Platform | Native Android (minSdk 26, targetSdk 34, compileSdk 34) | Full control of Bluetooth and Wi-Fi radios |
| Language | Kotlin 2.0.x | Standard for Android |
| UI | Jetpack Compose + Material 3 | Fast to write, no XML |
| P2P transport | Google Nearby Connections API (`play-services-nearby`) | Handles BLE, Bluetooth and Wi-Fi Direct automatically |
| Strategy | `Strategy.P2P_CLUSTER` | Many-to-many topology |
| Message format | JSON via `org.json` (switch to Protobuf later) | Zero setup |
| State | `ViewModel` + `StateFlow` | Survives rotation, clean separation |
| Build | Gradle (Kotlin DSL) via command line | Works from VS Code |
| Later phases | Room (storage), FusedLocationProvider (GPS), Node.js + Express (gateway), Leaflet (map) | Not in Phase 1 |

**Why native and not Flutter or React Native:** Nearby Connections and foreground services are first-class in native Kotlin. Cross-platform wrappers add risk for no benefit in a 24-hour build.

**Limitation to state honestly:** Nearby Connections needs Google Play Services, so it works on Android phones with Google services (nearly all), not on iOS. iOS is future work.

---

## 3. Dev environment setup (VS Code on your laptop)

1. Install **JDK 17** (Temurin or Microsoft build). Set `JAVA_HOME`.
2. Install **Android command-line tools** (or Android Studio just to get the SDK, then use VS Code for editing). Set `ANDROID_HOME`.
3. Install SDK packages:
   ```
   sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0"
   ```
4. VS Code extensions: *Kotlin*, *Gradle for Java*, *XML*.
5. On each phone: **Settings > About phone > tap Build number 7 times > Developer options > USB debugging ON.**
6. Verify: `adb devices` lists your phone.

> Tip: if the agent struggles to scaffold Gradle by hand, create an "Empty Activity" (Compose) project once in Android Studio, open that folder in VS Code, and have the agent replace the code. This is the most reliable path.

---

## 4. Project structure

```
ResQMesh/
├── settings.gradle.kts
├── build.gradle.kts                 (root, plugin versions)
├── gradle.properties
├── gradle/wrapper/...
└── app/
    ├── build.gradle.kts
    └── src/main/
        ├── AndroidManifest.xml
        └── java/com/resqmesh/app/
            ├── MainActivity.kt          UI host, permission request
            ├── MainViewModel.kt         holds state, talks to MeshManager
            ├── mesh/
            │   ├── MeshManager.kt       Nearby: advertise, discover, connect, send, relay
            │   └── MeshMessage.kt       data class + JSON (de)serialize
            └── ui/
                ├── MeshScreen.kt        peers header, message list, input bar
                └── Theme.kt
```

Package name: `com.resqmesh.app`. App name: **ResQMesh**.

### Pinned versions (adjust if the build complains)

- Gradle wrapper 8.9
- Android Gradle Plugin 8.5.2
- Kotlin 2.0.20 with plugin `org.jetbrains.kotlin.plugin.compose` 2.0.20
- Compose BOM 2024.09.00
- `com.google.android.gms:play-services-nearby:19.3.0`
- `androidx.lifecycle:lifecycle-viewmodel-compose:2.8.6`
- `androidx.activity:activity-compose:1.9.2`

---

## 5. Implementation details

### 5.1 Manifest permissions

```xml
<uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
<uses-permission android:name="android.permission.BLUETOOTH_SCAN" />
<uses-permission android:name="android.permission.NEARBY_WIFI_DEVICES"
    android:usesPermissionFlags="neverForLocation" />
```

Runtime request at launch: on API 31+ ask for the three Bluetooth permissions; on API 33+ also `NEARBY_WIFI_DEVICES`; on API 30 and below ask for `ACCESS_FINE_LOCATION`. Start the mesh only after all are granted. Show a clear message if denied.

### 5.2 Message model (`MeshMessage.kt`)

```kotlin
data class MeshMessage(
    val id: String,        // UUID, unique per message
    val sender: String,    // device display name
    val text: String,
    val ttl: Int,          // starts at 5, decremented per hop
    val hops: Int,         // starts at 0, incremented per hop
    val timestamp: Long    // System.currentTimeMillis()
)
```
Provide `toJson()` and `fromJson()`. Keep these field names stable so later phases can extend them (type, priority, lat, lng, signature).

### 5.3 MeshManager responsibilities

- Generate a display name once: `"${Build.MODEL}-${random 100..999}"`.
- `start()`: call `startAdvertising` and `startDiscovery` together with `P2P_CLUSTER` and service ID `com.resqmesh.app`.
- **Connection tie-breaker:** in `onEndpointFound`, only call `requestConnection` if `myName < info.endpointName` (string compare) and the endpoint is not already connected or pending. This prevents both phones requesting at once.
- Auto-accept every connection in `onConnectionInitiated`.
- Track connected peers in a `StateFlow<Map<String,String>>` (endpointId to name).
- On `onDisconnected`, remove the peer, then restart discovery after a short delay so the phone reconnects when the other returns.
- **Send:** serialize to JSON, `sendPayload(connectedIds, Payload.fromBytes(...))`.
- **Receive and relay:**
  1. Parse message. If `id` is in the seen set, drop it.
  2. Add to seen set (use a `LinkedHashSet` capped around 2000 entries, evict oldest).
  3. Increment `hops`, emit to the UI message list.
  4. If `ttl - 1 > 0`, forward to all peers except the sender with updated `ttl` and `hops`.
- `stop()`: `stopAdvertising`, `stopDiscovery`, `stopAllEndpoints`.
- Expose a status string (Starting, Advertising, Connected to N, Error: reason).

### 5.4 UI (`MeshScreen.kt`)

- Top: app title, status line, "Connected phones: N" with names.
- Middle: scrolling message list, newest at the bottom. Each bubble shows sender, text, and `hops` ("direct" if 0, "via N hops" otherwise). Own messages are right-aligned.
- Bottom: text field and Send button. Send is disabled when there are zero peers, with a hint "Waiting for nearby phones...".
- Keep the screen awake while the app is open (`FLAG_KEEP_SCREEN_ON`) for Phase 1.

### 5.5 What NOT to build in Phase 1

No database, no GPS, no SOS, no server, no foreground service, no encryption, no map. These come later.

---

## 6. Acceptance criteria for Phase 1

1. `./gradlew assembleDebug` succeeds and creates `app/build/outputs/apk/debug/app-debug.apk`.
2. App installs and launches on Android 8 to 14 phones and asks for the correct permissions.
3. With Bluetooth and Location on, and Wi-Fi/mobile data connected to nothing, two phones show each other in the peers list within about 30 seconds.
4. With 3 phones, a message sent from any phone appears on the other two, exactly once each.
5. A phone that leaves and returns reconnects without restarting the app.
6. No crash when permissions are denied, or when Bluetooth is off (shows a message instead).

---

## 7. Build, install and test

```bash
# build
./gradlew assembleDebug

# install on a USB-connected phone (repeat per phone, or share the APK file)
adb install -r app/build/outputs/apk/debug/app-debug.apk

# watch logs
adb logcat | grep -i resqmesh
```

To install without a cable: send `app-debug.apk` via Drive or a messaging app, then on the phone allow "Install unknown apps" for that source.

**Test checklist:** Bluetooth ON, Location ON, grant all permissions, keep the app in the foreground, keep phones within a few meters for the first test.

**Common failures:**
- Nothing discovered: Location toggle is off, or a permission was denied.
- Advertise error: Google Play Services outdated on that phone.
- Phones connect to each other but not a third: wait 30-60 seconds, and make sure the tie-breaker logic is present.

---

## 8. Roadmap: later phases (one per hackathon round)

| Phase | Name | Deliverable | Key tech |
|---|---|---|---|
| 1 | **Foundation** (this spec) | Auto-connect mesh, chat, relay with dedup and TTL | Nearby Connections, Compose |
| 2 | **Persistence and multi-hop proof** | Messages survive restart; store-and-forward sync on reconnect; UI shows the hop path; test A-B-C with A and C apart | Room (SQLite) |
| 3 | **Disaster features** | SOS button with priority, GPS attached, resource requests (water, medical, trapped), SOS styling and sorting | FusedLocationProvider |
| 4 | **Resilience** | Foreground service so the mesh survives screen lock, battery-friendly duty cycling, reconnect logic | Foreground service type `connectedDevice` |
| 5 | **Internet gateway** | Any phone with internet uploads stored messages to a cloud relay; web dashboard shows SOS pins on a map | Node.js + Express, Leaflet + OpenStreetMap |
| 6 | **Simulator** | Web page with 30-50 virtual nodes, animated hops, kill-node button, clearly labeled "simulated" LoRa long-range link | HTML canvas / JS |
| 7 | **Security and polish** | Message signing to stop fake SOS, spam rate limiting, offline maps, final UI | Ed25519 (Tink), osmdroid |
| Future | **Long range** | ESP32 + LoRa bridge nodes (India: 865-867 MHz), iOS app, satellite or drone relays | Meshtastic or custom firmware |

### Message format growth

Phase 1: `id, sender, text, ttl, hops, timestamp`
Phase 3 adds: `type (CHAT/SOS/RESOURCE), priority, lat, lng`
Phase 7 adds: `signature, publicKey`

---

## 9. Pitch notes

- **Built and working:** phone mesh with relay, dedup and TTL.
- **Simulated (and labeled so):** large-scale behavior and LoRa long-range links.
- **Roadmap:** real LoRa nodes, iOS, encryption.
- **Competitors:** Bridgefy, Briar, Meshtastic. Differentiator is disaster-specific features plus the internet gateway bridge plus the simulator.
- Always keep a recorded backup demo video.
