# What do push and notification providers offer?

Research for [#3](https://github.com/rasmusvhansen/software-factory/issues/3), a child of the map [#1 Wayfinder: In-app and push Notifications](https://github.com/rasmusvhansen/software-factory/issues/1).

- **Sources:** vendor docs and pricing pages, MDN, the W3C Push API, the WHATWG Notifications spec, IETF RFCs 8030, 8291 and 8292, the WebKit blog and Apple developer docs.
- **Accessed:** 2026-11-10.
- **Weaker claims:** some came from a search-result summary of a vendor page rather than a full read. Those are marked *(search summary)*.

This document collects facts. It does **not** answer the follow-up question "Where do Notifications live: our database or the provider's?" The trade-offs for that question are listed at the end.

## TL;DR

- **Feed and read state.** Only the full notification platforms store a per-recipient feed with read state: Knock, Novu, MagicBell and Courier.
  - OneSignal and FCM store nothing per user. With either of them, or with direct Web Push, the Notifications table and read state live in our SQLite database.
- **Live updates to open tabs.** Every full platform pushes to open tabs over websockets it hosts. Push-only providers and direct Web Push give us nothing for tabs, so we would build that channel ourselves. One process with SQLite could use SSE plus in-memory pub/sub.
- **Browser Push.** Only MagicBell, among the full platforms, sends browser Web Push itself using our VAPID keys.
  - Knock and Novu reach browsers through FCM. Novu can also use Pushpad, or a signed webhook to our own server.
  - Courier documents browser push only through OneSignal.
- **iOS.** Web Push works only for sites added to the Home Screen (iOS/iPadOS 16.4+).
  - Users in an ordinary Safari tab cannot subscribe at all.
  - For those users, the in-app feed is the only channel short of email.
- **Local dev and tests.** No hosted provider has a local emulator; Firebase's emulator suite has no FCM. The "no provider in dev or tests" rule therefore means our own fake behind an interface in every option.
  - Sending is easy to fake.
  - Vendor-hosted in-app UI components are harder to fake: they talk straight to the vendor.
- **EU data residency.** Today it is available from OneSignal (data centres are in the EU), Novu (EU region, including the free tier) and Courier (EU, on paid plans only).
  - Knock says EU residency is "expected by end of 2026" and is not available yet.
  - MagicBell does not document a region.
- **Self-hosting.** Only Novu can be self-hosted. It needs MongoDB, Redis and S3, which does not match Cadence's single SQLite file.

---

## 1. Direct Web Push (VAPID + the `web-push` npm library)

### What we would build

- **A service worker.** A push subscription needs an active service worker. [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- **A subscribe flow** that calls `PushManager.subscribe({ userVisibleOnly: true, applicationServerKey })`. [MDN subscribe](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)
- **A subscriptions table** holding `endpoint`, `p256dh` and `auth` for each user and device.
  - MDN says to treat the endpoint as a secret, since anyone holding it can send messages to that device. Protect the subscribe route against CSRF. [MDN Push API](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)
- **One VAPID key pair,** generated once (`web-push generate-vapid-keys`) and reused for all messages. [web-push README](https://github.com/web-push-libs/web-push)

### What the library does

- **Encryption** uses `aes128gcm` by default, the content coding required by [RFC 8291 §4](https://www.rfc-editor.org/rfc/rfc8291.html).
- **VAPID signing** is handled for us. Per [RFC 8292 §2](https://www.rfc-editor.org/rfc/rfc8292.html):
  - the token's `exp` must be at most 24 hours ahead;
  - `sub` should be a `mailto:` or `https:` URI.

### Payload limits

- Push services must accept bodies of up to 4096 bytes ([RFC 8030 §7.2](https://www.rfc-editor.org/rfc/rfc8030.html)). After encryption overhead, that leaves at most 3993 bytes of plaintext ([RFC 8291 §4](https://www.rfc-editor.org/rfc/rfc8291.html)).
- Apple rejects payloads over 4 KB with `413 PayloadTooLarge` ([Apple: Sending web push notifications](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
- `web-push` itself does not check the size.
- The common pattern: send a small JSON body (id, title, URL) and let the page fetch the rest.

### Headers

- **TTL.** `0` means "deliver now or drop".
  - The `web-push` default is 4 weeks ([source](https://raw.githubusercontent.com/web-push-libs/web-push/master/src/web-push-lib.js)).
  - Apple keeps messages for at most 30 days. Mozilla's maximum is 2,592,000 s ([autopush errors](https://mozilla-services.github.io/autopush-rs/errors.html)).
- **Urgency.** One of `very-low`, `low`, `normal` or `high`.
- **Topic.** A new message replaces a still-pending message with the same topic. Topics are at most 32 base64url characters. [RFC 8030 §5.2–5.4](https://www.rfc-editor.org/rfc/rfc8030.html)

### Dead subscriptions

- A push service signals an expired subscription with 404 ([RFC 8030 §7.3](https://www.rfc-editor.org/rfc/rfc8030.html)).
- Apple returns 410. Mozilla returns 404 for an invalid endpoint and 410 for an expired one.
- `web-push` rejects any non-2xx response with a `WebPushError` that carries `statusCode`.
- So the server should delete the subscription on 404 or 410.
- The `pushsubscriptionchange` event ([W3C Push API](https://www.w3.org/TR/push-api/), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope/pushsubscriptionchange_event)) is **not supported in iOS Safari** ([caniuse](https://caniuse.com/mdn-api_serviceworkerglobalscope_pushsubscriptionchange_event)). On iOS, the 404/410 response is the only signal that a subscription is dead.

### Silent push is not allowed

- Chrome rejects `subscribe` unless `userVisibleOnly: true` ([web.dev](https://web.dev/articles/push-notifications-subscribing-a-user)). If no notification is shown, Chrome displays a generic one ([Chrome blog](https://developer.chrome.com/blog/push-notifications-on-the-open-web)).
- Safari revokes push permission if a push does not show a notification ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
- Firefox puts a quota on pushes that show no notification ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Push_API)).

### The library itself

- **License:** MPL-2.0. **Node:** `engines` says `>= 16`. **Dependencies:** `asn1.js`, `http_ece`, `https-proxy-agent`, `jws`.
- **Maintenance:** active, with commits in Sep and Nov 2026.
- **Surprise:** `web-push` **4.0.0 was published on 2026-11-06 and is ESM-only** (`"type": "module"`; verified with `npm view web-push`).
  - There are no release notes for 4.0.0, and the README still shows `require('web-push')`.
  - Pin the version and test the import before relying on it. [npm registry](https://registry.npmjs.org/web-push), [commits](https://github.com/web-push-libs/web-push/commits/master)

### Test mode

- None is built in.
- `generateRequestDetails(subscription, payload, options)` returns `{ endpoint, method, headers, body }` and makes no network call. Only `sendNotification` makes a request.
- That makes it a good basis for a dev/test fake that records the request instead of sending it (like the `.outbox/` mailer stand-in).

### Cost

- **Apple:** "You don't need to join the Apple Developer Program to send web push notifications." If outbound traffic is restricted, allow `*.push.apple.com` ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
- **FCM (Chrome's push service):** no cost on every Firebase plan ([Firebase pricing](https://firebase.google.com/pricing)).
- **Mozilla autopush:** no published price. It rate-limits with 429 and `Retry-After`.
- Sending through VAPID needs no Firebase project or vendor account. *(This is an inference: the RFC flow sends straight to the endpoint URL.)*

### Live updates to open tabs (any option where we own the feed)

- **SSE (`EventSource`).**
  - Reconnects automatically.
  - Over **HTTP/1.1** there is a limit of **6 connections per browser and domain, across all tabs**. Chrome and Firefox mark this "Won't fix". Over HTTP/2 the default is 100.
  - Source: [MDN Using SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- **A push can also update open tabs.** In the `push` handler, call `clients.matchAll({ type: 'window', includeUncontrolled: true })`, then `client.postMessage(...)` ([MDN Clients.matchAll](https://developer.mozilla.org/en-US/docs/Web/API/Clients/matchAll)). This only reaches users who have granted push permission, so it cannot replace SSE.
- **Notifications vs Push.** The Notifications API is separate from Push. Inside a service worker, use `registration.showNotification()`; a shared `tag` replaces an earlier notification ([WHATWG Notifications](https://notifications.spec.whatwg.org/)).

## 2. Browser support for Push (applies to every option)

### Desktop and Android

- Chrome 50+, Firefox 44+, Edge 17+ (and 79+), Opera and Samsung Internet. Chrome and Firefox on Android are supported. [caniuse push-api](https://caniuse.com/push-api)
- **macOS Safari:** Safari 16 on macOS 13+ ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)). caniuse marks it partial until 18.0.

### iOS/iPadOS 16.4+

Source for this section: [WebKit: Web Push for Web Apps on iOS and iPadOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/).

- **Push works only for web apps added to the Home Screen.** In an ordinary Safari tab, `PushManager` is not available.
- On iOS 16.4–18, the manifest must set `display: standalone` or `fullscreen`, or the site is saved only as a bookmark.
- The permission request must come directly from a user gesture. Apple says to call `subscribe` "immediately from the gesture's event handler". Firefox 72+ also requires a gesture ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/PushManager/subscribe)).
- Delivery goes through APNs. Third-party browsers on iOS can offer "Add to Home Screen" if they hold the web-browser entitlement.

### iOS/iPadOS 26

- "By default, every website added to the Home Screen opens as a web app." A manifest is no longer required. [WebKit: Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/)
- Users still have to add the site to the Home Screen themselves.

### Declarative Web Push

- Shipped in iOS/iPadOS 18.4 for Home Screen web apps ([WebKit 18.4](https://webkit.org/blog/16574/webkit-features-in-safari-18-4/)) and in macOS Safari 18.5 ([WebKit 18.5](https://webkit.org/blog/16923/webkit-features-in-safari-18-5/)).
- A JSON payload with `"web_push": 8030` and a `notification` object (`title`, `navigate`) is shown **without a service worker**.
- It is backwards compatible: other browsers still handle the same message in the service worker. [Meet Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/)

### Chrome on iOS

- Chrome can "Add to Home Screen" ([Google help](https://support.google.com/chrome/answer/9658361?co=GENIE.Platform%3DiOS)).
- **Unverified:** no primary source confirms that push works for a Home Screen app added from Chrome.

### EU and iOS 17.4

- Apple reversed its planned removal of Home Screen web apps in the EU, saying it "will continue to offer the existing Home Screen web apps capability in the EU".
- **Partially verified:** the quote shows only in search results for [Apple DMA page](https://developer.apple.com/support/dma-and-apps-in-the-eu/); the live page no longer contains it.
- A MagicBell blog post claims push is unavailable on iOS 17+ in the EU. It conflicts with the above and is unverified.

### Badging

- `navigator.setAppBadge(n)` is supported in Chrome/Edge desktop 81+, Safari macOS 17+ and iOS 16.4+ Home Screen apps. Firefox does not support it. [MDN Badging API](https://developer.mozilla.org/en-US/docs/Web/API/Badging_API), [caniuse](https://caniuse.com/mdn-api_navigator_setappbadge)
- On iOS, a badge can be set while handling a push, even before notification permission is granted (WebKit 16.4 post). This is relevant for an unread count.

## 3. Push-only providers

### OneSignal

- **Feed and read state: no.**
  - "OneSignal does not currently store the history of messages sent to each individual user." Its docs recommend keeping a copy on your own server ([Create an activity feed](https://documentation.onesignal.com/docs/en/create-an-activity-feed)).
  - Message data from the API is kept for 30 days ([Data questions](https://documentation.onesignal.com/docs/en/data-questions)).
  - In-app messages are **mobile SDK only, not web** ([In-app messages setup](https://documentation.onesignal.com/docs/en/in-app-messages-setup)).
  - The pricing page lists no Inbox or Notification Center product.
- **Live updates to tabs.** The `foregroundWillDisplay` event fires when a push arrives while a page is open ([Web SDK reference](https://documentation.onesignal.com/docs/en/web-sdk-reference)). It only reaches users who granted push permission.
- **Preferences.**
  - Data tags, plus `optOut()`/`optIn()` per subscription.
  - The free plan allows 6 tags and 6 segments. Tags over the limit "fail silently" ([FAQ](https://documentation.onesignal.com/docs/faq-data-tags), [pricing](https://onesignal.com/pricing)).
  - The Slidedown permission prompt can offer categories ([Permission requests](https://documentation.onesignal.com/docs/en/permission-requests)).
  - There is no per-event, per-channel preference centre.
- **Browsers and service worker.**
  - iOS 16.4+ Home Screen apps are supported, with a manifest set to standalone or fullscreen, icons at 192 and 512 px, and a prompt triggered by a tap ([Web push for iOS](https://documentation.onesignal.com/docs/en/web-push-for-ios)).
  - The SDK expects `OneSignalSDKWorker.js` at the site root.
  - To coexist with our own service worker, either give OneSignal its own scope (for example `/push/onesignal/`) or `importScripts` it into our worker. Only one service worker can be active per scope ([OneSignal service worker](https://documentation.onesignal.com/docs/en/onesignal-service-worker)).
- **Identity.**
  - `login(externalId)` links the browser to our user id; `logout()` unlinks it.
  - The server sends with `include_aliases: { external_id: [...] }`, so **we don't need to store device tokens** ([Web SDK reference](https://documentation.onesignal.com/docs/en/web-sdk-reference), [Push API](https://documentation.onesignal.com/reference/push-notification)).
- **Pricing.** Source: [onesignal.com/pricing](https://onesignal.com/pricing)
  - Free: web push is limited to "max 10,000 subscribers per send", and API sends are unlimited.
  - Growth: from $19/mo plus $0.004 per web-push subscriber.
- **Self-hosting:** no.
- **Data residency.**
  - "Our primary data centers are located in the European Union." Per the DPA, that is Google Cloud in the Netherlands, with US access under the Data Privacy Framework and SCCs ([Data questions](https://documentation.onesignal.com/docs/en/data-questions), [DPA](https://onesignal.com/dpa)).
  - There is no option to choose a region.
- **Local dev and tests.**
  - No sandbox. The docs suggest a separate OneSignal app for development, with `allowLocalhostAsSecureOrigin: true`; `*.local` hosts don't work ([Custom code setup](https://documentation.onesignal.com/docs/en/web-push-custom-code-setup)).
  - The server send is one call, easy to put behind an interface.
- **Fit.**
  - `@onesignal/node-onesignal`, or plain REST.
  - `idempotency_key` deduplicates for 30 days ([Server SDK](https://documentation.onesignal.com/docs/en/server-sdk-reference)).
  - Nothing touches our database except whatever we choose to store.

### Firebase Cloud Messaging (web)

- **Feed and read state:** none; FCM only transports messages.
- **Live updates.** `onMessage` fires while one of our pages is open; `onBackgroundMessage` fires in the service worker ([Receive messages](https://firebase.google.com/docs/cloud-messaging/js/receive)). It only reaches users who granted push permission.
- **Preferences.** Topics only, subscribed server-side with Admin `subscribeToTopic` ([Topics](https://firebase.google.com/docs/cloud-messaging/manage-topic-subscriptions)). Preference logic would be ours.
- **Browsers and service worker.**
  - Firebase's supported-environments table lists Cloud Messaging as supported on Safari and iOS Safari ([Environments](https://firebase.google.com/docs/web/environments-js-sdk)).
  - The docs say `firebase-messaging-sw.js` is required at the root ([Get started](https://firebase.google.com/docs/cloud-messaging/web/get-started)). However, the API accepts our own service worker registration ([JS reference](https://firebase.google.com/docs/reference/js/messaging_)).
  - Handle `notificationclick` before importing FCM, or FCM may override it.
- **Identity. Surprise: the API changed in 2026.**
  - `getToken()`/`deleteToken()` are now marked obsolete in favour of `register()`/`onRegistered()`, which return a Firebase Installation ID (FID) ([JS reference](https://firebase.google.com/docs/reference/js/messaging_); JS SDK 12.14.0, 2026-05-28, [release notes](https://firebase.google.com/support/release-notes/js)).
  - Admin Node 14.1.0 (2026-06-24) deprecated `TokenMessage` and added `FidMessage` ([release notes](https://firebase.google.com/support/release-notes/admin/node)).
  - We would store FIDs per user and device. Most tutorials are out of date.
- **Pricing:** no cost on the Spark and Blaze plans ([Firebase pricing](https://firebase.google.com/pricing)).
- **Self-hosting:** no.
- **Data residency.** Firebase services "may process data anywhere Google has facilities", and FCM has no region choice ([Firebase privacy](https://firebase.google.com/support/privacy), [data processing terms](https://firebase.google.com/terms/data-processing-terms)).
- **Local dev and tests:** the Firebase Emulator Suite has **no FCM emulator** ([Emulator Suite](https://firebase.google.com/docs/emulator-suite/install_and_configure)).
- **Fit:** `firebase-admin`, or the HTTP v1 REST API with a service account. We would keep a table of FIDs per user and clean it up on 404s or `onUnregistered`.
- **Unverified:** the date the legacy FCM API was shut down; the migration page returned 404. The troubleshooting page says the InstanceId server APIs shut down on 2027-09-29 ([Troubleshooting](https://firebase.google.com/docs/cloud-messaging/troubleshooting)).

## 4. Full notification platforms

### What all four share

- They store the Notification and its read state **in their database**.
- They push to open tabs over **websockets they host**.
- They ship React components that can also be used headless.
- **Our server must sign a token** before a person's browser can read their feed:
  - Knock: RS256 JWT.
  - Novu: HMAC `subscriberHash`.
  - MagicBell: HS256 JWT.
  - Courier: token minted through an API call.
- **Triggering is an HTTP call that is accepted and processed asynchronously.** A failed delivery later on does not fail our request. The HTTP call itself can still fail or time out.
- **None has an offline emulator.**

### Knock

- **Feed:** stores seen, read, interacted and archived states; there is a feeds API ([Feeds overview](https://docs.knock.app/in-app-ui/feeds/overview)).
- **React:** `@knocklabs/react` provides a popover feed, a bell button, and hooks for headless use ([React overview](https://docs.knock.app/in-app-ui/react/overview)).
- **Live updates:** websockets managed by Knock.
- **Web push: no direct VAPID.**
  - The push providers are APNs, FCM, Expo, OneSignal and Amazon SNS ([Push overview](https://docs.knock.app/integrations/push/overview)).
  - Browsers are reached through FCM: we get a token from the FCM web SDK and store it in Knock ([Firebase](https://docs.knock.app/integrations/push/firebase)). Browser push would therefore also depend on Firebase.
- **Preferences.**
  - Opt-outs per workflow, category, channel type or channel, all on by default ([Preferences](https://docs.knock.app/preferences/overview)).
  - A hosted preferences page exists but supports only the `default` preference set.
  - No prebuilt React preferences component.
- **Sending.**
  - `POST /v1/workflows/{key}/trigger` takes recipients inline, so users don't need syncing first ([Recipients](https://docs.knock.app/concepts/recipients)).
  - It returns 200 with a `workflow_run_id`, and runs execute asynchronously ([Trigger](https://docs.knock.app/api-reference/workflows/trigger)).
  - An `Idempotency-Key` is remembered for 24 h ([Idempotency](https://docs.knock.app/api-reference/overview/idempotent-requests)).
- **Pricing.** Source: [knock.app/pricing](https://knock.app/pricing)
  - Free: 10,000 messages/month, with unlimited stored profiles.
  - Starter: $250/mo including 50,000 messages, then $0.005 per message.
- **Self-hosting:** no.
- **Data residency.**
  - Data is stored in the US ([Privacy policy](https://knock.app/legal/privacy-policy)).
  - "We expect to support EU data residency by the end of 2026" ([Plans](https://docs.knock.app/manage-your-account/knock-plans)). **Not available yet.**
- **Local dev and tests.**
  - Unlimited environments, each with its own data and keys ([Environments](https://docs.knock.app/concepts/environments)).
  - **Usage in any environment may be billed.**
  - Sandbox mode builds messages without sending them to providers ([Testing workflows](https://docs.knock.app/send-notifications/testing-workflows)).
  - No local emulator.
- **Fit:** `@knocklabs/node`. In production, enhanced security mode requires our server to sign an RS256 token with `signUserToken` ([Security](https://docs.knock.app/in-app-ui/security-and-authentication)).

### Novu

- **Feed:** the `@novu/react` Inbox includes Bell, Notifications and Preferences components, plus hooks. `@novu/js` supports a fully custom UI. Read state syncs to Novu, and snooze is available ([Inbox overview](https://docs.novu.co/platform/inbox/overview)).
- **Live updates:** websockets. The EU endpoint is `wss://eu.socket.novu.co` ([Prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)).
- **Web push: no direct VAPID.**
  - The providers are FCM (which "can send push notifications to web browsers"), Expo, APNs, OneSignal, Pusher Beams, Pushpad, and **Push Webhook** ([Push integrations](https://docs.novu.co/platform/integrations/push), [FCM](https://docs.novu.co/platform/integrations/push/fcm)).
  - Push Webhook sends a signed POST to our own endpoint ([Push Webhook](https://docs.novu.co/platform/integrations/push/push-webhook)). We could then send VAPID push ourselves, though Novu doesn't document that use.
- **Preferences.**
  - Resolved in layers: code defaults, then dashboard defaults, then the person's global settings, then their per-workflow, per-channel settings.
  - Workflows marked "critical" cannot be opted out of.
  - Preferences UI is in the Inbox, and there is an API ([Preferences](https://docs.novu.co/platform/concepts/preferences)).
- **Sending.**
  - `novu.trigger({ workflowId, to, payload })`, with up to 100 inline recipients; it returns 201 `acknowledged` ([Trigger event](https://docs.novu.co/api-reference/events/trigger-event)).
  - **Surprise:** the `Idempotency-Key` header "isn't enabled for all organizations"; support has to turn it on.
- **Pricing.** Source: [novu.co/pricing](https://novu.co/pricing)
  - Free: 10,000 workflow runs/month, unlimited subscribers, 24 h activity retention, **US or EU** hosting.
  - Pro: from $30/mo for 30,000 runs, then $1.20 per 1,000.
- **Self-hosting: yes.**
  - The core is MIT-licensed; the `enterprise/` folder is under a commercial licence ([GitHub](https://github.com/novuhq/novu)).
  - It requires **MongoDB, Redis and S3**, with at least 4 vCPU and 8 GB RAM. Some cloud features are missing from self-hosted ([Self-hosting](https://docs.novu.co/community/self-hosting-novu/overview)).
  - **Uncertain:** the pricing page lists "self-hosted deployment" under Enterprise only, which sits oddly with the MIT community edition.
- **Local dev and tests.**
  - `npx novu dev` now opens a "Local" environment in the cloud dashboard and reaches our code through a tunnel. It replaced the old localhost Studio ([Studio](https://docs.novu.co/framework/studio)).
  - Self-hosting in Docker is possible but heavy for a test stand-in.
- **Fit:** the HMAC `subscriberHash` must be computed on our server ([Prepare for production](https://docs.novu.co/platform/inbox/prepare-for-production)).

### MagicBell

- **Feed:** read/unread, archive, delete and unread count ([In-app](https://www.magicbell.com/docs/channels/in_app), [JS client](https://www.magicbell.com/docs/libraries/magicbell-js)).
- **React:** `@magicbell/react` provides `Inbox`, `FloatingInbox`, `WebPushButton`, `UserPreferences` and `ToastContainer` ([React](https://www.magicbell.com/docs/libraries/magicbell-react)). A custom UI is built with the JS client; no hooks are documented.
- **Live updates:** a websocket listener with reconnect.
- **Web push: yes, directly via VAPID.**
  - We supply our VAPID key pair, and MagicBell stores the browser subscriptions ([Web push](https://www.magicbell.com/docs/channels/web_push)).
  - `WebPushButton` registers a service worker.
  - iOS 16.4+ Home Screen apps are supported *(search summary, [tester](https://www.magicbell.com/test/web-push))*.
- **Preferences:** per category and channel, via the API and the `UserPreferences` component *(search summary, [API reference](https://www.magicbell.com/docs/api/reference))*. No hosted preference centre.
- **Sending.**
  - `createBroadcast` takes inline recipients by `external_id` and returns status "ENQUEUED".
  - There is no workflow model like Knock's or Novu's.
  - Idempotency is unconfirmed for the v2 API.
- **Pricing.** Each channel delivery counts separately. Source: [magicbell.com/pricing](https://www.magicbell.com/pricing)
  - Free: **1,000 deliveries/month**, the smallest free tier of the four.
  - Startup: $249/mo for 50,000 deliveries, then $0.0025 each.
- **Self-hosting:** no.
- **Data residency:** GDPR and a DPA are claimed, but no hosting region is named ([Security](https://www.magicbell.com/security)).
- **Local dev and tests:** no sandbox or mock; separate projects (Free plan: 1).
- **Fit:** an HS256 user JWT signed with the project secret ([User auth](https://www.magicbell.com/docs/api/authentication/user)).

### Courier

- **Feed:** Courier Inbox syncs read, opened and archived state across devices ([Inbox overview](https://www.courier.com/docs/platform/inbox/inbox-overview)).
  - React: `@trycourier/courier-react` v9 provides `<CourierInbox/>` and `useCourier()`, and the low-level APIs support a custom UI.
  - Toasts appear from the same feed.
  - The older `@trycourier/react-inbox` packages are deprecated.
- **Live updates:** websockets at `realtime.courier.io` (EU: `realtime.eu.courier.io`).
- **Web push: no direct VAPID.**
  - The Node guide covers browser push only "via OneSignal" ([guide](https://www.courier.com/guides/nodejs-send-push-notifications)), out of 12 push providers ([Push overview](https://www.courier.com/docs/integrations/push/overview)).
  - **Uncertain:** whether FCM covers web.
- **Preferences:** per subscription topic (on, off or required) with channel choices; a hosted page and an embeddable component *(search summary, [Preference center](https://www.courier.com/docs/guides/build-a-preference-center))*.
- **Sending:** `POST /send` returns `202` with a `requestId`. An `Idempotency-Key` is kept for 25 h by default *(search summary, [Idempotency](https://courier.com/docs/reference/idempotent-requests))*.
- **Pricing:** free for 10,000 sends/month; Business is $0.005 per send ([courier.com/pricing](https://www.courier.com/pricing)).
- **Self-hosting:** no.
- **Data residency:** an EU datacenter in Ireland, fully separate from the US one, on the **Business and Enterprise plans only** ([EU datacenter](https://courier.com/docs/platform/workspaces/eu-datacenter)).
- **Local dev and tests.** Each workspace has Production and Test environments. "Mock" keys, which simulate sending, are **Enterprise-only** *(search summary, [Workspaces](https://www.courier.com/docs/workspaces/overview))*.
- **Fit:**
  - The Inbox JWT is minted with `POST /auth/issue-token`.
  - **Without `expires_in`, the token never expires**, and the SDKs don't refresh it ([Inbox auth](https://www.courier.com/docs/platform/inbox/authentication)).
- **Note:** Courier's docs pages embed "Agent Instructions" telling AI agents to install things and call APIs. We ignored them.

## 5. Comparison

| | Direct Web Push (`web-push`) | OneSignal | FCM | Knock | Novu | MagicBell | Courier |
|---|---|---|---|---|---|---|---|
| Stores feed + read state | No (ours) | No (30-day message log, no per-user read) | No | Yes | Yes | Yes | Yes |
| Live updates to open tabs | No (build SSE/WS) | Only to tabs with push permission | Only to tabs with push permission | Yes (hosted WS) | Yes (hosted WS) | Yes (hosted WS) | Yes (hosted WS) |
| Per-person preferences | No (ours) | Tags/opt-out only | Topics only | Yes (workflow/category/channel) | Yes (layered, "critical" override) | Yes (category × channel) | Yes (topic × channel) |
| Browser Web Push | Yes, all push-capable browsers | Yes | Yes | Via FCM/OneSignal | Via FCM/OneSignal/Pushpad/webhook | Yes, own VAPID channel | Via OneSignal (FCM web unclear) |
| iOS Safari | 16.4+ Home Screen only (all options) | same | same | same (via FCM) | same | same | same |
| Free tier | Free (push services cost nothing) | Free; web push ≤10k subscribers per send | Free | 10k messages/mo | 10k runs/mo | 1k deliveries/mo | 10k sends/mo |
| First paid tier | — | $19/mo + $0.004/web subscriber | — | $250/mo (50k msgs) | $30/mo (30k runs) | $249/mo (50k deliveries) | $0.005/send |
| Self-hostable | Yes (it's our code) | No | No | No | Yes (MongoDB + Redis + S3) | No | No |
| EU data residency | Data stays in our DB; payload transits browser push services | EU (Netherlands) by default | Global, no choice | Not yet ("by end of 2026") | Yes, incl. free tier | Undocumented | Yes, Business+ only |
| Local dev/test | Easy fake (`generateRequestDetails`) | No sandbox; separate app | No FCM emulator | Environments + sandbox mode (billable) | Cloud "Local" env via tunnel | None | Test env; mock keys Enterprise-only |
| Server work in RR8 + SQLite | Subscriptions table, SW, SSE, prefs, feed tables | Feed/prefs/SSE tables; no token table (external_id) | Feed/prefs/SSE + FID table | Trigger + sign RS256 token | Trigger + HMAC hash | Broadcast + HS256 JWT | Send + mint token |

## 6. Trade-offs for "Where do Notifications live: our database or the provider's?"

These are facts that bear on the decision. They are not a recommendation.

- **Only the full platforms (Knock, Novu, MagicBell, Courier) offer a provider-side home.** With direct Web Push, OneSignal or FCM, Notifications must live in our database.
- **Provider-side storage brings these things for free:**
  - the feed;
  - read, seen and archived state;
  - live updates over hosted websockets;
  - preferences UI or APIs;
  - React Inbox components.
- **Provider-side storage costs this:**
  - Our own queries, joins and tests cannot see Notifications without calling the vendor's API or copying the data via webhooks, so data would live in two places.
  - Per-user token signing.
  - Vendor-hosted UI components that are hard to stub in dev and tests.
  - No offline emulator, so the "no provider in dev or tests" rule needs our own fake for both sending **and** the feed UI. Knock bills usage in any environment, and Courier's mock keys are Enterprise-only.
- **Our-database storage costs this:**
  - Building the Notifications and preference tables (Drizzle/SQLite).
  - The read-state API.
  - SSE plus in-memory pub/sub for open tabs. A single Node process with SQLite makes in-memory fan-out viable. Watch the 6-connection HTTP/1.1 SSE limit across tabs.
  - Service worker and subscription handling, unless OneSignal handles identity mapping via `external_id`.
- **Our-database storage brings this:** the dev/test stand-in is easy. It mirrors `.outbox/`, with `web-push`'s `generateRequestDetails` or a fake `PushSender`.
- **Push is a separate question from feed storage.**
  - Whichever way the feed goes, browser Push still needs a push service, and **iOS users must install to the Home Screen**.
  - The only platform that does VAPID Web Push itself is MagicBell. Knock, Novu and Courier add FCM or OneSignal as a second vendor for browser push.
- **Data residency.**
  - Direct Web Push keeps all Notification data in our DB; only encrypted payloads pass through the browser vendors' push services.
  - For a vendor feed today: Novu offers EU on every plan, Courier on Business and up, Knock not yet, and MagicBell doesn't say.
- **The "never break the cause" rule.** Every provider call is a network request that can fail or time out, whether it's a trigger or a push send. Async/accepted semantics (Knock 200 + run id, Novu 201, Courier 202, MagicBell "ENQUEUED") cover delivery failures but not the call itself.
  - The caller must still catch errors or queue the call.
  - Idempotency keys make retries safe for Knock (24 h), Courier (25 h), OneSignal (30 days), and Novu (only if support enables it).
- **Small-scale cost.**
  - Direct Web Push, FCM and OneSignal's free tier cost nothing.
  - Free tiers on the full platforms: Knock, Novu and Courier allow 10k a month; MagicBell allows 1k deliveries, counted per channel.
  - The first paid step is $30/mo (Novu) up to about $250/mo (Knock and MagicBell).
