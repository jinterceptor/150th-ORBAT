<!-- /src/components/layout/Header.vue -->
<template>
  <div class="header-wrap" :style="{'--auth-x': authOffsetX + 'px', '--auth-y': authOffsetY + 'px'}">
    <header>
      <!-- Auth Indicator -->
      <div class="auth-indicator" v-if="isLoggedIn">
        <div class="auth-line">
          <span class="auth-role" :data-variant="authVariant">{{ authLabel }}</span>
          <span v-if="displayName" class="auth-name">· {{ displayName }}</span>
        </div>
        <button class="auth-logout" @click="onLogout">Logout</button>
      </div>

      <div class="title clipped-x-large-forward">
        <img class="logo" src="/faction-logos/Broadsword111.png" />
        <div class="title-container">
          <div id="title-first-line" class="title-row">
            <span id="title-header">UNSC TACNET</span>
          </div>
          <div class="title-row">
            <span id="subtitle-header">{{ header.subheaderTitle }}</span>
            <span id="subtitle-subheader">// {{ header.subheaderSubtitle }}</span>
          </div>
        </div>
      </div>

      <div class="rhombus"></div>

      <div class="planet-location-container">
        <video autoplay muted loop width="90px" height="90px">
          <source :src="`${planetPath}`" type="video/webm" />
        </video>

        <div class="location-info">
          <div class="location-row grid">
            <div id="year">
              <h4>Year</h4>
              <span class="subtitle">{{ header.year }}</span>
            </div>

            <div id="status" class="span-2">
              <h4>Status</h4>
              <span class="subtitle">{{ header.status }}</span>
            </div>
          </div>

          <div class="location-row grid">
            <div id="AO">
              <h4>AO</h4>
              <span class="subtitle">{{ header.AO }}</span>
            </div>

            <div id="planet">
              <h4>Planet</h4>
              <span class="subtitle">{{ header.planet }}</span>
            </div>

            <div id="system">
              <h4>System</h4>
              <span class="subtitle">{{ header.system }}</span>
            </div>
          </div>
        </div>
      </div>
    </header>

    <!-- Ambient News Ticker (below header) -->
    <div v-if="newsEnabled && normalizedNewsItems.length" class="news-ticker" aria-label="UNSC News Ticker">
      <div class="news-label">BROADCAST</div>
      <div class="news-viewport">
        <div
          class="news-track"
          :key="tickerKey"
          :style="{ '--ticker-duration': tickerDuration + 's', '--ticker-gap': tickerGapPx + 'px' }"
          @animationend="onTickerEnd"
        >
          <span class="news-text">{{ currentTickerText }}</span>
          <span class="news-sep" aria-hidden="true"> // </span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
/**
 * /src/components/layout/Header.vue
 * Add broadcasts by editing `defaultNewsItems` or by passing the `newsItems` prop.
 */
import {
  adminUser,
  isAdmin,
  adminLogout,
  subscribe as authSubscribe,
} from "@/utils/adminAuth";

const defaultNewsItems = [
  "TACTICAL UPDATE: Slipspace comms stable across local AO. Maintain emission control.",
  "FLEETCOM: UNSC logistics convoy rerouted. Expect delayed resupply window.",
  "ONI ADVISORY: OPSEC reminders in effect. Avoid publishing mission details outside TACNET.",
  "SITREP: Patrol activity increased near contested sectors. Proceed with caution.",
  "SYSTEM NOTICE: Training rotations updated. Check your squad channel for timings.",
];

