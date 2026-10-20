<div align="center" dir="rtl">

# PureTV: Minimalist Launcher & Prayer Times

<sub>مشغّل بسيط ومواقيت الصلاة لأجهزة Android TV</sub>

### A faster Android TV with an Arabic prayer-times home screen, built by Claude on a real TV

</div>

<p align="center">
  <img src="docs/screenshots/01-home.png" alt="New home screen: Arabic clock, prayer card, YouTube and Kick on the left, round Play Store and Settings icons on the bottom right" width="100%">
</p>

**PureTV** is a minimalist Android TV launcher with a built-in Arabic prayer-times dashboard. It's a fork of [**Ansub/tv-detox**](https://github.com/Ansub/tv-detox), a tiny open-source Android TV home screen.
I asked Claude to clean up my old Android TV stick, install tv-detox, and then turn it into PureTV by adding:

- a **Prayer Times dashboard for Makkah (مكة المكرمة)** in the Thmanyah font, with a live countdown to the next Adhan
- a **prayer screensaver** that appears when the TV is idle, in any app
- an **automatic Adhan** that plays at each prayer time with a **Stop (إيقاف)** button
- a **Sleep Mode (وضع النوم)** for dark bedrooms: pure black, dimmed, low-contrast
- an **Arabic settings panel**, an **Arabic clock** with Eastern Arabic numerals (١٢:٣٠), round Play Store and Settings icons in the bottom-right corner, and a custom background

PureTV was built over a few sessions on a real TV. Claude ran the commands; I pressed buttons on the remote and said yes or no.

---

## Contents

