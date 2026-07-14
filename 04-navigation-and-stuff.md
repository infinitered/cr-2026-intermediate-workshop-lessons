# Module 03: Cutting across routes - native navigation and other things

### Goal

Let's use Expo Router-specific native UI functionality to adopt modern platform-native UX that spans multiple screens.

### Concepts

- Platform native tab functionality
- Headers and scrolling
- Platform native shared element transitions
- Android dynamic colors

### Features to build

- Add native tabs
- Adopt large headers on iOS
- Move search to tab/header functionality
- Make a smooth zoom transition on iOS from game list to detail
- Switch color theme over to dynamic colors on Android

### Resources

- TBD

# Exercises

## Exercise 1: Native tabs

We waited until the afternoon, but could actually be one of the first things you do when starting to add more platform native UI to your app. Native tabs are a pretty straightforward replacement for cross-platform JS tabs.

1. Clear out everything in **src/app/(tabs)/\_layout.tsx** and add the imports and outer tabs structure:

```tsx
import { NativeTabs } from "expo-router/unstable-native-tabs"

export default function TabsLayout() {
  return <NativeTabs></NativeTabs>
}
```

2. Each tab is represented in here as a tab trigger. The name of the trigger must match the route name. Add these inside `<NativeTabs>`:

```tsx
<NativeTabs.Trigger name="index">
  <NativeTabs.Trigger.Label>Games</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>

<NativeTabs.Trigger name="queue">
  <NativeTabs.Trigger.Label>Queue</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>

<NativeTabs.Trigger name="settings">
  <NativeTabs.Trigger.Label>Settings</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>
```

3. We will want to follow platform conventions for the tab icons. iOS strongly encourages use of SF Symbols:

```diff
<NativeTabs.Trigger name="(home)/index">
+  <NativeTabs.Trigger.Icon sf={{ default: "gamecontroller", selected: "gamecontroller.fill" }} />
  <NativeTabs.Trigger.Label>Games</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>

<NativeTabs.Trigger name="queue">
+  <NativeTabs.Trigger.Icon sf={{ default: "list.bullet", selected: "list.bullet" }} />
  <NativeTabs.Trigger.Label>Queue</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>

<NativeTabs.Trigger name="settings">
+  <NativeTabs.Trigger.Icon  sf={{ default: "gearshape", selected: "gearshape.fill" }} />
  <NativeTabs.Trigger.Label>Settings</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>
```

Using separate "fill" icons give the selection state a distinct iOS look.

> Here's where you can look up all the different options for SF Symbols: TODO

4. We can just as easily use built-in Material Design icons on Android:

```diff
<NativeTabs.Trigger name="index">
-  <NativeTabs.Trigger.Icon sf={{ default: "gamecontroller", selected: "gamecontroller.fill" }} />
+  <NativeTabs.Trigger.Icon md="sports_esports" sf={{ default: "gamecontroller", selected: "gamecontroller.fill" }} />
  <NativeTabs.Trigger.Label>Games</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>

<NativeTabs.Trigger name="queue">
-  <NativeTabs.Trigger.Icon sf={{ default: "list.bullet", selected: "list.bullet" }} />
+ <NativeTabs.Trigger.Icon md="list" sf={{ default: "list.bullet", selected: "list.bullet" }} />
  <NativeTabs.Trigger.Label>Queue</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>

<NativeTabs.Trigger name="settings">
-  <NativeTabs.Trigger.Icon sf={{ default: "gearshape", selected: "gearshape.fill" }} />
+ <NativeTabs.Trigger.Icon md="settings" sf={{ default: "gearshape", selected: "gearshape.fill" }} />
  <NativeTabs.Trigger.Label>Settings</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>
``

> No shortage of options for Material Design icons, either! Check them out here: TODO

🏃**Try it.** Looks like tabs and the tabs look native!

## Exercise 2: Safe areas and large headers

You might be happy that the native tabs are up, but maybe a little less happy with the missing navigation headers and content flowing under the safe areas. We're going to a do a little prep first so we can make all the changes we need.

In particular, with the native tabs, we no longer get the option of configuring a per-tab JS stack, so now we need to do the same thing we did with the Games tab with every tab. These stacks within tabs might not be useful yet in our app for anything other than apperances, but having a stack outside the tabs and inside the tabs is a common high-leverage design pattern for Expo Router, so you'll quite likey use this in the future.

1. Create the following three folders inside **src/app/tabs**:
- queue
- settings

2. Create a **_layout.tsx** file and put this code inside of each one in order to give each tab its own stack:

```tsx
import { Stack } from "expo-router"

export default function Layout() {
  return (
    <Stack>
      <Stack.Screen name="index" />
    </Stack>
  )
}
```

4. Add a name for each stack, like this:

```diff
import { Stack } from "expo-router"

