# Module 01: Expo UI Appetizers -  universal and platform-specific native form components

### Goal

Expo UI is a great choice for forms. We can often change these to look more like the operating system defaults without feeling like we need to overhaul the entire app.

### Concepts

- Expo UI universal controls- a subset of sensible defaults that look good on Android, iOS, and web
- Breaking out into platform-specific controls when we feel like the universal controls aren't enough


### Features to build

- First pass on the Settings screen with Expo UI univeral controls
- Create our own controls based on Expo UI platform-specific bindings for the date picker and dropdown.

### Resources

- React Native docs
  - [Expo UI universal controls](https://docs.expo.dev/versions/latest/sdk/ui/universal/)

# Exercises

## Exercise 0: Build the app (if you haven't already)
1. Run `pnpm install`
2. Run `npx expo run:ios` or `npx expo run:android` (or both, you're gonna be working on both)

## Exercise 1: First pass with Expo UI universal controls

The Setting screen is mainly groups of fields and pretty standard controls- text inputs, switches, sliders, etc. Most of these are already in the universal controls library.

The universal components in @expo/ui are a single-API layer over the platform-native UI toolkits. On Android, they delegate to `@expo/ui/jetpack-compose`. On iOS, they delegate to `@expo/ui/swift-ui`. On web, they're JS implementations using `react-dom` or `react-native-web` and are picked per component to suit the control.

### Clearing things out and (almost) starting over.

1. You can go ahead delete everything inside of `<Screen>` inside of **app/screens/SettingScreen**. You won't need it anymore.

```tsx
<Screen preset="fixed" contentContainerStyle={$screenContent}>
   { /* It's gone! */ }
</Screen>
```

2. Import `Host` and other controls you need from `@expo/ui`:

```tsx
import { Host, FieldGroup, TextInput, Switch, Slider, Button, Picker, Row, Column, Text, RNHostView, Spacer } from "@expo/ui"
```

Add the host inside of screen, and the field group (all the controls go inside there, this helps lay them out):

```diff
<Screen preset="fixed" contentContainerStyle={$screenContent}>
+   <Host style={{ flex: 1 }} useViewportSizeMeasurement>
+      <FieldGroup>

+      </FieldGroup>
+   </Host>
</Screen>
```

SwiftUI and Jetpack Compose controls behave by different layout rules than built-in React Native components, so they need to live in the host and we need to tell the host what size it should be (e.g., the size of the screen minus the tab bar and status bar).

### Let's drop some controls!

We'll go group by group, ignoring controls that we can't port over yet.

1. Add the Profile group:

```tsx
  <FieldGroup.Section title="Profile">
    <TextInput
      placeholder="Enter your name"
      defaultValue={displayName}
      onChangeText={setDisplayName}
    />
  </FieldGroup.Section>
```

We're ignoring the birth date field because there's no date picker in the uinversal controls - we'll be back!

2. Now the shipping address group:

```tsx
<FieldGroup.Section title="Shipping Address">
  <TextInput
    placeholder="Street Address"
    defaultValue={shippingAddress.street1}
    onChangeText={(v) => setShippingAddress({ street1: v })}
    autoCapitalize="words"
  />
  <TextInput
    placeholder="Apt / Suite / Unit"
    defaultValue={shippingAddress.street2}
    onChangeText={(v) => setShippingAddress({ street2: v })}
    autoCapitalize="words"
  />
  <TextInput
    placeholder="City"
    defaultValue={shippingAddress.city}
    onChangeText={(v) => setShippingAddress({ city: v })}
    autoCapitalize="words"
  />
  <TextInput
    placeholder="State"
    defaultValue={shippingAddress.state}
    onChangeText={(v) =>
      setShippingAddress({ state: v })
    }
    autoCapitalize="words"
  />
  <TextInput
    placeholder="ZIP Code"
    defaultValue={shippingAddress.zip}
    onChangeText={(v) =>
      setShippingAddress({ zip: v.replace(/[^0-9]/g, "").slice(0, 5) })
    }
    keyboardType="number-pad"
    maxLength={5}
  />
</FieldGroup.Section>
```

3. Finally, queue preferences:

```tsx
<FieldGroup.Section title="Queue Preferences">
  <Switch value={hideMature} onValueChange={setHideMature} label="Hide Mature Content" />
  <Column>
    <Text>{`Minimum Rating: ${minRating} / 5`}</Text>
    <Slider value={minRating} onValueChange={setMinRating} min={1} max={5} step={1} />
  </Column>
  <Button label="Favorite Genres" onPress={() => router.push("/favorite-genres")} />
</FieldGroup.Section>
```

🏃**Try it.** Load that up on Android and iOS. Looks pretty good! That switch is ehhh on Android, maybe we'll work on that later.

## Exercise 2: DatePicker Story

## Exercise 3: Drop down to a better drop down



```
code sample
```

<details>
  <summary>expanding code sample</summary>

</details>

🏃**Try it.** Open up the app after changing the settings. How well can you navigate around? Log in (if not already), scroll down on the lists, switch tabs.

## Side Quests

- ???
- ???

## See the solution

Switch to branch: [`01-blending-in-solution`](https://github.com/infinitered/cr-2024-intermediate-workshop-template/tree/01-blending-in-solution)
