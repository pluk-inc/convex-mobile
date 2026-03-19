# Convex Mobile SDK

Rust-based Convex client compiled to native libraries for iOS, macOS, and Android.

## Project Structure

- `rust/` — Rust library (core client logic)
- `ios/` — Git submodule pointing to `pluk-inc/convex-swift` (Swift package)
- `android/` — Android/Kotlin wrapper

## Releasing an iOS build

The xcframework is distributed via GitHub Releases on `pluk-inc/convex-swift`, not committed to git.

### Prerequisites

- Rust toolchain with these targets installed:
  ```
  rustup target add aarch64-apple-ios aarch64-apple-ios-sim aarch64-apple-darwin x86_64-apple-ios x86_64-apple-darwin
  ```
- Xcode command line tools
- `gh` CLI (authenticated)
- `jq`

### Steps

1. **Update the version** in `rust/Cargo.toml` if needed.

2. **Build the xcframework with `--release`** from the `rust/` directory:
   ```
   cd rust
   ./build-ios.sh --release
   ```
   This compiles 5 targets (arm64 + x86_64 for simulator/macOS, arm64 for iOS device), creates universal fat binaries, builds the xcframework, zips it, and updates `ios/Package.swift` with the new version and checksum.

3. **Commit and push the `ios` submodule** (`pluk-inc/convex-swift`):
   ```
   cd ../ios
   git add Package.swift Sources/
   git commit -m "release: <version>"
   git push
   ```

4. **Create the GitHub Release** with the xcframework zip attached:
   ```
   gh release create <version> \
     ../rust/target/ios/libconvexmobile-rs.xcframework.zip \
     --repo pluk-inc/convex-swift \
     --title "<version>" \
     --notes "Release notes here"
   ```
   The release tag must match the `releaseTag` value in `ios/Package.swift`.

5. **Commit and push the parent repo** (`pluk-inc/convex-mobile`):
   ```
   cd ..
   git add rust/build-ios.sh ios
   git commit -m "release: <version>"
   git push
   ```

### How it works

- `build-ios.sh` compiles Rust for 5 Apple targets and produces universal (fat) binaries via `lipo`
- `IPHONEOS_DEPLOYMENT_TARGET=16.0` is set to avoid `___chkstk_darwin` linker errors with `aws-lc-sys`
- The `--release` flag zips the xcframework, computes the SPM checksum, and updates `releaseTag` and `releaseChecksum` in `ios/Package.swift` via `sed`
- `Package.swift` references the xcframework zip by URL from `pluk-inc/convex-swift` GitHub Releases
- SPM downloads and caches the zip at package resolution time

### XCFramework slices

| Slice | Architectures | Purpose |
|-------|--------------|---------|
| `ios-arm64` | arm64 | iPhone/iPad devices |
| `ios-arm64_x86_64-simulator` | arm64, x86_64 | iOS Simulator (Apple Silicon + Intel) |
| `macos-arm64_x86_64` | arm64, x86_64 | macOS (Apple Silicon + Intel) |
