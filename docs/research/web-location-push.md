# Location and push notifications in a plain mobile web app

Research for issue #3. Question: in a plain mobile browser with nothing installed, how reliable is the Geolocation API inside an ice rink, what do the permission prompts look like, and how easy is it to fake a location? And for web push: what works without installing on Android and desktop, and what does iOS require?

Researched 2026-10-11. Each claim names its source. Where no measurement exists, the figure is marked **(estimate)**.

## TL;DR

- **Location indoors works, but it is coarse.** Inside a large building, GNSS (satellite positioning) mostly fails. The fix comes from Wi-Fi, cell towers and Bluetooth instead. Expect a reported `accuracy` of about **10–50 m** in a well-mapped building, and **hundreds of metres to kilometres** when the phone falls back to cell towers or the user picked "approximate" location. A fix usually arrives in a few seconds. Waiting for a GPS-quality fix indoors can take much longer or never happen.
- **Treat `accuracy` as roughly a 1-sigma radius, not a guarantee.** The W3C spec says 95% confidence. The native platforms underneath browsers document their accuracy values at about 68%.
- **There are two permission layers on both platforms:** the browser's per-site prompt and the OS's per-app location setting (with a precise/approximate choice). Either one can quietly ruin the result. iOS Safari re-prompts often; the most it remembers a grant for is one day.
- **Faking a location is trivial for anyone motivated.** The server only ever receives two numbers the client chose to send. DevTools, extensions, mock-location apps, or a plain `curl` POST are all enough. The web API has no "mock" flag. Use geolocation as a **low-friction filter, not proof**.
- **Web push without installing works on Android (Chrome, Firefox, Samsung Internet) and on desktop (Chrome, Edge, Firefox, and Safari on macOS 13+).** **On iOS, web push requires iOS/iPadOS 16.4+ and a Home Screen web app.** The permission request must come from a user tap inside that installed app. A Safari tab gets nothing.

---

## 1. Geolocation API: what the spec promises

