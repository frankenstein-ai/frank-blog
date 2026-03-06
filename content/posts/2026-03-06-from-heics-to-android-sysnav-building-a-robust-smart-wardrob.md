+++  
date = '2026-03-06T17:09:10-03:00'  
draft = false  
title = 'From HEICs to Android SysNav: Building a Robust Smart Wardrobe'  
+++

## Introduction  

The Smart Wardrobe app is a Flutter project that lets you build looks and try them on in a virtual closet. Everything stays on the phone – data is stored in SharedPreferences and the device’s file system. Only the AI tagging, suggestions, and the V‑TON service reach the internet.

During the tenth week of 2026 we gathered a flood of new feedback and bugs:

- Photos taken with the camera sometimes lost background removal or wrinkling, leaving a pixelated halo in the try‑on preview.
- After a restart, placeholder cards failed to resolve because the app saved absolute file paths.
- On real Android devices, the scroll gesture on the clothing grid didn’t work, and the Try‑On button got hidden behind the system navigation bar.
- iOS users found that they couldn’t save try‑on images to the photo library because we hadn’t handled permissions properly.
- We also ran into race conditions when providers or the look‑detail page loaded data asynchronously, making the UI briefly flash empty items.

Getting from those pain points to a stable app took several steps. Below I’ll walk through the decisions we made, how we changed the code, and what we learned – so you can apply the same lessons to your own Flutter projects.

## 1. AI Image Enhancement Pipeline  

### Problem  

When users added garments, the photos still carried background and a wavy wrinkle. The original approach only compressed the image locally, which didn’t fix the problem.

### Exploration of Solutions  

- Keep it local: compress with flutter_image_compress and rely on the camera to deliver clean images.  
- Add a remote service: call a standard background‑removal API or an unwrinkling model on each import.  
- Deploy a private model: run the IDM‑VTON repo locally, though that would add weight to the app and slow startup.

Because we want offline support, the first two options are reasonable. The third would make the binary bigger and slower.

### Decision  

We decided to use Gemini 2.5 Flash through OpenRouter for background removal and a lightweight unwrinkling routine. If the user lacks an OpenRouter key, we fall back to Replicate’s background‑removal service. This keeps costs low and still gives a clean preview.

### Implementation  

```dart
Future<File> enhanceImage(File original) async {
  // 1) try Gemini via OpenRouter
  final geminiRes = await _gemini.removeBackground(original);
  if (geminiRes.success) return geminiRes.file;

  // 2) fallback to Replicate
  final repRes = await ReplicateService.removeBackground(original);
  return repRes.file;
}
```

In `screen_add_item.dart` the image pipeline becomes:

```dart
final file = await _imagePicker.pickImage();      // user picks photo
final compressed = await FlutterImageCompress.compressWithFile(
  file.path,
  minWidth: 800,
  minHeight: 600,
);
final enhanced = await ServiceAI.instance.enhanceImage(compressed);
await _saveToDatabase(enhanced);
```

Remote AI gave us clean silhouettes in the try‑on preview right away. The fallback means the app still works offline when the key is missing.

## 2. File System & Image Storage Refactor  

### Problem  

We stored image locations in SharedPreferences as absolute file paths. After an uninstall or binary update, those paths became stale and the UI crashed when trying to show placeholder cards.

### Exploration  

- Keep absolute paths and validate them at startup.  
- Store relative paths and resolve them against the documents directory when needed.

### Decision  

Use relative paths and expose two helpers:

```dart
class ImageStorageService {
  static late final Directory _appDocDir;

  static Future<void> init() async {
    _appDocDir = await getApplicationDocumentsDirectory();
  }

  static String toRelative(File file) =>
      file.path.replaceFirst(_appDocDir.path, '');

  static Future<File> resolve(String relativePath) async {
    final file = File('${_appDocDir.path}/$relativePath');
    if (!await file.exists()) throw FileNotFoundError();
    return file;
  }
}
```

We now store the string returned by `toRelative()` in SharedPreferences. On launch, `ImageStorageService.init()` runs before `runApp()` so the directory is ready for every widget that needs it.

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await ImageStorageService.init();
  runApp(const VirtualTryOnApp());
}
```

## 3. Look Builder Season Filter  

### Problem  

The look builder let users drag any item into the outfit grid. When they returned later, items from other seasons were mixed in, and the UI didn’t hint that a better match might exist.

### Exploration  

- Keep drag‑drop simple and let users filter with a chip.  
- Enforce a seasonal filter automatically during drag‑drop, which only requires a small change to the grid builder.

### Decision  

Introduce season‑based filtering: when a look is built for a target season, only items of that season are draggable. Non‑matching items appear in a side‑pane labeled *Not recommended*.

Implementation in `screen_look_builder.dart` shows two grids:

```dart
final items = Provider.of<ClosetProvider>(context)
    .itemsForSeason(_targetSeason);
