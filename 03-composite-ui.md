# Module 03: Expo UI Iron Chef - custom composite UI

### Goal

Now we cook. We're taking Expo UI to our main app screen - the **Games Feed** - and building a real composite UI: a native header menu that sorts and filters, a native search affordance, a native "View Options" sheet, and a native carousel on Android. The fun part is deciding, screen by screen, when to reach for a _universal_ control and when to fork into a _platform-specific_ one. Through it all, our shared logic (filter, sort, search) stays in one place and never forks per platform.

We'll also keep the screen itself thin. Rather than pile every native control into `GameFeedScreen`, each affordance gets its own component - `FilterMenu`, `FeedSearch`, `ViewOptionsSheet` - and the screen just holds the state that ties them together.

### Concepts

- Native stack headers vs. JS headers - and why `Stack.Toolbar` needs the former
- Declarative header config with `<Stack.Toolbar>` instead of imperative `navigation.setOptions`
- Extracting native controls into their own components; keeping the screen a thin orchestrator
- When to fork the _control_ per platform but share the _state_ (search)
- When the universal `@expo/ui` API is already the right answer (the sheet)
- Making one call site resolve to the right thing per platform (icons, carousel)

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

> ⚠️ `Stack.Toolbar` is `@experimental` in expo-router. The API is settling - expect a few rough edges (we document the ones we hit).

# Exercises

## Exercise 0: A declarative native toolbar

Our games feed has a plain header. We want a native header **menu** - the kind of overflow menu you'd build with `UIMenu` on iOS - that lets the user sort, switch between gallery and list, filter by genre, and open more options. Expo Router gives us `Stack.Toolbar` to do this declaratively, right in JSX. No refs, no effects.

### Give the screen a native stack header

Here's the catch: **`Stack.Toolbar` items only render in a _native_ stack header, not the JS header that `Tabs` draws.** Drop a `<Stack.Toolbar>` into a screen rendered directly by a tab and... nothing appears.

The fix is structural. We wrap the Games tab in its own nested **native `Stack`** and let that stack own the header.

1. Move the Games screen into a `(home)` group with its own layout:

```
src/app/(tabs)/
  _layout.tsx
  (home)/
    _layout.tsx      ← new: a nested native Stack
    index.tsx        ← moved here from (tabs)/index.tsx
  queue.tsx
  settings.tsx
```

2. Create the nested stack layout:

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

3. Then make the tab defer its header to the nested stack:

```diff
<Tabs.Screen
-  name="index"
+  name="(home)"
  options={{
    title: "Games",
+   headerShown: false, // the nested Stack owns the header now
    tabBarIcon: ({ color, size }) => <Ionicons name="game-controller" size={size} color={color} />,
  }}
/>
```

🏃**Try it.** Reload. The Games screen should look exactly the same - but now its header comes from a native stack, and `Stack.Toolbar` has somewhere to render.

### Keep the screen thin

The toolbar we're about to build only _flips_ state - the feed selector re-derives itself in response. So the screen's job shrinks to two things: hold the little bit of local state the controls share, and read the shared stores where the rest of the state lives.

The starter `GameFeedScreen` already renders the feed from a shared selector. Trim it down to just the state the child components need:

```tsx
import { useState } from "react";
import { ScrollView, ViewStyle } from "react-native";

import { EmptyState } from "@/components/EmptyState";
import { LoadingScreen } from "@/components/LoadingScreen";
import { Screen } from "@/components/Screen";
import { YearSection } from "@/components/YearSection";
import { $styles } from "@/theme/styles";

import { useFilteredGamesByYear } from "@/stores/gameFeed";
import { useSettings } from "@/stores/settings";

export function GameFeedScreen() {
  const [searchQuery, setSearchQuery] = useState("");
  const { yearGroups, isLoading, isError } = useFilteredGamesByYear(searchQuery);
  const { viewMode } = useSettings();

  const [viewOptionsOpen, setViewOptionsOpen] = useState(false);

  if (isLoading) {
    return <LoadingScreen />;
  }

  if (isError || !yearGroups || yearGroups.length === 0) {
    return (
      <Screen preset="fixed" contentContainerStyle={$centered}>
        <EmptyState
          heading="There's Nothing Here..."
          content="Try adjusting your filters or search query."
          ButtonProps={{ text: "Clear Search" }}
          buttonOnPress={() => setSearchQuery("")}
        />
      </Screen>
    );
  }

  return (
    <Screen preset="fixed" contentContainerStyle={$styles.flex1}>
      {/* native controls land here as we build them */}
      <ScrollView>
        {yearGroups.map((group) => (
          <YearSection key={group.year} year={group.year} games={group.games} viewMode={viewMode} />
        ))}
      </ScrollView>
    </Screen>
  );
}

const $centered: ViewStyle = {
  flex: 1,
  justifyContent: "center",
  alignItems: "center",
};
```

