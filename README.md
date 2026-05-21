# Minimal reproducer: `ios.icon` (Liquid Glass) build crash on Xcode 26.5

This is a minimal Expo project that demonstrates the `actool` crash on Xcode **26.5 (17F42)** when `ios.icon` points at any `.icon` directory (Apple's Liquid Glass icon format).

**Filed with Apple as FB20183399.**

## What this repo is

- `npx create-expo-app@latest --template blank` (unmodified) plus:
- `ios.icon: "./assets/app.icon"` in `app.json`
- `assets/app.icon/` — a single-layer Liquid Glass icon (a football on a gradient background) representative of Icon Composer 1.5 output.

The crash is content-independent: verified against 13 variants including a bare `{ }` icon.json with no `Assets/` directory. Drop in any other `.icon` from Icon Composer and the crash is identical.

## How to reproduce

```sh
npm install
npx expo prebuild --clean -p ios
xcodebuild -workspace ios/expoactoolliquidglassbug.xcworkspace \
  -scheme expoactoolliquidglassbug \
  -configuration Debug \
  -sdk iphonesimulator \
  -destination 'generic/platform=iOS Simulator' \
  CODE_SIGNING_ALLOWED=NO build
```

The build fails in the `CompileAssetCatalog` phase.

## Standalone reproducer (no Expo needed)

The bug lives in Apple's `actool`, not in Expo. To prove that, run:

```sh
mkdir /tmp/foo.icon
echo '{}' > /tmp/foo.icon/icon.json
mkdir -p /tmp/out
xcrun actool /tmp/foo.icon \
  --compile /tmp/out \
  --platform iphoneos \
  --app-icon foo \
  --output-partial-info-plist /tmp/p.plist \
  --output-format human-readable-text
```

Expected output on Xcode 26.5 (17F42):

```
error: Exception while running actool: *** -[__NSPlaceholderArray initWithObjects:count:]: attempt to insert nil object from objects[0]
Backtrace:
  0   __exceptionPreprocess (in CoreFoundation)
  1   objc_exception_throw (in libobjc.A.dylib)
  2   -[__NSPlaceholderArray initWithObjects:count:] (in CoreFoundation)
  3   +[NSArray arrayWithObjects:count:] (in CoreFoundation)
  4   -[IBICAbstractPlatformAdapter selectCatalogIconComposerItemsFromCollection:ignoringItems:forLocation:withOptions:populatingIssues:] (in AssetCatalogFoundation)
  5   -[IBICAbstractPlatformAdapter selectCatalogItemsForCARCompilerFromCollection:ignoringItems:forCompilingWithOptions:populatingIssues:] (in AssetCatalogFoundation)
  6   -[IBICAbstractPlatformAdapter selectCatalogItemsForCompilingCollection:options:] (in AssetCatalogFoundation)
  7   -[IBICAbstractPlatformAdapter compileCatalogCollection:options:queue:completionHandler:] (in AssetCatalogFoundation)
  ...
```

## Bisect

| Xcode | Build | actool result |
| --- | --- | --- |
| 26.4.1 | 17E202 | ✅ Processes `.icon` cleanly (no nil-objects crash) |
| 26.5 | 17F42 | ❌ Crashes on any `.icon` input |

Regression was introduced between 17E202 and 17F42, inside `AssetCatalogFoundation` / `IBPlatformTool`.

## Content-independence test

Reproduced identically with 13 variants:

- bare `{}` icon.json with no `Assets/` directory (this repo)
- empty `groups: []`
- single layer, no `glass`
- group with no `shadow` / no `translucency`
- `display-p3` linear-gradient fill replaced with plain sRGB hex
- `watchOS` removed from `supported-platforms`
- `squares: ["iOS"]` (instead of `"shared"`)
- SVG filenames stripped of spaces
- `--platform macosx` instead of `iphoneos`
- Full fully-featured `.icon` from Icon Composer 1.5

All produce the identical `selectCatalogIconComposerItemsFromCollection` nil-objects backtrace. The bug fires before content parsing.

## Workaround

Side-install Xcode 26.4.1 (developer.apple.com/download/all), install the iOS 26.4 platform component, build with:

```sh
DEVELOPER_DIR=/Applications/Xcode-26.4.1.app/Contents/Developer \
  npx expo prebuild --clean -p ios

DEVELOPER_DIR=/Applications/Xcode-26.4.1.app/Contents/Developer \
  xcodebuild ...
```

SDK 26.4 builds remain App-Store-acceptable as of May 2026.

## Environment

- macOS: 26.5 (25F71)
- Xcode: 26.5 (17F42)
- Expo SDK: 55 (issue is the same on SDK 54)
- See `package.json` for exact versions.

## References

- Apple Feedback: **FB20183399**
- Apple Developer Forums (related thread): https://developer.apple.com/forums/thread/799820
