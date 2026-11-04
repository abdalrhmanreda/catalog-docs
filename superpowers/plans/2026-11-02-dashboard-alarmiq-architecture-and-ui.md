# Dashboard: Alarmiq Architecture and UI Consistency Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move `katalog_dashboard` to the architecture used by Alarmiq with no behavior change, then make every screen use the same tokens and shared components.

**Architecture:** Feature-first code under `lib/features/<feature>/{data,logic,ui}`, with Cubits from `flutter_bloc` and dependency injection through GetIt. Cross-cutting code lives in `lib/core` and `lib/config`. Riverpod and the new stack coexist through a temporary bridge, so the app and its tests stay green after each task.

**Tech Stack:** Flutter web, `flutter_bloc`, `get_it`, `go_router`, `dio`, `fl_chart`, `flutter_test`.

**Spec:** The user's UI consistency brief (sections 1 to 8) and two decisions: the target project is `katalog_dashboard`, and the architecture move happens before the UI work. Reference code: `/Users/abdalrhmanreda/Codex/Flutter/alramiq` (read `CLAUDE.md`, `lib/core/di/dependency_injection.dart`, and `lib/features/timer`).

## Global Constraints

- Repository: `/Users/abdalrhmanreda/Codex/Backend/Django/katalog_dashboard`. Branch: `refactor/alarmiq-architecture`.
- Tasks 2 to 8 change no behavior, copy, or visuals. Test assertions stay the same. Only setup code, imports, and renamed class names may change in tests.
- The test count never drops below 245. `flutter analyze` and `flutter test` pass before every commit.
- Do not add `flutter_screenutil`, `easy_localization`, or `equatable`. This is a desktop-first web console, and it has no translations in scope.
- Keep the `K*` token names (`KSpace`, `KType`, `KRadius`). Renaming them adds churn with no benefit.
- Keep the visual direction from the mobile app: navy, Tajawal, light by default. The UI tasks standardize and do not redesign.
- Format only changed files with `dart format`.
- Keep the same API paths, permission slugs, and `X-Platform: web` header.

## Decisions

The user decided on 2026-11-02 that every screen uses Cubit, dialogs included.

1. **Every screen's state lives in a Cubit.** Detail and dashboard screens get their own feature Cubit. Every list screen uses `TableCubit<T>` in `lib/core/logic/table_cubit.dart`, which takes over the paging, search, filter, loading, and error state now in `DataTablePage`. `DataTablePage` creates it, and the feature passes a repo method as `fetch`.
2. **Every dialog submits through `SubmitCubit`** in `lib/core/logic/submit_cubit.dart`. It holds `idle`, `busy`, `success`, and `failure` with the `ApiError`. Text controllers stay in the widget's `State`, because Flutter ties them to the widget lifetime.
3. **Screens are named `*_screen.dart` and classes `*Screen`,** matching Alarmiq. Widgets go in `ui/widgets/`, and models go in `data/models/`.

## Target Structure

| Today | After |
|---|---|
| `lib/theme/tokens.dart` | `lib/config/colors/app_colors.dart` (`KColors`, `KTone`, `KPalette`), `lib/config/themes/app_tokens.dart` (`KType`, `KRadius`, `KSpace`, `KLayout`, `KMotion`) |
| `lib/theme/theme.dart` | `lib/config/themes/app_theme.dart` |
| `lib/theme/theme_mode.dart` | `lib/features/theme/logic/theme_cubit.dart` |
| `lib/core/router.dart` | `lib/config/routes/app_router.dart` |
| `lib/core/api/*` | `lib/core/network/*` |
| `lib/core/auth/token_store.dart` | `lib/core/services/token_store.dart` |
| `lib/core/auth/auth_controller.dart` | `lib/features/auth/logic/auth_cubit.dart`, `auth_state.dart`, `lib/features/auth/data/models/staff_profile.dart`, `lib/features/auth/data/repos/auth_repo.dart` |
| `lib/features/login/*` | `lib/features/auth/ui/screens/*` |
| `lib/shell/*` | `lib/features/main_layout/ui/*`, `lib/features/main_layout/data/sections.dart`, `lib/features/main_layout/logic/health_cubit.dart` |
| `lib/widgets/*` | `lib/core/widgets/*` |
| `lib/features/x/x_page.dart`, `x_models.dart` | `lib/features/x/ui/screens/x_screen.dart`, `lib/features/x/data/models/x_models.dart`, `lib/features/x/data/repos/x_repo.dart` |

