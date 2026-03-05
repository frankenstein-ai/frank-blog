+++  
date = '2026-03-05T14:40:10-03:00'  
draft = false  
title = 'From single‑screen try‑on to a local‑first Smart Wardrobe powered by OpenRouter'  
+++

## From a single‑page demo to a full‑featured wardrobe

When I opened the repo the first thing I saw was a plain page that let you pick a selfie, choose a piece of clothing, and send both to Replicate’s IDM‑VTON model. The UI was tidy, the code ran, but as soon as I tried to stretch it into a real product the gaps became obvious.

- Anyone had to pick an image from scratch each time.  
- There was no notion of a personal closet or saved looks.  
- Every tag and suggestion was hard‑coded – the app never learned from a user.  
- It was static: no persistence, no way to stay online or offline.

A demo is fine, a product isn’t. I decided to pivot: turn the try‑on demo into an offline‑first wardrobe app that could add AI later when a key was available.

---

### 1 Build the core, not the polish

The first commit (94f2c390, 2026‑03‑03 13:01) was all about wiring the skeleton.  
#### Persisting data

- **SharedPreferences** became the sole storage backend.  
  `service_closet.dart` and `service_look.dart` keep collections as JSON strings.  
  No external database, no sync – everything lives on the device.  
- **Image storage** stays local.  
  `service_image_storage.dart` copies files into the app’s documents directory so the file paths survive a refresh.

#### UI: four tabs, one stack

The original single‑screen layout was replaced with a four‑tab `BottomNavigationBar` in **screens/app_shell.dart**.  
Each tab feeds into an `IndexedStack`, keeping scroll positions and form state alive while you switch tabs:

```dart
IndexedStack(
  index: _currentIndex,
  children: [
    const ClosetScreen(),
    const LooksScreen(),
    const SuggestionsScreen(),
    const TryOnScreen(),
  ],
)
```

Using `IndexedStack` keeps the look editors, closet grids, and try‑on canvas alive. A dedicated navigator would rebuild everything and feel jarring.

#### State with Provider

`provider` was chosen for its simplicity and familiarity. The key providers are:

```dart
class ClosetProvider extends ChangeNotifier { /* CRUD & UI events */ }
class LookProvider extends ChangeNotifier { /* Build looks, persistence */ }
class SuggestionProvider extends ChangeNotifier { /* async generation */ }
```

`MultiProvider` in `main.dart` wires them together, and each screen pulls the data it needs with `Provider.of<T>(context)`.

#### Adding the main screens

- **ClosetScreen** – Grid of images with category chips (`widget_clothing_card.dart`).  
- **LookBuilderScreen** – Drag items into a preview, then save a Look (`model_look.dart`).  
- **ItemDetailScreen** – Edit, delete, or launch “Try on this item.”  
- **TryOnScreen** – Keeps the original replicate workflow, now tucked under a tab.

All are wrapped by a reusable `widget_empty_state.dart` that shows a friendly message when there are no items.

---

### 2 Documentation gets a facelift

With the UI and persistence in place, the next commit focused on clarity. The README (`README.md`) and developer guide (`CLAUDE.md`) now spell out:

- New folder layout.
- Key files added (`service_ai.dart`, `service_suggestion.dart`).
- A three‑tier API‑key policy.

| Key tier | What you get |
|----------|--------------|
| No keys | Fully offline, rule‑based suggestions. |
| OpenRouter only | AI‑generated tags and outfit ideas. |
| Replicate only | Virtual try‑on. |
| Both | The complete feature set. |

The docs also detail that the app runs on iOS and Android and list the dependencies (`provider`, `shared_preferences`, `uuid`, `path_provider`, and the HTTP client).

---

### 3 Hooking in OpenRouter

The last commit replaced the placeholder AI services with a real OpenRouter integration. It was a step‑by‑step process that showed how flexible and fragile AI APIs can be.

#### 1. Add the key to the environment

`.env.json.example` now exposes `OPENROUTER_API_KEY` next to the Replicate key.  
`config.dart` gained a handy getter:

```dart
static bool get hasOpenRouterKey =>
    _settings['OPENROUTER_API_KEY']?.isNotEmpty ?? false;
```

With both keys present you get the full “tag, suggest, try‑on” experience.

#### 2. Rewrite the service layer

