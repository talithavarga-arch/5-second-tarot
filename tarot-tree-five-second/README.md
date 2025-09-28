# The Tarot Tree — Five Second Tarot Reading

This is your single‑file HTML app. You can run it locally, push to GitHub, and deploy to Vercel.

## 1) Run locally (no build tools)
1. Double‑click `index.html` to open it in your browser **or** serve it on `http://localhost:8000`:
   ```bash
   cd tarot-tree-five-second
   python3 -m http.server 8000
   ```
   Then visit http://localhost:8000

> Note: The UI will work, but AI responses will **fail** until you add a backend proxy for your Gemini API key (see below).

## 2) Create a Git repo and push to GitHub
```bash
cd tarot-tree-five-second
git init
git add .
git commit -m "Initial commit: Tarot Tree five-second reading"
# Create a new empty repo on GitHub named tarot-tree-five-second
git branch -M main
git remote add origin https://github.com/<your-username>/tarot-tree-five-second.git
git push -u origin main
```

## 3) Deploy to Vercel (static)
- Go to vercel.com, **New Project** → Import your GitHub repo.
- Framework preset: **Other** (static).
- Build command: **None**. Output directory: **/** (root).

This will deploy `index.html` as a static site. The Gemini call (client‑side) still won’t work unless you expose a key (not recommended) or add a serverless API route.

## 4) (Recommended) Add a Vercel serverless function to proxy Gemini
Create `api/gemini.js` in your repo to keep your API key secret:

```js
export default async function handler(req, res) {
  if (req.method !== "POST") return res.status(405).json({ error: "Method not allowed" });
  try {
    const { prompt } = await req.json();
    const apiKey = process.env.GEMINI_API_KEY;
    const apiUrl = \`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-05-20:generateContent?key=\${apiKey}\`;
    const payload = { contents: [{ parts: [{ text: prompt }]}] };
    const r = await fetch(apiUrl, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload)
    });
    const data = await r.json();
    return res.status(r.ok ? 200 : 500).json(data);
  } catch (e) {
    console.error(e);
    return res.status(500).json({ error: "Proxy error" });
  }
}
```

Then update `index.html` to call your proxy instead of Google directly:

```js
// Replace callGeminiAPI with:
async function callGeminiAPI(promptText) {
  const r = await fetch("/api/gemini", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ prompt: promptText })
  });
  if (!r.ok) throw new Error(await r.text());
  const data = await r.json();
  const text = data?.candidates?.[0]?.content?.parts?.[0]?.text;
  return text || "Could not generate an interpretation at this time. Please try again.";
}
```

Finally, in Vercel Project Settings → **Environment Variables**:
- Key: `GEMINI_API_KEY`
- Value: your Gemini key
- Scope: Production (and Preview, if desired)
- Redeploy

## 5) Add a custom domain
- In Vercel Project → **Settings** → **Domains** → Add your domain.
- Update your DNS: set the domain’s A/ALIAS or CNAME to Vercel. Vercel will show exact instructions.
- Wait for DNS to propagate and HTTPS to issue automatically.

---

### Quick Troubleshooting
- **Blank/blocked AI responses locally:** You haven’t added a backend proxy yet (see step 4). The UI still works.
- **Images not loading:** The tarot image host occasionally rate-limits. Your UI shows the card name fallback, which is ok.
- **Git push fails:** Make sure the remote exists and you’re authenticated with GitHub CLI or a token.