A few things to notice:

- `searchQuery` is **local** to the screen - it's a transient UI concern, and we'll hand it to both the search control and `useFilteredGamesByYear` (Exercise 1).
- `viewOptionsOpen` is **local** too - it's the open/close state for the sheet, shared between the menu (which opens it) and the sheet (which closes it), so it lives in their common parent.
- `viewMode` comes from the shared **`useSettings`** store - it persists, and the menu will flip it. Everything else the menu touches (`sortOrder`, `hideMature`, genre selection) also lives in shared stores, so the menu component can read them directly without prop-drilling.

> 💡 The filtering itself lives in a single selector, `useFilteredGamesByYear` (search → genre → hideMature → sort). The controls only flip store state; the pipeline that reacts to it is platform-agnostic. Keep it that way - it pays off when we add Android.

### From imperative to declarative

If you've set header buttons before, you've maybe done it the imperative way:

```tsx
// the old way - don't do this
useLayoutEffect(() => {
  navigation.setOptions({
    headerRight: () => <FilterButton onPress={openMenu} />,
  });
}, [navigation, openMenu]);
```

`Stack.Toolbar` lets you describe the header as part of your render, and it re-renders with your state. We'll build that menu as its own component so the screen stays readable.

### Build the FilterMenu component

Create **app/components/FilterMenu.tsx**. It reads the shared stores itself and renders the toolbar; the only thing it needs from the screen is a callback to open the View Options sheet (which the screen owns). Start with the imports, props, and the store reads:

```tsx
// app/components/FilterMenu.tsx
import { Stack } from "expo-router";

import { SORT_OPTIONS } from "@/stores/gameFeed";
import { useSettings, type SortOrder } from "@/stores/settings";
import { useGenreFilter } from "@/stores/genreFilter";
import { useFeedGenres } from "@/services/api/games";
import { useToolbarIcons } from "@/utils/useToolbarIcons";
import { colors } from "@/theme/colors";

interface FilterMenuProps {
  handleOpenViewOptions: () => void;
}

export function FilterMenu({ handleOpenViewOptions }: FilterMenuProps) {
  const toolbarIcon = useToolbarIcons(colors.brandSurfaceText);
  const { selectedIds: genreIds, isSelected, toggleGenre, clearGenres } = useGenreFilter();
  const hasFilters = genreIds.length > 0;
  const { sortOrder, sortAscending, setSortOrder, setSortAscending, viewMode, setViewMode } =
    useSettings();
  const { data: genres = [] } = useFeedGenres();

  const handleSort = (order: SortOrder) => {
    if (order === sortOrder) {
      setSortAscending(!sortAscending); // tapping the active row toggles direction
    } else {
      setSortOrder(order);
      setSortAscending(true);
    }
  };

  return null; // we'll build the toolbar next
}
```

`sortOrder`/`sortAscending`/`viewMode` live in the shared `useSettings` store; the genre filter in `useGenreFilter`; the genre list comes from `useFeedGenres`. Nothing here forks per platform - the menu just calls these setters, and `useFilteredGamesByYear()` re-derives the visible feed automatically. `toolbarIcon` is the baseline's cross-platform icon resolver (`app/utils/useToolbarIcons.ts`) - it returns an SF Symbol on iOS and a rasterized Material Symbol on Android; **Exercise 3 unpacks why it has to work that way.**

Now the toolbar. Swap the `return null` for a single menu button anchored to the right. Give it the filter icon via the `toolbarIcon` resolver:

```tsx
return (
  <Stack.Toolbar placement="right">
    <Stack.Toolbar.Menu icon={toolbarIcon("filter")}>
      {/* menu sections go here */}
    </Stack.Toolbar.Menu>
  </Stack.Toolbar>
);
```

A `Stack.Toolbar.Menu` placed in the toolbar becomes the tappable header button; its children become the menu that opens when you tap it. The `icon` prop sets the glyph on that button.

### Build the menu sections

