<!-- src/views/admin/AdminGate.vue -->
<template>
  <div class="content-container">
    <section class="section-container" style="max-width:720px;margin:auto">
      <div class="section-header clipped-medium-backward">
        <img src="/icons/orbital.svg" alt="">
        <h1>UNSC Staff Access</h1>
      </div>

      <div class="section-content-container">
        <div class="grid">
          <div class="card">
            <h2>Member Access</h2>
            <p class="muted">View-only access to rosters and ops.</p>
            <router-link class="btn" :to="{ path: '/status' }">Enter as Member</router-link>
          </div>

          <div class="card">
            <h2>Staff Sign-in</h2>
            <label class="control"><span>Username</span><input v-model="u" type="text" autocomplete="username" /></label>
            <label class="control"><span>Password</span><input v-model="p" type="password" autocomplete="current-password" @keyup.enter="go" /></label>
            <button class="btn" :disabled="loading" @click="go">{{ loading ? 'Authorizing…' : 'Sign in' }}</button>
            <p v-if="err" class="err">{{ err }}</p>
            <p v-if="ok" class="ok">Authorized.</p>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { adminLogin } from "@/utils/adminAuth";

export default {
  name: "AdminGate",
  data: () => ({ u: "", p: "", loading: false, err: "", ok: false }),
  methods: {
    async go() {
      this.err = ""; this.ok = false;
      if (!this.u || !this.p) { this.err = "Enter username and password."; return; }
      this.loading = true;
      try {
        await adminLogin(this.u.trim(), this.p);
        this.ok = true;
        const target =
          typeof this.$route.query.redirect === "string" && this.$route.query.redirect.startsWith("/")
            ? this.$route.query.redirect
            : "/admin";
        this.$router.replace(target);
      } catch (e) {
        this.err = String(e?.message || e);
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>

<style scoped>
/* =========================
   UNSC TERMINAL ADMIN GATE
   Visual-only: NO logic changes.
   ========================= */

/* Window shell */
.section-container{
  border-radius: 16px;
  border: 1px solid rgba(170, 220, 255, 0.22);
  background: linear-gradient(180deg, rgba(8, 14, 20, 0.92), rgba(3, 6, 10, 0.95));
  box-shadow:
    0 0 0 1px rgba(170, 220, 255, 0.06) inset,
    0 0 26px rgba(120, 180, 255, 0.10),
    0 0 110px rgba(0, 0, 0, 0.6);
  overflow: hidden;
  position: relative;
  text-transform: uppercase;
  letter-spacing: 0.08em;
}

/* Scanlines + soft glow */
.section-container::before{
  content:"";
  position:absolute;
  inset:0;
  pointer-events:none;
  background: repeating-linear-gradient(
    to bottom,
    rgba(255, 255, 255, 0.02),
    rgba(255, 255, 255, 0.02) 1px,
    rgba(0, 0, 0, 0) 3px,
    rgba(0, 0, 0, 0) 6px
  );
  mix-blend-mode: overlay;
  opacity: 0.26;
  z-index: 0;
}
.section-container::after{
  content:"";
  position:absolute;
  inset:-20%;
  pointer-events:none;
  background: radial-gradient(circle at 30% 20%, rgba(120, 180, 255, 0.07), transparent 55%);
  opacity: 0.9;
  animation: gateFlicker 2.9s infinite;
  z-index: 0;
}
@keyframes gateFlicker{
  0%, 100% { transform: translate3d(0,0,0); opacity: .72; }
  12% { transform: translate3d(-1px, 1px, 0); opacity: .86; }
  24% { transform: translate3d(1px, -1px, 0); opacity: .70; }
  40% { transform: translate3d(0px, 2px, 0); opacity: .90; }
  65% { transform: translate3d(2px, 0px, 0); opacity: .78; }
}

/* Keep content above effects */
.section-header, .section-content-container{ position: relative; z-index: 1; }

/* Header chrome */
.section-header{
  display:flex;
  align-items:center;
  gap: 10px;
  padding: 12px 14px;
  border-bottom: 1px solid rgba(170, 220, 255, 0.12);
  background: rgba(0,0,0,0.16);
}
.section-header.clipped-medium-backward{
  clip-path: none !important;
  background: rgba(0,0,0,0.16) !important;
}

/* Window dots */
.section-header::before{
  content:"";
  width: 10px;
  height: 10px;
  border-radius: 999px;
  background: rgba(170, 220, 255, 0.22);
  box-shadow:
    16px 0 0 rgba(170, 220, 255, 0.22),
    32px 0 0 rgba(170, 220, 255, 0.22);
  margin-right: 8px;
  flex: 0 0 auto;
  opacity: .95;
}

.section-header img{
  width: 18px;
  height: 18px;
  opacity: .9;
  filter: drop-shadow(0 0 10px rgba(120,180,255,0.18));
}
.section-header h1{
  margin: 0;
  font-size: 12px;
  font-weight: 800;
  color: rgba(190, 230, 255, 0.92);
  letter-spacing: 0.14em;
}

/* Grid */
.grid{
  display:grid;
  grid-template-columns: 1fr 1fr;
  gap: 14px;
  padding: 16px;
}

/* Cards */
.card{
  border: 1px dashed rgba(170,220,255,0.20);
  background: rgba(0,0,0,0.22);
  border-radius: 14px;
  padding: 14px;
  display:grid;
  gap: 10px;
  box-shadow: 0 0 0 1px rgba(170,220,255,0.05) inset, 0 0 22px rgba(0,0,0,0.28);
}
.card h2{
  margin: 0;
  font-size: 12px;
  letter-spacing: 0.14em;
  color: rgba(226, 243, 255, 0.95);
}
.muted{
  margin: 0;
  color: rgba(158,197,230,0.92);
  letter-spacing: 0.06em;
  text-transform: none;
}

/* Controls */
.control{ display:grid; gap: 6px; }
.control span{
  color: rgba(158,197,230,0.92);
  font-size: 12px;
  letter-spacing: 0.10em;
}
.control input{
  background: rgba(5, 12, 20, 0.88);
  border: 1px solid rgba(170,220,255,0.22);
  border-radius: 12px;
  padding: 10px 10px;
  color: rgba(226, 243, 255, 0.95);
  box-shadow: 0 0 0 1px rgba(170,220,255,0.04) inset;
  outline: none;
}
.control input:focus{
  border-color: rgba(120,180,255,0.55);
  box-shadow: 0 0 0 2px rgba(120,180,255,0.22);
}

/* Buttons */
.btn{
  border: 1px solid rgba(170,220,255,0.22);
  background: linear-gradient(180deg, rgba(8,14,20,0.65), rgba(3,6,10,0.72));
  color: rgba(226, 243, 255, 0.95);
  border-radius: 12px;
  padding: 10px 12px;
  cursor:pointer;
  text-align:center;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  font-size: 12px;
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.06) inset,
    0 0 18px rgba(120,180,255,0.08);
}
.btn:hover{
  border-color: rgba(170,220,255,0.55);
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.08) inset,
    0 0 22px rgba(120,180,255,0.12);
}
.btn:disabled{
  opacity: .55;
  cursor: not-allowed;
}

/* Messages */
.err{ color:#ffb080; margin: 0; letter-spacing: .06em; text-transform:none; }
.ok{ color: rgba(120,255,190,0.92); margin: 0; letter-spacing: .08em; text-transform:none; }

@media (max-width:860px){
  .grid{ grid-template-columns: 1fr; padding: 14px; }
}
</style>