export default function Stack() {
  return (
    <Stack>
-      <Stack.Screen name="index" />
+      <Stack.Screen name="index" options={{ title: "Games"}} />
    </Stack>
  )
}
```

Repeat for "Queue" and "Settings".

5. We're going to de-style the Games tab so it matches the other ones. This will set the stage for other changes we're adding later. Remove the stack header styles from **src/app/(tabs)/(home)/index.tsx**:

```diff
<Stack
-  screenOptions={{
-    headerStyle: { backgroundColor: colors.brandSurface },
-    headerTintColor: colors.brandSurfaceText,
-    headerTitleStyle: { fontFamily: typeScale.headline1.fontFamily },
-  }}
>
  <Stack.Screen name="index" options={{ title: "Games" }} />
</Stack>
```

(you'll be able to remove some other references now, as well)

🏃**Try it.**  Each tab should have a plain stack header now.

### ~Interlude~ Fix bottom offsets

6. Add some bottom padding in **GameFeedScreen.tsx**:

```diff
+import { useSafeAreaInsets } from "react-native-safe-area-context"
// ...
+const { bottom } = useSafeAreaInsets()
// ...
-<ScrollView style={$styles.flex1}>
+<ScrollView contentContainerStyle={{ paddingBottom: bottom }} style={$styles.flex1}>
```

(That magnifying glass icon is going to look funny, we'll deal with that)

7. Add some bottom padding in **QueueScreen.tsx**:

```diff
+import { useSafeAreaInsets } from "react-native-safe-area-context"
// ...
+const { bottom } = useSafeAreaInsets()
// ...
-<View style={[themed($bottomButton)}]}>
+<View style={[themed($bottomButton), { paddingBottom: bottom + 16 }]}>
```

8. Add some bottom padding in **SettingsScreen.tsx**:

```diff
+import { useSafeAreaInsets } from "react-native-safe-area-context"
// ...
+const { bottom } = useSafeAreaInsets()
// ...
-<Screen preset="scroll" contentContainerStyle={[themed($container)]}>
+<Screen preset="scroll" contentContainerStyle={[themed($container), { paddingBottom: bottom }]}>
```

### Large headers

6. Before we move on, let's fix those 

6. Add the large header option to each layout, like this. Also required for collapsing on scroll is making it transparent:

```diff
import { Stack } from "expo-router"

export default function Stack() {
  return (
    <Stack>
-      <Stack.Screen name="index" options={{ title: "Games"}} />
+      <Stack.Screen name="index" options={{ title: "Games", headerLargeTitle: true, headerTransparent: true }} />
    </Stack>
  )
}
```

Repeat for the other two tabs.

7. You might be wondering about the shrink-on-scroll of the header that you see in a lot of apps. We can do this, but theres's some requirements for it to recognize scroll activity. Inside **GameFeedScreen**, update the Scrollview to include this prop:

```diff
<ScrollView
+  contentInsetAdjustmentBehavior="automatic"
>
```

This also takes care of keeping the content below the large header.

8. The scrollview also needs to be the topmost element on the screen. Remove the `Screen` component:

```diff
- <Screen preset="fixed">
+ <>
```

Also, move all the `Stack.Toolbar` stuff below the `ScrollView` (not sure why, but let's go with it - the scroll view is technically supported to be the top-level control, anyway)

Repeat for each tab.

🏃**Try it.**  Scrolling down should now minimize the header.

9. On iOS, the header may overlap with the content. We can fix this by making the headers transparent. It doesn't give nearly as good of a result on Android, so let's make it platform specific:

```diff
import { Stack } from "expo-router"

export default function Stack() {
  return (
    <Stack>
-      <Stack.Screen name="index" options={{ title: "Games", headerLargeTitle: true, headerTransparent: true }} />
+      <Stack.Screen name="index" options={{ title: "Games", headerLargeTitle: true, ...(Platform.OS === "ios" && { headerTransparent: true }), }} />
    </Stack>
  )
}
```

🏃**Try it.**  Now the filter button fits in quite nicely with the new header effects

## Exercise 3: Search Part 2

That stray search button is probably really gnawing at you right now. Fortunately, there's some great ways to integrate search directly into navigation headers and tabs.

### The header SearchBar

We have a hunch that we'll like this more on Android than iOS, but it kind of works for both, so let's give it a go.

1. Add the `Stack.SearchBar` component next to the other `Stack.Toolbar` components, inside the `<>`:

```tsx
<Stack.SearchBar
  placeholder="Search games..."
  onChangeText={(e) => setSearchQuery(e.nativeEvent.text)}
