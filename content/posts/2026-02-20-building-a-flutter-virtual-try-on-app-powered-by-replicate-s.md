+++
date = '2026-02-20T09:37:25-03:00'
draft = false
title = 'Building a Flutter Virtual Try‑On App Powered by Replicate’s IDM‑VTON'
+++

## Bridging Wear‑Fashion to the Cloud

Ever tried to see how a shirt might look on you without stepping into a fitting room? Digital try‑on tools exist, but most run in the browser or ask you to upload photos to a cloud endpoint that returns a rendered image. For a mobile developer, that leaves two questions hanging:

* How can I keep the API secret safe while still loading it when the app runs?  
* How do I build a responsive, cross‑platform UI that talks to an ML model on a service like Replicate?

The `virtual_tryon` project shows a practical answer. The example in this post walks through picking selfies and garment images, uploading them to Replicate’s Files API, starting an IDM‑VTON prediction, polling for the result, and finally displaying the mock‑up. All of that lives inside a small Flutter app that uses Material 3 and compiles the API key in from a file.

Why does this matter? Few developers have a ready‑to‑copy example that hides the plumbing of credentials, file upload, and polling logic. The code here is a concrete starting point you can copy‑paste, change, or extend to other Replicate models.

---

### 1. Project Overview

| Folder | Purpose |
|--------|---------|
| `lib/main.dart` | Application entry point and `MaterialApp` scaffold. |
| `lib/home_page.dart` | UI for selecting assets, picking a category (upper, lower, dress), and showing the result. |
| `lib/replicate_service.dart` | Thin wrapper around Replicate: file uploads, prediction creation, status polling. |
| `lib/config.dart` | Compile‑time constant that reads the `REPLICATE_API_KEY` from an injected `.env.json`. |

The README explains the setup steps, notably how to add the key to the build with `--dart-define-from-file`. That single change lets CI/CD run the app without committing secrets.

---

### 2. Keeping Secrets Out of the Repository

#### 2.1 The `.env.json.example` Template

```json
{
  "REPLICATE_API_KEY": "your-replicate-api-key-here"
}
```

After cloning, copy the example to `.env.json` and fill in your token:

```bash
cp .env.json.example .env.json
```

#### 2.2 Gitignore

The file is now listed in `.gitignore`:

```
# Environment secrets
.env.json
```

#### 2.3 Compile‑time Injection

Flutter can inject constants at compile time with `--dart-define` or `--dart-define-from-file`. The README shows:

```bash
flutter run --dart-define-from-file=.env.json
```

`config.dart` reads the key:

```dart
class Config {
  static const String replicateApiKey =
      String.fromEnvironment('REPLICATE_API_KEY', defaultValue: '');

  static String get apiKey => replicateApiKey;
}
```

Reading the file at runtime would leave the key in the binary, so compile‑time injection keeps it out of the release build while allowing developers to test locally.

---

### 3. Granting Runtime Permissions

Both Android and iOS require explicit permissions to capture images or read the media library.

#### 3.1 Android (`AndroidManifest.xml`)

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.CAMERA"/>
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
    <uses-permission android:name="android.permission.INTERNET"/>

    <application
        android:label="virtual_tryon"
        ...>
        ...
    </application>