export default {
  props: {
    planetPath: { type: String, required: true },
    header: { type: Object, required: true },
    authOffsetX: { type: Number, default: 330 },
    authOffsetY: { type: Number, default: 10 },

    newsEnabled: { type: Boolean, default: true },
    newsItems: { type: Array, default: () => defaultNewsItems },

    // Ambient pause between headlines
    newsMinDelayMs: { type: Number, default: 250 },
    newsMaxDelayMs: { type: Number, default: 1200 },

    // Scroll timing
    tickerBaseSeconds: { type: Number, default: 12 },
    tickerSecondsPerChar: { type: Number, default: 0.055 },

    // Slower overall multiplier (set to 1 for original speed)
    tickerSpeedMultiplier: { type: Number, default: 1.45 },

    // Space after the separator
    tickerGapPx: { type: Number, default: 70 },
  },
  data() {
    return {
      role: null,
      staffUser: null,
      unsub: null,

      tickerKey: 0,
      currentIndex: 0,
      tickerDuration: 18,

      _tickerTimeout: null,
      _lastIndex: -1,
    };
  },
  computed: {
    isLoggedIn() {
      return this.role === "member" || this.isStaff;
    },
    isStaff() {
      return isAdmin();
    },
    authVariant() {
      return this.isStaff ? "staff" : "member";
    },
    authLabel() {
      return this.isStaff ? "Staff" : "Member";
    },
    displayName() {
      if (!this.isStaff) return "";
      return (this.staffUser && this.staffUser.displayName) || "";
    },

    normalizedNewsItems() {
      const items = Array.isArray(this.newsItems) ? this.newsItems : [];
      return items
        .map((x) => (typeof x === "string" ? x : String(x?.text || x || "")))
        .map((s) => s.trim())
        .filter(Boolean);
    },
    currentTickerText() {
      const items = this.normalizedNewsItems;
      if (!items.length) return "";
      return items[this.currentIndex % items.length];
    },
  },
  created() {
    this.readAuth();
    this.unsub = authSubscribe(() => this.readAuth());
    window.addEventListener("storage", this.onStorage);

    this.startTicker();
  },
  beforeUnmount() {
    if (this.unsub) this.unsub();
    window.removeEventListener("storage", this.onStorage);
    this.stopTicker();
  },
  watch: {
    normalizedNewsItems() {
      this.startTicker();
    },
    newsEnabled() {
      this.startTicker();
    },
  },
  methods: {
    readAuth() {
      this.role = sessionStorage.getItem("authRole") || null;
      this.staffUser = adminUser();
    },
    onStorage(e) {
      if (!e) return;
      if (["admin:user", "admin:role", "admin:token", "admin:exp", "authRole"].includes(e.key)) {
        this.readAuth();
      }
    },
    async onLogout() {
      try { adminLogout(); } catch {}
      try { sessionStorage.removeItem("authRole"); } catch {}
      this.readAuth();
      if (this.$router?.currentRoute?.value?.path !== "/status") {
        this.$router.push("/status");
      }
    },

    startTicker() {
      this.stopTicker();
      if (!this.newsEnabled) return;
      if (!this.normalizedNewsItems.length) return;
      this.pickNextBroadcast(true);
    },
    stopTicker() {
      if (this._tickerTimeout) clearTimeout(this._tickerTimeout);
      this._tickerTimeout = null;
    },
    onTickerEnd() {
      const min = Math.max(0, Number(this.newsMinDelayMs) || 0);
      const max = Math.max(min, Number(this.newsMaxDelayMs) || min);
      const jitter = min + Math.floor(Math.random() * (max - min + 1));

      this.stopTicker();
      this._tickerTimeout = setTimeout(() => this.pickNextBroadcast(false), jitter);
    },
    pickNextBroadcast(force) {
      const items = this.normalizedNewsItems;
      if (!items.length) return;

      const next = this.randomIndex(items.length, force ? -1 : this._lastIndex);
      this._lastIndex = next;
      this.currentIndex = next;

      const textLen = items[next].length;
      const base = Math.max(4, Number(this.tickerBaseSeconds) || 12);
      const perChar = Math.max(0.01, Number(this.tickerSecondsPerChar) || 0.055);
      const mult = Math.max(0.2, Number(this.tickerSpeedMultiplier) || 1);

      const raw = base + textLen * perChar;
      this.tickerDuration = Math.max(8, Math.round(raw * mult * 10) / 10);

      this.tickerKey += 1; // restart animation from offscreen right
    },
    randomIndex(n, avoid) {
      if (n <= 1) return 0;
      let idx = Math.floor(Math.random() * n);
      if (idx === avoid) idx = (idx + 1 + Math.floor(Math.random() * (n - 1))) % n;
      return idx;
    },
  },
};
</script>

<style scoped>
.header-wrap{
  width: 100%;
  display: flex;
  flex-direction: column;
}

header{ border-radius: 0 !important; }

header .header-container,
header .inner,
header .topbar{
  display: flex;
  align-items: center;
}
header .planet-location-container{
  margin-left: auto !important;
  flex: 0 0 auto;
}

header{
  position: relative;
  border-radius: 0px;
  border: 1px solid rgba(170, 220, 255, 0.22);
  background: linear-gradient(180deg, rgba(8,14,20,0.90), rgba(3,6,10,0.94));
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.06) inset,
    0 0 26px rgba(120,180,255,0.10),
    0 0 110px rgba(0,0,0,0.55);
  overflow: hidden;
}

