<!-- /src/App.vue -->
<template>
  <div v-if="showLogin" class="login-overlay" :class="{ fading: isFading }">
    <div class="term">
      <div class="term-hdr">
        <span class="dot"></span><span class="dot"></span><span class="dot"></span>
        <div class="term-title">UNSC SECURE ACCESS TERMINAL // BRIEFING GRID</div>
        <div class="term-stamp">{{ stamp }}</div>
      </div>

      <div class="term-body">
        <div class="scanlines" aria-hidden="true"></div>
        <div class="flicker" aria-hidden="true"></div>

        <div class="logo-ghost" aria-hidden="true">
          <img src="/faction-logos/FUD_UNSC_Logo.png" alt="" />
        </div>

        <!-- Ambient terminal feed (starts "scrolling" after 5 lines by trimming immediately) -->
        <div class="typed-window">
          <div class="typed" v-html="typedHtml"></div>
        </div>

        <div class="gate">
          <div class="gate-title">ACCESS OPTIONS</div>

          <div class="login-options-wrap">
            <div class="login-options">
              <button class="login-option" :disabled="isFading" @click="memberLogin">
                <div class="opt-title">Member Access</div>
                <div class="opt-desc">Read-only // Mission Status & Roster</div>
              </button>

              <button class="login-option" :disabled="isFading" @click="openAdminLogin">
                <div class="opt-title">Officer / Staff</div>
                <div class="opt-desc">Authenticate // Admin & Deployment</div>
              </button>
            </div>
          </div>

          <div class="hint dim">
            SELECT ACCESS PATH. UNAUTHORIZED ACCESS IS PUNISHABLE UNDER THE UNIFIED MILITARY CODE.
          </div>
        </div>
      </div>

      <div class="term-ftr dim">
        <span>SYS: OK</span>
        <span>IFF: VALID</span>
        <span>NET: LINK</span>
        <span>SEC: GREEN</span>
      </div>
    </div>

    <AdminLoginModal
      v-if="showAdminModal"
      @close="closeAdminLogin"
      @success="onAdminLoginSuccess"
    />
  </div>

  <template v-else>
    <div class="page-wrapper">
      <Header :planet-path="planetPath" :class="{ animate: animate }" :header="header" />
      <Sidebar :animate="animate" :class="{ animate: animate }" />
    </div>

    <div id="router-view-container">
      <router-view
        :key="$route.fullPath"
        :animate="animate"
        :initial-slug="initialSlug"
        :missions="missions"
        :events="events"
        :members="members"
        :orbat="orbat"
        :reserves="reserves"
      />
    </div>
  </template>

  <audio ref="startupAudio" preload="auto">
    <source src="/sound/startup.ogg" type="audio/ogg" />
  </audio>
</template>

<script>
import Header from "./components/layout/Header.vue";
import Sidebar from "./components/layout/Sidebar.vue";
import AdminLoginModal from "@/components/modals/AdminLoginModal.vue";
import Config from "@/assets/info/general-config.json";
import Papa from "papaparse";

