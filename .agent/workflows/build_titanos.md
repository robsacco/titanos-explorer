---
description: Builds the TitanOS iOS project to verify compilation
---

1. Navigate to the project directory
   - Check if you are in the root or a subdirectory. You need to be where `titanos.xcodeproj` or `titanos.xcworkspace` is located.
   - Ideally: `cd /Users/robsacco/Projects/titanos-explorer/titanos-refactored/titanos`

2. Run the build command
   - Use `xcodebuild` to build the `titanos` scheme for the iOS Simulator.
   - Command: `xcodebuild -scheme titanos -destination 'platform=iOS Simulator,name=iPhone 15 Pro' clean build | xcbeautify` (or just `cat` if xcbeautify is not available)
   - If that fails, standard output is fine: `xcodebuild -scheme titanos -destination 'platform=iOS Simulator,name=iPhone 15 Pro' clean build`

3. Check the output
   - Look for `** BUILD SUCCEEDED **`.
   - If it fails, analyze the error messages in the output log.
