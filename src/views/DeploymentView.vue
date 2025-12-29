<!-- src/views/DeploymentView.vue -->
<template>
  <div class="deployment">
    <div class="topbar">
      <div class="left">
        <h1 class="title">Deployment</h1>
        <div class="muted small">
          Plan chalks, assign roles/certs, and keep to the point cap.
        </div>
      </div>

      <div class="right">
        <button class="btn ghost" @click="openOverview">Overview</button>
        <button class="btn" @click="resetPlan">Reset</button>
        <button class="btn pri" @click="saveLocal" :disabled="busy">Save (Local)</button>
        <button class="btn" @click="loadLocal" :disabled="busy">Load (Local)</button>
      </div>
    </div>

    <div class="grid">
      <div class="panel">
        <div class="panel-head">
          <h2>Chalks</h2>
          <div class="panel-actions">
            <button class="btn small" @click="addUnit">Add Chalk</button>
          </div>
        </div>

        <div class="units">
          <div v-for="u in plan.units" :key="u.key" class="unit">
            <div class="unit-head">
              <div class="unit-title">
                <input v-model="u.title" class="title-input" placeholder="Unit / Chalk name" />
                <div class="muted small">Key: {{ u.key }}</div>
              </div>

              <div class="unit-actions">
                <button class="btn ghost small" @click="duplicateUnit(u.key)">Duplicate</button>
                <button class="btn danger small" @click="removeUnit(u.key)">Remove</button>
              </div>
            </div>

            <div class="unit-meta">
              <div class="muted small">
                Points: <b>{{ unitPoints(u.key) }}</b> / {{ SQUAD_POINT_CAP }}
                <span v-if="unitPoints(u.key) > SQUAD_POINT_CAP" class="warn">Over cap</span>
              </div>
              <div class="muted small">
                Filled: <b>{{ filledCount(u) }}</b> / {{ u.slots.length }}
              </div>
            </div>

            <div class="slots">
              <div v-for="(s, idx) in u.slots" :key="`${u.key}-${idx}`" class="slot">
                <div class="slot-top">
                  <div class="slot-left">
                    <div class="slot-role">
                      <input v-model="s.role" class="role-input" placeholder="Role" />
                      <span class="muted small">FT: {{ s.fireteam || "—" }}</span>
                    </div>

                    <div class="slot-person">
                      <div class="name">{{ displayName(s) }}</div>
                      <div class="muted small">
                        <span v-if="s.id">Cert: {{ s.cert || "—" }}</span>
                        <span v-else>Unassigned</span>
                        <span v-if="s.disposable" class="chip">Disposable</span>
                      </div>
                    </div>
                  </div>

                  <div class="slot-right">
                    <button class="btn small" @click="openPicker(u.key, idx)">Assign</button>
                    <button class="btn ghost small" @click="vacateSlot(u.key, idx)">Vacate</button>
                  </div>
                </div>

                <div class="slot-notes">
                  <textarea v-model="s.notes" placeholder="Notes…" />
                </div>
              </div>
            </div>

            <div class="unit-footer">
              <div class="left">
                <button class="btn small" @click="addSlot(u.key)">Add Slot</button>
                <button class="btn ghost small" @click="normalize(u.key)">Normalize</button>
              </div>
              <div class="right">
                <button class="btn" @click="clearGroup(u.key)">Clear Group</button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="panel">
        <div class="panel-head">
          <h2>Roster</h2>
          <div class="panel-actions">
            <button class="btn small" @click="refreshOrbat" :disabled="busy">Refresh</button>
          </div>
        </div>

        <div class="muted small">
          Members loaded: <b>{{ people.length }}</b>
        </div>

        <div class="roster">
          <div v-for="p in people" :key="p.id" class="person">
            <div class="p-name">{{ p.name }}</div>
            <div class="muted small">{{ p.role || "Member" }}</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Assignment Modal -->
    <div v-if="picker.open" class="squad-overlay" @click.self="closePicker">
      <div class="squad-modal compact">
        <div class="compact-header">
          <h3>Assign to {{ pickerTitle }}</h3>
          <button class="squad-close" @click="closePicker">✕</button>
        </div>

        <div class="picker-toolbar">
          <input v-model="picker.query" class="search" placeholder="Search name / callsign / role" />
          <label class="check smallish">
            <input type="checkbox" v-model="picker.onlyFree" />
            <span class="check-label">Only unassigned</span>
          </label>
          <select v-model="picker.sort" class="sort">
            <option value="assigned_last">Assigned last</option>
            <option value="assigned_first">Assigned first</option>
            <option value="name_asc">Name A–Z</option>
            <option value="name_desc">Name Z–A</option>
            <option value="rank_desc">Rank (senior→junior)</option>
            <option value="rank_asc">Rank (junior→senior)</option>
          </select>
        </div>

        <!-- ✅ this is now constrained to the modal and scrolls -->
        <div class="squad-modal-scroll compact-list">
          <div
            v-for="p in filteredAndSorted"
            :key="p.id"
            class="pick-row compact"
            :class="{ assigned: !!findAssignment(p.id) }"
          >
            <div class="row-left">
              <div class="name">
                <span class="rank">{{ p.rank || "" }}</span>
                <span class="callsign" v-if="p.callsign">[{{ p.callsign }}]</span>
                <span class="full">{{ p.name }}</span>
              </div>
              <div class="muted small">
                <span>{{ p.role || "Member" }}</span>
                <span v-if="findAssignment(p.id)" class="chip">
                  Assigned: {{ formatAssignment(findAssignment(p.id)) }}
                </span>
              </div>
            </div>

            <div class="row-right">
              <button class="btn primary small" @click.stop="selectPersonnel(p)">Select</button>
            </div>
          </div>
        </div>

        <div v-if="detailError" class="error">{{ detailError }}</div>
      </div>
    </div>

    <!-- Overview Modal -->
    <div v-if="overview.open" class="squad-overlay" @click.self="closeOverview">
      <div class="squad-modal overview">
        <div class="compact-header">
          <h3>Overview</h3>
          <button class="squad-close" @click="closeOverview">✕</button>
        </div>

        <div class="squad-modal-scroll">
          <div v-for="u in plan.units" :key="`ov-${u.key}`" class="overview-card">
            <div class="overview-top">
              <div>
                <div class="overview-title">{{ u.title || u.key }}</div>
                <div class="muted small">
                  Points: {{ unitPoints(u.key) }} / {{ SQUAD_POINT_CAP }} • Filled: {{ filledCount(u) }}
                </div>
              </div>
            </div>

            <div class="overview-body">
              <div v-for="(s, i) in u.slots" :key="`ov-${u.key}-${i}`" class="slot-row">
                <div class="slot-left">
                  <div class="slot-title">{{ s.role || `Slot ${i + 1}` }}</div>
                  <div class="muted small">{{ s.fireteam || "—" }}</div>
                </div>
                <div class="slot-right">
                  <span v-if="s.id" class="chip">{{ s.name }}</span>
                  <span v-else class="muted small">Unassigned</span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="compact-footer">
          <button class="btn" @click="closeOverview">Close</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