## Review Focus

1. Leaving a screen while its request is still running must not throw "Cannot emit new states after calling close". The old code guarded with `mounted`. Test in Task 4.
2. Going from `/users/A` to `/users/B` must load B. A `BlocProvider` creates its Cubit once, so this can show stale data if the key is wrong. Test in Task 4.
3. Signing out and then signing in as another account must not keep the first account's permissions or data, because `AuthCubit` is a singleton. Test in Task 3.
4. Reloading on a deep link while the session restores must return to that link after restore. The router's refresh now comes from a Cubit stream. Test in Task 3.
5. Long Arabic names and long emails must not overflow the new shared `SectionCard` and `FactRow`. Test in Task 9.

---

## Part 1: Architecture

### Task 1: Baseline branch

**Files:** every currently uncommitted file (36 entries in `git status`), including the user's own rebrand edits and the App updates work.

- [ ] **Step 1:** Run `flutter test`. Expected: `All tests passed!` with 245 tests.
- [ ] **Step 2:** Run `git switch -c refactor/alarmiq-architecture`.
- [ ] **Step 3:** Run `git add -A`, confirm that `build/` and secrets are not staged, and commit with the subject `feat: apply mobile app theme and add app updates and user forms`.

### Task 2: Move files with no logic change

**Files:** the Target Structure table, except the auth and theme logic split (Task 3) and feature repos (Tasks 4 to 7). Move auth files as they are for now, and move every test import to match.

**Interfaces:** Produces the final paths. Later tasks only add `logic/` and `data/repos/` files.

- [ ] **Step 1:** Move files with `git mv` and split `tokens.dart` into the two files named above. Add `library` re-export only if an import cycle forces it.
- [ ] **Step 2:** Rename `*Page` classes and files to `*Screen` under `ui/screens/`, and update tests.
- [ ] **Step 3:** Run `flutter analyze && flutter test`. Expected: no issues, 245+ tests pass.
- [ ] **Step 4:** Confirm that `git diff -M --stat` reports moves, not rewrites. Commit with `refactor: move dashboard files to feature-first layout`.

### Task 3: DI, auth, theme, router

**Files:**
- Create: `lib/core/di/dependency_injection.dart`, `lib/core/di/riverpod_bridge.dart` (temporary), `lib/features/auth/logic/auth_cubit.dart`, `auth_state.dart`, `lib/features/auth/data/repos/auth_repo.dart`, `lib/features/theme/logic/theme_cubit.dart`, `test/support/test_di.dart`
- Modify: `lib/main.dart`, `lib/app.dart`, `lib/config/routes/app_router.dart`, `lib/core/network/api_client.dart`, `test/support/app_harness.dart`, `test/support/fake_auth.dart`, `test/core/auth_controller_test.dart`, `test/core/api_client_test.dart`

**Interfaces:**
- Produces: `final getIt = GetIt.instance;` and `void setupDependencies({Dio? dio, String baseUrl = kApiBaseUrl})`, which registers singletons for `TokenStore`, `Dio`, `ApiClient`, `AuthRepo`, `AuthCubit`, `ThemeCubit`, and `GoRouter`.
- Produces: `class AuthCubit extends Cubit<AuthState>` with `Future<void> signIn(String, String)`, `Future<void> restore()`, `Future<void> signOut({bool revoke = true})`, and `bool can(String)`. `AuthState` keeps its current constructors and `can`.
- Produces: `class ThemeCubit extends Cubit<ThemeMode>` with `void toggle()`. It starts at `ThemeMode.light`.
- Produces: `Future<void> setupTestDi({required FakeAdapter adapter, required AuthState auth})` and `tearDown` reset through `getIt.reset()`.
- Bridge: `apiClientProvider`, `authControllerProvider`, and `themeModeProvider` read from GetIt and mirror Cubit state, so unmigrated screens keep working until Task 8 deletes the bridge.
- The router refreshes from a `ChangeNotifier` that listens to `AuthCubit.stream`. `ApiClient.onSessionExpired` calls `getIt<AuthCubit>().signOut(revoke: false)`.

- [ ] **Step 1: Write the failing tests** in `test/core/auth_controller_test.dart`:

```dart
test('signing in as another account drops the first permissions', () async {
  // sign in as A holding users.read, sign out, sign in as B without it
  expect(cubit.can('users.read'), isFalse);
});

testWidgets('a deep link survives session restore', (tester) async {
  // start at /users/1 with a stored refresh token and auth unknown
  expect(routerPath(), '/users/1');
});
```

