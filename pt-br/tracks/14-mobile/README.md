# Track 14 — Mobile

Construa aplicações mobile de qualidade em produção usando tecnologias web. Seja visando o browser com PWAs ou as lojas de apps nativas com React Native, este track cobre a arquitetura, as ferramentas e os fluxos de deploy que entregam produtos reais.

**Tempo estimado:** 5–6 semanas

---

## Tópicos

1. [PWA Fundamentals](pwa-fundamentals.md) — service workers, web manifest, offline-first, install prompt, push notifications, Workbox
2. [React Native Basics](react-native-basics.md) — Expo vs bare workflow, componentes principais, StyleSheet, flexbox, navegação, código específico por plataforma
3. [React Native Advanced](react-native-advanced.md) — otimização de performance, módulos nativos, Reanimated, Gesture Handler, deep linking
4. [Mobile Navigation](mobile-navigation.md) — navegadores stack/tab/drawer, navegação aninhada, padrão de fluxo de autenticação, deep links, universal links
5. [Mobile State Management](mobile-state-management.md) — Zustand + MMKV, React Query no mobile, padrões de sincronização offline, atualizações otimistas
6. [Mobile Deployment](mobile-deployment.md) — EAS Build, submissão na App Store e Play Store, OTA updates, CI/CD para mobile, assinatura de apps
7. [Testes Mobile](mobile-testing.md) — Jest + Expo, React Native Testing Library, mocks de módulos nativos, Detox E2E, Maestro, CI
8. [Segurança Mobile](mobile-security.md) — OWASP Mobile Top 10, expo-secure-store, certificate pinning, autenticação biométrica, detecção de jailbreak, ofuscação de código
9. [APIs Nativas do Dispositivo](native-device-apis.md) — expo-camera, expo-location, notificações push (FCM/APNs), sensores, haptics, expo-media-library

---

## Pré-requisitos

- Track 02 — Frontend (React, TypeScript, CSS)
- Track 01 — Foundations (async/await, fetch, sistema de módulos)
- Track 03 — Backend (REST APIs — apps mobile são consumidores de API)

---

## O Que Você Vai Construir

- Uma PWA que funciona totalmente offline, instala na tela inicial e recebe push notifications
- Um app React Native com navegação em tabs e stack, estado de autenticação persistente e UI específica por plataforma
- Um app mobile com atualizações otimistas de UI e sincronização em segundo plano que lida de forma elegante com conectividade intermitente
- Um pipeline completo de CI/CD com EAS Build + EAS Update que publica OTA updates a cada merge para a main
- Um app React Native com testes completos: testes unitários com Jest, testes de componentes com RNTL e fluxos E2E com Detox
- Um app com segurança reforçada: certificate pinning, autenticação biométrica e armazenamento seguro de tokens
- Um app que usa câmera, localização e notificações push com fluxos adequados de permissão

---

## Filosofia do Track

Mobile é um ambiente com restrições: CPU limitada, rede intermitente, tela pequena e usuários que desinstalam apps imediatamente se a performance for ruim. Cada padrão deste track foi escolhido porque sobrevive a essas restrições. Você vai aprender a medir antes de otimizar, projetar para offline-first desde o primeiro dia e fazer deploys com confiança usando pipelines de build automatizados.