export default {
  name: "App",
  components: { Header, Sidebar, AdminLoginModal },

  data() {
    return {
      showLogin: true,
      isFading: false,
      showAdminModal: false,

      // --- Ambient terminal feed ---
      feedPool: [
        "UNITED NATIONS SPACE COMMAND // SECURE MILNET",
        "NODE: ORBITAL BRIEFING SYSTEM (OBS)",
        "ROUTE: KHARON REACH RELAY // HEKATE AO",
        "AUTHORITY: NAVSPECWARCOM // 150TH RRG",
        "» ENCRYPTION SUITE: SWORD/VAULT // ACTIVE",
        "» THREAT FILTER: ENABLED // CONTENT SANITIZED",
        "» IFF PACKET: RECEIVED // VERIFIED",
        "» WRITE ACCESS: STAFF CREDENTIALS REQUIRED",
        "» TELEMETRY: SYNCHRONIZED",
        "» SATCOM: UPLINK STABLE",
        "» EDGE CACHE: PRIMED",
        "» INTRUSION MONITOR: ARMED",
        "» PACKET INTEGRITY: PASS",
        "» AUTH HANDSHAKE: LISTENING",
        "» LOCAL SESSION: QUARANTINE OK",
      ],

      typedLines: [],
      currentTarget: "",
      currentText: "",
      currentCharIndex: 0,

      bootTimer: null,
      stamp: "",

      // SIMPLE: start trimming after first 5 lines
      maxLines: 5,

      interLinePauseMs: 210,
      blankLineChance: 0.08,
      lineRepeatAvoid: 6,
      lastPickIndices: [],

      animate: Config.animate,
      initialSlug: Config.initialSlug,
      planetPath: Config.planetPath,
      header: Config.header,

      missions: [],
      events: [],
      members: [],
      orbat: [],
      reserves: [],
    };
  },

  computed: {
    typedHtml() {
      const escape = (s) =>
        String(s)
          .replaceAll("&", "&amp;")
          .replaceAll("<", "&lt;")
          .replaceAll(">", "&gt;");

      const renderLine = (l) => {
        if (l === "") return `<div class="line spacer"></div>`;
        const isFlavor = l.trim().startsWith("»");
        return `<div class="line ${isFlavor ? "dim" : ""}">${escape(l)}</div>`;
      };

      const committed = this.typedLines.map(renderLine).join("");
      const live = this.currentText ? renderLine(this.currentText) : "";
      return `${committed}${live}<div class="line dim"><span class="caret"></span></div>`;
    },
  },

  created() {
    this.setTitleFavicon(`${Config.defaultTitle} UNSC BRIEFING`, Config.icon);

    this.importMissions(import.meta.glob("@/assets/missions/**/*.md", { query: "?raw", import: "default" }));
    this.importEvents(import.meta.glob("@/assets/events/*.md", { query: "?raw", import: "default" }));

    const membersUrl =
      "https://docs.google.com/spreadsheets/d/e/2PACX-1vRq9fpYoWY_heQNfXegQ52zvOIGk-FCMML3kw2cX3M3s8blNRSH6XSRUdtTo7UXaJDDkg4bGQcl3jRP/pub?gid=1185035639&single=true&output=csv";
    const refDataUrl =
      "https://docs.google.com/spreadsheets/d/e/2PACX-1vRq9fpYoWY_heQNfXegQ52zvOIGk-FCMML3kw2cX3M3s8blNRSH6XSRUdtTo7UXaJDDkg4bGQcl3jRP/pub?gid=107253735&single=true&output=csv";
    const opsUrl =
      "https://docs.google.com/spreadsheets/d/e/2PACX-1vRq9fpYoWY_heQNfXegQ52zvOIGk-FCMML3kw2cX3M3s8blNRSH6XSRUdtTo7UXaJDDkg4bGQcl3jRP/pub?gid=1115158828&single=true&output=csv";

    this.loadMembersCSV(membersUrl)
      .then(() => this.loadRefDataCSV(refDataUrl))
      .then(() => this.loadOpsCSV(opsUrl).catch((err) => console.warn("Ops CSV failed; continuing.", err)));

    const role = sessionStorage.getItem("authRole");
    if (role === "member" || role === "staff") {
      this.showLogin = false;
    }
  },

  mounted() {
    this.updateStamp();
    this.seedInitialFeed();
    this.startAmbientFeed();
  },

  beforeUnmount() {
    if (this.bootTimer) window.clearTimeout(this.bootTimer);
  },

  methods: {
    updateStamp() {
      const d = new Date();
      const pad = (n) => String(n).padStart(2, "0");
      this.stamp = `UTC ${pad(d.getUTCHours())}:${pad(d.getUTCMinutes())}:${pad(d.getUTCSeconds())}`;
    },

    seedInitialFeed() {
      if (!this.showLogin) return;
      this.typedLines = [
        "UNITED NATIONS SPACE COMMAND // SECURE MILNET",
        "NODE: ORBITAL BRIEFING SYSTEM (OBS)",
        "» AUTH HANDSHAKE: LISTENING",
        "» ENCRYPTION SUITE: SWORD/VAULT // ACTIVE",
        "» THREAT FILTER: ENABLED // CONTENT SANITIZED",
      ].slice(-this.maxLines);
      this.pickNextTarget();
    },

    pickNextTarget() {
      if (Math.random() < this.blankLineChance) {
        this.currentTarget = "";
        this.currentText = "";
        this.currentCharIndex = 0;
        return;
      }

      const poolLen = this.feedPool.length;
      let idx = Math.floor(Math.random() * poolLen);
      let guard = 0;
      while (this.lastPickIndices.includes(idx) && guard < 20) {
        idx = Math.floor(Math.random() * poolLen);
        guard += 1;
      }

      this.lastPickIndices.push(idx);
      if (this.lastPickIndices.length > this.lineRepeatAvoid) this.lastPickIndices.shift();

      const base = this.feedPool[idx];
      this.currentTarget = this.withTelemetry(base);
      this.currentText = "";
      this.currentCharIndex = 0;
    },

    withTelemetry(line) {
      const pad = (n) => String(n).padStart(2, "0");
      const now = new Date();
      const t = `${pad(now.getUTCHours())}:${pad(now.getUTCMinutes())}:${pad(now.getUTCSeconds())}`;
      const hex = () => Math.floor(Math.random() * 0xffff).toString(16).toUpperCase().padStart(4, "0");
      const pct = () => `${Math.floor(80 + Math.random() * 20)}%`;

      if (line.includes("TELEMETRY")) return `» TELEMETRY: SYNCHRONIZED // ${pct()} // T+${t}`;
      if (line.includes("SATCOM")) return `» SATCOM: UPLINK STABLE // CH-${1 + Math.floor(Math.random() * 6)} // ${t}`;
      if (line.includes("PACKET INTEGRITY")) return `» PACKET INTEGRITY: PASS // CRC ${hex()}-${hex()} // ${t}`;
      if (line.includes("EDGE CACHE")) return `» EDGE CACHE: PRIMED // SECTOR ${hex()} // ${t}`;
      if (line.includes("INTRUSION MONITOR")) return `» INTRUSION MONITOR: ARMED // WATCH ${hex()} // ${t}`;
      if (line.includes("IFF PACKET")) return `» IFF PACKET: RECEIVED // TAG ${hex()} // ${t}`;

      return line;
    },

    trimToWindow() {
      // start "scrolling" immediately after 5 lines
      if (this.typedLines.length > this.maxLines) {
        this.typedLines.splice(0, this.typedLines.length - this.maxLines);
      }
    },

    commitCurrentLine() {
      this.typedLines.push(this.currentTarget === "" ? "" : this.currentTarget);
      this.trimToWindow();
    },

    startAmbientFeed() {
      if (!this.showLogin) return;

      const minDelay = 12;
      const maxDelay = 30;

      const tick = () => {
        this.updateStamp();

        if (!this.showLogin) {
          this.bootTimer = null;
          return;
        }

        if (this.currentTarget === "") {
          this.commitCurrentLine();
          this.pickNextTarget();
          this.bootTimer = window.setTimeout(tick, Math.max(120, this.interLinePauseMs));
          return;
        }

        this.currentText = this.currentTarget.slice(0, this.currentCharIndex + 1);
        this.currentCharIndex += 1;

        if (this.currentCharIndex >= this.currentTarget.length) {
          this.commitCurrentLine();
          this.pickNextTarget();
          this.bootTimer = window.setTimeout(tick, this.interLinePauseMs);
          return;
        }

        const jitter = Math.floor(minDelay + Math.random() * (maxDelay - minDelay));
        const extra = Math.random() < 0.06 ? 90 : 0;
        this.bootTimer = window.setTimeout(tick, jitter + extra);
      };

      tick();
    },

    memberLogin() {
      if (this.isFading) return;
      sessionStorage.setItem("authRole", "member");
      this.fadeAndEnter("/status");
    },

    openAdminLogin() {
      if (this.isFading) return;
      this.showAdminModal = true;
    },
    closeAdminLogin() {
      this.showAdminModal = false;
    },
    onAdminLoginSuccess() {
      sessionStorage.setItem("authRole", "staff");
      this.showAdminModal = false;
      this.fadeAndEnter("/admin");
    },

    fadeAndEnter(targetPath) {
      this.isFading = true;

      if (this.bootTimer) {
        window.clearTimeout(this.bootTimer);
        this.bootTimer = null;
      }

      const a = this.$refs.startupAudio;
      if (a && typeof a.play === "function") {
        a.currentTime = 0;
        a.play().catch(() => {});
      }

      window.setTimeout(() => {
        this.showLogin = false;
        this.$router.replace(targetPath);
      }, 650);
    },

    setTitleFavicon(title, iconPath) {
      if (title) document.title = String(title);
      if (!iconPath) return;

      let link = document.querySelector("link[rel*='icon']");
      if (!link) {
        link = document.createElement("link");
        link.rel = "icon";
        document.head.appendChild(link);
      }
      link.href = iconPath;
    },

    async importMissions(glob) {
      const entries = Object.entries(glob);
      const out = [];
      for (const [path, loader] of entries) {
        const raw = await loader();
        const name = path.split("/").pop().replace(".md", "");
        out.push({ slug: name, raw });
      }
      this.missions = out.sort((a, b) => a.slug.localeCompare(b.slug));
    },

    async importEvents(glob) {
      const entries = Object.entries(glob);
      const out = [];
      for (const [path, loader] of entries) {
        const raw = await loader();
        const name = path.split("/").pop().replace(".md", "");
        out.push({ slug: name, raw });
      }
      this.events = out.sort((a, b) => a.slug.localeCompare(b.slug));
    },

    // Remaining CSV/promotion methods unchanged...
    promotionLadderFor(rankName) {
      const r = String(rankName || "").trim().toUpperCase();

      if (r.includes("SR") && r.includes("PILOT")) return { nextRank: "Chief Pilot", nextAt: 16 };
      if (r.includes("CHIEF") && r.includes("PILOT")) return { nextRank: "Master Chief Pilot", nextAt: 24 };

      if (r.includes("MASTER") && r.includes("CHIEF")) return { nextRank: null, nextAt: null };
      if (r.includes("CHIEF")) return { nextRank: "Master Chief", nextAt: 24 };

      if (r.includes("SERGEANT") && r.includes("FIRST")) return { nextRank: "Chief", nextAt: 20 };
      if (r.includes("GUNNERY")) return { nextRank: "First Sergeant", nextAt: 16 };
      if (r.includes("STAFF")) return { nextRank: "Gunnery Sergeant", nextAt: 12 };
      if (r.includes("SERGEANT")) return { nextRank: "Staff Sergeant", nextAt: 8 };

      if (r.includes("CORPORAL")) return { nextRank: "Sergeant", nextAt: 6 };
      if (r.includes("LANCE")) return { nextRank: "Corporal", nextAt: 4 };
      if (r.includes("PRIVATE FIRST") || r.includes("PFC")) return { nextRank: "Lance Corporal", nextAt: 2 };
      if (r.includes("PRIVATE")) return { nextRank: "Private First Class", nextAt: 1 };

      return { nextRank: null, nextAt: null };
    },
    opsToNextPromotion(member) {
      const ops = Number(member?.opsAttended);
      if (!Number.isFinite(ops)) return null;
      const ladder = this.promotionLadderFor(member?.rank);
      if (!ladder || !ladder.nextAt) return null;
      return Math.max(0, ladder.nextAt - ops);
    },
    nextPromotionRank(member) {
      const ladder = this.promotionLadderFor(member?.rank);
      return ladder?.nextRank || null;
    },

    loadMembersCSV(csvUrl) {
      return new Promise((resolve, reject) => {
        Papa.parse(csvUrl, {
          download: true,
          header: true,
          skipEmptyLines: true,
          complete: (results) => {
            this.members = (results.data || []).map((r) => ({
              ...r,
              opsAttended: 0,
              nextPromotionOps: null,
              nextPromotionRank: null,
            }));
            resolve();
          },
          error: reject,
        });
      });
    },

    loadRefDataCSV(csvUrl) {
      return new Promise((resolve, reject) => {
        Papa.parse(csvUrl, {
          download: true,
          header: true,
          skipEmptyLines: true,
          complete: (results) => {
            const rows = results.data || [];

            const orbat = rows
              .filter((r) => String(r?.type || "").toLowerCase() === "orbat")
              .map((r) => ({ ...r }));
            const reserves = rows
              .filter((r) => String(r?.type || "").toLowerCase() === "reserve")
              .map((r) => ({ ...r }));

            this.orbat = orbat;
            this.reserves = reserves;

            resolve();
          },
          error: reject,
        });
      });
    },

    loadOpsCSV(csvUrl) {
      return new Promise((resolve, reject) => {
        Papa.parse(csvUrl, {
          download: true,
          header: true,
          skipEmptyLines: true,
          complete: (results) => {
            const rows = results.data || [];
            const byName = new Map();

            rows.forEach((r) => {
              const name = String(r?.name || r?.Name || "").trim();
              const ops = Number(r?.ops || r?.Ops || r?.opsAttended || r?.OpsAttended);
              if (!name) return;
              if (!Number.isFinite(ops)) return;
              byName.set(name.toLowerCase(), ops);
            });

            this.members = (this.members || []).map((m) => {
              const key = String(m?.name || m?.Name || "").trim().toLowerCase();
              const ops = byName.get(key) ?? 0;
              const nextPromotionOps = this.opsToNextPromotion({ ...m, opsAttended: ops });
              const nextPromotionRank = this.nextPromotionRank({ ...m, opsAttended: ops });

              return { ...m, opsAttended: ops, nextPromotionOps, nextPromotionRank };
            });

            resolve();
          },
          error: reject,
        });
      });
    },
  },
};
</script>