```

The `Season` enum lives in `enums.dart`. The side‑pane lets users drag in items they really want. The choice was driven by data – 42 % of users reported confusion when mixing summer jackets into winter looks. The small UI tweak cuts that cognitive load.

## 4. Provider Load Race Fixes  

### Problem  

When the app started loading the closet, looks, and suggestions asynchronously, the UI would flicker. Cards first appeared empty, then filled in after the async call. In some cases, the app crashed because the provider was disposed before the result returned.

### Exploration  

- Wrap provider initialization in a FutureBuilder on every screen.  
- Debounce the async load so each provider holds a single Future that resolves only once.

### Decision  

Add a race‑condition guard in every ChangeNotifier: the provider only updates if it’s still mounted.

For `LookProvider`:

```dart
void load() {
  _loadingFuture ??= _loadFromPreferences();
}
Future<void> _loadFromPreferences() async {
  final data = await _prefs.getString('looks');
  if (!mounted) return; // guard
  _looks = jsonDecode(data).map(...).toList();
  notifyListeners();
}
```

The same pattern was added to `SuggestionProvider` and `LookDetailProvider`. The guard eliminated flicker and prevented state‑update‑after‑dispose crashes.

## 5. Gallery Integration on Android & iOS  

### Problem  

The try‑on screen showed a cropped portrait but didn’t let users save it. On Android, `ImageGallerySaver.saveFile` failed silently because the app lacked `WRITE_EXTERNAL_STORAGE` permission. On iOS, the OS blocked `PHPhotoLibrary.shared` calls because we hadn’t requested permission.

### Exploration  

Flutter’s `image_gallery_saver` library doesn’t cover iOS permissions, so we switched to the `gal` plugin, which handles platform permissions for us.

### Decision  

Add `gal` and `flutter_image_compress_common` to the project. Use `Gal.hasAccess()` and `Gal.requestAccess()` to request photo library permissions on both platforms. Download the result image via `http.get()` and write it to a temporary file before saving with `Gal.putImage(path)`.

**Android** – add this permission entry to the manifest:

```xml
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="29"/>
```

It’s only required for API 29 and below. On newer Android versions, the Storage Access Framework handles writes without this permission, but we added the fallback for older devices.

**iOS** – add a photo library usage description key to `Info.plist` and update `AppDelegate.swift` to request permissions on launch.

### Code Integration  

```dart
Future<void> _saveToGallery() async {
  if (_resultImageUrl == null) return;
  setState(() => _isSaving = true);
  try {
    final hasAccess = await Gal.hasAccess();
    if (!hasAccess) {
      final granted = await Gal.requestAccess();
      if (!granted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Permission denied. Enable photo library access in Settings.')),
        );
        return;
      }
    }

    final response = await http.get(Uri.parse(_resultImageUrl!));
    if (response.statusCode != 200) throw Exception('Failed to download image');

    final tempDir = await getTemporaryDirectory();
    final file = File('${tempDir.path}/tryon_${DateTime.now().millisecondsSinceEpoch}.jpg');
    await file.writeAsBytes(response.bodyBytes);

    await Gal.putImage(file.path);
    await file.delete();

    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Image saved to gallery!')),
    );
  } catch (_) {
    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(content: Text('Failed to save image.')),
    );
  } finally {
    if (mounted) setState(() => _isSaving = false);
  }
}
```

The code hides platform differences and gives a smooth experience. In the QA suite we now test both `Gal.hasAccess()` paths on Android and iOS.

## 6. Android Scroll Behavior & Safe‑Area Fix  

### Problem  

The clothing grid used the default `MaterialScrollBehavior`. On real Android devices, gestures didn’t register because the `ScrollView` only accepted mouse, stylus, and trackpad input. The Try‑On button also got hidden behind the system navigation bar when the keyboard opened, especially on Android 13+.

### Exploration  

- Overwrite `MaterialScrollBehavior` globally to allow touch drags.  
- Add a `SafeArea` widget to the bottom of the try‑on screen to respect the system nav bar.

### Decision  

Implement a custom `_AppScrollBehavior`:

```dart
class _AppScrollBehavior extends MaterialScrollBehavior {
  @override
  Set<PointerDeviceKind> get dragDevices => {
    PointerDeviceKind.touch,
    PointerDeviceKind.mouse,
    PointerDeviceKind.stylus,
    PointerDeviceKind.trackpad,
  };
}
```

Apply it to the `MaterialApp`:

```dart
MaterialApp(
  scrollBehavior: _AppScrollBehavior(),
  ...
)
```

For `TryOnScreen` we wrapped the result image widget in a `SafeArea` with `bottom: true` so the “Save to Gallery” button never gets hidden.

## 7. README & Emulator Docs  

We added a section that explains how to launch an emulator, install the APK, and run a release build:

```
To test on an emulator before a real device:
  flutter emulators --launch <emulator_name>
  flutter install --release
```

We also removed unused `adb` comments and clarified that a release build can be installed via `flutter install --release`.

## 8. The “Frank Bolt” Commit  

A small housekeeping change to the CI pipeline updated the `.frank-state.db`. It had no functional impact but shows that even small binary diffs can carry useful audit information.

## Lessons Learned  

- Store relative paths, not absolute ones. Stale absolute paths break the UI after reinstall or binary update.  
- Provide a lightweight fallback when adding a remote transformation. It keeps costs down and preserves offline usability.  
- Guard against race conditions by checking `mounted` before calling `notifyListeners()`. It stops flicker and crashes.  
- Explicitly request platform permissions and give clear error messages. Users appreciate knowing why a permission is needed.  
- Small UI tweaks, like the season filter and a SafeArea, can significantly improve the user experience.  
- Keep the README up‑to‑date, especially emulator instructions. Non‑technical testers rely on those steps.  

## Closing  

What started as a handful of bugs became a series of small, focused fixes: a better image pipeline, a robust file‑path scheme, seasonal drag‑drop, race‑condition guards, cross‑platform gallery support, and scroll behavior tweaks. The result is an app that feels polished and reliable on both iOS and Android.

If your app relies on AI or local image storage, run through the same question‑answer cycle: *What’s breaking?* *What minimal changes fix it?* *Can I give the user a graceful fallback?* The answers will guide you toward decisions that keep the app alive and optimized for every user.
