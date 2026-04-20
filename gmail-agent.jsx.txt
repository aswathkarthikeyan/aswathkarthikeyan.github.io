import { useState, useRef, useEffect, useCallback } from "react";

const GEMINI_URL =
  "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent";
const GMAIL_BASE = "https://gmail.googleapis.com/gmail/v1/users/me";

const INJECT_CSS = `
  @import url('https://fonts.googleapis.com/css2?family=Sora:wght@300;400;500;600;700&display=swap');
  * { box-sizing:border-box; margin:0; padding:0; }
  ::-webkit-scrollbar { width:4px; }
  ::-webkit-scrollbar-track { background:transparent; }
  ::-webkit-scrollbar-thumb { background:#1e3a52; border-radius:4px; }
  @keyframes fadeUp { from{opacity:0;transform:translateY(10px)} to{opacity:1;transform:translateY(0)} }
  @keyframes blink  { 0%,100%{opacity:1} 50%{opacity:0.3} }
  @keyframes spin   { to{transform:rotate(360deg)} }
  .msg-anim { animation: fadeUp 0.22s ease both; }
  .blink    { animation: blink 1.4s ease infinite; }
  .spin     { animation: spin 0.75s linear infinite; }
  .si {
    width:100%; background:#0c1929; border:1.5px solid #1b3350;
    border-radius:12px; color:#cde8f7; font-family:'Sora',sans-serif;
    font-size:15px; padding:14px 18px; outline:none; transition:border-color 0.2s;
  }
  .si:focus { border-color:#00c9a7; }
  .si::placeholder { color:#2e5570; }
  .ta {
    flex:1; background:#0c1929; border:1.5px solid #1b3350; border-radius:14px;
    color:#cde8f7; font-family:'Sora',sans-serif; font-size:15px;
    padding:14px 18px; outline:none; resize:none; transition:border-color 0.2s;
    line-height:1.5; max-height:140px;
  }
  .ta:focus { border-color:#00c9a7; }
  .ta::placeholder { color:#2e5570; }
  .bp {
    background:linear-gradient(135deg,#00c9a7,#0090d4);
    border:none; border-radius:12px; color:#050e1a;
    font-family:'Sora',sans-serif; font-size:15px; font-weight:700;
    cursor:pointer; transition:opacity 0.15s,transform 0.1s; padding:14px 24px;
  }
  .bp:active  { transform:scale(0.97); }
  .bp:disabled{ opacity:0.35; cursor:not-allowed; transform:none; }
  .bg {
    background:transparent; border:1.5px solid #1b3350; border-radius:10px;
    color:#4a7a9a; font-family:'Sora',sans-serif; font-size:13px;
    cursor:pointer; padding:8px 16px; transition:all 0.15s;
  }
  .bg:hover { border-color:#00c9a7; color:#00c9a7; }
  .chip {
    display:inline-flex; align-items:center; gap:6px;
    background:#0c1929; border:1px solid #1b3350; border-radius:20px;
    padding:7px 14px; font-size:12px; color:#4a7a9a; cursor:pointer;
    transition:all 0.15s; white-space:nowrap; font-family:'Sora',sans-serif;
  }
  .chip:hover { border-color:#00c9a7; color:#00c9a7; }
  a { color:#00c9a7; }
`;

function b64dec(s) {
  return atob(s.replace(/-/g, "+").replace(/_/g, "/"));
}
function buildMime(to, subject, body) {
  const raw = [
    "To: " + to,
    "Subject: " + subject,
    "MIME-Version: 1.0",
    "Content-Type: text/plain; charset=utf-8",
    "",
    body,
  ].join("\r\n");
  return btoa(unescape(encodeURIComponent(raw)))
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
}

