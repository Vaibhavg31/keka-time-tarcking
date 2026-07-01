// ==UserScript==
// @name         Keka Kinetic Time — Physics Ball Tracker
// @namespace    http://tampermonkey.net/
// @version      3.1.0
// @description  Premium desktop companion — builds in-tab, pops to floating window
// @author       KineticTime
// @match        https://ezeetechnosys.keka.com/*
// @match        https://*.keka.com/*
// @exclude      https://*.keka.com/login*
// @icon         https://www.google.com/s2/favicons?sz=64&domain=keka.com
// @grant        none
// @run-at       document-end
// @noframes
// ==/UserScript==

(function () {
  'use strict';

  if (window.__KINETIC_TIME__) return;
  window.__KINETIC_TIME__ = true;

  const EIGHT_HOURS = 8 * 60;
  const HALF_HOURS  = 4 * 60;
  const TICK_MS     = 4000;
  const STORAGE_KEY = 'kt_pos';
  const MODE_KEY    = 'kt_mode';
  const HALF_KEY    = 'kt_half';
  const POPUP_NAME  = 'kinetic-time-tracker';
  const LOG_SEL     = '.modal-body form div[formarrayname="logs"]';
  const ROW_SEL     = '.ng-untouched.ng-pristine.ng-valid';
  const START_SEL   = '.d-flex.align-items-center .w-120.mr-20 .text-small';
  const END_SEL     = '.d-flex.align-items-center .w-120:not(.mr-20) .text-small';
  const MAX_PARTS   = 50;
  const MAX_RIPPLES = 5;
  const W = 450;
  const H = 320;

  // ── DATA (unchanged) ────────────────────────────────────

  function parseTime(s) {
      if (!s || s === 'MISSING') return null;
      const p = s.trim().toLowerCase().split(/\s+/);
      const [hms, ap] = p.length === 2 ? p : [p[0], p[1]];
      let [h, m] = hms.split(':').map(Number);
      if (ap === 'pm' && h !== 12) h += 12;
      if (ap === 'am' && h === 12) h = 0;
      return isNaN(h) ? null : { hours: h, minutes: m };
  }

  function durMin(a, b) {
      const s = parseTime(a);
      const e = b === 'MISSING'
          ? { hours: new Date().getHours(), minutes: new Date().getMinutes() }
          : parseTime(b);
      if (!s || !e) return 0;
      let d = (e.hours - s.hours) * 60 + (e.minutes - s.minutes);
      if (d < 0) d += 1440;
      return d > 720 ? 0 : d;
  }

  function fmtHM(m) {
      const h = Math.floor(m / 60), r = m % 60;
      return h === 0 ? `${r}m` : r === 0 ? `${h}h` : `${h}h ${r}m`;
  }

  function clockAt(baseMin, add) {
      const d = new Date();
      d.setHours(0, 0, 0, 0);
      d.setMinutes(baseMin + add);
      return d.toLocaleTimeString('en-IN', { hour: '2-digit', minute: '2-digit', hour12: true });
  }

  function processRows(container) {
      if (!container) return null;
      let total = 0, brk = 0, first = null, prev = null;
      container.querySelectorAll(ROW_SEL).forEach((row, i) => {
          const st = row.querySelector(START_SEL)?.textContent.trim();
          const en = row.querySelector(END_SEL)?.textContent.trim() || 'MISSING';
          if (i === 0) first = st;
          else if (prev) { const g = durMin(prev, st); if (g > 0) brk += g; }
          total += durMin(st, en);
          prev = en === 'MISSING' ? null : en;
      });
      return {
          totalMinutes: total, breakMinutes: brk, firstStart: first,
          totalLabel: `${Math.floor(total / 60)}h ${total % 60}m`
      };
  }

  const TIME_RE  = /\b(\d{1,2}:\d{2}\s*(?:am|pm))\b/gi;
  const RANGE_RE = /\d{1,2}:\d{2}\s*(?:am|pm)\s*[-–]\s*\d{1,2}:\d{2}\s*(?:am|pm)/gi;

  function fallbackPairs() {
      let label = null;
      for (const el of document.querySelectorAll('*')) {
          if (el.children.length > 8) continue;
          if (/^biometric\s+logs$/i.test((el.innerText || '').trim())) { label = el; break; }
      }
      if (!label) return [];
      let c = label;
      for (let d = 0; d < 10; d++) {
          const p = c.parentElement;
          if (!p) break;
          if (((p.innerText || '').replace(RANGE_RE, '').match(TIME_RE) || []).length >= 2) { c = p; break; }
          c = p;
      }
      const tok = [...(c.innerText || '').replace(RANGE_RE, '').matchAll(TIME_RE)].map(m => m[1].trim());
      const pairs = [];
      for (let i = 0; i < tok.length; i += 2) pairs.push({ s: tok[i], e: tok[i + 1] || 'MISSING' });
      return pairs;
  }

  function pairsToSnap(pairs) {
      let total = 0, brk = 0, first = null, prev = null;
      pairs.forEach(({ s, e }, i) => {
          if (!parseTime(s)) return;
          if (i === 0) first = s;
          else if (prev) { const g = durMin(prev, s); if (g > 0) brk += g; }
          total += durMin(s, e);
          prev = e === 'MISSING' ? null : e;
      });
      return { totalMinutes: total, breakMinutes: brk, firstStart: first,
          totalLabel: `${Math.floor(total / 60)}h ${total % 60}m` };
  }

  function getSnap() {
      const c = document.querySelector(LOG_SEL);
      if (c) { const r = processRows(c); if (r) { window.__KT_SNAP__ = r; return r; } }
      if (window.__KT_SNAP__) return window.__KT_SNAP__;
      const p = fallbackPairs();
      return p.length ? pairsToSnap(p) : { totalMinutes: 0, breakMinutes: 0, firstStart: null, totalLabel: '0h 0m' };
  }

  // ── PREMIUM THEME ─────────────────────────────────────

  const THEME = {
      navy: '#060a14', midnight: '#0b1120', deep: '#111827',
      purple: '#7c3aed', cyan: '#22d3ee', green: '#34d399',
      orange: '#fb923c', gold: '#fbbf24', core: '#f59e0b',
      bloom: 'rgba(124,58,237,0.35)', ring: '#34d399', ring2: '#22d3ee',
  };

  const PALETTES = [
      { bg: ['#060a14', '#0f172a', '#1e1b4b'], a: '#22d3ee', b: '#f59e0b', ring: '#34d399', accent: '#a78bfa' },
      { bg: ['#070b14', '#0c1929', '#172554'], a: '#38bdf8', b: '#fb923c', ring: '#4ade80', accent: '#818cf8' },
      { bg: ['#0a0a12', '#12121f', '#1a1035'], a: '#c084fc', b: '#fbbf24', ring: '#2dd4bf', accent: '#f472b6' },
  ];

  const HINTS = ['tap core to copy · trigger pulse', 'click cards to copy', 'drag to move', '⧉ pop out to window', 'Ctrl+Shift+T hide/show'];

  const pick = a => a[Math.floor(Math.random() * a.length)];
  const rand = (a, b) => a + Math.random() * (b - a);
  const lerp = (a, b, t) => a + (b - a) * t;
  const easeInOut = t => t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2;
  const easeOutCubic = t => 1 - Math.pow(1 - t, 3);
  const easeOutElastic = t => {
      if (t <= 0 || t >= 1) return t;
      return Math.pow(2, -10 * t) * Math.sin((t - 0.075) * (2 * Math.PI) / 0.3) + 1;
  };

  let shiftTargetMin = (() => {
      try { return localStorage.getItem(HALF_KEY) === '1' ? HALF_HOURS : EIGHT_HOURS; }
      catch { return EIGHT_HOURS; }
  })();

  function getShiftMin() { return shiftTargetMin; }

  function setShiftMode(halfDay) {
      shiftTargetMin = halfDay ? HALF_HOURS : EIGHT_HOURS;
      try { localStorage.setItem(HALF_KEY, halfDay ? '1' : '0'); } catch { /* */ }
      const root = refs.root;
      if (root) {
          root.querySelectorAll('.kt-shift-opt').forEach(btn => {
              btn.classList.toggle('active', (btn.dataset.h === '4') === halfDay);
          });
      }
      pushData();
  }

  // ── ENGINE ────────────────────────────────────────────

  class Ball {
      constructor(x, y, r, color, role) {
          this.x = x; this.y = y; this.r = r;
          this.vx = 0; this.vy = 0;
          this.color = color; this.role = role;
          this.mass = r * r;
          this.alpha = role === 'striker' ? 0 : 1;
          this.fadeOut = false; this.label = '';
      }
  }

  const E = {
      W, H,
      balls: [], particles: [], ripples: [],
      dust: [], nebulae: [], stars: [], orbitDots: [],
      main: null, satellites: [],
      phase: 'idle', palette: PALETTES[0], variant: 0,
      impactT: 0, revealT: 0, incomingT: 0,
      timeStr: '0:00', timeSub: '', timeVisible: false,
      ringTarget: 0, ringCurrent: 0, ringSecondary: 0,
      displayMinutes: 0, targetMinutes: 0, displayPct: 0, targetPct: 0,
      breathe: 0, sweep: 0, shake: 0, nextStrike: 0,
      animId: null, running: false, paused: false,
      dataHash: '', lastHint: 0, gravity: 0, strikeLock: false,
      completionBurst: 0, shootingStar: null, mouseX: 0.5, mouseY: 0.5,
  };

  function initAmbience() {
      E.stars = Array.from({ length: 55 }, () => ({
          x: Math.random() * W, y: Math.random() * H,
          r: rand(0.3, 1.6), phase: rand(0, Math.PI * 2),
          speed: rand(0.008, 0.025), depth: rand(0.3, 1),
      }));
      E.dust = Array.from({ length: 35 }, () => ({
          x: Math.random() * W, y: Math.random() * H,
          r: rand(0.5, 2), depth: rand(0.2, 1),
          vx: rand(-0.15, 0.15), vy: rand(-0.08, 0.08),
          a: rand(0.15, 0.5),
      }));
      E.nebulae = [
          { x: W * 0.25, y: H * 0.3, r: 120, c: '124,58,237', phase: 0 },
          { x: W * 0.75, y: H * 0.65, r: 100, c: '34,211,238', phase: 1.2 },
          { x: W * 0.5, y: H * 0.85, r: 90, c: '52,211,153', phase: 2.4 },
      ];
      E.orbitDots = Array.from({ length: 6 }, (_, i) => ({
          angle: (i / 6) * Math.PI * 2, speed: rand(0.008, 0.014), r: rand(2, 3.5),
      }));
  }

  function addParticle(x, y, vx, vy, life, color, size) {
      if (E.particles.length >= MAX_PARTS) E.particles.shift();
      E.particles.push({ x, y, vx, vy, life, max: life, color, size });
  }

  function addRipple(x, y, maxR, color) {
      if (E.ripples.length >= MAX_RIPPLES) E.ripples.shift();
      E.ripples.push({ x, y, r: 8, max: maxR, life: 1, color });
  }

  function removeStrikers() { E.balls = E.balls.filter(b => b.role !== 'striker'); }

  function spawnStrike(soft) {
      if (E.strikeLock) return;
      E.strikeLock = true;
      setTimeout(() => { E.strikeLock = false; }, 1400);

      removeStrikers();
      if (!soft) E.particles = [];
      E.ripples = [];
      if (!soft) E.palette = pick(PALETTES);
      E.variant = Math.floor(Math.random() * 5);
      E.phase = 'incoming';
      E.incomingT = 0;
      E.gravity = E.variant === 2 ? 0.06 : 0;

      const cx = W / 2, cy = H / 2 + 4;
      if (!E.main) {
          E.main = new Ball(cx, cy, 56, E.palette.b, 'main');
          E.balls = [E.main];
      } else {
          E.main.x = cx; E.main.y = cy;
          E.main.vx = E.main.vy = 0;
          E.main.r = 56;
          E.main.color = E.palette.b;
          E.main.alpha = 1;
          E.balls = [E.main, ...E.satellites];
      }

      const side = Math.floor(Math.random() * 4);
      const margin = 50;
      let sx, sy;
      if (side === 0)      { sx = -margin; sy = rand(60, H - 60); }
      else if (side === 1) { sx = W + margin; sy = rand(60, H - 60); }
      else if (side === 2) { sx = rand(80, W - 80); sy = -margin; }
      else                   { sx = rand(80, W - 80); sy = H + margin; }

      const dx = cx - sx, dy = cy - sy, len = Math.hypot(dx, dy) || 1;
      const speed = E.variant === 3 ? rand(4.5, 6.5) : E.variant === 4 ? rand(9, 12) : rand(7, 9);
      const striker = new Ball(sx, sy, rand(16, 22), E.palette.a, 'striker');
      striker.vx = (dx / len) * speed + rand(-0.3, 0.3);
      striker.vy = (dy / len) * speed + rand(-0.3, 0.3);
      E.balls.push(striker);

      if (E.variant === 1) {
          const s2 = new Ball(W - sx, H - sy, rand(12, 16), E.palette.ring, 'striker');
          const dx2 = cx - s2.x, dy2 = cy - s2.y, l2 = Math.hypot(dx2, dy2) || 1;
          s2.vx = (dx2 / l2) * speed * 0.8; s2.vy = (dy2 / l2) * speed * 0.8;
          E.balls.push(s2);
      }
      applyBg();
  }

  function onImpact(x, y) {
      if (E.phase !== 'incoming') return;
      E.phase = 'impact';
      E.impactT = 0;
      E.shake = 0.6;
      const n = E.variant === 1 ? 24 : 18;
      for (let i = 0; i < n; i++) {
          const ang = rand(0, Math.PI * 2), spd = rand(1, 4);
          addParticle(x, y, Math.cos(ang) * spd, Math.sin(ang) * spd,
              rand(16, 30), i % 2 ? E.palette.a : E.palette.ring, rand(1.2, 2.8));
      }
      addRipple(x, y, 100, 'rgba(52,211,153,0.5)');
      addRipple(x, y, 140, 'rgba(255,255,255,0.15)');
      E.balls.filter(b => b.role === 'striker').forEach(s => { s.fadeOut = true; s.vx *= 0.35; s.vy *= 0.35; });
  }

  function collide(b1, b2) {
      const dx = b2.x - b1.x, dy = b2.y - b1.y;
      const dist = Math.hypot(dx, dy);
      if (dist >= b1.r + b2.r - 1 || dist < 0.001) return false;
      const nx = dx / dist, ny = dy / dist;
      const overlap = (b1.r + b2.r - dist) * 0.5;
      const tot = b1.mass + b2.mass;
      b1.x -= nx * overlap * (b2.mass / tot); b1.y -= ny * overlap * (b2.mass / tot);
      b2.x += nx * overlap * (b1.mass / tot); b2.y += ny * overlap * (b1.mass / tot);
      const dvn = (b1.vx - b2.vx) * nx + (b1.vy - b2.vy) * ny;
      if (dvn > 0) {
          const imp = (1.6 * dvn) / tot;
          b1.vx -= imp * b2.mass * nx; b1.vy -= imp * b2.mass * ny;
          b2.vx += imp * b1.mass * nx; b2.vy += imp * b1.mass * ny;
      }
      return true;
  }

  function updateSatellites(leftMin, breakMin, isOT) {
      const cx = W / 2, cy = H / 2 + 4;
      if (E.satellites.length < 2) {
          E.satellites = [
              new Ball(cx + 85, cy - 35, 13, E.palette.accent, 'satellite'),
              new Ball(cx - 85, cy + 25, 11, E.palette.a, 'satellite'),
          ];
          E.satellites[0].angle = 0; E.satellites[1].angle = Math.PI;
          E.satellites[0].orbit = 88; E.satellites[1].orbit = 72;
          E.satellites[0].speed = 0.009; E.satellites[1].speed = -0.011;
          E.balls = [E.main, ...E.satellites].filter(Boolean);
      }
      E.satellites[0].label = isOT ? `+${fmtHM(leftMin)}` : fmtHM(leftMin);
      E.satellites[1].label = fmtHM(breakMin);
      E.satellites[0].color = E.palette.accent;
      E.satellites[1].color = E.palette.a;
  }

  function maybeShootingStar() {
      if (E.shootingStar || Math.random() > 0.002) return;
      E.shootingStar = {
          x: rand(0, W * 0.6), y: rand(0, H * 0.4),
          vx: rand(4, 7), vy: rand(1.5, 3), life: 1,
      };
  }

  function physics() {
      const { main } = E;
      if (!main) return;

      const dt = E.paused ? 0.3 : 1;
      E.breathe += 0.018 * dt;
      E.sweep += 0.012 * dt;
      E.ringCurrent = lerp(E.ringCurrent, E.ringTarget, 0.035 * dt);
      E.ringSecondary = lerp(E.ringSecondary, E.ringTarget * 0.85, 0.02 * dt);
      E.displayMinutes = lerp(E.displayMinutes, E.targetMinutes, 0.06 * dt);
      E.displayPct = lerp(E.displayPct, E.targetPct, 0.045 * dt);
      if (E.completionBurst > 0) E.completionBurst = Math.max(0, E.completionBurst - 0.015 * dt);

      E.balls.forEach(b => {
          if (b.role === 'striker') {
              b.alpha = Math.min(1, b.alpha + 0.05);
              if (E.gravity) b.vy += E.gravity;
              if (b.fadeOut) { b.alpha = Math.max(0, b.alpha - 0.035); b.vx *= 0.9; b.vy *= 0.9; }
          }
          if (b.role === 'satellite' && main) {
              b.angle += b.speed * dt;
              b.x = main.x + Math.cos(b.angle) * b.orbit;
              b.y = main.y + Math.sin(b.angle) * b.orbit * 0.42;
          }
          if (b.role === 'main' && (E.phase === 'rest' || E.phase === 'reveal')) {
              const t = Date.now() * 0.001;
              const br = 1 + Math.sin(E.breathe) * 0.012;
              main.r = 56 * br;
              main.x = W / 2 + Math.sin(t * 0.7) * 1.5;
              main.y = H / 2 + 4 + Math.cos(t * 0.85) * 1.2;
          }
          b.x += b.vx * dt; b.y += b.vy * dt;
          b.vx *= 0.997; b.vy *= 0.997;
          if (b.role === 'striker') {
              [['x', 'vx', 0, W], ['y', 'vy', 0, H]].forEach(([pos, vel, min, max]) => {
                  if (b[pos] - b.r < min) { b[pos] = b.r; b[vel] = Math.abs(b[vel]) * 0.55; }
                  if (b[pos] + b.r > max) { b[pos] = max - b.r; b[vel] = -Math.abs(b[vel]) * 0.55; }
              });
          }
      });

      if (E.phase === 'incoming') {
          E.incomingT++;
          let hit = false;
          E.balls.filter(b => b.role === 'striker' && b.alpha > 0.5).forEach(s => {
              if (collide(s, main)) { hit = true; onImpact((s.x + main.x) / 2, (s.y + main.y) / 2); }
          });
          if (!hit && E.incomingT > 160) onImpact(main.x, main.y);
      }

      if (E.phase === 'impact') {
          E.impactT += 0.06 * dt;
          E.shake *= 0.8;
          if (E.impactT >= 1) { E.phase = 'reveal'; E.revealT = 0; E.timeVisible = true; removeStrikers(); }
      }

      if (E.phase === 'reveal') {
          E.revealT = Math.min(1, E.revealT + 0.032 * dt);
          if (E.revealT >= 1) { E.phase = 'rest'; E.nextStrike = Date.now() + rand(20000, 32000); }
      }

      if (E.phase === 'rest' && Date.now() > E.nextStrike) spawnStrike(true);

      E.balls = E.balls.filter(b => !(b.role === 'striker' && b.fadeOut && b.alpha <= 0));

      E.dust.forEach(d => {
          d.x += d.vx * d.depth * dt; d.y += d.vy * d.depth * dt;
          if (d.x < 0) d.x = W; if (d.x > W) d.x = 0;
          if (d.y < 0) d.y = H; if (d.y > H) d.y = 0;
      });

      if (!E.paused) maybeShootingStar();
      if (E.shootingStar) {
          const s = E.shootingStar;
          s.x += s.vx * dt; s.y += s.vy * dt; s.life -= 0.02 * dt;
          if (s.life <= 0) E.shootingStar = null;
      }

      E.particles = E.particles.filter(p => {
          p.x += p.vx * dt; p.y += p.vy * dt; p.vy += 0.04; p.life--;
          return p.life > 0;
      });
      E.ripples = E.ripples.filter(r => {
          r.r = lerp(r.r, r.max, 0.08 * dt); r.life -= 0.028 * dt;
          return r.life > 0;
      });
  }

  // ── RENDER ────────────────────────────────────────────

  function hexRgb(hex) {
      const n = parseInt(hex.replace('#', ''), 16);
      return { r: (n >> 16) & 255, g: (n >> 8) & 255, b: n & 255 };
  }
  function tint(hex, amt) {
      const { r, g, b } = hexRgb(hex);
      return `rgb(${Math.min(255, r + amt)},${Math.min(255, g + amt)},${Math.min(255, b + amt)})`;
  }
  function shade(hex, amt) {
      const { r, g, b } = hexRgb(hex);
      return `rgb(${Math.max(0, r - amt)},${Math.max(0, g - amt)},${Math.max(0, b - amt)})`;
  }

  function drawBackground(ctx, now) {
      const t = now * 0.001;
      const cols = E.palette.bg;
      const bg = ctx.createLinearGradient(0, 0, W, H);
      bg.addColorStop(0, cols[0]);
      bg.addColorStop(0.5, cols[1] || cols[0]);
      if (cols[2]) bg.addColorStop(1, cols[2]);
      ctx.fillStyle = bg;
      ctx.fillRect(0, 0, W, H);

      E.nebulae.forEach(n => {
          const ox = Math.sin(t * 0.15 + n.phase) * 18;
          const oy = Math.cos(t * 0.12 + n.phase) * 12;
          const g = ctx.createRadialGradient(n.x + ox, n.y + oy, 0, n.x + ox, n.y + oy, n.r);
          g.addColorStop(0, `rgba(${n.c},0.14)`);
          g.addColorStop(0.6, `rgba(${n.c},0.04)`);
          g.addColorStop(1, 'transparent');
          ctx.fillStyle = g;
          ctx.fillRect(0, 0, W, H);
      });

      const parX = (E.mouseX - 0.5) * 12;
      const parY = (E.mouseY - 0.5) * 8;

      E.dust.forEach(d => {
          ctx.beginPath();
          ctx.arc(d.x + parX * d.depth, d.y + parY * d.depth, d.r * d.depth, 0, Math.PI * 2);
          ctx.fillStyle = `rgba(200,220,255,${d.a * d.depth * 0.35})`;
          ctx.fill();
      });

      E.stars.forEach(s => {
          const a = 0.2 + 0.5 * Math.abs(Math.sin(now * s.speed + s.phase)) * s.depth;
          ctx.beginPath();
          ctx.arc(s.x + parX * s.depth * 0.5, s.y + parY * s.depth * 0.5, s.r, 0, Math.PI * 2);
          ctx.fillStyle = `rgba(255,255,255,${a.toFixed(2)})`;
          ctx.fill();
      });

      if (E.shootingStar) {
          const s = E.shootingStar;
          const g = ctx.createLinearGradient(s.x, s.y, s.x - 40, s.y - 18);
          g.addColorStop(0, `rgba(255,255,255,${s.life * 0.9})`);
          g.addColorStop(1, 'transparent');
          ctx.strokeStyle = g;
          ctx.lineWidth = 2;
          ctx.beginPath();
          ctx.moveTo(s.x, s.y);
          ctx.lineTo(s.x - 45, s.y - 20);
          ctx.stroke();
      }
  }

  function drawRings(ctx, mx, my, r) {
      const prog = E.ringCurrent;
      const br = 1 + Math.sin(E.breathe) * 0.04;

      ctx.save();
      ctx.translate(mx, my);

      // outer ambient glow
      const halo = ctx.createRadialGradient(0, 0, r * 0.5, 0, 0, r + 38);
      halo.addColorStop(0, 'rgba(52,211,153,0.12)');
      halo.addColorStop(0.6, 'rgba(124,58,237,0.06)');
      halo.addColorStop(1, 'transparent');
      ctx.fillStyle = halo;
      ctx.beginPath();
      ctx.arc(0, 0, r + 38, 0, Math.PI * 2);
      ctx.fill();

      // track ring
      ctx.beginPath();
      ctx.arc(0, 0, (r + 16) * br, 0, Math.PI * 2);
      ctx.strokeStyle = 'rgba(255,255,255,0.06)';
      ctx.lineWidth = 6;
      ctx.stroke();

      // secondary thin ring (counter)
      ctx.save();
      ctx.rotate(-E.sweep * 0.7);
      ctx.setLineDash([4, 10]);
      ctx.beginPath();
      ctx.arc(0, 0, (r + 24) * br, 0, Math.PI * 2);
      ctx.strokeStyle = 'rgba(34,211,238,0.2)';
      ctx.lineWidth = 1.5;
      ctx.stroke();
      ctx.setLineDash([]);
      ctx.restore();

      // main progress ring with glow
      const start = -Math.PI / 2;
      const end = start + Math.PI * 2 * prog;
      ctx.shadowColor = THEME.green;
      ctx.shadowBlur = 14;
      const ringGrad = ctx.createLinearGradient(-r, 0, r, 0);
      ringGrad.addColorStop(0, THEME.cyan);
      ringGrad.addColorStop(0.5, THEME.green);
      ringGrad.addColorStop(1, THEME.gold);
      ctx.beginPath();
      ctx.arc(0, 0, (r + 16) * br, start, end);
      ctx.strokeStyle = ringGrad;
      ctx.lineWidth = 6;
      ctx.lineCap = 'round';
      ctx.stroke();
      ctx.shadowBlur = 0;

      // highlight dot on progress
      if (prog > 0.02) {
          const hx = Math.cos(end) * (r + 16) * br;
          const hy = Math.sin(end) * (r + 16) * br;
          ctx.beginPath();
          ctx.arc(hx, hy, 4, 0, Math.PI * 2);
          ctx.fillStyle = '#fff';
          ctx.shadowColor = THEME.green;
          ctx.shadowBlur = 10;
          ctx.fill();
          ctx.shadowBlur = 0;
      }

      // light sweep
      ctx.save();
      ctx.rotate(E.sweep);
      const sw = ctx.createLinearGradient(0, -r - 30, 0, r + 30);
      sw.addColorStop(0, 'transparent');
      sw.addColorStop(0.48, 'transparent');
      sw.addColorStop(0.5, 'rgba(255,255,255,0.08)');
      sw.addColorStop(0.52, 'transparent');
      sw.addColorStop(1, 'transparent');
      ctx.fillStyle = sw;
      ctx.fillRect(-r - 30, -r - 30, (r + 30) * 2, (r + 30) * 2);
      ctx.restore();

      if (E.completionBurst > 0) {
          ctx.beginPath();
          ctx.arc(0, 0, (r + 20) * (1 + E.completionBurst * 0.15), 0, Math.PI * 2);
          ctx.strokeStyle = `rgba(251,191,36,${E.completionBurst * 0.6})`;
          ctx.lineWidth = 3;
          ctx.stroke();
      }

      ctx.restore();
  }

  function drawEnergyCore(ctx, b) {
      const br = 1 + Math.sin(E.breathe) * 0.015;
      const R = b.r * br;

      drawRings(ctx, b.x, b.y, R);

      // floor shadow
      ctx.beginPath();
      ctx.ellipse(b.x, b.y + R + 8, R * 0.9, R * 0.2, 0, 0, Math.PI * 2);
      ctx.fillStyle = 'rgba(0,0,0,0.45)';
      ctx.fill();

      // outer glass shell
      const shell = ctx.createRadialGradient(b.x - R * 0.3, b.y - R * 0.35, R * 0.05, b.x, b.y, R);
      shell.addColorStop(0, 'rgba(255,220,150,0.95)');
      shell.addColorStop(0.35, THEME.core);
      shell.addColorStop(0.7, '#ea580c');
      shell.addColorStop(1, 'rgba(120,40,10,0.9)');
      ctx.beginPath();
      ctx.arc(b.x, b.y, R, 0, Math.PI * 2);
      ctx.fillStyle = shell;
      ctx.shadowColor = THEME.orange;
      ctx.shadowBlur = 28;
      ctx.fill();
      ctx.shadowBlur = 0;

      // inner light
      const inner = ctx.createRadialGradient(b.x - R * 0.15, b.y - R * 0.2, 0, b.x, b.y, R * 0.65);
      inner.addColorStop(0, 'rgba(255,255,255,0.55)');
      inner.addColorStop(0.4, 'rgba(255,200,80,0.25)');
      inner.addColorStop(1, 'transparent');
      ctx.beginPath();
      ctx.arc(b.x, b.y, R * 0.65, 0, Math.PI * 2);
      ctx.fillStyle = inner;
      ctx.fill();

      // specular
      ctx.beginPath();
      ctx.arc(b.x - R * 0.28, b.y - R * 0.32, R * 0.18, 0, Math.PI * 2);
      ctx.fillStyle = 'rgba(255,255,255,0.7)';
      ctx.fill();

      // orbit micro-dots
      E.orbitDots.forEach(d => {
          d.angle += d.speed;
          const ox = b.x + Math.cos(d.angle + E.sweep) * (R + 28);
          const oy = b.y + Math.sin(d.angle + E.sweep) * (R + 28) * 0.5;
          ctx.beginPath();
          ctx.arc(ox, oy, d.r, 0, Math.PI * 2);
          ctx.fillStyle = `rgba(34,211,238,${0.4 + Math.sin(d.angle * 3) * 0.2})`;
          ctx.fill();
      });
  }

  function drawSatellite(ctx, b) {
      if (!b || b.alpha <= 0.01) return;
      ctx.save();
      ctx.globalAlpha = b.alpha;
      const g = ctx.createRadialGradient(b.x - b.r * 0.3, b.y - b.r * 0.3, 0, b.x, b.y, b.r);
      g.addColorStop(0, tint(b.color, 50));
      g.addColorStop(1, shade(b.color, 20));
      ctx.beginPath();
      ctx.arc(b.x, b.y, b.r, 0, Math.PI * 2);
      ctx.fillStyle = g;
      ctx.shadowColor = b.color;
      ctx.shadowBlur = 8;
      ctx.fill();
      ctx.shadowBlur = 0;
      if (b.label) {
          ctx.font = '600 9px Inter,system-ui,sans-serif';
          ctx.textAlign = 'center';
          ctx.textBaseline = 'middle';
          ctx.fillStyle = '#fff';
          ctx.fillText(b.label, b.x, b.y);
      }
      ctx.restore();
  }

  function drawStriker(ctx, b) {
      if (!b || b.alpha <= 0.01) return;
      ctx.save();
      ctx.globalAlpha = b.alpha;
      const g = ctx.createRadialGradient(b.x - b.r * 0.3, b.y - b.r * 0.3, 0, b.x, b.y, b.r);
      g.addColorStop(0, tint(b.color, 60));
      g.addColorStop(1, b.color);
      ctx.beginPath();
      ctx.arc(b.x, b.y, b.r, 0, Math.PI * 2);
      ctx.fillStyle = g;
      ctx.shadowColor = b.color;
      ctx.shadowBlur = 12;
      ctx.fill();
      ctx.restore();
  }

  function formatDisplayMinutes(m) {
      const h = Math.floor(m / 60);
      const min = Math.round(m % 60);
      return `${h}:${String(min).padStart(2, '0')}`;
  }

  function render(ctx) {
      const now = Date.now();
      ctx.clearRect(0, 0, W, H);
      drawBackground(ctx, now);

      E.ripples.forEach(r => {
          ctx.beginPath();
          ctx.arc(r.x, r.y, r.r, 0, Math.PI * 2);
          ctx.strokeStyle = r.color;
          ctx.globalAlpha = r.life * 0.5;
          ctx.lineWidth = 2;
          ctx.stroke();
      });
      ctx.globalAlpha = 1;

      E.balls.filter(b => b.role === 'striker').forEach(b => drawStriker(ctx, b));
      E.balls.filter(b => b.role === 'satellite').forEach(b => drawSatellite(ctx, b));

      if (E.main) drawEnergyCore(ctx, E.main);

      if (E.main && E.timeVisible) {
          const pop = E.phase === 'reveal' ? easeOutElastic(E.revealT) : 1;
          const alpha = E.phase === 'reveal' ? easeOutCubic(E.revealT) : 1;
          ctx.save();
          ctx.translate(E.main.x, E.main.y);
          ctx.scale(0.5 + pop * 0.5, 0.5 + pop * 0.5);
          ctx.globalAlpha = alpha;

          const disp = formatDisplayMinutes(E.displayMinutes);
          ctx.font = '700 32px Inter,system-ui,sans-serif';
          ctx.textAlign = 'center';
          ctx.textBaseline = 'middle';
          ctx.fillStyle = 'rgba(0,0,0,0.35)';
          ctx.fillText(disp, 1, -6);
          ctx.fillStyle = '#ffffff';
          ctx.shadowColor = 'rgba(255,255,255,0.5)';
          ctx.shadowBlur = 6;
          ctx.fillText(disp, 0, -7);
          ctx.shadowBlur = 0;

          ctx.font = '500 11px Inter,system-ui,sans-serif';
          ctx.fillStyle = 'rgba(255,255,255,0.85)';
          ctx.fillText(E.timeSub, 0, 18);
          ctx.restore();
      }

      E.particles.forEach(p => {
          ctx.globalAlpha = p.life / p.max;
          ctx.fillStyle = p.color;
          ctx.beginPath();
          ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2);
          ctx.fill();
      });
      ctx.globalAlpha = 1;

      const hint = E.phase === 'incoming' ? 'pulse incoming…' : (E.currentHint || '');
      if (hint && E.phase !== 'impact') {
          ctx.font = '400 10px Inter,system-ui,sans-serif';
          ctx.fillStyle = 'rgba(255,255,255,0.2)';
          ctx.textAlign = 'center';
          ctx.fillText(hint, W / 2, H - 14);
      }
  }

  function isUiVisible() {
      return detached ? !uiDoc.hidden : !document.hidden;
  }

  function frame() {
      if (!E.running) return;
      if (isUiVisible()) physics();
      const canvas = refs.canvas;
      if (canvas) {
          const ctx = canvas.getContext('2d');
          const sh = E.shake * 4;
          ctx.save();
          if (sh > 0.1) ctx.translate((Math.random() - 0.5) * sh, (Math.random() - 0.5) * sh);
          render(ctx);
          ctx.restore();
      }
      E.animId = requestAnimationFrame(frame);
  }

  function startLoop() {
      if (E.running) return;
      E.running = true;
      E.animId = requestAnimationFrame(frame);
  }

  function stopLoop() {
      E.running = false;
      if (E.animId) cancelAnimationFrame(E.animId);
  }

  // ── UI ────────────────────────────────────────────────

  let drag = null, inertia = null, refs = {};
  let uiWin = window, uiDoc = document, popupRef = null, detached = false, pendingPop = false, bridgeEl = null;

  const CSS_TEXT = `
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

#kineticTime {
--kt-glass: rgba(12,18,32,0.72);
--kt-border: rgba(255,255,255,0.12);
box-sizing: border-box;
position: fixed; top: 20px; right: 20px;
z-index: 2147483646;
width: ${W}px;
border-radius: 28px;
overflow: hidden;
font-family: 'Inter', system-ui, -apple-system, sans-serif;
background: var(--kt-glass);
backdrop-filter: blur(40px) saturate(1.4);
-webkit-backdrop-filter: blur(40px) saturate(1.4);
border: 1px solid var(--kt-border);
box-shadow:
  0 0 0 1px rgba(255,255,255,0.04) inset,
  0 32px 80px rgba(0,0,0,0.55),
  0 8px 32px rgba(124,58,237,0.12),
  0 0 60px rgba(34,211,238,0.06);
transform: translateY(0) scale(1);
opacity: 1;
transition: opacity 0.45s cubic-bezier(0.22,1,0.36,1),
            transform 0.45s cubic-bezier(0.22,1,0.36,1),
            filter 0.45s ease;
will-change: transform, opacity;
}
#kineticTime.kt-enter {
animation: ktEnter 0.85s cubic-bezier(0.22,1,0.36,1) forwards;
}
#kineticTime.kt-exit {
opacity: 0;
transform: translateY(16px) scale(0.94);
filter: blur(6px);
pointer-events: none;
}
@keyframes ktEnter {
from { opacity: 0; transform: translateY(28px) scale(0.9); filter: blur(8px); }
to   { opacity: 1; transform: none; filter: none; }
}

#ktBg { position: absolute; inset: 0; transition: background 1.4s ease; z-index: 0; opacity: 0.5; }
#ktInner { position: relative; z-index: 1; }
#ktGlassHighlight {
position: absolute; top: 0; left: 0; right: 0; height: 1px;
background: linear-gradient(90deg, transparent, rgba(255,255,255,0.25), transparent);
z-index: 3; pointer-events: none;
}

#ktHead {
display: flex; justify-content: space-between; align-items: flex-start;
padding: 16px 22px 8px; cursor: grab; user-select: none; gap: 12px;
}
#ktHead:active { cursor: grabbing; }
#ktHeadLeft { display: flex; flex-direction: column; gap: 8px; min-width: 0; }
#ktTitle {
font-size: 13px; font-weight: 600; letter-spacing: 0.3px;
color: rgba(255,255,255,0.55);
}
#ktShiftToggle {
display: inline-flex; align-items: center; gap: 0;
padding: 2px; border-radius: 10px;
background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.08);
pointer-events: auto; cursor: default;
}
.kt-shift-opt {
border: none; background: transparent; color: rgba(255,255,255,0.45);
font: 600 10px Inter, system-ui, sans-serif; letter-spacing: 0.3px;
padding: 5px 12px; border-radius: 8px; cursor: pointer;
transition: all 0.25s cubic-bezier(0.22,1,0.36,1);
}
.kt-shift-opt:hover { color: rgba(255,255,255,0.75); }
.kt-shift-opt.active {
color: #fff; background: rgba(124,58,237,0.45);
box-shadow: 0 2px 8px rgba(124,58,237,0.25);
}
#ktBtns { display: flex; flex-shrink: 0; padding-top: 2px; }
#ktLive {
display: inline-flex; align-items: center; gap: 6px;
font-size: 10px; font-weight: 600; color: #4ade80;
background: rgba(74,222,128,0.1);
border: 1px solid rgba(74,222,128,0.2);
padding: 4px 10px; border-radius: 20px; margin-left: 10px;
transition: transform 0.3s cubic-bezier(0.22,1,0.36,1), box-shadow 0.3s ease;
}
#ktLive i {
width: 6px; height: 6px; border-radius: 50%; background: #4ade80;
box-shadow: 0 0 8px rgba(74,222,128,0.8);
animation: ktLivePulse 2.5s ease-in-out infinite;
}
@keyframes ktLivePulse {
0%, 100% { opacity: 1; transform: scale(1); }
50% { opacity: 0.6; transform: scale(0.75); }
}
#ktLive:hover { transform: scale(1.04); box-shadow: 0 0 16px rgba(74,222,128,0.2); }

#ktBtns button {
width: 32px; height: 32px; border: 1px solid rgba(255,255,255,0.08);
border-radius: 10px; background: rgba(255,255,255,0.06);
color: rgba(255,255,255,0.7); font-size: 16px; cursor: pointer;
margin-left: 8px; transition: all 0.25s cubic-bezier(0.22,1,0.36,1);
backdrop-filter: blur(8px);
}
#ktBtns button:hover {
background: rgba(255,255,255,0.14); color: #fff;
transform: translateY(-2px) scale(1.05);
box-shadow: 0 4px 16px rgba(0,0,0,0.2);
}
#ktBtns button:active { transform: scale(0.95); }

#ktCanvas {
display: block; width: 100%; height: ${H}px;
cursor: pointer; margin: 0 0 4px;
}

#ktProgressWrap { padding: 0 22px 16px; }
#ktPctRow {
display: flex; justify-content: space-between; align-items: baseline;
margin-bottom: 10px;
}
#ktPct {
font-size: 22px; font-weight: 700; color: #fff;
letter-spacing: -0.5px;
transition: color 0.6s ease;
}
#ktPctLabel {
font-size: 10px; font-weight: 600; text-transform: uppercase;
letter-spacing: 1.2px; color: rgba(255,255,255,0.35);
}
#ktBar {
height: 5px; background: rgba(255,255,255,0.06);
border-radius: 9px; overflow: hidden; position: relative;
}
#ktBarFill {
height: 100%; width: 0%; border-radius: 9px;
background: linear-gradient(90deg, #6366f1, #22d3ee, #34d399);
transition: width 1s cubic-bezier(0.22,1,0.36,1);
position: relative;
}
#ktBarFill::after {
content: ''; position: absolute; inset: 0;
background: linear-gradient(90deg, transparent, rgba(255,255,255,0.35), transparent);
animation: ktShimmer 2.8s ease-in-out infinite;
}
@keyframes ktShimmer {
0% { transform: translateX(-100%); }
100% { transform: translateX(200%); }
}

#ktStats {
display: grid; grid-template-columns: repeat(4, 1fr);
gap: 1px; background: rgba(255,255,255,0.04);
margin: 0; opacity: 0; transform: translateY(10px);
transition: all 0.7s cubic-bezier(0.22,1,0.36,1);
}
#ktStats.show { opacity: 1; transform: none; }

#ktCredit {
text-align: center; font-size: 10px; font-weight: 500;
letter-spacing: 0.45px; color: rgba(255,255,255,0.3);
padding: 10px 22px 16px; margin: 0;
border-top: 1px solid rgba(255,255,255,0.06);
user-select: none; pointer-events: none; line-height: 1.4;
}
#ktCredit em { font-style: normal; color: rgba(167,139,250,0.65); font-weight: 600; }

.kt-stat {
background: rgba(0,0,0,0.22);
padding: 16px 8px; text-align: center; cursor: pointer;
transition: background 0.3s ease, transform 0.3s cubic-bezier(0.22,1,0.36,1),
            box-shadow 0.3s ease;
position: relative; overflow: hidden;
}
.kt-stat::before {
content: ''; position: absolute; inset: 0;
background: radial-gradient(circle at var(--mx,50%) var(--my,50%), rgba(124,58,237,0.15), transparent 70%);
opacity: 0; transition: opacity 0.3s ease;
}
.kt-stat:hover::before { opacity: 1; }
.kt-stat:hover {
background: rgba(255,255,255,0.06);
transform: translateY(-2px);
box-shadow: 0 4px 20px rgba(124,58,237,0.1);
}
.kt-stat:active { transform: scale(0.97); }
.kt-stat label {
display: block; font-size: 9px; text-transform: uppercase;
letter-spacing: 1px; color: rgba(255,255,255,0.38);
margin-bottom: 6px; font-weight: 600;
}
.kt-stat strong {
font-size: 13px; color: #fff; font-weight: 600; display: block;
transition: transform 0.4s cubic-bezier(0.22,1,0.36,1);
}

#ktToast {
position: fixed; bottom: 28px; left: 50%;
transform: translateX(-50%) translateY(12px);
background: rgba(12,18,32,0.92); color: #fff;
font-size: 12px; font-weight: 500; padding: 10px 20px;
border-radius: 14px; border: 1px solid rgba(255,255,255,0.1);
opacity: 0; transition: all 0.35s cubic-bezier(0.22,1,0.36,1);
z-index: 2147483647; pointer-events: none;
backdrop-filter: blur(20px);
box-shadow: 0 8px 32px rgba(0,0,0,0.4);
}
#ktToast.show { opacity: 1; transform: translateX(-50%) translateY(0); }

#kineticTime.kt-done #ktBarFill {
background: linear-gradient(90deg, #16a34a, #22c55e, #4ade80);
}
#kineticTime.kt-done #ktPct { color: #4ade80; }
#kineticTime.kt-done #ktPctLabel { color: rgba(74,222,128,0.55); }
#kineticTime.kt-done #ktLeft strong { color: #4ade80; }
#kineticTime.kt-done .kt-stat[data-k="left"] label { color: rgba(74,222,128,0.6); }

#kineticTime.kt-overtime #ktBarFill {
background: linear-gradient(90deg, #15803d, #22c55e, #86efac);
animation: ktOtPulse 2.5s ease-in-out infinite;
}
#kineticTime.kt-overtime #ktPct { color: #86efac; }
#kineticTime.kt-overtime #ktLeft strong { color: #86efac; }
#kineticTime.kt-overtime .kt-stat[data-k="left"] label { color: rgba(134,239,172,0.65); }
@keyframes ktOtPulse {
0%, 100% { filter: brightness(1); }
50% { filter: brightness(1.12); }
}

#kineticTime.kt-min .kt-body { display: none; }
#kineticTime.kt-min #ktCredit { display: none; }
#kineticTime.kt-min { width: ${W}px; }

#kineticTime.kt-detached {
position: relative; top: auto !important; left: auto !important; right: auto !important;
width: 100%; border-radius: 0; box-shadow: none;
}
#kineticTime.kt-detached #ktToast {
position: absolute; bottom: 20px;
}
`;

  function injectCSS(doc) {
      const d = doc || document;
      if (d.getElementById('ktCSS')) return;
      const s = d.createElement('style');
      s.id = 'ktCSS';
      s.textContent = CSS_TEXT;
      d.head.appendChild(s);
  }

  function injectRippleCSS(doc) {
      const d = doc || document;
      if (d.getElementById('ktRippleCSS')) return;
      const rippleStyle = d.createElement('style');
      rippleStyle.id = 'ktRippleCSS';
      rippleStyle.textContent = `@keyframes ktRipple{from{transform:scale(0);opacity:0.6}to{transform:scale(4);opacity:0}}`;
      d.head.appendChild(rippleStyle);
  }

  function applyBg() {
      const el = refs.root?.querySelector('#ktBg');
      if (!el) return;
      const [a, b, c] = E.palette.bg;
      el.style.background = c
          ? `linear-gradient(150deg,${a},${b},${c})`
          : `linear-gradient(150deg,${a},${b})`;
  }

  function toast(msg) {
      if (!refs.toast) return;
      refs.toast.textContent = msg;
      refs.toast.classList.add('show');
      setTimeout(() => refs.toast.classList.remove('show'), 1600);
  }

  function animateClose(root, toastEl, cb) {
      root.classList.add('kt-exit');
      setTimeout(() => {
          stopLoop();
          root.remove();
          toastEl.remove();
          bridgeEl?.remove();
          if (popupRef && !popupRef.closed) popupRef.close();
          cb();
      }, 450);
  }

  function showBridge() {
      if (bridgeEl) return;
      bridgeEl = document.createElement('div');
      bridgeEl.id = 'ktBridge';
      bridgeEl.style.cssText = `
position:fixed;bottom:14px;right:14px;z-index:2147483645;
padding:8px 14px;border-radius:12px;font:600 11px Inter,system-ui,sans-serif;
color:#4ade80;background:rgba(6,10,20,0.92);border:1px solid rgba(74,222,128,0.25);
backdrop-filter:blur(12px);cursor:pointer;user-select:none;
box-shadow:0 8px 24px rgba(0,0,0,0.35);
transition:transform 0.2s ease, box-shadow 0.2s ease;
`;
      bridgeEl.innerHTML = '<span style="opacity:0.7">📡</span> Tracker in window · click to focus';
      bridgeEl.onmouseenter = () => { bridgeEl.style.transform = 'translateY(-2px)'; };
      bridgeEl.onmouseleave = () => { bridgeEl.style.transform = ''; };
      bridgeEl.onclick = () => { if (popupRef && !popupRef.closed) popupRef.focus(); else dockToTab(); };
      document.body.appendChild(bridgeEl);
  }

  function hideBridge() {
      bridgeEl?.remove();
      bridgeEl = null;
  }

  function onPointerMove(e) {
      const root = refs.root;
      if (!root) return;
      if (drag) {
          drag.vx = e.clientX - drag.lx;
          drag.vy = e.clientY - drag.ly;
          drag.lx = e.clientX; drag.ly = e.clientY;
          if (detached) return;
          root.style.left = `${Math.max(8, Math.min(e.clientX - drag.ox, uiWin.innerWidth - drag.w - 8))}px`;
          root.style.top  = `${Math.max(8, Math.min(e.clientY - drag.oy, uiWin.innerHeight - drag.h - 8))}px`;
          root.style.right = 'auto';
      }
      const r = root.getBoundingClientRect();
      E.mouseX = (e.clientX - r.left) / r.width;
      E.mouseY = (e.clientY - r.top) / r.height;
      root.querySelectorAll('.kt-stat').forEach(card => {
          const cr = card.getBoundingClientRect();
          card.style.setProperty('--mx', `${((e.clientX - cr.left) / cr.width) * 100}%`);
          card.style.setProperty('--my', `${((e.clientY - cr.top) / cr.height) * 100}%`);
      });
  }

  function onPointerUp() {
      const root = refs.root;
      if (!root || !drag) return;
      const { vx, vy, w, h } = drag;
      drag = null;
      if (detached) return;
      const r = root.getBoundingClientRect();
      localStorage.setItem(STORAGE_KEY, JSON.stringify({ top: Math.round(r.top), left: Math.round(r.left) }));

      let momentum = { x: vx, y: vy };
      function glide() {
          if (Math.abs(momentum.x) < 0.3 && Math.abs(momentum.y) < 0.3) return;
          const cur = root.getBoundingClientRect();
          const nl = Math.max(8, Math.min(cur.left + momentum.x, uiWin.innerWidth - w - 8));
          const nt = Math.max(8, Math.min(cur.top + momentum.y, uiWin.innerHeight - h - 8));
          root.style.left = `${nl}px`;
          root.style.top = `${nt}px`;
          momentum.x *= 0.92; momentum.y *= 0.92;
          inertia = uiWin.requestAnimationFrame(glide);
      }
      if (Math.abs(vx) > 1 || Math.abs(vy) > 1) glide();
  }

  function bindPointerHandlers() {
      uiWin.addEventListener('pointermove', onPointerMove);
      uiWin.addEventListener('pointerup', onPointerUp);
  }

  function unbindPointerHandlers() {
      uiWin.removeEventListener('pointermove', onPointerMove);
      uiWin.removeEventListener('pointerup', onPointerUp);
  }

  function onVisibilityChange() {
      E.paused = !isUiVisible();
  }

  function bindVisibility() {
      document.removeEventListener('visibilitychange', onVisibilityChange);
      document.addEventListener('visibilitychange', onVisibilityChange);
      if (detached && uiDoc !== document) {
          uiDoc.removeEventListener('visibilitychange', onVisibilityChange);
          uiDoc.addEventListener('visibilitychange', onVisibilityChange);
      }
  }

  function popOutToWindow() {
      if (detached && popupRef && !popupRef.closed) {
          popupRef.focus();
          return true;
      }

      let pos = { top: 80, left: 120 };
      try {
          const p = JSON.parse(localStorage.getItem(STORAGE_KEY) || 'null');
          if (p) pos = p;
      } catch { /* */ }

      const features = [
          `width=${W + 24}`, `height=${H + 248}`,
          `left=${Math.max(0, pos.left)}`, `top=${Math.max(0, pos.top)}`,
          'menubar=no', 'toolbar=no', 'location=no', 'status=no', 'resizable=yes', 'scrollbars=no',
      ].join(',');

      popupRef = window.open('about:blank', POPUP_NAME, features);
      if (!popupRef) {
          pendingPop = true;
          toast('Popup blocked — click ⧉ or anywhere on page');
          return false;
      }

      pendingPop = false;
      const pdoc = popupRef.document;
      pdoc.open();
      pdoc.write(`<!DOCTYPE html><html><head><title>Shift Tracker</title></head>
<body style="margin:0;padding:0;background:#060a14;overflow:hidden;"></body></html>`);
      pdoc.close();

      injectCSS(pdoc);
      injectRippleCSS(pdoc);

      unbindPointerHandlers();
      uiWin = popupRef;
      uiDoc = pdoc;
      detached = true;

      refs.root.classList.add('kt-detached');
      refs.root.style.top = refs.root.style.left = refs.root.style.right = '';
      pdoc.body.appendChild(refs.root);
      pdoc.body.appendChild(refs.toast);

      bindPointerHandlers();
      bindVisibility();
      E.paused = false;

      const popBtn = refs.root.querySelector('#ktPop');
      if (popBtn) { popBtn.textContent = '⊟'; popBtn.title = 'Dock back to tab'; }

      localStorage.setItem(MODE_KEY, 'window');
      showBridge();
      toast('Popped out — free-floating window');

      popupRef.addEventListener('beforeunload', () => {
          if (detached) dockToTab(true);
      });

      return true;
  }

  function dockToTab(fromPopupClose) {
      if (!detached || !refs.root) return;

      unbindPointerHandlers();
      if (uiDoc !== document) uiDoc.removeEventListener('visibilitychange', onVisibilityChange);

      refs.root.classList.remove('kt-detached');
      document.body.appendChild(refs.root);
      document.body.appendChild(refs.toast);

      try {
          const p = JSON.parse(localStorage.getItem(STORAGE_KEY) || 'null');
          if (p) {
              refs.root.style.top = `${p.top}px`;
              refs.root.style.left = `${p.left}px`;
              refs.root.style.right = 'auto';
          }
      } catch { /* */ }

      uiWin = window;
      uiDoc = document;
      detached = false;
      popupRef = null;

      bindPointerHandlers();
      bindVisibility();
      E.paused = document.hidden;

      const popBtn = refs.root.querySelector('#ktPop');
      if (popBtn) { popBtn.textContent = '⧉'; popBtn.title = 'Open in separate window'; }

      localStorage.setItem(MODE_KEY, 'tab');
      hideBridge();

      if (!fromPopupClose) toast('Docked back to tab');
  }

  function afterUIReady() {
      if (localStorage.getItem(MODE_KEY) === 'tab') return;
      setTimeout(() => {
          if (!popOutToWindow()) pendingPop = true;
      }, 900);
  }

  function buildUI() {
      injectCSS();
      initAmbience();

      const root = document.createElement('div');
      root.id = 'kineticTime';
      root.className = 'kt-enter';
      root.innerHTML = `
<div id="ktBg"></div>
<div id="ktGlassHighlight"></div>
<div id="ktInner">
<div id="ktHead">
  <div id="ktHeadLeft">
    <span id="ktTitle">Shift Tracker <span id="ktLive"><i></i>LIVE</span></span>
    <div id="ktShiftToggle">
      <button type="button" class="kt-shift-opt active" data-h="8">Full · 8h</button>
      <button type="button" class="kt-shift-opt" data-h="4">Half · 4h</button>
    </div>
  </div>
  <div id="ktBtns">
    <button id="ktPop" title="Open in separate window">⧉</button>
    <button id="ktMin" title="Minimize">−</button>
    <button id="ktClose" title="Close">×</button>
  </div>
</div>
<div class="kt-body">
  <canvas id="ktCanvas" width="${W}" height="${H}"></canvas>
  <div id="ktProgressWrap">
    <div id="ktPctRow">
      <span id="ktPctLabel">Progress</span>
      <span id="ktPct">0%</span>
    </div>
    <div id="ktBar"><div id="ktBarFill"></div></div>
  </div>
  <div id="ktStats">
    <div class="kt-stat" data-k="worked"><label>Worked</label><strong id="ktWorked">—</strong></div>
    <div class="kt-stat" data-k="left"><label id="ktLeftLabel">Left</label><strong id="ktLeft">—</strong></div>
    <div class="kt-stat" data-k="break"><label>Break</label><strong id="ktBreak">—</strong></div>
    <div class="kt-stat" data-k="exit"><label>Done</label><strong id="ktExit">—</strong></div>
  </div>
</div>
<div id="ktCredit">Powered by <em>Vaibhav</em></div>
</div>`;
      document.body.appendChild(root);

      const toastEl = document.createElement('div');
      toastEl.id = 'ktToast';
      document.body.appendChild(toastEl);

      refs = {
          root, toast: toastEl,
          stats: root.querySelector('#ktStats'),
          worked: root.querySelector('#ktWorked'),
          left: root.querySelector('#ktLeft'),
          leftLabel: root.querySelector('#ktLeftLabel'),
          breakEl: root.querySelector('#ktBreak'),
          exit: root.querySelector('#ktExit'),
          barFill: root.querySelector('#ktBarFill'),
          pct: root.querySelector('#ktPct'),
          pctLabel: root.querySelector('#ktPctLabel'),
          canvas: root.querySelector('#ktCanvas'),
      };

      setShiftMode(shiftTargetMin === HALF_HOURS);

      root.querySelectorAll('.kt-shift-opt').forEach(btn => {
          btn.addEventListener('click', e => {
              e.stopPropagation();
              setShiftMode(btn.dataset.h === '4');
          });
      });

      try {
          const p = JSON.parse(localStorage.getItem(STORAGE_KEY) || 'null');
          if (p) { root.style.top = `${p.top}px`; root.style.left = `${p.left}px`; root.style.right = 'auto'; }
      } catch { /* */ }

      const head = root.querySelector('#ktHead');
      head.addEventListener('pointerdown', e => {
          if (e.target.closest('button')) return;
          const r = root.getBoundingClientRect();
          drag = { ox: e.clientX - r.left, oy: e.clientY - r.top, w: r.width, h: r.height, lx: e.clientX, ly: e.clientY, vx: 0, vy: 0 };
          if (inertia) uiWin.cancelAnimationFrame(inertia);
      });

      bindPointerHandlers();

      root.querySelector('#ktPop').onclick = () => {
          if (detached) dockToTab();
          else popOutToWindow();
      };

      root.querySelector('#ktMin').onclick = () => {
          root.classList.toggle('kt-min');
          root.querySelector('#ktMin').textContent = root.classList.contains('kt-min') ? '+' : '−';
      };

      root.querySelector('#ktClose').onclick = () => {
          animateClose(root, toastEl, () => {
              localStorage.removeItem(MODE_KEY);
              window.__KINETIC_TIME__ = false;
          });
      };

      refs.canvas.addEventListener('click', () => {
          navigator.clipboard?.writeText(`${E.timeStr} · ${E.timeSub}`);
          toast('Time copied');
          if (E.phase === 'rest') spawnStrike(true);
      });

      root.querySelectorAll('.kt-stat').forEach(el => {
          el.addEventListener('click', e => {
              const map = {
                  worked: refs.worked.textContent, left: refs.left.textContent,
                  break: refs.breakEl.textContent, exit: refs.exit.textContent,
              };
              navigator.clipboard?.writeText(map[el.dataset.k] || '');
              toast('Copied');
              const ripple = document.createElement('span');
              ripple.style.cssText = `position:absolute;border-radius:50%;background:rgba(255,255,255,0.2);
                width:20px;height:20px;left:${e.offsetX - 10}px;top:${e.offsetY - 10}px;
                animation:ktRipple 0.5s ease forwards;pointer-events:none;`;
              el.appendChild(ripple);
              setTimeout(() => ripple.remove(), 500);
          });
      });

      injectRippleCSS();
      bindVisibility();

      applyBg();
      spawnStrike(false);
      startLoop();

      if (Date.now() - E.lastHint > 8000) {
          E.lastHint = Date.now();
          E.currentHint = pick(HINTS);
      }
      setInterval(() => {
          if (Date.now() - E.lastHint > 10000) {
              E.lastHint = Date.now();
              E.currentHint = pick(HINTS);
          }
      }, 10000);

      afterUIReady();
  }

  let pushDebounce = null;
  let displayedPctUI = 0;

  function pushData() {
      const d = getSnap();
      const target = getShiftMin();
      const total = Math.max(0, d.totalMinutes);
      const brk   = Math.max(0, d.breakMinutes);
      const left  = Math.max(0, target - total);
      const overtime = Math.max(0, total - target);
      const pct   = Math.min(1, total / target);
      const isDone = total >= target;
      const isOT   = overtime > 0;
      const h = Math.floor(total / 60), m = total % 60;

      const hash = `${total}-${brk}-${target}`;
      const changed = hash !== E.dataHash;
      const wasComplete = E.dataHash && E.targetPct >= 1;
      E.dataHash = hash;

      E.targetMinutes = total;
      E.targetPct = pct;
      E.timeStr = `${h}:${String(m).padStart(2, '0')}`;
      E.timeSub = isOT
          ? `Complete · +${fmtHM(overtime)} OT`
          : isDone
              ? 'Shift complete ✓'
              : `${Math.round(pct * 100)}% · ${fmtHM(left)} left`;
      E.ringTarget = pct;

      const base = d.firstStart ? parseTime(d.firstStart) : null;
      const bm = base ? base.hours * 60 + base.minutes : null;

      if (refs.worked) {
          refs.worked.textContent = d.totalLabel;
          refs.leftLabel.textContent = isOT ? 'Overtime' : (isDone ? 'Status' : 'Left');
          refs.left.textContent = isOT
              ? `+${fmtHM(overtime)}`
              : isDone ? 'Done ✓' : fmtHM(left);
          refs.breakEl.textContent = fmtHM(brk);
          refs.exit.textContent   = bm != null ? clockAt(bm, target + brk) : '--';

          displayedPctUI = lerp(displayedPctUI, Math.min(100, pct * 100), 0.15);
          refs.barFill.style.width = `${Math.min(100, displayedPctUI)}%`;
          refs.pct.textContent = isOT
              ? `+${fmtHM(overtime)}`
              : isDone
                  ? '100% ✓'
                  : `${Math.round(displayedPctUI)}%`;
          refs.pctLabel.textContent = isOT ? 'Overtime' : (isDone ? 'Complete' : 'Progress');
          refs.stats.classList.add('show');
          refs.root.classList.toggle('kt-done', isDone);
          refs.root.classList.toggle('kt-overtime', isOT);

          if (isDone && !wasComplete) E.completionBurst = 1;
      }

      updateSatellites(isOT ? overtime : left, brk, isOT);
      const title = `${E.timeStr} · ${d.totalLabel}`;
      if (detached && popupRef && !popupRef.closed) popupRef.document.title = title;
      else document.title = title;

      if (changed && E.phase === 'rest' && E.timeVisible) spawnStrike(true);
  }

  function enhanceModal(container) {
      const d = processRows(container);
      if (!d) return;
      window.__KT_SNAP__ = d;
      pushData();

      let bar = container.querySelector('.kt-modal');
      if (!bar) {
          bar = document.createElement('div');
          bar.className = 'kt-modal';
          bar.style.cssText = 'margin:14px;padding:16px;border-radius:16px;background:#f8fafc;border:1px solid #e2e8f0;font-family:Inter,system-ui,sans-serif;display:grid;grid-template-columns:repeat(4,1fr);gap:12px;';
          container.appendChild(bar);
      }
      const target = getShiftMin();
      const left = Math.max(0, target - d.totalMinutes);
      const overtime = Math.max(0, d.totalMinutes - target);
      const base = d.firstStart ? parseTime(d.firstStart) : null;
      const bm = base ? base.hours * 60 + base.minutes : null;
      const leftVal = overtime > 0 ? `+${fmtHM(overtime)} OT` : (left ? fmtHM(left) : 'Done');
      const cells = [
          ['Worked', d.totalLabel], ['Left', leftVal],
          ['Break', fmtHM(d.breakMinutes)], ['Exit', bm != null ? clockAt(bm, target + d.breakMinutes) : '--']
      ];
      bar.innerHTML = cells.map(([l, v]) =>
          `<div style="text-align:center"><div style="font-size:10px;color:#94a3b8;text-transform:uppercase;letter-spacing:.6px;font-weight:600">${l}</div><div style="font-size:15px;font-weight:600;color:#1e293b;margin-top:5px">${v}</div></div>`
      ).join('');
  }

  // ── BOOT ────────────────────────────────────────────────

  buildUI();
  pushData();
  setInterval(pushData, TICK_MS);

  const obs = new MutationObserver(() => {
      clearTimeout(pushDebounce);
      pushDebounce = setTimeout(() => {
          const c = document.querySelector(LOG_SEL);
          if (c) enhanceModal(c);
          else pushData();
      }, 600);
  });
  obs.observe(document.body, { childList: true, subtree: true });

  document.addEventListener('keydown', e => {
      if (e.ctrlKey && e.shiftKey && e.key === 'T' && refs.root) {
          const hidden = refs.root.style.display === 'none';
          if (hidden) {
              refs.root.style.display = '';
              refs.root.classList.remove('kt-exit');
              refs.root.classList.add('kt-enter');
          } else {
              refs.root.style.display = 'none';
          }
      }
  });

  document.addEventListener('pointerdown', () => {
      if (pendingPop) popOutToWindow();
  }, true);

})();
