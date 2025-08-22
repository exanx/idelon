<div align="center">
  <img src="assets/zyrenmascotfull.png" alt="Zyren AI Mascot" width="250"/>
  <h1>Zyren Chat</h1>
  <p><strong>A Private, Secure, and Infinitely Customizable AI Chat Experience.</strong></p>
  
  <p>
    <a href="https://zyren.web.app"><strong>Try the Live Demo →</strong></a>
  </p>
  
  <p>
    <img src="https://img.shields.io/badge/Hosting-Firebase-orange?style=for-the-badge&logo=firebase" alt="Firebase Hosting">
    <img src="https://img.shields.io/badge/Proxy-Cloudflare-f38020?style=for-the-badge&logo=cloudflare" alt="Cloudflare Workers">
    <img src="https://img.shields.io/badge/Language-JavaScript-F7DF1E?style=for-the-badge&logo=javascript" alt="JavaScript">
    <img src="https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge" alt="License: MIT">
  </p>
</div>

---

## 🚀 What is Zyren Chat?

Zyren Chat is a sleek, modern chat interface that puts **your privacy and control first**. Unlike other AI chat services, Zyren operates on a "Bring Your Own Key" (BYOK) model. This means you connect your personal API keys from various AI providers, and your conversations happen directly between you and the AI, secured by an enterprise-grade serverless proxy.

**We never see your API keys or your chat conversations. Ever.**

This project started as a fork of Gemna Chat and has been rebuilt to offer a completely independent, secure, and rebranded experience.

## ✨ Key Features

-   🔐 **Ultimate Privacy & Security:** All API calls are routed through a secure Cloudflare Worker. Your keys are encrypted in your private database and are never exposed to the browser.
-   🔑 **Bring Your Own Key (BYOK):** Full support for your personal API keys from:
    -   Google Gemini
    -   Groq
    -   OpenRouter
-   ☁️ **Cloud Synchronization:** With Firebase Authentication, all your characters, chats, and settings are securely synced across your devices.
-   🤖 **Custom AI Personas:** Create unique AI characters with distinct personalities, languages, and texting styles. Or, create complex group chats with multiple AI participants.
-   🎨 **Customizable Interface:** Choose from multiple themes that mimic popular messaging apps (iMessage, WhatsApp, etc.) and toggle between light and dark modes.
-   🔄 **Data Portability:** Easily import and export your characters and chat histories as `.json` files.
-   🗣️ **Text-to-Speech:** Have the AI's responses read aloud with a voice that intelligently matches its persona (powered by Gemini).
-   🖼️ **AI Image Generation:** Use `/image` and `/sendpic` commands to generate images directly in the chat.
-   📱 **Progressive Web App (PWA):** Install Zyren Chat on your desktop or mobile device for a native-app-like experience.

## 🏛️ Secure Architecture Explained

The core of Zyren Chat's privacy model is its architecture. Your browser **never** communicates directly with the AI provider's API. Instead, it follows this secure flow:

```mermaid
graph TD
    A[User's Browser] -- 1. Sends Prompt & Firebase Auth Token --> B(Cloudflare Worker Proxy);
    B -- 2. Verifies Token & Fetches User's Data --> C{Firestore Database};
    C -- 3. Securely Returns Encrypted API Key --> B;
    B -- 4. Decrypts Key & Forwards Prompt --> D[External AI API (Gemini, Groq, etc.)];
    D -- 5. Sends Response Back --> B;
    B -- 6. Forwards AI Response to User --> A;
