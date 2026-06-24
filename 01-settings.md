# Module 01: Expo UI Appetizers -  universal and platform-specific native form components

### Goal

Expo UI is a great choice for forms. We can often change these to look more like the operating system defaults without feeling like we need to overhaul the entire app.

### Concepts

- Expo UI universal controls- a subset of sensible defaults that look good on Android, iOS, and web
- Breaking out into platform-specific controls when we feel like the universal controls aren't enough

### New components

TODO: add a little description for all the new Expo UI components used.


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

There is no date picker in the univeral component library. Therefore, we'll make our own "universal" component that we'll then import into the Settings screen and use it inline with the rest of the univeral components.

The universal components under the hood use platform-specific file extensions to create distinct Android, iOS, and web components that can be imported into other files as a single universal component, e.g.,:

- **MyComponent.tsx**
- **MyComponent.ios.tsx**
- **MyComponent.android.tsx**

For each platform, Metro Bundler will only include the platform-specific file if it's available. If you're building for iOS, the ".ios" file becomes "the" MyComponent file. The unprefixed version is the default or fallback, so it would be used for web in this example.

1. In the **app/components** folder, let's create our platform-specific files (we only need them and not the fallback since we're not targeting web). These will be: **DatePicker.android.tsx** and **DatePicker.ios.tsx**.

2. Both files have the same basic boilerplate, add this to both:

```tsx
type DatePickerProps = {
  title: string
  value: Date
  onDateChange: (date: Date) => void
  maximumDate?: Date
}

export function DatePicker({ title, value, onDateChange, maximumDate }: DatePickerProps) {

}
```

### iOS DatePicker

This one is pretty easy. We'll just drop in the Expo UI SwiftUI date picker.

1. Add these imports to **DatePicker.ios.tsx**:

```tsx
import { DatePicker as SwiftUIDatePicker } from "@expo/ui/swift-ui"
import { datePickerStyle } from "@expo/ui/swift-ui/modifiers"
```

> Modifiers are a way to access a wide variety of properties that are available on SwiftUI and Jetpack Compose components. Some modifiers can apply to many different components.

2. Implement the date picker inside the component function:

```tsx
<SwiftUIDatePicker
  title={title}
  selection={value}
  onDateChange={onDateChange}
  displayedComponents={["date"]}
  range={maximumDate ? { end: maximumDate } : undefined}
  modifiers={[datePickerStyle("compact")]}
/>
```

3. In **SettingsScreen.tsx** import your DatePicker and add it under the Profile section:

```tsx
 <DatePicker
  title="Birth Date"
  value={dateValue}
  onDateChange={(date: Date) => setBirthDate(date.toISOString())}
  maximumDate={new Date()}
/>
```

🏃**Try it.** Don't load it on Android yet, it'll crash!

### Android DatePicker

Android's date picker is really just a dialog, so our major customization here will be adding the field group row UI.

1. Add the basic UI:

```tsx
<Row alignment="center">
  <Text>{title}</Text>
  <Spacer flexible />
  <Text>{formatted}</Text>
</Row>
```

2. Add a function to format the date before the component return statement:

```tsx
 const formatted = value.toLocaleDateString("en-US", {
    month: "short",
    day: "numeric",
    year: "numeric",
  })
```

🏃**Try it.** Android doesn't crash now! But you can't pick a date, either:

3. We're going to add the dialog component right under the formatted text, but not inside a container control because they don't need to be arranged relative to each other (remember, one is a dialog). Wrap everything in the return statement:

```diff
+ <>
<Row alignment="center">
  <Text>{title}</Text>
  <Spacer flexible />
  <Text>{formatted}</Text>
</Row>
+ </>
```

4. Before the `</>`, add the date picker:

```tsx
<DatePickerDialog
  initialDate={value.toISOString()}
  onDateSelected={(date: Date) => {
    onDateChange(date)
  }}
  selectableDates={maximumDate ? { end: maximumDate } : undefined}
/>
```

5. This won't do anything yet, because the dialog is hidden by default. Add some state to track if the dialog is shown:

```tsx
  const [showDialog, setShowDialog] = useState(false)
```

6. Add a function in the display row to show the dialog when the date is tapped:

```diff
+<Row alignment="center" onPress={() => setShowDialog(true)}>
-<Row alignment="center">
  <Text>{title}</Text>
  <Spacer flexible />
  <Text>{formatted}</Text>
</Row>
```

7. Make the dialog display conditional and dismiss it whene selecting a date or tapping away from it:

```diff
+{showDialog && (
  <DatePickerDialog
    initialDate={value.toISOString()}
    onDateSelected={(date: Date) => {
      onDateChange(date)
+      setShowDialog(false)
    }}
+    onDismissRequest={() => setShowDialog(false)}
    selectableDates={maximumDate ? { end: maximumDate } : undefined}
  />
+)}
```

🏃**Try it.** Should be ready for all your date-pickin' needs! 

## Exercise 3: Drop down to a better drop down



```
code sample
```

<details>
  <summary>expanding code sample</summary>

</details>

🏃**Try it.** Open up the app after changing the settings. How well can you navigate around? Log in (if not already), scroll down on the lists, switch tabs.

## Side Quests

Notice that we kind of broke the "universal" nature of our app with the DatePicker and Dropdown controls- they no longer support web!

It's not big deal for us because we're just planning on targeting mobile, but you can fix it. If you add a plain **DatePicker.tsx** or **Picker.tsx**, that will be the fallback implementation when the ios and android suffixed files are not in use.

Suggestions:
- For the DatePicker, use the old one from before we converted the Settings screen.
- For the Dropdown, use the `Picker` from the Expo UI universal components (I think it looks OK on web).

## See the solution

Switch to branch: [`01-blending-in-solution`](https://github.com/infinitered/cr-2024-intermediate-workshop-template/tree/01-blending-in-solution)
