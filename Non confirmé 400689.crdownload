'use strict';
/*
  Strike Zone – serveur multijoueur gratuit et sans dépendance (Node.js 18+).
  - Sert le jeu (index.html) et les WebSocket sur le même port.
  - Recherche d'adversaire, parties privées par code, relais des positions, classement.
*/
const http = require('http'), fs = require('fs'), path = require('path'), crypto = require('crypto');
const PORT = +process.env.PORT || 3000;
const DIR = __dirname;
const GUID = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';
const MAXLEN = 4096, MAXCLIENTS = 400, MAXHOSTS = 300, MAXLB = 3000;
const GK = ['rn', 'ch', 'x', 'y', 'dx', 'dy', 'hp', 'sn', 'sx', 'sy', 'sdx', 'sdy', 'bm'];
const cl = (v, a, b) => { v = Math.floor(+v); return isNaN(v) ? a : Math.max(a, Math.min(b, v)); };
const cleanName = n => String(n || '').replace(/[\u0000-\u001f<>&"']/g, '').trim().slice(0, 14);

/* ---------- classement (mémoire + fichier, ou Upstash si configuré) ---------- */
const LB = new Map();
const LBFILE = path.join(DIR, 'lb.json');
const UP_URL = process.env.UPSTASH_REDIS_REST_URL, UP_TOK = process.env.UPSTASH_REDIS_REST_TOKEN;
async function up(cmd) {
  const r = await fetch(UP_URL, { method: 'POST', headers: { Authorization: 'Bearer ' + UP_TOK, 'Content-Type': 'application/json' }, body: JSON.stringify(cmd) });
  const j = await r.json();
  return j.result;
}
async function loadLB() {
  try {
    if (UP_URL && UP_TOK) {
      const a = (await up(['HGETALL', 'sz_lb'])) || [];
      for (let i = 0; i < a.length; i += 2) { try { LB.set(a[i], JSON.parse(a[i + 1])); } catch (e) {} }
      console.log('Classement chargé depuis Upstash :', LB.size, 'joueur(s)');
    } else if (fs.existsSync(LBFILE)) {
      const o = JSON.parse(fs.readFileSync(LBFILE, 'utf8'));
      for (const k in o) LB.set(k, o[k]);
      console.log('Classement chargé depuis lb.json :', LB.size, 'joueur(s)');
    }
  } catch (e) { console.log('Chargement du classement impossible :', e.message); }
}
let saveT = null; const dirty = new Set();
function persist(id) {
  dirty.add(id);
  if (saveT) return;
  saveT = setTimeout(async () => {
    saveT = null; const ids = [...dirty]; dirty.clear();
    try {
      if (UP_URL && UP_TOK) { for (const k of ids) { const r = LB.get(k); if (r) await up(['HSET', 'sz_lb', k, JSON.stringify(r)]); } }
      else fs.writeFileSync(LBFILE, JSON.stringify(Object.fromEntries(LB)));
    } catch (e) { console.log('Sauvegarde du classement impossible :', e.message); }
  }, 4000);
}
function trimLB() {
  if (LB.size <= MAXLB) return;
  const s = [...LB.entries()].sort((a, b) => a[1].lv - b[1].lv || a[1].w - b[1].w);
  for (let i = 0; i < s.length - MAXLB; i++) LB.delete(s[i][0]);
}
function topRows(c) {
  return [...LB.entries()]
    .sort((a, b) => b[1].lv - a[1].lv || b[1].w - a[1].w || a[1].t - b[1].t)
    .slice(0, 50).map(([id, r]) => ({ n: r.n, lv: r.lv, w: r.w, ch: r.ch, me: id === c.id }));
}

/* ---------- HTTP : sert le jeu ---------- */
const server = http.createServer((req, res) => {
  const u = req.url.split('?')[0];
  if (u === '/healthz') { res.writeHead(200, { 'Content-Type': 'text/plain' }); return res.end('ok'); }
  if (u === '/' || u === '/index.html') {
    return fs.readFile(path.join(DIR, 'index.html'), (e, d) => {
      if (e) { res.writeHead(500); return res.end('index.html manquant'); }
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8', 'Cache-Control': 'no-cache' });
      res.end(d);
    });
  }
  res.writeHead(404); res.end('Not found');
});

/* ---------- WebSocket minimal (RFC 6455) ---------- */
function sendFrame(s, op, payload) {
  if (s.destroyed || !s.writable) return;
  const len = payload.length; let h;
  if (len < 126) h = Buffer.from([128 | op, len]);
  else if (len < 65536) { h = Buffer.alloc(4); h[0] = 128 | op; h[1] = 126; h.writeUInt16BE(len, 2); }
  else { h = Buffer.alloc(10); h[0] = 128 | op; h[1] = 127; h.writeUInt32BE(0, 2); h.writeUInt32BE(len, 6); }
  s.write(Buffer.concat([h, payload]));
}
const send = (c, o) => sendFrame(c.s, 1, Buffer.from(JSON.stringify(o)));
function closeC(c, code) {
  try { const b = Buffer.alloc(2); b.writeUInt16BE(code, 0); sendFrame(c.s, 8, b); c.s.end(); } catch (e) {}
  drop(c);
}

/* ---------- état du jeu ---------- */
const clients = new Set(), queue = [], hosts = new Map();
function newClient(s) {
  const c = { s, id: '', name: '', lv: 0, ch: 0, partner: null, code: '', alive: true, tok: 120, tl: Date.now(), over: 0 };
  clients.add(c); return c;
}
function leave(c) {
  const qi = queue.indexOf(c); if (qi >= 0) queue.splice(qi, 1);
  if (c.code) { hosts.delete(c.code); c.code = ''; }
  if (c.partner) { const p = c.partner; c.partner = null; if (p.partner === c) { p.partner = null; send(p, { t: 'left' }); } }
}
function drop(c) {
  if (!clients.has(c)) return;
  clients.delete(c); leave(c);
  try { c.s.destroy(); } catch (e) {}
}
function match(a, b, lv, priv) {
  a.partner = b; b.partner = a;
  send(a, { t: 'match', team: 0, lv, ch: b.ch, name: b.name, priv });
  send(b, { t: 'match', team: 1, lv, ch: a.ch, name: a.name, priv });
}
const CODE_AL = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
function genCode() { let s = ''; for (let i = 0; i < 5; i++) s += CODE_AL[crypto.randomInt(CODE_AL.length)]; return s; }

function onMsg(c, txt) {
  const now = Date.now();
  c.tok = Math.min(120, c.tok + (now - c.tl) * 0.09); c.tl = now;
  if (c.tok < 1) { if (++c.over > 300) closeC(c, 1008); return; }
  c.tok--;
  let m; try { m = JSON.parse(txt); } catch (e) { return; }
  if (!m || typeof m.t !== 'string') return;
  switch (m.t) {
    case 'hello':
      c.id = /^[a-z0-9]{8,40}$/i.test(String(m.id)) ? String(m.id) : '';
      c.name = cleanName(m.name); c.ch = cl(m.ch, 0, 4);
      send(c, { t: 'hi', online: clients.size }); break;
    case 'queue': {
      leave(c); c.lv = cl(m.lv, 0, 78); c.ch = cl(m.ch, 0, 4);
      let o = null;
      while (queue.length) { const x = queue.shift(); if (clients.has(x) && x !== c) { o = x; break; } }
      if (o) match(o, c, Math.min(o.lv, c.lv), false); else queue.push(c);
      break;
    }
    case 'host': {
      leave(c); if (hosts.size >= MAXHOSTS) { send(c, { t: 'err', m: 'Serveur plein, réessaie plus tard.' }); break; }
      c.lv = cl(m.lv, 0, 78); c.ch = cl(m.ch, 0, 4);
      let code; do { code = genCode(); } while (hosts.has(code));
      c.code = code; hosts.set(code, c); send(c, { t: 'hosted', code }); break;
    }
    case 'join': {
      leave(c); c.ch = cl(m.ch, 0, 4);
      const code = String(m.code || '').toUpperCase().slice(0, 8), h = hosts.get(code);
      if (!h || h === c) { send(c, { t: 'err', m: 'Code introuvable ou partie déjà commencée.' }); break; }
      hosts.delete(code); h.code = ''; match(h, c, h.lv, true); break;
    }
    case 'cancel': case 'leave': leave(c); break;
    case 'g': {
      if (!c.partner || !m.d || typeof m.d !== 'object') break;
      const d = {}; for (const k of GK) { const v = m.d[k]; if (typeof v === 'number' && isFinite(v)) d[k] = v; }
      send(c.partner, { t: 'g', d }); break;
    }
    case 'lb': send(c, { t: 'lb', rows: topRows(c) }); break;
    case 'score': {
      if (!c.id) break;
      const n = cleanName(m.name); if (!n) break;
      LB.set(c.id, { n, lv: cl(m.lv, 0, 79), w: cl(m.w, 0, 99999), ch: cl(m.ch, 0, 4), t: Date.now() });
      trimLB(); persist(c.id); send(c, { t: 'lb', rows: topRows(c) }); break;
    }
  }
}

server.on('upgrade', (req, socket) => {
  const key = req.headers['sec-websocket-key'];
  if (!key || String(req.headers.upgrade || '').toLowerCase() !== 'websocket' || clients.size >= MAXCLIENTS) { socket.destroy(); return; }
  const acc = crypto.createHash('sha1').update(key + GUID).digest('base64');
  socket.write('HTTP/1.1 101 Switching Protocols\r\nUpgrade: websocket\r\nConnection: Upgrade\r\nSec-WebSocket-Accept: ' + acc + '\r\n\r\n');
  socket.setNoDelay(true);
  const c = newClient(socket);
  let buf = Buffer.alloc(0), frag = null, fragLen = 0;
  socket.on('data', d => {
    buf = Buffer.concat([buf, d]);
    for (;;) {
      if (buf.length < 2) break;
      const b0 = buf[0], b1 = buf[1], fin = !!(b0 & 128), op = b0 & 15, masked = !!(b1 & 128);
      let len = b1 & 127, off = 2;
      if (len === 126) { if (buf.length < 4) break; len = buf.readUInt16BE(2); off = 4; }
      else if (len === 127) { if (buf.length < 10) break; if (buf.readUInt32BE(2)) { closeC(c, 1009); return; } len = buf.readUInt32BE(6); off = 10; }
      if (len > MAXLEN) { closeC(c, 1009); return; }
      const need = off + (masked ? 4 : 0) + len;
      if (buf.length < need) break;
      let payload = buf.slice(off + (masked ? 4 : 0), need);
      if (masked) { const mk = buf.slice(off, off + 4); payload = Buffer.from(payload); for (let i = 0; i < payload.length; i++) payload[i] ^= mk[i & 3]; }
      buf = buf.slice(need);
      if (op === 8) { closeC(c, 1000); return; }
      if (op === 9) { sendFrame(socket, 10, payload); continue; }
      if (op === 10) { c.alive = true; continue; }
      if (op === 1 || op === 0) {
        if (op === 1) { frag = [payload]; fragLen = payload.length; }
        else if (frag) { frag.push(payload); fragLen += payload.length; if (fragLen > MAXLEN) { closeC(c, 1009); return; } }
        if (fin && frag) { const txt = Buffer.concat(frag).toString('utf8'); frag = null; onMsg(c, txt); }
      }
    }
  });
  socket.on('close', () => drop(c));
  socket.on('error', () => drop(c));
});

setInterval(() => {
  for (const c of [...clients]) {
    if (!c.alive) { drop(c); continue; }
    c.alive = false; sendFrame(c.s, 9, Buffer.alloc(0));
  }
}, 30000);

loadLB().then(() => server.listen(PORT, () => console.log('Strike Zone prêt sur le port ' + PORT)));
