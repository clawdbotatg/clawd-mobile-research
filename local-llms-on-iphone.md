# Local LLMs on iPhone: how they run, whether apps can share models, and the "Ollama for iOS" question

*Research report, 2026-07-14. Compiled from a multi-source, adversarially-verified deep-research pass (23 sources fetched, 25 top claims verified by 3-vote panels; 21 confirmed, 4 refuted). Primary sources are Apple/Google developer docs and WWDC sessions; confidence noted per section.*

## TL;DR

- **How apps run open models today:** each app links an inference runtime (MLX, llama.cpp, LiteRT-LM, Core ML) *into its own binary* and bundles or downloads its own copy of the weights. There is no system-level model store for third-party open-source models.
- **Can two apps share one downloaded model?** **Yes — but only if both apps are from the same developer team**, via an **App Group shared container**. That is the only Apple-sanctioned mechanism. Cross-developer sharing does not exist on iOS.
- **Is an Ollama-like shared daemon possible?** **No.** iOS has no third-party daemons, apps are suspended shortly after backgrounding, and even Apple's 2026 "any LLM provider" extension of Foundation Models ships providers as Swift packages compiled into each app — not as a shared service. The closest platform answer is Apple's own **Foundation Models framework**: one OS-owned ~3B model shared by *all* apps, zero download, zero app-size cost.
- **If you ship two apps needing the same model:** put both in an App Group, download the weights once into the shared container (background `URLSession` can write directly into it), and have both apps read from there. Where the system model is good enough, use Foundation Models instead and skip downloading entirely.

---

## 1. How iOS apps run open-source LLMs today

There is no OS-level "model runtime" for third-party models. Every app that runs an open model does it **in-process**: the inference library is compiled into the app, and the weights live in the app's sandbox (bundled in the binary, or downloaded on first run into the app's own container). The main runtimes as of mid-2026:

| Runtime | Status (2026) | Notes |
|---|---|---|
| **MLX (mlx-swift)** | Active, Apple-blessed | Apple-native open-source stack; Apple-tuned mixed 3/4/6/8-bit quantization, its own Metal kernels. A 2025 production study (arXiv 2511.05502) recommends it as the default for Apple-centric deployments *(medium confidence — benchmarked on Mac hardware, extrapolated to iPhone)*. Apple itself shipped an `MLXLanguageModel` provider at WWDC26. |
| **llama.cpp (Metal)** | Widely used in community apps | Named in the research question but *no claim about it survived verification* in this pass — treat specifics (speed, GGUF ecosystem state on iOS) as an open question, not as absent. |
| **Core ML** | Active | Apple's general ML framework; used via swift-transformers conversions. More conversion friction for LLMs than MLX. |
| **MediaPipe LLM Inference API** (Google) | **Maintenance-only** as of mid-2026 | Ran Gemma-3 1B, Gemma-2 2B, Phi-2, StableLM, Falcon fully on-device via CocoaPods. Google explicitly directs iOS projects to migrate to the **LiteRT-LM Swift API**. Its documented workflow is strictly per-app (`Bundle.main.path` / app Documents) — no sharing story. *(high confidence, primary Google docs updated 2026-06-12)* |
| **LiteRT-LM** (Google) | Active — MediaPipe's successor | Google's forward path for on-device LLMs on iOS. |
| **Foundation Models** (Apple) | Active, iOS 26+ | Not an open-source-model runtime per se — see §3. |

**Key structural fact:** in all of the third-party-runtime cases, the model weights are an ordinary file in the app's sandbox. That's what makes the sharing question below a *filesystem* question, and iOS filesystem sandboxing is strict.

## 2. Can two apps share the same downloaded model? (the core question)

### Same developer team: yes — App Groups *(high confidence, primary Apple docs)*

**App Groups are Apple's only sanctioned mechanism for two iOS apps to share files**, including multi-GB model weights:

- Both apps declare the same App Group entitlement (e.g. `group.com.yourteam.models`). Membership requires **the same Apple developer team** — the entitlement is validated against your Team ID.
- Each app gets a real filesystem URL to the shared container via `FileManager.containerURL(forSecurityApplicationGroupIdentifier:)` and can read/write it freely. Apple documents **no file-size or file-type limits** on the container.
- **Downloads can land directly in the shared container:** set `sharedContainerIdentifier` on a background `URLSession` configuration and the OS downloads straight into the App Group container — including while the app isn't running. So App A can background-download an 800 MB GGUF/MLX model once, and App B reads the same file. No double download, no double disk.

