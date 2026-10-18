# [App Name]

## Basic Information

- **App Name**: 
- **Package ID**: 
- **Version Code**: 
- **Version Name**: 
- **Min SDK**: 29
- **Target SDK**: 36
- **Status**: Active / In Development / Deprecated

## Description

[Brief description of what this app does]

## API Keys & Credentials

### Required Environment Variables

Add these to `local.properties` in the root project:

```properties
# [App Name] Keys
KEY_NAME=your_key_here
```

### Key Details

| Key Name | Purpose | Where to get it | Notes |
|----------|---------|-----------------|-------|
| KEY_NAME | Description | URL or service | Any important notes |

## Features

- Feature 1
- Feature 2
- Feature 3

## Dependencies

### Core Modules
- List core dependencies

### External Services
- Firebase (if applicable)
- AdMob (if applicable)
- Other APIs

## Build Configuration

### Build Commands

```bash
# Debug build
./gradlew :app:[app-name]:assembleDebug

# Release build
./gradlew :app:[app-name]:assembleRelease

# Install on device
./gradlew :app:[app-name]:installDebug
```

### Flavors

[If applicable, describe product flavors]

## Testing

```bash
# Unit tests
./gradlew :app:[app-name]:test

# Instrumented tests
./gradlew :app:[app-name]:connectedAndroidTest
```

## Release Notes

### v1.0.0 (YYYY-MM-DD)
- Initial release

## Notes

[Any additional notes, gotchas, or important information]
