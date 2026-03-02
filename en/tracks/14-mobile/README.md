# Track 14 — Mobile

Build production-quality mobile applications using web technologies. Whether you're targeting the browser with PWAs or native app stores with React Native, this track covers the architecture, tooling, and deployment workflows that ship real products.

**Estimated time:** 5–6 weeks

---

## Topics

1. [PWA Fundamentals](pwa-fundamentals.md) — service workers, web manifest, offline-first, install prompt, push notifications, Workbox
2. [React Native Basics](react-native-basics.md) — Expo vs bare workflow, core components, StyleSheet, flexbox, navigation, platform-specific code
3. [React Native Advanced](react-native-advanced.md) — performance optimization, native modules, Reanimated, Gesture Handler, deep linking
4. [Mobile Navigation](mobile-navigation.md) — stack/tab/drawer navigators, nested navigation, auth flow pattern, deep links, universal links
5. [Mobile State Management](mobile-state-management.md) — Zustand + MMKV, React Query on mobile, offline sync patterns, optimistic updates
6. [Mobile Deployment](mobile-deployment.md) — EAS Build, App Store and Play Store submission, OTA updates, CI/CD for mobile, app signing
7. [Mobile Testing](mobile-testing.md) — Jest + Expo config, React Native Testing Library, mocking native modules, Detox E2E, Maestro, CI workflows
8. [Mobile Security](mobile-security.md) — OWASP Mobile Top 10, expo-secure-store, certificate pinning, biometric auth, jailbreak detection, code obfuscation
9. [Native Device APIs](native-device-apis.md) — expo-camera, expo-location, push notifications (FCM/APNs), sensors, haptics, expo-media-library

---

## Prerequisites

- Track 02 — Frontend (React, TypeScript, CSS)
- Track 01 — Foundations (async/await, fetch, module system)
- Track 03 — Backend (REST APIs — mobile apps are API consumers)

---

## What You'll Build

- A PWA that works fully offline, installs on the home screen, and receives push notifications
- A React Native app with tab and stack navigation, persistent auth state, and platform-specific UI
- A mobile app with optimistic UI updates and background sync that gracefully handles intermittent connectivity
- A complete EAS Build + EAS Update CI/CD pipeline that deploys OTA updates on every merge to main
- A fully tested React Native app with Jest unit tests, RNTL component tests, and Detox E2E flows
- A security-hardened app with certificate pinning, biometric auth, and secure token storage
- An app using camera, location, and push notifications with proper permission flows

---

## Track Philosophy

Mobile is a constrained environment: limited CPU, intermittent network, small screen, and users who uninstall apps immediately if performance is poor. Every pattern in this track is chosen because it survives those constraints. You will learn to measure before optimizing, design for offline-first from day one, and ship confidently with automated build pipelines.
