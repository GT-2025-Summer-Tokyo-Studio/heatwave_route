
# 🔗 Project Sync and Deployment Instructions

This guide explains how to connect your local project to a new GitHub organization repository, push your updates, and deploy both the frontend and backend services.

---

## ✅ 1. Sync Local Project with New GitHub Org Repo

If you already added the new remote URL:

### Option A – Safe Sync (Recommended)
```bash
git pull origin main --rebase
git push origin main
```

### Option B – Overwrite Remote (Force Push)
⚠️ Use this only if you're sure your local version should replace the remote:
```bash
git push origin main --force
```

---

## 🌐 2. Deploy Flask Backend to Render

1. Go to [https://render.com](https://render.com)
2. Click **"New Web Service"**
3. Connect your GitHub repo: `GT-2025-Summer-Tokyo-Studio/flask_server`
4. Use the following deployment settings:

| Setting           | Value                          |
|------------------|--------------------------------|
| Name             | flask-server (or any name)     |
| Environment      | Python                         |
| Build Command    | pip install -r requirements.txt|
| Start Command    | gunicorn app:app               |
| Python Version   | Auto (from runtime.txt)        |

5. Click **"Create Web Service"**

Render will build and host your app. You’ll get a **public URL** like:
```
https://flask-server.onrender.com
```
Which is ready to connect to your frontend.

---

## 🖥️ 3. Deploy Frontend Web Tool (heatwave_route)

Assuming the frontend is a Vite-based app:

### Install Node Modules
```bash
npm install
```

### Start Local Dev Server (for development)
```bash
npm run dev
```

### Deploy to GitHub Pages (for production)

1. Build the project:
```bash
npm run build
```

2. Install deployment helper:
```bash
npm install gh-pages --save-dev
```

3. Deploy to GitHub Pages:
```bash
npx gh-pages -d dist
```

Make sure your GitHub repo has the correct `gh-pages` settings to serve the `/dist` folder.

---

## 🧩 Endpoints for Testing

Test the deployed backend:
```
GET https://your-render-url.onrender.com/query-shelters?address=nihonbashi%20station
GET https://your-render-url.onrender.com/query-routes?address=nihonbashi%20station&shelter_id=1
```

These will return JSON or GeoJSON output that your frontend can consume.
