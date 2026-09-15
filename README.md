# Rajmata Polytechnic — College Website (MERN)

A working MERN scaffold built to match the layout and structure of the
reference site (ice.edu.in): sticky nav with dropdown programs menu, hero,
quick-facts strip, program listing, "why us," placements, testimonials,
admissions CTA, and a footer — plus a real Express/MongoDB API behind the
courses, notices, and contact-form pages.

The visual design is original (not copied from the reference site) — navy
and amber palette, a drafting-grid texture, and Space Grotesk/IBM Plex Sans
type, meant to evoke an engineering institute rather than a generic template.
Sample content (college name, address, programs) is placeholder — replace it
with your client's actual details before handing it off.

## Structure

```
college-website/
  backend/     Express + Mongoose API
  frontend/    React (Vite) client
```

## Prerequisites

- Node.js 18+
- MongoDB running locally, or a free MongoDB Atlas cluster

## 1. Backend setup

```bash
cd backend
npm install
cp .env.example .env
# edit .env if your MongoDB URI is different (e.g. an Atlas connection string)
npm run seed     # loads sample courses + notices into the database
npm run dev      # starts the API on http://localhost:5000
```

## 2. Frontend setup

In a second terminal:

```bash
cd frontend
npm install
npm run dev       # starts the site on http://localhost:5173
```

The dev server proxies `/api/*` requests to `http://localhost:5000`
(configured in `vite.config.js`), so both servers need to be running for
courses, notices, and the contact form to work. If nothing appears on the
Courses or Notices pages, that's the API/database not running — the pages
are built to fail gracefully rather than crash.

## What's wired up vs. what's a placeholder

**Working end-to-end:**
- Courses: list + detail pages read live from MongoDB via the API
- Notices: list reads live from MongoDB
- Contact form: posts to the API and saves to the `contactmessages`
  collection

**Placeholder / needs your input before going live:**
- College name, address, phone, email (currently "Rajmata Polytechnic" —
  find-and-replace across `frontend/src`)
- Photography — the "why us" section and hero currently use a grid-texture
  placeholder block where a real campus/lab photo should go
- About/Admissions page copy — written generically, adjust for the actual
  institute
- No authentication yet on the POST/PUT/DELETE routes (`/api/courses`,
  `/api/notices`) — add an admin login before exposing those publicly, since
  right now anyone who finds the API could edit content.

## Admin Panel Architecture (Firebase Authentication - Free Tier)

The proposal document promised staff a login-protected way to manage notices, events, and course details without touching code. This scaffold has the backend REST routes ready (`POST/PUT/DELETE /api/courses`, `POST/DELETE /api/notices`). 

To secure these routes using **Firebase Authentication (Spark Free Tier)**, follow this pattern:

### 1. Firebase Project Setup (Free Tier)
1. Go to [Firebase Console](https://console.firebase.google.com/) and create a new project.
2. Under **Authentication** -> **Sign-in method**, enable **Email/Password**.
3. Under **Authentication** -> **Users**, manually add authorized staff/admin email accounts (e.g., `admin@rajmata.edu.in`).

### 2. Backend Middleware Verification (`backend/`)
1. Install Firebase Admin SDK:
   ```bash
   cd backend
   npm install firebase-admin
   ```
2. Download your service account JSON key from Firebase Console (**Project Settings** -> **Service Accounts**).
3. Create an authentication middleware `backend/middleware/auth.js`:
   ```javascript
   const admin = require("firebase-admin");

   admin.initializeApp({
     credential: admin.credential.cert({
       projectId: process.env.FIREBASE_PROJECT_ID,
       clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
       privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, "\n"),
     }),
   });

   async function verifyFirebaseToken(req, res, next) {
     const authHeader = req.headers.authorization;
     if (!authHeader || !authHeader.startsWith("Bearer ")) {
       return res.status(401).json({ error: "Unauthorized: Missing token" });
     }
     const idToken = authHeader.split("Bearer ")[1];
     try {
       const decodedToken = await admin.auth().verifyIdToken(idToken);
       req.user = decodedToken;
       next();
     } catch (err) {
       return res.status(403).json({ error: "Forbidden: Invalid or expired token" });
     }
   }

   module.exports = verifyFirebaseToken;
   ```
4. Protect write endpoints in `courses.js` & `notices.js`:
   ```javascript
   const verifyFirebaseToken = require("../middleware/auth");
   router.post("/", verifyFirebaseToken, async (req, res) => { ... });
   router.put("/:id", verifyFirebaseToken, async (req, res) => { ... });
   router.delete("/:id", verifyFirebaseToken, async (req, res) => { ... });
   ```

### 3. Frontend Admin Dashboard (`frontend/`)
1. Install Firebase Client SDK:
   ```bash
   cd frontend
   npm install firebase
   ```
2. Initialize Firebase Client (`frontend/src/firebase.js`):
   ```javascript
   import { initializeApp } from "firebase/app";
   import { getAuth } from "firebase/auth";

   const firebaseConfig = {
     apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
     authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
     projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
   };

   const app = initializeApp(firebaseConfig);
   export const auth = getAuth(app);
   ```
3. Add `/admin` Route & Protected Forms (`frontend/src/pages/Admin.jsx`):
   - **Login View**: Uses `signInWithEmailAndPassword(auth, email, password)`.
   - **Dashboard View**: Displays forms to create/delete notices and edit courses.
   - **API Requests**: Passes the ID token via headers:
     ```javascript
     const token = await auth.currentUser.getIdToken();
     fetch("/api/notices", {
       method: "POST",
       headers: {
         "Content-Type": "application/json",
         Authorization: `Bearer ${token}`
       },
       body: JSON.stringify(noticeData)
     });
     ```

---

## Comprehensive Step-by-Step Deployment Guide

Follow these steps to deploy the application on **GitHub**, **MongoDB Atlas (Free M0)**, **Railway (Free Tier Backend)**, and **Vercel (Free Hobby Frontend)**.

### Step 1: Upload Code to GitHub
1. Open your terminal in the root folder (`college-website`):
   ```bash
   git init
   git add .
   git commit -m "Initial commit of college website"
   ```
2. Create a new repository on [GitHub](https://github.com/new) named `college-website`.
3. Push your repository:
   ```bash
   git remote add origin https://github.com/YOUR_USERNAME/college-website.git
   git branch -M main
   git push -u origin main
   ```

### Step 2: Database Setup (MongoDB Atlas Free Tier)
1. Go to [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) and create an **M0 Free Cluster**.
2. Create a database user in **Database Access** (e.g. username: `college_admin`, password: `your_secure_password`).
3. In **Network Access**, click **Add IP Address** -> Select **Allow Access from Anywhere** (`0.0.0.0/0`) so Railway cloud instances can connect.
4. Click **Database** -> **Connect** -> **Drivers** and copy your URI connection string:
   `mongodb+srv://college_admin:<password>@cluster0.xxxx.mongodb.net/college_website?retryWrites=true&w=majority`
5. Seed initial data directly into MongoDB Atlas:
   ```bash
   cd backend
   MONGO_URI="mongodb+srv://college_admin:your_secure_password@cluster0.xxxx.mongodb.net/college_website?retryWrites=true&w=majority" node seed.js
   ```

### Step 3: Deploy Backend on Render (Free Web Service)
1. Log in to [Render.com](https://render.com) using your GitHub account.
2. Click **New +** -> **Web Service**.
3. Select **Build and deploy from a Git repository** and connect your `college-website` repository.
4. Configure Web Service settings:
   - **Name**: `college-website-backend`
   - **Root Directory**: `backend`
   - **Environment / Runtime**: `Node`
   - **Build Command**: `npm install`
   - **Start Command**: `node server.js`
   - **Instance Type**: **Free**
5. Add Environment Variables (under **Environment** tab):
   - `MONGO_URI`: `mongodb+srv://college_admin:<password>@cluster0.xxxx.mongodb.net/college_website?retryWrites=true&w=majority`
   - `PORT`: `5000`
   - `CLIENT_ORIGIN`: `*` (or your Vercel URL once deployed)
6. Click **Create Web Service**.
7. Copy your backend URL once deployed (e.g., `https://college-website-backend.onrender.com`).

### Step 4: Deploy Frontend on Vercel (Free Hobby Tier)
1. Log in to [Vercel](https://vercel.com) using your GitHub account.
2. Click **Add New...** -> **Project** -> Select your `college-website` repository.
3. Configure Project Settings:
   - **Framework Preset**: Vite
   - **Root Directory**: Click Edit -> Select `frontend`
4. Configure Environment Variables:
   - Add Key: `VITE_API_URL`
   - Value: `https://college-website-backend.onrender.com` *(your Render backend URL from Step 3)*
5. Click **Deploy**. Vercel will build the frontend and provide your live link (e.g., `https://college-website.vercel.app`).

### Step 5: Final Verification
1. Visit your Vercel frontend URL.
2. Verify that **Courses** and **Notices** load data from MongoDB Atlas via your Render API.
3. Submit a test inquiry on the **Contact** page to verify database write operations.
