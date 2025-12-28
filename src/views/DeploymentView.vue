<!-- File: src/views/DeploymentView.vue -->
<template>
  <div id="deploymentView" class="content-container" :class="{ animate: animateView }" :style="{ 'animation-delay': animationDelay }">
    <section class="section-container deployment-window">
      <div class="header-shell">
        <div class="section-header clipped-medium-backward-pilot">
          <img src="/icons/protocol.svg" alt="" />
          <h1>DEPLOYMENT — {{ currentUnit?.title || 'Chalk' }}</h1>
        </div>
        <div class="rhombus-back">&nbsp;</div>
      </div>

      <div class="section-content-container deploy-scroll" :class="{ animate: animateView }">
        <div class="panel">
          <!-- Top actions: prominent Overview -->
          <div class="top-actions">
            <button type="button" class="btn primary large" :disabled="!apiBase" @click="openOverview">
              Overview
            </button>
          </div>

          <!-- Toolbar -->
          <div class="detail-toolbar">
            <div class="toolbar-left">
              <label class="muted small">Chalk</label>
              <select class="select chalk-picker" v-model="detailKey">
                <option v-for="u in chalkUnits" :key="u.key" :value="u.key">{{ u.title }}</option>
              </select>
              <button class="btn ghost small" @click="fillFromRoster(detailKey)">Reset</button>
              <button class="btn ghost small" @click="clearGroup(detailKey)">Clear</button>
            </div>

            <div class="toolbar-right">
              <span class="muted small">{{ filledCount(currentUnit) }} / {{ currentUnit?.slots.length || 0 }} assigned</span>
              <span class="divider" />
              <span v-if="currentUnit" class="pts big" :class="{ over: pointsUsed > SQUAD_POINT_CAP }">
                Points: {{ pointsUsed }} / {{ SQUAD_POINT_CAP }}
              </span>
              <span class="divider" />
              <span class="muted small">{{ authModeLabel }}</span>
            </div>
          </div>

          <!-- Exec URL warning -->
          <div v-if="!apiBase" class="warn">
            Apps Script /exec URL missing. Use your Netlify proxy to avoid CORS:
            <ul style="margin:.3rem 0 .1rem .9rem">
              <li><code>&lt;DeploymentView :execUrl="'/.netlify/functions/gas'" /&gt;</code></li>
              <li><code>window.DEPLOYMENT_EXEC_URL = "/.netlify/functions/gas"</code></li>
              <li><code>localStorage.setItem('deploymentExecUrl','/.netlify/functions/gas')</code></li>
              <li class="muted">Legacy: meta <code>apps-script-exec</code>, <code>window.APP_EXEC_URL</code>, <code>localStorage.execUrl</code></li>
            </ul>
          </div>

          <div v-if="debugInfo" class="muted small" style="opacity:.8">Status: {{ debugInfo }}</div>
          <div v-if="apiError" class="warn">{{ apiError }}</div>
          <div v-if="!currentUnit" class="muted">No chalk selected.</div>

          <!-- Cards grid -->
          <div v-else class="group-card">
            <div v-if="detailError" class="warn">{{ detailError }}</div>

            <div class="squad-modal-meta" :class="{ invalid: pointsUsed > SQUAD_POINT_CAP }">
              <div class="squad-title">
                <h2 style="margin:0">{{ currentUnit.title }}</h2>
                <p class="subtitle">{{ filledCount(currentUnit) }} / {{ currentUnit.slots.length }} PERSONNEL</p>
                <div class="loadout-status">
                  <span class="points">LOADOUT: {{ pointsUsed }}/{{ SQUAD_POINT_CAP }} PTS</span>
                  <span v-if="pointsUsed > SQUAD_POINT_CAP" class="warn">⚠ EXCEEDS CAP</span>
                  <span v-else class="ok">✓ VALID</span>
                </div>
              </div>
              <div class="squad-tag"><span>{{ squadInitials(currentUnit.title) }}</span></div>
            </div>

            <div class="squad-members-grid">
              <!-- key placed on <template> because v-for is on template -->
              <template v-for="(slot, sIdx) in currentUnit.slots" :key="`wrap-${detailKey}-${sIdx}`">
                <div v-if="isFireteamHeader(slot, sIdx, currentUnit.slots)" class="fireteam-row">
                  <span class="fireteam-title">{{ fireteamLabel(slot) }}</span>
                </div>
                <div
                  class="member-card"
                  :class="{ vacant: slot.origStatus === 'VACANT' && !slot.id, closed: slot.origStatus === 'CLOSED' }"
                >
                  <!-- VACANT / CLOSED -->
                  <template v-if="(slot.origStatus === 'VACANT' && !slot.id) || slot.origStatus === 'CLOSED'">
                    <div class="member-header">
                      <div class="member-header-text">
                        <h3>{{ (slot.origStatus || 'VACANT').toUpperCase() }}</h3>
                        <p class="rank-line">
                          <span class="rank">{{ slot.role || 'Slot' }}</span>
                          <span class="id">UNFILLED SLOT</span>
                        </p>
                      </div>
                      <div style="display:flex; gap:.35rem; margin-left:auto;">
                        <button
                          type="button"
                          class="btn ghost xsmall"
                          :disabled="slot.origStatus === 'CLOSED'"
                          @click.stop="openPicker(detailKey, sIdx)"
                        >
                          {{ slot.origStatus === 'CLOSED' ? 'Closed' : 'Assign' }}
                        </button>
                        <button type="button" class="btn ghost xsmall" @click.stop="removeSlot(detailKey, sIdx)">–</button>
                      </div>
                    </div>

                    <div class="member-body">
                      <div class="member-column left">
                        <p class="detail-line">
                          <strong>Role:</strong>
                          <span class="role-accent">{{ slot.role || 'Slot' }}</span>
                        </p>
                      </div>
                      <div class="member-column right">
                        <p><strong>Certifications:</strong></p>
                        <span class="cert-none">N/A</span>
                      </div>
                    </div>

                    <div class="member-footer">
                      <span>SLOT STATUS: {{ slot.origStatus || 'VACANT' }}</span>
                      <span>UNSC SYSTEMS DATABASE</span>
                    </div>
                  </template>

                  <!-- FILLED -->
                  <template v-else>
                    <div class="member-header">
                      <div class="member-header-text">
                        <div class="name-line">
                          <img
                            v-if="rankFor(slot.id)"
                            class="rank-icon"
                            :src="rankIcon(rankFor(slot.id))"
                            :alt="rankFor(slot.id)"
                            :title="rankFor(slot.id)"
                            @error="onRankImgError($event)"
                          />
                          <h3>{{ (slot.name || 'UNKNOWN').toUpperCase() }}</h3>
                        </div>
                        <p class="rank-line">
                          <span class="rank">{{ slot.role || 'N/A' }}</span>
                          <span class="id">ID: {{ slot.id || 'N/A' }}</span>
                        </p>
                      </div>
                      <div style="display:flex; gap:.35rem; margin-left:auto;">
                        <button v-if="slot.id" type="button" class="btn ghost xsmall" @click.stop="clearSlot(detailKey, sIdx)">Clear</button>
                        <button type="button" class="btn ghost xsmall" @click.stop="removeSlot(detailKey, sIdx)">–</button>
                      </div>
                    </div>

                    <div class="member-body">
                      <div class="member-column left">
                        <p class="detail-line">
                          <strong>Role:</strong>
                          <span class="role-accent">{{ slot.role || 'Unassigned' }}</span>
                        </p>

                        <div class="loadout-row">
                          <label class="disposable">
                            <input
                              type="checkbox"
                              :checked="!!slot.disposable"
                              @change="onToggleDisposable(detailKey, sIdx, $event.target.checked)"
                            />
                            Disposable Rocket ({{ DISPOSABLE_COST }}pt)
                          </label>
                        </div>

                        <div class="loadout-row">
                          <label class="primary-label">Assigned Certification</label>
                          <select
                            class="loadout-select"
                            :value="slot.cert || ''"
                            @change="onChangeCert(detailKey, sIdx, $event.target.value)"
                          >
                            <option value="">None / Standard</option>
                            <option v-for="c in getCertsForPersonId(slot.id)" :key="c" :value="c">
                              {{ c }}{{ certPointSuffix(c) }}
                            </option>
                          </select>
                        </div>
                      </div>

                      <div class="member-column right">
                        <p><strong>Certifications:</strong></p>
                        <div class="cert-list">
                          <div v-for="(label, cidx) in certLabels" :key="label" class="cert-row">
                            <span class="cert-checkbox" :class="{ checked: (slot.cert || '') === label || hasCertId(slot.id, cidx) }">
                              <span v-if="(slot.cert || '') === label || hasCertId(slot.id, cidx)" class="checkbox-dot"></span>
                            </span>
                            <span class="cert-label">{{ label }}</span>
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
              <button
                type="button"
                class="btn primary"
                :disabled="busy || !apiBase"
                @click="saveRemote(detailKey)"
              >
                {{ busy ? 'Saving…' : 'Save Chalk (Remote)' }}
              </button>
              <button type="button" class="btn" :disabled="busy || !apiBase" @click="loadRemote(detailKey)">
                Load Chalk (Remote)
              </button>
              <span v-if="remoteMeta[detailKey]?.savedBy" class="muted small">Last saved by {{ remoteMeta[detailKey].savedBy }}</span>
            </div>
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

        <div class="squad-modal-scroll compact-list">
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
            <button class="btn ghost small" @click="refreshOverview" :disabled="overview.loading">
              {{ overview.loading ? 'Refreshing…' : 'Refresh' }}
            </button>
            <button class="btn small" @click="clearAllSaved" :disabled="overview.loading">Clear All Saved</button>
            <button class="squad-close" @click="closeOverview">✕</button>
          </div>
        </div>

        <div v-if="overview.error" class="warn" style="margin-top:.5rem">{{ overview.error }}</div>

        <div class="squad-modal-scroll">
          <div class="overview-grid two-by-two">
            <div v-for="u in chalkUnits" :key="u.key" class="overview-card">
              <div class="overview-head">
                <div class="left">
                  <div class="title">{{ u.title }}</div>
                  <div class="meta">
                    <template v-if="ovItem(u.key)">
                      <span v-if="ovItem(u.key).savedBy">by {{ ovItem(u.key).savedBy }}</span>
                      <span v-if="ovItem(u.key).savedAt">at {{ formatDate(ovItem(u.key).savedAt) }}</span>
                      <span v-if="ovItem(u.key).points !== undefined">· {{ ovItem(u.key).points }} pts</span>
                      <span v-if="!ovItem(u.key).slots?.length" class="chip muted">Saved record found but empty — showing default</span>
                    </template>
                    <template v-else>
                      <span class="chip muted">Not saved — showing default</span>
                      <span>· {{ slotsPoints(u.slots) }} pts</span>
                    </template>
                  </div>
                </div>
                <div class="right">
                  <button class="btn small" :disabled="!apiBase || overview.loading" @click="loadRemote(u.key)">
                    Load Saved
                  </button>
                </div>
              </div>

              <div class="overview-body">
                <div
                  class="slot-row"
                  v-for="(s, idx) in (ovItem(u.key) && ovItem(u.key).slots?.length ? ovItem(u.key).slots : u.slots)"
                  :key="idx"
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
/* cookie reader */
function readCookie(name) {
  try {
    const target = `${name}=`;
    const parts = document.cookie ? document.cookie.split(';') : [];
    for (let i = 0; i < parts.length; i++) {
      const v = parts[i].trim();
      if (v.startsWith(target)) return decodeURIComponent(v.substring(target.length));
    }
    return '';
  } catch { return ''; }
}

