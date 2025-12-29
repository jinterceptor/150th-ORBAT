<!-- File: src/views/DeploymentView.vue -->
<template>
  <div id="deploymentView" :class="{ animate: animateView }" :style="{ 'animation-delay': animationDelay }">
    <section class="deployment-window section-container">
      <div class="header-shell">
        <div class="section-header">
          <h2>DEPLOYMENT PLANNER</h2>
          <div class="top-actions">
            <button type="button" class="btn ghost" @click="openOverview">Overview</button>
          </div>
        </div>
      </div>

      <div class="deploy-scroll">
        <div class="section-content-container">
          <div class="panel">
            <div class="toolbar">
              <div class="toolbar-left">
                <label class="label">Chalk</label>
                <select v-model="detailKey" class="select">
                  <option v-for="u in plan.units" :key="u.key" :value="u.key">{{ u.title }}</option>
                </select>

                <button type="button" class="btn" @click="fillFromRoster(detailKey)" :disabled="busy">
                  Reset from ORBAT
                </button>
              </div>

              <div class="toolbar-right">
                <span class="muted small">{{ authModeLabel }}</span>
                <span v-if="detailError" class="error">{{ detailError }}</span>
                <span v-if="apiError" class="error">{{ apiError }}</span>
              </div>
            </div>

            <div v-if="currentUnit" class="unit-shell">
              <div class="unit-head">
                <div class="unit-title">
                  <span class="unit-badge">{{ squadInitials(currentUnit.title) }}</span>
                  <div class="unit-text">
                    <h3>{{ currentUnit.title }}</h3>
                    <div class="unit-sub">
                      <span class="muted small">Filled: {{ filledCount(currentUnit) }} / {{ currentUnit.slots.length }}</span>
                      <span v-if="isPointsUnit(currentUnit.title)" class="muted small">
                        | Points: {{ calcUnitPoints(currentUnit) }} / {{ SQUAD_POINT_CAP }}
                      </span>
                    </div>
                  </div>
                </div>

                <div class="unit-tools">
                  <button type="button" class="btn ghost" @click="clearGroup(detailKey)">Clear chalk</button>
                </div>
              </div>

              <div class="squad-members-grid">
                <template v-for="(slot, sIdx) in currentUnit.slots" :key="`wrap-${detailKey}-${sIdx}`">
                  <div
                    v-if="isFireteamHeader(slot, sIdx, currentUnit.slots)"
                    :key="`ft-${detailKey}-${sIdx}`"
                    class="fireteam-row"
                  >
                    <span class="fireteam-title">{{ fireteamLabel(slot, sIdx, currentUnit.slots) }}</span>
                  </div>

                  <div
                    :key="`slot-${detailKey}-${sIdx}`"
                    class="member-card"
                    :class="{ vacant: slot.origStatus === 'VACANT' && !slot.id, closed: slot.origStatus === 'CLOSED' }"
                  >
                    <!-- VACANT / CLOSED -->
                    <template v-if="(slot.origStatus === 'VACANT' && !slot.id) || slot.origStatus === 'CLOSED'">
                      <div class="member-header">
                        <div class="member-id">
                          <span class="member-status">{{ slot.origStatus === 'CLOSED' ? 'CLOSED' : 'VACANT' }}</span>
                        </div>
                        <div class="member-actions">
                          <button
                            v-if="slot.origStatus !== 'CLOSED'"
                            type="button"
                            class="btn small"
                            @click.stop="closeSlot(detailKey, sIdx)"
                            title="Close slot"
                          >
                            Close
                          </button>
                          <button
                            v-if="slot.origStatus === 'CLOSED'"
                            type="button"
                            class="btn small"
                            @click.stop="openSlot(detailKey, sIdx)"
                            title="Re-open slot"
                          >
                            Open
                          </button>
                          <button
                            type="button"
                            class="btn small"
                            @click.stop="removeSlot(detailKey, sIdx)"
                            title="Remove slot"
                          >
                            Remove
                          </button>
                        </div>
                      </div>

                      <div class="member-body">
                        <div class="member-row">
                          <span class="muted">Role</span>
                          <span class="value">{{ slot.role || '—' }}</span>
                        </div>

                        <div class="member-row">
                          <span class="muted">Cert</span>
                          <span class="value">{{ slot.cert || '—' }}</span>
                        </div>

                        <div class="member-row">
                          <span class="muted">Disposable</span>
                          <span class="value">{{ slot.disposable ? 'Yes' : 'No' }}</span>
                        </div>
                      </div>

                      <div class="member-footer">
                        <button
                          type="button"
                          class="btn primary small"
                          :disabled="slot.origStatus === 'CLOSED'"
                          @click.stop="openPicker(detailKey, sIdx)"
                        >
                          {{ slot.origStatus === 'CLOSED' ? 'Closed' : 'Assign' }}
                        </button>
                        <span>UNSC SYSTEMS DATABASE</span>
                      </div>
                    </template>

                    <!-- FILLED -->
                    <template v-else>
                      <div class="member-header">
                        <div class="member-id">
                          <img
                            v-if="rankLabel(getRankForPersonId(slot.id))"
                            class="rank-icon"
                            :src="rankIcon(rankLabel(getRankForPersonId(slot.id)))"
                            :alt="rankLabel(getRankForPersonId(slot.id))"
                            :title="rankLabel(getRankForPersonId(slot.id))"
                            @error="onRankImgError($event)"
                          />
                          <div class="name-block">
                            <div class="name">{{ (slot.name || 'UNKNOWN').toUpperCase() }}</div>
                            <div class="sub muted">ID: {{ slot.id }}</div>
                          </div>
                        </div>

                        <div class="member-actions">
                          <button type="button" class="btn small" @click.stop="clearSlot(detailKey, sIdx)" title="Unassign">
                            Clear
                          </button>
                        </div>
                      </div>

                      <div class="member-body filled">
                        <div class="member-columns">
                          <div class="member-column left">
                            <p><strong>Role:</strong></p>
                            <p class="role-big">{{ slot.role || '—' }}</p>

                            <p><strong>Cert:</strong></p>
                            <div class="cert-row-inline">
                              <select class="select small" v-model="slot.cert" @change="setSlotCert(detailKey, sIdx, slot.cert)">
                                <option v-for="c in certOptionsForId(slot.id)" :key="c" :value="c">
                                  {{ c }}{{ certPointSuffix(c) }}
                                </option>
                              </select>
                            </div>

                            <p><strong>Disposable:</strong></p>
                            <label class="check">
                              <input
                                type="checkbox"
                                :checked="!!slot.disposable"
                                @change="toggleDisposable(detailKey, sIdx, $event.target.checked)"
                              />
                              <span class="check-label">Carry disposable (+{{ DISPOSABLE_COST }})</span>
                            </label>
                          </div>

                          <div class="member-column right">
                            <p><strong>Certifications:</strong></p>
                            <div class="cert-list">
                              <div v-for="(label, cidx) in certLabels" :key="label" class="cert-row">
                                <span
                                  class="cert-checkbox"
                                  :class="{ checked: (slot.cert || '') === label || hasCertId(slot.id, cidx) }"
                                >
                                  <span v-if="(slot.cert || '') === label || hasCertId(slot.id, cidx)" class="checkbox-dot"></span>
                                </span>
                                <span class="cert-label">{{ label }}</span>
                              </div>
                            </div>
                          </div>
                        </div>
                      </div>

                      <div class="member-footer">
                        <button
                          type="button"
                          class="btn primary small"
                          :disabled="slot.origStatus === 'CLOSED'"
                          @click.stop="openPicker(detailKey, sIdx)"
                        >
                          {{ slot.id ? 'Swap' : (slot.origStatus === 'CLOSED' ? 'Closed' : 'Assign') }}
                        </button>
                        <span>UNSC SYSTEMS DATABASE</span>
                      </div>
                    </template>
                  </div>
                </template>
              </div>

              <div class="actions-row">
                <button type="button" class="btn ghost" @click.stop="addSlot(detailKey)">Add slot</button>

                <span class="divider" />

                <button type="button" class="btn primary" :disabled="busy || !apiBase" @click="saveRemote(detailKey)">
                  {{ busy ? 'Saving…' : 'Save Chalk (Remote)' }}
                </button>

                <button type="button" class="btn" :disabled="busy || !apiBase" @click="loadRemote(detailKey)">
                  Load Chalk (Remote)
                </button>

                <span v-if="remoteMeta[detailKey]?.savedBy" class="muted small">
                  Last saved by {{ remoteMeta[detailKey].savedBy }}
                </span>
              </div>
            </div>

            <div v-else class="muted">No chalk selected.</div>
          </div>
        </div>
      </div>
    </section>

    <!-- Assign/Swap Picker (compact) -->
    <div v-if="picker.open" class="squad-overlay" @click.self="closePicker">
      <div class="squad-modal compact">
        <div class="compact-header">
          <h3>Assign to {{ currentSlotTitle }}</h3>
          <button class="squad-close" @click="closePicker">✕</button>
        </div>

        <div class="picker-toolbar">
          <input v-model="picker.query" placeholder="Search by name / callsign / role" class="search" @keydown.stop />
          <label class="check smallish">
            <input type="checkbox" v-model="picker.onlyFree" />
            <span class="check-label">Only unassigned</span>
          </label>
          <select v-model="picker.sort" class="sort">
            <option value="name_asc">Name A–Z</option>
            <option value="name_desc">Name Z–A</option>
            <option value="role_asc">Role A–Z</option>
            <option value="rank_desc">Rank (senior→junior)</option>
            <option value="rank_asc">Rank (junior→senior)</option>
            <option value="assigned_first">Assigned first</option>
            <option value="assigned_last">Assigned last</option>
          </select>
        </div>

        <div class="squad-modal-scroll">
          <div
            v-for="p in filteredAndSortedPersonnel"
            :key="p.id"
            class="pick-row compact"
            :class="{ assigned: !!findAssignment(p.id) }"
          >
            <div class="row-left">
              <img
                v-if="rankLabel(p.rank)"
                class="rank-icon small"
                :src="rankIcon(rankLabel(p.rank))"
                :alt="rankLabel(p.rank)"
                :title="rankLabel(p.rank)"
                @error="onRankImgError($event)"
              />
              <span class="name">{{ (p.name || 'UNKNOWN').toUpperCase() }}</span>
            </div>
            <div class="row-mid">
              <span class="role">{{ p.role || '—' }}</span>
              <span v-if="p.element" class="element">Element: {{ p.element }}</span>
              <span v-if="findAssignment(p.id)" class="chip">Assigned: {{ formatAssignment(findAssignment(p.id)) }}</span>
            </div>
            <div class="row-right">
              <button type="button" class="btn primary small" @click.stop="selectPersonnel(p)">Select</button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Overview Modal -->
    <div v-if="overview.open" class="squad-overlay" @click.self="closeOverview">
      <div class="squad-modal overview">
        <div class="compact-header">
          <h3>Chalk Loadouts Overview</h3>
          <div style="display:flex; gap:.5rem;">
            <button type="button" class="btn" @click="closeOverview">Close</button>
          </div>
        </div>

        <div class="squad-modal-scroll">
          <div class="overview-grid">
            <div v-for="u in plan.units" :key="u.key" class="overview-card">
              <div class="overview-head">
                <div class="overview-title">{{ u.title }}</div>
                <div class="overview-sub muted small">
                  Filled: {{ filledCount(u) }} / {{ u.slots.length }}
                  <span v-if="isPointsUnit(u.title)"> | Points: {{ calcUnitPoints(u) }}/{{ SQUAD_POINT_CAP }}</span>
                </div>
              </div>

              <div class="overview-slots">
                <div
                  v-for="(s, idx) in (ovItem(u.key) && ovItem(u.key).slots?.length ? ovItem(u.key).slots : u.slots)"
                  :key="idx"
                  class="overview-slot"
                >
                  <div class="slot-left">
                    <span class="slot-name">{{ s.name || 'VACANT' }}</span>
                    <span class="slot-id" v-if="s.id">ID: {{ s.id }}</span>
                  </div>
                  <div class="slot-right">
                    <span class="slot-role">{{ s.role || '—' }}</span>
                    <span class="slot-cert" v-if="s.cert">{{ s.cert }}</span>
                    <span class="slot-disp" v-if="s.disposable">Disp</span>
                  </div>
                </div>

                <div v-if="!((ovItem(u.key) && ovItem(u.key).slots?.length) || u.slots?.length)" class="overview-empty">—</div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script>
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