Native menus support **inline sections** - visually grouped rows separated by dividers. Each one is a nested `<Stack.Toolbar.Menu inline>`, and inside it `MenuAction` is a row:

- `isOn` renders a checkmark - perfect for single-select like sort order or view mode.
- `subtitle` renders secondary text under the label.
- Tapping a row fires `onPress`.

1. Add the **sort section** - tap a row to select it, tap the selected row again to flip direction (this uses the `handleSort` helper you added above):

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

2. Add the **view-mode section** - gallery vs. list:

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

3. Now a **filter submenu**. Here's a neat trick: a nested `Menu` _without_ `inline` becomes a "Filter ›" row that opens a deeper menu. Give it a `Label`, then list the genres plus an "All Items" reset:

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

4. Finally, a **view options** row. This is where the `handleOpenViewOptions` prop pays off - the menu doesn't own the sheet, it just asks the screen to open it (we wire the sheet up in Exercise 2):

```tsx
<Stack.Toolbar.Menu inline>
  <Stack.Toolbar.MenuAction onPress={handleOpenViewOptions}>View Options</Stack.Toolbar.MenuAction>
</Stack.Toolbar.Menu>
```

### Signal active state on the button itself

When a filter is applied, the menu button should _look_ active. `Stack.Toolbar.Menu` takes a `variant` and a `tintColor` - so let's make the trigger prominent when there's a filter on:

```diff
- <Stack.Toolbar.Menu icon={toolbarIcon("filter")}>
+ <Stack.Toolbar.Menu
+   icon={toolbarIcon("filter")}
+   variant={hasFilters ? "prominent" : "plain"}
+   tintColor={hasFilters ? colors.tint : undefined}
+ >
```

### Mount it in the screen

Drop the component into `GameFeedScreen`'s return, above the `ScrollView`. Pass it the callback that flips the sheet's open state:

```diff
  <Screen preset="fixed" contentContainerStyle={$styles.flex1}>
+   <FilterMenu handleOpenViewOptions={() => setViewOptionsOpen(true)} />
+
    <ScrollView>
```

Don't forget the import:

```tsx
import { FilterMenu } from "@/components/FilterMenu";
```

🏃**Try it.** Open the app and tap the header menu. Sort by Rating, then tap Rating again - the subtitle flips between Ascending/Descending and the feed re-sorts. Switch to List. Open Filter ›, pick a genre - the button turns "prominent" and the feed filters in place. Hit Remove Filter. Notice you wrote **zero** imperative navigation code, and `GameFeedScreen` never learned how the menu works - it just handed it one callback.

### Side Quests

- Add a `subtitle` to the **Filter ›** submenu row showing the selected genres (e.g. "Action, Puzzle"). _Heads up: this is only possible on iOS at the moment._
- Make the menu button's SF Symbol change when a filter is active.
- Mark "Remove Filters" as `destructive` and see how iOS renders it.

## Exercise 1: Native search

A feed wants search. Rather than build a custom text input, we'll use each platform's _native_ search affordance. iOS and Android put search in different places by convention, so we deliberately fork the _control_ while sharing the _state_.

The screen already owns the state and hands it to the shared selector (you set this up in Exercise 0):

```tsx
const [searchQuery, setSearchQuery] = useState("");
const { yearGroups, isLoading, isError } = useFilteredGamesByYear(searchQuery);
```

`useFilteredGamesByYear` folds the query into its pipeline (`game.name` includes it, ahead of genre → hideMature → sort), so neither platform touches that logic - they only call `setSearchQuery`.

We'll build the search as a **platform-split component**, `FeedSearch`, that both platforms render at the same call site. The screen never branches on `Platform.OS` - Metro picks the right file per platform.

1. Create the **base** file, **app/components/FeedSearch.tsx**. It defines the shared props and renders nothing - it's the fallback (e.g. web), and both real variants import the props type from here:

```tsx
// app/components/FeedSearch.tsx
export type FeedSearchProps = {
  searchQuery: string;
  onChangeText: (text: string) => void;
};

export function FeedSearch(_props: FeedSearchProps) {
  return null;
}
```

2. Drop it into the feed once, above the `ScrollView`. It renders the right control per platform and no-ops on the fallback:

```diff
    <FilterMenu handleOpenViewOptions={() => setViewOptionsOpen(true)} />
+
+   <FeedSearch searchQuery={searchQuery} onChangeText={setSearchQuery} />

    <ScrollView>
```