/* cookie reader */
function readCookie(name) {
  try {
    const target = `${name}=`;
    const parts = document.cookie ? document.cookie.split(";") : [];
    for (let i = 0; i < parts.length; i++) {
      const v = parts[i].trim();
      if (v.startsWith(target)) return decodeURIComponent(v.substring(target.length));
    }
    return "";
  } catch {
    return "";
  }
}

/* optional Netlify Identity */
async function netlifyIdentityToken() {
  try {
    const id = window.netlifyIdentity;
    if (!id) return "";
    const user = id.currentUser && id.currentUser();
    if (!user) return "";
    const jwt = await user.jwt(true);
    return jwt || "";
  } catch {
    return "";
  }
}

export default {
  name: "DeploymentView",

  data() {
    return {
      busy: false,
      detailError: "",

      apiBase: "",
      secret: "PLEX",
      deviceId: "",

      SQUAD_POINT_CAP: 100,
      DISPOSABLE_COST: 3,

      CERT_POINTS: {
        "": 0,
        Rifleman: 0,
        Autorifleman: 2,
        Marksman: 2,
        Breacher: 2,
        Grenadier: 2,
        Pilot: 3,
        RTO: 2,
        PJ: 3,
        NCO: 1,
        Officer: 1,
      },

      ROLE_ORDER: [
        "officer",
        "nco",
        "rto",
        "pj",
        "pilot",
        "autorifleman",
        "grenadier",
        "breacher",
        "marksman",
        "rifleman",
      ],

      people: [],
      plan: {
        units: [],
      },

      picker: {
        open: false,
        query: "",
        sort: "assigned_last",
        onlyFree: false,
        unitKey: "",
        slotIdx: -1,
      },

      overview: {
        open: false,
        loading: false,
        error: "",
        items: {},
      },
    };
  },

  computed: {
    pickerTitle() {
      const g = this.plan.units.find((u) => u.key === this.picker.unitKey);
      const s = g?.slots?.[this.picker.slotIdx];
      return s?.role || `Slot ${this.picker.slotIdx + 1}`;
    },

    filteredAndSorted() {
      const q = (this.picker.query || "").trim().toLowerCase();
      const onlyFree = !!this.picker.onlyFree;

      let list = Array.isArray(this.people) ? [...this.people] : [];
      if (q) {
        list = list.filter((p) => {
          const name = String(p.name || "").toLowerCase();
          const callsign = String(p.callsign || "").toLowerCase();
          const role = String(p.role || "").toLowerCase();
          return name.includes(q) || callsign.includes(q) || role.includes(q);
        });
      }

      if (onlyFree) {
        list = list.filter((p) => !this.findAssignment(p.id));
      }

      const sort = this.picker.sort || "assigned_last";
      const byName = (a, b) => String(a.name || "").localeCompare(String(b.name || ""));
      const byRankAsc = (a, b) => Number(a.rank || 0) - Number(b.rank || 0);
      const byRankDesc = (a, b) => Number(b.rank || 0) - Number(a.rank || 0);

      const assignedFirst = (a, b) => {
        const aa = !!this.findAssignment(a.id);
        const bb = !!this.findAssignment(b.id);
        if (aa === bb) return 0;
        return aa ? -1 : 1;
      };
      const assignedLast = (a, b) => {
        const aa = !!this.findAssignment(a.id);
        const bb = !!this.findAssignment(b.id);
        if (aa === bb) return 0;
        return aa ? 1 : -1;
      };

      if (sort === "name_asc") list.sort(byName);
      else if (sort === "name_desc") list.sort((a, b) => byName(b, a));
      else if (sort === "rank_asc") list.sort(byRankAsc);
      else if (sort === "rank_desc") list.sort(byRankDesc);
      else if (sort === "assigned_first") list.sort((a, b) => assignedFirst(a, b) || byName(a, b));
      else list.sort((a, b) => assignedLast(a, b) || byName(a, b));

      return list;
    },
  },

  created() {
    try {
      this.apiBase = localStorage.getItem("orbat_api_base") || "";
    } catch {}
    this.ensureDeviceId();
    this.loadLocal();
  },

  methods: {
    /* ========= basics ========= */
    ensureDeviceId() {
      try {
        const key = "orbatDeviceId";
        const existing = localStorage.getItem(key);
        if (existing && /^[a-zA-Z0-9_.-]{8,}$/.test(existing)) {
          this.deviceId = existing;
          return;
        }
        const id = this.makeDeviceId();
        localStorage.setItem(key, id);
        this.deviceId = id;
      } catch {
        this.deviceId = this.makeDeviceId();
      }
    },

    makeDeviceId() {
      const r =
        crypto && crypto.getRandomValues
          ? crypto.getRandomValues(new Uint8Array(12))
          : Array.from({ length: 12 }, () => Math.floor(Math.random() * 256));
      return Array.from(r).map((b) => b.toString(16).padStart(2, "0")).join("");
    },

    normalizeRole(role) {
      const t = String(role || "").trim().toLowerCase();
      if (!t) return "";
      if (/\bofficer\b/.test(t)) return "officer";
      if (/\bnco\b/.test(t)) return "nco";
      if (/\brto\b/.test(t)) return "rto";
      if (/\bpj\b/.test(t)) return "pj";
      if (/\bpilot\b/.test(t)) return "pilot";
      if (/\bautorifle/.test(t)) return "autorifleman";
      if (/\bgrenad/.test(t)) return "grenadier";
      if (/\bbreach/.test(t)) return "breacher";
      if (/\bmarks/.test(t)) return "marksman";
      if (/\brifle/.test(t)) return "rifleman";
      return t;
    },

    rolePriority(role) {
      const key = this.normalizeRole(role);
      const idx = this.ROLE_ORDER.indexOf(key);
      return idx === -1 ? 10000 : idx;
    },

    sortSlotsByRole(slots) {
      return slots
        .map((s, i) => ({ s, i, p: this.rolePriority(s.role) }))
        .sort((a, b) => a.p - b.p || a.i - b.i)
        .map((x) => x.s);
    },

    padSlots(arr, min) {
      const out = arr.slice();
      while (out.length < min)
        out.push({ id: null, name: null, role: "", origStatus: "VACANT", cert: "", disposable: false });
      return out;
    },

    displayName(slot) {
      return slot.name || (slot.origStatus === "VACANT" ? "— Vacant —" : "— Empty —");
    },

    filledCount(g) {
      if (!g) return 0;
      return g.slots.reduce((n, s) => n + (s.id ? 1 : 0), 0);
    },

    unitPoints(unitKey) {
      const g = this.plan.units.find((u) => u.key === unitKey);
      if (!g) return 0;
      return g.slots.reduce((sum, s) => {
        const certPts = this.CERT_POINTS[s.cert] ?? 0;
        const disp = s.disposable ? this.DISPOSABLE_COST : 0;
        return sum + certPts + disp;
      }, 0);
    },

    wouldExceedCap(unitKey, delta) {
      return this.unitPoints(unitKey) + Number(delta || 0) > this.SQUAD_POINT_CAP;
    },

    /* ========= persistence ========= */
    persistPlan() {
      try {
        localStorage.setItem("deploy_plan", JSON.stringify(this.plan));
      } catch {}
    },

    saveLocal() {
      this.persistPlan();
    },

    loadLocal() {
      try {
        const raw = localStorage.getItem("deploy_plan");
        if (!raw) return;
        const parsed = JSON.parse(raw);
        if (parsed && Array.isArray(parsed.units)) this.plan = parsed;
      } catch {}
    },

    resetPlan() {
      this.plan = { units: [] };
      this.persistPlan();
    },

    /* ========= units / slots ========= */
    addUnit() {
      const idx = this.plan.units.length + 1;
      this.plan.units.push({
        key: `chalk_${idx}`,
        title: `Chalk ${idx}`,
        slots: this.padSlots([], 8).map((s, i) => ({ ...s, role: i === 0 ? "Officer" : "Rifleman", fireteam: "" })),
        fireteams: [],
      });
      this.persistPlan();
    },

    duplicateUnit(key) {
      const g = this.plan.units.find((u) => u.key === key);
      if (!g) return;
      const idx = this.plan.units.length + 1;
      const copy = JSON.parse(JSON.stringify(g));
      copy.key = `${g.key}_copy_${idx}`;
      copy.title = `${g.title || g.key} (Copy)`;
      this.plan.units.push(copy);
      this.persistPlan();
    },

    removeUnit(key) {
      this.plan.units = this.plan.units.filter((u) => u.key !== key);
      this.persistPlan();
    },

    addSlot(key) {
      const g = this.plan.units.find((u) => u.key === key);
      if (!g) return;
      g.slots.push({ id: null, name: null, role: "Rifleman", fireteam: "", origStatus: "VACANT", cert: "", disposable: false });
      this.persistPlan();
    },

    normalize(key) {
      const g = this.plan.units.find((u) => u.key === key);
      if (!g) return;
      g.slots = this.sortSlotsByRole(g.slots.map((s) => ({ ...s, role: s.role || "Rifleman" })));
      this.persistPlan();
    },

    vacateSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];

      const slots = g.slots.slice();
      const prev = slots[slotIdx];
      if (!prev) return;

      slots[slotIdx] = { ...prev, id: null, name: null, cert: "", disposable: false, origStatus: "VACANT" };

      const newG = { ...g, slots: this.sortSlotsByRole(slots) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    clearGroup(unitKey) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx < 0) return;

      const g = this.plan.units[idx];
      const emptied = g.slots.map((s) => ({
        ...s,
        id: null,
        name: null,
        cert: "",
        disposable: false,
        origStatus: "VACANT",
      }));

      const newG = { ...g, slots: this.sortSlotsByRole(emptied) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    /* ========= picker ========= */
    openPicker(unitKey, slotIdx) {
      this.picker.open = true;
      this.picker.unitKey = unitKey;
      this.picker.slotIdx = slotIdx;
      this.picker.query = "";
      this.picker.onlyFree = false;
      this.picker.sort = "assigned_last";
      this.detailError = "";
    },

    closePicker() {
      this.picker.open = false;
      this.picker.unitKey = "";
      this.picker.slotIdx = -1;
      this.detailError = "";
    },

    findAssignment(personId) {
      if (!personId) return null;
      for (const g of this.plan.units) {
        for (let i = 0; i < g.slots.length; i++) {
          if (g.slots[i]?.id === personId) return { unitKey: g.key, slotIdx: i, title: g.title || g.key };
        }
      }
      return null;
    },

    formatAssignment(a) {
      return `${a.title} #${a.slotIdx + 1}`;
    },

    /* ✅ FIXED: removed the stray duplicated block that broke builds */
    selectPersonnel(p) {
      if (!this.picker.open) return;

      const from = this.findAssignment(p.id);
      const gIdx = this.plan.units.findIndex((u) => u.key === this.picker.unitKey);
      if (gIdx < 0) return;

      const g = this.plan.units[gIdx];
      const target = g.slots[this.picker.slotIdx];

      const chosenCertDefault = target.cert || p.role || "";

      const prevPts = (this.CERT_POINTS[target.cert] ?? 0) + (target.disposable ? this.DISPOSABLE_COST : 0);
      const nextPts = this.CERT_POINTS[chosenCertDefault] ?? 0;
      const delta = nextPts - prevPts;

      if (this.wouldExceedCap(g.key, Math.max(0, delta))) {
        this.detailError = `Point cap ( ${this.SQUAD_POINT_CAP} ) would be exceeded.`;
        return;
      }
      this.detailError = "";

      if (from && target?.id && !(from.unitKey === g.key && from.slotIdx === this.picker.slotIdx)) {
        const srcIdx = this.plan.units.findIndex((u) => u.key === from.unitKey);
        const srcGroup = this.plan.units[srcIdx];

        const tmp = { ...target };
        const newTarget = { ...target, id: p.id, name: p.name || null, cert: chosenCertDefault, disposable: false };

        const newSrcSlots = srcGroup.slots.slice();
        newSrcSlots[from.slotIdx] = { ...newSrcSlots[from.slotIdx], id: tmp.id, name: tmp.name, cert: tmp.cert, disposable: !!tmp.disposable };

        const newSrc = { ...srcGroup, slots: this.sortSlotsByRole(newSrcSlots) };

        const newGSlots = g.slots.slice();
        newGSlots[this.picker.slotIdx] = newTarget;
        const newG = { ...g, slots: this.sortSlotsByRole(newGSlots) };

        this.plan.units = this.plan.units.map((u, i) => (i === gIdx ? newG : i === srcIdx ? newSrc : u));
      } else {
        const newSlots = g.slots.slice();
        newSlots[this.picker.slotIdx] = { ...target, id: p.id, name: p.name || null, cert: chosenCertDefault, disposable: false };
        const newG = { ...g, slots: this.sortSlotsByRole(newSlots) };
        this.plan.units = this.plan.units.map((u, i) => (i === gIdx ? newG : u));
      }

      this.persistPlan();
      this.closePicker();
    },

    /* ========= overview ========= */
    openOverview() {
      this.overview.open = true;
    },
    closeOverview() {
      this.overview.open = false;
    },

    /* ========= stubs for your existing integration ========= */
    async refreshOrbat() {
      this.busy = true;
      try {
        // keep your existing logic here (this file is a paste-friendly single-file view)
      } finally {
        this.busy = false;
      }
    },

    async apiPost(action, body, raw = false) {
      if (!this.apiBase) throw new Error("execUrl missing");

      const ls = typeof localStorage !== "undefined" ? localStorage : null;
      const ss = typeof sessionStorage !== "undefined" ? sessionStorage : null;

      const readJson = (s) => {
        try {
          return s ? JSON.parse(s) : null;
        } catch {
          return null;
        }
      };
      const readStr = (k) => String(ss?.getItem(k) || ls?.getItem(k) || "").trim();

      const adminUser = readJson(ss?.getItem("admin:user") || "");
      const adminRole = (readStr("admin:role") || readStr("authRole") || String(adminUser?.role || "")).trim();
      const adminTokenRaw = (readStr("admin:token") || readStr("authToken") || "").trim();
      const adminAuth = adminTokenRaw ? (adminTokenRaw.toLowerCase().startsWith("bearer ") ? adminTokenRaw : `Bearer ${adminTokenRaw}`) : "";

      const cookieToken = readCookie("orbat_token");
      const legacyAuth = cookieToken ? `Bearer ${cookieToken}` : "";

      let niAuth = "";
      try {
        const jwt = await netlifyIdentityToken();
        if (jwt) niAuth = `Bearer ${jwt}`;
      } catch {}

      const authHeader = adminAuth || legacyAuth || niAuth;

      const payload = {
        secret: this.secret || "PLEX",
        action,
        deviceId: this.deviceId,
        ...body,
      };

      const urlObj = new URL(this.apiBase, typeof window !== "undefined" ? window.location.origin : "http://localhost");
      const url = urlObj.toString();

      const res = await fetch(url, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          ...(authHeader ? { Authorization: authHeader } : {}),
          ...(adminRole ? { "X-Role": adminRole } : {}),
        },
        body: JSON.stringify(payload),
        credentials: "include",
      });

      if (raw) return res;
      const data = await res.json().catch(() => null);
      return data;
    },
  },
};
</script>