Source: [W3C Geolocation, CR Snapshot 26 March 2026](https://www.w3.org/TR/geolocation/), [MDN `GeolocationCoordinates.accuracy`](https://developer.mozilla.org/en-US/docs/Web/API/GeolocationCoordinates/accuracy).

- **Secure context only.** Calls from non-HTTPS pages fail with `PERMISSION_DENIED` (code 1). Local development over plain HTTP on a phone will not work, so use HTTPS or a tunnel.
- **Permissions Policy.** The default allowlist is `'self'`, so cross-origin iframes need `allow="geolocation"`.
- **`accuracy`** is "the accuracy value indicating the 95% confidence level in meters", a radius around lat/long.
- **`enableHighAccuracy`** is only a *hint*. The user agent may ignore it, and it "may result in slower response times or increased power consumption".
- **`timeout`** counts from when acquisition starts. Time spent waiting for the permission prompt is excluded. The default is `Infinity`, so **always set a timeout**.
- **`maximumAge`** lets the browser return a cached position. Set `0` for the vote check so a position from the user's home isn't reused.
- **Errors:** `PERMISSION_DENIED` (1), `POSITION_UNAVAILABLE` (2), `TIMEOUT` (3).
- The spec itself states that "no guarantee is given that the API returns the device's actual location".

### The `accuracy` semantics caveat

Browsers on phones take the platform's location estimate:

- Chrome on Android uses Google Location Services/Fused Location Provider ([Chrome Help](https://support.google.com/chrome/answer/142065?hl=en&co=GENIE.Platform%3DAndroid)).
- Safari uses iOS Core Location.

The native definitions differ from the W3C's 95%:

- Android `Location.getAccuracy()` is documented as the horizontal radius at **68% confidence** ([Android `Location` reference](https://developer.android.com/reference/android/location/Location)).
- Apple's `CLLocation.horizontalAccuracy` is described in Apple developer forum answers as a 2D RMS error, roughly a 63–68% circle ([Apple Developer Forums thread 699996](https://developer.apple.com/forums/thread/699996); that is a forum post, not the official doc page).

**Inference (not verified in browser source):** the number a web page receives is probably closer to a 1-sigma radius than to 95%. So the true position will fairly often lie *outside* the reported circle. Don't treat a tight accuracy circle near the boundary as conclusive in either direction.

## 2. How accurate is it indoors in an ice rink?

### How the phone locates itself

Apple says Location Services uses "GPS and Bluetooth (where they're available)" plus "crowd-sourced Wi-Fi hotspots and cellular towers" ([Apple Support 102647](https://support.apple.com/en-us/102647), [Apple Location Services & Privacy](https://www.apple.com/legal/privacy/data/en/location-services/)). Android's fused provider works the same way. Under a concrete and steel arena roof, satellite signals are largely blocked. The fix then depends on **how many Wi-Fi access points the platform's crowd-sourced database knows nearby**. Arenas with public or venue Wi-Fi and dense surroundings should do better than an isolated rink.

### Measurements found

| Source | Setting | Figure |
|---|---|---|
| Palathinkal et al., [*Evaluating Smartphone GNSS Accuracy for Geofenced 6 GHz Operations*](https://arxiv.org/html/2603.26706v1), arXiv 2603.26706 (2026). Android, Fused Location Provider, reported accuracy (68% radius) | Hotel interior, Las Vegas, walking | median reported accuracy **41.7 m** (Galaxy S22), **28.8 m** (S24), **47.9 m** (Pixel 10 Pro) |
| same | University building, ground floor (Notre Dame) | median **10.3 m** indoors vs **4.1 m** outdoors 15 m away |
| same | University building (Iowa State) | median **16.4 m** indoors vs **3.9 m** outdoors |
| Zandbergen, *Accuracy of iPhone Locations: A Comparison of Assisted GPS, WiFi and Cellular Positioning*, Transactions in GIS 13(s1), 2009 ([abstract](https://proceedings.esri.com/library/userconf/proc09/uc/abstracts/a1870.html)). Old iPhone 3G, *true* error vs ground truth | A-GPS / Wi-Fi / cellular, by positioning mode | median error **8 m** / **74 m** / **600 m** |
| [Android location permissions doc](https://developer.android.com/develop/sensors-and-location/location/permissions) | Platform statement | precise: "usually within about 50 meters … sometimes … a few meters"; **approximate: "within about 3 square kilometers"** |

Caveats:

- I found **no published measurement taken inside an ice rink or arena with a browser.** The Palathinkal figures are the platform's *self-reported* uncertainty, not error against ground truth.
- The Zandbergen numbers are 2008 hardware. They show the order of magnitude of each fallback mode, not current performance.

### Estimates for a Swiss rink

These are **(estimates)** combining the sources above. They need a field test.

- Phone with precise location on, venue Wi-Fi present or a well-mapped urban site: reported `accuracy` **~10–50 m**. Position may drift a few tens of metres between seats.
- Sparse Wi-Fi, so the phone leans on cell towers: **~100 m to 1 km+**.
- User chose **approximate** location (iOS "Precise Location" off, or Android "Approximate"): **km scale**. Android documents about 3 km². Apple doesn't publish a figure. **(estimate: several km.)**
- **Time to fix (estimate):** a Wi-Fi/cell-based fix usually returns in **~1–5 s** after permission is granted. With `enableHighAccuracy: true` the browser may keep waiting for satellites that never lock indoors. In practice, set a timeout of ~10–15 s and accept the best fix seen so far, using `watchPosition` and stopping after a few readings. I found no primary measurement of browser time-to-first-fix indoors.

A Swiss National League arena is roughly 100–150 m across, and its car parks and forecourt extend beyond that. A typical indoor fix is therefore comparable in size to the building itself.

## 3. Permission prompts

### iOS Safari (and every iOS browser, since all use WebKit)

Two layers:

1. **OS layer:** Settings → Privacy & Security → Location Services → *Safari Websites*: Never / Ask Next Time / While Using the App, plus a **Precise Location** toggle ([Apple Support 102647](https://support.apple.com/en-us/102647) covers Precise Location generally, from iOS 14). If this is set to **Never**, every site is refused. If Precise is off, sites get only an approximate position.
2. **Site layer:** Safari shows a modal naming the site ("*example.com* would like to use your current location") with **Allow / Don't Allow**.
   - Grants are short-lived. Users report a "remember for one day" behaviour ([Apple Community 255005639](https://discussions.apple.com/thread/255005639)).
   - WebKit maintainers confirmed that after repeated allows, iOS grants location for **24 hours** without prompting ([WebKit bug 275479](https://bugs.webkit.org/show_bug.cgi?id=275479), dup of 293089). Before that fix, `navigator.permissions.query({name:'geolocation'})` kept reporting `prompt` even during an active grant, so **don't rely on the Permissions API on iOS to decide whether to show your pre-prompt UI**.
   - Expect most fans to see the prompt **once per game day**.

### Android Chrome

1. **Site layer:** "By default, Chrome asks you when a site wants to use your location". It shows a dialog naming the origin; the user allows or blocks. Per-site settings are under Settings → Site settings → Location ([Chrome Help 142065](https://support.google.com/chrome/answer/142065?hl=en&co=GENIE.Platform%3DAndroid)). The exact button labels vary by Chrome version.
2. **OS layer:** Chrome is an Android app, so it needs Android location permission itself. Since Android 12 the user can grant it **approximate** instead of precise. If they do, every site gets about 3 km² accuracy "regardless of which location permissions your app declares" ([Android location permissions](https://developer.android.com/develop/sensors-and-location/location/permissions)). If device location is switched off, Chrome asks the user to turn it on.
3. Chrome's "Allow this time" one-time grants (Chrome 116+) are **desktop only**. A one-time geolocation grant also pauses as soon as the tab is backgrounded ([Chrome blog: One-time permissions](https://developer.chrome.com/blog/one-time-permissions)).

### Practical prompt UX

- Never request location on page load. Show your own explainer first ("We check you're at the rink. Your location is used once and not stored"), then call `getCurrentPosition` from the button tap. On desktop Chrome, a request not tied to a user gesture is downgraded to a quiet chip ([Chrome blog: permissions chip](https://developer.chrome.com/blog/permissions-chip)).
- When the user taps Block / Don't Allow, the browser won't ask again by itself. The page has to tell the user how to re-enable location in settings, with separate instructions for iOS and Android.

## 4. How easily can location be faked?

Very easily. The server receives a latitude/longitude pair the client chose to send.

- **Desktop Chrome DevTools:** Sensors panel → Geolocation → any preset or custom coordinates ([Chrome DevTools: Sensors](https://developer.chrome.com/docs/devtools/sensors)). The phone layout can be emulated in the same tool.
- **Android:** Developer options → "Select mock location app", then install any GPS mock app ([Android developer options](https://developer.android.com/studio/debug/dev-options)). Native Android apps can detect mocks (`Location.isMock()`), but `GeolocationCoordinates` exposes **no such flag** to the web ([W3C spec](https://www.w3.org/TR/geolocation/)).
- **iOS:** harder on the device itself without a computer, but Xcode/developer-mode location simulation and paid desktop spoofing tools exist. **(Not verified against Apple docs here.)**
- **No browser needed at all:** replay the vote request with `curl` and arbitrary coordinates. Client-side checks only matter if the server repeats them.
- **Social fake:** a fan at home asks a friend in the rink to vote for them, or a remote fan tethers to the friend's hotspot.

**Conclusion:** geolocation stops casual remote voting (the fan on the sofa clicking a link) and nothing more. Make the rule and its copy honest about that.

## 5. Implications for the rink-presence rule

Suggested starting point. All numbers are to be tuned after a field test.

1. **Geofence:** store per venue a centre point plus a radius sized to the building footprint plus margin, **~150–250 m** for a typical rink. A polygon is better if the venue sits next to housing.
2. **Use `accuracy`, don't ignore it.** Let `d` = distance from the fix to the venue centre and `a` = `accuracy`.
   - Accept if `d - a <= R`, i.e. the uncertainty circle overlaps the geofence. This is lenient to indoor drift.
   - Reject if `a > A_max`. A suggested **`A_max` ≈ 1000 m** catches approximate-location and cell-only fixes. Show a message such as "Your location is too imprecise. Turn on Precise Location for Safari/Chrome." with platform-specific steps.
   - Optionally, a grey zone (`a` between ~150 m and 1000 m) could accept with a flag for later review instead of blocking.
3. **Collection settings:** `{ enableHighAccuracy: true, timeout: 15000, maximumAge: 0 }`. Better still, run `watchPosition` for up to ~10–15 s, keep the reading with the smallest `accuracy`, and stop early once `a` is under ~50 m.
4. **Check on the server.** The client sends `{lat, lng, accuracy, timestamp}`; Django recomputes distance and the rule. Store the check result, not the raw trail, for privacy and data minimisation (Swiss FADP). Raw coordinates can be discarded right after the check.
5. **Check once per game, not per vote.** One successful check unlocks voting for that game for the session. That avoids repeated prompts on iOS, and votes typically open late in or after the game.
6. **Handle each failure explicitly:**
   - `PERMISSION_DENIED` → explain how to re-enable; also offer a fallback (see 7).
   - `POSITION_UNAVAILABLE` / `TIMEOUT` → retry button; suggest joining venue Wi-Fi or switching Wi-Fi on (Wi-Fi scanning helps even when not connected).
   - Insecure context → only happens in dev, so ensure HTTPS everywhere.
7. **Consider a second, stronger presence signal** if clubs care about fraud. One option is a short code or QR shown on the arena scoreboard/jumbotron, rotated every few minutes. It's cheap, works when geolocation fails, and is harder to fake remotely than coordinates (though it can still be relayed by phone). Geolocation could remain the default with the code as fallback, or both could be combined.
8. **Plan a field test** at one real rink with a few iOS and Android phones (precise and approximate, Wi-Fi on and off, several seat sections). Log `accuracy`, distance to centre and time-to-fix. The estimates above are not a substitute.

## 6. Web push

### Without installation: Android and desktop

Support data from [caniuse: Push API](https://caniuse.com/push-api):

| Platform | Push in a normal browser tab (no install)? | Notes |
|---|---|---|
| Chrome / Samsung Internet / Firefox on Android | **Yes** | Android wakes the browser for incoming pushes even when it's closed ([web.dev Push FAQ](https://web.dev/articles/push-notifications-faq)). On Android 13+, the *browser app* also needs the OS notification permission. Chrome normally asks for it itself. |
| Chrome / Edge / Firefox desktop | **Yes** | Delivered only while the browser process is running. "The only time a push won't be received is when the browser is completely closed" ([web.dev Push FAQ](https://web.dev/articles/push-notifications-faq)). |
| Safari on macOS | **Yes**, Safari 16+ on **macOS 13 Ventura+** | Push needs "an explicit user gesture" to request. "Violations of the userVisibleOnly promise will result in a push subscription being revoked" ([WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/)). |
| Safari / any browser on **iOS / iPadOS** | **No** | See below. |

The flow is standard everywhere:

1. Register a service worker.
2. On a user tap, call `Notification.requestPermission()`.
3. Call `pushManager.subscribe({ userVisibleOnly: true, applicationServerKey: <VAPID public key> })`.
4. POST the subscription to Django.
5. The server sends encrypted payloads, for example with `pywebpush`, to the subscription's endpoint (FCM, Mozilla autopush, or `*.push.apple.com`).

Every push must show a notification; silent pushes are not allowed.

### iOS / iPadOS

Source: [WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/).

- **Version:** iOS / iPadOS **16.4 or later**.
- **Home Screen web app only.** The fan must open the site in Safari, then Share → **Add to Home Screen**, then open it *from the Home Screen icon*. Web Push in a Safari tab is not covered by the post. In practice, the APIs are not exposed there **(developer reports, verify with feature detection)**. Since 16.4, third-party browsers on iOS can also offer Add to Home Screen.
- **Manifest:** the post requires a Web App Manifest with `display: "standalone"` or `"fullscreen"`. From **iOS 26**, every site added to the Home Screen opens as a web app by default unless the user switches off "Open as Web App" ([WebKit: Safari 26 beta](https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/)). Still ship a proper manifest for older iOS.
- **Permission flow:** inside the installed web app, `Notification.requestPermission()` must be called "in response to direct user interaction", such as a tap on "Notify me when voting opens". iOS then shows the system prompt.
  - Users manage the permission per web app in Settings → Notifications.
  - Notifications appear on the Lock Screen, in Notification Center and on a paired Apple Watch, and respect Focus modes.
  - The Badging API is supported.
  - No Apple Developer account is needed; allow `*.push.apple.com` if you filter outbound traffic.
- **Declarative Web Push** (iOS/iPadOS **18.4+**, macOS 15.5+): the push payload itself describes the notification (`"web_push": 8030`, plus `title` and a `navigate` URL). It needs no service worker JS to display and is immune to the silent-push revocation penalty ([WebKit: Meet Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/)). The post does not say the Home Screen requirement was lifted.

### Implications for Firststar

- Push is a **good optional extra for Android and desktop fans**: one tap and no install.
- On iOS it is a **multi-step funnel**: Safari → Share → Add to Home Screen → open the icon → tap "Notify me" → allow. Many fans won't finish it, so **do not make vote discovery depend on push.** Rely on the in-rink announcement, scoreboard QR and club social channels; push is a bonus.
- Detect support with `'PushManager' in window && 'Notification' in window && 'serviceWorker' in navigator`. On iOS Safari, where it's missing, show a small "Add to Home Screen to get alerts" hint rather than a broken button.
- The Home Screen web app has **its own storage and cookies, separate from Safari** (standard WebKit behaviour; not re-verified here). A fan logged in in Safari will need to log in again in the installed app. Prefer a login method that is easy to repeat, such as a magic link or passkey.
- Each push must show a notification: send only "Voting for *Team A vs Team B* is open". No silent or data-only pushes.

## Open questions / to verify on devices

- Real `accuracy` and time-to-fix inside a Swiss rink (field test).
- Exact current Chrome-Android geolocation prompt labels, and whether Android Chrome now offers a one-time option.
- iOS approximate-location radius as seen by a web page.
- Whether the `accuracy` value the browser exposes is passed through unchanged from the native APIs (68%) or rescaled to 95%.
