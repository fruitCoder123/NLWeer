# Investigation: replacing the current page mechanism with HorizontalPager

Date: 2026-10-24  
Repository: `CNugteren/NLWeer` / fork `fruitCoder123/NLWeer`  
Branch investigated: `add_HorizontalPager` (currently identical to `master`)  
Scope: investigation only; no source changes were made.

## 1. Executive summary

The app does **not** currently use a pager. It uses the Android Navigation Component:

- one `NavHostFragment` in `content_main.xml`;
- a dynamically built navigation graph in `MainActivity.createNavGraph()`;
- one Fragment destination per weather map/page;
- drawer navigation and two floating previous/next buttons that call `navController.navigate(...)`.

`HorizontalPager` is a **Jetpack Compose** API (`androidx.compose.foundation.pager.HorizontalPager`). This project is entirely View/XML/Fragment based and has no Compose setup. Therefore there are two realistic options:

1. **Recommended for this codebase: use `ViewPager2` + `FragmentStateAdapter`.**
   This adds real horizontal swiping while keeping every existing Fragment, layout, WebView and chart intact. It is the View-system equivalent of `HorizontalPager` and is a much smaller and safer change.
2. **Use the actual Compose `HorizontalPager`.**
   This requires adding Compose to the build and either rewriting the page UIs in Compose or embedding the existing Fragments/custom Views inside Compose. The latter is possible but awkward; the former is a large rewrite.

If the requirement is literally “use `HorizontalPager`”, option 2 is the only correct API choice. If the requirement is “let the user swipe horizontally between weather pages”, option 1 is strongly recommended.

## 2. Current mechanism in detail

### 2.1 Page inventory and order

`app/src/main/java/foss/cnugteren/nlweer/MapData.kt` defines:

- `KnmiItems`, `BuienradarItems`, and `ALL_ITEMS = KNMI_ITEMS + BUIENRADAR_ITEMS`.
- Each `MapData` has a `stringId`, `navId`, `defaultVisible`, `parentMenuId`, `mapUrl`, image size and geo coordinates.
- `ALL_ITEMS` is already the logical left-to-right ordering that the previous/next buttons use.

The two “source” parent menu items are `R.id.knmi_menu` and `R.id.buienradar_menu`. Pages are visible only if:

- the source (`knmi_enable` / `buienradar_enable`) is enabled, **and**
- the individual page is enabled in `settings_nav_view_knmi` / `settings_nav_view_buienradar`.

### 2.2 Navigation graph

`MainActivity.kt`:

- `createNavGraph()` (around line 125) creates the graph programmatically with start destination `R.id.nav_knmi_rain_m1`.
- For every item in `ALL_ITEMS` it registers either:
  - `KnmiTextFragment` for `nav_knmi_text`;
  - `KnmiSixDayForecastFragment` for `nav_knmi_sixdayforecast`;
  - `KnmiPluimFragment` for `nav_knmi_pluim`;
  - `BuienradarChartFragment` for `nav_buienradar_chart`;
  - `BuienradarPluimFragment` for `nav_buienradar_pluim`;
  - `MapFragment` for everything else.
- It also registers `nav_empty`, `nav_about` and `nav_settings`.

`app/src/main/res/navigation/mobile_navigation.xml` still exists only to declare the `R.id.nav_*` constants; the real graph is built in code.

`MainActivity` wires the drawer and toolbar with:

```kotlin
setupActionBarWithNavController(navController, appBarConfiguration)
navView.setupWithNavController(navController)
```

### 2.3 Drawer menu

`app/src/main/res/menu/activity_main_drawer.xml` contains all page IDs grouped under `knmi_menu` and `buienradar_menu`.

`MainActivity.setMenuItemsVisibility()` (around line 184) reads preferences and sets `MenuItem.isVisible`.

Important: this method currently updates **only the drawer menu**. With a pager, the same filtering logic must also drive the **pager page list**, otherwise hidden pages would still be swipeable or the drawer/page order would diverge.

### 2.4 Previous/next buttons

`MainActivity.kt` lines 76–99:

- find the current destination ID;
- find its index in `ALL_ITEMS`;
- scan forward/backward through a doubled list until a menu item and its parent are visible;
- call `navController.navigate(item.navId)`.

