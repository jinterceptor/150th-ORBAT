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

        <div class="typed" v-html="typedHtml"></div>

        <div class="gate">
          <div class="gate-title">ACCESS OPTIONS</div>

          <div class="login-options">
            <button class="login-option" :disabled="!typedDone || isFading" @click="memberLogin">
              <div class="opt-title">Member Access</div>
              <div class="opt-desc">Read-only // Mission Status & Roster</div>
            </button>

            <button class="login-option" :disabled="!typedDone || isFading" @click="openAdminLogin">
              <div class="opt-title">Officer / Staff</div>
              <div class="opt-desc">Authenticate // Admin & Deployment</div>
            </button>
          </div>

          <div class="hint dim" v-if="!typedDone">
            INITIALIZING… INPUT LOCKED UNTIL HANDSHAKE COMPLETES
          </div>
          <div class="hint dim" v-else>
            SELECT ACCESS PATH. UNAUTHORIZED ACCESS IS PUNISHABLE UNDER THE UNIFIED MILITARY CODE.
          </div>
        </div>
      </div>

      <div class="term-ftr dim">
        <span>SYS: OK</span>
        <span>IFF: VALID</span>
        <span>NET: {{ typedDone ? "LINK" : "SYNC" }}</span>
        <span>SEC: {{ typedDone ? "GREEN" : "AMBER" }}</span>
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
      <!-- transition intentionally removed -->
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

      // --- UNSC terminal boot ---
      fullLines: [
        "UNITED NATIONS SPACE COMMAND // SECURE MILNET",
        "NODE: ORBITAL BRIEFING SYSTEM (OBS)",
        "ROUTE: KHARON REACH RELAY // HEKATE AO",
        "AUTHORITY: NAVSPECWARCOM // 150TH RRG",
        "",
        "» ENCRYPTION SUITE: SWORD/VAULT // ACTIVE",
        "» THREAT FILTER: ENABLED // CONTENT SANITIZED",
        "» IFF PACKET: RECEIVED // VERIFIED",
        "» WRITE ACCESS: STAFF CREDENTIALS REQUIRED",
        "",
        "HANDSHAKE: PENDING // STANDBY",
      ],
      typedLines: [],
      typedCharIndex: 0,
      lineIndex: 0,
      typedDone: false,
      bootTimer: null,
      stamp: "",

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

      const html = this.typedLines
        .map((l) => {
          if (l === "") return `<div class="line spacer"></div>`;
          const isFlavor = l.trim().startsWith("»");
          return `<div class="line ${isFlavor ? "dim" : ""}">${escape(l)}</div>`;
        })
        .join("");

      if (!this.typedDone) return `${html}<div class="line dim"><span class="caret"></span></div>`;
      return html;
    },
  },

  created() {
    this.setTitleFavicon(`${Config.defaultTitle} UNSC BRIEFING`, Config.icon);

    // preload markdown
    this.importMissions(import.meta.glob("@/assets/missions/**/*.md", { query: "?raw", import: "default" }));
    this.importEvents(import.meta.glob("@/assets/events/*.md", { query: "?raw", import: "default" }));

    // async CSVs
    const membersUrl = "https://docs.google.com/spreadsheets/d/e...o7UXaJDDkg4bGQcl3jRP/pub?gid=1185035639&single=true&output=csv";
    const refDataUrl = "https://docs.google.com/spreadsheets/d/e...To7UXaJDDkg4bGQcl3jRP/pub?gid=107253735&single=true&output=csv";
    const opsUrl = "https://docs.google.com/spreadsheets/d/e/2PA...o7UXaJDDkg4bGQcl3jRP/pub?gid=1115158828&single=true&output=csv";

    this.loadMembersCSV(membersUrl)
      .then(() => this.loadRefDataCSV(refDataUrl))
      .then(() => this.loadOpsCSV(opsUrl).catch((err) => console.warn("Ops CSV failed; continuing.", err)));

    // if already authenticated earlier, skip overlay
    const role = sessionStorage.getItem("authRole");
    if (role === "member" || role === "staff") {
      this.showLogin = false;
    }
  },

  mounted() {
    this.updateStamp();
    this.startTyping();
  },

  beforeUnmount() {
    if (this.bootTimer) window.clearTimeout(this.bootTimer);
  },

  methods: {
    // --- terminal boot helpers ---
    updateStamp() {
      const d = new Date();
      const pad = (n) => String(n).padStart(2, "0");
      this.stamp = `UTC ${pad(d.getUTCHours())}:${pad(d.getUTCMinutes())}:${pad(d.getUTCSeconds())}`;
    },

    startTyping() {
      if (!this.showLogin) return;

      const minDelay = 12;
      const maxDelay = 30;

      const tick = () => {
        this.updateStamp();

        if (this.lineIndex >= this.fullLines.length) {
          this.typedDone = true;
          this.bootTimer = null;
          return;
        }

        const target = this.fullLines[this.lineIndex];

        if (target === "") {
          this.typedLines.push("");
          this.lineIndex += 1;
          this.typedCharIndex = 0;
          this.bootTimer = window.setTimeout(tick, 140);
          return;
        }

        if (this.typedLines.length <= this.lineIndex) this.typedLines.push("");

        const next = target.slice(0, this.typedCharIndex + 1);
        this.typedLines.splice(this.lineIndex, 1, next);
        this.typedCharIndex += 1;

        if (this.typedCharIndex >= target.length) {
          this.lineIndex += 1;
          this.typedCharIndex = 0;
          this.bootTimer = window.setTimeout(tick, 220);
          return;
        }

        const jitter = Math.floor(minDelay + Math.random() * (maxDelay - minDelay));
        const extra = Math.random() < 0.06 ? 90 : 0;
        this.bootTimer = window.setTimeout(tick, jitter + extra);
      };

      tick();
    },

    // MEMBER → set role then fade out + route
    memberLogin() {
      if (this.isFading) return;
      sessionStorage.setItem("authRole", "member");
      this.fadeAndEnter("/status");
    },

    // OFFICER/STAFF → open modal (AdminLoginModal handles credentials)
    openAdminLogin() {
      if (this.isFading) return;
      this.showAdminModal = true;
    },
    closeAdminLogin() {
      this.showAdminModal = false;
    },
    onAdminLoginSuccess() {
      // why: ensure guards see staff role immediately
      sessionStorage.setItem("authRole", "staff");
      this.showAdminModal = false;
      this.fadeAndEnter("/admin");
    },

    fadeAndEnter(targetPath) {
      this.isFading = true;

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
  min-height: 360px;
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

.typed {
  position: relative;
  z-index: 2;
  font-size: 14px;
  line-height: 1.7;
  text-shadow: 0 0 10px rgba(120, 180, 255, 0.10);
  min-height: 200px;
}

.line { margin: 2px 0; }
.dim { opacity: 0.7; }
.line.spacer { height: 10px; }

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

.login-options {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 14px;
  width: min(760px, 100%);
}

.login-option {
  background: rgba(0, 0, 0, 0.35);
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

/* Router transitions kept off intentionally to avoid route/animation race
.hud-enter-active { animation: hud-in 420ms ease-out both; }
.hud-leave-active { animation: hud-out 220ms ease-in both; }
@keyframes hud-in { 0%{opacity:0;filter:blur(2px);} 100%{opacity:1;filter:blur(0);} }
@keyframes hud-out { 0%{opacity:1;filter:blur(0);} 100%{opacity:0;filter:blur(2px);} }
*/
</style>