const TOOLS = [
  {
    functionDeclarations: [
      {
        name: "list_emails",
        description: "List recent Gmail emails with subject, from, date, snippet.",
        parameters: {
          type: "OBJECT",
          properties: {
            query: { type: "STRING", description: "Optional Gmail filter" },
            maxResults: { type: "INTEGER", description: "How many to return (default 8)" },
          },
        },
      },
      {
        name: "read_email",
        description: "Read full body of one email by its message ID.",
        parameters: {
          type: "OBJECT",
          properties: { id: { type: "STRING" } },
          required: ["id"],
        },
      },
      {
        name: "search_emails",
        description: "Search Gmail (from:, subject:, has:attachment, is:unread, etc.)",
        parameters: {
          type: "OBJECT",
          properties: { query: { type: "STRING" } },
          required: ["query"],
        },
      },
      {
        name: "create_draft",
        description: "Save a draft. Always use this first unless user explicitly says to send.",
        parameters: {
          type: "OBJECT",
          properties: {
            to: { type: "STRING" },
            subject: { type: "STRING" },
            body: { type: "STRING" },
          },
          required: ["to", "subject", "body"],
        },
      },
      {
        name: "send_email",
        description: "Send an email. Only call if user explicitly confirmed they want to send.",
        parameters: {
          type: "OBJECT",
          properties: {
            to: { type: "STRING" },
            subject: { type: "STRING" },
            body: { type: "STRING" },
          },
          required: ["to", "subject", "body"],
        },
      },
      {
        name: "reply_to_email",
        description: "Reply to a thread, optionally as a draft.",
        parameters: {
          type: "OBJECT",
          properties: {
            threadId: { type: "STRING" },
            to: { type: "STRING" },
            subject: { type: "STRING" },
            body: { type: "STRING" },
            asDraft: { type: "BOOLEAN", description: "true = save as draft instead of sending" },
          },
          required: ["threadId", "to", "subject", "body"],
        },
      },
    ],
  },
];

