# Mobile App — Design & Build Specification

Framework: **Flutter** (Dart)
Platform target: Android (APK) + iOS
Style: Dark mode, minimalist, bright accent colour, fewest clicks to get things done.

---

## Visual Design

### Colour Palette

| Role            | Value (placeholder — finalise before build) |
|-----------------|----------------------------------------------|
| Background      | `#0D0D0D`                                    |
| Surface         | `#1A1A1A`                                    |
| Surface variant | `#242424`                                    |
| Accent (primary)| `#7C3AED` (violet — adjust to brand)         |
| On-accent       | `#FFFFFF`                                    |
| Text primary    | `#F5F5F5`                                    |
| Text secondary  | `#888888`                                    |
| Destructive     | `#EF4444`                                    |
| Archive         | `#F59E0B`                                    |

All colours are set once in `lib/theme/app_theme.dart` and referenced throughout. No hardcoded colours in widgets.

### Typography

- Font: **Inter** (Google Fonts)
- Task title: `16px`, medium weight
- Labels / metadata: `12px`, secondary colour
- Due date overdue: accent colour

### Motion

- List reorder: standard drag handle animation
- Swipe to archive: slide with amber overlay, undo snackbar (4 s)
- Screen transitions: fade + slight slide (< 200 ms)

---

## App Structure

```
lib/
├── main.dart
├── theme/
│   └── app_theme.dart
├── services/
│   ├── api_service.dart        # HTTP client, JWT attach
│   └── auth_service.dart       # Login, token storage
├── models/
│   ├── task.dart
│   └── label.dart
├── screens/
│   ├── login/
│   │   └── login_screen.dart
│   ├── tasks/
│   │   ├── task_list_screen.dart
│   │   ├── task_detail_screen.dart
│   │   └── widgets/
│   │       ├── task_tile.dart
│   │       └── add_task_bar.dart
│   ├── archive/
│   │   └── archive_screen.dart
│   └── settings/
│       └── settings_screen.dart
└── shared/
    └── widgets/
        └── label_chip.dart
```

---

## Navigation

Bottom tab bar with three tabs:

| Tab      | Icon              | Screen          |
|----------|-------------------|-----------------|
| Tasks    | check_circle      | Task list       |
| Archive  | inventory_2       | Archived tasks  |
| Settings | settings          | Settings        |

No drawer. No hamburger. Tabs are always visible.

---

## Login Screen

- Full-screen dark background with a **subtle brand image** (abstract/geometric, placed behind a frosted overlay).
- Centred **logo** (SVG asset: `assets/logo.svg`) above the form.
- Email + password text fields, both dark-surface styled.
- Single **"Sign in"** button in accent colour, full width, rounded.
- No "forgot password", no sign-up — single user only.
- On success → replace navigation stack with main tab shell.
- JWT stored in `SharedPreferences` for now; key: `auth_token`. *(Migrate to `flutter_secure_storage` in a later sprint.)*

---

## Task List Screen

### Layout

- `ReorderableListView` — user drags to reorder; order is persisted to the API via `sort_order` field.
- Each row is a `TaskTile` (see below).
- Sticky **label filter bar** beneath the app bar — horizontal scroll of chips: `All`, then one per label. Tapping filters in-place, no navigation.
- **Floating `+` button** (bottom-right, accent colour). Taps opens an inline bottom sheet with a single text field — type the title and press Enter / "Add". Task is created immediately with `status: pending`, due date defaulting to **7 days from today**.

### TaskTile

```
[ drag handle ] [ label dot ]  Title text               [ due date ]
                               secondary line: label name
```

- Tap → navigate to `TaskDetailScreen`.
- **Swipe left → Archive**. Amber slide-away animation. Snackbar: "Archived — Undo" (4 s). Undo calls unarchive endpoint.
- Long-press activates reorder mode (drag handle becomes prominent).
- Overdue due dates shown in accent colour.

### Empty State

Centred illustration + "No tasks. Hit + to add one."

---

## Task Detail Screen

Full screen, back arrow to return to list.

All fields are **inline-editable** — tap any field to edit it in place; changes auto-save on blur (no explicit Save button).

| Field       | Control                          | Default            |
|-------------|----------------------------------|--------------------|
| Title       | Tappable text → text field       | —                  |
| Label       | Tappable chip → bottom sheet picker | none            |
| Status      | Segmented control: Pending / In Progress / Done | pending |
| Due date    | Tappable date → date picker      | today + 7 days     |
| Description | Tappable multiline text → editor | —                  |

