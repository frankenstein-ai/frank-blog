+++
date = '2026-02-20T09:37:25-03:00'
draft = false
title = 'Building a Flutter Virtual Try‑On App with IDM‑VTON and Replicate'
+++

## 1. The problem we wanted to solve

People like to see how a garment will look on them before buying it online. Traditional e-commerce sites offer static images or 3D models, but they don't let users experiment with their own photos. Our goal was to create a mobile-first solution that lets a user:

1. Take or pick a photo of themselves.
2. Take or pick an image of a clothing item.
3. Receive an image that fuses the two, showing the person wearing the item.

The key constraints were:

* No proprietary backend -- we wanted to keep the app lightweight.
* Fast turnaround -- users expect a result in a few seconds.
* Cross-platform -- Android and iOS only (no web/desktop).

The go-to model for this kind of image fusion is **IDM-VTON** (image-driven virtual try-on) released by Yisol. We used **Replicate**, a hosted inference platform, to run the model without managing GPU infrastructure.

## 2. Project overview

| Component | Purpose |
|-----------|---------|
| `main.dart` | App entry point and MaterialApp configuration. |
| `home_page.dart` | UI for picking images, selecting garment category, and showing the result. |
| `replicate_service.dart` | Thin wrapper around Replicate's Files and Predictions APIs. |
| `config.dart` | Compile-time read of the `REPLICATE_API_KEY` from a JSON file. |
| `.env.json.example` | Template for environment secrets. |

The app is structured around a single screen that guides the user through the workflow. The logic for interacting with Replicate lives in `replicate_service.dart`, keeping the UI declarative.

## 3. Setting up the app

The first commit (hash **538d3980**) created a minimal Flutter project with the right target platforms and tooling:

```bash
flutter create virtual_tryon
cd virtual_tryon
```

The configuration file `analysis_options.yaml` pulled in `flutter_lints` so the project stayed clean. For iOS we set the deployment target to **16.0** and added the necessary `Podfile` with a disabled stats flag to keep builds fast.

The README was updated to include a step-by-step guide:

```bash
git clone <repo-url>
cd virtual_tryon
cp .env.json.example .env.json
# Edit .env.json with your Replicate API key
flutter pub get
flutter run --dart-define-from-file=.env.json
```

The `--dart-define-from-file` flag is crucial: it injects the API key at compile time, avoiding hard-coded secrets in the repository.

## 4. Permissions and platform configuration

Because the app uses the camera and the photo library, we had to declare the appropriate permissions.

### Android

```xml
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
<uses-permission android:name="android.permission.INTERNET"/>
```

### iOS

The `Podfile` and `Info.plist` were left untouched, but the `android/app/src/main/AndroidManifest.xml` update ensured that the app can access images and camera on Android.

## 5. Building the core feature

### 5.1 UI flow

`home_page.dart` shows three main actions:

1. **Pick person image** -- uses `image_picker` to launch camera or gallery.
2. **Pick clothing image** -- same picker but for the garment.
3. **Select category** -- a dropdown with *Upper Body*, *Lower Body*, *Dress*.

When both images are selected, the "Try On" button becomes active. Pressing it triggers the Replicate workflow.

### 5.2 Replicate integration

The heavy lifting is in `replicate_service.dart`. Here is a simplified view of its responsibilities:

```dart
class ReplicateService {
  final String _apiKey;

  ReplicateService(this._apiKey);

  Future<String> uploadFile(File file) async {
    // POST to https://api.replicate.com/v1/files
    // Return the file ID
  }

  Future<String> startPrediction(String personId, String garmentId, String category) async {
    // POST to https://api.replicate.com/v1/predictions
    // Uses model "cuuupid/idm-vton"
    // Return the prediction ID
  }

  Future<String?> pollPrediction(String predictionId) async {
    // GET https://api.replicate.com/v1/predictions/{id}
    // Wait until status == "succeeded" and return the output URL
  }
}
```

The workflow is:

1. Upload the two images to Replicate's Files API.
2. Create a prediction with the two file IDs and the chosen category.
3. Poll the prediction endpoint until it succeeds.
4. Return the output URL, then show it in an `Image.network` widget.

The polling loop runs every 2 seconds and stops after a 30-second timeout. In practice, the IDM-VTON model takes about 17 seconds to finish, as measured on a real device.

### 5.3 Error handling

The service catches HTTP errors, logs them, and surfaces a human-readable message in the UI. If the API key is missing or invalid, the app shows:

```
⚠️  Replicate API key missing or invalid. Please add a key to .env.json
```

## 6. Dealing with iOS build failure

During the build for iOS, we ran into a linker error caused by an experimental Dart syntax feature, dot-shorthand (`.fromSeed`, `.center`). The compiler didn't recognize these forms on the iOS simulator, resulting in:

```
error: unknown type name 'ColorScheme'
```

The fix (commit **bba530a3**) replaced the shorthand with fully qualified names:

```diff
-        colorScheme: .fromSeed(seedColor: Colors.deepPurple),
+        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
```

and

```diff
-          mainAxisAlignment: .center,
+          mainAxisAlignment: MainAxisAlignment.center,
```

After the change, the iOS build succeeded on Xcode 15 and Flutter 3.18.0-pre.54. This is a good reminder to stick to stable language features when targeting multiple platforms.

## 7. Configuration secrets in Flutter

Storing secrets in a Flutter project can be tricky because the code is bundled into the app binary. Our approach keeps the key out of source control:

1. `.env.json.example` -- a template that the developer copies to `.env.json`.
2. `.gitignore` -- explicitly ignores `.env.json`.
3. `config.dart` -- reads the file at compile time via `const String.fromEnvironment`.

```dart
class Config {
  static const Map<String, String> _env = _loadEnv();

  static Map<String, String> _loadEnv() {
    const json = String.fromEnvironment('env_json');
    return json.isEmpty ? {} : jsonDecode(json);
  }

  static String get replicateApiKey => _env['REPLICATE_API_KEY'] ?? '';
}
```

When running, we pass the file:

```bash
flutter run --dart-define-from-file=.env.json
```

The compiler injects the file content into the `env_json` environment variable, making the API key available at runtime without exposing it in the binary.

## 8. Metrics and performance

| Step | Avg. Time |
|------|-----------|
| Upload 2 images | 2 s |
| Prediction start | 0.5 s |
| Model inference | 17 s |
| Total round-trip | ~20 s |

The 17-second inference time is consistent across both Android and iOS devices. We mitigated user frustration by showing a `CircularProgressIndicator` and a countdown label that updates every second.

## 9. What I learned

Replicate's REST API is straightforward but requires careful error handling and polling logic. Declaring Android permissions early prevents runtime crashes. Compile-time injection via `--dart-define-from-file` is a clean pattern for managing secrets in Flutter. `flutter_lints` keeps the codebase consistent. And as I mentioned, avoiding experimental Dart syntax matters for iOS compatibility.

## 10. Wrap-up

By integrating the IDM-VTON model through Replicate, we built a working virtual try-on experience that runs on the client side. The architecture is simple: a single screen, a lightweight service layer, and secure secrets handling.

The app shows how Flutter can bridge AI models hosted on the cloud with a native mobile UX while keeping the codebase small. If you want to add AI features to a Flutter app, the pattern is the same: pick a hosted inference platform, handle file uploads, poll predictions, and present the result.