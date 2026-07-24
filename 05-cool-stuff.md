# Module 05: Cool Stuff - deep OS integration with a share extension

### Goal

This one isn't strictly about Expo UI, but it's something we thought was pretty neat. It's actually been a requested feature back from [2017](https://expo.canny.io/feature-requests/p/share-extension-ios-share-intent-android). If you've been around the Expo world that long, you might be familiar with this post! Luckily for us, support for this landed in [Expo 55](https://expo.dev/changelog/sdk-55#experimental-support-for-receiving-shared-data-in-expo-sharing), what a time to be alive!

We'll be using native code to reach _outside_ our app and integrate with the operating system itself. We'll build a **share target** so that when a user shares an image from Photos (or any other app like the browser), our app shows up in the iOS/Android share sheet, catches the image, and lets them add it as a new game - all without leaving the share flow.

Then we'll build the receiving screen with the same Expo UI universal controls from Module 01, reuse the `Dropdown` we already made, and finish with a native close button using `Stack.Toolbar` from Module 04.

### Concepts

- Registering our app as a **share target** with `expo-sharing`, including a native iOS share extension
- Deep-linking from the share extension back into our Expo Router app via `+native-intent`
- Reading incoming shared content with the `useIncomingShare` hook
- Reusing Expo UI universal controls and our own platform-specific `Dropdown` on a brand-new screen
- Adding a native header button with `Stack.Toolbar`

### New components / APIs

- [`expo-sharing`](https://docs.expo.dev/versions/latest/sdk/sharing/) - the `useIncomingShare` hook and the config-plugin that generates the native share extension
- [Expo Router `+native-intent`](https://docs.expo.dev/router/advanced/native-intent/) - intercept and rewrite incoming deep links before they hit your routes
- [`Stack.Toolbar`](https://docs.expo.dev/router/advanced/stack/#toolbar) - native header/toolbar buttons and menus

### Resources

- [Expo config plugins](https://docs.expo.dev/config-plugins/introduction/) (for understanding the share extension setup)
- [Apple - Share Extensions](https://developer.apple.com/design/human-interface-guidelines/sharing-and-actions)

### Features to build

- A `/shared` route that receives shared content
- An "Add a Game" screen that reads the shared image and collects a title, year, and genre
- A native close button in the header

# Exercises

## Exercise 0: Build the app (if you haven't already)

1. Run `pnpm install`
2. Run `npx expo run:ios` or `npx expo run:android`

> Heads up: a share extension is _native_ code. If you were starting from a bare template you'd have to prebuild before the extension would show up in the share sheet. We've already wired all of that up for you (see the next section), so a normal dev-client build is all you need.

## The plumbing that's already done for you

You do **not** need to write anything in this section - it's already in the repo. But it's worth understanding what's happening, because it's the part that makes the "share into my app" magic work, and it's the part that requires a rebuild (which is why we did it for you ahead of time).

### `app.config.ts` - registering the share target

Down in the `plugins` array of **app.config.ts** you'll find the `expo-sharing` plugin, configured for both platforms:

```ts
[
  "expo-sharing",
  {
    ios: {
      enabled: true,
      extensionBundleIdentifier: "com.infinitered.cr2026intermediateworkshop.share-extension",
      appGroupId: "group.com.infinitered.cr2026intermediateworkshop",
      activationRule: {
        supportsText: true,
        supportsWebUrlWithMaxCount: 1,
        supportsImageWithMaxCount: 3,
      },
    },
    android: {
      enabled: true,
      singleShareMimeTypes: ["text/plain", "image/*"],
      multipleShareMimeTypes: ["image/*"],
    },
  },
],
```

A few things worth calling out:

- **iOS gets a whole separate native target.** `extensionBundleIdentifier` is the bundle ID of the share _extension_ - a mini app that iOS runs inside the system share sheet. The `appGroupId` is the shared container both the extension and the main app can read from, which is how the shared image gets handed off.
- **`activationRule`** is what tells iOS "our app can accept these kinds of things." We opted into text, a single web URL, and up to 3 images. That's why our app shows up when you share a photo, but not when you share, say, a contact card.
- **Android** uses MIME types instead of an extension. `singleShareMimeTypes` / `multipleShareMimeTypes` decide which intents our app advertises support for.

### `plugins/withShareExtensionDevClient.ts` - a little native patching

The stock share extension deep-links into the app using the _production_ URL scheme (`intermediateworkshop://expo-sharing`). During development you're running the **dev client**, which uses a different scheme (`exp+intermediateworkshop://...`). Our custom config plugin patches the generated Swift so the extension tries the dev-client scheme first, then the production one:

```swift
// Try dev client scheme first (exp+<scheme>), then production scheme
if let devUrl = URL(string: "exp+\(hostAppScheme)://expo-sharing") {
  openURL(devUrl)
}
if let prodUrl = URL(string: "\(hostAppScheme)://expo-sharing") {
  openURL(prodUrl)
}
```

It also includes a workaround for an iOS 26 beta crash in `SLComposeServiceViewController`. This is a good example of the config-plugin escape hatch: when a library's generated native code doesn't quite fit your setup, you can reach in and rewrite it at prebuild time instead of forking the library.

### `src/app/+native-intent.ts` - routing the share into a screen

When the extension fires `exp+intermediateworkshop://expo-sharing`, that URL lands in our app. `+native-intent.ts` intercepts it and rewrites it to a normal route:

```ts
export function redirectSystemPath({ path, initial }: { path: string; initial: boolean }) {
  try {
    const url = new URL(path);
    if (url.hostname === "expo-sharing") {
      return "/shared";
    }
    return path;
  } catch {
    return path;
  }
}
```

So the whole trip is: **share sheet → native extension → deep link → `+native-intent` → `/shared` route.** Everything from here on out is plain React we get to write ourselves.

🏃**Try it.** Before writing any screen code, open Photos on your simulator, pick an image, tap the share button, and find your app in the share sheet. It'll open the app to a blank `/shared` route for now. That's the native half working end to end.

## Exercise 1: The share route

The `/shared` route is a thin wrapper - it sets the header title and renders our screen. It also happens to be a great spot to configure the `Stack.Screen` options.

1. Create **src/app/shared.tsx**:

```tsx
import { Stack } from "expo-router";

import { SharedContentScreen } from "@/screens/SharedContentScreen";

export default function SharedRoute() {
  return (
    <>
      <Stack.Screen
        options={{
          title: "Add a Game",
          headerShown: true,
        }}
      />
      <SharedContentScreen />
    </>
  );
}
```

Now let's build `SharedContentScreen`.

## Exercise 2: Reading the shared image

The `useIncomingShare` hook does the heavy lifting of pulling the shared payload out of that shared container and resolving it into something we can render.

1. Create **app/screens/SharedContentScreen.tsx** with just the hook and a couple of loading/empty states to start:

```tsx
import { ActivityIndicator, type ViewStyle } from "react-native";
import { useIncomingShare } from "expo-sharing";
import { Column, Host, Text } from "@expo/ui";

import { Screen } from "@/components/Screen";
import { colors } from "@/theme/colorsDark";
import { spacing } from "@/theme/spacing";

export function SharedContentScreen() {
  const { resolvedSharedPayloads, isResolving, clearSharedPayloads } = useIncomingShare();

  // We only care about the first image that was shared in.
  const sharedImage = resolvedSharedPayloads.find((p) => p.contentType === "image");
  const imageUri = sharedImage?.contentUri;

  if (isResolving) {
    return (
      <Screen preset="fixed" contentContainerStyle={$center}>
        <ActivityIndicator color={colors.tint} />
        <Host matchContents>
          <Text textStyle={{ color: colors.textDim, textAlign: "center" }}>
            Loading shared image...
          </Text>
        </Host>
      </Screen>
    );
  }

  if (!imageUri) {
    return (
      <Screen preset="fixed" contentContainerStyle={$center}>
        <Host matchContents>
          <Column spacing={spacing.xs} alignment="center">
            <Text textStyle={{ fontSize: 20, fontWeight: "600" }}>Nothing shared yet</Text>
            <Text textStyle={{ color: colors.textDim, textAlign: "center" }}>
              Share an image from another app to add a new game.
            </Text>
          </Column>
        </Host>
      </Screen>
    );
  }

  return null; // the form comes next
}

const $center: ViewStyle = {
  flex: 1,
  justifyContent: "center",
  alignItems: "center",
  padding: spacing.lg,
  gap: spacing.xs,
};
```

A couple of things to notice:

- `resolvedSharedPayloads` is an array - a share can carry multiple items, and each has a `contentType` (`"image"`, `"text"`, `"website"`, etc.) and a `contentUri`. We just grab the first image.
- `isResolving` is `true` while the native side is copying the shared file into a place we can read. That's our loading state.

🏃**Try it.** Share an image into the app. You should briefly see the loading spinner, then... nothing (we return `null`). Open the app directly (not via share) and you'll get the "Nothing shared yet" empty state. Both states working? Great, let's show the image and build the form.

## Exercise 3: The "new game" form with Expo UI

Now the fun part. We've got an image; let's show it as a cover and collect the details for a new game with the same universal controls we've been using all workshop. And we get to reuse the `Dropdown` we built back in Module 01!

1. Beef up the imports:

```tsx
import { useMemo, useState } from "react";
import { ActivityIndicator, Alert, View, type ImageStyle, type ViewStyle } from "react-native";
import { Image } from "expo-image";
import { router } from "expo-router";
import { useIncomingShare } from "expo-sharing";
import { Button, Column, FieldGroup, Host, Text, TextInput } from "@expo/ui";

import { Dropdown } from "@/components/Dropdown";
import { Screen } from "@/components/Screen";
import { useFeedGenres } from "@/services/api/games";
import { colors } from "@/theme/colorsDark";
import { spacing } from "@/theme/spacing";
```

2. Add the form state and a little validation up near the top of the component (after the `imageUri` line):

```tsx
const { data: genres = [] } = useFeedGenres();
const genreItems = useMemo(
  () => genres.map((g) => ({ label: g.name, value: String(g.id) })),
  [genres],
);

const [title, setTitle] = useState("");
const [year, setYear] = useState("");
const [genreId, setGenreId] = useState("");

const canSubmit = title.trim().length > 0 && year.length === 4 && genreId.length > 0;
```

3. Add the dismiss and submit handlers. `clearSharedPayloads()` is important - it tells the native side we're done with this share, so the next one comes in clean:

```tsx
function dismiss() {
  clearSharedPayloads();
  router.back();
}

function handleAddGame() {
  Alert.alert("Game added!", `${title.trim()} (${year}) is in your collection.`, [
    { onPress: () => dismiss() },
  ]);
}
```

4. Now replace that `return null` with the real screen. Cover image up top (a plain React Native `View` + `expo-image`), then the Expo UI form inside a `Host`:

```tsx
return (
  <Screen preset="fixed" contentContainerStyle={$container}>
    <View style={$cover}>
      <Image source={{ uri: imageUri }} style={$coverImage} contentFit="cover" />
    </View>

    <Host style={$form}>
      <FieldGroup>
        <FieldGroup.Section title="New Game">
          <TextInput
            placeholder="Game title"
            defaultValue={title}
            onChangeText={setTitle}
            autoCapitalize="words"
          />
          <TextInput
            placeholder="Release year"
            defaultValue={year}
            onChangeText={(v) => setYear(v.replace(/[^0-9]/g, "").slice(0, 4))}
            keyboardType="number-pad"
            maxLength={4}
          />
          <Dropdown
            title="Genre"
            selectedValue={genreId}
            onValueChange={setGenreId}
            items={genreItems}
          />
        </FieldGroup.Section>

        <Host matchContents>
          <Button label="Add to Collection" onPress={handleAddGame} disabled={!canSubmit} />
        </Host>
      </FieldGroup>
    </Host>
  </Screen>
);
```

5. And the styles to go with it:

```tsx
const $container: ViewStyle = {
  flex: 1,
};

const $form: ViewStyle = {
  flex: 1,
};

const $cover: ViewStyle = {
  height: 200,
  overflow: "hidden",
  backgroundColor: colors.border,
};

const $coverImage: ImageStyle = {
  width: "100%",
  height: "100%",
};
```

Some things to appreciate here:

- The `Dropdown` is the _exact_ component we built in Module 01 - platform-specific SwiftUI/Jetpack Compose under the hood, but we import it like any other control. That's the payoff of the platform-file pattern: build it once, reuse it everywhere.
- `TextInput`, `FieldGroup`, and `Button` are all universal controls, so this form looks native on both platforms with zero extra work.
- The `disabled={!canSubmit}` gives us native-feeling validation - the button greys out until the form's happy.

🏃**Try it.** Share an image, fill in a title, a 4-digit year, and pick a genre. The "Add to Collection" button lights up when everything's valid, and tapping it pops an alert and sends you back. The whole share-into-add-a-game loop works!

## Exercise 4: A native close button with `Stack.Toolbar`

Right now the only way out is the system back gesture. Let's add a proper close (X) button to the header using `Stack.Toolbar`, just like we used in Module 04. We already have a `close` icon defined in the `useToolbarIcons` helper (`xmark` on iOS, `close` on Android), so this is quick.

1. Add the imports - grab `Stack` from `expo-router` and the toolbar icon helper:

```diff
- import { router } from "expo-router"
+ import { router, Stack } from "expo-router"
  import { useIncomingShare } from "expo-sharing"
  import { Button, Column, FieldGroup, Host, Text, TextInput } from "@expo/ui"

  import { Dropdown } from "@/components/Dropdown"
  import { Screen } from "@/components/Screen"
  import { useFeedGenres } from "@/services/api/games"
  import { colors } from "@/theme/colorsDark"
  import { spacing } from "@/theme/spacing"
+ import { useToolbarIcons } from "@/utils/useToolbarIcons"
```

2. Call the icon resolver near your other hooks (remember: hooks go before any of those early `return`s!):

```tsx
const toolbarIcon = useToolbarIcons(colors.brandSurfaceText);
```

3. Drop the toolbar in at the top of the form's returned JSX, just inside `<Screen>`:

```tsx
<Stack.Toolbar placement="right">
  <Stack.Toolbar.Button icon={toolbarIcon("close")} accessibilityLabel="Close" onPress={dismiss} />
</Stack.Toolbar>
```

`placement="right"` puts it on the trailing edge of the header. Because it calls our existing `dismiss()`, it clears the shared payload _and_ navigates back, same as everything else.

🏃**Try it.** Share an image, and you should see a native X button top-right. Tap it and you're back where you started, share cleared. Nice and tidy.

## Side Quests

- **Handle more than images.** The `activationRule` also accepts text and web URLs. Try reading a shared `website` payload and pre-filling the title from the page, or handle a shared block of text.
- **Multi-image support.** iOS is configured for up to 3 images. Show a little strip of thumbnails and let the user pick which one becomes the cover.
- **`expo-paste-input`.** Another neat OS-integration control worth exploring - a text input that offers a native "paste" affordance for images and rich content.

## See the solution

Switch to branch: [`05-cool-stuff`](https://github.com/infinitered/cr-2026-intermediate-workshop-template/tree/05-cool-stuff)