- **Archive button** at the bottom (amber, outlined). Archives and pops back to list.
- No delete on this screen — deletion is not exposed in the mobile app (handled via web if needed).

---

## Labels

Fixed set — displayed as coloured chips. Stored as a string enum on the task.

| Label          | Slug             | Chip colour  |
|----------------|------------------|--------------|
| Start With     | `start_with`     | `#6366F1`    |
| Potential User | `potential_user` | `#06B6D4`    |
| Outreach       | `outreach`       | `#10B981`    |
| Features       | `features`       | `#F59E0B`    |
| House          | `house`          | `#EC4899`    |

A task has exactly zero or one label.

---

## Archive Screen

- Same visual style as task list.
- Tasks sorted by `archived_at` descending (most recently archived first).
- No reorder.
- **Swipe right → Unarchive**. Green slide-away. Snackbar: "Restored".
- Shows `archived_at` date beneath the title in secondary colour.

---

## Settings Screen

### Stale Task Notifications

- Toggle: **"Notify me about overdue tasks"** (default: on).
- Picker: **"Remind me after"** — options: 1 day, 2 days, 3 days, 1 week overdue.
- Implementation: local scheduled notifications via `flutter_local_notifications`.
  - On app foreground, check tasks whose `due_date < now - threshold` and schedule a local notification if none is already pending for that task.
  - No push infrastructure required — all local.

### About

- App version (pulled from `pubspec.yaml` at build time via `package_info_plus`).
- "Download latest APK" link — points to `{WEB_URL}/task-tracker-vX.X.X.apk`.

---

## API Integration

Base URL configured in `lib/services/api_service.dart`:

```dart
const String _baseUrl = String.fromEnvironment('API_URL', defaultValue: 'http://10.0.2.2:3000');
```

`10.0.2.2` is the Android emulator loopback to host. Override at build time:

```bash
flutter build apk --dart-define=API_URL=https://api.yourdomain.com
```

All requests attach `Authorization: Bearer <token>` from `AuthService`.
On `401` response → clear token → push login screen.

### Backend API (mobile support)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/login` | Body: `{ "email", "password" }`. Returns `{ "token": "JWT" }`. |
| GET | `/labels` | Returns fixed label set: `[{ "slug", "name", "color" }, ...]`. |
| GET | `/tasks` | List tasks. Query: `?archived=true` for archive; `?label=slug` to filter by label. |
| POST | `/tasks` | Create task. Body: `title` (required); optional: `description`, `status`, `label`, `due_date`, `sort_order`. Defaults: `status: "pending"`, `due_date`: 7 days from now. |
| GET | `/tasks/:id` | Get one task. |
| PATCH | `/tasks/:id` | Update task (partial). Fields: `title`, `description`, `status`, `label`, `due_date`, `sort_order`. |
| PATCH | `/tasks/reorder` | Bulk reorder. Body: `[{ "id": "uuid", "sort_order": 0 }, ...]`. |
| POST | `/tasks/:id/archive` | Set `archived_at` (for swipe-to-archive / undo). |
| POST | `/tasks/:id/unarchive` | Clear `archived_at`. |
| DELETE | `/tasks/:id` | Delete task (e.g. from web; mobile does not expose delete). |

---

## Data Model (Flutter side)

```dart
class Task {
  final String id;
  String title;
  String? description;
  TaskStatus status;       // pending | in_progress | done
  TaskLabel? label;        // start_with | potential_user | outreach | features | house
  DateTime? dueDate;
  int sortOrder;           // user-defined ordering
  DateTime? archivedAt;    // null = active
  final DateTime createdAt;
  DateTime updatedAt;
}
```

---

## Build

See root `Makefile`. The release APK is built with:

```bash
make apk VERSION=1.2.0
```

Output: `web/public/task-tracker-v1.2.0.apk`
The web SPA serves it as a static download link.

### pubspec.yaml version

Keep `version:` in `mobile/flutter/pubspec.yaml` in sync with the `VERSION` passed to `make apk`. The Makefile patches it automatically before building.

---

## Assets

```
mobile/flutter/assets/
├── logo.svg
└── login_bg.png        # dark abstract/geometric background
```

Register in `pubspec.yaml`:

```yaml
flutter:
  assets:
    - assets/logo.svg
    - assets/login_bg.png
```

---

## Dependencies (pubspec.yaml)

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.0
  shared_preferences: ^2.2.3
  flutter_local_notifications: ^17.0.0
  package_info_plus: ^8.0.0
  google_fonts: ^6.2.1
  intl: ^0.19.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0
```