- [ ] **Step 2:** Run `flutter test test/core/auth_controller_test.dart`. Expected: compile failure because `AuthCubit` is not defined.
- [ ] **Step 3:** Implement the interfaces above. Port every existing `AuthController` test to `AuthCubit`, with the same names and assertions.
- [ ] **Step 4:** Run `flutter analyze && flutter test`. Expected: all pass.
- [ ] **Step 5:** Commit with `refactor(auth): replace auth and theme providers with cubits and GetIt`.

### Task 4: Migrate users and overview

These set the pattern for later tasks.

**Files:**
- Create: `lib/features/users/data/repos/users_repo.dart`, `lib/features/users/logic/user_detail_cubit.dart`, `user_detail_state.dart`, `plan_options_cubit.dart`, `lib/features/overview/data/repos/overview_repo.dart`, `lib/features/overview/logic/overview_cubit.dart`, `overview_state.dart`
- Modify: the users and overview screens and `test/features/users_page_test.dart`, `test/features/overview_page_test.dart`

**Interfaces:**
- Produces the pattern: `class XRepo { XRepo(this._api); }` holds every API call for the feature with typed returns. Cubits take repos through their constructor and are registered as GetIt factories. Screens create them with `BlocProvider(create: (_) => getIt<X>()..load(id))`.
- Every Cubit guards async results with `if (isClosed) return;` before `emit`.
- `PlanOptionsCubit` replaces `activePlansProvider` and `planFilterOptionsProvider`.

- [ ] **Step 1: Write the failing tests** in `test/features/users_page_test.dart`:

```dart
testWidgets('leaving user detail during its request does not throw', (t) async {
  // hold the GET open, navigate to /users, complete the GET
  expect(t.takeException(), isNull);
});

testWidgets('moving from one user to another loads the second', (t) async {
  // go /users/A, then go /users/B
  expect(find.text('Omar Said'), findsWidgets);
  expect(find.text('Nour Hassan'), findsNothing);
});
```

- [ ] **Step 2:** Run the file. Expected: the moving test fails, or the file does not compile until the Cubit exists.
- [ ] **Step 3:** Implement. Key the detail `BlocProvider` by the route id.
- [ ] **Step 4:** Run `flutter analyze && flutter test`. Expected: all pass.
- [ ] **Step 5:** Commit with `refactor(users): move users and overview to repos and cubits`.

### Task 5: Migrate messages, deliveries, inactivity, plans, app updates

**Files:** one `data/repos/*_repo.dart` per feature. Create `MessageDetailCubit` and `AppUpdatesCubit` (the load state of `AppUpdatesScreen`). Messages, deliveries, inactivity, and plans lists use the shared table with no Cubit.

**Interfaces:** Consumes the Task 4 pattern.

- [ ] **Step 1:** Migrate each feature. Do not edit test assertions.
- [ ] **Step 2:** Run `flutter analyze && flutter test`. Expected: all pass.
- [ ] **Step 3:** Commit with `refactor: move vault and product features to repos and cubits`.

### Task 6: Migrate support, notifications, legal

**Files:** `support_repo.dart`, `notifications_repo.dart`, `legal_repo.dart`, `TicketDetailCubit`. FAQ, category, lead, and legal dialogs call their repos per Decision 2.

- [ ] **Step 1:** Migrate each feature. Do not edit test assertions.
- [ ] **Step 2:** Run `flutter analyze && flutter test`. Expected: all pass.
- [ ] **Step 3:** Commit with `refactor: move support, notifications and legal to repos and cubits`.

### Task 7: Migrate access and system

**Files:** `access_repo.dart`, `system_repo.dart`, `RolesCubit` (roles and permission matrix), `LogStatsCubit`, `DeploymentsCubit` (current release panel).

- [ ] **Step 1:** Migrate each feature. Do not edit test assertions.
- [ ] **Step 2:** Run `flutter analyze && flutter test`. Expected: all pass.
- [ ] **Step 3:** Commit with `refactor: move access and system to repos and cubits`.

### Task 8: Migrate shell and remove Riverpod

**Files:**
- Create: `lib/features/main_layout/logic/health_cubit.dart`
- Modify: `app_shell`, `section_gate`, `session_pages`, `lib/app.dart`, `lib/main.dart`, `pubspec.yaml`, `README.md` (project structure section)
- Delete: `lib/core/di/riverpod_bridge.dart`

