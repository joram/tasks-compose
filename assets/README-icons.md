# App icon

- **task-tracker-app-icon.png** — Original design (checkmark-in-shape, task-tracker theme).
- **task-tracker-app-icon-square.png** — Square 1024×1024 version for store and launcher.

To use in the Flutter app you can:

1. **flutter_launcher_icons**  
   Add to `pubspec.yaml` and run:
   ```yaml
   dev_dependencies:
     flutter_launcher_icons: ^0.13.1
   flutter_launcher_icons:
     android: true
     ios: true
     image_path: "assets/task-tracker-app-icon-square.png"
   ```
   Then from `mobile/flutter`: `dart run flutter_launcher_icons`.

2. **Manual**  
   Resize `task-tracker-app-icon-square.png` to Android mipmap sizes (e.g. mdpi 48, hdpi 72, xhdpi 96, xxhdpi 144, xxxhdpi 192) and replace the `ic_launcher.png` files in `android/app/src/main/res/mipmap-*`.
