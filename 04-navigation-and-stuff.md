# Module 03: Cutting across routes - native navigation and other things

### Goal

Let's use Expo Router-specific native UI functionality to adopt modern platform-native UX that spans multiple screens.

### Concepts

- something
- something else

### Features to build

- Update Settings screen controls
- Update queue and filter lists to use native list controls with sorting and deletion gestures

### Resources

- React Native docs
  - [blahblah](https://reactnative.dev/docs/text#allowfontscaling)

# Exercises

## Exercise 1: Native tabs

We waited until the afternoon, but could actually be one of the first things you do when starting to add more platform native UI to your app. Native tabs are a pretty straightforward replacement for cross-platform JS tabs.

1. Clear out everything in **src/app/(tabs)/\_layout.tsx** and add the imports and outer tabs structure:

```tsx
import { NativeTabs } from "expo-router/unstable-native-tabs";

export default function TabsLayout() {
  return <NativeTabs></NativeTabs>;
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
<NativeTabs.Trigger name="index">
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

In particular, with the native tabs, we no longer get the option of configuring a per-tab stack... at least not without defining stacks in each tab. These stacks within tabs might not be useful yet in our app for anything other than apperances, but having a stack outside the tabs and inside the tabs is a common high-leverage design pattern for Expo Router, so you'll quite likey use this in the future.

1. Create the following three folders inside **src/app/tabs**:
- index
- queue
- settings

TODO: add picture of file tree

2. Create a **_layout.tsx** file and put this code inside of each one in order to give each tab its own stack:

```tsx
import { Stack } from "expo-router"

export default function Stack() {
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

🏃**Try it.**  Each tab should have a stack header now.

### Large headers

5: TODO: I'm not sure if bottom offset will break when these other screens are using Expo UI. If it does, then we'll add a step to fix.

6. Add the large header option to each layout, like this:

```diff
import { Stack } from "expo-router"

export default function Stack() {
  return (
    <Stack>
-      <Stack.Screen name="index" options={{ title: "Games"}} />
+      <Stack.Screen name="index" options={{ title: "Games", headerLargeTitle: true }} />
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

8. The scrollview also needs to be the topmost element on the screen. Remove the `Screen` component:

```diff
- <Screen preset="fixed">
+ <>
```

Repeat for each tab.

🏃**Try it.**  Scrolling down should now minimize the header.

9. On iOS, the header may overlap with the content. We can fix this by making the headers transparent. It doesn't give nearly as good of a result on Android, so let's make it platform specific:

```diff
import { Stack } from "expo-router"

export default function Stack() {
  return (
    <Stack>
-      <Stack.Screen name="index" options={{ title: "Games", headerLargeTitle: true }} />
+      <Stack.Screen name="index" options={{ title: "Games", headerLargeTitle: true }} />
    </Stack>
  )
}
```

### Toolbar buttons

With our new headers, let's refactor our header buttons some to fit better with our new design.

10. In **GameFeedScreen.tsx**, Add the toolbar button just under the `<>`:

```tsx
<Stack.Toolbar placement="right">
  <Stack.Toolbar.Button
    icon={
      Platform.OS === "ios"
        ? showFilters
          ? "line.3.horizontal.decrease.circle.fill"
          : "line.3.horizontal.decrease.circle"
        : FilterList
    }
    onPress={() => setShowFilters((v) => !v)}
  />
</Stack.Toolbar>
```

Import the Material icon we're using:

```tsx
import FilterList from "@expo/material-symbols/filter_list.xml"
```

11. Remove any old toolbar code, such as what was in `useLayoutEffect`

🏃**Try it.**  Now the filter button fits in quite nicely with the new header effects

## Exercise 3: Search

Do android first, see how it affects iOS

Then disable the inline search for iOS, do the new tab

## Exercise 4: More tab tricks

- minimize on scroll
- scroll to top
- built-in keyboard avoidance

## Exercise 5: Zoom transitions

## Exercise 6: Android colors

### Subheading

Blah Blah

```
code sample
```

<details>
  <summary>expanding code sample</summary>

</details>

🏃**Try it.** Open up the app after changing the settings. How well can you navigate around? Log in (if not already), scroll down on the lists, switch tabs.

## Side Quests

- Experimental native stack

## See the solution

Switch to branch: [`01-blending-in-solution`](https://github.com/infinitered/cr-2024-intermediate-workshop-template/tree/01-blending-in-solution)