1. [The device](#the-device)
2. [How Claude was connected to the TV](#how-claude-was-connected-to-the-tv)
3. [Step by step: what was done](#step-by-step-what-was-done)
4. [Screenshots](#screenshots)
5. [What changed in the code](#what-changed-in-the-code)
6. [Build and install it yourself](#build-and-install-it-yourself)
7. [Undo everything](#undo-everything)
8. [Credits and licences](#credits-and-licences)

(See also the [Changelog](#changelog).)

---
 
## The device

| | |
|---|---|
| **Device** | **Xiaomi Mi TV Stick** (model `MiTV-AESP0`, codename `aquaman`) |
| **System** | Android TV 10 (API 29), Amlogic chip |
| **Memory** | 1 GB RAM (982 MB usable), 8 GB storage |
| **Screen** | 1280×720 UI, 1080p video output |
| **HDMI inputs** | none (it's a stick), so the home screen shows no HDMI tiles |

With only 1 GB of RAM, every background app matters, which is why the cleanup came first.

---

## How Claude was connected to the TV

Claude doesn't run on the TV, and it didn't run directly on my network either. It ran in a **cloud session** that was *linked* to my Windows PC through the **Claude desktop app**. The PC sits on the same home network as the TV, so the PC was the bridge:

```
 ┌──────────────────┐   linked computer    ┌─────────────────────────────┐    adb over Wi-Fi    ┌────────────────────────┐
 │  Claude          │  (Claude desktop app │  Windows PC                 │  (network debugging, │  Xiaomi Mi TV Stick    │
 │  cloud session   │ ───────────────────▶ │  • folder access (approved) │ ───────────────────▶ │  MiTV-AESP0            │
 │  writes jobs,    │ ◀─────────────────── │  • TV Detox helper (adb)    │ ◀─────────────────── │  Android TV 10         │
 │  reads results   │   files in & out     │  • Android SDK + JDK        │    TCP port 5555     │  same home network     │
 └──────────────────┘                      └─────────────────────────────┘                      └────────────────────────┘
```

How it worked in practice:

1. **Linking.** I opened the conversation in the Claude desktop app on my PC. That gave the session file access to folders on the PC, but only the ones I approved: the Android SDK folder and a `TV Detox` folder on the Desktop.
2. **The helper.** The session can move files to and from the PC, but it can't open a terminal there. So Claude wrote a small PowerShell helper ([`tools/windows-helper/`](tools/windows-helper)). I double-clicked it once.
   - It watches `job.txt` in its folder and runs **only** the allowed tools: `adb`, `aapt2`, `javac`, `java` and `keytool`.
   - It writes everything they print to `result.txt`.
   - Claude writes a job, the helper runs it, and Claude reads the result.
   - Closing the window stops it.
3. **adb over Wi-Fi.** On the TV I turned on Developer options → USB debugging. The PC connected with `adb connect <TV_IP>:5555`, and I pressed **Allow** on the TV's "Allow debugging?" prompt once.
4. **Building.** The TV app was compiled on the PC with its Android SDK and JDK (aapt2, javac and d8). It was packaged and signed in the cloud session, sent back to the PC and installed with `adb install`.
5. **Seeing the TV.** Claude checked its work with real screenshots (`adb shell screencap`) and memory readings (`/proc/meminfo`, `dumpsys meminfo`).

> Privacy note: this write-up uses `<TV_IP>` as a placeholder. No IP addresses, account names, passwords or keys are included. The signing key (`key.jks`) is git-ignored.

---

## Step by step: what was done

Every change was approved before it ran, apps were **turned off, not uninstalled**, and each one was logged with an undo command.

### 1. Connect to the TV
- Developer options → **USB debugging** on, then `adb connect <TV_IP>:5555`, then **Allow** on the TV.
- Device found: `MiTV-AESP0` (Xiaomi Mi TV Stick), Android 10.

### 2. Inventory, before changing anything
- **Free memory before:** **291 MB** of 982 MB. Storage: 0.9 GB free of 4.8 GB.
- Every package was sorted into three groups:
  - **Apps I use:** YouTube, Kick, Stremio, Downloader, ADB AppControl and others.
  - **Needed by the TV:** Google Play, Gboard keyboard, the remote keyboard bridge, Bluetooth, Mi Bluetooth, system UI, WebView and Xiaomi system services. These were never touched.
  - **Safe to turn off.**

### 3. Cleanup: 24 apps turned off (`pm disable-user --user 0 <pkg>`)

| Group | Packages |
|---|---|
| Google extras | `katniss` (Assistant/voice search, 76 MB), `apps.mediashell` (Chromecast, 27 MB), `backdrop` (Ambient mode), `feedback`, `marvin.talkback`, `tts`, `syncadapters.calendar`, `syncadapters.contacts`, `onetimeinitializer` |
| Xiaomi extras | `mitv.updateservice` (24 MB), `tvsetup.partnercustomizer` (19 MB, sponsored rows), `xiaomi.statistic`, 2× `autoinstalls.config`, `xm.webcontent`, `xiaomo.tv.milegal`, `mitv.videoplayer`, `mitv.download.service` |
| Phone leftovers | `printspooler`, `camera2`, `gallery3d`, `htmlviewer`, `dynsystem`, `dreams.basic` |

Screen mirroring was kept on purpose. **Free memory after the cleanup: 340 MB.**

### 4. Install the tv-detox home screen
- Built from source with **YouTube** and **Kick** as the default apps.
- Ran the same steps as `install.sh`:
  - installed the app;
  - made it the HOME role holder and set it as the home activity;
  - turned off the stock Google TV launcher (`com.google.android.tvlauncher`, 45 MB) so the Home button opens the new screen.
- **Free memory after installing: 361 MB.**

### 5. Prayer Times dashboard (Makkah)
- Prayer times are worked out **on the device**, with no internet, using the **Umm al-Qura** method:
  - Fajr at the sun 18.5° below the horizon;
  - Asr when an object's shadow equals its length;
  - Isha 90 minutes after Maghrib, or 120 minutes in Ramadan.
- On 19 Oct 2026 the results matched the official Umm al-Qura times to within a minute (Asr came out 15:25 against 15:26).
- The full-screen dashboard is in the **Thmanyah** font. It shows all five prayers, highlights the next one in gold with a live **HH:MM:SS countdown**, and shows the Gregorian and Hijri (Umm al-Qura) dates and sunrise. For 20 minutes after each Adhan it shows "حان الآن وقت صلاة …".
- Two ways to open it:
  - **By hand:** a "مواقيت الصلاة" card next to the clock on the home screen. Press OK on it.
  - **Automatically:** as the TV's **screensaver**, after an idle time you set.

### 6. Animations, screensaver and sleep
- Animations were **kept** (my choice).
- The default screensaver was replaced by the prayer screen: `screensaver_components = com.ansub.tvhome/.PrayerDream`, starting after 10 minutes.
- Sleep was set to **never**. You can change both from the TV in **الإعدادات**.

### 7. Restart and measure
- Rebooted. Afterwards the new home screen and the prayer screensaver were still in place.
- Free memory: **291 MB before → 319 MB** 45 seconds after the reboot, while startup services were still settling. It was 340–361 MB in normal use after the cleanup.

### 8. Later updates
- **Settings** became a round **gear** button that opens an Arabic settings panel. **Add app** moved inside that panel.
- **Play Store** and **Settings** moved to the **bottom right** as plain round icons with no card behind them. A soft ring appears only when one is selected. The app tiles stay on the left.
- **Arabic clock:** Thmanyah font, Eastern Arabic numerals, Arabic date.
- **Background:**
  - First a pixel-art GIF. On the stick it used about 40% of the CPU.
  - Then a 1080p xBR-upscaled H.264 video, decoded by the stick's video chip.
  - Finally a **still 1080p wallpaper**, which uses no CPU at all.
- **Automatic Adhan** at every prayer time, with a Stop button.
- **Sleep Mode (وضع النوم)**, a button on the Prayer Times page:
  - the background turns pure black (`#000000`);
  - the window brightness drops to 5% on TVs that support it;
  - every label switches to a muted, low-contrast amber and grey scheme;
  - the layout drifts slightly every minute.
  It's remembered between visits, and the screensaver follows it too. Pressing the button again (**إلغاء وضع النوم**) turns it off.
- **Credits** at the very bottom of the settings panel, directly under the TV settings row: `Dev by XE3Y`, in small, light, faint grey text.

---

## Screenshots

All captured on the real TV with `adb shell screencap`, at 1280×720.

| | |
|---|---|
| **Home screen.** Arabic clock (Thmanyah, ٦:٢٧ ص), Arabic date, next-prayer card, YouTube and Kick on the left, and round Play Store and Settings icons on the bottom right. <br><img src="docs/screenshots/01-home.png" width="100%"> | **Prayer Times dashboard.** All five prayers, the next one in gold, live countdown, Gregorian and Hijri dates, sunrise. <br><img src="docs/screenshots/02-prayer-screen.png" width="100%"> |
| **Settings panel (الإعدادات).** Idle time, sleep time, automatic Adhan, test Adhan, add or remove home-screen apps, Android settings. <br><img src="docs/screenshots/03-settings.png" width="100%"> | **Adhan overlay.** Shown over any app at prayer time, with the Stop (إيقاف) button already selected. <br><img src="docs/screenshots/04-adhan.png" width="100%"> |
| **Prayer screensaver.** The same dashboard runs as the TV's screensaver and drifts slightly every minute to protect the screen. <br><img src="docs/screenshots/05-screensaver.png" width="100%"> | **Sleep Mode (وضع النوم).** Pure black (`#000000`) background, soft dim amber countdown, low-contrast grey times, and a faint outline on the next prayer instead of a gold fill. <br><img src="docs/screenshots/06-sleep-mode.png" width="100%"> |
| **Round shortcuts.** Settings selected: a soft ring and a slight zoom appear only on the selected icon. <br><img src="docs/screenshots/08-home-settings-selected.png" width="100%"> | **Settings panel, bottom.** The TV settings row with the `Dev by XE3Y` credit under it. <br><img src="docs/screenshots/07-settings-credits.png" width="100%"> |
| **Upstream tv-detox**, for comparison (from the original repo). <br><img src="docs/upstream-home-screen.png" width="100%"> | |

---

## Changelog

| Version | What changed |
|---|---|
| 1 | Cleanup (24 apps turned off), tv-detox home screen with YouTube and Kick, stock launcher turned off |
| 2 | Prayer Times dashboard for Makkah (Umm al-Qura), prayer card on the home screen, prayer screensaver, idle and sleep settings |
| 3 | Gear Settings button, Add app moved into Settings, Arabic clock with Eastern Arabic numerals, custom background (GIF, then 1080p video, then a still wallpaper) |
| 4 | Automatic Adhan at every prayer time, with a Stop overlay, a separate Fajr Adhan, and a test button |
| 5 | Sleep Mode (وضع النوم): pure black, dimmed, low-contrast night palette, also used by the screensaver |
| 6 | Play Store and Settings moved to the bottom right as round icons with no cards; apps stay on the left |
| 7 | `Dev by XE3Y` credit at the bottom of the settings panel |
| 8 | Project renamed **PureTV: Minimalist Launcher & Prayer Times**; the app shows as "PureTV" on the TV (package ID unchanged) |

---

## What changed in the code

The app stays what tv-detox is: plain Java, no libraries, no Gradle, no XML layouts. All the UI is built in code.

### New files (`src/com/ansub/tvhome/`)

| File | What it does |
|---|---|
| `PrayerTimes.java` | Pure-math Umm al-Qura calculator for Makkah (21.4225 N, 39.8262 E, UTC+3): solar declination, equation of time, the Fajr/Asr/Maghrib/Isha angles and Isha +90 (+120 in Ramadan). About 1 ms per day. |
| `Prayer.java` | Shared helpers: today's and tomorrow's times, the next prayer, Eastern Arabic digits, 12-hour ص/م formatting, the Hijri date (ICU `islamic-umalqura`), Ramadan detection, loading the Thmanyah font, and reading and writing the idle and sleep timeouts. |
| `PrayerView.java` | The dashboard view: right-to-left layout, five cards, a gold highlight on the next prayer, a countdown that updates every second, and in screensaver mode a small drift each minute to prevent burn-in. `setSleepMode()` swaps in the black, low-contrast night palette and restores the day colours afterwards. |
| `PrayerActivity.java` | Opens the dashboard full screen from the home-screen card. Back closes it; Menu or **الإعدادات** opens the settings panel; **وضع النوم** turns sleep mode on and off (saved in preferences, and window brightness set to 0.05). |
| `PrayerDream.java` | The same dashboard as an Android `DreamService` (screensaver). It follows sleep mode and closes itself when the Adhan starts. |
| `HomeSettings.java` | Arabic side panel with: prayer-screen idle time, TV sleep time, automatic Adhan on/off, test Adhan, add or remove home-screen apps (with icons), and a link to Android settings. A small, muted **Dev by XE3Y** credit sits at the very bottom. |
| `Adhan.java` | `BroadcastReceiver` and scheduler. It sets one exact alarm (`setExactAndAllowWhileIdle`) for the next prayer and sets the following one again whenever an alarm fires, the TV boots, the clock or time zone changes, or the app is updated. |
| `AdhanActivity.java` | Full-screen overlay that plays the Adhan (Fajr has its own file). It takes transient audio focus so YouTube or Kick pauses, shows a focused **إيقاف** button, and stops on OK, Back or Home. It closes by itself when the Adhan ends. The **تجربة الأذان الآن** (test) button in settings opens it straight away, showing the *next* prayer's name. |

### Changed files

| File | Change |
|---|---|
| `MainActivity.java` | Next-prayer card (OK opens the dashboard; hold OK or Menu opens settings) · Arabic clock with Eastern Arabic numerals in Thmanyah · photo background drawn at screen size, with a video fallback and a gradient fallback · Play Store and gear Settings as round icons on the bottom right (`addCircle`), with no card and a ring only when selected; the gear opens the settings panel · **Add app** tile removed (it's in settings now) · idle timer that opens the dashboard when the prayer screensaver isn't the system screensaver · schedules the next Adhan · rebuilds the tiles when the app list changes in settings |
| `LineIcon.java` | New `GEAR` icon: a round gear with 8 teeth, drawn as an outline in code like the other icons |
| `AndroidManifest.xml` | `supportsRtl="true"`, which Arabic layouts need · `PrayerActivity`, `AdhanActivity` (single instance, so finishing returns you to the app you were in) · `PrayerDream` service · `Adhan` receiver (boot, time and time-zone changes, package updates) · permissions: `WRITE_SETTINGS`, `WRITE_SECURE_SETTINGS`, `RECEIVE_BOOT_COMPLETED`, `SYSTEM_ALERT_WINDOW` |
| `AndroidManifest.xml` (name) | App label `PureTV`, version `2.0.0-puretv`. The package ID stays `com.ansub.tvhome`, so updates install over the existing app |
| `apps.txt` | Default top row: YouTube (`com.google.android.youtube.tv`) and Kick (`com.kick.mobile`) |
| `build.sh` | Packs `assets/` with `-A assets` and keeps media files uncompressed (`-0 mp3 -0 otf -0 jpg -0 mp4`) so Android can play them directly from the APK |
| `install.sh` | Grants the optional permissions: write settings (so the TV can change its own sleep time) and *display over other apps* (so the Adhan can appear over YouTube on Android 10+) |

### Lessons learned on a real TV
- **Right-to-left needs `android:supportsRtl="true"`.** Without it, Android forces left-to-right, and the Arabic layout came out mirrored and jumbled.
- **Arabic fonts are taller than Latin ones.** Turning off `setIncludeFontPadding` and tightening the sizes kept the cards on a 720p screen.
- **Android 10 blocks background apps from opening screens.** The Adhan overlay only works over other apps with the *display over other apps* app-op.
- **An animated GIF background is expensive on a 1 GB stick** (about 40% CPU). An H.264 video moves the work to the video chip. A still wallpaper costs nothing.
- **Arabic TextViews often measure to the full width of their column.** Centred labels need an explicit `Gravity.CENTER`.

### Resource use (measured on the stick)

| | Home screen app memory | CPU on the home screen |
|---|---|---|
| tv-detox + prayer (gradient) | ~19 MB | ~0% |
| with GIF background | ~29 MB | ~40% (app) |
| with 1080p video background | ~17 MB | ~0% app, ~36% video system |
| with still wallpaper (current) | ~20 MB | ~0% |

---

## Build and install it yourself

1. **Add the assets you're allowed to use.** See [`assets/README.md`](assets/README.md) for the Thmanyah font, your Adhan recordings and your wallpaper. They aren't in the repo because of their licences.
2. **Build** (needs Java 17+ and the Android SDK, platform 34 and build-tools 34):
   ```sh
   sh build.sh
   ```
3. **Turn on network debugging on the TV**, then install:
   ```sh
   sh install.sh <TV_IP> "com.google.android.youtube.tv,com.kick.mobile"
   ```
4. Optional, to make the prayer screen the TV's screensaver (here after 10 minutes, with sleep set to never):
   ```sh
   adb -s <TV_IP>:5555 shell settings put secure screensaver_enabled 1
   adb -s <TV_IP>:5555 shell settings put secure screensaver_components com.ansub.tvhome/.PrayerDream
   adb -s <TV_IP>:5555 shell settings put system screen_off_timeout 600000
   adb -s <TV_IP>:5555 shell settings put secure sleep_timeout -1
   ```

On Windows without a shell, [`tools/windows-helper`](tools/windows-helper) is the small runner Claude used. It runs only adb and the SDK/JDK build tools listed in `job.txt`.

---

## Undo everything

```sh
# home screen: turns the stock launcher back on and removes this app
sh undo.sh <TV_IP>

# cleanup: turn any app back on
adb -s <TV_IP>:5555 shell pm enable <package>

# permissions
adb -s <TV_IP>:5555 shell appops set com.ansub.tvhome SYSTEM_ALERT_WINDOW default
adb -s <TV_IP>:5555 shell appops set com.ansub.tvhome WRITE_SETTINGS default
adb -s <TV_IP>:5555 shell pm revoke com.ansub.tvhome android.permission.WRITE_SECURE_SETTINGS
```

---

## Credits and licences

- **tv-detox** by [Ansub Khan](https://github.com/Ansub/tv-detox), MIT licence, kept in [`LICENSE`](LICENSE). The original README is in [`docs/UPSTREAM-README.md`](docs/UPSTREAM-README.md).
- **Thmanyah font** (خط ثمانية) by Thmanyah. It's free to use in apps, but redistributing or hosting the font file isn't allowed, so it's **not included**. Get it from <https://font.thmanyah.com/>. Licence: <https://font.thmanyah.com/licenses>. This project isn't affiliated with or endorsed by Thmanyah.
- **Adhan recordings:** not included. The build shown used recordings from [abodehq/Athan-MP3](https://github.com/abodehq/Athan-MP3) (Ibrahim Al-Arkani, and Malek Chebae for Fajr).
- **Prayer-time method:** Umm al-Qura University, Makkah. Results were checked against the [Aladhan API](https://aladhan.com) (method 4).
- **Developed by XE3Y**, with Claude (Anthropic) working through the Claude desktop app's linked-computer file access and adb.

PureTV's changes are released under the same MIT licence as upstream. "PureTV" is this fork's project name and isn't affiliated with tv-detox's author.