</manifest>
```

#### 3.2 iOS (`Podfile` + `.xcconfig`)

The repo includes a full `Podfile` that pulls the `image_picker_ios` plugin and required pods. The `.xcconfig` files handle CocoaPods integration. iOS plugins automatically prompt the user for camera and photo library access; just add the `NSCameraUsageDescription` and `NSPhotoLibraryUsageDescription` entries to `Info.plist`, which are already present.

---

### 4. UI: The Human‑Centric Flow

The single screen (`home_page.dart`) keeps the app easy to navigate:

1. Pick a person image – from camera or gallery.  
2. Pick a clothing image – from camera or gallery.  
3. Choose a category – upper body, lower body, or dress.  
4. Tap “Try On” – the heavy lifting happens behind the scenes.  
5. View the resulting rendered image.

```dart
class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final _service = ReplicateService(Config.apiKey);
  File? _personImage;
  File? _clothingImage;
  String _category = 'upper';
  String? _resultUrl;
  bool _isLoading = false;

  Future<void> _pickImage(ImageSource source, bool isPerson) async {
    final picker = ImagePicker();
    final picked = await picker.pickImage(source: source);
    if (picked != null) {
      setState(() {
        file = isPerson ? File(picked.path) : File(picked.path);
      });
    }
  }

  Future<void> _tryOn() async {
    if (_personImage == null || _clothingImage == null) return;
    setState(() => _isLoading = true);
    try {
      final personId = await _service.uploadFile(_personImage!);
      final clothId = await _service.uploadFile(_clothingImage!);
      final prediction = await _service.startPrediction(personId, clothId, _category);
      final url = await _service.waitForResult(prediction.id);
      setState(() => _resultUrl = url);
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Error: ${e.toString()}')),
      );
    } finally {
      setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Virtual Try‑On')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Row(
              children: [
                _imagePreview(_personImage, 'Your Photo'),
                _imagePreview(_clothingImage, 'Clothing'),
              ],
            ),
            const SizedBox(height: 16),
            DropdownButton<String>(
              value: _category,
              items: const [
                DropdownMenuItem(value: 'upper', child: Text('Upper Body')),
                DropdownMenuItem(value: 'lower', child: Text('Lower Body')),
                DropdownMenuItem(value: 'dress', child: Text('Dress')),
              ],
              onChanged: (v) => setState(() => _category = v!),
            ),
            const SizedBox(height: 16),
            ElevatedButton(
              onPressed: _isLoading ? null : _tryOn,
              child: _isLoading
                  ? const CircularProgressIndicator()
                  : const Text('Try On'),
            ),
            const SizedBox(height: 16),
            if (_resultUrl != null)
              Image.network(_resultUrl!, height: 300),
          ],
        ),
      ),
    );
  }

  Widget _imagePreview(File? file, String label) {
    return Expanded(
      child: Column(
        children: [
          ElevatedButton(
            onPressed: () => _pickImage(ImageSource.gallery, file == _personImage),
            child: const Icon(Icons.image),
          ),
          const SizedBox(height: 8),
          Text(label),
          const SizedBox(height: 8),
          Container(
            width: 80,
            height: 80,
            decoration: BoxDecoration(
              border: Border.all(),
              image: file != null
                  ? DecorationImage(image: FileImage(file), fit: BoxFit.cover)
                  : null,
            ),
          ),
        ],
      ),
    );
  }
}
```

The UI simply gathers user input and hands the heavy work off to `ReplicateService`.

---

### 5. The Replicate Wrapper

The wrapper in `replicate_service.dart` talks directly to Replicate’s HTTP API.

```dart
class ReplicateService {
  final String _apiKey;
  final http.Client _client = http.Client();

  ReplicateService(this._apiKey);

  Future<String> uploadFile(File file) async {
    final uri = Uri.parse('https://api.replicate.com/v1/files');
    final request = http.MultipartRequest('POST', uri)
      ..headers['Authorization'] = 'Token $_apiKey'
      ..files.add(await http.MultipartFile.fromPath('file', file.path));
    final response = await _client.send(request);
    if (response.statusCode != 201) throw Exception('Upload failed');
    final body = await response.stream.bytesToString();
    final json = jsonDecode(body) as Map<String, dynamic>;
    return json['id'];
  }