<style scoped>
.deployment{display:flex;flex-direction:column;gap:1rem;padding:1.2rem;color:#dce6f1}
.topbar{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start;flex-wrap:wrap}
.title{margin:0 0 .25rem;font-size:1.6rem}
.muted{color:rgba(220,230,241,0.72)}
.small{font-size:.86rem}
.smallish{font-size:.92rem}

.grid{display:grid;grid-template-columns:1.4fr .6fr;gap:1rem}
@media (max-width:1100px){.grid{grid-template-columns:1fr}}

.panel{background:rgba(0,0,0,0.24);border:1px solid rgba(255,255,255,0.12);border-radius:1rem;padding:1rem;box-shadow:0 0 18px rgba(0,0,0,0.35)}
.panel-head{display:flex;align-items:center;justify-content:space-between;margin-bottom:.7rem}
.panel-head h2{margin:0;font-size:1.15rem}
.panel-actions{display:flex;gap:.5rem;flex-wrap:wrap}

.btn{appearance:none;border:1px solid rgba(220,230,241,0.18);background:rgba(0,0,0,0.3);color:#dce6f1;border-radius:.65rem;padding:.55rem .8rem;cursor:pointer}
.btn:hover{border-color:rgba(220,230,241,0.35)}
.btn.small{padding:.35rem .6rem;border-radius:.55rem;font-size:.9rem}
.btn.pri,.btn.primary{background:rgba(30,144,255,0.24);border-color:rgba(30,144,255,0.55)}
.btn.pri:hover,.btn.primary:hover{border-color:rgba(30,144,255,0.85)}
.btn.ghost{background:transparent}
.btn.danger{background:rgba(255,90,90,0.15);border-color:rgba(255,90,90,0.55)}
.btn.danger:hover{border-color:rgba(255,90,90,0.9)}

.units{display:flex;flex-direction:column;gap:1rem}
.unit{background:rgba(0,0,0,0.25);border:1px solid rgba(255,255,255,0.12);border-radius:1rem;padding:1rem}
.unit-head{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start;flex-wrap:wrap}
.title-input{font-size:1.05rem;font-weight:700;background:transparent;border:1px solid rgba(255,255,255,0.12);color:#dce6f1;border-radius:.65rem;padding:.5rem .6rem;min-width:240px}
.unit-actions{display:flex;gap:.5rem;flex-wrap:wrap}
.unit-meta{display:flex;gap:1rem;flex-wrap:wrap;margin-top:.6rem}
.warn{color:rgba(255,190,80,0.95)}

.slots{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:.85rem;margin-top:.85rem}
@media (max-width:980px){.slots{grid-template-columns:1fr}}
.slot{background:rgba(0,0,0,0.22);border:1px solid rgba(255,255,255,0.1);border-radius:1rem;padding:.8rem;display:flex;flex-direction:column;gap:.6rem}
.slot-top{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start}
.slot-left{display:flex;flex-direction:column;gap:.45rem}
.slot-role{display:flex;gap:.6rem;align-items:center;flex-wrap:wrap}
.role-input{background:transparent;border:1px solid rgba(255,255,255,0.12);color:#dce6f1;border-radius:.65rem;padding:.45rem .55rem;min-width:200px}
.slot-person .name{font-weight:700}
.slot-right{display:flex;gap:.5rem;flex-wrap:wrap}
.slot-notes textarea{width:100%;min-height:64px;resize:vertical;background:rgba(0,0,0,0.25);border:1px solid rgba(255,255,255,0.12);border-radius:.8rem;color:#dce6f1;padding:.55rem .65rem}
.unit-footer{display:flex;justify-content:space-between;gap:1rem;align-items:center;flex-wrap:wrap;margin-top:.8rem}
.chip{display:inline-flex;align-items:center;gap:.35rem;border-radius:999px;border:1px solid rgba(255,255,255,0.14);padding:.2rem .55rem;background:rgba(0,0,0,0.25);font-size:.85rem}

.roster{display:flex;flex-direction:column;gap:.55rem;margin-top:.7rem;max-height:60vh;overflow:auto;padding-right:.25rem}
.person{border:1px solid rgba(255,255,255,0.1);border-radius:.85rem;padding:.55rem .6rem;background:rgba(0,0,0,0.16)}
.p-name{font-weight:800}

/* ===== Modals (FIXED) ===== */
.squad-overlay{position:fixed;inset:0;background:rgba(0,0,0,0.85);z-index:9999;display:flex;align-items:center;justify-content:center;padding:1rem;box-sizing:border-box;overflow:auto}
@media (max-height:740px){.squad-overlay{align-items:flex-start}}
.squad-modal{background-color:#050811;color:#dce6f1;width:95vw;max-width:1200px;max-height:90vh;max-height:calc(100dvh - 2rem - env(safe-area-inset-top,0px) - env(safe-area-inset-bottom,0px));border-radius:.8rem;box-shadow:0 0 24px rgba(0,0,0,0.9);padding:1.1rem 1.2rem 1.2rem;display:flex;flex-direction:column;overflow:hidden}
.squad-modal.compact{max-width:1000px}
.squad-modal.overview{max-width:1400px}

.squad-close{background:transparent;border:1px solid rgba(220,230,241,0.18);color:#dce6f1;border-radius:999px;padding:.2rem .75rem;font-size:1rem;cursor:pointer}
.squad-close:hover{border-color:rgba(220,230,241,0.35)}

.compact-header{display:flex;justify-content:space-between;gap:1rem;align-items:center;margin-bottom:.6rem}
.picker-toolbar{display:flex;gap:.6rem;align-items:center;flex-wrap:wrap;margin-bottom:.6rem}
.picker-toolbar .search{flex:1 1 300px;min-width:240px;background:rgba(0,0,0,0.35);border:1px solid rgba(255,255,255,0.12);color:#dce6f1;border-radius:.65rem;padding:.5rem .6rem}
.picker-toolbar .sort{background:rgba(0,0,0,0.35);border:1px solid rgba(255,255,255,0.12);color:#dce6f1;border-radius:.65rem;padding:.5rem .6rem}
.check{display:flex;align-items:center;gap:.45rem;cursor:pointer}
.check input{accent-color:#1e90ff}

.squad-modal-scroll{flex:1 1 auto;min-height:0;overflow-y:auto;overflow-x:hidden;overscroll-behavior:contain;scrollbar-gutter:stable both-edges;padding-right:.35rem}
.compact-list{padding-right:.15rem}
.pick-row{display:flex;justify-content:space-between;gap:1rem;align-items:center;padding:.6rem .6rem;border:1px solid rgba(255,255,255,0.1);border-radius:.85rem;margin-bottom:.55rem;background:rgba(0,0,0,0.18)}
.pick-row.assigned{border-color:rgba(30,144,255,0.35);background:rgba(30,144,255,0.08)}
.pick-row.compact .name{display:flex;gap:.45rem;align-items:center;flex-wrap:wrap}
.row-left{display:flex;flex-direction:column;gap:.15rem}
.row-right{display:flex;gap:.5rem;align-items:center}
.callsign{opacity:.85}
.rank{opacity:.8}

.overview-card{border:1px solid rgba(255,255,255,0.12);border-radius:1rem;background:rgba(0,0,0,0.22);padding:.9rem;display:flex;flex-direction:column;gap:.65rem;margin-bottom:1rem}
.overview-top{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start;flex-wrap:wrap}
.overview-title{font-weight:800}
.overview-body{display:flex;flex-direction:column;gap:.55rem}
.slot-row{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start;border:1px solid rgba(255,255,255,0.1);border-radius:.85rem;padding:.55rem .6rem;background:rgba(0,0,0,0.16)}
.slot-left{display:flex;flex-direction:column;gap:.15rem}
.slot-right{display:flex;align-items:center;gap:.4rem}
.compact-footer{display:flex;justify-content:flex-end;gap:.6rem;align-items:center;margin-top:.8rem}

.error{margin-top:.6rem;border:1px solid rgba(255,90,90,0.55);background:rgba(255,90,90,0.14);border-radius:.85rem;padding:.55rem .65rem}
</style>