- [ ] **Step 1:** Migrate the remaining widgets to `context.watch<AuthCubit>()`. Provide `AuthCubit` and `ThemeCubit` with `MultiBlocProvider` in `lib/app.dart`.
- [ ] **Step 2:** Delete the bridge and remove `flutter_riverpod` from `pubspec.yaml`.
- [ ] **Step 3:** Run `grep -rn riverpod lib test pubspec.yaml`. Expected: no output.
- [ ] **Step 4:** Run `flutter analyze && flutter test && flutter build web --no-pub`. Expected: no issues, 245+ tests pass, `✓ Built build/web`.
- [ ] **Step 5:** Commit with `refactor: remove Riverpod`.

---

## Part 2: UI consistency

Audit results that drive these tasks, measured on 2026-11-02:

| Finding | Count | Task |
|---|---|---|
| `_Section` copies (users, messages) | 2 | 9 |
| `_Fact` copies (users, messages, support) | 3 | 9 |
| Toast copies (`hideCurrentSnackBar`) | 4 | 9 |
| Generic failure text copies | 3 | 9 |
| Direct `CircularProgressIndicator` | 8 | 9 |
| `_FormDialog` alongside `EditorDialog` | 2 frames | 10 |
| Save buttons with no loading state, unlike login | all dialogs | 10 |
| Raw sizes and token arithmetic (`KSpace.s2 + 2`, `vertical: 1`) | about 91 | 11 |
| Icon sizes in use: 16, 18, 20, 22, 24, 56 | 6 sizes | 11 |
| Raw `TextStyle(` outside theme | 1 | 11 |
| Non-directional `EdgeInsets.only` or `Alignment.centerLeft` | 18 | 12 |
| Fraunces and Inter fonts still shipped and licensed; Tajawal has no license file | 1 | 13 |

Already consistent: 0 hardcoded colors, 0 `Colors.*`, 0 literal radii. Not used in this product, so out of scope: bottom sheets, FABs, and mobile bottom navigation.

### Task 9: Shared display components and feedback

**Files:**
- Create: `lib/core/widgets/section_card.dart`, `fact_row.dart`, `app_loader.dart`, `app_feedback.dart`
- Modify: every screen that holds a copy
- Test: `test/widgets/shared_widgets_test.dart`

**Interfaces:**
- Produces: `SectionCard({required String title, Widget? trailing, required Widget child})`, `FactRow(String label, String value, {Widget? child})`, and `AppLoader()`.
- Produces: `void showAppToast(BuildContext, String)`, `const kGenericFailure`, and `Future<bool> attempt(BuildContext, Future<void> Function(), {String? success})`.

- [ ] **Step 1: Write the failing tests:**

```dart
testWidgets('a long Arabic name in FactRow does not overflow', (t) async {
  // 320 wide, value = 'عبد الرحمن ' * 12
  expect(t.takeException(), isNull);
});

testWidgets('a child in FactRow keeps its own width', (t) async {
  // StatusChip child narrower than the value column
  expect(t.getSize(find.byType(StatusChip)).width, lessThan(200));
});
```

- [ ] **Step 2:** Run the file. Expected: compile failure because `FactRow` is not defined.
- [ ] **Step 3:** Implement. Replace all copies, using the variant with the most complete states as the base.
- [ ] **Step 4:** Run `grep -rnE 'class _(Section|Fact)\b|hideCurrentSnackBar|CircularProgressIndicator' lib`. Expected: matches only in `core/widgets`, plus the login and submit button spinners.
- [ ] **Step 5:** Run `flutter analyze && flutter test`. Expected: all pass. Commit with `refactor(ui): share section, fact, loader and feedback widgets`.

### Task 10: One dialog frame and one submit button

**Files:**
- Create: `lib/core/widgets/editor_dialog.dart` (moved from `support_widgets.dart`, with `FormRow`), `lib/core/widgets/submit_button.dart`
- Modify: every dialog, and the login form
- Test: `test/widgets/shared_widgets_test.dart`

**Interfaces:** Produces `SubmitButton({Key? key, required String label, required VoidCallback? onPressed, bool busy = false})`. `busy` shows a small spinner and disables the button at the same width.

- [ ] **Step 1: Write the failing test:**