<style>
#app { min-height: 100vh; overflow: hidden !important; }

.login-overlay {
  position: fixed;
  inset: 0;
  z-index: 99999;
  display: grid;
  place-items: center;
  background: radial-gradient(ellipse at center, rgba(6, 12, 18, 0.96), rgba(0, 0, 0, 0.985));
  opacity: 1;
  transition: opacity 0.8s ease;
}
.login-overlay.fading { opacity: 0; pointer-events: none; }

.term {
  width: min(920px, 94vw);
  border-radius: 16px;
  border: 1px solid rgba(170, 220, 255, 0.22);
  background: linear-gradient(180deg, rgba(8, 14, 20, 0.92), rgba(3, 6, 10, 0.95));
  overflow: hidden;
  box-shadow:
    0 0 0 1px rgba(170, 220, 255, 0.06) inset,
    0 0 26px rgba(120, 180, 255, 0.10),
    0 0 110px rgba(0, 0, 0, 0.6);
  color: rgba(190, 230, 255, 0.92);
  font-family: "Titillium Web", sans-serif;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

.term-hdr {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px 14px;
  border-bottom: 1px solid rgba(170, 220, 255, 0.12);
  background: rgba(0, 0, 0, 0.16);
}
.dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: rgba(170, 220, 255, 0.22);
  box-shadow: 0 0 8px rgba(120, 180, 255, 0.12);
}
.term-title { font-size: 12px; opacity: 0.9; }
.term-stamp { margin-left: auto; font-size: 12px; opacity: 0.6; letter-spacing: .12em; }