header::before{
  content:"";
  position:absolute;
  inset:0;
  pointer-events:none;
  background: repeating-linear-gradient(
    to bottom,
    rgba(255,255,255,0.02),
    rgba(255,255,255,0.02) 1px,
    rgba(0,0,0,0) 3px,
    rgba(0,0,0,0) 6px
  );
  mix-blend-mode: overlay;
  opacity: 0.22;
  z-index: 0;
}
header::after{
  content:"";
  position:absolute;
  inset:-20%;
  pointer-events:none;
  background: radial-gradient(circle at 30% 20%, rgba(120,180,255,0.07), transparent 58%);
  opacity: .85;
  animation: headerFlicker 3.1s infinite;
  z-index: 0;
}
@keyframes headerFlicker{
  0%,100%{ transform: translate3d(0,0,0); opacity:.70; }
  12%{ transform: translate3d(-1px,1px,0); opacity:.86; }
  25%{ transform: translate3d(1px,-1px,0); opacity:.68; }
  42%{ transform: translate3d(0,2px,0); opacity:.90; }
  70%{ transform: translate3d(2px,0,0); opacity:.76; }
}
header > *{ position: relative; z-index: 1; }

.rhombus{ opacity: .18; }

.auth-indicator {
  position: absolute;
  left: var(--auth-x, 315px);
  top: var(--auth-y, 10px);
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 8px 10px;
  border: 1px solid rgba(170,255,210,.35);
  border-radius: 999px;
  background: rgba(0,0,0,.35);
  color: rgba(170,255,210,.92);
  font-family: "Titillium Web", sans-serif;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  line-height: 1;
  z-index: 2;
}
.auth-line { display: inline-flex; align-items: center; gap: 6px; }
.auth-role {
  font-weight: 800;
  font-size: 12px;
  padding: 2px 8px;
  border-radius: 999px;
  border: 1px solid rgba(170,255,210,.35);
}
.auth-role[data-variant="member"] { opacity: .9; }
.auth-role[data-variant="staff"]  { border-color: rgba(30,144,255,.75); }
.auth-name { font-size: 12px; opacity: .9; }
.auth-logout {
  background: transparent;
  border: 1px solid rgba(170,255,210,.35);
  border-radius: 999px;
  padding: 2px 10px;
  color: rgba(170,255,210,.92);
  cursor: pointer;
  font-size: 11px;
  letter-spacing: .1em;
  text-transform: uppercase;
}
.auth-logout:hover { border-color: rgba(170,255,210,.9); }

.location-row.grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  column-gap: 1.2rem;
  align-items: end;
}
.span-2 { grid-column: span 2; }
.location-row h4 {
  text-transform: uppercase;
  letter-spacing: 0.12em;
  font-size: 0.7rem;
}
.subtitle { font-size: 0.85rem; letter-spacing: 0.08em; }

/* =========================
   News Ticker
   ========================= */
.news-ticker{
  height: 32px;
  display: grid;
  grid-template-columns: auto 1fr;
  align-items: center;
  gap: 10px;
  padding: 0 12px;
  border: 1px solid rgba(170,220,255,0.14);
  border-top: none;
  background: linear-gradient(180deg, rgba(8,14,20,0.75), rgba(3,6,10,0.88));
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.06) inset,
    0 0 26px rgba(120,180,255,0.08);
}

.news-label{
  font-family: "Titillium Web", sans-serif;
  font-size: 11px;
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: rgba(190,230,255,0.92);
  border: 1px solid rgba(170,220,255,0.18);
  background: rgba(0,0,0,0.18);
  border-radius: 999px;
  padding: 3px 10px;
  white-space: nowrap;
}

.news-viewport{
  overflow: hidden;
  width: 100%;
  mask-image: linear-gradient(to right, transparent 0%, black 7%, black 93%, transparent 100%);
}

.news-track{
  --ticker-duration: 18s;
  --ticker-gap: 70px;
  display: inline-block;
  white-space: nowrap;
  will-change: transform;
  padding-right: var(--ticker-gap);
  animation: tickerScroll var(--ticker-duration) linear forwards;
}

.news-text{
  font-family: "Titillium Web", sans-serif;
  font-size: 12px;
  letter-spacing: 0.10em;
  color: rgba(226,243,255,0.92);
  text-transform: uppercase;
  text-shadow: 0 0 14px rgba(120,180,255,0.10);
}

.news-sep{
  font-family: "Titillium Web", sans-serif;
  font-size: 12px;
  letter-spacing: 0.14em;
  color: rgba(158,197,230,0.85);
  text-transform: uppercase;
  opacity: 0.95;
}

/* Start fully offscreen (viewport), end fully left + gap */
@keyframes tickerScroll{
  0%   { transform: translate3d(100vw, 0, 0); opacity: 0.95; }
  5%   { opacity: 1; }
  100% { transform: translate3d(calc(-100% - var(--ticker-gap)), 0, 0); opacity: 0.95; }
}
</style>