These two listeners can be reduced to one line each with a pager (`setCurrentItem(index + 1, true)` / `index - 1`).

### 2.5 Page Fragment behaviour

All page fragments:

- inflate a ViewBinding layout and reload their content in `onCreateView()`;
- call `activity.toggleNavigationButtons(true)` in `onCreateView()`;
- have a `SwipeRefreshLayout` and a `refreshPage()` method;
- update location through `setLocation(...)` where relevant.

`MapFragment` is the important special case:

- It discovers **its own destination ID** with `NavHostFragment.findNavController(this)` and `currentDestination?.id` (`MapFragment.kt` around lines 41–42), then looks up URL/image/coordinates in `ALL_ITEMS`.
- It scales a custom `DrawWebView`, draws the GPS circle in `onDraw`, and loads the map via `loadDataWithBaseURL`.
- Under a pager, `findNavController()` will no longer return the current page. The page identity must instead be passed as Fragment arguments (`navId`) or the adapter must set it directly.

### 2.6 Activity coupling to the “current page”

`MainActivity` assumes there is exactly one current fragment inside the nested NavHost:

- `onClickRefresh()` uses `navHostFragment.childFragmentManager.fragments[0]`;
- `onClickShare()` uses the same and expects a `MapFragment`;
- `locationListener` calls `setLocation` on `fragments[0]` only.

With a pager, multiple page Fragments can exist at the same time (typically the current page plus one neighbour). These three places must be changed to resolve the page from the pager adapter/current item instead of `fragments[0]`.

### 2.7 Start page / default view

`MainActivity.setStartFragment()` changes the navigation graph’s start destination based on `settings_default_view_listpreference`.

With a pager this becomes “find the index of the preferred `navId` in the visible page list and use it as `initialPage`”. The `nav_empty` case also needs a defined behaviour (an empty placeholder page, or fall back to the first page).

## 3. Option A (recommended): ViewPager2 + FragmentStateAdapter

This is the non-Compose horizontal pager. It preserves the current Fragments and layouts.

### 3.1 Dependency

```gradle
implementation 'androidx.viewpager2:viewpager2:1.1.0'
```

`ViewPager2` 1.1.0 is stable and works with `minSdk 21`.

### 3.2 Layout

Replace the `<fragment android:name="androidx.navigation.fragment.NavHostFragment">` in `content_main.xml` with:

```xml
<androidx.viewpager2.widget.ViewPager2
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:overScrollMode="never" />
```

The `fab_previous` / `fab_next` buttons can stay exactly where they are.

### 3.3 Page model

Create a small resolved page list, for example:

```kotlin
data class PagerPage(val item: MapData, val fragmentClass: Class<out Fragment>)
```

Build it from `ALL_ITEMS` using the same rules as `setMenuItemsVisibility()`:

- parent menu enabled (`knmi_enable` / `buienradar_enable`);
- page selected/visible according to the preference set.

`ALL_ITEMS` already gives the order. Keeping this as the single source of truth avoids duplicating the visibility logic.

### 3.4 Adapter

Use `FragmentStateAdapter` and map each `navId` to its Fragment class:

```kotlin
class WeatherPagerAdapter(
    activity: FragmentActivity,
    private val pages: List<PagerPage>
) : FragmentStateAdapter(activity) {

    override fun getItemCount() = pages.size

    override fun createFragment(position: Int): Fragment {
        val navId = pages[position].item.navId
        return when (navId) {
            R.id.nav_knmi_text -> KnmiTextFragment()
            R.id.nav_knmi_sixdayforecast -> KnmiSixDayForecastFragment()
            R.id.nav_knmi_pluim -> KnmiPluimFragment()
            R.id.nav_buienradar_chart -> BuienradarChartFragment()
            R.id.nav_buienradar_pluim -> BuienradarPluimFragment()
            else -> MapFragment().apply {
                arguments = bundleOf("navId" to navId)
            }
        }
    }
}
```

`MapFragment` should read `navId` from `arguments` instead of `NavHostFragment.findNavController(this).currentDestination?.id`.

`FragmentStateAdapter` gives each Fragment a tag `f<position>`; the current Fragment can therefore be found with:

