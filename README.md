# Avinox Ride — Privacy & Security Review

A static (and partial dynamic) security and privacy review of the **Avinox Ride** Android app,
conducted before installation. Avinox Ride is the companion app for the **Avinox drive system**,
an e‑bike mid‑drive platform developed by DJI / SZ Avinox Innovation Co., Ltd.

> **Scope & intent.** This is an independent, good‑faith review of a publicly distributed app,
> performed on a copy obtained legitimately, for the reviewer's own install decision. It documents
> *capabilities present in the code* and *gaps between the app's behaviour and its published legal
> terms*. It does **not** claim that the vendor actively exfiltrates any particular data, and it
> contains no exploit code. Findings are based on decompilation and string analysis and may be
> incomplete or contain misreadings. Corrections welcome via issues.

**Written for:** security‑ and privacy‑conscious readers evaluating the Avinox Ride app / drive system.

---

## In plain English (if you're not technical)

**What is this?** Avinox Ride is the phone app that pairs with an Avinox‑powered e‑bike (the motor
system is made by DJI, the drone company). This write‑up looks at what the app *can* do on your phone
and bike, before deciding whether to install it. Think of it as reading the ingredients label.

**Is it safe to install?** It looks like a genuine, properly‑built app from the real manufacturer —
not a fake or obvious malware. It even has tamper protection that stops modified copies from running.
So the basics are fine. The concerns below are about **privacy and control**, not a virus.

**The main things worth knowing:**

- **It carries "extra passengers."** Mixed into the app are several advertising/notification
  components made by Chinese tech companies (the biggest is one called *Getui*). Their code is
  *capable* of gathering a lot about your phone — the list of other apps you have installed, nearby
  Wi‑Fi networks, your location, and device ID numbers. We can't prove from the outside how much it
  actually sends, and modern Android blocks a chunk of it, but the capability is in there, and the
  company's own privacy policy never mentions these components by name.