export default function GmailAgent() {
  const [screen, setScreen] = useState("setup");
  const [gKey,   setGKey]   = useState("");
  const [cId,    setCId]    = useState("");
  const [token,  setToken]  = useState(null);
  const [email,  setEmail]  = useState("");
  const [msgs,   setMsgs]   = useState([]);
  const [hist,   setHist]   = useState([]);
  const [input,  setInput]  = useState("");
  const [busy,   setBusy]   = useState(false);
  const [status, setStatus] = useState("");
  const [gisOk,  setGisOk]  = useState(false);

  const bottomRef = useRef(null);
  const taRef     = useRef(null);

  useEffect(() => {
    const style = document.createElement("style");
    style.textContent = INJECT_CSS;
    document.head.appendChild(style);
    const sc = document.createElement("script");
    sc.src = "https://accounts.google.com/gsi/client";
    sc.onload = () => setGisOk(true);
    document.head.appendChild(sc);
  }, []);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [msgs, busy]);

  const gFetch = useCallback(
    async (path, opts = {}) => {
      const r = await fetch(GMAIL_BASE + path, {
        ...opts,
        headers: {
          Authorization: "Bearer " + token,
          "Content-Type": "application/json",
          ...(opts.headers || {}),
        },
      });
      if (!r.ok) {
        const e = await r.json().catch(() => ({}));
        throw new Error((e && e.error && e.error.message) || "Gmail error " + r.status);
      }
      return r.json();
    },
    [token]
  );

  const hdr = (msg, name) =>
    ((msg.payload && msg.payload.headers) || []).find(
      (h) => h.name.toLowerCase() === name.toLowerCase()
    )?.value || "";

  const extractBody = (payload) => {
    if (!payload) return "";
    if (payload.mimeType === "text/plain" && payload.body && payload.body.data) {
      return b64dec(payload.body.data);
    }
    for (const p of payload.parts || []) {
      const t = extractBody(p);
      if (t) return t;
    }
    return "";
  };

  const execFn = useCallback(
    async (name, args) => {
      if (name === "list_emails") {
        const q   = args.query || "";
        const max = Math.min(args.maxResults || 8, 20);
        const d   = await gFetch("/messages?maxResults=" + max + "&q=" + encodeURIComponent(q));
        if (!d.messages || !d.messages.length) return { emails: [], note: "No emails found." };
        const emails = await Promise.all(
          d.messages.map(async (m) => {
            const msg = await gFetch(
              "/messages/" + m.id +
                "?format=metadata&metadataHeaders=Subject&metadataHeaders=From&metadataHeaders=Date"
            );
            return {
              id: m.id,
              subject: hdr(msg, "Subject"),
              from: hdr(msg, "From"),
              date: hdr(msg, "Date"),
              snippet: msg.snippet,
            };
          })
        );
        return { emails };
      }

      if (name === "read_email") {
        const msg = await gFetch("/messages/" + args.id + "?format=full");
        return {
          id: msg.id,
          threadId: msg.threadId,
          subject: hdr(msg, "Subject"),
          from: hdr(msg, "From"),
          to: hdr(msg, "To"),
          date: hdr(msg, "Date"),
          body: extractBody(msg.payload) || msg.snippet,
        };
      }

      if (name === "search_emails") {
        const d = await gFetch("/messages?maxResults=10&q=" + encodeURIComponent(args.query));
        if (!d.messages || !d.messages.length) return { emails: [], count: 0 };
        const emails = await Promise.all(
          d.messages.map(async (m) => {
            const msg = await gFetch(
              "/messages/" + m.id +
                "?format=metadata&metadataHeaders=Subject&metadataHeaders=From&metadataHeaders=Date"
            );
            return {
              id: m.id,
              subject: hdr(msg, "Subject"),
              from: hdr(msg, "From"),
              date: hdr(msg, "Date"),
              snippet: msg.snippet,
            };
          })
        );
        return { emails, count: emails.length };
      }

      if (name === "create_draft") {
        const d = await gFetch("/drafts", {
          method: "POST",
          body: JSON.stringify({ message: { raw: buildMime(args.to, args.subject, args.body) } }),
        });
        return { success: true, draftId: d.id };
      }

      if (name === "send_email") {
        const d = await gFetch("/messages/send", {
          method: "POST",
          body: JSON.stringify({ raw: buildMime(args.to, args.subject, args.body) }),
        });
        return { success: true, messageId: d.id };
      }

      if (name === "reply_to_email") {
        const raw = buildMime(args.to, args.subject, args.body);
        const payload = { raw, threadId: args.threadId };
        if (args.asDraft) {
          const d = await gFetch("/drafts", {
            method: "POST",
            body: JSON.stringify({ message: payload }),
          });
          return { success: true, draftId: d.id };
        }
        const d = await gFetch("/messages/send", {
          method: "POST",
          body: JSON.stringify(payload),
        });
        return { success: true, messageId: d.id };
      }

      return { error: "Unknown function: " + name };
    },
    [gFetch]
  );

  const sendMsg = useCallback(
    async (override) => {
      const txt = (override || input).trim();
      if (!txt || busy) return;
      setInput("");
      const newMsgs = [...msgs, { role: "user", text: txt }];
      setMsgs(newMsgs);
      setBusy(true);
      let conv = [...hist, { role: "user", parts: [{ text: txt }] }];
      try {
        for (let i = 0; i < 12; i++) {
          setStatus("Thinking...");
          const sysText =
            "You are a smart Gmail assistant for " +
            (email || "the user") +
            ".\n" +
            "Rules:\n" +
            "- When composing: always show the full draft to the user and ask Shall I save this as a draft or send it? before calling create_draft or send_email.\n" +
            "- Only call send_email if the user explicitly confirmed sending.\n" +
            "- When listing emails: format as a numbered list with sender, subject, date, and snippet. Never expose raw message IDs.\n" +
            "- Be concise. Use markdown.";
          const res = await fetch(GEMINI_URL + "?key=" + gKey, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              systemInstruction: { parts: [{ text: sysText }] },
              contents: conv,
              tools: TOOLS,
              generationConfig: { temperature: 0.65, maxOutputTokens: 2048 },
            }),
          });
          const data = await res.json();
          if (data.error) throw new Error(data.error.message);
          const content = data.candidates && data.candidates[0] && data.candidates[0].content;
          if (!content) throw new Error("Empty response from Gemini.");
          conv.push(content);
          const fnCalls = (content.parts || []).filter((p) => p.functionCall);
          if (!fnCalls.length) {
            const reply =
              (content.parts || [])
                .filter((p) => p.text)
                .map((p) => p.text)
                .join("") || "(no reply)";
            setMsgs([...newMsgs, { role: "assistant", text: reply }]);
            setHist(conv);
            break;
          }
          const results = [];
          for (const part of fnCalls) {
            const fn   = part.functionCall;
            const name = fn.name;
            const args = fn.args;
            setStatus("Running: " + name.replace(/_/g, " ") + "...");
            let result;
            try {
              result = await execFn(name, args);
            } catch (e) {
              result = { error: e.message };
            }
            results.push({ functionResponse: { name, response: result } });
          }
          conv.push({ role: "user", parts: results });
        }
      } catch (e) {
        setMsgs([...newMsgs, { role: "error", text: "Error: " + e.message }]);
      } finally {
        setBusy(false);
        setStatus("");
        setTimeout(() => taRef.current && taRef.current.focus(), 80);
      }
    },
    [input, busy, msgs, hist, gKey, email, execFn]
  );

  const connectGmail = () => {
    if (!window.google || !cId) return;
    window.google.accounts.oauth2
      .initTokenClient({
        client_id: cId,
        scope:
          "https://www.googleapis.com/auth/gmail.modify https://www.googleapis.com/auth/userinfo.email",
        callback: async (r) => {
          if (!r.access_token) return;
          setToken(r.access_token);
          try {
            const info = await (
              await fetch("https://www.googleapis.com/oauth2/v2/userinfo", {
                headers: { Authorization: "Bearer " + r.access_token },
              })
            ).json();
            setEmail(info.email || "");
          } catch (_) {}
        },
      })
      .requestAccessToken();
  };

  const canLaunch = gKey.trim().length > 10 && cId.trim().length > 10 && !!token;

  const quick = [
    { l: "📥 Latest",     m: "Show my 8 latest emails" },
    { l: "🔴 Unread",     m: "Show unread emails" },
    { l: "✉️ Compose",   m: "Help me write a new email" },
    { l: "📎 With files", m: "Find emails with attachments" },
    { l: "🔍 Search",     m: "I need to find a specific email" },
  ];

  const rootStyle = {
    minHeight: "100vh",
    background: "#060b14",
    color: "#cde8f7",
    fontFamily: "'Sora', sans-serif",
    display: "flex",
    flexDirection: "column",
  };

  const teal = { color: "#00c9a7" };

  // ── SETUP ──────────────────────────────────────────────────────────────────
  if (screen === "setup") {
    return (
      <div style={rootStyle}>
        <div
          style={{
            flex: 1,
            display: "flex",
            flexDirection: "column",
            alignItems: "center",
            justifyContent: "center",
            padding: "24px 20px",
          }}
        >
          <div style={{ textAlign: "center", marginBottom: 36 }}>
            <div
              style={{
                width: 62,
                height: 62,
                borderRadius: 18,
                margin: "0 auto 14px",
                background: "linear-gradient(135deg,#00c9a7,#0090d4)",
                display: "flex",
                alignItems: "center",
                justifyContent: "center",
                fontSize: 28,
                boxShadow: "0 0 36px #00c9a730",
              }}
            >
              ⚡
            </div>
            <div style={{ fontSize: 26, fontWeight: 700, letterSpacing: -0.5 }}>
              Gmail<span style={teal}>AI</span>
            </div>
            <div style={{ fontSize: 13, color: "#2d5a78", marginTop: 5 }}>
              Your inbox, commanded by Gemini
            </div>
          </div>

          <div
            style={{
              width: "100%",
              maxWidth: 460,
              background: "#0c1929",
              border: "1px solid #1b3350",
              borderRadius: 20,
              padding: "28px 24px",
              display: "flex",
              flexDirection: "column",
              gap: 22,
            }}
          >
            {/* Step 1 */}
            <div>
              <div
                style={{
                  fontSize: 11,
                  fontWeight: 600,
                  letterSpacing: 1.2,
                  color: "#00c9a7",
                  marginBottom: 8,
                  textTransform: "uppercase",
                }}
              >
                Step 1 — Gemini API Key
              </div>
              <input
                className="si"
                type="password"
                placeholder="AIza..."
                value={gKey}
                onChange={(e) => setGKey(e.target.value)}
              />
              <div style={{ fontSize: 11, color: "#2d5a78", marginTop: 6, lineHeight: 1.5 }}>
                Free at{" "}
                <a href="https://aistudio.google.com" target="_blank" rel="noreferrer">
                  aistudio.google.com
                </a>{" "}
                — 1500 requests per day
              </div>
            </div>

            {/* Step 2 */}
            <div>
              <div
                style={{
                  fontSize: 11,
                  fontWeight: 600,
                  letterSpacing: 1.2,
                  color: "#00c9a7",
                  marginBottom: 8,
                  textTransform: "uppercase",
                }}
              >
                Step 2 — Google OAuth Client ID
              </div>
              <input
                className="si"
                type="text"
                placeholder="xxxx.apps.googleusercontent.com"
                value={cId}
                onChange={(e) => setCId(e.target.value)}
              />
              <div style={{ fontSize: 11, color: "#2d5a78", marginTop: 6, lineHeight: 1.5 }}>
                Create at{" "}
                <a href="https://console.cloud.google.com" target="_blank" rel="noreferrer">
                  console.cloud.google.com
                </a>{" "}
                — Enable Gmail API, Web app type
              </div>
            </div>

            {/* Step 3 */}
            <div>
              <div
                style={{
                  fontSize: 11,
                  fontWeight: 600,
                  letterSpacing: 1.2,
                  color: "#00c9a7",
                  marginBottom: 10,
                  textTransform: "uppercase",
                }}
              >
                Step 3 — Connect Gmail
              </div>
              {token ? (
                <div
                  style={{
                    display: "flex",
                    alignItems: "center",
                    gap: 10,
                    background: "#081a12",
                    border: "1px solid #00c9a740",
                    borderRadius: 12,
                    padding: "12px 16px",
                  }}
                >
                  <span style={{ fontSize: 18 }}>✅</span>
                  <div>
                    <div style={{ fontSize: 13, fontWeight: 600, color: "#00c9a7" }}>
                      Connected
                    </div>
                    <div style={{ fontSize: 12, color: "#2a7a5a" }}>{email}</div>
                  </div>
                </div>
              ) : (
                <button
                  className="bp"
                  style={{ width: "100%" }}
                  onClick={connectGmail}
                  disabled={!gisOk || !cId.trim()}
                >
                  {gisOk ? "🔗  Sign in with Google" : "Loading..."}
                </button>
              )}
            </div>

            {/* Launch */}
            <button
              className="bp"
              style={{ width: "100%", padding: "16px", fontSize: 16, marginTop: 4 }}
              disabled={!canLaunch}
              onClick={() => {
                setMsgs([
                  {
                    role: "assistant",
                    text: "Hey! I am your Gmail AI.\n\nI can read, search, compose, draft, and send emails for you. What do you need?",
                  },
                ]);
                setScreen("chat");
              }}
            >
              Launch Agent
            </button>
          </div>

          <div
            style={{
              marginTop: 18,
              fontSize: 11,
              color: "#1a3a50",
              textAlign: "center",
              maxWidth: 360,
              lineHeight: 1.7,
            }}
          >
            Zero data leaks. Your emails travel Google to Gemini only.
            <br />
            No backend. No storage. No third party.
          </div>
        </div>
      </div>
    );
  }

  // ── CHAT ───────────────────────────────────────────────────────────────────
  return (
    <div style={{ ...rootStyle, height: "100vh", overflow: "hidden" }}>
      {/* Topbar */}
      <div
        style={{
          display: "flex",
          alignItems: "center",
          justifyContent: "space-between",
          padding: "12px 18px",
          borderBottom: "1px solid #0e2035",
          background: "#080d18",
          flexShrink: 0,
        }}
      >
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div
            style={{
              width: 34,
              height: 34,
              borderRadius: 10,
              background: "linear-gradient(135deg,#00c9a7,#0090d4)",
              display: "flex",
              alignItems: "center",
              justifyContent: "center",
              fontSize: 15,
            }}
          >
            ⚡
          </div>
          <div>
            <div style={{ fontSize: 14, fontWeight: 600 }}>
              Gmail<span style={teal}>AI</span>
            </div>
            <div style={{ fontSize: 11, color: "#2a5a7a" }}>{email}</div>
          </div>
        </div>

        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          {busy && (
            <div style={{ display: "flex", alignItems: "center", gap: 7 }}>
              <div
                className="spin"
                style={{
                  width: 13,
                  height: 13,
                  border: "2px solid #0e2035",
                  borderTop: "2px solid #00c9a7",
                  borderRadius: "50%",
                }}
              />
              <span style={{ fontSize: 11, color: "#2a6a8a" }}>{status}</span>
            </div>
          )}
          <button
            className="bg"
            onClick={() => {
              setMsgs([]);
              setHist([]);
            }}
          >
            Clear
          </button>
          <button className="bg" onClick={() => setScreen("setup")}>
            Settings
          </button>
        </div>
      </div>

      {/* Messages */}
      <div
        style={{
          flex: 1,
          overflowY: "auto",
          padding: "18px 14px 10px",
          display: "flex",
          flexDirection: "column",
          gap: 14,
        }}
      >
        {msgs.map((m, i) => (
          <div
            key={i}
            className="msg-anim"
            style={{
              display: "flex",
              flexDirection: m.role === "user" ? "row-reverse" : "row",
              gap: 9,
              alignItems: "flex-end",
            }}
          >
            {m.role !== "user" && (
              <div
                style={{
                  width: 30,
                  height: 30,
                  borderRadius: 9,
                  flexShrink: 0,
                  background:
                    m.role === "error"
                      ? "linear-gradient(135deg,#c0392b,#e74c3c)"
                      : "linear-gradient(135deg,#00c9a7,#0090d4)",
                  display: "flex",
                  alignItems: "center",
                  justifyContent: "center",
                  fontSize: 13,
                }}
              >
                {m.role === "error" ? "!" : "⚡"}
              </div>
            )}
            <div
              style={{
                maxWidth: "82%",
                background:
                  m.role === "user"
                    ? "#0f2035"
                    : m.role === "error"
                    ? "#1a0808"
                    : "#0c1929",
                border:
                  "1px solid " +
                  (m.role === "user"
                    ? "#1b4060"
                    : m.role === "error"
                    ? "#4a1010"
                    : "#1b3350"),
                borderRadius:
                  m.role === "user"
                    ? "16px 4px 16px 16px"
                    : "4px 16px 16px 16px",
                padding: "12px 15px",
                fontSize: 14,
                lineHeight: 1.7,
                color: m.role === "error" ? "#f08080" : "#cde8f7",
                whiteSpace: "pre-wrap",
                wordBreak: "break-word",
              }}
            >
              {m.text}
            </div>
          </div>
        ))}

        {busy && (
          <div
            className="msg-anim"
            style={{ display: "flex", gap: 9, alignItems: "flex-end" }}
          >
            <div
              style={{
                width: 30,
                height: 30,
                borderRadius: 9,
                background: "linear-gradient(135deg,#00c9a7,#0090d4)",
                display: "flex",
                alignItems: "center",
                justifyContent: "center",
                fontSize: 13,
              }}
            >
              ⚡
            </div>
            <div
              style={{
                background: "#0c1929",
                border: "1px solid #1b3350",
                borderRadius: "4px 16px 16px 16px",
                padding: "14px 18px",
                display: "flex",
                gap: 6,
                alignItems: "center",
              }}
            >
              {[0, 1, 2].map((n) => (
                <div
                  key={n}
                  className="blink"
                  style={{
                    width: 7,
                    height: 7,
                    borderRadius: "50%",
                    background: "#00c9a7",
                    animationDelay: n * 0.2 + "s",
                  }}
                />
              ))}
            </div>
          </div>
        )}
        <div ref={bottomRef} />
      </div>

      {/* Quick chips */}
      {msgs.length <= 1 && (
        <div
          style={{
            display: "flex",
            gap: 8,
            padding: "6px 14px 8px",
            overflowX: "auto",
            flexShrink: 0,
          }}
        >
          {quick.map((a) => (
            <button key={a.l} className="chip" onClick={() => sendMsg(a.m)}>
              {a.l}
            </button>
          ))}
        </div>
      )}

      {/* Input */}
      <div
        style={{
          padding: "10px 14px 18px",
          borderTop: "1px solid #0e2035",
          background: "#080d18",
          flexShrink: 0,
          display: "flex",
          gap: 10,
          alignItems: "flex-end",
        }}
      >
        <textarea
          ref={taRef}
          className="ta"
          rows={1}
          placeholder="Ask me anything about your Gmail..."
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => {
            if (e.key === "Enter" && !e.shiftKey) {
              e.preventDefault();
              sendMsg();
            }
          }}
        />
        <button
          className="bp"
          style={{ padding: "13px 18px", flexShrink: 0, minWidth: 50, fontSize: 18 }}
          onClick={() => sendMsg()}
          disabled={busy || !input.trim()}
        >
          {busy ? (
            <div
              className="spin"
              style={{
                width: 15,
                height: 15,
                border: "2px solid #0a1f0f",
                borderTop: "2px solid #050e1a",
                borderRadius: "50%",
                margin: "0 auto",
              }}
            />
          ) : (
            "↑"
          )}
        </button>
      </div>
    </div>
  );
}