```tsx
import { FeedSearch } from "@/components/FeedSearch";
```

Nothing shows up yet - the base variant is a no-op. Let's fill in the two platforms.

### iOS: a bottom toolbar search pill

iOS convention (think Photos, Music) is a search affordance at the bottom. We add a `Stack.Toolbar` with `placement="bottom"`. Collapsed, it's just a magnifier button pushed to the right with a `Spacer`. Tapped, it expands into a focused pill `TextField`.

Because this lives in its own `FeedSearch.ios.tsx`, we **don't** need a `Platform.OS !== "android"` guard - the file extension _is_ the guard. Metro only bundles this file for iOS.

Create **app/components/FeedSearch.ios.tsx**:

```tsx
// app/components/FeedSearch.ios.tsx
import { ComponentRef, useRef, useState } from "react";
import { Pressable, useWindowDimensions, View } from "react-native";
import { Stack } from "expo-router";
import { SymbolView } from "expo-symbols";

import type { FeedSearchProps } from "@/components/FeedSearch";
import { TextField } from "@/components/TextField";
import { useToolbarIcons } from "@/utils/useToolbarIcons";
import { colors } from "@/theme/colors";

export function FeedSearch({ searchQuery, onChangeText }: FeedSearchProps) {
  const { width: windowWidth } = useWindowDimensions();
  const [searchActive, setSearchActive] = useState(false);
  const searchInputRef = useRef<ComponentRef<typeof TextField>>(null);
  const toolbarIcon = useToolbarIcons(colors.brandSurfaceText);

  const closeSearch = () => {
    setSearchActive(false);
    onChangeText("");
  };
  const clearSearch = () => {
    onChangeText("");
    searchInputRef.current?.focus(); 
  };

  return (
    <Stack.Toolbar placement="bottom">
      {searchActive ? (
        <>
          <Stack.Toolbar.View>
            <TextField
              ref={searchInputRef}
              autoFocus
              value={searchQuery}
              onChangeText={onChangeText}
              placeholder="Search games"
              returnKeyType="search"
              containerStyle={{ width: windowWidth - 96 }}
              inputWrapperStyle={{ borderRadius: 999 }} 
              LeftAccessory={(props) => (
                <View style={props.style}>
                  <SymbolView name="magnifyingglass" tintColor={colors.textDim} size={18} />
                </View>
              )}
              RightAccessory={
                searchQuery.length > 0
                  ? (props) => (
                      <Pressable onPress={clearSearch} style={props.style} hitSlop={8}>
                        <SymbolView name="xmark.circle.fill" tintColor={colors.textDim} size={18} />
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

Walking through it: collapsed, the toolbar is a flexible `Spacer` that pushes a magnifier `Button` to the right. Tap it → `searchActive` flips → it expands into a pill `TextField` (`inputWrapperStyle={{ borderRadius: 999 }}`) with a `magnifyingglass` `LeftAccessory`. The `RightAccessory` clear button only renders when there's text - same as the system search fields, and it clears then refocuses so you keep typing. The `xmark` close `Button` is separate: it collapses the whole pill.

🏃**Try it.** On iOS, tap the magnifier in the bottom bar, type "star" - the feed filters in place. The "✕" inside the field clears the text (and keeps you typing); the close button collapses the pill. Don't bother on Android yet - that's next. It looks very slick on iOS 26+!

### Android: a docked search bar

Android convention is a search bar docked at the **top** of the content. We isolate this in **app/components/FeedSearch.android.tsx** so the Android-only `@expo/ui/jetpack-compose` import never loads on iOS:

```tsx
// app/components/FeedSearch.android.tsx
import { View, type ViewStyle } from "react-native";
import { DockedSearchBar, Host, Icon, Text } from "@expo/ui/jetpack-compose";

import { useToolbarIcons } from "@/utils/useToolbarIcons";
import type { FeedSearchProps } from "@/components/FeedSearch";
import { colors } from "@/theme/colors";