Practical caveats surfaced during verification (worth engineering around):

- **No cross-process locking is provided** — if both apps might write (or one might re-download while the other reads), coordinate with `NSFileCoordinator`.
- **Container lifecycle:** the shared container is deleted only when *all* apps in the group are uninstalled.
- Un-verified in this pass (open questions): how Settings attributes the storage across the sibling apps, eviction behavior under disk pressure, and whether two processes can safely mmap the same weights file concurrently. Test these before shipping.

### Different developers: no

There is **no Apple-sanctioned mechanism** for apps from different teams to share downloaded files. No shared model cache, no cross-team App Groups, no public "model provider" extension point that carries weights. The only "workaround" is both apps shipping under one Team ID — i.e., becoming the same developer.

### What about Background Assets / On-Demand Resources?

Apple's **Background Assets** framework is built exactly for large downloads like ML weights (Apple's own Foundation Models *adapter* workflow uses it): "essential" packs arrive before first launch, "prefetch" packs download without blocking launch, processing happens in a Background Download extension while the app isn't running. **But its sharing scope is one app + that app's own extension** (`BAAppGroupID` links an app to *its* extension, not to a sibling app). Apple-hosted asset packs are explicitly per-app ("Each Apple hosted asset pack can be used on one app" — WWDC25 session 325). In the legacy non-managed flow the download does land in an App Group container, so a same-team sibling *can* read it — but that capability comes from App Groups, not from anything Background Assets sanctions. *(high confidence, primary Apple docs)*

## 3. Is there room for an "Ollama for iPhone"? Essentially no — and Apple built the alternative

### Why the daemon model can't exist on iOS

Ollama's whole design — a persistent background server that many clients hit over localhost — has no legal shape on iOS:

