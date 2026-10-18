# Snapchat 14.28.0.54 (Android) — deep teardown

**Package:** `com.snapchat.android` · targetSdk 36 / minSdk 23 · universal APK, 204 MB
**SHA-256:** `f63dffb9f89abbe8d1e97c0dc6a7856eeee6e132f1d8cf3853e87dc31d11e3b7`
**Signing cert:** `O=Snapchat, OU=Captain Captcha, CN=Daniel Smith`, SHA-256 `2F1CAAFCA1ED30D0B4E38863EEFABEA0E815711FA4CF79B822519A8259D95A58` — Snap Inc's genuine, long-standing key (a MOD/tampered build would be re-signed). **Verified authentic.**
**Method:** static only — signature verify, apktool manifest decode, dex string/URL/gRPC extraction, REA inventory; JADX decompile for class detail. App never installed/run. Signing/attestation and ranking internals not reversed.

## 1. Architecture at a glance (vs TikTok)
- **Lean code:** 10 dex / **82 MB** (TikTok: 56 dex / 356 MB). Much more is **native C++** and a reactive single-surface UI.
- **Only 30 activities** (TikTok: 498) — Snapchat is a **single-activity, camera-first** app; everything is custom-rendered surfaces, not Android activities. But **328 exported** components (mostly SDK/service entry points).

## 2. Backend infrastructure (849 URLs)
Top domains: `snapchat.com`(364), `sc-cdn.net`(191, CDN/Bolt), `snap.com`, `snapads.com`(ads), `sc-jpl.com`(map+search: `ms.sc-jpl.com`, `search.sc-jpl.com`).
- **Multi-cloud, explicit in code:** `gcp.api.snapchat.com`, `aws.api.snapchat.com`, `europe-west1-gcp.api.snapchat.com`, `staging-us-central1-gcp…` → runs on **both Google Cloud and AWS**, region-split (EU west1, US central1).
- CDN: `bolt-gcdn.sc-cdn.net`, `gcs.sc-cdn.net`, `cf-st.sc-cdn.net` (Bolt = Snap's media CDN).
- Partners in-build: Spotify, Netflix, YouTube, Adobe, Amazon; payments **PayPal, Braintree, Razorpay, CardinalCommerce** (3-D Secure).

## 3. Permissions (79)
Camera, RECORD_AUDIO, **ACCESS_BACKGROUND_LOCATION** + FINE + MEDIA_LOCATION (Snap Map), READ/WRITE_CONTACTS, READ_MEDIA_IMAGES/VIDEO, Bluetooth scan/connect (Spectacles), READ_PHONE_STATE, USE_BIOMETRIC, the FOREGROUND_SERVICE_* set incl. LOCATION, MEDIA_PROJECTION, MICROPHONE, CONNECTED_DEVICE, ADSERVICES_AD_ID.

## 4. Security / identity / encryption (a major differentiator)
- **Fidelius = end-to-end encryption.** `FideliusIdentityService/InitializeDeviceKey`, `/GetFriendKeys`; `FideliusRecryptService/Initiate|Poll|AcknowledgeRecrypt` — device/friend key exchange and key-rotation ("recrypt"). Snap encrypts Snaps between friends.
- **Janus = auth/identity.** `LoginService/AppLogin|LoginWithPassword|LoginWith1TLv1|v3|AnswerChallenge|Reactivate`, `RegistrationService`, `AccountRecoveryService`, `ChallengeOrchestrationService/VerifyChallenge`. Plus **passkeys** (`PasskeyExternalService`) and **2FA** (`TwoFaExternalService`).
- **Attestation / anti-abuse:** `snap.security.caid.CAIDService/NotifyCAID` (client attestation ID) and `snap.security.duplex.SecurityDuplexService/SubmitHermodClientPayload` + `AckHermodEvent` — **Hermod** is Snap's device-integrity/anti-abuse telemetry channel (their analog to TikTok's Argus). Native crypto libs back it.
- Age assurance: `UpdateBirthdateService/GetAgeVerificationOptions|VerifyAgeAnswerChallenge`; `complianceengine.ComplianceEngineService`; `identity.PermissionSettingsMesh`.

## 5. The ranking / recommendation engines
- **Spotlight (their TikTok-style FYP):** `billboard.services.api.RankingService/GetRanking` + `creativetools.compute.ComputeFeedService/ComputeFeed`.
- **Valis = the friend-graph/ranking engine** — `Valis/GetFriendClusters`, `ValisPreferences/GetMutedFriends|MuteFriend` → computes who your "best friends" are and clusters your graph (drives Best Friends, Quick Add, chat ordering).
- **Send-To ranking:** `search.sendtoranking.SendToRankingService/GetCandidateFeatures(ByContext)` — ranks *who to send a snap to* on the share screen (a core engagement nudge).
- **Memories uses ML embeddings:** `memories.embedding.MemoriesEmbeddingService/UploadSnapEmbeddings` + `MemoriesFaceTagPreviewService` (face-tagging your saved snaps). `FinetuningService` = per-user model fine-tuning. `snapindex.clientservice` = on-device search index.

