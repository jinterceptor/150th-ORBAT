<!-- /src/components/layout/Header.vue -->
<template>
  <header :style="{'--auth-x': authOffsetX + 'px', '--auth-y': authOffsetY + 'px'}">
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
        <!-- PRIMARY TITLE (FIXED) -->
        <div id="title-first-line" class="title-row">
          <span id="title-header">UNSC TACNET</span>
        </div>
        <!-- SECONDARY LINE (CONFIG-DRIVEN) -->
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
        <!-- ROW 1 -->
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

        <!-- ROW 2 -->
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
</template>

<script>
import {
  adminUser,
  isAdmin,
  adminLogout,
  subscribe as authSubscribe,
} from "@/utils/adminAuth";

export default {
  props: {
    planetPath: { type: String, required: true },
    header: { type: Object, required: true },
    authOffsetX: { type: Number, default: 330 }, // default shift right
    authOffsetY: { type: Number, default: 10 },  // shift down
  },
  data() {
    return {
      role: null,          // 'member' | 'staff' | null (overlay)
      staffUser: null,     // { username, displayName } from admin session
      unsub: null,
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
  },
  created() {
    this.readAuth();
    this.unsub = authSubscribe(() => this.readAuth());
    window.addEventListener("storage", this.onStorage);
  },
  beforeUnmount() {
    if (this.unsub) this.unsub();
    window.removeEventListener("storage", this.onStorage);
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
  },
};
</script>

<style scoped>
/* =========================
   UNSC TERMINAL HEADER THEME
   Visual-only: NO template/script changes.
   ========================= */

header{
  position: relative;
  border-radius: 16px;
  border: 1px solid rgba(170, 220, 255, 0.22);
  background: linear-gradient(180deg, rgba(8,14,20,0.90), rgba(3,6,10,0.94));
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.06) inset,
    0 0 26px rgba(120,180,255,0.10),
    0 0 110px rgba(0,0,0,0.55);
  overflow: hidden;
}

/* Scanlines + subtle glow */
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

/* Keep header content above effects */
header > *{ position: relative; z-index: 1; }

/* Auth indicator -> terminal pill */
.auth-indicator{
  border: 1px solid rgba(170,220,255,0.22);
  background: rgba(0,0,0,0.28);
  color: rgba(190,230,255,0.92);
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.05) inset,
    0 0 18px rgba(120,180,255,0.10);
}
.auth-role{
  border-color: rgba(170,220,255,0.24);
  background: rgba(0,0,0,0.18);
}
.auth-role[data-variant="staff"]{
  border-color: rgba(120,255,190,0.45);
  color: rgba(120,255,190,0.92);
}
.auth-logout{
  border-color: rgba(170,220,255,0.22);
  color: rgba(190,230,255,0.92);
}
.auth-logout:hover{
  border-color: rgba(170,220,255,0.55);
  box-shadow: 0 0 0 1px rgba(170,220,255,0.08) inset;
}

/* Title block -> terminal chrome */
.title{
  border: 1px solid rgba(170,220,255,0.14);
  background: rgba(0,0,0,0.18);
  box-shadow: 0 0 0 1px rgba(170,220,255,0.05) inset;
}

/* Neutralize clipped flourish to match the new window look */
.title.clipped-x-large-forward{
  clip-path: none !important;
  border-radius: 14px;
}

/* Logo glow */
.logo{
  filter: drop-shadow(0 0 14px rgba(120,180,255,0.14));
}

/* Header typography */
#title-header{
  letter-spacing: 0.18em;
  text-transform: uppercase;
  color: rgba(226,243,255,0.96);
}
#subtitle-header{
  color: rgba(190,230,255,0.92);
  letter-spacing: 0.12em;
}
#subtitle-subheader{
  color: rgba(158,197,230,0.90);
  letter-spacing: 0.12em;
}

/* Right-side planet/loc panel -> glass */
.planet-location-container{
  border-left: 1px solid rgba(170,220,255,0.12);
}
.location-info{
  border: 1px dashed rgba(170,220,255,0.18);
  background: rgba(0,0,0,0.16);
  border-radius: 14px;
  box-shadow: 0 0 0 1px rgba(170,220,255,0.05) inset;
}

/* Labels */
.location-row h4{
  color: rgba(158,197,230,0.92);
}
.subtitle{
  color: rgba(226,243,255,0.92);
}

/* Decorative rhombus -> tone down */
.rhombus{ opacity: .18; }


header { position: relative; }

/* Auth indicator pill (position via CSS variables) */
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

/* Existing layout styles */
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
</style>