/>
```

2. Remove the `FeedSearch` component to get rid of the old Android search bar.

3. Remove the whole `Stack.Toolbar` conditional section that starts with this to remove the old iOS search bar:

```tsx
{Platform.OS !== "android" && (
        <Stack.Toolbar placement="bottom">
```

🏃**Try it.**  Check out the search bar in the header on Android. It also shows up on iOS, but below the header when you swipe down a bit.

### The iOS search tab

Let's suppose we don't like the way iOS does it and would rather have a separate search tab.

4. Disable the search bar on iOS:

```diff
+{Platform.OS !== "ios" ? (
<Stack.SearchBar
  placeholder="Search games..."
  onChangeText={(e) => setSearchQuery(e.nativeEvent.text)}
/>
+/>) : null}
```

5. Add a **search** folder inside **src/app/(tabs)** and create a **_layout.tsx** and an **index.tsx** file inside the folder.

6. Add this code to **_layout.tsx**:

```tsx
import { Stack } from "expo-router"

export default function Layout() {
  return <Stack />
}
```

7. Add this code to **index.tsx**. We're just implementing a filter-able list of games, nothing we haven't done already. That said, if you have time and the desire, feel free to make your own!

<details>
  <summary>Expand for all the code</summary>
```tsx
import { useMemo, useState } from "react"
import { Image, View } from "react-native"
import { Stack, useRouter } from "expo-router"
import {
  Button,
  ContentUnavailableView,
  Host,
  HStack,
  List,
  RNHostView,
  Text,
  VStack,
} from "@expo/ui/swift-ui"
import { buttonStyle, font, foregroundStyle, listStyle } from "@expo/ui/swift-ui/modifiers"

import { useGamesByYear } from "@/services/api/games"
import type { Game } from "@/services/api/types"

export default function SearchScreen() {
  const router = useRouter()
  const [searchText, setSearchText] = useState("")
  const { data: yearGroups } = useGamesByYear()

  const allGames = useMemo(() => {
    if (!yearGroups) return []
    return yearGroups.flatMap((group) => group.games)
  }, [yearGroups])

  const filteredGames = useMemo(() => {
    if (!searchText.trim()) return allGames
    const query = searchText.toLowerCase()
    return allGames.filter((game) => game.name.toLowerCase().includes(query))
  }, [allGames, searchText])

  return (
    <>
      <Stack.SearchBar
        placeholder="Search games..."
        onChangeText={(e) => setSearchText(e.nativeEvent.text)}
      />
      <Host style={{ flex: 1 }}>
        {filteredGames.length > 0 ? (
          <List modifiers={[listStyle("plain")]}>
            {filteredGames.map((game) => (
              <Button
                key={game.id}
                onPress={() => router.push(`/game/${game.id}`)}
                modifiers={[buttonStyle("plain")]}
              >
                <SearchRow game={game} />
              </Button>
            ))}
          </List>
        ) : (
          <ContentUnavailableView
            title={searchText ? "No Results" : "Search for a game"}
            systemImage={searchText ? "magnifyingglass" : "gamecontroller"}
            description={
              searchText ? `No games match "${searchText}"` : "Type to search for games"
            }
          />
        )}
      </Host>
    </>
  )
}

function SearchRow({ game }: { game: Game }) {
  return (
    <HStack spacing={12} alignment="center">
      <VStack modifiers={[frame({ width: 48, height: 48 }), clipShape("roundedRectangle", 6)]}>
        <RNHostView>
          {game.background_image ? (
            <Image source={{ uri: game.background_image }} style={{ width: 48, height: 48 }} />
          ) : (
            <View style={{ width: 48, height: 48, backgroundColor: "#ccc" }} />
          )}
        </RNHostView>
      </VStack>
      <VStack alignment="leading" spacing={2}>
        <Text modifiers={[font({ size: 15, weight: "semibold" })]}>{game.name}</Text>
        <Text
          modifiers={[
            font({ size: 12 }),
            foregroundStyle({ type: "hierarchical", style: "secondary" }),
          ]}
        >
          {game.genres?.map((g) => g.name).join(", ") ?? ""}
        </Text>
      </VStack>
    </HStack>
  )
}
```
</details>

8. In **src/app/(tabs)/_layout.tsx**, Add the conditional search tab after the index tab:

```tsx
<NativeTabs.Trigger name="search" role="search" hidden={Platform.OS !== "ios"}>
  <NativeTabs.Trigger.Label>Search</NativeTabs.Trigger.Label>
</NativeTabs.Trigger>
```

Note the "search" role.

🏃**Try it.** Searching now brings up an entire separate view.

## Exercise 4: Zoom transitions

iOS includes some cool features for doing platform-native shared element transitions. The zoom transition causes a view to morph from one screen to its equivalent size and position on the next screen. It works particularly well for images.

1. Let's first try just adding the zoom transition "source". Go to **app/components/GameCard.tsx** and surround the touchable with `Link.AppleZoom`:

```diff
+<Link.AppleZoom>
  <TouchableOpacity
    activeOpacity={0.8}
    style={themed($cardOuter)}
    onPress={() => router.push(`/game/${game.id}`)}
  >
    <!-- -->
  </TouchableOpacity>
+</Link.AppleZoom>
```

(`Link` is imported from `expo-router`)

2. You also need to switch the touchable to use a surrounding Link in order to be compatible with this Expo Router-based functionality:

```diff
+<Link href={`/game/${game.id}`} asChild>
<Link.AppleZoom>
  <TouchableOpacity
    activeOpacity={0.8}
    style={themed($cardOuter)}
-    onPress={() => router.push(`/game/${game.id}`)}
  >
    <!-- -->
  </TouchableOpacity>
</Link.AppleZoom>
+</Link>
```

🏃**Try it.** It tries to do something! Interesting...

3. Let's go implement a "zoom target", which gives a hint as to what the source should be transitioning to. Add `Link.AppleZoomTarget` in **app/components/GameDetailScreen.tsx**

```diff
{game.background_image (
+  <Link.AppleZoomTarget>
    <Image source={{ uri: game.background_image }} style={$heroImage} blurRadius={3} />
+  </Link.AppleZoomTarget>
)}
```

🏃**Try it.** Oof, that doesn't look great.

### Making a zoom-able view

We're going to eventually have the cover image take over the entire top of the screen, for a cool effect that takes over the header. 

4. First, let's fix the image as it is. Remove the blur:

```diff
{game.background_image (
  <Link.AppleZoomTarget>
-    <Image source={{ uri: game.background_image }} style={$heroImage} blurRadius={3} />
+    <Image source={{ uri: game.background_image }} style={$heroImage} />
  </Link.AppleZoomTarget>
)}
```

And a small tweak to the `$heroImage` style:

```diff
const $heroImage: ImageStyle = {
  width: "100%",
- height: 180
+ aspectRatio: 3 / 4,
}
```

🏃**Try it.** A small tweak that helps a good bit!

4. Let's get the image to actually take over the header by making it transparent. Add this just inside **src/app/game/[id].tsx**:

```tsx
<Stack.Screen
  options={{
    title: game?.name ?? "",
    ...(Platform.OS === "ios"
      ? { headerTransparent: true, title: "" }
      : { headerShown: true }),
  }}
/>
```

Don't forget to import `Platform` from `react-native` and `Stack` from `expo-router`.


🏃**Try it.** Looks cool, but what about the buttons?

5. Add the buttons back as native stack toolbar buttons. This code goes right below `Stack.Screen`:

```tsx
{Platform.OS === "ios" && (
  <Stack.Toolbar placement="left">
    <Stack.Toolbar.Button icon="chevron.backward" onPress={() => router.back()} />
  </Stack.Toolbar>
)}
<Stack.Toolbar placement="right">
  <Stack.Toolbar.Button
    icon={Platform.OS === "ios" ? "square.and.arrow.up" : ShareAndroid}
    onPress={() => {
      if (!game) return
      const message = game.website
        ? `Check out ${game.name}! ${game.website}`
        : `Check out ${game.name}!`
      Share.share({ message })
    }}
  />
</Stack.Toolbar>
```

Also add an import to a Material Symbols share icon:

```tsx
import ShareAndroid from "@expo/material-symbols/share.xml"
```

🏃**Try it.** Looks good, and works, too! Also a good opportunity to see what happens on Android. Android doesn't have the zoom transition, so leaving the header looks solid.

6. The zoom transition works from the Search tab, as well! Tweak **src/app/(tabs)/search/index.tsx**:

```diff
<RNHostView>
+ <Link.AppleZoom>
  {game.background_image ? (
    <Image source={{ uri: game.background_image }} style={{ width: 48, height: 48 }} />
  ) : (
    <View style={{ width: 48, height: 48, backgroundColor: "#ccc" }} />
  )}
+ </Link.AppleZoom>
</RNHostView>
```

## Exercise 5: Android dynmaic/ Material You colors

You can access platform-native color pallettes via the `Color` API in `expo-router` [link](https://docs.expo.dev/router/reference/color/). This includes the static platform colors for iOS and Android. But, since the Jetpack Compose Expo UI components default to Material You on Android 12+ (where their colors adapt to your wallpaper background or a selection of a theme in the OS settings), we can use the `useMaterialColors` hook to access those directly.

OH WAIT, this is wrong, hang on...


## Side Quests

- Experimental native stack: TBD

## See the solution

Switch to branch: [`01-blending-in-solution`](https://github.com/infinitered/cr-2024-intermediate-workshop-template/tree/01-blending-in-solution)
