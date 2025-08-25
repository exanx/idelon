# Self-Hosting Guide for Idolon Chat

This guide provides a comprehensive, step-by-step walkthrough for deploying your own private and secure instance of Idolon Chat. By following these instructions, you will use a free-tier combination of Firebase and Cloudflare to host the application and a secure proxy for your API keys.

**The core principle of this setup is that your API keys are never exposed to the browser.**

---

## ► Prerequisites

Before you begin, you will need the following:

1.  **Accounts:**
    *   A **[GitHub](https://github.com/)** account (to clone the repository).
    *   A **[Google / Firebase](https://firebase.google.com/)** account (free Spark plan is sufficient).
    *   A **[Cloudflare](https://dash.cloudflare.com/sign-up)** account (free plan is sufficient).

2.  **Command Line Tools:**
    *   **Node.js and npm:** [Install from the official website](https://nodejs.org/).
    *   **Firebase CLI:** In your terminal, run `npm install -g firebase-tools`.
    *   **Cloudflare Wrangler CLI:** In your terminal, run `npm install -g wrangler`.

---

## ► Part 1: Firebase Setup (Database & Authentication)

First, we'll create the backend foundation for your application.

### Step 1.1: Create a New Firebase Project
1.  Go to the [Firebase Console](https://console.firebase.google.com/) and click **"Add project"**.
2.  Enter a project name, for example, `idolon-chat-personal`.
3.  Continue through the setup. You can disable Google Analytics.

### Step 1.2: Create a Web App and Get Your Config
1.  Once the project is created, click the **Web icon (`</>`)** on the project dashboard.
2.  Give the app a nickname (e.g., "Idolon Chat").
3.  **Do not** check the box for "Also set up Firebase Hosting". We will do this manually later.
4.  Click "Register app".
5.  Firebase will show you a `firebaseConfig` object. **Copy this entire object into a temporary text file.** This is critical for connecting your website to this backend.

### Step 1.3: Enable Authentication Methods
1.  From the Firebase menu on the left, go to **Build → Authentication**.
2.  Click **"Get Started"**.
3.  On the "Sign-in method" tab, enable both **Google** and **Email/Password** providers.

### Step 1.4: Set Up Firestore Database & Security Rules
1.  Go to **Build → Firestore Database** and click **"Create database"**.
2.  Choose to start in **production mode**. This is the secure default.
3.  Select a server location (choose one close to you).
4.  After the database is created, click the **Rules** tab.
5.  Replace the default rules with the following code. This ensures a user can only read their own data.
    ```
    rules_version = '2';
    service cloud.firestore {
      match /databases/{database}/documents {
        // A user can only read their OWN documents. This is enforced by Firebase.
        match /users/{userId}/{document=**} {
          allow read: if request.auth != null && request.auth.uid == userId;
        }
      }
    }
    ```
6.  Click **Publish**.

**Firebase setup is complete! You now have a secure, empty backend.**

---

## ► Part 2: Cloudflare Worker Setup (The Secure Proxy)

Next, we will create the serverless proxy that will protect your API keys.

### Step 2.1: Create the Worker Project
1.  Open your terminal in a convenient location (like your main projects folder).
2.  Run the `wrangler init` command to create the project. Name it appropriately.
    ```bash
    wrangler init idolon-api-proxy
    ```
3.  Answer the setup prompts exactly as shown below:
    *   What would you like to start with? → **"Hello World" Worker**
    *   Which language do you want to use? → **JavaScript**
    *   Do you want to deploy your application? → **Yes**

### Step 2.2: Configure the Worker
1.  Navigate into the new worker directory: `cd idolon-api-proxy`
2.  Install the necessary dependency: `npm install crypto-js`
3.  Open the `package.json` file and **replace its entire contents** with this configuration. This ensures the worker can bundle Node.js packages correctly.
    ```json
    {
      "name": "idolon-api-proxy",
      "version": "0.0.0",
      "private": true,
      "scripts": {
        "deploy": "wrangler deploy",
        "start": "wrangler dev"
      },
      "devDependencies": {
        "wrangler": "3.57.2"
      },
      "dependencies": {
        "crypto-js": "^4.2.0"
      },
      "main": "src/index.js",
      "compatibility_date": "2025-08-21",
      "node_compat": true
    }
    ```

### Step 2.3: Add the Worker Code
1.  Open the file `src/index.js`.
2.  **Replace its entire contents** with the final, working proxy code below.
3.  **You must update two variables:**
    *   `FIREBASE_PROJECT_ID`: Set this to your Firebase Project ID (e.g., `idolon-chat-personal`).
    *   `Access-Control-Allow-Origin`: Set this to your final Firebase Hosting URL (e.g., `https://idolon-chat-personal.web.app`).

```javascript
// idolon-api-proxy/src/index.js
import CryptoJS from 'crypto-js';

// --- (1) UPDATE THIS ---
const FIREBASE_PROJECT_ID = 'your-firebase-project-id'; // e.g., 'idolon-chat-personal'

// Helper function to decrypt data
const decryptData = (ciphertext, key) => {
  if (!ciphertext || typeof ciphertext !== 'string' || !ciphertext.startsWith('U2F')) { return ciphertext; }
  try {
    const bytes = CryptoJS.AES.decrypt(ciphertext, key);
    const decryptedText = bytes.toString(CryptoJS.enc.Utf8);
    return decryptedText ? JSON.parse(decryptedText) : '';
  } catch (e) { console.warn('Decryption failed on server:', e); return ciphertext; }
};

// Fetches a document from Firestore using the REST API
async function getFirestoreDocument(path, authToken) {
  const url = `https://firestore.googleapis.com/v1/projects/${FIREBASE_PROJECT_ID}/databases/(default)/documents/${path}`;
  const response = await fetch(url, { headers: { 'Authorization': `Bearer ${authToken}` } });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(`Firestore fetch failed: ${error.error.message}`);
  }
  const data = await response.json();
  return data.fields ? formatFirestoreResponse(data.fields) : {};
}

// Formats the Firestore REST API response into a simple JS object
function formatFirestoreResponse(fields) {
    const result = {};
    for (const key in fields) {
        const valueType = Object.keys(fields[key])[0];
        result[key] = fields[key][valueType];
    }
    return result;
}

export default {
  async fetch(request, env, ctx) {
    if (request.method === 'OPTIONS') { return handleOptions(request); }
    const idToken = request.headers.get('Authorization')?.split('Bearer ')[1];
    if (!idToken) { return new Response(JSON.stringify({ error: 'Authentication token is required.' }), { status: 401, headers: corsHeaders() }); }
    
    const { provider, payload } = await request.json();
    const uid = JSON.parse(atob(idToken.split('.')[1])).user_id;

    let apiKey; let apiUrl; let apiHeaders = { 'Content-Type': 'application/json' };

    try {
      const [settings, secrets] = await Promise.all([
        getFirestoreDocument(`users/${uid}/settings/main`, idToken),
        getFirestoreDocument(`users/${uid}/secrets/encryptionKey`, idToken)
      ]);
      const encryptionKey = secrets.key;
      if (!encryptionKey) throw new Error("Encryption key not found.");

      switch (provider) {
        case 'gemini':
          const geminiKeyValue = settings.apiKeys.values[0].mapValue.fields.key.stringValue;
          apiKey = decryptData(geminiKeyValue, encryptionKey);
          const model = settings.selectedModel.stringValue || 'gemini-1.5-flash-latest';
          apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${apiKey}`;
          break;
        case 'groq':
          apiKey = decryptData(settings.groqApiKey, encryptionKey);
          apiUrl = 'https://api.groq.com/openai/v1/chat/completions';
          apiHeaders['Authorization'] = `Bearer ${apiKey}`;
          break;
        case 'openrouter':
          apiKey = decryptData(settings.openRouterApiKey, encryptionKey);
          apiUrl = 'https://openrouter.ai/api/v1/chat/completions';
          apiHeaders['Authorization'] = `Bearer ${apiKey}`;
          apiHeaders['HTTP-Referer'] = 'https://your-app-name.web.app'; // Replace with your app's URL
          apiHeaders['X-Title'] = 'Idolon Chat';
          break;
        default: throw new Error('Invalid API provider.');
      }
      if (!apiKey) { throw new Error('API key is missing or could not be decrypted.'); }
    } catch (error) {
      console.error("Error processing request in worker:", error);
      return new Response(JSON.stringify({ error: `Could not retrieve API key: ${error.message}` }), { status: 500, headers: corsHeaders() });
    }

    try {
      const response = await fetch(apiUrl, { method: 'POST', headers: apiHeaders, body: JSON.stringify(payload) });
      const responseBody = await response.json();
      if (!response.ok) { throw new Error(responseBody.error?.message || 'External API call failed.'); }
      return new Response(JSON.stringify(responseBody), { status: 200, headers: corsHeaders() });
    } catch (error) {
      return new Response(JSON.stringify({ error: error.message }), { status: 502, headers: corsHeaders() });
    }
  },
};

function corsHeaders() {
    return {
        // --- (2) UPDATE THIS ---
        'Access-Control-Allow-Origin': 'https://your-firebase-hosting-url.web.app', // e.g., 'https://idolon-chat-personal.web.app'
        'Access-Control-Allow-Methods': 'POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Authorization, Content-Type',
    };
}
function handleOptions(request) {
    if (request.headers.get('Origin') !== null && request.headers.get('Access-Control-Request-Method') !== null && request.headers.get('Access-Control-Request-Headers') !== null) {
        return new Response(null, { headers: corsHeaders() });
    } else {
        return new Response(null, { headers: { Allow: 'POST, OPTIONS' } });
    }
}
```

### Step 2.4: Securely Add Your Firebase Service Key
1.  Go to your Firebase project's settings (<span class="path">⚙️ → Project settings → Service accounts</span>).
2.  Click **"Generate new private key"** to download the project's unique JSON key file.
3.  Open the file and copy its **entire content**.
4.  In your terminal (inside the `idolon-api-proxy` folder), run:
    ```bash
    wrangler secret put FIREBASE_SERVICE_ACCOUNT
    ```
5.  Paste the JSON content when prompted and press Enter.

### Step 2.5: Deploy the Worker
1.  In the same terminal, run the final deploy command:
    ```bash
    wrangler deploy
    ```
2.  **Success!** Your secure proxy is now live. **Copy the final URL** provided by Wrangler (e.g., `https://idolon-api-proxy.your-user.workers.dev`).

---

## ► Part 3: Frontend Setup & Final Deployment

This final part connects your website to your new backend and proxy.

### Step 3.1: Configure the Website Code
1.  Clone or download the Idolon Chat source code from GitHub.
2.  Open the `index.html` file in your code editor.
3.  Find the `firebaseConfig` object and **replace it with the one you saved in Step 1.2**.
4.  Find the `callApiThroughProxy` function and **replace the placeholder `proxyUrl` with the worker URL you copied in Step 2.5**.

### Step 3.2: Deploy the Website
1.  Create a new folder for your project (e.g., `idolon-chat-live`) and navigate into it.
2.  Run `firebase init hosting`.
    *   Choose **"Use an existing project"** and select the Firebase project you created.
    *   Set `public` as your public directory.
    *   Configure as a single-page app by answering **Yes**.
    *   **Do not** set up automatic builds with GitHub.
3.  Copy your modified `index.html` and all other necessary assets (like the `char_imgs` folder, `manifest.json`, etc.) into the `public` folder that was just created.
4.  Finally, deploy your website:
    ```bash
    firebase deploy --only hosting
    ```

**You are done! Your new, fully secure, and private instance of Idolon Chat is now live.**