## 6. Snap Map — the location machine (very extensive)
Dedicated service cluster: `map.eagle.EagleBackend/InferCurrentLocation` + `RemovePlaceVisit`/`RemoveAllPlacesVisits` (**stores a history of places you visit**), `map.locationcontext/GetLocationContext|GetFriendsIcons|GetGroupLocationContext`, `map.actionmoji.Actionmoji` (your Bitmoji's map pose/school), `map.pins.Pins`, `map.garfield.tiles.Tiles`/`playlist`, `map.valhalla.Valhalla` + `map.slippy.Slippy` (tile/routing), `map.peliasproxy`/`mapsearchproxy` (geocoding/search), `maps.footsteps.Footsteps/StreamMemories` (links location history to Memories), `maps.device.MapDevice`.
Tokens: `ghost_mode`, `live_location`, `background_location`, `inferredlocation` → continuous/background location with a Ghost Mode opt-out.

## 7. Generative AI (heavily built-out)
- **Minerva** (`cameos.minerva.MinervaService`): `GenerateAISnap`, `GenerateAISong`, `GenerateAIFont`, `GenerateAICameraMedia`, `GenerateCaption`, `GenerateEnhancedMediaFromPrompt`, `ProcessMedia`; `RemixStickerService/Remix`.
- **Dreams** (`cameos.genai.dreams.Service/Generate`) + `genai.identity` (upload your face for AI selfies), `generative_backgrounds`.
- **My AI** chatbot (tokens present), `perception.asr.AutomatedSpeechRecognition` (speech-to-text).
- **Screenshop** (`perception.showcase.screenshop.ScreenshopService/Shoppable`) — shop products detected in images.

## 8. Creative tools & content
Lenses/AR: `lenses.RemoteApiService/PerformApiCall`, `lenses.lensgator.LensGatorService`, `lens.le.LensExplorerService`, `lens.prompt.LensPromptService`, `lens.inlenscreation`. Creative: `customsticker`, `customoji`, **Giphy** (`GiphyService/GiphySearch|GiphyTrending`), `creativetools.search`, `feed`, `items`, `userdata`. Content delivery: `content.v2.MediaDeliveryService`/`MediaOriginService`. Premium: `premiumstories.PremiumStories/GetPlaybackStory`. Also `polls.PollService`, `lists.api.Lists/CreateListStory`, `music.music_service`, `aura.api.AuraService` (astrology).

## 9. Bitmoji & commerce
`bitmoji.avatar`, `bitmoji.profile`, `bitmoji.accounts`, **`bitmoji.fashion.v1/v2/PurchaseIAPItem`** (buy Bitmoji outfits). Payments: `payments.commerce.order.OrderService/GetOrderHistory`. Subscription: Snapchat+ (`/snapchat.com/plus`). Rewards: `incentive_campaign.IncentiveCampaignGrantService/GrantReward`.

## 10. Engagement / "addiction" mechanics (grounded in tokens)
- **Streaks** — `streakexpiration` appears **106×**, `streakcount`, `hourglass` (the ⌛ about-to-expire warning). A daily mutual obligation with an explicit expiry timer = loss-aversion engine. (Snapchat+ even sells **streak restores**.)
- **Charms (83×)** — auto-generated Friendship Charms/badges that gamify each friendship.
- **Snap Score (21×)** — a running point total for sending/receiving, pure number-go-up.
- **Read receipts / "opened" / "delivered" (27×)** + **typing indicators** + **screenshot detection (37×)** — real-time social-pressure signals that compel fast reciprocation.
- **Quick Add** (friend suggestions via `friending.server` + Valis clusters + `ContactBook/FullSyncContactBookUpload` contact upload) — constantly widens the graph.
- **Best Friends** (`bfs.api.PinBestFriendService`) + Send-To ranking — surfaces the people you're most likely to message.
- **Camera-first open** — the app opens straight to the camera to prompt creation, not consumption.
- Push: `notification.PushNotificationService` + `PushNotificationDataRegistryService` + `notif.DeviceStateReceiver`.

## 11. Safety / countermeasures
`abuse.conversationsafety.ConversationSafetyService/GetConvoSafetyPrompt`, `abuse.support.ReportService/SendReport`, `AppealService`, **`ReformSchoolService/GetQuiz|ScoreQuiz`** (a remediation quiz for policy violators), age verification, Ghost Mode (Map opt-out), Family Center / compliance engine. As with TikTok, safety tooling exists alongside the engagement loops rather than throttling them.

## Honest limits
gRPC service/method names, domains, permissions, and tokens are **real** (string literals + decoded manifest). Request/response **payloads, ranking weights, Fidelius/Hermod crypto internals, and the native code are not reversed** — gRPC bodies are protobuf and much logic is native/obfuscated. The engagement analysis maps named, in-build mechanics to behavioral-design principles, not recovered proprietary math.

## Deliverables (`C:\Users\orion\Downloads\snapchat-re\`)