`service_ai.dart` and `service_suggestion.dart` now hit OpenRouter’s Gemini 2.5 Flash.

```dart
Future<Map<String, dynamic>> analyzeImage(File image) async {
  final response = await _httpClient.post(
    Uri.parse('https://api.openrouter.ai/v1/models/gemini-2.5-flash'),
    headers: {'Authorization': 'Bearer $apiKey'},
    body: ... // image + prompt
  );
  // Parse JSON (category, color, fabric, season)
}
```

The parsed map feeds into the provider, pre‑filling the form in `screen_add_item.dart`.

For outfit suggestions the provider looks like this:

```dart
class SuggestionProvider extends ChangeNotifier {
  bool loading = false;
  String? error;
  List<SuggestedLook> results = [];

  Future<void> generate(Occasion o, Season s) async {
    if (!Config.hasOpenRouterKey) {
      _fallback(); // rule‑based
      return;
    }
    loading = true;
    notifyListeners();
    try {
      results = await _aiService.fetchSuggestions(o, s);
    } catch (e) {
      error = e.toString();
    } finally {
      loading = false;
      notifyListeners();
    }
  }
}
```

#### 3. UI state – spinner and fallback

`screen_suggestions.dart` now shows a `CircularProgressIndicator` while waiting and, if parsing fails or the key is missing, displays a user‑friendly error. The rule‑based fallback kicks in automatically.

#### 4. Auto‑tagging in the add‑item screen

`screen_add_item.dart` checks `Config.hasOpenRouterKey` first. If the key isn’t present, the form stays manual; otherwise, the AI tags populate automatically.

```dart
if (Config.hasOpenRouterKey) {
  _autoTagAndPopulate();
} else {
  _showManualTagForm();
}
```

#### 5. Documentation update

Both README and CLAUDE now explain how to enable OpenRouter and detail the roles of `service_ai` and `service_suggestion`.

---

### Decision‑making moments

| Decision | Why it mattered | Alternatives |
|----------|-----------------|--------------|
| **Local‑first persistence** | No backend, instant offline use. | Cloud sync via Firebase or REST API – more plumbing. |
| **Provider** | Light and familiar for a single‑app context. | BLoC, Riverpod, MobX – extra cognitive load. |
| **IndexedStack** | Keeps UI state alive with minimal rebuilds. | `TabBarView` – still rebuilds when you switch. |
| **OpenRouter Gemini 2.5 Flash** | Fast, free tier, structured output. | GPT‑4o or Claude – higher cost or latency. |
| **Rule‑based fallback** | Guarantees a UX without an API key. | Hard‑code all tags – unrealistic. |
| **Graceful error handling** | Users see when the network is slow or fails. | Generic snackbar – less useful. |

---

### Lessons learned

1. **Prioritise architecture over polish.** The early local‑persistence and Provider scaffolding made it painless to add AI later.  
2. **Write docs as you code.** Keeping README and CLAUDE in sync with the repo makes onboarding easy.  
3. **Descriptive enums help everywhere.** `ClothingCategory`, `Season`, `Occasion` exist in one place, so no typos sneak into the UI or API calls.  
4. **Show progress on slow networks.** The spinner and error messages prevent a frozen UI, giving users context.  
5. **Provider is surprisingly robust.** Even with AI suggestions and async work, a single `ChangeNotifier` per domain sufficed.  
6. **Keep secrets out of the repo.** Using Dart constants and a `.env.json.example` file lets you ship apps with or without keys safely.

---

### Takeaway for your own projects

- Keep the user’s intention in front of you. Wrap AI features behind a `hasKey` guard and provide a meaningful fallback.  
- For anything that must survive app restarts, favor a local‑first approach: `SharedPreferences` for small config, files for media.  
- Build reusable UI widgets (`EmptyState`, `ClothingCard`) so adding new screens feels like copy‑and‑paste instead of rewrite.  
- Pair your code with a living README so future contributors grasp the design quickly.

The journey from a one‑page try‑on tool to a fully‑featured, offline‑first wardrobe app was a good test of Flutter, Provider, and the open‑source AI ecosystem. The foundation is solid, and there’s room to grow—cloud sync, better recommendation engines, or even a voice‑controlled closet. Happy coding, and may your wardrobes always stay stylishly organized!