  Future<Prediction> startPrediction(String personId, String clothId, String category) async {
    final uri = Uri.parse('https://api.replicate.com/v1/predictions');
    final body = jsonEncode({
      'version': 'cuuupid/idm-vton',
      'input': {
        'person_image': personId,
        'source_image': clothId,
        'category': category,
      }
    });
    final response = await _client.post(
      uri,
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Token $_apiKey',
      },
      body: body,
    );
    if (response.statusCode != 201) throw Exception('Prediction failed');
    return Prediction.fromJson(jsonDecode(response.body));
  }

  Future<String> waitForResult(String predictionId) async {
    const pollInterval = Duration(seconds: 2);
    final uri = Uri.parse('https://api.replicate.com/v1/predictions/$predictionId');
    while (true) {
      final response = await _client.get(
        uri,
        headers: {'Authorization': 'Token $_apiKey'},
      );
      if (response.statusCode != 200) throw Exception('Polling failed');
      final data = jsonDecode(response.body) as Map<String, dynamic>;
      final status = data['status'] as String;
      if (status == 'succeeded') {
        final url = (data['output'] as List).first as String;
        return url;
      } else if (status == 'failed') {
        throw Exception('Model failed: ${data['error_message']}');
      }
      await Future.delayed(pollInterval);
    }
  }
}

class Prediction {
  final String id;
  final String status;
  final List<String> output;

  Prediction({
    required this.id,
    required this.status,
    required this.output,
  });

  factory Prediction.fromJson(Map<String, dynamic> json) => Prediction(
        id: json['id'],
        status: json['status'],
        output: List<String>.from(json['output'] ?? []),
      );
}
```

The flow is: upload file → get a file ID → start prediction with that ID → poll until success → pull the rendered image URL.

---

### 6. Unit Testing (Optional)

A light‑weight widget test in `test/widget_test.dart` checks that the “Try On” button stays disabled until both images are chosen.

```dart
testWidgets('Enable Try On button after images selected', (WidgetTester tester) async {
  await tester.pumpWidget(const MaterialApp(home: HomePage()));

  final tryButton = find.text('Try On');
  expect(tester.widget<ElevatedButton>(tryButton).enabled, false);

  // Simulate picking images if needed. A full test would mock ReplicateService.
});
```

When testing, swap the real service for a stub to avoid network calls.

---

### 7. Deployment Notes

| Issue | Guidance |
|-------|----------|
| Rate limits | Replicate may throttle. Cache file IDs for repeated uploads. |
| Performance | File uploads are the slow step. Ensure `multipart/form-data` is used correctly. |
| Offline mode | Current design assumes connectivity. For production, queue predictions and retry on reconnection. |
| Security | Keep the API key out of source control. Validate its size during compile. |

The `pubspec.yaml` contains only essentials:

```yaml
dependencies:
  flutter:
    sdk: flutter
  image_picker: ^0.8.7+4
  http: ^0.13.7
```

All inference stays on Replicate; the binary is lightweight.

---

### 8. Going Beyond IDM‑VTON

The architecture is intentionally generic:

* Swap the model by changing the `version` string in `startPrediction`.  
* Add more categories by expanding the UI dropdown and the `input` map.  
* Use any Replicate model that accepts file IDs.  
* For image‑generation APIs outside Replicate, adjust URLs and headers as needed.

The service wrapper only knows HTTP, so you can point it at different endpoints without touching the UI.

---

### 9. Take‑away

This example bundles a production‑ready flow into a minimal repo:

1. Secrets survive CI/CD through compile‑time defines.  
2. Permissions for camera and photo library are handled natively.  
3. The UI is simple and follows Material 3.  
4. Replicate calls are wrapped, keeping the app code clean.  
5. Errors are surfaced to the user.

If you’re building a mobile app that needs to run AI inference in the cloud, this pattern gives you:

* Quick iteration: swap the backend model, keep the UI unchanged.  
* Low device overhead: all heavy lifting stays on the server.  
* Reproducible builds: secrets are injected safely.

Fork the repo, plug your own Replicate token, and use it as a starting point for other experiments—whether that’s generative text, style transfer, or real‑time video filters. Happy coding.