export function FeedSearch({ onChangeText }: FeedSearchProps) {
  const searchIcon = useToolbarIcons(colors.text)("search");

  return (
    <View style={$wrapper}>
      <Host useViewportSizeMeasurement style={$host}>
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

The Android variant only uses `onChangeText` - the `DockedSearchBar` keeps its own query internally and reports it live, so it ignores `searchQuery`. Same props type, different needs.

> ⚠️ **Jetpack Compose layout gotcha.** A bare `<Host matchContents>` proposes _unbounded_ width, so `DockedSearchBar` (which fills its width) collapses to nothing. The fix: wrap the `Host` in a plain RN `View` with an explicit height, give the `Host` `flex: 1` plus `useViewportSizeMeasurement` (so it proposes the viewport width). Don't also pass `fillMaxWidth()` - the bar has its own M3 default width.

> ⚠️ **New platform files don't hot-reload reliably.** After _creating_ `FeedSearch.android.tsx`, Fast Refresh often keeps serving a stale bundle and the component looks broken/absent. Force a full reload (kill + relaunch from the dev-client launcher, or restart Metro) before concluding anything is wrong.

🏃**Try it.** On Android, type into the top docked bar - the feed filters in place, same as iOS. Two different controls, one `searchQuery`, one call site.

### Side Quests

- The Android `DockedSearchBar` has **no clear ("✕") button** - it can't host a trailing icon and its query is uncontrolled. See what you can do about it.
- Debounce `setSearchQuery` and observe whether it matters for this dataset size.

## Exercise 2: The universal "View Options" bottom sheet

Now the opposite for this exercise: **when the universal API _is_ the right answer.** The "View Options" row we wired into the menu back in Exercise 0 should open a sheet with a sort picker and a "Hide Mature" toggle. `@expo/ui`'s root (universal) export bridges SwiftUI ↔ Jetpack Compose, so `BottomSheet` + `FieldGroup` + `Picker` + `Switch` give us **one code path** that renders natively on both platforms - no `.ios`/`.android` split needed.

1. Create **app/components/ViewOptionsSheet.tsx**. It takes `isOpen`/`onClose` from the screen (which owns that state) and reads `sortOrder`/`hideMature` from the shared `useSettings` store:

```tsx
// app/components/ViewOptionsSheet.tsx
import { SORT_OPTIONS } from "@/stores/gameFeed";
import { useSettings, type SortOrder } from "@/stores/settings";
import { BottomSheet, FieldGroup, Picker, Switch } from "@expo/ui";

interface ViewOptionsSheetProps {
  isOpen: boolean;
  onClose: () => void;
}

export function ViewOptionsSheet({ isOpen, onClose }: ViewOptionsSheetProps) {
  const { sortOrder, setSortOrder, hideMature, setHideMature } = useSettings();

  return (
    <BottomSheet isPresented={isOpen} onDismiss={onClose} snapPoints={["half", "full"]}>
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
  );
}
```

`sortOrder` and `hideMature` come from the shared `useSettings` store - the same values the Settings screen reads - so changing them here is reflected everywhere, and `useFilteredGamesByYear` re-derives the feed automatically.

2. Mount it in `GameFeedScreen`, at the bottom of the return. It's driven by the same `viewOptionsOpen` state the menu already toggles via `handleOpenViewOptions`:

```diff
    </ScrollView>
+
+   <ViewOptionsSheet isOpen={viewOptionsOpen} onClose={() => setViewOptionsOpen(false)} />
  </Screen>
```

```tsx
import { ViewOptionsSheet } from "@/components/ViewOptionsSheet";
```

Now the loop is closed: the menu's "View Options" row calls `handleOpenViewOptions` → the screen flips `viewOptionsOpen` → the sheet presents. The screen owns the state; the two components on either side of it stay dumb.

> ⚠️ **`snapPoints` gotcha.** Pass `snapPoints={["half", "full"]}`. Omit them and the sheet defaults to `fitToContents`, which collapses a `FieldGroup` to ~0 height - you'll get an invisible sheet and think it's broken.

> 💡 Import discipline: `@expo/ui` (root) is universal. `@expo/ui/swift-ui` and `@expo/ui/jetpack-compose` are platform-only. Reach for a platform import _only_ when the universal API lacks what you need (as with search). Here it didn't - so one import, one component file, both platforms.

🏃**Try it.** On both iOS and Android: open the header menu → View Options. The sheet slides up natively, the picker changes sort order, the toggle hides mature games - and the feed updates live behind the sheet. Same JSX, two native renderers.

### Side Quests

- iOS polish: give the sheet a bold title and a "Done" button.
- Android polish: we used the universal Picker in the `<ViewOptionsSheet />`. It doesn't look the greatest, wire up our `<Dropdown />` from the previous module for a cleaner widget!

## Exercise 3: Android parity - icons & a native carousel

Two Android gaps remain. Both teach the same meta-skill: **make one declaration serve both platforms by resolving per-platform under the hood.**

### Cross-platform toolbar icons

Run Exercise 0's menu on Android and the trigger icon vanishes. Root cause: Android's `Stack.Toolbar` Menu/Button **drop SF Symbol names** - they only render an `ImageSourcePropType`. So we need an SF Symbol on iOS and a real image source on Android, from a _single_ call site.

The trick: rasterize **Material Symbols** to image sources via `expo-symbols` (`unstable_getMaterialSymbolSourceAsync`), and pair each logical icon with both an SF name and a Material name behind a hook.

That hook already ships in the baseline - you've been calling it (`toolbarIcon`) since Exercise 0. Here's what it does, in **app/utils/useToolbarIcons.ts**:

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

Because your `FilterMenu` trigger already sets `icon={toolbarIcon("filter")}`, the resolver does the platform split for you - no change needed. On iOS it hands `Stack.Toolbar.Menu` the SF Symbol name; on Android the same call yields a rasterized Material Symbol image source, so the button keeps its glyph instead of vanishing. The iOS search magnifier and close buttons resolve the same way in `FeedSearch.ios.tsx`, and the Android docked bar's leading icon in `FeedSearch.android.tsx`.

> 💡 **Menu _rows_ carry no icons - on purpose.** Notice the sort/view/filter `MenuAction` rows never got an `icon`. Android overflow menus are conventionally **text-only**, so we let the `isOn` checkmark carry selection on every row, on both platforms. Only the toolbar _trigger_ button and the search bars keep glyphs (those affordances are idiomatic with icons). Keeping row icons off sidesteps the SF-symbol-dropping problem for the rows entirely.

> 💡 Valid names: SF Symbols live in `node_modules/sf-symbols-typescript/dist/index.d.ts` (grep with single quotes - they're single-quoted in the union). Material names are keys of `node_modules/expo-symbols/build/android/symbols.json`.

### A native carousel gallery on Android

iOS gallery view is a horizontal `FlatList` of cards (`GameGallery.tsx`). On Android we can do better with Material 3's `HorizontalMultiBrowseCarousel` - native masking + morph animations as you swipe. The neat part: we **host a RN card inside the Compose tree** with `RNHostView`, so we don't rebuild the card in Compose.

The card the carousel hosts, `GameCarouselCard`, already ships in the baseline (`app/components/GameCarouselCard.tsx`) - it's just a `Pressable` that fills its slot with the cover image and the title overlaid along the bottom edge. We only need to import it; the interesting part is the Compose carousel that arranges it.

Create **app/components/GameGallery.android.tsx** - the carousel that hosts the card:

```tsx
// app/components/GameGallery.android.tsx
import { ViewStyle } from "react-native";
import { Host, HorizontalMultiBrowseCarousel, RNHostView } from "@expo/ui/jetpack-compose";

import { GameCarouselCard } from "./GameCarouselCard";
import type { Game } from "../services/api/types";

const CAROUSEL_HEIGHT = 220;
const PREFERRED_ITEM_WIDTH = 220; // focused item; the carousel sizes peeking items around it

export function GameGallery({ games }: { games: Game[] }) {
  const carouselKey = games.map((game) => game.id).join(",")

  return (
    <Host key={carouselKey} style={{ height: CAROUSEL_HEIGHT } satisfies ViewStyle}>
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

Because this is `GameGallery.android.tsx`, Metro automatically picks it for Android and the plain `GameGallery.tsx` (FlatList) for iOS - `YearSection` imports `"./GameGallery"` and never knows the difference.

> ⚠️ **`RNHostView` image sizing.** An RN `<Image>` inside `RNHostView` needs a concrete flex/size (the card uses `flex: 1` to fill the slot). `position: absolute`/`absoluteFill` images never lay out and silently fail to load inside the Compose host.

🏃**Try it.** On Android: the filter trigger icon renders (Material Symbol), and switching to Gallery view shows the native carousel - swipe and watch items mask/morph at the edges. On iOS: SF Symbols and the FlatList gallery, unchanged. One `YearSection`, one `toolbarIcon(...)` resolver, two native experiences.

### Side Quests

- Render Gallery/List on Android as a Material `SegmentedButton` instead of menu rows.

## See the solution

Switch to branch: [`03-composite-ui`](https://github.com/infinitered/cr-2026-intermediate-workshop-template/tree/03-composite-ui)
