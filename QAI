const http = require('http');
const fs = require('fs');
const path = require('path');
const { URL } = require('url');

const PORT = process.env.PORT || 5050;
const HOST = '0.0.0.0';
const GEMINI_API_KEY = process.env.GEMINI_API_KEY || process.env.GOOGLE_API_KEY || '';
const PERPLEXITY_API_KEY = process.env.PERPLEXITY_API_KEY || '';
const BRIDGE_PORT = process.env.BRIDGE_PORT || 8675;
const PUBLIC_BASE_URL = process.env.PUBLIC_BASE_URL || '';

const staticHtml = path.join(__dirname, 'qgenesis-android-shell.html');
const sessions = [];

function send(res, status, data, type = 'application/json') {
  res.writeHead(status, {
    'Content-Type': type,
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET,POST,OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization'
  });
  res.end(type === 'application/json' ? JSON.stringify(data, null, 2) : data);
}

function readBody(req) {
  return new Promise((resolve, reject) => {
    let data = '';
    req.on('data', chunk => data += chunk);
    req.on('end', () => {
      try { resolve(data ? JSON.parse(data) : {}); }
      catch (e) { reject(e); }
    });
    req.on('error', reject);
  });
}

async function callGemini(prompt) {
  if (!GEMINI_API_KEY) return { ok: false, provider: 'gemini', error: 'Missing GEMINI_API_KEY or GOOGLE_API_KEY' };
  const model = 'gemini-2.5-flash';
  const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${GEMINI_API_KEY}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] })
  });
  const json = await response.json();
  if (!response.ok) return { ok: false, provider: 'gemini', status: response.status, error: json.error?.message || 'Gemini request failed', raw: json };
  const text = json.candidates?.[0]?.content?.parts?.map(p => p.text).join('\n') || 'No text returned.';
  return { ok: true, provider: 'gemini', model, text, raw: json };
}

async function callPerplexity(prompt) {
  if (!PERPLEXITY_API_KEY) return { ok: false, provider: 'perplexity', error: 'Missing PERPLEXITY_API_KEY' };
  const model = 'sonar';
  const response = await fetch('https://api.perplexity.ai/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${PERPLEXITY_API_KEY}`
    },
    body: JSON.stringify({
      model,
      messages: [
        { role: 'system', content: 'You are the retrieval layer for QGenesis Android bridge. Keep responses compact and operational.' },
        { role: 'user', content: prompt }
      ]
    })
  });
  const json = await response.json();
  if (!response.ok) return { ok: false, provider: 'perplexity', status: response.status, error: json.error?.message || 'Perplexity request failed', raw: json };
  const text = json.choices?.[0]?.message?.content || 'No text returned.';
  return { ok: true, provider: 'perplexity', model, text, raw: json };
}

const server = http.createServer(async (req, res) => {
  if (req.method === 'OPTIONS') return send(res, 204, {});

  const parsed = new URL(req.url, `http://${req.headers.host}`);

  if (req.method === 'GET' && parsed.pathname === '/') {
    const html = fs.readFileSync(staticHtml, 'utf8');
    return send(res, 200, html, 'text/html; charset=utf-8');
  }

  if (req.method === 'GET' && parsed.pathname === '/api/status') {
    return send(res, 200, {
      ok: true,
      service: 'qgenesis-bridge-server',
      port: PORT,
      bridge_port: BRIDGE_PORT,
      public_base_url: PUBLIC_BASE_URL,
      providers: {
        gemini: !!GEMINI_API_KEY,
        perplexity: !!PERPLEXITY_API_KEY
      },
      sessions: sessions.length,
      now: new Date().toISOString()
    });
  }

  if (req.method === 'POST' && parsed.pathname === '/api/providers/test') {
    const body = await readBody(req).catch(() => ({}));
    const prompt = body.prompt || 'Return a one-line status for QGenesis Android bridge.';
    const provider = body.provider || 'gemini';
    const result = provider === 'perplexity' ? await callPerplexity(prompt) : await callGemini(prompt);
    return send(res, result.ok ? 200 : 500, result);
  }

  if (req.method === 'POST' && parsed.pathname === '/api/chat/route') {
    const body = await readBody(req).catch(() => ({}));
    const prompt = body.prompt || 'Summarize bridge state.';
    const primary = body.primary || 'gemini';
    const fallback = body.fallback || 'perplexity';

    let result = primary === 'perplexity' ? await callPerplexity(prompt) : await callGemini(prompt);
    if (!result.ok && fallback && fallback !== primary) {
      result = fallback === 'perplexity' ? await callPerplexity(prompt) : await callGemini(prompt);
      result.fallback_used = true;
    }

    sessions.unshift({ at: new Date().toISOString(), prompt, primary, fallback, ok: result.ok, provider: result.provider });
    if (sessions.length > 25) sessions.pop();

    return send(res, result.ok ? 200 : 500, {
      ok: result.ok,
      provider: result.provider,
      fallback_used: !!result.fallback_used,
      text: result.text || result.error,
      detail: result.error || null
    });
  }

  if (req.method === 'GET' && parsed.pathname === '/api/history') {
    return send(res, 200, { ok: true, sessions });
  }

  return send(res, 404, { ok: false, error: 'Not found' });
});

server.listen(PORT, HOST, () => {
  console.log(`QGenesis bridge server live on http://${HOST}:${PORT}`);
  console.log(`Public URL: ${PUBLIC_BASE_URL || 'not set'}`);
  console.log(`Perplexity bridge reference port: ${BRIDGE_PORT}`);
});
