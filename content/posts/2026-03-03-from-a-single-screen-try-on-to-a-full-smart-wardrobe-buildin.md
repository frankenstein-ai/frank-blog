+++  
date = '2026-03-03T13:01:07-03:00'  
draft = false  
title = 'From a single‑screen try‑on to a full smart wardrobe: building an offline‑first Flutter app'  
+++

## The problem that started it all

I started this project with one clear goal: let you upload a selfie and a clothes photo, and pull back a fresh image of you wearing that item. The first release was a single screen that hooked up image picking, the Replicate API, and a basic viewer. It worked, but it also felt like a toy.

The real issues were obvious:

* **No wardrobe –** Once the photo was gone, the next try‑on meant re‑uploading or dragging the file from the gallery again.
* **No styling –** I dreamed of a virtual closet where you could mix pieces, match colors, and get fashion suggestions; none of that happened.
* **No offline mode –** The UI only worked when an internet connection let Replicate finish the request.

Those frustrations pushed us to do a full re‑architecture. The goal was to turn the app into a personal wardrobe that lived on the device.

## A jump to a local‑first microworld

We made a two‑part decision:

1. **Keep everything that doesn’t need an API local** – the clothing catalog, the list of outfits, and the AI suggestions all lived on the device.
2. **Make the AI try‑on optional** – the “Try‑On” tab still hit Replicate; the rest of the app could run smoothly offline.

With that in mind we swapped the single `HomePage` for a shell that hides four independent stacks:

```
AppShell
 ├── Closet tab
 ├── Looks tab
 ├── Suggestions tab
 └── Try‑On tab
```

The shell uses an `IndexedStack`, so the state of each tab stays alive, even when it’s not the active view. That was the first commit that replaced `home_page.dart` with `app_shell.dart`.

## Choosing state management

Flutter offers plenty of state‑management patterns. After a quick experiment and a bit of trade‑off talk, we settled on a classic Provider approach:

* Each feature has its own `ChangeNotifier` that knows how to load, add, and remove items.
* A singleton is created at app launch (`ChangeNotifierProvider(create: (_) => ClosetProvider()..load())`). The call to `..load()` pulls the persisted data straight away.
* The provider exposes a simple interface that talks to in‑memory collections and `SharedPreferences`.

Why did we pick Provider over Riverpod or Bloc? Roughly:

```
Provider – simple, integrates with widgets, little boilerplate.
Riverpod – more powerful, but too much learning curve for this sprint.
Bloc – great for reactive streams, but our data set is small and mostly list edits.
```

That choice let us add `provider_closet.dart`, `provider_look.dart`, and `provider_suggestion.dart` quickly. The `main.dart` now wraps the `MaterialApp` inside a `MultiProvider`.

## Persisting data with SharedPreferences

At first glance `SharedPreferences` feels too primitive for a clothing catalog. But the data is lightweight: each item is a JSON‑serializable map with fields like `id`, `imagePath`, `category`, `seasons`, `tags`, and `notes`. One JSON string per list is all we need.

The provider reads that string on startup, deserialises it, and fills the in‑memory list:

```dart
class ClosetProvider extends ChangeNotifier {
  final List<ClothingItem> _items = [];
  List<ClothingItem> get items => List.unmodifiable(_items);

  Future<void> load() async {
    final prefs = await SharedPreferences.getInstance();
    final raw = prefs.getString('closet_items');
    if (raw != null) {
      final list = jsonDecode(raw) as List<dynamic>;
      _items.addAll(list.map((e) => ClothingItem.fromJson(e)));
    }
    notifyListeners();
  }

  Future<void> add(ClothingItem item) async {
    _items.add(item);
    await _persist();
    notifyListeners();
  }
}
```

The same pattern works for Looks and Suggestions. The separate services (`service_closet.dart`, `service_look.dart`, `service_suggestion.dart`) encapsulate business logic and persistence, keeping providers lean and testable.

An early snag was that large lists could exceed the max string size in `SharedPreferences`. The workaround was to persist only when a change happened and keep a cache in memory. For the MVP that’s fine. If we ever hit scale, we’ll move to Hive or Sqflite, but for now the repo stays tight.

## Adding domain logic: enums and mapping

Switching from Replicate’s internal `GarmentCategory` to a UI‑centric `ClothingCategory` made the code easier to reason about. Here’s the enum we introduced:

```dart
enum ClothingCategory {
  top('Top'),
  bottom('Bottom'),
  dress('Dress'),
  shoes('Shoes'),
  accessory('Accessory'),
  outerwear('Outerwear');
}
```

A helper moves our local enum into the one expected by Replicate:

```dart
GarmentCategory? toGarmentCategory() {
  return switch (this) {
    ClothingCategory.top || ClothingCategory.outerwear => GarmentCategory.upperBody,
    ClothingCategory.bottom => GarmentCategory.lowerBody,
    ClothingCategory.dress => GarmentCategory.dresses,
    ... // other mappings
  };
}
```

With that bridge, the rest of the app can stay agnostic of the backend’s terminology. That small tweak kept the boundaries clean and made the AI calls feel like just another feature.

---

The result? A lightweight, responsive app that behaves like a digital wardrobe. You can add clothes, mix looks, and get AI‑powered recommendations, all while staying fully functional offline. The biggest win was learning how to keep everything that can live on the device on the device, and how a few well‑chosen patterns can save a ton of friction.

Ready to try it out? Just drop your photo in the Closet, pick an item, and hit Try‑On. If the network’s spotty, you’ll still see the outfit instantly. The next step for me is to add a small suggestion engine that learns from what you pair, but that’s a story for another post.
