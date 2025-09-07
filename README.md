# Solace
# Personal Journal + AI Companion

A simple local-first journal with export, "save to memories", and an **optional AI Companion** that replies **on the page**.  
No accounts. All data stored in your browser's `localStorage`.

## Files
- `index.html` — journal + AI panel (demo mode + secure proxy mode)
- `memory.html` — browse/manage saved memories
- `My Journal.css` — styles
- (No build step; static site)

## Publish on GitHub Pages
1. Create a new repo and add these files.
2. Commit & push.
3. In your repo: **Settings → Pages → Build and deployment**
   - Source: **Deploy from a branch**
   - Branch: **main** (or `master`) / folder **/** (root)
4. Your site goes live at `https://<your-username>.github.io/<repo>/`
5. **Downloadable:** GitHub already provides **Code → Download ZIP** for the whole project.  
   You can also create a **Release** to offer a zip on the Releases page.

## Export data
- On the Journal page, click **Export Entries (.json)**.
- On the Memories page, click **Export Memories (.json)**.

## AI Companion (on-page replies)

### Quick Demo (no key)
- Works offline. Produces safe, supportive templated replies.
- In **Connection Settings**, choose **Demo**. No setup needed.

### Secure mode (recommended)
Browsers shouldn’t expose your API key. Use a tiny proxy that forwards your message to OpenAI and returns the model’s reply.

#### Option A: Cloudflare Worker (free tier ok)
1. Create a Worker in Cloudflare dashboard.
2. Paste the code below.
3. Add a Worker **secret** named `OPENAI_API_KEY` with your key.
4. Deploy, copy the Worker URL (e.g., `https://your-worker.workers.dev/therapist`).
5. In the app (**Connection Settings**), select **Secure (via Proxy)** and paste the Worker URL.

**Worker code (copy/paste):**
```js
export default {
  async fetch(request, env) {
    if (request.method !== "POST") {
      return new Response("POST only", { status: 405 });
    }
    const { message } = await request.json();
    if (!message) return new Response(JSON.stringify({ error: "message required" }), { status: 400 });

    // Call OpenAI Chat Completions
    const resp = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${env.OPENAI_API_KEY}`
      },
      body: JSON.stringify({
        model: "gpt-4o-mini",
        temperature: 0.7,
        messages: [
          { role: "system", content: "You are a supportive, non-judgmental journaling companion. Use short paragraphs, reflect feelings, ask gentle follow-ups. Avoid clinical diagnoses." },
          { role: "user", content: message }
        ]
      })
    });

    if (!resp.ok) {
      const t = await resp.text();
      return new Response(JSON.stringify({ error: "openai_error", detail: t }), { status: 500 });
    }
    const data = await resp.json();
    const reply = data?.choices?.[0]?.message?.content || "(no response)";
    return new Response(JSON.stringify({ reply }), { headers: { "Content-Type": "application/json" } });
  }
};
