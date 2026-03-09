# Quick Installation

## 1. Install dotenv Package

```bash
npm install dotenv --save
```

## 2. Copy Files to Your Project

Copy these files to your Parachord project root:
- `.env` - Your actual credentials (don't commit!)
- `.env.example` - Template for other developers
- `.gitignore` - Protects sensitive files
- `main.js` - Updated to use environment variables

## 3. Verify Setup

Check that your project structure looks like:
```
parachord/
├── .env              ← Your credentials (in .gitignore!)
├── .env.example      ← Template (safe to commit)
├── .gitignore        ← Protects .env file
├── main.js           ← Updated with dotenv
├── app.js
├── preload.js
├── index.html
├── package.json
└── node_modules/
```

## 4. Restart App

```bash
npm start
```

## 5. Check Console

You should see (in terminal where you ran `npm start`):
```
=== Electron App Starting ===
```

No errors about missing credentials!

## 6. Test Spotify

1. Click "Connect Spotify" in app
2. Complete OAuth flow
3. Should see: `Token exchange successful`

---

## ✅ What Changed

### Before (Hardcoded - DO NOT DO THIS):
```javascript
const clientId = 'YOUR_CLIENT_ID_HERE';  // ❌ Never hardcode credentials!
const clientSecret = 'YOUR_CLIENT_SECRET_HERE';  // ❌ Security risk!
```

### After (Environment Variables):
```javascript
require('dotenv').config();
const clientId = process.env.SPOTIFY_CLIENT_ID;
const clientSecret = process.env.SPOTIFY_CLIENT_SECRET;
```

---

## 🔒 Security Benefits

✅ Credentials stored in `.env` (not committed to git)
✅ `.gitignore` protects `.env` file
✅ `.env.example` provides template without secrets
✅ Other developers can use their own credentials
✅ Easy to rotate credentials if compromised

---

See **API_CREDENTIALS_SETUP.md** for full documentation!
