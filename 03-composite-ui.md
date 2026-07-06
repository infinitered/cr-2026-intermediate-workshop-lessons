# Module 03: Expo UI Iron Chef - custom composite UI

### Goal

Let's use Expo UI in our main app screen, refactoring a custom composite UI and optimizing use of platform-native and RN components. We'll lean on the extra functionality the platform-native controls give us, such as: native menus, sheets, search,and lists. All while keeping our shared logic (filtering, sorting, search) in one place
that never forks per platform.

The screen we're building toward is the **Games Feed**: a native header menu that sorts/filters/switches view modes, a native search affordance, a native "View Options" bottom sheet, and on Android a native carousel. By the end you'll have a single screen that reaches for the _right_ control on each platform.

### Concepts

- **Native stack headers vs. JS headers**: why `Stack.Toolbar` items only render in a
  _native_ stack, and the layout restructure that unlocks them.
- **Declarative header config**: replacing imperative `useLayoutEffect` /
  `navigation.setOptions` with `<Stack.Toolbar>`.
- **`Stack.Toolbar` anatomy**: `placement`, `Menu`, `MenuAction`, `inline` sections,
  nested submenus, `Label`, and communicating selection (`isOn` checkmark, `subtitle`)
  and active state (`variant` + `tintColor`).
- **Universal vs. platform**: `@expo/ui` (universal) bridges SwiftUI ↔
  Jetpack Compose; `@expo/ui/swift-ui` and `@expo/ui/jetpack-compose` are
  platform-specific escape hatches. Prefer universal; split only where it has gaps.
- **Keeping logic platform-agnostic**: the controls fork per platform, the data
  pipeline (`filteredYearGroups`) does not.

### Features to build

- A declarative native header **filter menu** (sort, view mode, genre filter, view options)
- A native **search** affordance (iOS bottom toolbar pill / Android docked search bar)
- A universal **"View Options" bottom sheet** (sort + hide-mature toggle)
- **Android parity**: cross-platform toolbar icons + a native carousel gallery

### Resources