async function netlifyIdentityToken() {
  try {
    const id = window.netlifyIdentity;
    if (!id) return "";
    const user = id.currentUser();
    if (!user) return "";
    const token = await user.jwt();
    return token || "";
  } catch {
    return "";
  }
}

export default {
  name: "DeploymentView",
  props: {
    animate: { type: Boolean, default: true },
    orbat: { type: Array, default: () => [] },
    execUrl: { type: String, default: "" },
    secret: { type: String, default: "PLEX" },
    token: { type: String, default: "" },
    defaultsCsvUrl: { type: String, default: "" },
    rankIconBase: {
      type: String,
      default: () =>
        typeof window !== "undefined" && window.RANK_ICON_BASE ? window.RANK_ICON_BASE : "/ranks",
    },
    rankIconExt: { type: String, default: "png" },
  },
  data() {
    return {
      animateView: false,
      animationDelay: "0ms",
      detailKey: "",
      detailError: "",
      apiError: "",
      busy: false,
      plan: { units: [] },
      picker: { open: false, unitKey: "", slotIdx: -1, query: "", onlyFree: false, sort: "name_asc" },
      personnel: [],
      STORAGE_KEY: "deploymentPlan2",
      versions: {},
      remoteMeta: {},
      deviceId: "",
      debugInfo: "",
      MIN_CHALK_SLOTS: 12,
      ROLE_ORDER: ["squad lead", "team leader", "corpsman 1", "corpsman 2"],
      SQUAD_POINT_CAP: 10,
      DISPOSABLE_COST: 1,
      CERT_POINTS: {
        Rifleman: 0,
        Grenadier: 1,
        Breacher: 1,
        Marksman: 1,
        Autorifleman: 1,
        Corpsman: 1,
        RTO: 1,
        Engineer: 1,
        Pilot: 0,
        Crew: 0,
      },
      overview: { open: false },
      rankImgFailed: {},
    };
  },
  computed: {
    apiBase() {
      const base = String(this.execUrl || "").trim();
      return base;
    },
    currentUnit() {
      return this.plan.units.find((u) => u.key === this.detailKey) || null;
    },
    currentSlotTitle() {
      try {
        const g = this.plan.units.find((u) => u.key === this.picker.unitKey);
        const slot = g?.slots?.[this.picker.slotIdx];
        if (!g || !slot) return "slot";
        const role = slot.role ? ` (${slot.role})` : "";
        return `${g.title} #${this.picker.slotIdx + 1}${role}`;
      } catch {
        return "slot";
      }
    },
    authModeLabel() {
      const explicitToken = String(this.token || "").trim();
      const adminToken = String(localStorage?.getItem("admin:token") || "").trim();
      const hasCookie = !!readCookie("nf_jwt");
      const mode =
        explicitToken ? "token prop"
        : adminToken ? "admin token"
        : hasCookie ? "netlify cookie"
        : "public";
      return `Auth: ${mode}`;
    },
    certLabels() {
      return Object.keys(this.CERT_POINTS || {});
    },
    filteredAndSortedPersonnel() {
      const q = String(this.picker.query || "").trim().toLowerCase();
      const onlyFree = !!this.picker.onlyFree;

      const filtered = (this.personnel || []).filter((p) => {
        if (!p) return false;
        if (onlyFree && this.findAssignment(p.id)) return false;
        if (!q) return true;
        const hay = `${p.name || ""} ${p.callsign || ""} ${p.role || ""} ${p.rank || ""}`.toLowerCase();
        return hay.includes(q);
      });

      const sort = String(this.picker.sort || "name_asc");
      const byName = (a, b) =>
        String(a?.name || "").localeCompare(String(b?.name || ""), undefined, { sensitivity: "base" });
      const byRole = (a, b) =>
        String(a?.role || "").localeCompare(String(b?.role || ""), undefined, { sensitivity: "base" });

      const rankScore = (r) => {
        const label = this.rankLabel(r);
        const order = [
          "O-10",
          "O-9",
          "O-8",
          "O-7",
          "O-6",
          "O-5",
          "O-4",
          "O-3",
          "O-2",
          "O-1",
          "W-5",
          "W-4",
          "W-3",
          "W-2",
          "W-1",
          "E-9",
          "E-8",
          "E-7",
          "E-6",
          "E-5",
          "E-4",
          "E-3",
          "E-2",
          "E-1",
        ];
        const idx = order.indexOf(label);
        return idx === -1 ? 999 : idx;
      };

      const byRankDesc = (a, b) => rankScore(a.rank) - rankScore(b.rank);
      const byRankAsc = (a, b) => rankScore(b.rank) - rankScore(a.rank);

      const byAssignedFirst = (a, b) => (this.findAssignment(a.id) ? 0 : 1) - (this.findAssignment(b.id) ? 0 : 1) || byName(a, b);
      const byAssignedLast = (a, b) => (this.findAssignment(a.id) ? 1 : 0) - (this.findAssignment(b.id) ? 1 : 0) || byName(a, b);

      const arr = filtered.slice();
      arr.sort((a, b) => {
        switch (sort) {
          case "name_desc":
            return -byName(a, b);
          case "role_asc":
            return byRole(a, b) || byName(a, b);
          case "rank_desc":
            return byRankDesc(a, b) || byName(a, b);
          case "rank_asc":
            return byRankAsc(a, b) || byName(a, b);
          case "assigned_first":
            return byAssignedFirst(a, b);
          case "assigned_last":
            return byAssignedLast(a, b);
          case "name_asc":
          default:
            return byName(a, b);
        }
      });
      return arr;
    },
  },
  created() {
    this.animateView = !!this.animate;
    this.ensureDeviceId();
    this.personnel = this.buildPersonnelPool(this.orbat || []);
    this.ensureUnitsBuilt(this.orbat || []);
  },
  methods: {
    filledCount(unit) {
      if (!unit || !Array.isArray(unit.slots)) return 0;
      return unit.slots.filter((s) => s && s.id).length;
    },

    triggerFlicker(delayMs = 0) {
      try {
        this.animateView = false;
        this.animationDelay = `${delayMs}ms`;
        clearTimeout(this._flickerTimer);
        this._flickerTimer = setTimeout(() => {
          this.animateView = true;
          this._flickerTimer = setTimeout(() => {
            this.animateView = false;
          }, 420);
        }, 10);
      } catch {
        // ignore
      }
    },

    ensureUnitsBuilt(orbat) {
      const built = this.buildUnitsFromOrbat(orbat).filter((u) => this.isPointsUnit(u.title));
      if (built.length === 0) {
        this.plan.units = this.makeDefaultChalks(4, this.MIN_CHALK_SLOTS);
        this.detailKey = this.plan.units[0].key;
        this.debugInfo = "Fallback to default Chalk 1–4 (no/invalid ORBAT).";
        return;
      }
      built.forEach((u) => u.slots.forEach((s) => {
        if (typeof s.disposable === "undefined") s.disposable = false;
      }));
      this.plan.units = built;
      if (!this.detailKey && this.plan.units.length) this.detailKey = this.plan.units[0].key;
      this.debugInfo = `Loaded ${this.plan.units.length} chalk(s) from ORBAT.`;
    },

    makeDefaultChalks(count, size) {
      const arr = [];
      for (let i = 1; i <= count; i++) {
        const title = `Chalk ${i}`;
        const key = this.keyFromName(title);
        const slots = this.padSlots([], size);
        arr.push({ key, title, slots });
      }
      return arr;
    },

    ensureDeviceId() {
      try {
        const key = "orbatDeviceId";
        const existing = localStorage.getItem(key);
        if (existing && /^[a-zA-Z0-9_.-]{8,}$/.test(existing)) { this.deviceId = existing; return; }
        const id = `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 10)}`;
        this.deviceId = id;
        localStorage.setItem(key, id);
      } catch {
        this.deviceId = "";
      }
    },

    isChalk(title) { return /chalk\s*\d+/i.test(String(title || "")); },
    isPointsUnit(title) { return this.isChalk(title) || /squad/i.test(String(title || "")); },

    keyFromName(name) {
      return String(name || "")
        .trim()
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, "-")
        .replace(/(^-|-$)/g, "");
    },

    squadInitials(title) {
      const t = String(title || "").trim();
      if (!t) return "??";
      const m = t.match(/chalk\s*(\d+)/i);
      if (m) return `C${m[1]}`;
      return t.split(/\s+/).slice(0, 2).map((w) => w[0]?.toUpperCase() || "").join("");
    },

    padSlots(slots, minLen) {
      const out = Array.isArray(slots) ? slots.slice() : [];
      while (out.length < minLen) out.push({ id: null, name: null, role: "", origStatus: "VACANT", cert: "", disposable: false });
      return out;
    },

    normalizeRole(role) {
      return String(role || "").trim().toLowerCase();
    },

    rolePriority(role) {
      const key = this.normalizeRole(role);
      const idx = this.ROLE_ORDER.indexOf(key);
      return idx === -1 ? 10000 : idx;
    },

    sortSlotsByRole(slots, fireteams = []) {
      const list = Array.isArray(slots) ? slots.slice() : [];
      const declared = (Array.isArray(fireteams) ? fireteams : [])
        .map((ft) => this.normalizeFireteamName(ft))
        .filter(Boolean);
      const declaredIdx = new Map(declared.map((l, i) => [l, i]));

      const fireteamWeight = (label) => {
        const l = this.normalizeFireteamName(label || "");
        if (!l) return 9999;
        if (declaredIdx.has(l)) return declaredIdx.get(l);
        if (l === "HEADQUARTERS") return -10;
        const m = l.match(/^FIRETEAM\s+(\d+)$/i);
        if (m) return Number(m[1]) || 0;
        const abcd = ["ALPHA", "BRAVO", "CHARLIE", "DELTA"];
        const idx = abcd.indexOf(l);
        if (idx !== -1) return 20 + idx;
        return 200 + l.charCodeAt(0);
      };

      return list
        .map((s, i) => {
          const statusW = s?.origStatus === "CLOSED" ? 2 : (s?.id ? 0 : 1);
          const ftLabel = this.fireteamLabel(s, i, list) || "";
          return { s, i, statusW, ftW: fireteamWeight(ftLabel), roleW: this.rolePriority(s?.role || "") };
        })
        .sort((a, b) => a.ftW - b.ftW || a.statusW - b.statusW || a.roleW - b.roleW || a.i - b.i)
        .map((x) => x.s);
    },

    sortSlotsByFireteam(slots, fireteams = []) {
      return this.sortSlotsByRole(slots, fireteams);
    },

    extractCertsFromMember(member) {
      const arr = member?.certifications;
      if (Array.isArray(arr) && arr.length) {
        const out = [];
        arr.forEach((c) => out.push(String(c)));
        return out;
      }
      return [];
    },

    ensureSlotCert(slot, fallbackRole) {
      const available = this.getCertsForPersonId(slot?.id);
      if (available.includes(slot?.cert)) return slot.cert;
      if (available.includes(fallbackRole)) return fallbackRole;
      return available[0] || slot?.cert || fallbackRole || "";
    },

    getCertsForPersonId(personId) {
      const p = (this.personnel || []).find((x) => String(x.id) === String(personId));
      const from = (p && Array.isArray(p.certifications)) ? p.certifications : [];
      if (from.length) return from.map(String);
      return this.certLabels.slice();
    },

    getRankForPersonId(personId) {
      const p = (this.personnel || []).find((x) => String(x.id) === String(personId));
      return p ? p.rank : "";
    },

    hasCertId(personId, certIndex) {
      const certs = this.getCertsForPersonId(personId);
      return !!certs[certIndex];
    },

    certOptionsForId(personId) {
      const opts = this.getCertsForPersonId(personId);
      return opts.length ? opts : this.certLabels.slice();
    },

    certPointSuffix(cert) {
      const pts = this.CERT_POINTS[cert] ?? 0;
      return pts ? ` (+${pts})` : "";
    },

    titleCase(s) {
      return String(s || "")
        .split(/\s+/)
        .filter(Boolean)
        .map((w) => w[0]?.toUpperCase() + w.slice(1))
        .join(" ");
    },

    buildPersonnelPool(orbat) {
      try {
        const out = [];
        const addMember = (m, elementName = "") => {
          if (!m) return;
          const id = m.id ?? m.personId ?? m.uid ?? m.uniqueId ?? (m.callsign || m.name || "");
          out.push({
            id: String(id),
            name: m.name || m.fullName || m.displayName || "",
            callsign: m.callsign || "",
            role: m.role || m.slot || "",
            rank: m.rank || m.grade || "",
            certifications: m.certs || m.certifications || [],
            element: elementName || m.element || m.fireteam || "",
          });
        };

        if (Array.isArray(orbat)) {
          (orbat || []).forEach((unit) => {
            if (Array.isArray(unit.members)) unit.members.forEach((m) => addMember(m));
            if (Array.isArray(unit.personnel)) unit.personnel.forEach((m) => addMember(m));
            if (Array.isArray(unit.fireteams)) {
              unit.fireteams.forEach((ft) => {
                const el = String(ft?.name || "").trim();
                (ft.slots || []).forEach((s) => addMember(s.member || s, el));
              });
            }
          });
        } else if (orbat && typeof orbat === "object") {
          Object.values(orbat).forEach((unit) => {
            if (Array.isArray(unit.members)) unit.members.forEach((m) => addMember(m));
            if (Array.isArray(unit.personnel)) unit.personnel.forEach((m) => addMember(m));
          });
        }

        const map = {};
        out.forEach((p) => {
          const k = String(p.id || p.name || Math.random());
          if (!map[k]) map[k] = p;
        });
        return Object.values(map);
      } catch {
        return [];
      }
    },

    buildUnitsFromOrbat(orbat) {
      const units = [];
      (orbat || []).forEach((sq) => {
        const key = this.keyFromName(sq.squad);
        const fireteams = (sq.fireteams || []).map((ft) => String(ft.name || "").trim()).filter(Boolean);
        const slots = [];
        (sq.fireteams || []).forEach((ft) => {
          (ft.slots || []).forEach((s) => {
            const status = String(s?.status || (s?.member ? "FILLED" : "VACANT")).toUpperCase();
            const origStatus = ["VACANT", "CLOSED"].includes(status) ? status : "FILLED";
            const member = s?.member || null;
            const slot = {
              id: member?.id ? String(member.id) : null,
              name: member?.name || null,
              role: s?.role || member?.slot || "",
              fireteam: String(ft.name || "").trim(),
              origStatus,
              cert: "",
              disposable: false,
            };
            if (slot.id) slot.cert = this.ensureSlotCert(slot, slot.role);
            slots.push(slot);
          });
        });

        let finalSlots = this.sortSlotsByFireteam(slots, fireteams);
        if (this.isChalk(sq.squad)) finalSlots = this.padSlots(finalSlots, this.MIN_CHALK_SLOTS);
        if (this.isPointsUnit(sq.squad)) units.push({ key, title: sq.squad, slots: finalSlots, fireteams });
      });
      return units;
    },

    buildUnitFromOrbatByKey(orbat, unitKey) {
      const unit = (orbat || []).find((sq) => this.keyFromName(sq.squad) === unitKey);
      if (!unit) return null;
      const fireteams = (unit.fireteams || []).map((ft) => String(ft.name || "").trim()).filter(Boolean);
      const slots = [];
      (unit.fireteams || []).forEach((ft) => {
        (ft.slots || []).forEach((s) => {
          const status = String(s?.status || (s?.member ? "FILLED" : "VACANT")).toUpperCase();
          const origStatus = ["VACANT", "CLOSED"].includes(status) ? status : "FILLED";
          const member = s?.member || null;
          const slot = {
            id: member?.id ? String(member.id) : null,
            name: member?.name || null,
            role: s?.role || member?.slot || "",
            fireteam: String(ft.name || "").trim(),
            origStatus,
            cert: "",
            disposable: false,
          };
          if (slot.id) slot.cert = this.ensureSlotCert(slot, slot.role);
          slots.push(slot);
        });
      });

      let finalSlots = this.sortSlotsByRole(slots, fireteams);
      if (this.isChalk(unit.squad)) finalSlots = this.padSlots(finalSlots, this.MIN_CHALK_SLOTS);
      return { key: unitKey, title: unit.squad, slots: finalSlots, fireteams };
    },

    calcUnitPoints(unit) {
      if (!unit || !unit.slots) return 0;
      return unit.slots.reduce((sum, s) => {
        if (!s.id) return sum;
        const certPts = this.CERT_POINTS[s.cert] ?? 0;
        const dispPts = s.disposable ? this.DISPOSABLE_COST : 0;
        return sum + certPts + dispPts;
      }, 0);
    },

    wouldExceedCap(unitKey, delta) {
      const unit = this.plan.units.find((u) => u.key === unitKey);
      if (!unit) return false;
      return this.calcUnitPoints(unit) + delta > this.SQUAD_POINT_CAP;
    },

    openPicker(unitKey, slotIdx) {
      const g = this.plan.units.find((u) => u.key === unitKey);
      if (!g || g.slots[slotIdx]?.origStatus === "CLOSED") return;
      this.picker = { ...this.picker, open: true, unitKey, slotIdx, query: "", onlyFree: false, sort: this.picker.sort || "name_asc" };
    },

    closePicker() { this.picker.open = false; },

    findAssignment(personId) {
      if (!personId) return null;
      for (const u of this.plan.units || []) {
        for (let i = 0; i < (u.slots || []).length; i++) {
          const s = u.slots[i];
          if (s && String(s.id) === String(personId)) return { unitKey: u.key, slotIdx: i };
        }
      }
      return null;
    },

    selectPersonnel(p) {
      if (!this.picker.open) return;
      const from = this.findAssignment(p.id);
      const gIdx = this.plan.units.findIndex((u) => u.key === this.picker.unitKey);
      if (gIdx < 0) return;
      const g = this.plan.units[gIdx];
      const target = g.slots[this.picker.slotIdx];

      const available = this.getCertsForPersonId(p.id);
      const chosenCertDefault = available.includes(target.cert) ? target.cert : (available[0] || target.role || p.role || "");

      const prevPts = (this.CERT_POINTS[target.cert] ?? 0) + (target.disposable ? this.DISPOSABLE_COST : 0);
      const nextPts = (this.CERT_POINTS[chosenCertDefault] ?? 0);
      const delta = nextPts - prevPts;
      if (this.wouldExceedCap(g.key, Math.max(0, delta))) { this.detailError = `Point cap ( ${this.SQUAD_POINT_CAP} ) would be exceeded.`; return; }
      this.detailError = "";

      if (from && target?.id && !(from.unitKey === g.key && from.slotIdx === this.picker.slotIdx)) {
        const srcIdx = this.plan.units.findIndex((u) => u.key === from.unitKey);
        const srcGroup = this.plan.units[srcIdx];
        const tmp = { ...target };
        const newTarget = { ...target, id: p.id, name: p.name, role: target.role || p.role || "", cert: chosenCertDefault, disposable: false };

        const newSrcSlots = srcGroup.slots.slice();
        newSrcSlots[from.slotIdx] = { ...newSrcSlots[from.slotIdx], id: tmp.id || null, name: tmp.name || null, cert: tmp.cert || "", disposable: !!tmp.disposable };
        const newSrc = { ...srcGroup, slots: this.sortSlotsByRole(newSrcSlots, srcGroup.fireteams || []) };

        const newGSlots = g.slots.slice();
        newGSlots[this.picker.slotIdx] = newTarget;
        const newG = { ...g, slots: this.sortSlotsByRole(newGSlots, g.fireteams || []) };

        this.plan.units = this.plan.units.map((u, i) => (i === gIdx ? newG : i === srcIdx ? newSrc : u));
        this.persistPlan();
        this.closePicker();
        this.triggerFlicker(0);
        return;
      }

      const newSlots = g.slots.slice();
      newSlots[this.picker.slotIdx] = { ...target, id: p.id, name: p.name, role: target.role || p.role || "", cert: chosenCertDefault, disposable: false };
      const newG = { ...g, slots: this.sortSlotsByRole(newSlots, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === gIdx ? newG : u));
      this.persistPlan();
      this.closePicker();
      this.triggerFlicker(0);
    },

    clearSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx === -1) return;
      const g = this.plan.units[idx];
      const slot = g.slots[slotIdx];
      if (!slot) return;

      const newSlots = g.slots.slice();
      newSlots[slotIdx] = { ...slot, id: null, name: null, cert: "", disposable: false, origStatus: "VACANT" };

      const newG = { ...g, slots: this.sortSlotsByRole(newSlots, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    clearGroup(unitKey) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx === -1) return;
      const g = this.plan.units[idx];
      const emptied = (g.slots || []).map((s) => ({ ...s, id: null, name: null, cert: "", disposable: false, origStatus: s.origStatus === "CLOSED" ? "CLOSED" : "VACANT" }));
      const newG = { ...g, slots: this.sortSlotsByFireteam(emptied, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    addSlot(unitKey) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx === -1) return;
      const g = this.plan.units[idx];
      const newSlots = (g.slots || []).slice();
      newSlots.push({ id: null, name: null, role: "", origStatus: "VACANT", cert: "", disposable: false });
      const newG = { ...g, slots: this.sortSlotsByRole(newSlots, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    removeSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx === -1) return;
      const g = this.plan.units[idx];
      const newSlots = (g.slots || []).slice();
      newSlots.splice(slotIdx, 1);
      const newG = { ...g, slots: this.sortSlotsByRole(newSlots, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    closeSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx === -1) return;
      const g = this.plan.units[idx];
      const slot = g.slots[slotIdx];
      const newSlots = g.slots.slice();
      newSlots[slotIdx] = { ...slot, id: null, name: null, cert: "", disposable: false, origStatus: "CLOSED" };
      const newG = { ...g, slots: this.sortSlotsByRole(newSlots, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    openSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx === -1) return;
      const g = this.plan.units[idx];
      const slot = g.slots[slotIdx];
      const newSlots = g.slots.slice();
      newSlots[slotIdx] = { ...slot, origStatus: "VACANT" };
      const newG = { ...g, slots: this.sortSlotsByRole(newSlots, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    setSlotCert(unitKey, slotIdx, nextCert) {
      const uIdx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (uIdx < 0) return;
      const unit = this.plan.units[uIdx];
      const slot = unit.slots[slotIdx];
      if (!slot || !slot.id) return;

      const prevPts = (this.CERT_POINTS[slot.cert] ?? 0) + (slot.disposable ? this.DISPOSABLE_COST : 0);
      const nextPts = (this.CERT_POINTS[nextCert] ?? 0) + (slot.disposable ? this.DISPOSABLE_COST : 0);
      const delta = nextPts - prevPts;
      if (this.wouldExceedCap(unitKey, Math.max(0, delta))) { this.detailError = `Point cap ( ${this.SQUAD_POINT_CAP} ) would be exceeded.`; return; }
      this.detailError = "";

      const newSlots = unit.slots.slice();
      newSlots[slotIdx] = { ...slot, cert: nextCert };
      const newU = { ...unit, slots: this.sortSlotsByRole(newSlots, unit.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === uIdx ? newU : u));
      this.persistPlan();
    },

    toggleDisposable(unitKey, slotIdx, checked) {
      const uIdx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (uIdx < 0) return;
      const unit = this.plan.units[uIdx];
      const slot = unit.slots[slotIdx];
      if (!slot || !slot.id) return;

      const prev = slot.disposable ? this.DISPOSABLE_COST : 0;
      const next = checked ? this.DISPOSABLE_COST : 0;
      const delta = next - prev;
      if (this.wouldExceedCap(unitKey, Math.max(0, delta))) { this.detailError = `Point cap ( ${this.SQUAD_POINT_CAP} ) would be exceeded.`; return; }
      this.detailError = "";

      const newSlots = unit.slots.slice();
      newSlots[slotIdx] = { ...slot, disposable: checked === true };
      const newU = { ...unit, slots: newSlots };
      this.plan.units = this.plan.units.map((u, i) => (i === uIdx ? newU : u));
      this.persistPlan();
    },

    fillFromRoster(unitKey) {
      const rebuilt = this.buildUnitFromOrbatByKey(this.orbat, unitKey);
      if (!rebuilt) { this.detailError = "No matching unit in ORBAT."; return; }
      const idx = this.plan.units.findIndex((u) => u.key === unitKey);
      if (idx < 0) return;

      const keepLen = Math.max(this.plan.units[idx].slots.length, rebuilt.slots.length);
      while (rebuilt.slots.length < keepLen) rebuilt.slots.push({ id: null, name: null, role: "", origStatus: "VACANT", cert: "", disposable: false });

      this.plan.units = this.plan.units.map((u, i) => (i === idx ? rebuilt : u));
      this.persistPlan();
      this.detailError = "";
      this.triggerFlicker(0);
    },

    inferFireteamIndex(slotIdx, slots) {
      try {
        let n = 0;
        for (let i = 0; i <= slotIdx; i++) {
          const r = String(slots?.[i]?.role || "").toLowerCase();
          if (r.includes("team leader")) n += 1;
        }
        return n;
      } catch {
        return 0;
      }
    },

    normalizeFireteamName(raw) {
      const t = String(raw || "").trim();
      if (!t) return "";
      const lower = t.toLowerCase();

      const map = { a: "ALPHA", alpha: "ALPHA", b: "BRAVO", bravo: "BRAVO", c: "CHARLIE", charlie: "CHARLIE", d: "DELTA", delta: "DELTA", hq: "HEADQUARTERS" };
      if (map[lower]) return map[lower];

      const m1 = lower.match(/^(?:ft|fireteam)\s*[-#:]?\s*(\d+)$/i);
      if (m1) return `FIRETEAM ${m1[1]}`;

      const m2 = lower.match(/^(?:fire\s*team|fireteam)\s*[-#:]?\s*(.+)$/i);
      if (m2 && m2[1]) return `FIRETEAM ${String(m2[1]).trim().toUpperCase()}`;

      return t.toUpperCase();
    },

    fireteamLabel(slot, slotIdx = 0, slots = []) {
      const explicit = slot?.fireteam || slot?.fireTeam || slot?.element || slot?.team || slot?.group || slot?.section || "";
      if (explicit) return this.normalizeFireteamName(explicit);

      const role = String(slot?.role || "").toLowerCase();
      if (role.includes("squad lead") || role.includes("squad leader")) return "HEADQUARTERS";

      if (role.includes("team leader")) {
        const idx = this.inferFireteamIndex(slotIdx, slots) || 1;
        return `FIRETEAM ${idx}`;
      }
      return "";
    },

    isFireteamHeader(slot, slotIdx, slots) {
      const label = this.fireteamLabel(slot, slotIdx, slots);
      if (!label) return false;
      if (slotIdx === 0) return true;
      const prev = this.fireteamLabel(slots?.[slotIdx - 1], slotIdx - 1, slots);
      return label !== prev;
    },

    persistPlan() {
      try {
        localStorage.setItem(this.STORAGE_KEY, JSON.stringify(this.plan || {}));
      } catch {
        // ignore
      }
    },

    openOverview() {
      this.overview.open = true;
    },

    closeOverview() {
      this.overview.open = false;
    },

    ovItem(unitKey) {
      const u = this.plan.units.find((x) => x.key === unitKey);
      return u || null;
    },

    rankLabel(rank) {
      const r = String(rank || "").trim();
      if (!r) return "";
      const m = r.match(/(O-\d+|W-\d+|E-\d+)/i);
      return m ? m[1].toUpperCase() : r.toUpperCase();
    },

    rankIcon(label) {
      const base = String(this.rankIconBase || "/ranks").replace(/\/$/, "");
      const ext = String(this.rankIconExt || "png").replace(/^\./, "");
      const safe = String(label || "").trim().replace(/[^A-Z0-9_-]/g, "");
      return `${base}/${safe}.${ext}`;
    },

    onRankImgError(e) {
      try {
        const src = e?.target?.src || "";
        this.rankImgFailed[src] = true;
      } catch {
        // ignore
      }
    },

    unitPayload(unit) {
      return {
        title: unit.title,
        slots: (unit.slots || []).map((s) => ({
          id: s?.id ?? null,
          name: s?.name ?? null,
          role: s?.role || "",
          fireteam: s?.fireteam || s?.element || "",
          cert: s?.cert || "",
          disposable: !!s?.disposable,
        })),
      };
    },

    async fetchRemoteMeta(unitKey) {
      if (!unitKey || !this.apiBase) return;
      try {
        const resp = await this.apiPost("config:get", { unitId: unitKey });
        if (resp?.ok && resp?.data) {
          this.remoteMeta = { ...this.remoteMeta, [unitKey]: { savedBy: resp.data.savedBy || "", savedAt: resp.data.savedAt || "" } };
        }
      } catch {
        // ignore
      }
    },

    parseConfigJSONSlots(rec) {
      const txt = typeof rec?.configJSON === "string" ? rec.configJSON
        : typeof rec?.data?.configJSON === "string" ? rec.data.configJSON
        : null;
      if (!txt) return [];
      try {
        const obj = JSON.parse(txt);
        const slots = obj?.slots || obj?.config?.slots || [];
        if (!Array.isArray(slots)) return [];
        return slots.map((s) => ({
          id: s?.id ?? null,
          name: s?.name ?? null,
          role: s?.role || "",
          fireteam: s?.fireteam || s?.element || "",
          cert: s?.cert || "",
          disposable: !!s?.disposable,
        }));
      } catch {
        return [];
      }
    },

    extractSlotsFromAny(rec) {
      const normSlots = (arr) =>
        (Array.isArray(arr) ? arr : []).map((s) => ({
          id: s?.id ?? null,
          name: s?.name ?? null,
          role: s?.role || "",
          fireteam: s?.fireteam || s?.element || "",
          cert: s?.cert || "",
          disposable: !!s?.disposable,
        }));

      const tryParse = (val) => {
        if (!val) return null;
        if (typeof val === "string") {
          try { return JSON.parse(val); } catch { return null; }
        }
        if (typeof val === "object") return val;
        return null;
      };

      const obj = tryParse(rec?.configJSON) || tryParse(rec?.data?.configJSON) || tryParse(rec?.data) || tryParse(rec);
      const slots = obj?.slots || obj?.config?.slots || obj?.data?.slots;
      return normSlots(slots);
    },

    async apiPost(action, body) {
      const ss = typeof sessionStorage !== "undefined" ? sessionStorage : null;
      const ls = typeof localStorage !== "undefined" ? localStorage : null;
      const readJson = (s) => { try { return s ? JSON.parse(s) : null; } catch { return null; } };
      const readStr = (k) => String(ss?.getItem(k) || ls?.getItem(k) || "").trim();

      const decodeJwtPayload = (token) => {
        try {
          const rawTok = String(token || "").replace(/^Bearer\s+/i, "").trim();
          const parts = rawTok.split(".");
          if (parts.length < 2) return null;
          const b64url = parts[1].replace(/-/g, "+").replace(/_/g, "/");
          const pad = "=".repeat((4 - (b64url.length % 4)) % 4);
          const json = atob(b64url + pad);
          return JSON.parse(json);
        } catch {
          return null;
        }
      };

      const adminUser = readJson(ss?.getItem("admin:user") || "");
      const adminRole = (readStr("admin:role") || readStr("authRole") || String(adminUser?.role || "")).trim();
      const adminTokenRaw = (readStr("admin:token") || readStr("authToken") || "").trim();
      const adminAuth = adminTokenRaw
        ? (adminTokenRaw.toLowerCase().startsWith("bearer ") ? adminTokenRaw : `Bearer ${adminTokenRaw}`)
        : "";

      const jwtPayload = decodeJwtPayload(adminAuth || adminTokenRaw);
      const jwtUserMeta = jwtPayload?.user_metadata || {};
      const jwtAppMeta = jwtPayload?.app_metadata || {};

      const adminDisplay = String(
        adminUser?.displayName ||
        adminUser?.display_name ||
        adminUser?.full_name ||
        adminUser?.fullName ||
        adminUser?.name ||
        adminUser?.username ||
        adminUser?.user ||
        adminUser?.email ||
        jwtUserMeta?.full_name ||
        jwtUserMeta?.name ||
        jwtUserMeta?.display_name ||
        jwtUserMeta?.user_name ||
        jwtPayload?.name ||
        jwtPayload?.email ||
        ""
      ).trim();

      const jwtRoles = Array.isArray(jwtAppMeta?.roles) ? jwtAppMeta.roles : [];
      const jwtRole = String(jwtRoles[0] || "").trim();

      let niUser = null;
      try { niUser = window?.netlifyIdentity?.currentUser?.() || null; } catch {}
      const niDisplay = String(
        niUser?.user_metadata?.full_name ||
        niUser?.user_metadata?.name ||
        niUser?.user_metadata?.display_name ||
        niUser?.email ||
        ""
      ).trim();
      const niAuth = niUser?.token?.access_token ? `Bearer ${niUser.token.access_token}` : "";

      const legacyUser = readJson(readStr("user"));
      const legacyDisplay = String(
        legacyUser?.displayName ||
        legacyUser?.fullName ||
        legacyUser?.name ||
        legacyUser?.username ||
        legacyUser?.login ||
        legacyUser?.email ||
        ""
      ).trim();
      const legacyRole = String(legacyUser?.role || "").trim();
      const legacyAuth = readStr("Authorization");

      const candidateUser = adminDisplay || niDisplay || legacyDisplay;
      const candidateRole = (adminRole || jwtRole || legacyRole).trim();
      const authHeader = adminAuth || legacyAuth || niAuth;

      const payload = {
        secret: this.secret || "PLEX",
        action,
        deviceId: this.deviceId,
        ...body,
      };

      const urlObj = new URL(this.apiBase, typeof window !== "undefined" ? window.location.origin : "http://localhost");
      const isDirectGAS = /script\.google\.com|googleusercontent\.com/.test(urlObj.hostname);

      const headers = { "Content-Type": "application/json" };
      if (authHeader) headers["Authorization"] = authHeader;

      let finalUrl = this.apiBase;
      if (isDirectGAS) {
        const u = new URL(this.apiBase);
        if (candidateUser) u.searchParams.set("X-User", candidateUser);
        if (candidateRole) u.searchParams.set("X-Role", candidateRole);
        if (authHeader) u.searchParams.set("Authorization", authHeader);
        finalUrl = u.toString();
      } else {
        if (candidateUser) headers["X-User"] = candidateUser;
        if (candidateRole) headers["X-Role"] = candidateRole;
      }

      const res = await fetch(finalUrl, { method: "POST", headers, body: JSON.stringify(payload) });
      const txt = await res.text();
      try { return JSON.parse(txt); } catch { return { ok: false, error: txt || `HTTP ${res.status}` }; }
    },

    async saveRemote(unitKey) {
      if (!unitKey || !this.apiBase) return;
      this.apiError = "";
      this.busy = true;
      try {
        const unit = this.plan.units.find((u) => u.key === unitKey);
        if (!unit) throw new Error("Unknown unit");
        const config = this.unitPayload(unit);
        const resp = await this.apiPost("config:save", { unitId: unitKey, config });
        if (!resp?.ok) throw new Error(resp?.error || "Save failed");
        this.versions = { ...this.versions, [unitKey]: Number(resp?.data?.version || resp?.version || 0) };
        await this.fetchRemoteMeta(unitKey);
        this.triggerFlicker(0);
      } catch (e) {
        this.apiError = String(e?.message || e);
      } finally {
        this.busy = false;
      }
    },

    async loadRemote(unitKey) {
      if (!unitKey) return;
      this.apiError = "";
      this.busy = true;
      try {
        const resp = await this.apiPost("config:get", { unitId: unitKey });
        const { ok, data, error } = resp || {};
        if (!ok) throw new Error(error || "Load failed");
        if (!data) { this.apiError = "No remote config yet for this Chalk."; return; }

        let nextSlots = this.parseConfigJSONSlots(data);
        if (!nextSlots.length) nextSlots = this.extractSlotsFromAny(data);

        const idx = this.plan.units.findIndex((u) => u.key === unitKey);
        if (idx === -1) return;
        const curr = this.plan.units[idx];

        const maxLen = Math.max((curr.slots || []).length, (nextSlots || []).length);
        const merged = [];
        for (let i = 0; i < maxLen; i++) {
          const base = (curr.slots || [])[i] || {};
          const inc = (nextSlots || [])[i] || {};
          merged.push({
            ...base,
            ...inc,
            role: (inc.role || base.role || ""),
            fireteam: (inc.fireteam || base.fireteam || base.element || ""),
            cert: (inc.cert || base.cert || ""),
            disposable: (typeof inc.disposable === "boolean" ? inc.disposable : !!base.disposable),
            id: (inc.id ?? base.id ?? null),
            name: (inc.name ?? base.name ?? null),
            origStatus: (inc.id || base.id) ? "FILLED" : (base.origStatus || "VACANT"),
          });
        }

        const toApply = (nextSlots.length ? merged : (curr.slots || [])).map((s) => ({
          ...s,
          origStatus: s.id ? "FILLED" : (s.origStatus || "VACANT"),
        }));
        const padded = this.isChalk(curr.title) ? this.padSlots(toApply, this.MIN_CHALK_SLOTS) : toApply;
        const nextUnit = { ...curr, slots: this.sortSlotsByRole(padded, curr.fireteams || []) };

        this.plan.units = this.plan.units.map((u, i) => (i === idx ? nextUnit : u));
        this.versions = { ...this.versions, [unitKey]: Number(data.version || 0) };
        this.triggerFlicker(0);
      } catch (e) {
        this.apiError = String(e?.message || e);
      } finally {
        this.busy = false;
      }
    },

    formatAssignment(assign) {
      if (!assign) return "";
      const u = this.plan.units.find((x) => x.key === assign.unitKey);
      return u ? `${u.title} #${assign.slotIdx + 1}` : `${assign.unitKey}#${assign.slotIdx + 1}`;
    },
  },
  watch: {
    orbat: {
      deep: true,
      handler(newV) {
        this.personnel = this.buildPersonnelPool(newV || []);
        const oldKey = this.detailKey;
        this.ensureUnitsBuilt(newV || []);
        if (!this.plan.units.find((u) => u.key === oldKey) && this.plan.units.length) {
          this.detailKey = this.plan.units[0].key;
        }
      },
    },
    detailKey: {
      immediate: true,
      handler(k) {
        if (k) this.fetchRemoteMeta(k);
      },
    },
  },
};
</script>

<style scoped>
#deploymentView { color: #e6f3ff; }

#deploymentView{display:grid;grid-template-columns:1fr;gap:1.2rem;align-items:start;width:calc(100dvw - 90px);height:calc(100dvh - 95px);min-height:0;overflow:hidden;padding:18px 18px calc(18px + env(safe-area-inset-bottom, 0px)) 18px;box-sizing:border-box}
.deployment-window.section-container{max-width:none!important;width:auto;height:100%;display:flex;flex-direction:column;overflow:hidden}
.header-shell{height:52px;overflow:hidden}.section-header,section-content-container{width:100%}
.deploy-scroll{flex:1 1 auto;min-height:0;max-height:none;overflow-y:auto;scrollbar-gutter:stable both-edges;padding-bottom:calc(12px + env(safe-area-inset-bottom, 0px))}

.panel{border:1px dashed rgba(30,144,255,0.35);background:rgba(0,10,30,0.18);border-radius:.6rem;padding:.8rem .9rem;overflow:visible}
.top-actions{display:flex;justify-content:flex-end;gap:.6rem}
.toolbar{display:flex;justify-content:space-between;gap:1rem;flex-wrap:wrap;align-items:center;margin-bottom:.8rem}
.toolbar-left{display:flex;gap:.6rem;align-items:center;flex-wrap:wrap}
.toolbar-right{display:flex;gap:.6rem;align-items:center;flex-wrap:wrap;justify-content:flex-end}
.label{opacity:.85;font-weight:600}
.select{background:#040a14;border:1px solid rgba(30,144,255,.35);color:#e6f3ff;border-radius:.35rem;padding:.35rem .45rem}
.btn{border:1px solid rgba(30,144,255,.35);background:rgba(30,144,255,.10);color:#e6f3ff;border-radius:.45rem;padding:.4rem .7rem;cursor:pointer}
.btn.primary{background:rgba(30,144,255,.22);border-color:rgba(30,144,255,.55)}
.btn.ghost{background:transparent}
.btn.small{padding:.25rem .55rem;font-size:.9rem}
.muted{opacity:.78}
.small{font-size:.9rem}
.error{color:#ff7a7a;font-weight:700}

.unit-shell{display:flex;flex-direction:column;gap:.8rem}
.unit-head{display:flex;justify-content:space-between;gap:1rem;align-items:flex-start;flex-wrap:wrap}
.unit-title{display:flex;gap:.8rem;align-items:center}
.unit-badge{width:48px;height:48px;border-radius:12px;display:grid;place-items:center;border:1px solid rgba(30,144,255,.45);background:rgba(0,0,0,.35);font-weight:900}
.unit-text h3{margin:0;font-size:1.15rem}
.unit-sub{display:flex;gap:.4rem;flex-wrap:wrap}
.unit-tools{display:flex;gap:.5rem;flex-wrap:wrap}

.squad-members-grid{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:.8rem}
@media (max-width: 1000px){.squad-members-grid{grid-template-columns:1fr}}

.member-card{border:1px solid rgba(30,144,255,.25);background:rgba(0,0,0,.28);border-radius:.75rem;overflow:hidden;display:flex;flex-direction:column}
.member-card.vacant{background:rgba(0,0,0,.18)}
.member-card.closed{opacity:.65;border-style:dashed}
.member-header{display:flex;justify-content:space-between;align-items:flex-start;gap:.6rem;padding:.65rem .75rem .5rem;border-bottom:1px solid rgba(30,144,255,.18)}
.member-id{display:flex;gap:.6rem;align-items:center}
.member-status{font-weight:800;letter-spacing:.06em}
.rank-icon{width:32px;height:32px;object-fit:contain}
.rank-icon.small{width:24px;height:24px}
.name-block .name{font-weight:900;letter-spacing:.03em}
.member-actions{display:flex;gap:.4rem;flex-wrap:wrap}
.member-body{padding:.6rem .75rem;display:flex;flex-direction:column;gap:.45rem}
.member-body.filled{padding:.6rem .75rem .75rem}
.member-row{display:flex;justify-content:space-between;gap:.8rem}
.member-columns{display:grid;grid-template-columns:1fr 1fr;gap:1rem}
@media (max-width: 800px){.member-columns{grid-template-columns:1fr}}
.role-big{font-size:1.05rem;font-weight:800;margin:.15rem 0 .25rem}
.cert-list{display:flex;flex-direction:column;gap:.35rem}
.cert-row{display:flex;align-items:center;gap:.55rem}
.cert-checkbox{width:14px;height:14px;border-radius:4px;border:1px solid rgba(30,144,255,.35);display:grid;place-items:center}
.cert-checkbox.checked{border-color:rgba(120,255,170,.65)}
.checkbox-dot{width:8px;height:8px;border-radius:999px;background:rgba(120,255,170,.75)}
.member-footer{margin-top:auto;padding:.55rem .75rem;border-top:1px solid rgba(30,144,255,.18);display:flex;justify-content:space-between;align-items:center;gap:.7rem}
.actions-row{display:flex;gap:.6rem;flex-wrap:wrap;align-items:center;margin-top:.5rem}
.divider{flex:0 0 1px;width:1px;height:28px;background:rgba(30,144,255,.25)}

.squad-overlay{position:fixed;inset:0;background:rgba(0,0,0,0.65);backdrop-filter:blur(4px);display:grid;place-items:center;padding:18px;z-index:9999}
.squad-modal{width:min(1100px,96vw);max-height:90vh;min-height:0;overflow:hidden;border:1px solid rgba(30,144,255,0.35);border-radius:1rem;background:rgba(0,0,0,0.9);padding:1.1rem 1.2rem 1.2rem;display:flex;flex-direction:column}
.squad-modal.compact{max-width:1000px;width:min(1000px,96vw);max-height:86vh;min-height:0;overflow:hidden}
.squad-modal-header{display:flex;justify-content:space-between;align-items:center;margin-bottom:.4rem}
.squad-close{background:transparent;border:1px solid rgba(220,230,255,.35);color:#e6f3ff;border-radius:999px;padding:.2rem .75rem;font-size:1rem;cursor:pointer}
.compact-header{display:flex;justify-content:space-between;align-items:center}
.picker-toolbar{display:flex;gap:.8rem;align-items:center;padding:.4rem 0 .2rem;flex-wrap:wrap}
.search{flex:1;min-width:220px;padding:.4rem .6rem;border:1px solid rgba(30,144,255,.35);border-radius:.35rem;background:#040a14;color:#e6f3ff}
.sort{min-width:180px;background:#040a14;border:1px solid rgba(30,144,255,.35);color:#e6f3ff;border-radius:.35rem;padding:.3rem .4rem}
.check{display:flex;align-items:center;gap:.4rem}.check .check-label{opacity:.9}
.smallish{font-size:.9rem}
.pick-row.compact{display:grid;grid-template-columns:1fr 1fr auto;gap:.6rem;align-items:center;border:1px solid rgba(30,144,255,.25);border-radius:.5rem;padding:.45rem .6rem;margin:.35rem 0}
.pick-row.assigned{background:rgba(30,144,255,0.08)}
.row-left{display:flex;align-items:center;gap:.5rem}
.row-left .name{font-weight:700;letter-spacing:.02em}
.row-mid{display:flex;gap:.6rem;flex-wrap:wrap;align-items:center;color:#9ec5e6}
.row-mid .chip{border:1px solid rgba(30,144,255,.45);border-radius:.35rem;padding:.05rem .4rem}
.row-right{display:flex;justify-content:flex-end}

.overview-grid{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:1rem}
@media (max-width: 1000px){.overview-grid{grid-template-columns:1fr}}
.overview-card{border:1px solid rgba(30,144,255,.25);border-radius:.75rem;background:rgba(0,0,0,.25);overflow:hidden}
.overview-head{padding:.7rem .8rem;border-bottom:1px solid rgba(30,144,255,.18)}
.overview-title{font-weight:900}
.overview-slots{padding:.65rem .8rem;display:flex;flex-direction:column;gap:.35rem}
.overview-slot{display:flex;justify-content:space-between;gap:1rem;border:1px solid rgba(30,144,255,.16);border-radius:.5rem;padding:.35rem .55rem}
.slot-left{display:flex;flex-direction:column}
.slot-name{font-weight:800}
.slot-id{color:#9ec5e6;font-size:.9rem}
.slot-right{display:flex;gap:.5rem;align-items:center}
.slot-cert{color:#9ec5e6}
.slot-disp{border:1px solid rgba(120,255,170,.45);color:#caffe9;border-radius:.3rem;padding:.05rem .35rem;font-size:.8rem}
.overview-empty{text-align:center;color:#9ec5e6;padding:.5rem 0}

.squad-modal-scroll{flex:1 1 auto;min-height:0;overflow-y:auto;scrollbar-gutter:stable both-edges;padding-right:.35rem}
.squad-modal-scroll::-webkit-scrollbar{width:10px}
.squad-modal-scroll::-webkit-scrollbar-thumb{background:rgba(158,197,230,0.35);border-radius:999px}
.squad-modal-scroll::-webkit-scrollbar-track{background:rgba(0,0,0,0.22)}

.fireteam-row{grid-column:1 / -1;display:flex;align-items:center;gap:.6rem;padding:.25rem .2rem}
.fireteam-title{font-weight:900;letter-spacing:.06em;color:#9ec5e6;text-transform:uppercase}
</style>
