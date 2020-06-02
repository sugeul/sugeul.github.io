---
title: "Xcodebuild hangs on '[CP] Prepare Artifacts'"
date: 2020-06-02 15:26:00 +0900
categories: iOS build
---

# case
1. I am working for iOS app service company and we are currently trying to implement a CD/CI system.
Because main service is an App, we need a build server that can run xcode and we cannot avoid this heavy and machine-dependant solution.
So thus I got a Mac Mini 3 weeks ago, and I tried to build a iOS app project with xcodebuild (bundled CLI in Xcode).
2. But it didn't work and hung on below line:

```
[14:54:00]: ▸ Running script '[CP] Check Pods Manifest.lock'
[14:54:00]: ▸ Running script '[CP] Prepare Artifacts'
```

3. Unfortunately it was not proceeded any futher.
Actually, I tried first with [Fastlane](https://fastlane.tools/) and suddenly it stops, but I found that the problem is not from Fastlane but Xcodebuild itself.

# investigation

First I tried to run xcodebuild with auto-generated options(by Fastlane).
(I found this line among logs.)
```
[14:52:43]: $ set -o pipefail \
&& xcodebuild -workspace {app name}.xcworkspace -scheme {scheme} \
-destination 'generic/platform=iOS' \
-archivePath /Users/{account name}/Library/Developer/Xcode/Archives/2020-06-02/{app name build time}.xcarchive archive \
| tee /Users/{account name}/Library/Logs/gym/{scheme}-{target}.log | xcpretty
```

Of course it won't work!
But I could get more detailed logs: specifically script names what they call.
Bottom of logs, I found a suspect script.

```
PhaseScriptExecution [CP]\ Check\ Pods\ Manifest.lock /Users/{account name}/Library/Developer/Xcode/DerivedData/{app name}-exxqzfkyglyqzddfquixpniuhhgt/Build/Intermediates.noindex/ArchiveIntermediates/{app name}/IntermediateBuildFilesPath/{app name}.build/Release-iphoneos/{app name}.build/Script-8BCA08A542D112C1643305D0.sh
    cd {project path}
    /bin/sh -c /Users/{account name}/Library/Developer/Xcode/DerivedData/{app name}-exxqzfkyglyqzddfquixpniuhhgt/Build/Intermediates.noindex/ArchiveIntermediates/{app name}/IntermediateBuildFilesPath/{app name}.build/Release-iphoneos/{app name}.build/Script-8BCA08A542D112C1643305D0.sh

PhaseScriptExecution [CP]\ Prepare\ Artifacts /Users/{account name}/Library/Developer/Xcode/DerivedData/{app name}-exxqzfkyglyqzddfquixpniuhhgt/Build/Intermediates.noindex/ArchiveIntermediates/{app name}/IntermediateBuildFilesPath/{app name}.build/Release-iphoneos/{app name}.build/Script-BAF0AFE0439786DEDB034137.sh
    cd {project path}
    /bin/sh -c /Users/{account name}/Library/Developer/Xcode/DerivedData/{app name}-exxqzfkyglyqzddfquixpniuhhgt/Build/Intermediates.noindex/ArchiveIntermediates/{app name}/IntermediateBuildFilesPath/{app name}.build/Release-iphoneos/{app name}.build/Script-BAF0AFE0439786DEDB034137.sh
```

I opened the script:
## Script-BAF0AFE0439786DEDB034137.sh
```
#!/bin/sh
"${PODS_ROOT}/Target Support Files/Pods-{app name}/Pods-{app name}-artifacts.sh"
```

We can find `PODS_ROOT` from the logs: (It might be `{your project's home}/pods`)
```
    export PODS_BUILD_DIR=...
    export PODS_CONFIGURATION_BUILD_DIR=...
    export PODS_ROOT=...
    export PODS_TARGET_SRCROOT=...
    export PRECOMPS_INCLUDE_HEADERS_FROM_BUILT_PRODUCTS_DIR=YES
```

.. and following that:
## ${PODS_ROOT}/Target Support Files/Pods-{app name}/Pods-{app name}-artifacts.sh

```
#!/bin/sh
set -e
set -u
set -o pipefail

function on_error {
  echo "$(realpath -mq "${0}"):$1: error: Unexpected failure"
}
trap 'on_error $LINENO' ERR

if [ -z ${FRAMEWORKS_FOLDER_PATH+x} ]; then
  # If FRAMEWORKS_FOLDER_PATH is not set, then there's nowhere for us to copy
  # frameworks to, so exit 0 (signalling the script phase was successful).
  exit 0
fi

# This protects against multiple targets copying the same framework dependency at the same time. The solution
# was originally proposed here: https://lists.samba.org/archive/rsync/2008-February/020158.html
RSYNC_PROTECT_TMP_FILES=(--filter "P .*.??????")

ARTIFACT_LIST_FILE="${BUILT_PRODUCTS_DIR}/cocoapods-artifacts-${CONFIGURATION}.txt"
cat > $ARTIFACT_LIST_FILE

BCSYMBOLMAP_DIR="BCSymbolMaps"

record_artifact()
{
  echo "$1" >> $ARTIFACT_LIST_FILE
}

install_artifact()
{
  local source="$1"
  local destination="$2"
  local record=${3:-false}

  # Use filter instead of exclude so missing patterns don't throw errors.
  echo "rsync --delete -av "${RSYNC_PROTECT_TMP_FILES[@]}" --links --filter \"- CVS/\" --filter \"- .svn/\" --filter \"- .git/\" --filter \"- .hg/\" \"${source}\" \"${destination}\""
  rsync --delete -av "${RSYNC_PROTECT_TMP_FILES[@]}" --links --filter "- CVS/" --filter "- .svn/" --filter "- .git/" --filter "- .hg/" "${source}" "${destination}"

  if [[ "$record" == "true" ]]; then
    artifact="${destination}/$(basename "$source")"
    record_artifact "$artifact"
  fi
}

# Copies a framework to derived data for use in later build phases
install_framework()
{
  if [ -r "${BUILT_PRODUCTS_DIR}/$1" ]; then
    local source="${BUILT_PRODUCTS_DIR}/$1"
  elif [ -r "${BUILT_PRODUCTS_DIR}/$(basename "$1")" ]; then
    local source="${BUILT_PRODUCTS_DIR}/$(basename "$1")"
  elif [ -r "$1" ]; then
    local source="$1"
  fi

  local record_artifact=${2:-true}
  local destination="${CONFIGURATION_BUILD_DIR}"

  if [ -L "${source}" ]; then
    echo "Symlinked..."
    source="$(readlink "${source}")"
  fi

  install_artifact "$source" "$destination" "$record_artifact"

  if [ -d "${source}/${BCSYMBOLMAP_DIR}" ]; then
    # Locate and install any .bcsymbolmaps if present
    find "${source}/${BCSYMBOLMAP_DIR}/" -name "*.bcsymbolmap"|while read f; do
      install_artifact "$f" "$destination" "true"
    done
  fi
}

install_xcframework() {
  local basepath="$1"
  local dsym_folder="$2"
  local embed="$3"
  shift
  local paths=("$@")

  # Locate the correct slice of the .xcframework for the current architectures
  local target_path=""
  local target_arch="$ARCHS"

  # Replace spaces in compound architectures with _ to match slice format
  target_arch=${target_arch// /_}

  local target_variant=""
  if [[ "$PLATFORM_NAME" == *"simulator" ]]; then
    target_variant="simulator"
  fi
  if [[ ! -z ${EFFECTIVE_PLATFORM_NAME+x} && "$EFFECTIVE_PLATFORM_NAME" == *"maccatalyst" ]]; then
    target_variant="maccatalyst"
  fi
  for i in ${!paths[@]}; do
    if [[ "${paths[$i]}" == *"$target_arch"* ]] && [[ "${paths[$i]}" == *"$target_variant"* ]]; then
      # Found a matching slice
      echo "Selected xcframework slice ${paths[$i]}"
      target_path=${paths[$i]}
      break;
    fi
  done

  if [[ -z "$target_path" ]]; then
    echo "warning: [CP] Unable to find matching .xcframework slice in '${paths[@]}' for the current build architectures ($ARCHS)."
    return
  fi

  install_framework "$basepath/$target_path" "$embed"

  if [[ -z "$dsym_folder" || ! -d "$dsym_folder" ]]; then
    return
  fi

  dsyms=($(ls "$dsym_folder"))


  local target_dsym=""
  for i in ${!dsyms[@]}; do
    install_artifact "$dsym_folder/${dsyms[$i]}" "$CONFIGURATION_BUILD_DIR" "true"
  done
}


if [[ "$CONFIGURATION" == "Debug" ]]; then
  install_xcframework "${PODS_ROOT}/mobile-ffmpeg-full/mobileffmpeg.xcframework" "" "false" "ios-x86_64-simulator/mobileffmpeg.framework" "ios-x86_64-maccatalyst/mobileffmpeg.framework" "ios-arm64/mobileffmpeg.framework"
  ...
  install_xcframework "${PODS_ROOT}/mobile-ffmpeg-full/wavpack.xcframework" "" "false" "ios-x86_64-maccatalyst/wavpack.framework" "ios-arm64/wavpack.framework" "ios-x86_64-simulator/wavpack.framework"
fi
if [[ "$CONFIGURATION" == "Release" ]]; then
  install_xcframework "${PODS_ROOT}/mobile-ffmpeg-full/mobileffmpeg.xcframework" "" "false" "ios-x86_64-simulator/mobileffmpeg.framework" "ios-x86_64-maccatalyst/mobileffmpeg.framework" "ios-arm64/mobileffmpeg.framework"
  ...
  install_xcframework "${PODS_ROOT}/mobile-ffmpeg-full/wavpack.xcframework" "" "false" "ios-x86_64-maccatalyst/wavpack.framework" "ios-arm64/wavpack.framework" "ios-x86_64-simulator/wavpack.framework"
fi

echo "Artifact list stored at $ARTIFACT_LIST_FILE"

cat "$ARTIFACT_LIST_FILE"

```

to inspect the moment it hangs at, I added `set -x` to top of the script.

# break-through

After that, I found that code hangs at below line!
```
ARTIFACT_LIST_FILE="${BUILT_PRODUCTS_DIR}/cocoapods-artifacts-${CONFIGURATION}.txt"
cat > $ARTIFACT_LIST_FILE
```
so.. what for `cat > $ARTIFACT_LIST_FILE`? cat reads inputs but nothing was put in and it hung forever.

I commented out the line, then..

## fastlane release
```
[15:41:03]: ▸ total size is 573512  speedup is 1.00
[15:41:03]: ▸ Artifact list stored at /Users/{account name}/Library/Developer/Xcode/DerivedData/{app name}-exxqzfkyglyqzddfquixpniuhhgt/Build/Intermediates.noindex/ArchiveIntermediates/{app name}/BuildProductsPath/Release-iphoneos/cocoapods-artifacts-Release.txt
[15:41:03]: ▸ cat: /Users/{account name}/Library/Developer/Xcode/DerivedData/{app name}-exxqzfkyglyqzddfquixpniuhhgt/Build/Intermediates.noindex/ArchiveIntermediates/{app name}/BuildProductsPath/Release-iphoneos/cocoapods-artifacts-Release.txt: No such file or directory
[15:41:03]: ▸ /Users/{account name}/Vlogr2/Vlogr/Pods/Target Support Files/Pods-{app name}/Pods-{app name}-artifacts.sh: line 7: realpath: command not found
[15:41:03]: ▸ :227: error: Unexpected failure
```

As the log, we need the file(even it is empty). so I changed cat >` with `touch`. then..

## fastlane release

```
15:45:59]: ▸ Generating 'Vlogr.app.dSYM'
[15:46:02]: ▸ Running script 'Run Script'
[15:46:03]: ▸ Running script '[CP] Copy Pods Resources'
[15:46:03]: ▸ Copying .../*.appex
[15:46:03]: ▸ Running script '[CP] Embed Pods Frameworks'
[15:46:04]: ▸ Copying .../*.framework
[15:46:04]: ▸ Signing .../*.framework
[15:46:04]: ▸ Touching {app name}.app
[15:46:06]: ▸ Signing .../{app name}.app
[15:46:11]: ▸ Touching {app name}.app.dSYM
[15:46:11]: ▸ Archive Succeeded
```

done! it works!
I stil don't know what for `cat >` in that line, but we actually didn't need any lines ot `ARTIFACT_LIST_FILE`.
It was very hard and took 3 weeks to investigate(in part time). Hope this article may help someone who may surffer same pain. :-)


