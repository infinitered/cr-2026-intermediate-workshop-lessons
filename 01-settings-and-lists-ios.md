# Module 01: Expo UI Seasoning -  opportunistic native components and lists (iOS edition)

### Goal

Let's use Expo UI in some ancillary screens, where we can basically adopt platform standard native component layouts without much customization.

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

## Exercise 0: Build the app (if you haven't already)
1. `yarn`
2. `npx expo run:ios` or `npx expo run:android`

## Exercise 1: Expo UI low-hanging fruit - settings forms!

The Settings screen is the perfect place to start laying in platform native UI components. They're typically no-frills forms where you don't need to worry too much about app-specific UX or branding. There's only upside here, and few, if any design decisions to make. Just swap out one set of components for another!

Let's branch out into a platform-specific page.

1. Create **screens/SettingsScreen.ios.ts**, and give it some basic scaffolding:

```ts
// TBD (just an empty page) + the same hook we used in the other form for editing data
```

> All Expo UI code goes in a `<Host>` component

🏃**Try it.** Your universal UI should now be gone on iOS, and you've got...nothing!

### Add those sections one-by-one

We're going to craft the classic iOS form with Swift UI, with the light gray background, the subheadings, the white controls inside the sections. Good stuff! It's a `Form`, plus `Section`'s, and then the actual controls in each section.

1. Add the form:

```diff
+ // TBD
```

Now let's add these sections. We'll try a variety of controls here, but they all work about the same- set a value and onChange handler, and you're good to go!

2. Add the profile section:

```tsx
<Section title="Profile">
  <TextField
    placeholder="Display Name"
    defaultValue={displayName}
    onValueChange={setDisplayName}
  />
  <DatePicker
    title="Birth Date"
    selection={dateValue}
    displayedComponents={["date"]}
    range={{ end: new Date() }}
    onDateChange={(date) => setBirthDate(date.toISOString())}
  />
</Section>
```

3. Now, shipping address:
```tsx
<Section title="Shipping Address">
  <TextField
    placeholder="Street Address"
    defaultValue={shippingAddress.street1}
    onValueChange={(v) => setShippingAddress({ street1: v })}
  />
  <TextField
    placeholder="Apt / Suite / Unit"
    defaultValue={shippingAddress.street2}
    onValueChange={(v) => setShippingAddress({ street2: v })}
  />
  <TextField
    placeholder="City"
    defaultValue={shippingAddress.city}
    onValueChange={(v) => setShippingAddress({ city: v })}
  />
  <Picker
    label="State"
    selection={shippingAddress.state || ""}
    onSelectionChange={(sel) => setShippingAddress({ state: String(sel) })}
    modifiers={[pickerStyle("menu")]}
  >
    {US_STATES.map((s) => (
      <SwiftText key={s.abbr} modifiers={[tag(s.abbr)]}>
        {s.name}
      </SwiftText>
    ))}
  </Picker>
  <TextField
    placeholder="ZIP Code"
    defaultValue={shippingAddress.zip}
    onValueChange={(v) => setShippingAddress({ zip: v })}
  />
</Section>
```

4. Add a toggle and picker for content preferences:

```tsx
<Section title="Content Preferences">
  <Toggle label="Hide Mature Content" isOn={hideMature} onIsOnChange={setHideMature} />
  <Picker
    label="Sort Order"
    selection={sortOrder}
    onSelectionChange={(sel) => setSortOrder(sel as (typeof SORT_OPTIONS)[number])}
    modifiers={[pickerStyle("menu")]}
  >
    {SORT_OPTIONS.map((option) => (
      <SwiftText key={option} modifiers={[tag(option)]}>
        {option}
      </SwiftText>
    ))}
  </Picker>
</Section>
```

5. Add a slider and tappable field for queue preferences (ooo, and a footer, too!):

```tsx
 <Section
  title="Queue Preferences"
  footer={<SwiftText>Only games rated {minRating}+ out of 5 will be suggested.</SwiftText>}
>
  <Slider
    value={minRating}
    min={1}
    max={5}
    step={1}
    label={<SwiftText>Minimum Rating: {minRating}</SwiftText>}
    minimumValueLabel={<SwiftText>1</SwiftText>}
    maximumValueLabel={<SwiftText>5</SwiftText>}
    onValueChange={setMinRating}
  />
  <SwiftText
    modifiers={[
      contentShape(shapes.rectangle()),
      onTapGesture(() => router.push("/favorite-genres")),
    ]}
  >
    Favorite Genres
  </SwiftText>
```

> What's missing? Oh yeah, styles! You really don't need any!

🏃**Try it.** It should be a form, that's it!

## Exercise 2: Lists 101

The next more straightforward conversion are lists. At least some lists. Configuration lists are straightforward in the same way forms are: less is more, you're better off using the platform-default UI than trying to heavily customize it.

We'll start with the Favorite Genres list. It's really two lists that we can add/remove items between, but doesn't need much in the way of customization other than that.

1. Create **app/screens/FavoriteGenresScreen.ios.tsx** and add scaffolding:

```tsx
// TBD
```

Let's lay out the visuals without adding any of the editing features.

One interesting bit. This will be a list with multiple sections, but only one section will actually have list items and thus be editable.

2. Add read-only-visuals

```tsx
// List
// ListItems (favorite genres)
// Section with labeled content (available genres)
```

> Check out the tags. This lets us take standardized elements like `Label`'s and add things in standard positions, like that plus button.

3. Let's enable adding since there's nothing really special about that, just a tap gesture

```tsx
// TBD
```

### Setting up sorting and deleting

4. Setup the edit mode environment. This will add the ability to sort and delete:

```tsx
// TBD
```

5. Add the onDelete handler:

```tsx
// TBD
```

6. Add the onMove handler:

```tsx
// TBD
```

🏃**Try it.** You should be able to add, remove, and sort. Try refreshing the app to make sure it maintains state.

## Exercise 3: Lists 201

Let's move over to the Queue list. This one is similar in that it supports sorting and deleting, but we will use `HStack` and `VStack` to customize the list items, as well as move the edit functionality behind a button.

1. Create and scaffold **app/screens/QueueScreen.ios.tsx**:

```tsx
// TBD
```

2. Setup a basic list with simple labels for now

```tsx
// TBD
```

3. Add the edit mode

```tsx
// TBD
```

4. Setup the custom list item UI

```tsx
// TBD
```

5. Highlight the next shipment

```tsx
// TBD
```

🏃**Try it.** Sorting and deleting and adding should all work great.

## Side Quests

- ???
- ???

## See the solution

Switch to branch: [`01-blending-in-solution`](https://github.com/infinitered/cr-2024-intermediate-workshop-template/tree/01-blending-in-solution)
