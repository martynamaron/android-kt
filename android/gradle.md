## Gradle

Gradle is a build automation system that uses a domain-specific language to describe the app's project structure, configuration, and dependencies.

- project-level `build.gradle` file: contains configuration options and dependencies common to all modules in the project
- app (or module) level `build.gradle` file: module-specific dependencies. The `app` module is always present in Android projects and its `build.gradle` file contains app-level configuration such as version numbers, SDK levels, etc.