- Third-party **daemons don't exist** on iOS; apps are **suspended shortly after backgrounding** (Apple DTS's canonical "iOS Background Execution Limits" post). A "server" app stops serving the moment the user switches to the client app — which is precisely when the client needs it.
- Background execution modes (audio, location, BGTasks…) are purpose-scoped and none permits "keep serving inference requests indefinitely."
- Apps exposing an OpenAI-compatible HTTP server on localhost *do* exist on the App Store (e.g. "Local LLM Server"-style apps), but they only work **while foregrounded** (or briefly after), which defeats the multi-app-sharing use case on a phone. *(This corner — App Store policy details and real-world behavior of such apps — was the weakest-covered area of the research; the impossibility conclusion is inferred from the confirmed per-app distribution model and Apple's documented background limits, not from a single primary source saying "no Ollama.")*

### Apple's answer: one system model, shared by every app *(high confidence, primary Apple sources)*

The **Foundation Models framework** (iOS/iPadOS/macOS/visionOS 26+) is architecturally the "shared local LLM" — just Apple's, not yours:

- Every app gets Swift API access (`SystemLanguageModel.default`, `LanguageModelSession`) to the **same on-device ~3B-parameter model that powers Apple Intelligence**, plus Private Cloud Compute models for heavier requests.
- The weights are **built into the OS and updated with OS updates** — zero app-size cost, zero download, zero duplication across apps. This is exactly the deduplication an Ollama would provide, delivered as platform infrastructure.
- Hard constraints: **iOS 26+ and Apple Intelligence-compatible hardware with Apple Intelligence enabled** (further gated by region and model-download state). Apps must check `SystemLanguageModel` availability at runtime; older iPhones get nothing. And it's Apple's model — you don't pick the weights (LoRA-style adapters exist for tuning).

### WWDC 2026: the framework opens up — but not into a shared runtime

As of WWDC 2026, Foundation Models accepts **nearly any LLM provider, local or server-based**, via public `LanguageModel` / `LanguageModelExecutor` protocols; Apple open-sourced conforming implementations (incl. an MLX-based provider) and vendors like Anthropic and Google ship conforming packages. Everything is driven through the same `LanguageModelSession` API. **The catch for this question:** third-party providers are distributed as **ordinary Swift packages compiled into each app**. It unifies the *API*, not the *weights* — two apps using the same MLX provider still each carry their own model copy unless they share it themselves via an App Group. *(high confidence, WWDC26 session 339; note: a claim that this gives blanket access to the Hugging Face MLX-community catalog on-device was refuted — don't assume specific catalog coverage.)*

## 4. Recommendations for shipping two apps that need the same model

Assuming both apps are yours (same team) — which matches the situation prompting this research:

1. **First ask whether Apple's system model is enough.** If the ~3B Foundation Models model can do the job and the iOS 26 / Apple Intelligence device floor is acceptable for your audience, use it in both apps: no download at all, and it's the only true cross-app sharing on the platform. It can be the fast path on new devices with your own model as fallback on older ones.
2. **Otherwise: one App Group, one download.** Put both apps in `group.com.yourteam.models`. Whichever app the user opens first downloads the weights via a background `URLSession` with `sharedContainerIdentifier` set, straight into the shared container. Both apps resolve the model path through a tiny shared Swift package (also a good home for version/manifest logic).
3. **Coordinate access.** Guard downloads/updates with `NSFileCoordinator` (or a lock file + version manifest) so App B never reads a half-written file and both apps agree on the current model version. Exclude the weights from iCloud backup (`isExcludedFromBackup`).
4. **Optionally add Background Assets per app** for pre-first-launch delivery — but treat the App Group container as the single source of truth so a Background Assets download by either app still lands where both can read it (legacy non-managed flow).
5. **Converge on one runtime + one weight format across both apps** (MLX is the best-supported Apple-native choice, and the WWDC26 provider protocols make it easy to keep app code uniform) — sharing weights only works if both apps consume the same file.
6. **Don't build a "server app."** A foregrounded localhost server can't serve another app the moment the user switches to it. On iOS the sharing layer is the filesystem (App Group), not a socket.

If the two apps were from *different* developers, the honest answer is: each ships its own copy, and the only mitigation is choosing the smallest adequate model — or both adopting the system Foundation Models model.

---

## Refuted during verification (do not cite)

- "Seven runtimes benchmarked head-to-head incl. ANEMLL / Core AI framework" (apple-silicon-llm-bench repo claim) — 0/3 votes.
- "MLC-LLM ships an official iOS SDK + OpenAI-compatible REST server as a concrete iOS path" — 0/3 votes.
- "Foundation Models' MLX backend opens the Hugging Face MLX-community catalog to iOS apps" — 0/3 votes.
- "Foundation Models is fully offline / free / transmits nothing" — 1/2 refuted (the on-device model is on-device, but Private Cloud Compute is part of the framework; don't oversimplify).

## Open questions worth a follow-up pass

- Current state of **llama.cpp on iOS** (Metal backend, GGUF ecosystem, which shipping apps use it) vs MLX and LiteRT-LM on iPhone-class hardware — the most-used community runtime had no surviving verified claim here.
- Real App Store review posture on **foregrounded OpenAI-compatible localhost servers**.
- Whether the WWDC26 **MLX provider** actually runs arbitrary open models on iPhone (formats, catalogs, memory limits).
- **Multi-GB weights in an App Group** in practice: storage attribution in Settings, disk-pressure eviction, concurrent mmap from two processes.

## Key sources

- Apple: [Foundation Models](https://developer.apple.com/documentation/FoundationModels) · [SystemLanguageModel](https://developer.apple.com/documentation/foundationmodels/systemlanguagemodel) · [WWDC25 session 286](https://developer.apple.com/videos/play/wwdc2025/286/) · [WWDC26 session 339](https://developer.apple.com/videos/play/wwdc2026/339/) · [Configuring App Groups](https://developer.apple.com/documentation/Xcode/configuring-app-groups) · [Background Assets](https://developer.apple.com/documentation/BackgroundAssets) · [iOS Background Execution Limits (DTS)](https://developer.apple.com/forums/thread/685525) · [Apple Newsroom on Foundation Models](https://www.apple.com/newsroom/2025/09/apples-foundation-models-framework-unlocks-new-intelligent-app-experiences/)
- Google: [MediaPipe LLM Inference for iOS](https://ai.google.dev/edge/mediapipe/solutions/genai/llm_inference/ios) (maintenance notice + LiteRT-LM migration)
- Study: [On-device LLM deployment survey, arXiv 2511.05502](https://arxiv.org/pdf/2511.05502) (MLX recommendation; Mac-hardware benchmarks)
