---
name: app-development-guide
description: Moodle App development environment setup and workflow guide. Use when setting up a Moodle mobile app development environment, understanding the project structure, running or building the app, writing tests, or contributing to the Moodle App codebase.
---
# Moodle App Development Guide

Source: https://moodledev.io/general/app/development/development-guide

## Overview

The Moodle App is an Angular and Ionic Framework application that provides mobile access to Moodle sites. This guide covers setting up the development environment, running the app, and contributing to the codebase.

## Technology Stack

- **Framework**: Angular
- **Mobile UI**: Ionic Framework
- **Language**: TypeScript
- **Package Manager**: npm
- **Node Version Manager**: nvm (recommended)

## Getting Started

### Prerequisites

1. Install Node.js using nvm to ensure the correct version:
   ```bash
   nvm install
   nvm use
   ```

2. Clone the Moodle App repository:
   ```bash
   git clone https://github.com/moodlehq/moodleapp.git
   cd moodleapp
   ```

3. Install dependencies:
   ```bash
   npm install
   ```

### Running the App

Launch the app in a Chromium-based browser:
```bash
npm start
```

This is the primary way to develop and test the app without needing native device SDKs.

### Native Development

For building and testing on native iOS or Android devices:
- Install the appropriate platform SDKs (Android Studio or Xcode).
- Follow the Ionic Framework guides for native platform setup.

## Development Environment

### Recommended Editor

**VSCode** is the recommended editor, as:
- The core team uses it.
- The repository includes VSCode-specific settings (`.vscode/` directory).
- Built-in ESLint integration enforces code style automatically.

### Code Style Enforcement

- ESLint is used to enforce the [Moodle App Coding Style](https://moodledev.io/general/development/policies/codingstyle-moodleapp).
- Configure your editor to show ESLint warnings and errors in real time.
- Treat all ESLint warnings as errors.

## Project Structure

```
moodleapp/
├── src/
│   ├── app/             # App module, root component
│   ├── core/            # Core services, directives, pipes
│   ├── addons/          # Add-on features (plugins, modules)
│   └── assets/          # Static assets
├── scripts/             # Build and development scripts
└── www/                 # Compiled output (do not edit directly)
```

## Customization vs. Plugin Support

- **App core changes / custom apps**: Use this development guide.
- **Adding mobile support to Moodle plugins**: Use the [Moodle App Plugins Development Guide](https://moodledev.io/general/app/development/plugins-development-guide) instead.

## Key Development Topics

### Debugging

- Use browser DevTools in Chromium for debugging when running via `npm start`.
- Use the Network tab to inspect web service calls.
- Use the Angular DevTools Chrome extension for component and state debugging.

### Testing

- Unit tests use Jest.
- Run tests with:
  ```bash
  npm test
  ```

### Accessibility

- Follow WCAG accessibility guidelines.
- Use semantic HTML and ARIA attributes.
- Test with screen readers.

### Push Notifications

- The app supports custom push notifications via the Moodle Mobile plugin infrastructure.
- Refer to the push notification documentation for sending custom notifications.

## Workflow Summary

1. Clone the repository and install dependencies.
2. Run `npm start` to launch the app in your browser.
3. Edit TypeScript, HTML, and SCSS files.
4. Changes are live-reloaded automatically.
5. Ensure ESLint passes before submitting changes.
6. Run `npm test` to verify unit tests pass.

## References

- [Moodle App Development Guide](https://moodledev.io/general/app/development/development-guide)
- [Moodle App Coding Style](https://moodledev.io/general/development/policies/codingstyle-moodleapp)
- [Moodle App Plugins Development Guide](https://moodledev.io/general/app/development/plugins-development-guide)
- [Moodle App Repository on GitHub](https://github.com/moodlehq/moodleapp)
- [Setting Up the Development Environment](https://moodledev.io/general/app/development/setup)
