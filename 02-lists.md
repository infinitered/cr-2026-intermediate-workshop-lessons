# Module 02: Expo UI Seasoning - making lists native

### Goal

Expo UI lets you tap into all those micro-interactions that come with platform-native controls (especially when it comes to SwiftUI). Let's overhaul some screens with lists to show how to balance your unique design with the introduction of platform-native components that espouse the design sensibilities of the OS itself.

### Concepts

- Using Expo UI list controls
- Using the standard list row components
- Fully-customizing components in Expo UI

### New components

- Jetpack Compose:
  - [LazyColumn](https://docs.expo.dev/versions/latest/sdk/ui/jetpack-compose/lazycolumn/)
  - [ElevatedCard](https://docs.expo.dev/versions/latest/sdk/ui/jetpack-compose/card/)
  - [Row](https://docs.expo.dev/versions/latest/sdk/ui/jetpack-compose/row/)
  - [IconButton](https://docs.expo.dev/versions/latest/sdk/ui/jetpack-compose/iconbutton/)
- Swift UI:
  - [List](https://docs.expo.dev/versions/latest/sdk/ui/swift-ui/list/)
  - [Section](https://docs.expo.dev/versions/latest/sdk/ui/swift-ui/section/)
  - [LabeledContent](https://docs.expo.dev/versions/latest/sdk/ui/swift-ui/label/)
  - [VStack](https://docs.expo.dev/versions/latest/sdk/ui/swift-ui/vstack/) and [HStack](https://docs.expo.dev/versions/latest/sdk/ui/swift-ui/hstack/)

### Features to build

- Update Settings screen controls
- Update queue and filter lists to use native list controls with sorting and deletion gestures

### Resources
- [Expo UI Jetpack Compose modifiers](https://docs.expo.dev/versions/latest/sdk/ui/jetpack-compose/modifiers/)
- [Expo UI SwiftUI modifiers](https://docs.expo.dev/versions/latest/sdk/ui/swift-ui/modifiers/)

# Exercises

## Exercise 1: A simple list: the favorite genres screen (iOS)

iOS especially has strong opinions about lists and especially actions you can take on list items, like deleting or sorting them. Let's use this to make a favorite genres selector screen that has a much more native user experience.

1. Create the **app/screens/FavoriteGenresScreen.ios.tsx** file. Use this as the starting template (this is just the imports and the hooks that we need to keep around), along with the `Host` wrapper required for Expo UI components:

```tsx
import { LoadingScreen } from "@/components/LoadingScreen";

import type { FeedGenre } from "@/services/api/games";
import { useFavoriteGenresService } from "@/services/favoriteGenresService";

import {
  Host,
  List,
  Section,
  Label,
  Image,
  LabeledContent,
} from "@expo/ui/swift-ui";
import {
  environment,
  tag,
  onTapGesture,
  moveDisabled,
} from "@expo/ui/swift-ui/modifiers";
import { useState } from "react";

export function FavoriteGenresScreen() {
  const { favoriteGenres, otherGenres, isLoading, isFavorite, toggleFavorite } =
    useFavoriteGenresService();

  if (isLoading) {
    return <LoadingScreen />;
  }

  return <Host style={{ flex: 1 }}></Host>;
}
```

2. Something cool about Swift UI lists is that everything inside of `List` is inside of the list, but different elements can have different interactions. Since our list has separate sections of favorites and available genres, we're going to make those separate sections. Add this inside `Host`:

```tsx
<List modifiers={[environment("editMode", "active")]}>
  <Section title="Your Favorites">
    <List.ForEach>
      {favoriteGenres.map((genre) => (
        <Label key={genre.id} title={genre.name} modifiers={[tag(genre.id)]} />
      ))}
    </List.ForEach>
  </Section>
  <Section title="Available Genres">
    {otherGenres.map((genre) => (
      <LabeledContent
        key={genre.id}
        label={genre.name}
        modifiers={[tag(genre.id)]}
      >
        <Image
          systemName="plus.circle"
          size={22}
        />
      </LabeledContent>
    ))}
  </Section>
</List>
```

3. It should start to look like how we want it to look. Let's add some interactions, namely adding and removing favorites:

```diff
+<List.ForEach onDelete={(indices) => { indices.forEach(index => toggleFavorite(favoriteGenres[index].id)); }}>
-<List.ForEach>
  {favoriteGenres.map((genre) => (
    <Label
      key={genre.id}
      title={genre.name}
      modifiers={[tag(genre.id)]}
    />
  ))}
</List.ForEach>
</Section>
<Section title="Available Genres">
  {otherGenres.map((genre) => (
    <LabeledContent
      key={genre.id}
      label={genre.name}
+      modifiers={[tag(genre.id), onTapGesture(() => toggleFavorite(genre.id))]}
-      modifiers={[tag(genre.id)]}
    >
      <Image systemName="plus.circle" size={22} />
    </LabeledContent>
  ))}
</Section>
```

🏃**Try it.** Adding and removing genres should work! Removing might be a little confusing since you have to swipe and there's no confirmation.

Let's improve the delete interaction by making it a little more obvious. Swift UI lists support different modes, including an edit mode which brings deletion and other interactions front-and-center. You might be familiar with being in a list in iOS, choosing to "edit", and then controls for deletion, sort, and multi-select appear.

4. Update your code to support edit mode:

```diff
<Host style={$host}>
- <List>
+  <List modifiers={[environment("editMode", "active")]}>
    <Section title="Your Favorites">
```

5. Now deletion has the icon you can tap on the left as well as the confirmation step. But it also has sort handles, which we don't need here. Let's remove that with a modifier:

```diff
<List.ForEach onDelete={handleDelete}>
  {favoriteGenres.map((genre) => (
    <Label
      key={genre.id}
      title={genre.name}
-      modifiers={[tag(genre.id)]}
+.     modifiers={[tag(genre.id), moveDisabled(true)]}
    />
  ))}
</List.ForEach>
```

That should take care of the sort handles.

6. Just for fun, let's add multi-select, so you can see all that edit mode can do:

```diff
- <List modifiers={[environment("editMode", "active")]}>
+ <List modifiers={[environment("editMode", "active")]} selection={[]}>
```

If you were actually adding selection, you would pass in a list of indices and also implement event handlers, but just setting the prop is enough to turn on the UI element.

7. Remove that `selection` prop. We don't need it!

8. For finishing touches, you can add an empty state for the favorites section:

```diff
<Section title="Your Favorites">
+  {favoriteGenres.length === 0 ? (
+    <Label
+      title="Add genres below to personalize recommendations"
+      systemImage="info.circle"
+   />
+  ) : (
    <List.ForEach onDelete={handleDelete}>
      {favoriteGenres.map((genre) => (
        <Label
          key={genre.id}
          title={genre.name}
          modifiers={[tag(genre.id), moveDisabled(true)]}
        />
      ))}
    </List.ForEach>
+  )}

```

🏃**Try it.** I think we've covered our bases!

### What about Android?

Jetpack Compose doesn't have as much built-in to offer here other than syncing up some icons and container styles with Material Design. If you have time for this, check out the Side Quests section below.

## Exercise 2: Lists with custom elements: the queue screen

Once again, there's more fun to be had here with iOS, but we'll be styling both versions of the Queue screen this time.

### iOS

1. Create the **app/screens/QueueScreen.ios.tsx** file. Use this as the starting template (this is just the imports and the hooks that we need to keep around), along with the `Host` wrapper required for Expo UI components:

```tsx
import { LoadingScreen } from "@/components/LoadingScreen"
import { Image as ExpoImage } from "expo-image"
import { router, Stack } from "expo-router"
import {
  Host,
  List,
  Section,
  Button,
  HStack,
  VStack,
  Text as SwiftText,
  RNHostView,
} from "@expo/ui/swift-ui"
import {
  environment,
  tag,
  onTapGesture,
  badge,
  foregroundStyle,
  font,
} from "@expo/ui/swift-ui/modifiers"
import { useState } from "react";
import { useQueueService } from "@/services/queueService"

export function QueueScreen() {
  const { queuedGames, availableGames, isLoading, chooseNextGame, removeFromQueue, moveInQueue } = useQueueService()

    if (isLoading) {
      return <LoadingScreen />
    }

  return (
    <Host style={{ flex: 1}}></Host>
  )
}
```

2. Let's start out with a basic list along with the button to choose the next game. We're going to back away from some of our custom styling for a moment so we can get the basic interactions right, just adding the needed text and structure. Add this inside of `Host`:

```tsx
<List>
  <List.ForEach onDelete={handleDelete} onMove={handleMove}>
    {queuedGames.map((game, index) => (
      <HStack
        key={game.id}
        spacing={12}
        alignment="center"
        >
          <SwiftText
            modifiers={[
              foregroundStyle({ type: "hierarchical", style: "secondary" }),
              font({ weight: "bold", size: 12 }),
            ]}
          >
            {String(index + 1)}
          </SwiftText>
          <VStack>
          <VStack alignment="leading" spacing={2}>
            <SwiftText modifiers={[font({ weight: "semibold" })]}>{game.name}</SwiftText>
            {game.genres && game.genres.length > 0 ? (
              <SwiftText
                modifiers={[
                  font({ size: 12 }),
                  foregroundStyle({ type: "hierarchical", style: "secondary" }),
                ]}
              >
                {game.genres.map((g) => g.name).join(", ")}
              </SwiftText>
            ) : null}
          </VStack>
        </VStack>
      </HStack>
    ))}
  </List.ForEach>
</List>
```

This might look like a lot to unpack, but it's pretty straightforward. `HStack` lays out children horizontally, `VStack` lays them out vertically. Combined with text components, we used this to put the queue position on the left and the title and genres stacked on top of each other to the right.

🏃**Try it.** The text of each row should look about right. Try to hold and drag, and swipe and then delete. These should work, too.

3. The built-in sorting and deleting features are fine, but it can be nice to have a dedicated edit mode to make this functionality more front-and-center. Let's try turning on the SwiftUI list edit mode on to see how it works.

```diff
-<List>
+<List modifiers={[environment("editMode", "active")]}>
```

🏃**Try it.** Nice editing controls! Love those drag handles!

4. OK, but we don't want edit mode on all the time. Let's make it conditional. Let's add state to track edit mode:

```tsx
const [editMode, setEditMode] = useState(false)

const handleToggleEdit = () => {
  setEditMode((prev) => {
    return !prev
  })
}
```

Then update the List modifier to use the state:

```diff
-<List modifiers={[environment("editMode", "active")]}>
+<List modifiers={[environment("editMode", editMode ? "active" : "inactive")]}>
```

5. But we need some button to change the state. There might be better options later, but for now we'll put a button in the navigation header. [This layout in the Apple SwiftUI documentation](https://developer.apple.com/documentation/swiftui/editbutton) would be nice but we're not quite there yet with the rest of the UI, so let's do something close to that by putting an edit button in the top right just inside the `<List />` component:

```diff
<Section
+  header={
+    <HStack>
+      <Spacer />
+      <Button
+        label={editMode ? "Done" : "Edit"}
+        onPress={() => setEditMode((prev) => !prev)}
+      />
+    </HStack>
  }
```

6. Now that we have edit mode wired up, let's put back the button for randomly choosing the next game. It'll make it easier for us to test things. Add this just inside the `List`, below the `Section`:

```tsx
{!editMode ? (
  <Section>
    <Button
      label={availableGames.length > 0 ? "Choose My Next Game" : "All Games Queued!"}
      onPress={chooseNextGame}
    />
  </Section>
) : null}
```

7. Now we'll bring back the game thumbnail. We want to use `expo-image`, but that's not an Expo UI component. So, we'll use an `RNHostView`, which allows us to embed React Native components within Expo UI components. Add this below the `SwiftText` component for displaying the number for the position in the queue:

```tsx
{game.background_image ? (
  <RNHostView matchContents>
    <ExpoImage
      source={game.background_image}
      style={{
        width: 40,
        height: 40,
        borderRadius: 6,
      }}
    />
  </RNHostView>
) : null
}
```

🏃**Try it.** Should have all the elements it had before.

#### Finishing touches

Our queue should show the games that will be shipped to us next. Let's use modifiers to add some context there.

1. Define a `SHIPMENT_SIZE` constant at the top of the file:

```tsx
const SHIPMENT_SIZE=3
```

2. Add the tag modifier to the `HStack`:

```diff
<HStack
  key={game.id}
  spacing={12}
  alignment="center"
+  modifiers={[
+    tag(game.id),
+    ...(index < SHIPMENT_SIZE ? [badge("Next")] : []),
+  ]}
>
```

3. While we're in there, you should be able to view the game from this screen. Add a tap gesture modifier:

```diff
<HStack
  key={game.id}
  spacing={12}
  alignment="center"
  modifiers={[
    tag(game.id),
    ...(index < SHIPMENT_SIZE ? [badge("Next")] : []),
+    onTapGesture(() => {
+      if (!editMode) router.push(`/game/${game.id}`)
+    }),
  ]}
>
```

4. Also add a footer to tell users what the "Next" badge means. Add this property to the top `Section`:

```tsx
footer={
  <SwiftText>The first {SHIPMENT_SIZE} games will be in your next delivery!</SwiftText>
}
```

🏃**Try it.** The Next tag should show on the intended list items as you move things around, and how you can view the full game page from the queue, as well.

### Android

Since Android lists don't have the same swipe to delete and sorting functionality, we will focus on a cosmetic upgrade, using standard Jetpack Compose and Material icons.

1. Create the **app/screens/QueueScreen.android.tsx** file. Use this as the starting template (this is just the imports and the hooks that we need to keep around), along with the `Host` wrapper required for Expo UI components:

```tsx
import { Image as ExpoImage } from "expo-image"
import { router } from "expo-router"
import ArrowDownward from "@expo/material-symbols/arrow_downward.xml"
import ArrowUpward from "@expo/material-symbols/arrow_upward.xml"
import Cancel from "@expo/material-symbols/cancel.xml"
import {
  Host,
  LazyColumn,
  ElevatedCard,
  Button,
  Row,
  HorizontalDivider,
  Icon,
  IconButton,
  Text as ComposeText,
  RNHostView,
  useMaterialColors,
} from "@expo/ui/jetpack-compose"
import { clickable, fillMaxWidth, padding, size, weight } from "@expo/ui/jetpack-compose/modifiers"

import { useQueueService } from "@/services/queueService"
import { LoadingScreen } from "@/components/LoadingScreen"
import type { Game } from "@/services/api/types"
import { colors } from "@/theme/colors"

export function QueueScreen() {
  const { queuedGames, availableGames, isLoading, chooseNextGame } = useQueueService()

    if (isLoading) {
      return <LoadingScreen />
    }

  return (
    <Host style={{ flex: 1}}></Host>
  )
}

```

2. You might not expect it, but the name of the Android list component is `LazyColumn`. Let's scaffold out a basic list. Add this inside host:

```tsx
<LazyColumn contentPadding={{ start: 16, end: 16, bottom: 24 }}>
  {queuedGames.map((game, index) => (
    <QueueCard key={game.id} game={game} index={index} total={queuedGames.length} />
  ))}

  <HorizontalDivider modifiers={[padding(0, 16, 0, 8)]} />

  <Button onClick={chooseNextGame} modifiers={[fillMaxWidth()]}>
    <ComposeText>
      {availableGames.length > 0 ? "Choose My Next Game" : "All Games Queued!"}
    </ComposeText>
  </Button>
</LazyColumn>
```

Below the screen component, create the `QueueCard`:

```tsx
function QueueCard({ game, index, total }: { game: Game; index: number; total: number }) {
  const isFirst = index === 0
  const isLast = index === total - 1
  const { removeFromQueue, moveInQueue } = useQueueService()

  return (
    <ElevatedCard
      modifiers={[padding(0, 4, 0, 4)]}
    >
      <Row
        verticalAlignment="center"
        horizontalArrangement={{ spacedBy: 12 }}
        modifiers={[fillMaxWidth(), padding(16, 12, 16, 12)]}
      >
        <ComposeText style={{ typography: "titleSmall" }}>{String(index + 1)}</ComposeText>
        <ComposeText style={{ typography: "bodyMedium" }} modifiers={[weight(1)]}>
          {game.name}
        </ComposeText>
      </Row>
    </ElevatedCard>
  )
}
```

That should get us a basic layout. Now we'll add more visuals and functionality back.

3. Just like with the iOS version, let's use an `RNHostView` to include the game image. Add this directly after the sequence number:

```tsx
{game.background_image ? (
  <RNHostView matchContents>
    <ExpoImage
      source={game.background_image}
      style={{
        width: 40,
        height: 40,
        borderRadius: 6,
      }}
    />
  </RNHostView>
) : null}
```

4. Let's make the whole thing pressable, showing you the game details on tap:

```diff
<ElevatedCard
-  modifiers={[padding(0, 4, 0, 4)]}
+  modifiers={[padding(0, 4, 0, 4), clickable(() => router.push(`/game/${game.id}`))]}
>
```

5. Add some icons for the sorting and delete operations after the game title:

```tsx
<IconButton
  onClick={() => moveInQueue(game.id, "up")}
  enabled={!isFirst}
  modifiers={[size(28, 28)]}
>
  <Icon source={ArrowUpward} size={20} />
</IconButton>
<IconButton
  onClick={() => moveInQueue(game.id, "down")}
  enabled={!isLast}
  modifiers={[size(28, 28)]}
>
  <Icon source={ArrowDownward} size={20} />
</IconButton>
<IconButton onClick={() => removeFromQueue(game.id)} modifiers={[size(28, 28)]}>
  <Icon source={Cancel} size={20} />
</IconButton>
```

🏃**Try it.** It should look a lot like the old screen, but with an Android platform-native twist, and should work the same as well.

#### What about the colors?

The colors of the components might not make a lot of sense yet. They're actually based on the theme set on the operating system in Android 12+.

1. Let's try changing the theme to see what happens. Go to Settings -> Wallpaper & Style, and then change the theme colors. See how it affects the colors on this screen.

2. We can let Jetpack Compose set the colors automatically for all components based on a seed color. There's probably no perfect option based on our current theme, but let's try one:

```diff
<Host
  style={{ flex: 1 }}
+  seedColor={colors.brandSurface}
>
```

3. The delete button should probably be red...right? Let's use the standard material error red for this.

Add the `useMaterialColors` hook:

```tsx
const materialColors = useMaterialColors()
```

Set the theme for the color of the icon button:

```diff
<IconButton onClick={() => removeFromQueue(game.id)} modifiers={[size(28, 28)]}>
-  <Icon source={Cancel} size={20} />
+  <Icon source={Cancel} size={20} tint={materialColors.error }/>
</IconButton>
```

🏃**Try it.** The seed color probably isn't ideal, and you might wonder if we'd be better off embracing the user's chosen theme colors. We'll come back to this later!

## Side Quests

## Style sync-up for Android on the favorite genres screen

Let's do something to make that Favorite Genres screen a little more Jetpack-composey, even though they'res not much special we can do there. We'll use `ListItem` to experiment with the standardized controls for making items in lists, typical in things like settings menus.

1. Create a new **FavoriteGenres.android.tsx** next to the others. Fill it with some boilerplate:

```tsx
import { ActivityIndicator } from "react-native"
import {
  Host,
  LazyColumn,
  ListItem,
  HorizontalDivider,
  Text as ComposeText,
  Icon,
  IconButton,
} from "@expo/ui/jetpack-compose"
import { clickable, padding } from "@expo/ui/jetpack-compose/modifiers"
import AddCircle from "@expo/material-symbols/add_circle.xml"
import Remove from "@expo/material-symbols/remove.xml"

import { useFeedGenres, type FeedGenre } from "@/services/api/games"
import { useFavoriteGenres } from "@/stores/favoriteGenres"
import { Screen } from "@/components/Screen"
import { colors } from "@/theme/colors"

export function FavoriteGenresScreen() {
  const { data: allGenres = [], isLoading } = useFeedGenres()
  const { ids: favoriteIds, addFavoriteGenre, removeFavoriteGenre } = useFavoriteGenres()

  if (isLoading) {
    return (
      <Screen preset="fixed" contentContainerStyle={{ flex: 1 }}>
        <ActivityIndicator size="large" color={theme.colors.tint} />
      </Screen>
    )
  }

  const favoriteGenres = favoriteIds
    .map((id) => allGenres.find((g) => g.id === id))
    .filter(Boolean) as FeedGenre[]

  const otherGenres = allGenres.filter((g) => !favoriteIds.includes(g.id))

  return (
    <Host style={{ flex: 1 }}>

    </Host>
  )
}
```

2. Add a `LazyColumn` inside the `Host` and iterate over favorites and non-favorite genres:

```tsx
<LazyColumn>
{favoriteGenres.map((genre) => (
    <ListItem key={genre.id} colors={{ containerColor: "transparent" }}>
      <ListItem.HeadlineContent>
        <ComposeText>{genre.name}</ComposeText>
      </ListItem.HeadlineContent>
      <ListItem.SupportingContent>
        <ComposeText>
          {genre.gameCount} {genre.gameCount === 1 ? "game" : "games"}
        </ComposeText>
      </ListItem.SupportingContent>
      <ListItem.TrailingContent>
        <IconButton onClick={() => removeFavoriteGenre(genre.id)}>
          <Icon
            source={Remove}
            tint={colors.error}
            size={24}
          />
        </IconButton>
      </ListItem.TrailingContent>
    </ListItem>
  ))}
  {otherGenres.map((genre) => (
    <ListItem
      key={genre.id}
      colors={{ containerColor: "transparent" }}
      modifiers={[clickable(() => addFavoriteGenre(genre.id))]}
    >
      <ListItem.HeadlineContent>
        <ComposeText>{genre.name}</ComposeText>
      </ListItem.HeadlineContent>
      <ListItem.SupportingContent>
        <ComposeText>
          {genre.gameCount} {genre.gameCount === 1 ? "game" : "games"}
        </ComposeText>
      </ListItem.SupportingContent>
      <ListItem.TrailingContent>
        <IconButton onClick={() => addFavoriteGenre(genre.id)}>
          <Icon
            source={AddCircle}
            tint={colors.brandAccent}
            size={24}
          />
        </IconButton>
      </ListItem.TrailingContent>
    </ListItem>
  ))}
</LazyColumn>
```

In this case, we're using custom Android XML drawables for the add/remove buttons. The `Icon` component can import them directly; they don't need to be embedded in the native project setup or anything like that.

3. Let's add some titles before each section:

```tsx
 <ComposeText style={{ typography: "titleMedium" }} modifiers={[padding(0, 16, 0, 8)]}>
  Your Favorites
</ComposeText>
```

```tsx
<ComposeText style={{ typography: "titleMedium" }} modifiers={[padding(0, 8, 0, 8)]}>
  Available Genres
</ComposeText>
```

4. And a divider between the two sections:

```tsx
<HorizontalDivider modifiers={[padding(0, 8, 0, 8)]} />
```

5. Then fix the padding on the list iself so the headers aren't squished up against the edge:

```diff
-<LazyColumn contentPadding={{ start: 16, end: 16, bottom: 24 }}>
+<LazyColumn contentPadding={{ start: 16, end: 16, bottom: 24 }}>
```

🏃**Try it.** What other ideas might you have for this screen? Open to suggestions!

## See the solution

Switch to branch: [`02-lists`](https://github.com/infinitered/cr-2026-intermediate-workshop-template/tree/02-lists)