/* optional Netlify Identity */
async function netlifyIdentityToken() {
  try {
    const id = window.netlifyIdentity;
    if (!id) return '';
    const user = id.currentUser();
    if (!user) return '';
    const token = await user.jwt();
    return token || '';
  } catch { return ''; }
}

export default {
  name: "DeploymentView",
  props: {
    animate: { type: Boolean, default: true },
    orbat: { type: Array, default: () => [] },
    execUrl: { type: String, default: "" },
    secret: { type: String, default: "PLEX" },
    token:  { type: String, default: "" },
    defaultsCsvUrl: { type: String, default: "" },
    rankIconBase: { type: String, default: () => (typeof window !== 'undefined' && window.RANK_ICON_BASE) ? window.RANK_ICON_BASE : '/ranks' },
    rankIconExt:  { type: String, default: 'png' },
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
        Rifleman: 0, Grenadier: 1, Breacher: 1, RTO: 1,
        "Machine Gunner": 2, Marksman: 2, "Combat Engineer": 2,
        "Anti Tank": 3, Corpsmen: 3, PJ: 3, Pilot: 3, NCO: 0, Officer: 0,
      },
      certLabels: [
        "Rifleman","Machine Gunner","Anti Tank","Corpsmen","Combat Engineer",
        "Marksman","Breacher","Grenadier","Pilot","RTO","PJ","NCO","Officer",
      ],
      identityToken: "",
      RANK_ORDER: {
        "GEN": 1, "COL": 2, "MAJ": 3, "CPT": 4, "1LT": 5, "2LT": 6,
        "CWO4": 7, "CWO3": 8, "CWO2": 9, "WO": 10,
        "GYSGT": 11, "SSGT": 12, "SSG": 13, "SGT": 14, "CPL": 15, "LCPL": 16, "PFC": 17, "PV2": 18, "PVT": 19
      },
      overview: {
        open: false,
        loading: false,
        error: "",
        items: {} // key -> { version, savedBy, savedAt, slots[], points }
      }
    };
  },
  async created() {
    this.ensureDeviceId();
    // build personnel pool and units from orbat (helpers below)
    this.personnel = this.buildPersonnelPool(this.orbat);
    this.ensureUnitsBuilt(this.orbat);
    this.identityToken = await netlifyIdentityToken();
  },
  mounted() { this.triggerFlicker(0); },
  computed: {
    chalkUnits() { return this.plan.units; },
    currentUnit() { return this.plan.units.find(u => u.key === this.detailKey) || null; },
    currentSlotTitle() {
      if (!this.picker.open) return "";
      const g = this.plan.units.find(u => u.key === this.picker.unitKey);
      return g ? `${g.title} #${this.picker.slotIdx + 1}` : "";
    },
    filteredPersonnel() {
      const q = this.picker.query.trim().toLowerCase();
      const base = this.personnel.filter(
        p => !q ||
          (p.name || "").toLowerCase().includes(q) ||
          (p.callsign || "").toLowerCase().includes(q) ||
          (p.role || "").toLowerCase().includes(q)
      );
      return this.picker.onlyFree ? base.filter(p => !this.findAssignment(p.id)) : base;
    },
    filteredAndSortedPersonnel() {
      const list = this.filteredPersonnel.slice();
      const sort = this.picker.sort || "name_asc";
      const rankVal = (r) => {
        const code = this.rankLabel(r || "");
        return this.RANK_ORDER[code] || 999;
      };
      const assignedWeight = (p) => (this.findAssignment(p.id) ? 0 : 1);

      if (sort === "name_asc" || sort === "name_desc") {
        list.sort((a,b) => (a.name||"").localeCompare(b.name||"", undefined, {sensitivity:"base"}));
        if (sort === "name_desc") list.reverse();
      } else if (sort === "role_asc") {
        list.sort((a,b) => (a.role||"").localeCompare(b.role||"", undefined, {sensitivity:"base"}));
      } else if (sort === "rank_desc") {
        list.sort((a,b) => rankVal(a.rank) - rankVal(b.rank));
      } else if (sort === "rank_asc") {
        list.sort((a,b) => rankVal(b.rank) - rankVal(a.rank));
      } else if (sort === "assigned_first") {
        list.sort((a,b) => assignedWeight(a) - assignedWeight(b));
      } else if (sort === "assigned_last") {
        list.sort((a,b) => assignedWeight(b) - assignedWeight(a));
      }
      return list;
    },
    authToken() {
      const tProp = (this.token || "").trim();
      if (tProp) return tProp;
      const tId = (this.identityToken || "").trim();
      if (tId) return tId;
      try {
        const ls = typeof localStorage !== "undefined" ? localStorage : null;
        const ss = typeof sessionStorage !== "undefined" ? sessionStorage : null;
        const keys = ["token","authToken","jwt","access_token","userToken","GAS_TOKEN","GAS_JWT"];
        for (const k of keys) {
          const v = (ls?.getItem(k) || ss?.getItem(k) || "").trim();
          if (v) return v;
        }
      } catch {}
      try {
        const ck = ["token","authToken","jwt","access_token"];
        for (const c of ck) {
          const v = (readCookie(c) || "").trim();
          if (v) return v;
        }
      } catch {}
      return "";
    },
    authModeLabel() { return this.authToken ? "User mode (token)" : "Device mode (anonymous)"; },
    apiBase() {
      const direct = (this.execUrl || "").trim();
      if (direct) return direct;
      try { if (typeof window !== "undefined" && window.DEPLOYMENT_EXEC_URL) return String(window.DEPLOYMENT_EXEC_URL).trim(); } catch {}
      try {
        const ls = typeof localStorage !== "undefined" ? localStorage : null;
        const ss = typeof sessionStorage !== "undefined" ? sessionStorage : null;
        const ns = ls?.getItem("deploymentExecUrl") || ss?.getItem("deploymentExecUrl") || "";
        if (ns && ns.trim()) return ns.trim();
      } catch {}
      try {
        const meta = document?.querySelector('meta[name="apps-script-exec"]')?.content || "";
        if (meta && meta.trim()) return meta.trim();
      } catch {}
      try {
        if (typeof APP_EXEC_URL !== "undefined" && APP_EXEC_URL) return String(APP_EXEC_URL).trim();
        if (typeof window !== "undefined" && window.APP_EXEC_URL) return String(window.APP_EXEC_URL).trim();
      } catch {}
      try {
        const ls = typeof localStorage !== "undefined" ? localStorage : null;
        const ss = typeof sessionStorage !== "undefined" ? sessionStorage : null;
        const a = ls?.getItem("execUrl") || ss?.getItem("execUrl") || "";
        if (a && a.trim()) return a.trim();
      } catch {}
      return "";
    },
    pointsUsed() { return this.calcUnitPoints(this.currentUnit); },
  },
  methods: {
    /* ORBAT normalization: supports fireteams as Array or Object map */
    fireteamsArray(fireteams) {
      if (Array.isArray(fireteams)) return fireteams;
      if (fireteams && typeof fireteams === "object") return Object.values(fireteams);
      return [];
    },

    /* rank helpers */
    rankLabel(raw) {
      const t = String(raw || "").trim();
      if (!t) return "";
      const map = {
        pvt:"PVT", pv2:"PV2", pfc:"PFC", lcpl:"LCPL", cpl:"CPL", sgt:"SGT",
        ssg:"SSG", ssgt:"SSGT", gysgt:"GYSGT",
        "2lt":"2LT", "1lt":"1LT", lt:"LT", cpt:"CPT", capt:"CPT", maj:"MAJ",
        col:"COL", gen:"GEN", wo:"WO", cwo2:"CWO2", cwo3:"CWO3", cwo4:"CWO4"
      };
      const k = t.toLowerCase();
      return map[k] || t.toUpperCase();
    },
    extractRank(member) {
      const r = member?.rank || member?.grade;
      if (r) return this.rankLabel(r);
      const candidates = [member?.name, member?.callsign].filter(Boolean);
      for (const c of candidates) {
        const tok = String(c).trim().split(/\s+/)[0] || "";
        const norm = this.rankLabel(tok);
        if (norm && /^[A-Z0-9]{2,6}$/.test(norm)) {
          const known = ["PVT","PV2","PFC","LCPL","CPL","SGT","SSG","SSGT","GYSGT","2LT","1LT","LT","CPT","MAJ","COL","GEN","WO","CWO2","CWO3","CWO4"];
          if (known.includes(norm)) return norm;
        }
      }
      return "";
    },
    rankFor(personId) {
      const p = this.personnel.find(pp => String(pp.id) === String(personId));
      return p?.rank ? this.rankLabel(p.rank) : "";
    },
    rankIcon(code) {
      if (!code) return "";
      const base = (this.rankIconBase || "/ranks").replace(/\/+$/,"");
      const ext = (this.rankIconExt || "png").replace(/^\.+/,"");
      return `${base}/${code}.${ext}`;
    },
    onRankImgError(ev) { ev.target.style.display = 'none'; },

    squadInitials(name) {
      if (!name) return "UNSC";
      const parts = String(name).trim().split(/\s+/);
      if (parts.length === 1) return parts[0].slice(0, 3).toUpperCase();
      return parts.map((p, i) => (i === parts.length - 1 && /\d+/.test(p) ? p : p[0]))
                  .join("").toUpperCase();
    },

    /* ===== STRICT configJSON parser (matches how chalks save/load) ===== */
    parseConfigJSONSlots(rec) {
      const txt = typeof rec?.configJSON === 'string' ? rec.configJSON
                : typeof rec?.data?.configJSON === 'string' ? rec.data.configJSON
                : null;
      if (!txt) return [];
      try {
        const obj = JSON.parse(txt);
        const slots = obj?.slots || obj?.config?.slots || [];
        if (!Array.isArray(slots)) return [];
        return slots.map(s => ({
          id: s?.id ?? null,
          name: s?.name ?? null,
          role: s?.role || "",
          cert: s?.cert || "",
          disposable: !!s?.disposable,
        }));
      } catch {
        return [];
      }
    },

    /* ===== Fallback extractor (very tolerant) ===== */
    extractSlotsFromAny(rec) {
      const normSlots = (arr) => (Array.isArray(arr) ? arr : []).map(s => ({
        id: s?.id ?? null,
        name: s?.name ?? null,
        role: s?.role || "",
        cert: s?.cert || "",
        disposable: !!s?.disposable
      }));
      const tryParse = (val) => {
        if (!val) return null;
        if (typeof val === "string") {
          try { return JSON.parse(val); } catch { return null; }
        }
        if (typeof val === "object") return val;
        return null;
      };

      if (Array.isArray(rec?.slots)) return normSlots(rec.slots);

      const topConfigJSON = tryParse(rec?.configJSON || rec?.loadoutJSON);
      if (topConfigJSON) {
        if (Array.isArray(topConfigJSON.slots)) return normSlots(topConfigJSON.slots);
        if (Array.isArray(topConfigJSON?.config?.slots)) return normSlots(topConfigJSON.config.slots);
      }
      const topConfig = tryParse(rec?.config);
      if (topConfig) {
        if (Array.isArray(topConfig.slots)) return normSlots(topConfig.slots);
        if (Array.isArray(topConfig?.config?.slots)) return normSlots(topConfig.config.slots);
      }

      const d = rec?.data || {};
      if (Array.isArray(d.slots)) return normSlots(d.slots);
      const dConfigJSON = tryParse(d.configJSON || d.loadoutJSON);
      if (dConfigJSON) {
        if (Array.isArray(dConfigJSON.slots)) return normSlots(dConfigJSON.slots);
        if (Array.isArray(dConfigJSON?.config?.slots)) return normSlots(dConfigJSON.config.slots);
      }
      const dConfig = tryParse(d.config);
      if (dConfig) {
        if (Array.isArray(dConfig.slots)) return normSlots(dConfig.slots);
        if (Array.isArray(dConfig?.config?.slots)) return normSlots(dConfig.config.slots);
      }

      const recordish = rec?.record || rec?.item || {};
      if (Array.isArray(recordish.slots)) return normSlots(recordish.slots);
      const rConfigJSON = tryParse(recordish.configJSON);
      if (rConfigJSON) {
        if (Array.isArray(rConfigJSON.slots)) return normSlots(rConfigJSON.slots);
        if (Array.isArray(rConfigJSON?.config?.slots)) return normSlots(rConfigJSON.config.slots);
      }
      const rConfig = tryParse(recordish.config);
      if (rConfig) {
        if (Array.isArray(rConfig.slots)) return normSlots(rConfig.slots);
        if (Array.isArray(rConfig?.config?.slots)) return normSlots(rConfig.config.slots);
      }

      return [];
    },

    /* Overview helpers */
    ovItem(key) { return this.overview.items[key]; },
    formatDate(ts) {
      try { const d = new Date(ts); if (!isFinite(d)) return String(ts); return d.toLocaleString(); }
      catch { return String(ts); }
    },

    async openOverview() { this.overview.open = true; await this.refreshOverview(); },
    closeOverview() { this.overview.open = false; },

    async refreshOverview() {
      if (!this.apiBase) { this.overview.error = "execUrl missing"; return; }
      this.overview.loading = true; this.overview.error = "";
      try {
        const results = await Promise.allSettled(
          this.chalkUnits.map(u => this.apiPost("config:get", { unitId: u.key }))
        );

        const map = {};
        results.forEach((r, idx) => {
          const unitKey = this.chalkUnits[idx].key;
          if (r.status === "fulfilled" && r.value && r.value.ok && r.value.data) {
            const it = r.value.data;
            let slots = this.parseConfigJSONSlots(it);
            if (!slots.length) slots = this.extractSlotsFromAny(it);
            map[unitKey] = {
              version: Number(it.version || 0),
              savedBy: it.user || it.savedBy || "",
              savedAt: it.savedAt || it.updatedAt || it.timestamp || "",
              slots,
              points: this.slotsPoints(slots)
            };
          }
        });

        this.overview.items = map;
      } catch (e) {
        this.overview.error = String(e.message || e);
      } finally {
        this.overview.loading = false;
      }
    },

    async clearAllSaved() {
      if (!confirm("Clear ALL saved loadouts for all Chalks? This cannot be undone.")) return;
      this.overview.loading = true; this.overview.error = "";
      try {
        const res = await this.apiPost("config:clearAll", {});
        if (!res || res.ok !== true) {
          await Promise.allSettled(this.chalkUnits.map(u => this.apiPost("config:clear", { unitId: u.key })));
        }
        await this.refreshOverview();
      } catch (e) {
        this.overview.error = String(e.message || e);
      } finally {
        this.overview.loading = false;
      }
    },

    slotsPoints(slots) {
      return (slots || []).reduce((sum, s) => {
        if (!s.id) return sum;
        const certPts = this.CERT_POINTS[s.cert] ?? 0;
        const dispPts = s.disposable ? this.DISPOSABLE_COST : 0;
        return sum + certPts + dispPts;
      }, 0);
    },

    async resetPlan() {
      this.detailError = ""; this.apiError = ""; this.debugInfo = "";
      const url = this.defaultsCsvUrl || (typeof window !== "undefined" ? window.DEFAULTS_CSV_URL : "");
      if (url) {
        try {
          const res = await fetch(url, { cache: "no-store" });
          if (!res.ok) throw new Error(`CSV HTTP ${res.status}`);
          const text = await res.text();
          const defaults = this.parseCsvDefaults(text);
          const updated = this.applyCsvDefaults(defaults);
          if (!updated) throw new Error("No matching chalks in CSV.");
          this.persistPlan(); this.triggerFlicker(0);
          this.debugInfo = "Reset from latest CSV defaults."; return;
        } catch (e) { this.apiError = `CSV fallback: ${String(e.message || e)}`; }
      }
      this.ensureUnitsBuilt(this.orbat); this.persistPlan(); this.triggerFlicker(0);
      if (!this.debugInfo) this.debugInfo = "Fallback defaults applied (ORBAT/template).";
    },

    parseCsvDefaults(csvText) {
      const rows = this.csvToRows(csvText);
      if (!rows.length) return {};
      const headers = rows[0].map(h => String(h || "").trim());
      const idx = {
        chalk: this.findHeader(headers, /(chalk|unit)/i),
        slot:  this.findHeader(headers, /(slot|index|position|#)/i, true),
        role:  this.findHeader(headers, /(role)/i),
        cert:  this.findHeader(headers, /(cert)/i, true),
        disp:  this.findHeader(headers, /(disp(osable)?|launcher)/i, true),
      };
      if (idx.chalk < 0 || idx.role < 0) throw new Error("Missing required columns (Chalk/Unit and Role).");
      const out = {};
      for (let r = 1; r < rows.length; r++) {
        const row = rows[r]; if (!row || !row.length) continue;
        const chalkName = String(row[idx.chalk] || "").trim(); if (!chalkName) continue;
        const role = String(row[idx.role] || "").trim();
        const slotNumRaw = idx.slot >= 0 ? String(row[idx.slot] || "").trim() : "";
        const cert = idx.cert >= 0 ? String(row[idx.cert] || "").trim() : "";
        const dispRaw = idx.disp >= 0 ? String(row[idx.disp] || "").trim().toUpperCase() : "";
        const disposable = ["Y","YES","TRUE","1"].includes(dispRaw);
        const key = chalkName.toLowerCase();
        if (!out[key]) out[key] = [];
        out[key].push({ idx: Number(slotNumRaw) || out[key].length + 1, role, cert, disposable: !!disposable });
      }
      Object.keys(out).forEach(k => out[k].sort((a,b)=> (a.idx||9999) - (b.idx||9999)));
      return out;
    },

    applyCsvDefaults(defaultsByChalk) {
      let touched = 0;
      const nextUnits = this.plan.units.map(u => {
        const key = String(u.title || "").trim().toLowerCase();
        const rows = defaultsByChalk[key];
        if (!rows) return u;
        const slots = rows.map(row => ({
          id: null, name: null, role: this.titleCase(row.role || ""), origStatus: "VACANT",
          cert: row.cert || "", disposable: !!row.disposable,
        }));
        const padded = this.isChalk(u.title) ? this.padSlots(slots, this.MIN_CHALK_SLOTS) : slots;
        touched++;
        return { ...u, slots: this.sortSlotsByRole(padded) };
      });
      if (touched > 0) {
        this.plan.units = nextUnits;
        if (!this.plan.units.find(x => x.key === this.detailKey) && this.plan.units.length) {
          this.detailKey = this.plan.units[0].key;
        }
        return true;
      }
      return false;
    },

    csvToRows(text) {
      const rows = []; let i = 0, field = "", row = [], inQuotes = false;
      const pushField = () => { row.push(field); field = ""; };
      const pushRow = () => { rows.push(row); row = []; };
      while (i < text.length) {
        const ch = text[i];
        if (inQuotes) {
          if (ch === '"') { if (text[i+1] === '"') { field += '"'; i += 2; continue; } inQuotes = false; i++; continue; }
          field += ch; i++; continue;
        } else {
          if (ch === '"') { inQuotes = true; i++; continue; }
          if (ch === ',') { pushField(); i++; continue; }
          if (ch === '\n') { pushField(); pushRow(); i++; continue; }
          if (ch === '\r') { i++; continue; }
          field += ch; i++; continue;
        }
      }
      (field !== "" || row.length) && (pushField(), pushRow());
      return rows.filter(r => r.some(c => String(c).trim() !== ""));
    },

    findHeader(headers, regex, optional = false) {
      const idx = headers.findIndex(h => regex.test(String(h || "")));
      if (idx === -1 && !optional) return -1;
      return idx;
    },

    ensureUnitsBuilt(orbat) {
      const built = this.buildUnitsFromOrbat(orbat).filter(u => this.isPointsUnit(u.title));
      if (built.length === 0) {
        this.plan.units = this.makeDefaultChalks(4, this.MIN_CHALK_SLOTS);
        this.detailKey = this.plan.units[0].key;
        this.debugInfo = "Fallback to default Chalk 1–4 (no/invalid ORBAT).";
        return;
      }
      built.forEach(u => u.slots.forEach(s => { if (typeof s.disposable === "undefined") s.disposable = false; }));
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
        const id = this.makeDeviceId();
        localStorage.setItem(key, id);
        this.deviceId = id;
      } catch { this.deviceId = this.makeDeviceId(); }
    },

    makeDeviceId() {
      const r = (crypto && crypto.getRandomValues) ? crypto.getRandomValues(new Uint8Array(12)) : Array.from({length:12},()=>Math.floor(Math.random()*256));
      return Array.from(r).map(b => b.toString(16).padStart(2,'0')).join('');
    },

    async apiPost(action, body, raw = false) {
      if (!this.apiBase) throw new Error("execUrl missing");

      const ls = typeof localStorage !== "undefined" ? localStorage : null;
      const ss = typeof sessionStorage !== "undefined" ? sessionStorage : null;

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
        } catch { return null; }
      };

      // Preferred: staff/admin session created by src/utils/adminAuth.js
      const adminUser = readJson(ss?.getItem("admin:user") || "");
      const adminRole = (readStr("admin:role") || readStr("authRole") || String(adminUser?.role || "")).trim();
      const adminTokenRaw = (readStr("admin:token") || readStr("authToken") || "").trim();
      const adminAuth = adminTokenRaw
        ? (adminTokenRaw.toLowerCase().startsWith("bearer ") ? adminTokenRaw : `Bearer ${adminTokenRaw}`)
        : "";

      const jwtPayload = decodeJwtPayload(adminAuth || adminTokenRaw);
      const jwtUserMeta = jwtPayload?.user_metadata || jwtPayload?.user_metadata || {};
      const jwtAppMeta = jwtPayload?.app_metadata || {};

      const adminDisplay =
        String(
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

      // Optional: Netlify Identity widget (if present on site)
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

      // Legacy fallback
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
      if (isDirectGAS) {
        // Apps Script web apps often do not expose request headers; pass role/user via query for compatibility.
        if (candidateRole) urlObj.searchParams.set("X-Role", candidateRole);
        if (candidateUser) urlObj.searchParams.set("X-User", candidateUser);
        if (authHeader && !candidateRole) urlObj.searchParams.set("Authorization", authHeader);
      }
      const url = urlObj.toString();

      const res = await fetch(url, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          ...(authHeader ? { Authorization: authHeader } : {}),
          ...(candidateUser ? { "X-User": candidateUser } : {}),
          ...(candidateRole ? { "X-Role": candidateRole } : {}),
        },
        body: JSON.stringify(payload),
      });

      if (!res) throw new Error("No response from API");
      if (raw) return res;
      const json = await res.text();
      try { return JSON.parse(json || "{}"); } catch { return { ok: false, raw: json }; }
    },

    /* ======= Slot / UI actions ======= */

    fillFromRoster(unitKey) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];
      const sorted = (g.slots || []).slice().map(s => ({ ...s }));
      // arrange by ROLE_ORDER then existing filled status
      sorted.sort((a, b) => {
        const ra = this.ROLE_ORDER.indexOf((a.role || "").toLowerCase());
        const rb = this.ROLE_ORDER.indexOf((b.role || "").toLowerCase());
        if (ra !== rb) return (ra === -1 ? 999 : ra) - (rb === -1 ? 999 : rb);
        const fa = !!a.id ? 0 : 1;
        const fb = !!b.id ? 0 : 1;
        return fa - fb;
      });

      const newG = { ...g, slots: sorted };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
      this.detailError = "";
    },

    clearGroup(unitKey) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;

      const g = this.plan.units[idx];
      const emptied = g.slots.map(s => ({
        ...s,
        id: null,
        name: null,
        cert: "",
        disposable: false,
        origStatus: s.origStatus === "CLOSED" ? "CLOSED" : "VACANT",
      }));

      const newG = { ...g, slots: this.sortSlotsByFireteam(emptied, g.fireteams || []) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
      this.detailError = "";
    },

    addSlot(unitKey) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];
      const newSlots = g.slots.slice();
      newSlots.push({ id: null, name: null, role: "", origStatus: "VACANT", cert: "", disposable: false });
      const newG = { ...g, slots: this.sortSlotsByRole(newSlots) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
    },

    removeSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];
      const newSlots = g.slots.slice();
      newSlots.splice(slotIdx, 1);
      const newG = { ...g, slots: this.sortSlotsByRole(newSlots) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
      this.detailError = "";
    },

    onChangeCert(unitKey, slotIdx, nextCert) {
      const uIdx = this.plan.units.findIndex(u => u.key === unitKey);
      if (uIdx < 0) return;
      const unit = this.plan.units[uIdx];
      const slot = unit.slots[slotIdx];
      const prevPts = (this.CERT_POINTS[slot.cert] ?? 0) + (slot.disposable ? this.DISPOSABLE_COST : 0);
      const nextPts = (this.CERT_POINTS[nextCert] ?? 0) + (slot.disposable ? this.DISPOSABLE_COST : 0);
      const delta = nextPts - prevPts;
      if (this.wouldExceedCap(unitKey, Math.max(0, delta))) { this.detailError = `Point cap ( ${this.SQUAD_POINT_CAP} ) would be exceeded.`; return; }
      this.detailError = "";
      const newSlots = unit.slots.slice();
      newSlots[slotIdx] = { ...slot, cert: nextCert };
      const newU = { ...unit, slots: this.sortSlotsByRole(newSlots) };
      this.plan.units = this.plan.units.map((u, i) => (i === uIdx ? newU : u));
      this.persistPlan();
    },

    onToggleDisposable(unitKey, slotIdx, checked) {
      const uIdx = this.plan.units.findIndex(u => u.key === unitKey);
      if (uIdx < 0) return;
      const unit = this.plan.units[uIdx];
      const slot = unit.slots[slotIdx];
      const add = checked ? this.DISPOSABLE_COST : 0;
      const remove = !checked ? this.DISPOSABLE_COST : 0;
      const delta = add - remove;
      if (this.wouldExceedCap(unitKey, Math.max(0, delta))) { this.detailError = `Point cap ( ${this.SQUAD_POINT_CAP} ) would be exceeded.`; return; }
      this.detailError = "";
      const newSlots = unit.slots.slice();
      newSlots[slotIdx] = { ...slot, disposable: !!checked };
      const newU = { ...unit, slots: this.sortSlotsByRole(newSlots) };
      this.plan.units = this.plan.units.map((u, i) => (i === uIdx ? newU : u));
      this.persistPlan();
    },

    /* ======= Helper methods added/restored to avoid runtime errors ======= */

    filledCount(unit) {
      if (!unit || !Array.isArray(unit.slots)) return 0;
      return unit.slots.filter(s => s && s.id).length;
    },

    triggerFlicker(delayMs = 0) {
      try {
        this.animateView = false;
        this.animationDelay = `${delayMs}ms`;
        // small flicker to trigger CSS animation
        clearTimeout(this._flickerTimer);
        this._flickerTimer = setTimeout(() => {
          this.animateView = true;
          // turn off after animation to allow retrigger
          this._flickerTimer = setTimeout(() => { this.animateView = false; }, 420);
        }, 10);
      } catch (e) { /* noop */ }
    },

    buildPersonnelPool(orbat) {
      // Build a flat personnel array from ORBAT (tolerant)
      try {
        const out = [];
        const addMember = (m) => {
          if (!m) return;
          const id = m.id ?? m.personId ?? m.uid ?? m.uniqueId ?? (m.callsign || m.name || "") ;
          out.push({
            id: id,
            name: m.name || m.fullName || m.displayName || "",
            callsign: m.callsign || "",
            role: m.role || m.slot || "",
            rank: m.rank || m.grade || "",
            certifications: m.certs || m.certifications || [],
            element: m.element || m.fireteam || "",
          });
        };
        if (Array.isArray(orbat)) {
          (orbat || []).forEach(unit => {
            if (Array.isArray(unit.members)) unit.members.forEach(addMember);
            if (Array.isArray(unit.personnel)) unit.personnel.forEach(addMember);
            // check fireteams
            if (Array.isArray(unit.fireteams)) unit.fireteams.forEach(ft => (ft.slots||[]).forEach(s => addMember(s.member || s)));
          });
        } else if (orbat && typeof orbat === "object") {
          // object map
          Object.values(orbat).forEach(unit => {
            if (Array.isArray(unit.members)) unit.members.forEach(addMember);
            if (Array.isArray(unit.personnel)) unit.personnel.forEach(addMember);
          });
        }
        // de-duplicate by id
        const map = {};
        out.forEach(p => {
          const k = String(p.id || p.name || Math.random());
          if (!map[k]) map[k] = p;
        });
        return Object.values(map);
      } catch (e) { return []; }
    },

    // ---- Minimal shims / reasonable implementations so the view works ----

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

    selectPersonnel(person) {
      // assign selected personnel to the open picker slot
      if (!this.picker.open) return;
      const uIdx = this.plan.units.findIndex(u => u.key === this.picker.unitKey);
      if (uIdx < 0) return;
      const unit = this.plan.units[uIdx];
      const sIdx = this.picker.slotIdx;
      const newSlots = unit.slots.slice();
      newSlots[sIdx] = { ...(newSlots[sIdx]||{}), id: person.id, name: person.name || person.callsign || "", cert: newSlots[sIdx]?.cert || "", disposable: !!newSlots[sIdx]?.disposable, origStatus: newSlots[sIdx]?.origStatus || "FILLED" };
      this.plan.units = this.plan.units.map((u,i)=> i===uIdx ? {...unit, slots: this.sortSlotsByRole(newSlots)} : u);
      this.persistPlan();
      this.closePicker();
    },

    openPicker(unitKey, slotIdx) {
      this.picker.open = true;
      this.picker.unitKey = unitKey;
      this.picker.slotIdx = slotIdx;
      this.picker.query = "";
      this.picker.onlyFree = false;
      this.picker.sort = "name_asc";
    },

    closePicker() {
      this.picker.open = false;
      this.picker.unitKey = "";
      this.picker.slotIdx = -1;
      this.picker.query = "";
    },

    clearSlot(unitKey, slotIdx) {
      const uIdx = this.plan.units.findIndex(u => u.key === unitKey);
      if (uIdx < 0) return;
      const unit = this.plan.units[uIdx];
      const newSlots = unit.slots.slice();
      newSlots[slotIdx] = { ...newSlots[slotIdx], id: null, name: null, cert: "", disposable: false, origStatus: newSlots[slotIdx]?.origStatus || "VACANT" };
      this.plan.units = this.plan.units.map((u,i)=> i===uIdx ? {...unit, slots: this.sortSlotsByRole(newSlots)} : u);
      this.persistPlan();
    },

    removeSlotAssignment(personId) {
      // remove assignment for personId
      for (let ui = 0; ui < this.plan.units.length; ui++) {
        const u = this.plan.units[ui];
        const newSlots = u.slots.slice();
        let changed = false;
        for (let si = 0; si < newSlots.length; si++) {
          if (String(newSlots[si].id) === String(personId)) {
            newSlots[si] = { ...newSlots[si], id: null, name: null, cert: "", disposable: false, origStatus: newSlots[si]?.origStatus || "VACANT" };
            changed = true;
          }
        }
        if (changed) {
          this.plan.units = this.plan.units.map((x,i)=> i===ui ? {...u, slots: this.sortSlotsByRole(newSlots)} : x);
          this.persistPlan();
          return;
        }
      }
    },

    saveRemote(unitKey) {
      // simple remote save: post config:set
      if (!this.apiBase) { this.apiError = "execUrl missing"; return; }
      const unit = this.plan.units.find(u => u.key === unitKey);
      if (!unit) return;
      const payload = {
        unitId: unit.key,
        configJSON: JSON.stringify({ slots: unit.slots, title: unit.title })
      };
      this.busy = true;
      this.apiPost("config:set", payload).then(res => {
        if (!res || !res.ok) this.apiError = "Remote save failed";
        else this.remoteMeta[unitKey] = { savedBy: res.user || res.savedBy || "", savedAt: res.savedAt || res.timestamp || Date.now(), points: this.slotsPoints(unit.slots) };
      }).catch(e => { this.apiError = String(e.message || e); }).finally(()=> this.busy = false);
    },

    async loadRemote(unitKey) {
      if (!this.apiBase) { this.apiError = "execUrl missing"; return; }
      this.busy = true;
      try {
        const res = await this.apiPost("config:get", { unitId: unitKey });
        if (!res || !res.ok || !res.data) { this.apiError = "Remote load failed"; return; }
        const rec = res.data;
        let slots = this.parseConfigJSONSlots(rec);
        if (!slots.length) slots = this.extractSlotsFromAny(rec);
        const idx = this.plan.units.findIndex(u => u.key === unitKey);
        if (idx >= 0) {
          const g = this.plan.units[idx];
          const padded = this.isChalk(g.title) ? this.padSlots(slots, this.MIN_CHALK_SLOTS) : slots;
          this.plan.units = this.plan.units.map((u,i)=> i===idx ? {...g, slots: this.sortSlotsByRole(padded)} : u);
          this.persistPlan();
        }
      } catch (e) { this.apiError = String(e.message || e); } finally { this.busy = false; }
    },

    persistPlan() {
      try {
        localStorage.setItem(this.STORAGE_KEY, JSON.stringify(this.plan));
      } catch (e) { /* ignore */ }
    },

    wouldExceedCap(unitKey, addPoints) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      const unit = idx >= 0 ? this.plan.units[idx] : null;
      const curr = this.calcUnitPoints(unit);
      return (curr + (addPoints || 0)) > this.SQUAD_POINT_CAP;
    },

    calcUnitPoints(unit) {
      if (!unit) return 0;
      return (unit.slots || []).reduce((sum, s) => {
        if (!s || !s.id) return sum;
        const certPts = this.CERT_POINTS[s.cert] ?? 0;
        const dispPts = s.disposable ? this.DISPOSABLE_COST : 0;
        return sum + certPts + dispPts;
      }, 0);
    },

    findPersonById(id) {
      return this.personnel.find(p => String(p.id) === String(id)) || null;
    },

    getCertsForPersonId(id) {
      // return cert list; minimally returns certLabels
      return this.certLabels || [];
    },

    certPointSuffix(cert) {
      const pts = this.CERT_POINTS[cert] ?? 0;
      return pts ? ` (+${pts})` : "";
    },

    hasCertId(personId, cidx) {
      const p = this.findPersonById(personId);
      if (!p) return false;
      const label = this.certLabels[cidx];
      if (!label) return false;
      return (p.certifications || []).includes(label);
    },

    titleCase(s) { if (!s) return ""; return String(s).toLowerCase().split(/\s+/).map(w=> w.charAt(0).toUpperCase()+w.slice(1)).join(" "); },

    padSlots(slots, size) {
      const out = (slots || []).slice();
      while (out.length < size) out.push({ id: null, name: null, role: "", origStatus: "VACANT", cert: "", disposable: false });
      return out;
    },

    isChalk(title) {
      return /chalk/i.test(String(title || ""));
    },

    isPointsUnit(title) {
      // consider anything that looks like a chalk or squad
      return !!title;
    },

    keyFromName(name) { return String(name || "").toLowerCase().replace(/\s+/g, "-"); },

    buildUnitsFromOrbat(orbat) {
      // minimal conversion: if orbat is provided, map into {key,title,slots}
      const units = [];
      if (!orbat) return units;
      if (Array.isArray(orbat)) {
        orbat.forEach((u, i) => {
          const slots = (u.slots || u.members || []).map(s => ({
            id: s.id || s.personId || null,
            name: s.name || s.fullName || null,
            role: s.role || s.slot || "",
            cert: s.cert || "",
            disposable: !!s.disposable,
            origStatus: s.origStatus || (s.id ? "FILLED" : "VACANT"),
          }));
          const key = u.key || this.keyFromName(u.title || `unit-${i+1}`);
          units.push({ key, title: u.title || `Unit ${i+1}`, slots: this.padSlots(slots, this.MIN_CHALK_SLOTS), fireteams: u.fireteams || [] });
        });
      } else {
        Object.keys(orbat).forEach(k => {
          const u = orbat[k];
          const slots = (u.slots || u.members || []).map(s => ({
            id: s.id || s.personId || null,
            name: s.name || s.fullName || null,
            role: s.role || s.slot || "",
            cert: s.cert || "",
            disposable: !!s.disposable,
            origStatus: s.origStatus || (s.id ? "FILLED" : "VACANT"),
          }));
          units.push({ key: k, title: u.title || u.squad || k, slots: this.padSlots(slots, this.MIN_CHALK_SLOTS), fireteams: u.fireteams || [] });
        });
      }
      return units;
    },

    // sortSlotsByRole: basic stable ordering that keeps filled first and sorts by ROLE_ORDER then name
    sortSlotsByRole(slots) {
      const roleOrderMap = {};
      this.ROLE_ORDER.forEach((r, i) => roleOrderMap[r.toLowerCase()] = i);
      const cmp = (a,b) => {
        const as = !!a.id ? 0 : (a.origStatus === "CLOSED" ? 2 : 1);
        const bs = !!b.id ? 0 : (b.origStatus === "CLOSED" ? 2 : 1);
        if (as !== bs) return as - bs;
        const ra = roleOrderMap[(a.role||"").toLowerCase()] ?? 999;
        const rb = roleOrderMap[(b.role||"").toLowerCase()] ?? 999;
        if (ra !== rb) return ra - rb;
        return String((a.name||a.role||"")).localeCompare(String(b.name||b.role||""), undefined, {sensitivity:"base"});
      };
      return (slots || []).slice().sort(cmp);
    },

    // sortSlotsByFireteam: group by provided fireteams if any, otherwise return as-is
    sortSlotsByFireteam(slots, fireteams) {
      // if explicit fireteams exist, attempt to keep slots' fireteam positions (best effort).
      // For now, return slots unchanged.
      return slots;
    },

    // small UI helpers used earlier
    formatAssignment(assign) {
      if (!assign) return "";
      const u = this.plan.units.find(x => x.key === assign.unitKey);
      return u ? `${u.title} #${assign.slotIdx+1}` : `${assign.unitKey}#${assign.slotIdx+1}`;
    },

    // ---------- CSV/members loader helpers included for compatibility ----------
    loadMembersCSV(text) {
      try {
        const rows = this.csvToRows(text || "");
        // not performing heavy parsing here; user code may handle
        return rows;
      } catch (e) { return []; }
    },

    /* end methods */
  }
};
</script>
