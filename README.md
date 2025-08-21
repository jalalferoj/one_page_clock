<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <meta name="theme-color" content="#0f172a" />
  <title>Clock</title>
  <style>
    :root {
      --bg: radial-gradient(1200px 1200px at 10% 10%, #0b1224 0%, #0f172a 40%, #0b1020 100%);
      --card: rgba(255, 255, 255, 0.06);
      --text: #e5e7eb;
      --muted: #9ca3af;
      --accent: #60a5fa;
      --ring: rgba(96, 165, 250, 0.35);
      --shadow: 0 10px 30px rgba(0, 0, 0, 0.35);
    }
    * { box-sizing: border-box; }
    html, body {
      height: 100%;
      margin: 0;
      font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, "Helvetica Neue", Arial, "Noto Sans", "Apple Color Emoji", "Segoe UI Emoji";
      color: var(--text);
      background: var(--bg), #0f172a;
    }
    body {
      display: grid;
      place-items: center;
      padding: 24px;
    }
    .clock {
      width: min(680px, 92vw);
      background: linear-gradient(180deg, rgba(255,255,255,0.08), rgba(255,255,255,0.03));
      border: 1px solid rgba(255,255,255,0.12);
      border-radius: 20px;
      box-shadow: var(--shadow);
      backdrop-filter: blur(6px);
      padding: 28px 28px 22px;
    }
    .heading {
      display: flex; align-items: center; justify-content: space-between; gap: 12px;
      margin-bottom: 6px;
    }
    .title { font-weight: 600; letter-spacing: 0.4px; color: var(--muted); font-size: 14px; text-transform: uppercase; }
    .time {
      font-variant-numeric: tabular-nums;
      font-size: clamp(44px, 8vw, 86px);
      line-height: 1.05;
      letter-spacing: 1px;
      margin: 6px 0 12px;
      text-shadow: 0 2px 24px rgba(96,165,250,0.18);
    }
    .date { color: var(--muted); font-size: clamp(14px, 2.8vw, 18px); letter-spacing: 0.2px; }
    .controls {
      display: flex; flex-wrap: wrap; gap: 10px; margin-top: 18px;
    }
    .chip {
      display: inline-flex; align-items: center; gap: 10px;
      padding: 10px 14px; border-radius: 999px;
      background: var(--card); color: var(--text);
      border: 1px solid rgba(255,255,255,0.12);
      transition: border-color 140ms, box-shadow 140ms;
    }
    .chip:hover { border-color: var(--accent); box-shadow: 0 0 0 4px var(--ring) inset; }
    .chip input { appearance: none; width: 18px; height: 18px; border-radius: 4px;
      border: 1.5px solid rgba(255,255,255,0.5); display: grid; place-items: center; background: transparent; }
    .chip input:checked { background: var(--accent); border-color: var(--accent); }
    .chip input:checked::after { content: ""; width: 10px; height: 10px; border-radius: 2px; background: white; }
    .footer { margin-top: 16px; color: var(--muted); font-size: 12px; }
    .sr-only { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip: rect(0,0,0,0); white-space: nowrap; border: 0; }
  </style>
</head>
<body>
  <main class="clock" role="group" aria-labelledby="clock-title">
    <div class="heading">
      <div class="title" id="clock-title">Local Time</div>
      <div class="date" id="date"></div>
    </div>
    <div class="time" id="time" aria-live="polite">00:00:00</div>

    <div class="controls" aria-label="Clock settings">
      <label class="chip" for="toggle-24h">
        <input type="checkbox" id="toggle-24h" />
        <span>24‑hour</span>
      </label>

      <label class="chip" for="toggle-seconds">
        <input type="checkbox" id="toggle-seconds" checked />
        <span>Show seconds</span>
      </label>
    </div>

    <div class="footer">Time auto-updates from your device.</div>
  </main>

  <span class="sr-only" id="aria-announce" aria-live="polite"></span>

  <script>
    (function () {
      const timeEl = document.getElementById('time');
      const dateEl = document.getElementById('date');
      const is24hEl = document.getElementById('toggle-24h');
      const showSecEl = document.getElementById('toggle-seconds');
      const announcer = document.getElementById('aria-announce');

      // Load saved preferences
      const saved24h = localStorage.getItem('clock:is24h');
      const savedSec = localStorage.getItem('clock:showSeconds');
      if (saved24h !== null) is24hEl.checked = saved24h === '1';
      if (savedSec !== null) showSecEl.checked = savedSec === '1';

      is24hEl.addEventListener('change', () => {
        localStorage.setItem('clock:is24h', is24hEl.checked ? '1' : '0');
        announce(is24hEl.checked ? '24-hour format' : '12-hour format');
        update();
      });
      showSecEl.addEventListener('change', () => {
        localStorage.setItem('clock:showSeconds', showSecEl.checked ? '1' : '0');
        announce(showSecEl.checked ? 'Showing seconds' : 'Hiding seconds');
        update();
      });

      function pad(n) { return n.toString().padStart(2, '0'); }

      function formatTime(d, is24h, showSeconds) {
        let hours = d.getHours();
        const minutes = d.getMinutes();
        const seconds = d.getSeconds();
        let suffix = '';

        if (!is24h) {
          suffix = hours >= 12 ? ' PM' : ' AM';
          hours = hours % 12 || 12;
        }
        const base = `${pad(hours)}:${pad(minutes)}`;
        const sec = showSeconds ? `:${pad(seconds)}` : '';
        return base + sec + suffix;
      }

      function formatDate(d) {
        const opts = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
        return new Intl.DateTimeFormat(undefined, opts).format(d);
      }

      function update() {
        const now = new Date();
        timeEl.textContent = formatTime(now, is24hEl.checked, showSecEl.checked);
        dateEl.textContent = formatDate(now);
      }

      // Align updates to the top of each second
      function schedule() {
        const ms = 1000 - (Date.now() % 1000);
        setTimeout(() => { update(); schedule(); }, ms);
      }

      function announce(text) {
        announcer.textContent = text;
      }

      update();
      schedule();
    })();
  </script>
</body>
</html>