.term-body {
  position: relative;
  padding: 26px 20px 18px;
  min-height: 380px;
}

.logo-ghost {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  pointer-events: none;
  opacity: 0.12;
  filter: drop-shadow(0 0 24px rgba(0, 0, 0, 0.9));
}
.logo-ghost img { width: min(520px, 74vw); height: auto; }

/* With maxLines=5, the feed "scrolls" almost immediately and never collides with the divider */
.typed-window {
  position: relative;
  z-index: 2;
  height: 190px;
  overflow: hidden;
  padding-right: 6px;

  -webkit-mask-image: linear-gradient(to bottom, rgba(0,0,0,1) 78%, rgba(0,0,0,0) 100%);
  mask-image: linear-gradient(to bottom, rgba(0,0,0,1) 78%, rgba(0,0,0,0) 100%);
}

.typed {
  font-size: 14px;
  line-height: 1.7;
  text-shadow: 0 0 10px rgba(120, 180, 255, 0.10);
}

.line { margin: 2px 0; }
.dim { opacity: 0.7; }
.line.spacer { height: 10px; }

/* Optional: keep each line to a single terminal row so it never wraps into the gate area */
.line { white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }

.caret {
  display: inline-block;
  width: 10px;
  height: 14px;
  margin-right: 8px;
  border-left: 2px solid rgba(190, 230, 255, 0.9);
  animation: blink 1.1s infinite;
  transform: translateY(2px);
}
@keyframes blink {
  0%, 45% { opacity: 1; }
  46%, 100% { opacity: 0; }
}

