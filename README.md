# Intermediate Workshop - Chain React 2026

Workshop exercises for the Chain React 2026 Intermediate Workshop.

## Slides

## Contact
- [Find me on Expo Discord](https://chat.expo.dev) - I'm "keith".
- [Keith on X](https://twitter.com/llamaluvr)
- [https://www.linkedin.com/in/keith-kurak/](LinkedIn)
- [https://bsky.app/profile/keith.bsky.social](Bluesky)

Probably best place is LinkedIn or email ... keith [at] expo [dot] dev.

## How to use this repo

1. Fork the [companion repo](https://github.com/infinitered/cr-2026-intermediate-workshop-template). You'll start working right on `main`.
2. Start at the first module by opening up the file starting with "01-".
3. Do the rest of the modules in order.

## Prerequisites

- A local development environment ready for native iOS and Android React Native / Expo development, capable of running the `npx expo run:ios` and `npx expo run:android`, including recent versions of:
  - Xcode (version 26.4+)
  - Watchman
  - Cocoapods
  - JDK 17
  - Android Studio
  - iOS simulator
  - Android emulator
  - If you're not sure if you have all of these or if you have the right versions, check the [Expo Local App Development requirements](https://docs.expo.dev/guides/local-app-development/) for details on how to install these tools in order to enable local native development with the Expo CLI.
- Other general development tools:
  - [Node LTS version (22+)](https://pnpm.io/installation)
  - Visual Studio Code
  - Git (Github Desktop works great)
- Hardware:
  - A Mac is highly recommended for the full experience. You will not be able to do some of the exercises without access to an iOS device or simulator.
  - Simulators/emulators will work just fine for all the exercises, but bringing an Android or iOS device to test on is great, too!
- [pnpm](https://pnpm.io/installation)

## Test your setup before the workshop

Do these steps to ensure you'll be able to complete the workshop exercises.

1. Fork and clone the [template repo](https://github.com/infinitered/cr-2026-intermediate-workshop-template).

2. Restore dependencies with

```
pnpm install
```

3. Build and run the app on your iOS simulator:

```
npx expo run:ios
```

4. Build and run the app on your Android emulator:

```
npx expo run:android
```

If these steps don't work, refer to the [Expo Local Development guide](https://docs.expo.dev/guides/local-app-development/).

**Want to run on a device?** If you want to do some or all of the workshop on a device, you can also test with `npx expo run:ios --device` and/or `npx expo run:android --device`.