```kotlin
supportFragmentManager.findFragmentByTag("f${pager.currentItem}")
```

Alternatively keep a weak reference map in the adapter. Tag lookup is simpler but relies on the documented `f<position>` tag convention; a small `FragmentStateAdapter` subclass can expose a safe lookup helper.

### 3.5 MainActivity wiring

Remove or bypass `NavHostFragment` for weather pages. Keep the existing NavController only if settings/about still need it, or handle those with a separate `FragmentContainerView` / fragment transaction.

Minimum changes:

- build the visible page list;
- create the adapter and set it on the pager;
- set `pager.offscreenPageLimit = 1` (optional; see risks);
- register `pager.registerOnPageChangeCallback(...)`;
- drawer click handlers map `navId` → page index and call `pager.setCurrentItem(index, true)`;
- FABs call `setCurrentItem(currentItem ± 1, true)`, clamped to the page list;
- toolbar title and drawer checked item are updated in `onPageSelected`;
- `setStartFragment()` becomes `pager.setCurrentItem(indexOf(defaultNavId), false)`;
- after a settings change, rebuild the page list, `notifyDataSetChanged()`, preserve the current `navId` where still visible, and update the FAB visibility;
- refresh/share/location use the current Fragment from the adapter instead of `childFragmentManager.fragments[0]`.

### 3.6 Title and drawer state

`setupActionBarWithNavController` will no longer work for weather pages. In `onPageSelected`:

```kotlin
supportActionBar?.title = getString(pages[position].item.stringId)
navView.setCheckedItem(pages[position].item.navId)
```

The drawer can still be opened/closed normally. Settings/about can be launched as normal Fragment destinations or as a separate mode that temporarily hides the pager.

### 3.7 Keeping the Fragment lifecycle correct

`FragmentStateAdapter` destroys Fragments that are more than one page away from the current page (with the default offscreen limit). This is similar to the current behaviour, where navigating away destroys the previous Fragment’s view. Existing `_binding == null` guards in the async tasks already protect against late callbacks.

## 4. Option B: actual Compose HorizontalPager

### 4.1 Build changes needed

The project currently has no Compose. A true `HorizontalPager` needs only the Foundation pager + runtime, but the Compose compiler plugin must be configured:

```gradle
android {
    buildFeatures {
        compose = true
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.10" // compatible with Kotlin 1.9.22
    }
}

dependencies {
    implementation platform('androidx.compose:compose-bom:2024.06.00')
    implementation 'androidx.compose.ui:ui'
    implementation 'androidx.compose.foundation:foundation'
    implementation 'androidx.compose.runtime:runtime'
    // optionally material3 / activity-compose
}
```

The exact BOM version can be chosen later; it must match Kotlin 1.9.22’s Compose compiler (`1.5.10` is the matching compiler for Kotlin 1.9.22). `minSdk 21` is supported.

### 4.2 Where the pager lives

Keep the Activity/XML shell and place a `ComposeView` where the NavHost is now:

```xml
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/pager_compose_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

Then in `MainActivity`:

```kotlin
composeView.setContent {
    val state = rememberPagerState(
        initialPage = initialPage,
        pageCount = { pages.size }
    )
    HorizontalPager(
        state = state,
        beyondViewportPageCount = 1
    ) { position ->
        PagerPageContent(pages[position], position)
    }
}
```

Note: the `rememberPagerState(initialPage) { pageCount }` overload is the current one (Compose Foundation 1.6+). Older code used `rememberPagerState(pageCount = { ... })` plus a separate initial-page mechanism; pin the Compose version and use the matching signature.

`LaunchedEffect(state.currentPage)` can update the toolbar title, drawer checked item and FAB state. The FABs call `state.animateScrollToPage(...)` from a coroutine scope.

### 4.3 Content options inside the pager

There are three possible content strategies:

1. **Embed existing Fragments** with `AndroidView` + `FragmentContainerView` and `childFragmentManager`.
   - Reuses all current code but mixes two lifecycle systems.
   - Nesting Fragments inside Compose pages is fragile (recomposition can recreate containers, saved state is harder, and `FragmentStateAdapter`-style reuse is lost).
   - Not recommended.

2. **Embed only the custom Views** (`DrawWebView`) with `AndroidView`, and rewrite text/chart pages in Compose.
   - `MapFragment`’s logic (URL lookup, image scaling, location circle, SwipeRefresh) would move into a Compose page or a custom View wrapper.
   - `DrawWebView` can be created with `AndroidView(factory = { DrawWebView(it) }, update = { ... })`, but WebView lifecycle, `loadDataWithBaseURL`, and measure/layout scaling must be handled carefully.
   - The MPAndroidChart page and the JSoup text pages would need Compose equivalents or `AndroidView` wrappers around the existing Views.
   - Medium-to-large effort and a real regression risk for WebView/zoom/GIF behaviour.

3. **Full Compose migration of all pages**.
   - Cleanest long-term if the project wants Compose everywhere.
   - Requires rewriting six page types, theming, dark mode, pull-to-refresh, WebView wrappers, location overlay, and dynamic visibility.
   - Large effort, not justified by a pager change alone.

### 4.4 Interaction with SwipeRefreshLayout / WebView

If existing Android Views are embedded, the vertical `SwipeRefreshLayout` and the horizontal Compose pager can coexist, but the `DrawWebView` (a WebView) may consume horizontal drags when its content is wider than the screen. Expect to test and possibly disable horizontal scrolling inside the WebView or use `requestDisallowInterceptTouchEvent` logic.

## 5. Shared behaviour that must be preserved

| Behaviour | Current implementation | Pager requirement |
|---|---|---|
| Page order | `ALL_ITEMS` | Same list, filtered by visibility |
| Page types | `when` in `createNavGraph()` | Same `when` in adapter/Compose content |
| Source enable/disable | `setMenuItemsVisibility()` | Rebuild page list; update pager |
| Per-page visibility | preferences sets | Same; keep page list in sync |
| Start page | graph start destination | `initialPage` / `setCurrentItem` |
| Drawer navigation | `setupWithNavController` | click → find index → `setCurrentItem` |
| Title | `setupActionBarWithNavController` | update in page-change callback |
| FAB previous/next | graph navigation | `currentItem ± 1`, clamped |
| Refresh | current child fragment | current pager Fragment |
| Share | current child fragment | current pager Fragment |
| GPS updates | `fragments[0]` | current pager Fragment (and optionally neighbours) |
| About/settings/empty | graph destinations | separate destinations or special pager handling |

## 6. Suggested implementation phases (for the actual change)

1. Add `ViewPager2` dependency and replace the NavHost with `ViewPager2` in `content_main.xml`.
2. Introduce the resolved page-list model and `FragmentStateAdapter`; make `MapFragment` accept `navId` as an argument.
3. Wire drawer clicks, toolbar title, checked state, and FABs to the pager.
4. Move start-page selection and visibility rebuilds to the pager.
5. Update refresh/share/location to use the current pager Fragment.
6. Manually verify: swipe both directions, drawer jumps, hidden pages, source toggles, default start page, dark mode, GPS/manual location, pull-to-refresh, map scaling, WebView pages, rotation, and back from settings/about.
7. Only after the ViewPager2 version is stable, consider a separate Compose/`HorizontalPager` migration if that is a project goal.

## 7. Main risks

- **`MapFragment` destination lookup** breaks if moved to a pager without passing `navId`.
- **`toggleNavigationButtons` is called from every Fragment’s `onCreateView`**; adjacent pages may update FAB visibility while being pre-created. Move this responsibility to the pager’s page-change callback.
- **`fragments[0]` assumptions** in refresh/share/location break as soon as more than one page Fragment exists.
- **Hidden pages** must be filtered from the pager, not only from the drawer menu.
- **`nav_empty`/about/settings** are not pager pages; decide how they fit into the drawer/pager model.
- **WebView memory and touch handling**: multiple live WebViews/pager pages can increase memory and may interfere with horizontal swipe gestures.
- **Compose adoption** is a major architectural change; it should not be bundled into the pager change unless the project explicitly wants it.

## 8. Recommendation

Use **ViewPager2 + FragmentStateAdapter** for this branch. It delivers the desired horizontal paging with minimal disruption to the existing Fragment/WebView architecture. Reserve Compose `HorizontalPager` for a deliberate, larger Compose migration.