.gate {
  position: relative;
  z-index: 2;
  margin-top: 14px;
  padding-top: 14px;
  border-top: 1px solid rgba(170, 220, 255, 0.14);
}

.gate-title {
  font-size: 12px;
  opacity: 0.85;
  margin-bottom: 10px;
  letter-spacing: .16em;
}

.login-options-wrap {
  display: flex;
  justify-content: center;
  width: 100%;
}

.login-options {
  display: flex;
  justify-content: center;
  gap: 14px;
  width: min(760px, 100%);
  margin: 0 auto;
  flex-wrap: wrap;
}

.login-option {
  flex: 1 1 280px;
  max-width: 360px;

  background: rgba(0, 0, 0, 0.38);
  border: 1px solid rgba(170, 220, 255, 0.26);
  border-radius: 14px;
  padding: 16px 18px;
  cursor: pointer;
  color: rgba(190, 230, 255, 0.92);
  text-transform: uppercase;
  letter-spacing: .12em;
  transition: transform 120ms ease, border-color 140ms ease, opacity 140ms ease;
}
.login-option:hover { border-color: rgba(170, 220, 255, 0.9); transform: translateY(-1px); }
.login-option:disabled { opacity: 0.35; cursor: not-allowed; transform: none; }

.opt-title { font-size: 16px; font-weight: 800; margin-bottom: 6px; }
.opt-desc { font-size: 12px; opacity: .8; letter-spacing: .08em; }