```dart
testWidgets('a busy SubmitButton is disabled and keeps its width', (t) async {
  // compare width with busy false and true
  expect(button.onPressed, isNull);
  expect(busyWidth, idleWidth);
});
```

- [ ] **Step 2:** Run the file. Expected: compile failure because `SubmitButton` is not defined.
- [ ] **Step 3:** Implement. Replace `_FormDialog` with `EditorDialog`, and use `SubmitButton` in `EditorDialog`, login, and app updates.
- [ ] **Step 4:** Run `flutter analyze && flutter test`. Expected: all pass. Commit with `refactor(ui): use one dialog frame and submit button`.

### Task 11: Close token gaps

**Files:**
- Modify: `lib/config/themes/app_tokens.dart`, all screens with raw sizes
- Test: `test/theme/token_usage_test.dart`

**Interfaces:** Produces `KIconSize` (`sm = 16`, `md = 18`, `lg = 20`, `xl = 24`, `hero = 56`), and `KLayout.statusDot = 7`. The one 22 icon becomes `xl`.

- [ ] **Step 1: Write the failing test** that scans `lib/` outside `lib/config`, and fails on: `Color(0x`, `Colors.`, `TextStyle(`, `fontSize:`, `BorderRadius.circular(<digit>`, `BoxShadow(`, and `Icon(..., size: <digit>`.
- [ ] **Step 2:** Run it. Expected: FAIL, listing the raw icon sizes and one `TextStyle(`.
- [ ] **Step 3:** Replace those values with tokens. Replace token arithmetic with the nearest scale value. Where a real size has no token (a chart height, a dialog width), name it in `KLayout`.
- [ ] **Step 4:** Run `flutter analyze && flutter test`. Expected: all pass. Commit with `refactor(ui): enforce design tokens across screens`.

### Task 12: Layout and direction

**Files:** the 10 files with non-directional values, `lib/core/widgets/page_header.dart`, `test/shell/shell_test.dart`

- [ ] **Step 1: Write the failing test:**

```dart
testWidgets('the shell lays out right to left without overflow', (t) async {
  // pump /users at 1000 and 1440 under Directionality rtl
  expect(t.takeException(), isNull);
  // the sidebar is at the right edge
});
```

- [ ] **Step 2:** Run it, and confirm the failure comes from layout, not setup. If it passes, keep it as a regression test and continue.
- [ ] **Step 3:** Convert to `EdgeInsetsDirectional` and `AlignmentDirectional`. Use `matchTextDirection` on chevrons and arrows. Center the `PageHeader` row, so the back button lines up with the title.
- [ ] **Step 4:** Run `flutter analyze && flutter test`. Expected: all pass. Commit with `fix(ui): support right-to-left layout on every screen`.

### Task 13: Remove unused fonts and fix licensing

**Files:**
- Create: `assets/google_fonts/OFL-tajawal.txt`
- Modify: `lib/main.dart`
- Delete: `assets/google_fonts/Fraunces-*.ttf`, `Inter-*.ttf`, `OFL-fraunces.txt`, `OFL-inter.txt`

- [ ] **Step 1:** Download the license from `https://raw.githubusercontent.com/google/fonts/main/ofl/tajawal/OFL.txt`, and read it to confirm that it is the OFL text for Tajawal.
- [ ] **Step 2:** Register only `tajawal` in `lib/main.dart`, and delete the other font files after `grep -rn 'Fraunces\|Inter\b' lib web` shows no use.
- [ ] **Step 3:** Run `flutter analyze && flutter test && flutter build web --no-pub`. Expected: all pass. Commit with `chore: bundle only Tajawal with its license`.

### Task 14: Final consistency audit

**Files:** preview script in the session scratchpad, not committed. Fix any defect in the owning file.

- [ ] **Step 1:** Render every route to PNG at 1440 and 1000 wide, in light and dark. Also render one dialog, one empty state, and one error state.
- [ ] **Step 2:** Review them against the section 8 checklist in the brief: same design language, identical similar components, spacing, type scale, and token colors.
- [ ] **Step 3:** Fix what the review finds, adding a test for each functional defect.
- [ ] **Step 4:** Run `flutter analyze && flutter test && flutter build web --no-pub`, and run the app in a browser against the local backend to walk sign-in, users, and app updates. Expected: all pass, with no console errors. If the local backend cannot run, report that this check was skipped.
- [ ] **Step 5:** Commit with `fix(ui): resolve final consistency audit findings`.