- **Your bike has a "remote off switch."** If your bike has a SIM card in it (like a phone's SIM),
  the motor can be locked or shut off remotely over the mobile network. This is mainly an *anti‑theft*
  feature you control — if your bike is stolen, you can "mark it as lost" and the motor stops helping
  anyone else, which is genuinely useful. The flip side: the manufacturer's servers technically *could*
  switch off your motor too, and if someone broke into your account they could lock your bike. It
  doesn't make the bike undrivable — you can still pedal it like a normal (heavy) bike — it just stops
  the electric assist.

- **You can unplug the "tracking."** That SIM card is one you insert yourself, and there's an off
  switch in the app. **Take the SIM out (or leave it out) and the remote tracking and remote off‑switch
  stop working** — the app still runs your bike fine over Bluetooth when your phone is nearby. The only
  thing you lose is remote features like auto‑uploading rides and finding the bike if it's stolen. It's
  a trade‑off between privacy and theft‑recovery, and it's your choice.

- **The fine print doesn't match the app.** The company's Privacy Policy and Terms don't mention the
  SIM, the remote off‑switch, the anti‑theft tracking, or those Chinese data components at all. They're
  also a Shenzhen (China) company, and the Terms cap what they'd ever owe you at **\$100**. None of that
  is unusual for this kind of product, but you should go in knowing it.

**What should a normal person do?** You can use the app — just be a little deliberate:
1. Install it from the official Google Play Store (or Apple's App Store), not from random websites.
2. When it asks for permissions, allow Location "only while using the app," allow Bluetooth, and
   **say no to the "Phone" permission** and to any "help us improve the product" / analytics prompts.
3. Decide whether you want the bike's SIM in at all. Leave it out if you don't care about remote
   tracking; put it in if theft‑recovery matters more to you than privacy.
4. Protect the email/phone number you use for your Avinox account — that account is the "key" to the
   bike's lock.

The rest of this document is the technical evidence behind these points.

---

## App under review

| | |
|---|---|
| Package | `com.avinox.ride` |
| Version | 1.4.3 (versionCode 6396) |
| Distribution analysed | APKPure XAPK bundle (base APK + `arm64_v8a` + `xxhdpi` splits) |
| minSdk / targetSdk | 24 / 35 |
| Framework | Flutter (Dart AOT) over the DJI mobile SDK |
| Signing org | **SZ Avinox Innovation Co., Ltd** (Shenzhen, CN); cert CN `jiewen.xu` |
| Signing cert SHA‑256 | `09:8A:86:FF:AB:5E:F1:CB:AE:B0:A4:EC:FE:6D:5A:40:FC:7C:59:E9:20:4F:F4:69:74:83:4C:DD:70:89:97:61` |
| Tamper protection | Google Play Integrity / `com.pairip` licensing (license‑check variant) |

> **Verify before trusting any build.** If you install from the Play Store, you can compare the
> installed signing certificate against the SHA‑256 above. The DRM wrapper also means the app
> **refuses to run when sideloaded outside Play** (observed: it force‑closes after a license‑check
> failure) — a mild positive for tamper resistance.

---

## Summary of findings

| # | Area | Severity | One‑line |
|---|---|---|---|
| 1 | Third‑party telemetry SDKs | **Medium–High** | Bundled Chinese push/ID SDKs (Getui, ZXID) contain code to collect installed‑app lists, Wi‑Fi/location, and hardware identifiers. |
| 2 | Remote immobilization | **Medium** | The drive unit can be remotely locked / deactivated under server‑granted authority over the bike's own SIM. |
| 3 | Legal‑disclosure gaps | **Medium** | Neither the Privacy Policy nor Terms disclose the SIM/cellular link, remote lock, anti‑theft, or the named third‑party SDKs. |
| 4 | Cleartext traffic | **Low–Medium** | `usesCleartextTraffic=true`; Chinese map tiles load over plain HTTP. |
| 5 | Account‑takeover blast radius | **Low–Medium** | The account controls lock/unlock + mark‑stolen + component unbind; login is OTP‑only (no app‑level 2FA/biometric observed). |
| 6 | Exported components | **Low** | A few push‑SDK components are launchable by other apps without a custom permission. |
| 7 | Hardcoded client keys | **Info** | Firebase / MapTiler client keys are embedded (normal for client apps). |

Positive signals: Firebase Analytics & Crashlytics are **disabled by default** in the manifest; no
`ACCESS_BACKGROUND_LOCATION`; no contacts/SMS/call‑log/accounts permissions; no `QUERY_ALL_PACKAGES`;
the app's own first‑party code is thin and does no independent data collection.

---

## 1. Permissions

Declared permissions of note (full list below). The app does **not** request background location,
contacts, SMS, call log, accounts, or "see all installed apps" (`QUERY_ALL_PACKAGES`).

```
Location:   ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION, ACCESS_MEDIA_LOCATION,
            ACCESS_LOCATION_EXTRA_COMMANDS
Bluetooth:  BLUETOOTH, BLUETOOTH_ADMIN, BLUETOOTH_SCAN, BLUETOOTH_CONNECT, BLUETOOTH_ADVERTISE
Network:    INTERNET, ACCESS_NETWORK_STATE, ACCESS_WIFI_STATE, CHANGE_WIFI_STATE
Device:     READ_PHONE_STATE, CAMERA, VIBRATE, WAKE_LOCK
System:     SYSTEM_ALERT_WINDOW, REQUEST_IGNORE_BATTERY_OPTIMIZATIONS, SCHEDULE_EXACT_ALARM,
            RECEIVE_BOOT_COMPLETED, FOREGROUND_SERVICE, POST_NOTIFICATIONS, GET_TASKS
Storage:    READ/WRITE_EXTERNAL_STORAGE
Push (OEM): Xiaomi / Huawei / Oppo / Vivo / Getui push permissions
```

`READ_PHONE_STATE` + `ACCESS_WIFI_STATE` are the permissions the bundled SDKs use to reach device
and network identifiers; `SYSTEM_ALERT_WINDOW` and `GET_TASKS` are typical of Chinese push SDKs.

---

## 2. Third‑party SDKs & data‑collection capability

The riding/telemetry logic is Flutter + the native DJI SDK. Alongside it, the app bundles a stack of
**Chinese push‑notification and device‑ID SDKs**:

- **Getui / 个推** (`com.getui`, `com.igexin`) — push, plus a device‑info module (`com.getui.gtc.dim`).
- **ZXID** (`com.zx`, endpoint `zxid-m.mobileservice.cn`) — a device/one‑ID SDK loaded via Getui.
- **Xiaomi MiPush, Huawei HMS Push, Oppo, Vivo** push SDKs (dormant on non‑Chinese phones).

The Getui `dim` module and ZXID contain code paths to collect (identifiers in parentheses are the
Base64‑obscured keys found in the binary):

- **Installed apps** — five strategies incl. running `pm list packages` in a shell
  (`app_list_by_installed_package`, `app_list_by_pm_list`, `app_list_by_query_intent`,
  `app_list_by_local_dirs`, `app_list_by_complex_policy`).
- **Wi‑Fi** — current SSID/BSSID and **scan lists** of nearby networks (`wifi_info`, `wifi_scan_list`).
- **Local network neighbours** — via `ip neighbour`.
- **Location** — GPS + network location and cell‑tower info (`location_gps`, `cell_info`).
- **Hardware identifiers** — IMEI/IMSI/ICCID, hardware serial, Wi‑Fi MAC, Android ID, advertising IDs
  (`imei_main`, `imsi_main`, `iccid_main`, `serial_number`, `mac_wlan0`, `android_id`, `advertising_id`,
  plus vendor OAIDs).
- **Root/hook detection** — checks for Xposed / Substrate.

Getui also uses a **`DexClassLoader` whose class name is Base64‑hidden** (`SdkLoader`) and can pull
plugin code (`gdi.stub.PushExtension`) at runtime — so static analysis cannot fully bound its behaviour.

**Important mitigating context:**
- Modern Android blocks much of this: IMEI/IMSI/serial and real MAC are unavailable to apps on
  Android 10+, `ip neighbour` is blocked on 11+, and installed‑app visibility is restricted from 11+.
- **Location + Wi‑Fi scan still work** once location permission is granted (which the app needs for
  Bluetooth pairing anyway), so those remain reachable.
- Getui initialises from Dart via an `initGetuiPush` channel call, and the app has a consent gate
  (`isAgreePrivacy`) and a China‑mainland / other‑region switch. **Whether Getui strictly waits for
  consent, and whether it is confined to mainland users, could not be confirmed statically** — it
  would require a live capture on a licensed install (blocked here by the Play DRM).

---

## 3. Network endpoints, cleartext & region

`android:usesCleartextTraffic="true"` is set app‑wide and there is **no network security config**.
Production first‑party APIs use HTTPS, but some map tiles load over plain **HTTP**.

Representative first‑party / notable hosts:

```
api.avinox-ebike.com            first-party API (prod)   HTTPS
stag-api / pre-prod-api .*       staging/pre-prod (build-flag gated)
account.avinox-ebike.com         account
dl.avinox-ebike.com              AWS S3/CloudFront — user manuals (listing denied)
fm.avinox-ebike.com              firmware config API
firmware-dl.djicorp.com          DJI firmware CDN
*.djiservice.org / dji.com       DJI analytics & services (statistical-report.djiservice.org)
zxid-m.mobileservice.cn          ZXID device-ID service
*.getui.com / *.getui.net / *.gepush.com   Getui push
*.xiaomi.net / resolver.msg.xiaomi.com     MiPush
webrd/wprd*.is.autonavi.com      AutoNavi map tiles (HTTP)
```

A region setting ("Chinese Mainland" vs other) switches the API server; the app enforces a
`regionDomainList` allow‑list for the Avinox host.

---

## 4. Firmware distribution

Firmware update logic lives **entirely in the native DJI library** `libDJIUpgradeCore.so` — the app
is a thin UI over DJI's device‑upgrade SDK (the same stack used for DJI drones; symbols reference
`drone_sn`, `glass_sn`, `rc_sn`, geofence, etc.). The flow:

1. A config API (`fm.avinox-ebike.com`, path `loadconfig/geturl/a`, and DJI's
   `firmware-dl.djicorp.com/getfile/getallfile`) is queried with the bike serial, product id,
   current firmware versions, country, a token and a **request signature**
   (`GetServerCfgRequest` → `GetCfgVersionInfosRequest` → `GetCfgModuleDownloadUrlRequest`).
2. It returns a **per‑request, signed download URL**.
3. Files arrive as a signed config (`.cfg.sig`) plus DJI‑format module blobs (`dji_system.bin`
   packed in `tmpfirm.tar`), verified by **MD5 + config signature** before flashing.

Obtaining a firmware image requires a valid, account‑bound bike serial driving that authenticated,
signed API — i.e. it is not publicly downloadable. The public `dl.avinox-ebike.com` bucket serves
only user manuals and **denies listing** (`AccessDenied`).

---

## 5. Anti‑theft & remote immobilization ("kill switch")

The drive system includes a layered anti‑theft / immobilizer, built on DJI's SDK
(`dji::sdk::DeactivateMgr`, handlers `DeActivateHandlerWM260` (motor), `…PM320` (battery),
tamper‑resistant `RPMB` secure storage). Three layers:

1. **Mobile Key Lock (local).** Bluetooth proximity lock/unlock — *"As you bring paired mobile
   device close to bike, power on bike to automatically unlock."* State key `keyDeviceLockEnable`.
   Convenience only; not a theft control.
2. **Mark‑as‑lost / component lock (the immobilizer).** Endpoints
   `/api/v2/users/anti-theft/components/{bind,challenge,verify,report-lost,cancel-lost,unbind}`.
   Each major component is **cryptographically bound** to the owner account via challenge/response.
   App text: *"Open app to mark device as lost. After marking, if device account is different from
   original bound account, motor assistance will not be available."* Components can become
   **permanently locked** — *"Contact after‑sales support or your local dealer for assistance"* — and
   cannot be re‑bound to another account, killing the resale value of stolen parts. Enforced in
   component firmware, not merely a phone‑side flag.
3. **Deactivate (server‑authenticated).** `ffi_ActionDeActivateRequest` logs in to a server
   (`LoginServer` / `PreLoginServer`) and **`DownloadDeactivateDataFromServer`** per component — i.e.
   the **authority to lock/deactivate is granted by the backend**, not computed on the phone.

**Why this matters.** The bike carries its **own SIM** and supports **Cloud Remote Control**
(`REMOTECONTROLLER_AUTHORITY_DEVICE_LOCKED`, `setValueForNet keyDeviceLockEnable`). Combined with
server‑granted deactivate authority, **the vendor backend has the technical means to lock/immobilize
the drive unit remotely over cellular**, independent of the owner's phone. In normal use these
actions are owner‑initiated and are a legitimate anti‑theft feature (the app notes it *"cannot
replace a physical lock"*). As a risk, the same capability means (a) the vendor can disable motor
assist by policy or error; (b) a compromised account could lock your bike; (c) a backend breach
would expose that control. It is a **soft‑brick** (no motor assist; alarm sounds; location tracked),
not a hard brick — the bike still rolls as a heavy manual bike.

---

## 6. Connectivity is owner‑controllable (removable SIM)

The bike's cellular link is **optional and user‑controlled**:

- It uses a **physical, removable SIM you insert** (no evidence of a soldered/embedded eSIM):
  *"Insert SIM card into bike, then … go to Cloud to enable Bike Connectivity (SIM)."*
- There is an explicit in‑app **"Disable Bike Connectivity (SIM)?"** toggle.
- **Without a SIM / with connectivity disabled**, the bike still fully works and the app still controls
  it **over Bluetooth**. You lose only: 4G ride‑data cloud sync, cloud/Find‑My location tracking over
  cellular, and **Cloud Remote Control**.

Trade‑off: the server‑pushed remote lock and over‑cellular tracking only function with an enabled SIM.
Removing the SIM cuts the remote/tracking channel while keeping the local protections (cryptographic
component bind, BLE proximity lock, onboard alarm).

---

## 7. Privacy Policy & Terms — gap analysis

The **Privacy Policy** (SZ Avinox Innovation Co., Ltd., updated 2025‑04‑16) and the **Terms of Use**
are average‑to‑reasonable boilerplate. The concern is the **delta** between them and observed app
behaviour.

| Observed in the app / bike | Legal‑document disclosure |
|---|---|
| Chinese push/ID SDKs (Getui, ZXID, Xiaomi/Huawei/Oppo/Vivo) | **Not named** — only "third party service providers to provide application development" |
| Code to read hardware identifiers (IMEI/IMSI/ICCID/serial/MAC/Android ID/ad IDs) | **Not mentioned** — only "mobile device information such as operating system specifications" |
| Installed‑app enumeration, Wi‑Fi/cell scanning | **Not mentioned** |
| Bike SIM/4G, Cloud Remote Control, remote lock/deactivate, mark‑stolen | **Not mentioned in Privacy Policy *or* Terms** |
| Location tied to anti‑theft alarm, Find My, cloud control | Framed as ride‑sync only; collection frequency unstated |
| Shenzhen operator; traffic to Chinese services | Says data goes "globally"/"outside the EEA"; **China never named**; SCCs "on request" |
| Retention | No concrete periods ("only for the period necessary") |

**Terms highlights:** Services are *"AS IS"*; *"we do not guarantee the Services will be permanently
provided"*; Avinox *"is entitled to suspend or terminate your Service"*; aggregate liability **capped
at \$100**. Governing law defaults to California (England & Wales for EU, Victoria for AU); operating
entity is **SZ Avinox Innovation Co., Ltd., Shenzhen**. The EU Data Act terms allow Avinox to *"use
and share with third parties any non‑personal Available Data."*

**Two genuine disclosure failures:** (1) a server‑operated capability to disable your motor and track
your bike over its own SIM is **documented nowhere** in the legal pages; (2) the third‑party data
collection is **materially under‑described** and names none of the Chinese partners or the transfer to
China — even though much of that collection is dormant outside China or blocked by modern Android.

---

## Recommendations (for a cautious installer)

1. **Install from the Play Store**, not a sideloaded XAPK (the DRM blocks sideloads anyway, and Play
   gives updates). After install, verify the signing SHA‑256 above.
2. **Permissions:** Location "while using only"; allow Nearby devices (Bluetooth); **deny Phone**
   (`READ_PHONE_STATE`); grant Camera only for QR scanning.
3. **In‑app:** decline the Product Improvement Program and analytics; choose a non‑China region; leave
   cloud sync off unless you want the remote anti‑theft/alerts.
4. **Connectivity:** treat the bike SIM as opt‑in — insert/enable it only if you specifically want
   remote tracking/anti‑theft; otherwise leave it out and operate over Bluetooth.
5. **Account hygiene:** the account controls the immobilizer — protect the email/phone it uses
   (SIM‑swap and mailbox compromise are the realistic takeover paths; no app‑level 2FA/biometric was
   observed beyond OTP).
6. **Network:** optionally monitor/block `*.getui.com`, `*.getui.net`, `*.gepush.com`,
   `*.mobileservice.cn`, and `statistical-report.djiservice.org` at the DNS level and confirm pairing
   still works.
7. Consider a **work profile** (e.g. Shelter/Insular) to limit the push SDKs' view of your other apps.

---

## Methodology & tools

- Unpacked the XAPK; decoded resources/manifest with **apktool**; decompiled Java/Kotlin with **jadx**.
- Decompiled the Flutter/Dart AOT snapshot with **blutter**; analysed strings from all 21 native
  libraries.
- Extracted and inspected the v2/v3 signing certificate directly from the APK Signing Block.
- Attempted a dynamic run on an Android 13 emulator with **mitmproxy** (system‑trusted CA); the app's
  Play‑Integrity/`pairip` DRM force‑closed the sideloaded build, so live traffic capture of the app's
  own endpoints was not possible. Observations here are therefore **static** except where noted.

### Integrity of the analysed sample

```
XAPK  SHA-256  0180babd3ff88fd67b0f3f1fd268c5ea256b4979e358875a0e9aa9eb3cef9000
base  APK      044e7395380022d26e78b64bde1206670d9c586eb7c62a287e8d2909a16c1061  com.avinox.ride.apk
abi   split    41656da260f314dafeb3274285484129afac7a6c29a6ad0abf736a23e7c3072e  config.arm64_v8a.apk
dpi   split    47811f81ebec5334b4830e7f2c9d03a7ae638723e008d146058e5e93b1f69935  config.xxhdpi.apk
```

---

*Prepared as an independent static review. No affiliation with Avinox or DJI. Not legal advice.*