.hint { margin-top: 12px; font-size: 11px; letter-spacing: .14em; line-height: 1.7; }

.term-ftr {
  display: flex;
  justify-content: space-between;
  padding: 10px 14px;
  border-top: 1px solid rgba(170, 220, 255, 0.12);
  background: rgba(0, 0, 0, 0.16);
  font-size: 12px;
}

/* FX */
.scanlines {
  position: absolute;
  inset: 0;
  background: repeating-linear-gradient(
    to bottom,
    rgba(255, 255, 255, 0.02),
    rgba(255, 255, 255, 0.02) 1px,
    rgba(0, 0, 0, 0) 3px,
    rgba(0, 0, 0, 0) 6px
  );
  mix-blend-mode: overlay;
  opacity: 0.35;
  pointer-events: none;
}

.flicker {
  position: absolute;
  inset: -20%;
  background: radial-gradient(circle at 30% 20%, rgba(120, 180, 255, 0.07), transparent 55%);
  animation: flicker 2.6s infinite;
  pointer-events: none;
  opacity: 0.9;
}

@keyframes flicker {
  0%, 100% { transform: translate3d(0, 0, 0); opacity: 0.75; }
  10% { transform: translate3d(-1px, 1px, 0); opacity: 0.85; }
  20% { transform: translate3d(1px, -1px, 0); opacity: 0.7; }
  35% { transform: translate3d(0px, 2px, 0); opacity: 0.9; }
  60% { transform: translate3d(2px, 0px, 0); opacity: 0.78; }
}
</style>