- Expo docs
  - [`@expo/ui`](https://docs.expo.dev/versions/latest/sdk/ui/)
  - [`expo-symbols`](https://docs.expo.dev/versions/latest/sdk/symbols/)
  - [Expo Router - Stack](https://docs.expo.dev/router/advanced/stack/)
- Apple HIG
  - [Menus](https://developer.apple.com/design/human-interface-guidelines/menus)
- Material 3
  - [Menus](https://m3.material.io/components/menus/overview)
  - [Carousel](https://m3.material.io/components/carousel/overview)

# Exercises

## Exercise 0: A declarative native toolbar

Our games feed currently has a plain header. We want a native header **menu** that lets the user sort, switch between gallery/list view, filter by genre, and open more options -
the kind of overflow menu you'd build with `UIMenu` on iOS. Expo Router gives us `Stack.Toolbar` to do this declaratively, in JSX, right inside the screen.

### Step 1 - Give the screen a native stack header

**`Stack.Toolbar` items only render in a _native_ stack header, not the JS header that `Tabs` draws.** If you drop a `<Stack.Toolbar>` into a screen that's rendered directly by a tab, nothing appears.

The fix is structural: wrap the Games tab in its own nested **native `Stack`**, and let that stack own the header. The tab itself stops drawing a header.

First, move the Games screen into a `(home)` group with its own layout:

```
src/app/(tabs)/
  _layout.tsx
  (home)/
    _layout.tsx      ← new: a nested native Stack
    index.tsx        ← moved here from (tabs)/index.tsx
  queue.tsx
  settings.tsx
```

Create the nested stack layout:

```tsx
// src/app/(tabs)/(home)/_layout.tsx
import { Stack } from "expo-router";

import { useAppTheme } from "@/theme/context";
import { typeScale } from "@/theme/typography";

export default function HomeStackLayout() {
  const {
    theme: { colors },
  } = useAppTheme();

  return (
    <Stack
      screenOptions={{
        headerStyle: { backgroundColor: colors.brandSurface },
        headerTintColor: colors.brandSurfaceText,
        headerTitleStyle: { fontFamily: typeScale.headline1.fontFamily },
      }}
    >
      <Stack.Screen name="index" options={{ title: "Games" }} />
    </Stack>
  );
}
```

Then make the tab defer its header to the nested stack:

```tsx
// src/app/(tabs)/_layout.tsx
<Tabs.Screen
  name="(home)"
  options={{
    title: "Games",
    headerShown: false, // the nested Stack owns the header now
    tabBarIcon: ({ color, size }) => <Ionicons name="game-controller" size={size} color={color} />,
  }}
/>
```

🏃 **Try it.** Reload. The Games screen should look the same, but now its header comes from a native stack, and `Stack.Toolbar` will have somewhere to render.

### Step 2 - Replace imperative header config with `<Stack.Toolbar>`

If you've set header buttons before, you've maybe done it imperatively:

```tsx
// the old way - don't do this
useLayoutEffect(() => {
  navigation.setOptions({
    headerRight: () => <FilterButton onPress={openMenu} />,
  });
}, [navigation, openMenu]);
```

`Stack.Toolbar` lets you describe the header declaratively, as part of your screen's render. Start with a single menu button anchored to the right:

```tsx
import { Stack } from "expo-router";

// ...inside GameFeedScreen's return:
<Stack.Toolbar placement="right">
  <Stack.Toolbar.Menu>{/* menu sections go here */}</Stack.Toolbar.Menu>
</Stack.Toolbar>;
```

A `Stack.Toolbar.Menu` placed in the toolbar becomes the tappable header button; its children become the menu that opens. No refs, no effects - it re-renders with your state.

### Step 3 - Build the menu sections

Native menus support **inline sections** (visually grouped, separated rows). Use a
nested `<Stack.Toolbar.Menu inline>` per section. Inside each, `MenuAction` is a row:

- `isOn` renders a checkmark - perfect for single-select like sort order or view mode.
- `subtitle` renders secondary text under the label.
- Tapping a row fires `onPress`.

**Sort section** - tap a row to select it; tap the selected row again to flip direction:

```tsx
<Stack.Toolbar.Menu inline>
  {SORT_OPTIONS.map((order) => (
    <Stack.Toolbar.MenuAction
      key={order}
      isOn={sortOrder === order}
      subtitle={sortOrder === order ? (sortAscending ? "Ascending" : "Descending") : undefined}
      onPress={() => handleSort(order)}
    >
      {order}
    </Stack.Toolbar.MenuAction>
  ))}
</Stack.Toolbar.Menu>
```

```tsx
const handleSort = (order: SortOrder) => {
  if (order === sortOrder) {
    setSortAscending((prev) => !prev); // tapping the active row toggles direction
  } else {
    setSortOrder(order);
    setSortAscending(true);
  }
};
```

**View-mode section** - gallery vs. list:

```tsx
<Stack.Toolbar.Menu inline>
  <Stack.Toolbar.MenuAction isOn={viewMode === "gallery"} onPress={() => setViewMode("gallery")}>
    Gallery
  </Stack.Toolbar.MenuAction>
  <Stack.Toolbar.MenuAction isOn={viewMode === "list"} onPress={() => setViewMode("list")}>
    List
  </Stack.Toolbar.MenuAction>
</Stack.Toolbar.Menu>
```

**Filter submenu** - a nested `Menu` _without_ `inline` becomes a "Filter ›" row that opens a deeper menu. Give it a `Label`, then list genres plus an "All Items" reset:

```tsx
<Stack.Toolbar.Menu inline>
  <Stack.Toolbar.Menu>
    <Stack.Toolbar.Label>Filter</Stack.Toolbar.Label>
    <Stack.Toolbar.Menu inline>
      <Stack.Toolbar.MenuAction isOn={genreIds.length === 0} onPress={clearGenres}>
        All Items
      </Stack.Toolbar.MenuAction>
    </Stack.Toolbar.Menu>
    <Stack.Toolbar.Menu inline>
      {genres.map((genre) => (
        <Stack.Toolbar.MenuAction
          key={genre.id}
          isOn={isSelected(genre.id)}
          onPress={() => toggleGenre(genre.id)}
        >
          {genre.name}
        </Stack.Toolbar.MenuAction>
      ))}
    </Stack.Toolbar.Menu>
  </Stack.Toolbar.Menu>
  {hasFilters && (
    <Stack.Toolbar.MenuAction onPress={clearGenres}>
      {genreIds.length === 1 ? "Remove Filter" : "Remove Filters"}
    </Stack.Toolbar.MenuAction>
  )}
</Stack.Toolbar.Menu>
```

### Step 4 - Signal active state on the button itself

When a filter is applied, the menu button should _look_ active. `Stack.Toolbar.Menu`
takes a `variant` and a `tintColor`:

```tsx
<Stack.Toolbar.Menu
  variant={hasFilters ? "prominent" : "plain"}
  tintColor={hasFilters ? theme.colors.tint : undefined}
>
```

<details>
  <summary>Full assembled toolbar (Exercise 0 result)</summary>

```tsx
const SORT_OPTIONS: SortOrder[] = ["Name", "Rating", "Release Date"]

// ...inside GameFeedScreen:
const hasFilters = genreIds.length > 0

<Stack.Toolbar placement="right">
  <Stack.Toolbar.Menu
    variant={hasFilters ? "prominent" : "plain"}
    tintColor={hasFilters ? theme.colors.tint : undefined}
  >
    <Stack.Toolbar.Menu inline>
      {SORT_OPTIONS.map((order) => (
        <Stack.Toolbar.MenuAction
          key={order}
          isOn={sortOrder === order}
          subtitle={sortOrder === order ? (sortAscending ? "Ascending" : "Descending") : undefined}
          onPress={() => handleSort(order)}
        >
          {order}
        </Stack.Toolbar.MenuAction>
      ))}
    </Stack.Toolbar.Menu>

    <Stack.Toolbar.Menu inline>
      <Stack.Toolbar.MenuAction isOn={viewMode === "gallery"} onPress={() => setViewMode("gallery")}>
        Gallery
      </Stack.Toolbar.MenuAction>
      <Stack.Toolbar.MenuAction isOn={viewMode === "list"} onPress={() => setViewMode("list")}>
        List
      </Stack.Toolbar.MenuAction>
    </Stack.Toolbar.Menu>

    <Stack.Toolbar.Menu inline>
      <Stack.Toolbar.Menu>
        <Stack.Toolbar.Label>Filter</Stack.Toolbar.Label>
        <Stack.Toolbar.Menu inline>
          <Stack.Toolbar.MenuAction isOn={genreIds.length === 0} onPress={clearGenres}>
            All Items
          </Stack.Toolbar.MenuAction>
        </Stack.Toolbar.Menu>
        <Stack.Toolbar.Menu inline>
          {genres.map((genre) => (
            <Stack.Toolbar.MenuAction
              key={genre.id}
              isOn={isSelected(genre.id)}
              onPress={() => toggleGenre(genre.id)}
            >
              {genre.name}
            </Stack.Toolbar.MenuAction>
          ))}
        </Stack.Toolbar.Menu>
      </Stack.Toolbar.Menu>
      {hasFilters && (
        <Stack.Toolbar.MenuAction onPress={clearGenres}>
          {genreIds.length === 1 ? "Remove Filter" : "Remove Filters"}
        </Stack.Toolbar.MenuAction>
      )}
    </Stack.Toolbar.Menu>

    <Stack.Toolbar.Menu inline>
      <Stack.Toolbar.MenuAction onPress={() => setViewOptionsOpen(true)}>
        View Options
      </Stack.Toolbar.MenuAction>
    </Stack.Toolbar.Menu>
  </Stack.Toolbar.Menu>
</Stack.Toolbar>
```

</details>

🏃 **Try it.** Open the app. Tap the header menu. Sort by Rating, then tap Rating again - the subtitle flips between Ascending/Descending and the feed re-sorts. Switch to List.
Open Filter ›, pick a genre - the button turns "prominent" and the feed filters in place.
Hit Remove Filter. Notice you wrote **zero** imperative navigation code.

> 💡 The filtering itself lives in a single `filteredYearGroups` memo
> (search → genre → hideMature → sort). The menu only flips state; the pipeline that
> reacts to it is platform-agnostic. Keep it that way - it pays off when we add Android.

### Side Quests

- Add a `subtitle` to the **Filter ›** submenu row showing the selected genres
  (e.g. "Action, Puzzle"). _Heads up: this currently isn't possible. Trace it down through `node_modules`._
- Make the menu button's SF Symbol change when a filter is active.
- Mark "Remove Filters" as `destructive` and see how iOS renders it.

## Exercise 1: Native search

A feed wants search. Rather than build a custom text input, we'll use each platform's
_native_ search affordance - and this is our first lesson in **when one universal control
isn't the right answer**. iOS and Android put search in different places by convention, so
we deliberately fork the _control_ while sharing the _state_.

Both platforms write to a single piece of state:

```tsx
const [searchQuery, setSearchQuery] = useState("");
```

…and our existing `filteredYearGroups` memo already filters on it (`game.name` includes the
query). Neither platform touches that pipeline - they only call `setSearchQuery`.

### Step 1 - iOS: a bottom toolbar search pill

iOS convention (think Photos, Music) is a search affordance at the bottom. We add a
_second_ `Stack.Toolbar`, this time `placement="bottom"`. Collapsed, it's just a
magnifier button pushed to the right with a `Spacer`. Tapped, it expands into a focused
pill `TextField` with a magnifier on the left and a clear "✕" on the right.

```tsx
{
  Platform.OS !== "android" && (
    <Stack.Toolbar placement="bottom">
      {searchActive ? (
        <>
          <Stack.Toolbar.View>
            <TextField
              ref={searchInputRef}
              autoFocus
              value={searchQuery}
              onChangeText={setSearchQuery}
              placeholder="Search games"
              returnKeyType="search"
              containerStyle={{ width: windowWidth - 96 }}
              inputWrapperStyle={{ borderRadius: 999 }} // pill
              LeftAccessory={(props) => (
                <View style={props.style}>
                  <SymbolView name="magnifyingglass" tintColor={theme.colors.textDim} size={18} />
                </View>
              )}
              RightAccessory={
                searchQuery.length > 0
                  ? (props) => (
                      <Pressable onPress={clearSearch} style={props.style} hitSlop={8}>
                        <SymbolView
                          name="xmark.circle.fill"
                          tintColor={theme.colors.textDim}
                          size={18}
                        />
                      </Pressable>
                    )
                  : undefined
              }
            />
          </Stack.Toolbar.View>
          <Stack.Toolbar.Spacer width={8} />
          <Stack.Toolbar.Button icon={toolbarIcon("close")} onPress={closeSearch} />
        </>
      ) : (
        <>
          <Stack.Toolbar.Spacer />
          <Stack.Toolbar.Button
            icon={toolbarIcon("search")}
            onPress={() => setSearchActive(true)}
          />
        </>
      )}
    </Stack.Toolbar>
  );
}
```

```tsx
const closeSearch = () => {
  setSearchActive(false);
  setSearchQuery("");
};
const clearSearch = () => {
  setSearchQuery("");
  searchInputRef.current?.focus(); // clear but keep typing
};
```

Notice the `RightAccessory` clear button only renders when there's text - same as the
system search fields. The `xmark` close `Button` is separate: it collapses the whole pill.

> Why gate with `Platform.OS !== "android"`? On Android the bottom-toolbar host renders an
> empty Compose floating toolbar - no usable search button. So Android gets a different
> control entirely (next step).

### Step 2 - Android: a docked search bar, via a platform-split component

Android convention is a search bar docked at the **top** of the content. We isolate this in
a **platform-split component** so the Android-only `@expo/ui/jetpack-compose` import never
loads on iOS.

Create two files. The base is a deliberate **no-op** (iOS already has its bottom pill):

```tsx
// app/components/FeedSearch.tsx
export type FeedSearchProps = {
  /** Called live as the query changes. */
  onChangeText: (text: string) => void;
};

// On iOS the search lives in the bottom Stack.Toolbar, so this base variant renders nothing.
export function FeedSearch(_props: FeedSearchProps) {
  return null;
}
```

```tsx
// app/components/FeedSearch.android.tsx
import { View, type ViewStyle } from "react-native";
import { DockedSearchBar, Host, Icon, Text } from "@expo/ui/jetpack-compose";

import { useAppTheme } from "@/theme/context";
import { useToolbarIcons } from "@/utils/useToolbarIcons";

export function FeedSearch({ onChangeText }: { onChangeText: (t: string) => void }) {
  const { theme, themeContext } = useAppTheme();
  const searchIcon = useToolbarIcons(theme.colors.text)("search");

  return (
    <View style={$wrapper}>
      {/* Pin the Compose color scheme to the app theme, or the bar follows the device
          appearance (renders dark while the app is in light mode). */}
      <Host useViewportSizeMeasurement style={$host} colorScheme={themeContext}>
        <DockedSearchBar onQueryChange={onChangeText}>
          <DockedSearchBar.LeadingIcon>
            {searchIcon ? <Icon source={searchIcon} size={24} /> : null}
          </DockedSearchBar.LeadingIcon>
          <DockedSearchBar.Placeholder>
            <Text>Search games</Text>
          </DockedSearchBar.Placeholder>
        </DockedSearchBar>
      </Host>
    </View>
  );
}

const $wrapper: ViewStyle = { height: 72, paddingHorizontal: 16, paddingVertical: 8 };
const $host: ViewStyle = { flex: 1 };
```

Then drop it into the feed once, above the `ScrollView` - it renders on Android, no-ops on iOS:

```tsx
<FeedSearch onChangeText={setSearchQuery} />
```

> ⚠️ **Jetpack Compose layout gotcha.** A bare `<Host matchContents>` proposes _unbounded_
> width, so `DockedSearchBar` (which fills its width) collapses to nothing. The fix:
> wrap the `Host` in a plain RN `View` with an explicit height, give the `Host` `flex: 1`
> plus `useViewportSizeMeasurement` (so it proposes the viewport width). Don't also pass
> `fillMaxWidth()` - the bar has its own M3 default width.

> ⚠️ **New platform files don't hot-reload reliably.** After _creating_
> `FeedSearch.android.tsx`, Fast Refresh often keeps serving a stale bundle and the
> component looks broken/absent. Force a full reload (kill + relaunch from the dev-client
> launcher, or restart Metro) before concluding anything is wrong.

🏃 **Try it.** On iOS, tap the magnifier in the bottom bar, type "star" - the feed filters
to Star games in place; the "✕" clears the text, the close button collapses the pill. On
Android, type into the top docked bar - same in-place filter. Two different controls, one
`searchQuery`.

### Side Quests

- The Android `DockedSearchBar` has **no clear ("✕") button** - it can't host a trailing icon and its query is uncontrolled.
- Debounce `setSearchQuery` and observe whether it matters for this dataset size.

## Exercise 2: The universal "View Options" bottom sheet

Now the opposite lesson: **when the universal API _is_ the right answer.** The "View
Options" action we wired into the menu in Exercise 0 should open a sheet with a sort picker
and a "Hide Mature" toggle. `@expo/ui`'s root (universal) export bridges SwiftUI ↔ Jetpack
Compose, so `BottomSheet` + `FieldGroup` + `Picker` + `Switch` give us **one code path**
that renders natively on both platforms.

```tsx
import { BottomSheet, FieldGroup, Picker, Switch } from "@expo/ui";
```

Drive it from the `viewOptionsOpen` state the menu already toggles:

```tsx
<BottomSheet
  isPresented={viewOptionsOpen}
  onDismiss={() => setViewOptionsOpen(false)}
  snapPoints={["half", "full"]}
>
  <FieldGroup>
    <FieldGroup.Section title="Sort By">
      <Picker
        selectedValue={sortOrder}
        onValueChange={(value) => setSortOrder(value as SortOrder)}
        appearance="menu"
      >
        {SORT_OPTIONS.map((order) => (
          <Picker.Item key={order} label={order} value={order} />
        ))}
      </Picker>
    </FieldGroup.Section>
    <FieldGroup.Section title="Advanced">
      <Switch value={hideMature} onValueChange={setHideMature} label="Hide Mature Content" />
    </FieldGroup.Section>
  </FieldGroup>
</BottomSheet>
```

`sortOrder` and `hideMature` come from the shared `useSettings` store - the same values the
Settings screen reads - so changing them here is reflected everywhere, and the
`filteredYearGroups` memo re-runs automatically.

> ⚠️ **`snapPoints` gotcha.** Pass `snapPoints={["half", "full"]}`. Omit them and the sheet
> defaults to `fitToContents`, which collapses a `FieldGroup` to ~0 height - you'll get an
> invisible sheet and think it's broken.

> 💡 Import discipline: `@expo/ui` (root) is universal. `@expo/ui/swift-ui` and
> `@expo/ui/jetpack-compose` are platform-only. Reach for a platform import _only_ when the
> universal API lacks what you need (as with search). Here it didn't - so one import, both
> platforms.

🏃 **Try it.** On both iOS and Android: open the header menu → View Options. The sheet
slides up natively, the picker changes sort order, the toggle hides mature games - and the
feed updates live behind the sheet. Same JSX, two native renderers.

### Side Quests

- iOS polish: give the sheet a bold title and a "Done" button. (Deferred in the reference
  build - see if you can add it without breaking Android.)
- Persist `hideMature` across launches and confirm the Settings screen stays in sync.

## Exercise 3: Android parity - icons & a native carousel

Two Android gaps remain. Both teach the same meta-skill: **make one declaration serve both
platforms by resolving per-platform under the hood.**

### Step 1 - Cross-platform toolbar icons

Run Exercise 0's menu on Android and the icons vanish. Root cause: Android's
`Stack.Toolbar` Menu/Button/MenuAction **drop SF Symbol names** - they only render an
`ImageSourcePropType`. So we need an SF Symbol on iOS and a real image source on Android,
from a _single_ call site.

The trick: rasterize **Material Symbols** to image sources via `expo-symbols`
(`unstable_getMaterialSymbolSourceAsync`), and pair each logical icon with both an SF name
and a Material name behind a hook:

```tsx
// app/utils/useToolbarIcons.ts
const TOOLBAR_ICONS = {
  filter: { sf: "line.3.horizontal.decrease", material: "filter_list" },
  gallery: { sf: "square.grid.2x2", material: "grid_view" },
  list: { sf: "list.bullet", material: "view_list" },
  allItems: { sf: "rectangle.grid.3x3", material: "apps" },
  removeFilter: { sf: "minus.circle", material: "filter_list_off" },
  search: { sf: "magnifyingglass", material: "search" },
  close: { sf: "xmark", material: "close" },
} satisfies Record<string, { sf: SFSymbol; material: AndroidSymbol }>;

export function useToolbarIcons(color: string, size = 24) {
  const [sources, setSources] = useState<Partial<Record<ToolbarIconKey, ImageSourcePropType>>>({});

  useEffect(() => {
    if (Platform.OS !== "android") return;
    let cancelled = false;
    const keys = Object.keys(TOOLBAR_ICONS) as ToolbarIconKey[];
    Promise.all(
      keys.map((key) =>
        unstable_getMaterialSymbolSourceAsync(TOOLBAR_ICONS[key].material, size, color).then(
          (source) => [key, source] as const,
        ),
      ),
    ).then((entries) => {
      if (!cancelled) setSources(Object.fromEntries(entries.filter(([, s]) => s != null)));
    });
    return () => {
      cancelled = true;
    };
  }, [color, size]);

  // iOS: the SF Symbol name. Android: the rasterized image source (undefined until loaded).
  return (key: ToolbarIconKey) =>
    Platform.OS === "android" ? sources[key] : TOOLBAR_ICONS[key].sf;
}
```

Now `icon={toolbarIcon("filter")}` Just Works on both platforms. Switch the root menu from
a `<Stack.Toolbar.Icon sf=…>` child to the `icon={toolbarIcon("filter")}` prop so it flows
through the resolver too.

> 💡 **A platform _convention_ call, not a bug workaround.** Android overflow menus are
> conventionally **text-only**, so we keep menu-_row_ icons iOS-only and let the `isOn`
> checkmark carry selection on Android. The toolbar trigger button and the search magnifier
> _do_ keep their icons (toolbar actions and search bars are idiomatic with icons):
>
> ```tsx
> const menuIcon = (key: ToolbarIconKey) => (Platform.OS === "ios" ? toolbarIcon(key) : undefined);
> // ...use menuIcon(...) on MenuActions, toolbarIcon(...) on the trigger + search.
> ```

> 💡 Valid names: SF Symbols live in
> `node_modules/sf-symbols-typescript/dist/index.d.ts`; Material names are keys of
> `node_modules/expo-symbols/build/android/symbols.json`.

### Step 2 - A native carousel gallery on Android

iOS gallery view is a horizontal `FlatList` of cards (`GameGallery.tsx`). On Android we can
do better with Material 3's `HorizontalMultiBrowseCarousel` - native masking + morph
animations as you swipe. The neat part: we **host our existing RN card inside the Compose
tree** with `RNHostView`, so we don't rebuild the card in Compose.

```tsx
// app/components/GameGallery.android.tsx
import { ViewStyle } from "react-native";
import { Host, HorizontalMultiBrowseCarousel, RNHostView } from "@expo/ui/jetpack-compose";

import { GameCarouselCard } from "./GameCarouselCard";
import type { Game } from "../services/api/types";

const CAROUSEL_HEIGHT = 220;
const PREFERRED_ITEM_WIDTH = 220; // focused item; the carousel sizes peeking items around it

export function GameGallery({ games }: { games: Game[] }) {
  return (
    <Host style={{ height: CAROUSEL_HEIGHT } satisfies ViewStyle}>
      <HorizontalMultiBrowseCarousel
        preferredItemWidth={PREFERRED_ITEM_WIDTH}
        itemSpacing={8}
        contentPadding={{ start: 16, end: 16 }}
      >
        {games.map((game) => (
          <RNHostView key={game.id}>
            <GameCarouselCard game={game} />
          </RNHostView>
        ))}
      </HorizontalMultiBrowseCarousel>
    </Host>
  );
}
```

Because this is `GameGallery.android.tsx`, Metro automatically picks it for Android and the
plain `GameGallery.tsx` (FlatList) for iOS - `YearSection` imports `"./GameGallery"` and
never knows the difference.

> ⚠️ **`RNHostView` image sizing.** An RN `<Image>` inside `RNHostView` needs a concrete
> flex/size (the card uses `flex: 1` to fill the slot). `position: absolute`/`absoluteFill`
> images never lay out and silently fail to load inside the Compose host.

🏃 **Try it.** On Android: the filter menu icons render (Material Symbols), and switching to
Gallery view shows the native carousel - swipe and watch items mask/morph at the edges. On
iOS: SF Symbols and the FlatList gallery, unchanged. One `YearSection`, one
`toolbarIcon(...)` call site, two native experiences.

### Side Quests

- Render Gallery/List on Android as a Material `SegmentedButton` instead of menu rows.
- Decide whether `viewMode` should persist (move it into `useSettings`).

## See the solution

Switch to branch: [`03-composite-ui`](https://github.com/infinitered/cr-2026-intermediate-workshop-template/tree/03-composite-ui)
