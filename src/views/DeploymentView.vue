<!-- File: src/views/DeploymentView.vue -->
<template>
  <div
    id="deploymentView"
    class="content-container"
    :class="{ animate: animateView }"
    :style="{ 'animation-delay': animationDelay }"
  >
    <section id="deployment" class="section-container">
      <div style="height: 52px; overflow: hidden">
        <div class="section-header clipped-medium-backward">
          <img src="/icons/license.svg" alt="Deployment Icon" />
          <h1>Deployment Planner</h1>
        </div>
        <div class="rhombus-back">&nbsp;</div>
      </div>

      <div class="section-content-container">
        <div class="deployment-shell">
          <div class="topbar">
            <div class="left">
              <div class="title">Chalk Planner</div>
              <div class="subtitle">
                Drag / assign personnel into chalk slots. Save to Google Sheet via Apps Script.
              </div>
            </div>
            <div class="right">
              <button class="btn ghost" @click="openOverview" :disabled="busy">Overview</button>
              <button class="btn ghost" @click="clearAllLocal" :disabled="busy">Reset Local</button>
            </div>
          </div>

          <div class="planner-grid">
            <!-- LEFT: units list -->
            <div class="panel left">
              <div class="panel-header">
                <div class="h">Chalks</div>
                <div class="muted small">Select a chalk to edit.</div>
              </div>

              <div class="unit-list">
                <div
                  v-for="u in chalkUnits"
                  :key="u.key"
                  class="unit-item"
                  :class="{ active: detailKey === u.key }"
                  @click="openUnit(u.key)"
                >
                  <div class="unit-title">{{ u.title }}</div>
                  <div class="unit-meta">
                    <span>{{ filledCount(ovItem(u.key) && ovItem(u.key).slots?.length ? ovItem(u.key) : u) }} / {{ (ovItem(u.key) && ovItem(u.key).slots?.length ? ovItem(u.key).slots : u.slots).length }}</span>
                    <span class="divider" />
                    <span class="muted">pts: {{ slotsPoints(ovItem(u.key) && ovItem(u.key).slots?.length ? ovItem(u.key).slots : u.slots) }}</span>
                  </div>
                  <div class="unit-last" v-if="remoteMeta[u.key]">
                    <span v-if="remoteMeta[u.key].savedBy" class="muted small">by {{ remoteMeta[u.key].savedBy }}</span>
                    <span v-if="remoteMeta[u.key].savedAt" class="muted small">at {{ formatDate(remoteMeta[u.key].savedAt) }}</span>
                  </div>
                </div>
              </div>

              <div class="panel-footer">
                <div class="muted small">Data source: ORBAT sheet + deployment configs.</div>
              </div>
            </div>

            <!-- RIGHT: unit details -->
            <div class="panel right">
              <div class="panel-header">
                <div class="h">Details</div>
                <div class="muted small">
                  <span v-if="currentUnit">{{ currentUnit.title }}</span>
                  <span v-else>Pick a chalk…</span>
                </div>
              </div>

              <div class="panel-body">
                <div class="toolbar">
                  <div class="toolbar-left">
                    <button class="btn small" @click="addSlot(detailKey)" :disabled="!currentUnit || busy">
                      Add Slot
                    </button>
                    <button class="btn small ghost" @click="clearGroup(detailKey)" :disabled="!currentUnit || busy">
                      Clear Chalk
                    </button>
                  </div>

                  <div class="toolbar-right">
                    <button
                      class="btn small ghost"
                      @click="loadRemote(detailKey)"
                      :disabled="!currentUnit || busy || !apiBase"
                      :title="!apiBase ? 'Exec URL missing' : 'Load from Sheet'"
                    >
                      {{ busy ? 'Loading…' : 'Load' }}
                    </button>

                    <button
                      class="btn small primary"
                      @click="saveRemote(detailKey)"
                      :disabled="!currentUnit || busy || !apiBase"
                      :title="!apiBase ? 'Exec URL missing' : 'Save to Sheet'"
                    >
                      {{ busy ? 'Saving…' : 'Save' }}
                    </button>
                  </div>
                </div>

                <div class="stats-row">
                  <span class="muted small">Slots: {{ currentUnit ? currentUnit.slots.length : 0 }}</span>
                  <span class="divider" />
                  <span class="muted small">Filled: {{ currentUnit ? filledCount(currentUnit) : 0 }}</span>
                  <span class="divider" />
                  <span v-if="currentUnit" class="pts big" :class="{ over: pointsUsed > SQUAD_POINT_CAP }">
                    Points: {{ pointsUsed }} / {{ SQUAD_POINT_CAP }}
                  </span>
                  <span class="divider" />
                  <span class="muted small">{{ authModeLabel }}</span>
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
                    <template v-for="ft in currentUnitFireteams" :key="`ftwrap-${detailKey}-${ft.name}`">
                      <div class="fireteam-row">
                        <span class="fireteam-title">{{ ft.name.toUpperCase() }}</span>
                        <span class="fireteam-count">({{ ft.slots.length }} SLOTS)</span>
                      </div>

                      <template v-for="wrap in ft.slots" :key="`wrap-${detailKey}-${wrap.idx}`">
                        <div
                          class="member-card"
                          :class="{ vacant: wrap.slot.origStatus === 'VACANT' && !wrap.slot.id, closed: wrap.slot.origStatus === 'CLOSED' }"
                        >
                          <!-- VACANT / CLOSED -->
                          <template v-if="(wrap.slot.origStatus === 'VACANT' && !wrap.slot.id) || wrap.slot.origStatus === 'CLOSED'">
                            <div class="member-header">
                              <div class="member-header-text">
                                <h3>{{ (wrap.slot.origStatus || 'VACANT').toUpperCase() }}</h3>
                                <p class="rank-line">
                                  <span class="rank">{{ wrap.slot.role || 'Slot' }}</span>
                                  <span class="id">UNFILLED SLOT</span>
                                </p>
                              </div>
                              <div style="display:flex; gap:.35rem; margin-left:auto;">
                                <button
                                  type="button"
                                  class="btn ghost xsmall"
                                  :disabled="wrap.slot.origStatus === 'CLOSED'"
                                  @click.stop="openPicker(detailKey, wrap.idx)"
                                >
                                  {{ wrap.slot.origStatus === 'CLOSED' ? 'Closed' : 'Assign' }}
                                </button>
                                <button type="button" class="btn ghost xsmall" @click.stop="removeSlot(detailKey, wrap.idx)">–</button>
                              </div>
                            </div>

                            <div class="member-body">
                              <div class="member-column left">
                                <p class="detail-line">
                                  <strong>Role:</strong>
                                  <span class="role-accent">{{ wrap.slot.role || 'Slot' }}</span>
                                </p>
                              </div>
                              <div class="member-column right">
                                <p><strong>Certifications:</strong></p>
                                <span class="cert-none">N/A</span>
                              </div>
                            </div>

                            <div class="member-footer">
                              <span>SLOT STATUS: {{ wrap.slot.origStatus || 'VACANT' }}</span>
                              <span>UNSC SYSTEMS DATABASE</span>
                            </div>
                          </template>

                          <!-- FILLED -->
                          <template v-else>
                            <div class="member-header">
                              <div class="member-header-text">
                                <div class="name-line">
                                  <img
                                    v-if="rankFor(wrap.slot.id)"
                                    class="rank-icon"
                                    :src="rankIcon(rankFor(wrap.slot.id))"
                                    :alt="rankFor(wrap.slot.id)"
                                    :title="rankFor(wrap.slot.id)"
                                    @error="onRankImgError($event)"
                                  />
                                  <h3>{{ (wrap.slot.name || 'UNKNOWN').toUpperCase() }}</h3>
                                </div>
                                <p class="rank-line">
                                  <span class="rank">{{ wrap.slot.role || 'N/A' }}</span>
                                  <span class="id">ID: {{ wrap.slot.id || 'N/A' }}</span>
                                </p>
                              </div>
                              <div style="display:flex; gap:.35rem; margin-left:auto;">
                                <button v-if="wrap.slot.id" type="button" class="btn ghost xsmall" @click.stop="clearSlot(detailKey, wrap.idx)">Clear</button>
                                <button type="button" class="btn ghost xsmall" @click.stop="removeSlot(detailKey, wrap.idx)">–</button>
                              </div>
                            </div>

                            <div class="member-body">
                              <div class="member-column left">
                                <p class="detail-line">
                                  <strong>Role:</strong>
                                  <span class="role-accent">{{ wrap.slot.role || 'Unassigned' }}</span>
                                </p>

                                <div class="loadout-row">
                                  <label class="disposable">
                                    <input
                                      type="checkbox"
                                      :checked="!!wrap.slot.disposable"
                                      @change="onToggleDisposable(detailKey, wrap.idx, $event.target.checked)"
                                    />
                                    Disposable Rocket ({{ DISPOSABLE_COST }}pt)
                                  </label>
                                </div>

                                <div class="loadout-row">
                                  <label class="primary-label">Assigned Certification</label>
                                  <select
                                    class="loadout-select"
                                    :value="wrap.slot.cert || ''"
                                    @change="onChangeCert(detailKey, wrap.idx, $event.target.value)"
                                  >
                                    <option value="">None / Standard</option>
                                    <option v-for="c in getCertsForPersonId(wrap.slot.id)" :key="c" :value="c">
                                      {{ c }}{{ certPointSuffix(c) }}
                                    </option>
                                  </select>
                                </div>
                              </div>

                              <div class="member-column right">
                                <p><strong>Certifications:</strong></p>
                                <div class="cert-list">
                                  <div v-for="(label, cidx) in certLabels" :key="label" class="cert-row">
                                    <span class="cert-checkbox" :class="{ checked: (wrap.slot.cert || '') === label || hasCertId(wrap.slot.id, cidx) }">
                                      <span v-if="(wrap.slot.cert || '') === label || hasCertId(wrap.slot.id, cidx)" class="checkbox-dot"></span>
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
                                :disabled="wrap.slot.origStatus === 'CLOSED'"
                                @click.stop="openPicker(detailKey, wrap.idx)"
                              >
                                {{ wrap.slot.id ? 'Swap' : (wrap.slot.origStatus === 'CLOSED' ? 'Closed' : 'Assign') }}
                              </button>
                              <span>UNSC SYSTEMS DATABASE</span>
                            </div>
                          </template>
                        </div>
                      </template>

                      <div class="fireteam-divider"></div>
                    </template>
                  </div>

                  <div class="actions-row">
                    <div class="left">
                      <button class="btn ghost small" @click="clearGroup(detailKey)" :disabled="busy">Clear</button>
                      <button class="btn ghost small" @click="loadRemote(detailKey)" :disabled="busy || !apiBase">Load</button>
                      <button class="btn small primary" @click="saveRemote(detailKey)" :disabled="busy || !apiBase">Save</button>
                    </div>
                    <div class="right">
                      <button class="btn ghost small" @click="openPicker(detailKey, 0)" :disabled="busy">Assign</button>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Picker Modal -->
          <div v-if="picker.open" class="picker-overlay" @click.self="closePicker">
            <div class="picker-modal">
              <div class="picker-header">
                <div class="picker-title">
                  Assign Personnel
                  <span class="muted small" v-if="currentSlotTitle">→ {{ currentSlotTitle }}</span>
                </div>
                <button class="picker-close" @click="closePicker">✕</button>
              </div>

              <div class="picker-controls">
                <input
                  class="picker-search"
                  placeholder="Search name / role / element…"
                  v-model="picker.query"
                />
                <label class="picker-check">
                  <input type="checkbox" v-model="picker.onlyFree" />
                  Show only unassigned
                </label>
                <select class="picker-sort" v-model="picker.sort">
                  <option value="name_asc">Name (A→Z)</option>
                  <option value="name_desc">Name (Z→A)</option>
                  <option value="role_asc">Role (A→Z)</option>
                  <option value="role_desc">Role (Z→A)</option>
                  <option value="rank_desc">Rank (High→Low)</option>
                  <option value="rank_asc">Rank (Low→High)</option>
                  <option value="assigned_first">Assigned First</option>
                  <option value="assigned_last">Unassigned First</option>
                </select>
              </div>

              <div class="picker-list">
                <div v-if="!filteredPersonnel.length" class="muted">
                  No matching personnel.
                </div>

                <div
                  v-for="p in filteredPersonnel"
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

              <div class="picker-footer">
                <button class="btn ghost small" @click="clearCurrentSlot" :disabled="busy">Clear Slot</button>
                <span class="muted small">Tip: selecting a person already assigned swaps them.</span>
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
                          Load
                        </button>
                      </div>
                    </div>

                    <div class="overview-body">
                      <div class="slots-compact">
                        <div
                          v-for="(s, idx) in (ovItem(u.key) && ovItem(u.key).slots?.length ? ovItem(u.key).slots : u.slots)"
                          :key="idx"
                          class="slot-row"
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

        </div>
      </div>
    </section>
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
      overview: { open: false, loading: false, error: "" },
      identityToken: "",
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
        Rifleman: 0, Grenadier: 1, Breacher: 1, "Machine Gunner": 1,
        "Anti Tank": 2, Corpsmen: 1, "Combat Engineer": 1, Marksman: 1,
        Pilot: 2, RTO: 1, PJ: 1, NCO: 0, Officer: 0,
      },
      certLabels: [
        "Rifleman","Machine Gunner","Anti Tank","Corpsmen","Combat Engineer",
        "Marksman","Breacher","Grenadier","Pilot","RTO","PJ","NCO","Officer",
      ],
    };
  },
  async mounted() {
    this.triggerFlicker();
    this.loadLocal();

    try { this.identityToken = await netlifyIdentityToken(); } catch {}
    this.deviceId = this.getDeviceId();

    this.personnel = this.buildPersonnelPool(this.orbat);

    if (!this.plan.units.length) {
      const units = this.buildUnitsFromOrbat(this.orbat);
      this.plan = { units };
      this.persistPlan();
    }

    // optional defaults CSV
    if (this.defaultsCsvUrl) {
      try {
        const r = await fetch(this.defaultsCsvUrl);
        if (r.ok) {
          const txt = await r.text();
          const members = this.loadMembersCSV(txt);
          if (members?.length) this.debugInfo = `Loaded defaults CSV (${members.length} rows)`;
        }
      } catch {}
    }
  },
  watch: {
    orbat: {
      deep: true,
      handler() {
        // Rebuild personnel from ORBAT
        this.personnel = this.buildPersonnelPool(this.orbat);

        // Ensure local plan has same units set
        const nextUnits = this.buildUnitsFromOrbat(this.orbat);

        // Merge plan units by key (preserve user assignments)
        const byKey = new Map();
        (this.plan.units || []).forEach(u => byKey.set(u.key, u));

        const merged = nextUnits.map(u => {
          const old = byKey.get(u.key);
          if (!old) return u;

          // Preserve slot assignments by index, but keep new ORBAT slot metadata.
          const mergedSlots = (u.slots || []).map((s, idx) => {
            const prev = old.slots && old.slots[idx] ? old.slots[idx] : null;
            if (!prev) return s;
            return {
              ...s,
              id: prev.id ?? s.id ?? null,
              name: prev.name ?? s.name ?? null,
              cert: prev.cert ?? s.cert ?? "",
              disposable: !!(prev.disposable ?? s.disposable),
              origStatus: s.origStatus === "CLOSED" ? "CLOSED" : (prev.id ? "FILLED" : (s.id ? "FILLED" : "VACANT")),
            };
          });

          return { ...u, slots: mergedSlots };
        });

        this.plan = { units: merged };
        this.persistPlan();
      }
    }
  },
  computed: {
    chalkUnits() {
      return (this.plan.units || []).filter(u => this.isChalk(u.title));
    },
    currentUnit() {
      return (this.plan.units || []).find(u => u.key === this.detailKey) || null;
    },
    currentSlotTitle() {
      const u = this.currentUnit;
      if (!u) return "";
      const idx = this.picker.slotIdx;
      if (idx < 0 || idx >= u.slots.length) return "";
      const s = u.slots[idx];
      return `${s.role || "Slot"} (#${idx + 1})`;
    },
    filteredPersonnel() {
      const q = String(this.picker.query || "").trim().toLowerCase();
      const onlyFree = !!this.picker.onlyFree;
      const sort = this.picker.sort || "name_asc";

      let list = (this.personnel || []).slice();

      const match = (p) => {
        if (!q) return true;
        const hay = `${p.name || ""} ${p.callsign || ""} ${p.role || ""} ${p.element || ""}`.toLowerCase();
        return hay.includes(q);
      };

      list = list.filter(match);

      if (onlyFree) {
        list = list.filter(p => !this.findAssignment(p.id));
      }

      const rankVal = (raw) => {
        const r = String(raw || "").toUpperCase().replace(/\s+/g, "");
        const order = [
          "MAJ","CAPT","1STLT","2NDLT",
          "CWO5","CWO4","CWO3","CWO2","WO",
          "GYSGT","SSGT","SGT","CPL","LCPL",
          "SPC4","SPC3","SPC2","SPC","PFC","PVT","RCT",
          "HMC","HM1","HM2","HM3","HN","HA","HR",
        ];
        const idx = order.indexOf(r);
        return idx === -1 ? 999 : idx;
      };

      const assignedWeight = (p) => (this.findAssignment(p.id) ? 0 : 1);

      if (sort === "name_asc") {
        list.sort((a,b) => String(a.name||"").localeCompare(String(b.name||""), undefined, {sensitivity:"base"}));
      } else if (sort === "name_desc") {
        list.sort((a,b) => String(b.name||"").localeCompare(String(a.name||""), undefined, {sensitivity:"base"}));
      } else if (sort === "role_asc") {
        list.sort((a,b) => String(a.role||"").localeCompare(String(b.role||""), undefined, {sensitivity:"base"}));
      } else if (sort === "role_desc") {
        list.sort((a,b) => String(b.role||"").localeCompare(String(a.role||""), undefined, {sensitivity:"base"}));
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
    currentUnitFireteams() {
      const unit = this.currentUnit;
      if (!unit) return [];
      const slots = Array.isArray(unit.slots) ? unit.slots : [];

      const map = new Map();
      for (let idx = 0; idx < slots.length; idx += 1) {
        const slot = slots[idx] || {};
        const ftRaw = slot.fireteam ?? slot.fireTeam ?? slot.element ?? slot.team ?? slot.group ?? "";
        const name = this.normalizeFireteamName(ftRaw);
        if (!map.has(name)) map.set(name, []);
        map.get(name).push({ slot, idx });
      }

      const declaredNames = (Array.isArray(unit.fireteams) ? unit.fireteams : [])
        .map((x) => (typeof x === "string" ? x : x?.name))
        .filter(Boolean)
        .map((n) => this.normalizeFireteamName(n));

      const uniqueDeclared = [];
      for (const n of declaredNames) if (n && !uniqueDeclared.includes(n) && map.has(n)) uniqueDeclared.push(n);

      const remaining = Array.from(map.keys()).filter((n) => !uniqueDeclared.includes(n));
      remaining.sort((a, b) => {
        const ka = this.fireteamOrderKey(a);
        const kb = this.fireteamOrderKey(b);
        if (ka !== kb) return ka - kb;
        return String(a).localeCompare(String(b), undefined, { numeric: true, sensitivity: "base" });
      });

      const order = uniqueDeclared.concat(remaining);
      return order.map((name) => ({ name, slots: map.get(name) || [] })).filter((ft) => ft.slots.length);
    },
  },
  methods: {
    /* rank helpers */
    rankLabel(raw) {
      const t = String(raw || "").trim();
      if (!t) return "";
      const map = {
        pvt:"PVT", pfc:"PFC", spc:"SPC", spc2:"SPC2", spc3:"SPC3", spc4:"SPC4",
        lcpl:"LCPL", cpl:"CPL", sgt:"SGT", ssgt:"SSGT", gysgt:"GYSGT",
        "2ndlt":"2NDLT", "1stlt":"1STLT", capt:"CAPT", maj:"MAJ",
        ha:"HA", hn:"HN", hm3:"HM3", hm2:"HM2", hm1:"HM1", hmc:"HMC",
        wo:"WO", cwo2:"CWO2", cwo3:"CWO3", cwo4:"CWO4", cwo5:"CWO5",
      };
      const k = t.toLowerCase().replace(/[.\s]/g,"");
      return map[k] || t.toUpperCase();
    },
    rankFor(id) {
      const p = this.personnel.find(pp => String(pp.id) === String(id));
      return p?.rank || "";
    },
    rankIcon(rank) {
      const r = this.rankLabel(rank);
      if (!r) return "";
      const base = (this.rankIconBase || "/ranks").replace(/\/+$/,"");
      const ext = (this.rankIconExt || "png").replace(/^\./,"");
      return `${base}/${r}.${ext}`;
    },
    onRankImgError(ev) {
      try { ev.target.style.display = "none"; } catch {}
    },

    triggerFlicker(delayMs = 0) {
      this.animateView = false;
      this.animationDelay = `${delayMs}ms`;
      this.$nextTick(() => {
        requestAnimationFrame(() => { this.animateView = true; });
      });
    },

    /* local persistence */
    persistPlan() {
      try {
        const ls = typeof localStorage !== "undefined" ? localStorage : null;
        if (!ls) return;
        ls.setItem(this.STORAGE_KEY, JSON.stringify(this.plan || { units: [] }));
      } catch {}
    },
    loadLocal() {
      try {
        const ls = typeof localStorage !== "undefined" ? localStorage : null;
        if (!ls) return;
        const raw = ls.getItem(this.STORAGE_KEY);
        if (!raw) return;
        const data = JSON.parse(raw);
        if (data && Array.isArray(data.units)) this.plan = data;
      } catch {}
    },
    clearAllLocal() {
      try {
        const ls = typeof localStorage !== "undefined" ? localStorage : null;
        if (ls) ls.removeItem(this.STORAGE_KEY);
      } catch {}
      this.plan = { units: this.buildUnitsFromOrbat(this.orbat) };
      this.persistPlan();
      this.triggerFlicker(0);
    },

    /* device id for anonymous writes */
    getDeviceId() {
      try {
        const ls = typeof localStorage !== "undefined" ? localStorage : null;
        if (!ls) return "";
        const k = "deploymentDeviceId";
        let v = ls.getItem(k);
        if (v) return v;
        v = (crypto?.randomUUID ? crypto.randomUUID() : String(Math.random()).slice(2) + Date.now());
        ls.setItem(k, v);
        return v;
      } catch { return ""; }
    },

    /* UI */
    openUnit(key) { this.detailKey = key; },
    openPicker(unitKey, slotIdx) {
      if (!unitKey) return;
      this.picker.open = true;
      this.picker.unitKey = unitKey;
      this.picker.slotIdx = Number(slotIdx);
      this.picker.query = "";
      this.picker.onlyFree = false;
      this.picker.sort = "name_asc";
    },
    closePicker() {
      this.picker.open = false;
      this.picker.unitKey = "";
      this.picker.slotIdx = -1;
      this.picker.query = "";
      this.picker.onlyFree = false;
      this.picker.sort = "name_asc";
    },
    clearCurrentSlot() { if (!this.picker.open) return; this.clearSlot(this.picker.unitKey, this.picker.slotIdx); },

    /* overview */
    openOverview() { this.overview.open = true; this.refreshOverview(); },
    closeOverview() { this.overview.open = false; },
    async refreshOverview() {
      this.overview.loading = true;
      this.overview.error = "";
      try {
        for (const u of this.chalkUnits) {
          await this.fetchRemoteMeta(u.key);
        }
      } catch (e) {
        this.overview.error = String(e?.message || e);
      } finally {
        this.overview.loading = false;
      }
    },

    /* points */
    slotsPoints(slots) {
      let pts = 0;
      (slots || []).forEach(s => {
        if (!s?.id) return;
        const c = String(s.cert || "").trim();
        if (c && this.CERT_POINTS[c] !== undefined) pts += this.CERT_POINTS[c];
        if (s.disposable) pts += this.DISPOSABLE_COST;
      });
      return pts;
    },
    calcUnitPoints(unit) {
      if (!unit) return 0;
      return this.slotsPoints(unit.slots || []);
    },

    /* data helpers */
    keyFromName(name) { return String(name || "").trim().toLowerCase().replace(/\s+/g, "-"); },
    filledCount(g) { if(!g) return 0; return (g.slots || []).reduce((n, s) => n + (s.id ? 1 : 0), 0); },
    isChalk(title) { return /\bchalk\s*\d+\b/i.test(String(title || "")); },
    padSlots(arr, min) {
      const out = (arr || []).slice();
      while (out.length < min) out.push({ id:null, name:null, role:"", origStatus:"VACANT", cert:"", disposable:false, fireteam:"" });
      return out;
    },

    extractRank(member) {
      if (!member) return "";
      const r = member.rank || member.Rank || member.grade || member.Grade || "";
      return String(r || "").trim();
    },
    extractCertsFromMember(member) {
      if (!member) return [];
      const raw = member.certs || member.certifications || member.Certs || member.Certifications || [];
      if (Array.isArray(raw)) return raw.map(x => String(x).trim()).filter(Boolean);
      if (typeof raw === "string") return raw.split(/[,;|]/).map(x => x.trim()).filter(Boolean);
      return [];
    },

    getCertsForPersonId(personId) {
      const p = this.personnel.find(pp => String(pp.id) === String(personId));
      return p?.certs || [];
    },
    hasCertId(personId, idx) {
      const p = this.personnel.find(pp => String(pp.id) === String(personId));
      if (!p) return false;
      const label = this.certLabels[idx];
      return (p.certs || []).includes(label);
    },

    certPointSuffix(label) {
      const pts = this.CERT_POINTS[label] ?? 0;
      return pts ? ` (+${pts})` : "";
    },

    ensureSlotCert(slot, fallbackRole = "") {
      if (slot.cert) return slot.cert;
      const certs = this.getCertsForPersonId(slot.id) || [];
      return certs[0] || this.titleCase(String(fallbackRole || slot.role || ""));
    },
    titleCase(s) {
      const t = String(s || "").replace(/[_-]+/g, " ").trim();
      if (!t) return "";
      return t.replace(/\s+/g," ").toLowerCase().replace(/\b\w/g,m=>m.toUpperCase());
    },

    toFireteamsArray(fireteams) {
      if (Array.isArray(fireteams)) return fireteams;
      if (fireteams && typeof fireteams === "object") return Object.values(fireteams);
      return [];
    },

    normalizeFireteamName(raw) {
      const t = String(raw ?? "").trim();
      if (!t) return "Element";

      const lower = t.toLowerCase();

      if (lower === "element" || lower === "hq" || lower === "headquarters" || lower === "command") return "Element";
      if (lower === "1" || /^(?:ft|fire\s*team|fireteam)\s*[-#:]?\s*1$/.test(lower)) return "Fireteam 1";
      if (lower === "2" || /^(?:ft|fire\s*team|fireteam)\s*[-#:]?\s*2$/.test(lower)) return "Fireteam 2";
      if (lower === "3" || /^(?:ft|fire\s*team|fireteam)\s*[-#:]?\s*3$/.test(lower)) return "Fireteam 3";
      if (lower === "4" || /^(?:ft|fire\s*team|fireteam)\s*[-#:]?\s*4$/.test(lower)) return "Fireteam 4";

      const m = lower.match(/^(?:ft|fire\s*team|fireteam)\s*[-#:]?\s*(\d+)$/);
      if (m) return `Fireteam ${m[1]}`;

      if (lower.startsWith("fireteam")) {
        const rest = t.replace(/fireteam/i, "").trim();
        return rest ? `Fireteam ${rest}` : "Fireteam";
      }

      return t;
    },

    fireteamOrderKey(name) {
      const t = String(name ?? "").trim().toLowerCase();
      if (t === "fireteam 1") return 0;
      if (t === "fireteam 2") return 1;
      if (t === "fireteam 3") return 2;
      if (t === "fireteam 4") return 3;
      if (t === "element") return 90;
      return 50;
    },

    fireteamLabel(slot) {
      const raw = slot?.fireteam ?? slot?.fireTeam ?? slot?.element ?? slot?.team ?? slot?.group ?? "";
      return this.normalizeFireteamName(raw);
    },

    buildPersonnelPool(orbat) {
      const pool = [];
      (orbat || []).forEach(sq => {
        this.toFireteamsArray(sq.fireteams).forEach(ft => {
          if (!ft || typeof ft !== "object") return;
          (ft.slots || []).forEach((s, idx) => {
            if (s?.member) {
              const id = String(s.member.id ?? `${sq.squad}-${ft.name}-${idx}`);
              const certs = this.extractCertsFromMember(s.member);
              const rank = this.extractRank(s.member);
              const element = String(ft?.name || ft?.element || "").trim();
              pool.push({
                id,
                name: String(s.member.name || "Unknown"),
                callsign: String(s.member.callsign || ""),
                role: String(s.role || s.member.slot || ""),
                certs,
                rank,
                element
              });
            }
          });
        });
      });
      const seen = new Set();
      const out = [];
      for (const p of pool) if (!seen.has(p.id)) { seen.add(p.id); out.push(p); }
      return out;
    },

    buildUnitsFromOrbat(orbat) {
      const units = [];
      (orbat || []).forEach(sq => {
        const key = this.keyFromName(sq.squad);
        const fireteams = this.toFireteamsArray(sq.fireteams)
          .map((ft) => {
            if (typeof ft === "string") return ft;
            if (ft && typeof ft === "object") return String(ft.name || ft.element || ft.team || "").trim();
            return "";
          })
          .filter(Boolean);
        const slots = [];
        this.toFireteamsArray(sq.fireteams).forEach(ft => {
          if (!ft || typeof ft !== "object") return;
          (ft.slots || []).forEach(s => {
            const status = String(s?.status || (s?.member ? "FILLED" : "VACANT")).toUpperCase();
            const origStatus = ["VACANT", "CLOSED"].includes(status) ? status : "FILLED";
            const member = s?.member || null;
            const slot = {
              id: member?.id ? String(member.id) : null,
              name: member?.name || null,
              role: s?.role || member?.slot || "",
              fireteam: String(ft.name || ft.element || "").trim(),
              origStatus,
              cert: "",
              disposable: false
            };
            if (slot.id) slot.cert = this.ensureSlotCert(slot, slot.role);
            slots.push(slot);
          });
        });

        let finalSlots = this.sortSlotsByFireteam(slots, fireteams);
        if (this.isChalk(sq.squad)) finalSlots = this.padSlots(finalSlots, this.MIN_CHALK_SLOTS);

        units.push({ key, title: sq.squad, slots: finalSlots, fireteams });
      });

      // If ORBAT missing or empty, keep current plan units
      if (!units.length && this.plan?.units?.length) return this.plan.units;

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
      // If explicit fireteams exist, attempt to keep slots' fireteam positions (best effort).
      // Current implementation keeps original order (ORBAT already provides intended order).
      return (slots || []).slice();
    },

    /* assignments */
    findAssignment(personId) {
      const id = String(personId);
      for (const u of (this.plan.units || [])) {
        for (let i = 0; i < (u.slots || []).length; i += 1) {
          const s = u.slots[i];
          if (s?.id && String(s.id) === id) return { unitKey: u.key, slotIdx: i };
        }
      }
      return null;
    },

    formatAssignment(assign) {
      if (!assign) return "";
      const u = this.plan.units.find(x => x.key === assign.unitKey);
      return u ? `${u.title} #${assign.slotIdx+1}` : `${assign.unitKey}#${assign.slotIdx+1}`;
    },

    openOverviewLegacy() { this.openOverview(); },

    // slot operations
    addSlot(unitKey) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];
      const slots = (g.slots || []).slice();
      slots.push({ id:null, name:null, role:"", origStatus:"VACANT", cert:"", disposable:false, fireteam:"" });
      const next = { ...g, slots };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? next : u));
      this.persistPlan();
    },
    removeSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];
      const slots = (g.slots || []).slice();
      if (slots.length <= this.MIN_CHALK_SLOTS && this.isChalk(g.title)) return;
      slots.splice(slotIdx, 1);
      const next = { ...g, slots };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? next : u));
      this.persistPlan();
    },

    clearSlot(unitKey, slotIdx) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;

      const g = this.plan.units[idx];
      const slots = g.slots.slice();
      const prev = slots[slotIdx];
      if (!prev) return;

      const cleared = { ...prev, id: null, name: null, cert: "", disposable: false, origStatus: "VACANT" };
      slots.splice(slotIdx, 1);
      slots.push(cleared);

      const newG = { ...g, slots };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? newG : u));
      this.persistPlan();
      this.detailError = "";
    },

    clearGroup(unitKey) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;

      const g = this.plan.units[idx];
      const emptied = (g.slots || []).map(s => ({
        ...s,
        id: null,
        name: null,
        cert: "",
        disposable: false,
        origStatus: s.origStatus === "CLOSED" ? "CLOSED" : "VACANT",
      }));

      const next = { ...g, slots: this.sortSlotsByRole(this.isChalk(g.title) ? this.padSlots(emptied, this.MIN_CHALK_SLOTS) : emptied) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? next : u));
      this.persistPlan();
    },

    onToggleDisposable(unitKey, slotIdx, checked) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];
      const slots = g.slots.slice();
      const s = slots[slotIdx];
      if (!s) return;
      slots.splice(slotIdx, 1, { ...s, disposable: !!checked });
      const next = { ...g, slots: this.sortSlotsByRole(slots) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? next : u));
      this.persistPlan();
    },

    onChangeCert(unitKey, slotIdx, cert) {
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const g = this.plan.units[idx];
      const slots = g.slots.slice();
      const s = slots[slotIdx];
      if (!s) return;
      slots.splice(slotIdx, 1, { ...s, cert: String(cert || "") });
      const next = { ...g, slots: this.sortSlotsByRole(slots) };
      this.plan.units = this.plan.units.map((u, i) => (i === idx ? next : u));
      this.persistPlan();
    },

    /* remote API */
    async apiPost(action, payload) {
      const url = this.apiBase;
      if (!url) return { ok:false, error:"Missing exec URL" };

      const body = {
        secret: this.secret,
        action,
        ...(payload || {}),
      };

      const headers = { "Content-Type": "application/json" };
      const tok = this.authToken;
      if (tok) headers.Authorization = `Bearer ${tok}`;
      if (!tok) body.deviceId = this.deviceId;

      const res = await fetch(url, {
        method: "POST",
        headers,
        body: JSON.stringify(body),
      });

      const json = await res.json().catch(() => null);
      return json || { ok:false, error:"Invalid response" };
    },

    async fetchRemoteMeta(unitKey) {
      if (!unitKey || !this.apiBase) return;
      try {
        const resp = await this.apiPost("config:get", { unitId: unitKey });
        const { ok, data } = resp || {};
        if (!ok || !data) return;
        this.remoteMeta = {
          ...this.remoteMeta,
          [unitKey]: {
            savedBy: data.savedBy || "",
            savedAt: data.savedAt || "",
            points: data.points,
          }
        };
      } catch {}
    },

    parseConfigJSONSlots(data) {
      try {
        const raw = data?.configJSON;
        if (!raw) return [];
        const cfg = typeof raw === "string" ? JSON.parse(raw) : raw;
        const units = cfg?.units || cfg?.plan?.units || [];
        if (Array.isArray(units) && units.length) {
          // try match unitKey
          const u = units.find(x => x.key === data.unitId) || units[0];
          if (u && Array.isArray(u.slots)) return u.slots;
        }
        if (cfg && Array.isArray(cfg.slots)) return cfg.slots;
      } catch {}
      return [];
    },

    extractSlotsFromAny(data) {
      const raw = data?.configJSON;
      try {
        const cfg = typeof raw === "string" ? JSON.parse(raw) : raw;
        if (cfg?.slots && Array.isArray(cfg.slots)) return cfg.slots;
        if (cfg?.units && Array.isArray(cfg.units) && cfg.units[0]?.slots) return cfg.units[0].slots;
      } catch {}
      return [];
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
        const idx = this.plan.units.findIndex(u => u.key === unitKey);
        if (idx === -1) return;
        const curr = this.plan.units[idx];

        // Remote configs store only slot assignments/loadouts; keep ORBAT slot metadata (e.g., fireteam) intact.
        const incoming = Array.isArray(nextSlots) ? nextSlots : [];
        const baseSlots = Array.isArray(curr.slots) ? curr.slots : [];

        const merged = baseSlots.map((base, i) => {
          const inc = incoming[i] || {};
          const id = inc.id ?? null;
          const name = inc.name ?? null;
          const role = (inc.role && String(inc.role).trim()) ? inc.role : base.role;
          const cert = inc.cert || "";
          const disposable = !!inc.disposable;
          const origStatus = base.origStatus === "CLOSED" ? "CLOSED" : (id ? "FILLED" : "VACANT");
          return { ...base, id, name, role, cert, disposable, origStatus };
        });

        // If remote has extra slots (e.g., template changed), append them to Element.
        if (incoming.length > baseSlots.length) {
          for (let i = baseSlots.length; i < incoming.length; i += 1) {
            const inc = incoming[i] || {};
            merged.push({
              id: inc.id ?? null,
              name: inc.name ?? null,
              role: inc.role || "",
              cert: inc.cert || "",
              disposable: !!inc.disposable,
              fireteam: inc.fireteam || "",
              origStatus: inc.id ? "FILLED" : "VACANT",
            });
          }
        }

        const padded = this.isChalk(curr.title) ? this.padSlots(merged, this.MIN_CHALK_SLOTS) : merged;
        const nextUnit = { ...curr, slots: this.sortSlotsByRole(padded) };
        this.plan.units = this.plan.units.map((u, i) => (i === idx ? nextUnit : u));
        this.versions = { ...this.versions, [unitKey]: Number(data.version || 0) };
        await this.fetchRemoteMeta(unitKey);
        this.triggerFlicker(0);
      } catch (e) {
        this.apiError = String(e && e.message || e);
      } finally {
        this.busy = false;
      }
    },

    async saveRemote(unitKey) {
      if (!unitKey) return;
      const idx = this.plan.units.findIndex(u => u.key === unitKey);
      if (idx < 0) return;
      const u = this.plan.units[idx];

      this.apiError = "";
      this.busy = true;

      try {
        const payload = {
          unitId: unitKey,
          version: Number(this.versions[unitKey] || 0),
          configJSON: JSON.stringify({
            unitId: unitKey,
            slots: (u.slots || []).map(s => ({
              id: s.id || null,
              name: s.name || null,
              role: s.role || "",
              cert: s.cert || "",
              disposable: !!s.disposable,
            })),
          }),
          points: this.slotsPoints(u.slots || []),
        };

        const resp = await this.apiPost("config:save", payload);
        const { ok, error, data } = resp || {};
        if (!ok) throw new Error(error || "Save failed");
        this.versions = { ...this.versions, [unitKey]: Number(data?.version || payload.version || 0) };
        await this.fetchRemoteMeta(unitKey);
        this.triggerFlicker(0);
      } catch (e) {
        this.apiError = String(e && e.message || e);
      } finally {
        this.busy = false;
      }
    },

    async clearAllSaved() {
      if (!this.apiBase) return;
      this.overview.loading = true;
      this.overview.error = "";
      try {
        const resp = await this.apiPost("config:clearAll", {});
        if (!resp?.ok) throw new Error(resp?.error || "Clear all failed");
        this.remoteMeta = {};
        this.versions = {};
      } catch (e) {
        this.overview.error = String(e?.message || e);
      } finally {
        this.overview.loading = false;
      }
    },

    ovItem(unitKey) {
      const m = this.remoteMeta?.[unitKey];
      if (!m) return null;
      return {
        savedBy: m.savedBy || "",
        savedAt: m.savedAt || "",
        points: m.points,
        slots: m.slots,
      };
    },

    formatDate(iso) {
      try {
        const d = new Date(iso);
        if (Number.isNaN(d.getTime())) return String(iso || "");
        return d.toLocaleString();
      } catch { return String(iso || ""); }
    },

    selectPersonnel(p) {
      if (!p || !this.picker.open) return;
      const unitKey = this.picker.unitKey;
      const slotIdx = this.picker.slotIdx;

      const gIdx = this.plan.units.findIndex(u => u.key === unitKey);
      if (gIdx < 0) return;
      const g = this.plan.units[gIdx];
      const target = g.slots[slotIdx];
      if (!target) return;

      const existing = this.findAssignment(p.id);
      const chosenCertDefault = this.ensureSlotCert({ id: p.id, role: target.role }, target.role || p.role || "");

      if (existing && (existing.unitKey !== unitKey || existing.slotIdx !== slotIdx)) {
        const srcIdx = this.plan.units.findIndex(u => u.key === existing.unitKey);
        if (srcIdx < 0) return;
        const srcGroup = this.plan.units[srcIdx];
        const srcSlot = srcGroup.slots[existing.slotIdx];
        if (!srcSlot) return;

        const newSrcSlots = srcGroup.slots.slice();
        newSrcSlots[existing.slotIdx] = { ...srcSlot, id: null, name: null, cert: "", disposable: false, origStatus: "VACANT" };
        const newSrc = { ...srcGroup, slots: this.sortSlotsByRole(newSrcSlots) };

        const newGSlots = g.slots.slice();
        newGSlots[slotIdx] = { ...target, id: p.id, name: p.name, role: target.role || p.role || "", cert: chosenCertDefault, disposable: false, origStatus: "FILLED" };
        const newG = { ...g, slots: this.sortSlotsByRole(newGSlots) };

        this.plan.units = this.plan.units.map((u, i) => (i === gIdx ? newG : i === srcIdx ? newSrc : u));
      } else {
        const newSlots = g.slots.slice();
        newSlots[slotIdx] = { ...target, id: p.id, name: p.name, role: target.role || p.role || "", cert: chosenCertDefault, disposable: false, origStatus: "FILLED" };
        const newG = { ...g, slots: this.sortSlotsByRole(newSlots) };
        this.plan.units = this.plan.units.map((u, i) => (i === gIdx ? newG : u));
      }

      this.persistPlan();
      this.closePicker();
    },

    /* ---------- CSV/members loader helpers included for compatibility ---------- */
    csvToRows(text) {
      const out = [];
      const lines = String(text || "").split(/\r?\n/).filter(Boolean);
      if (!lines.length) return out;
      const parseLine = (line) => {
        const cells = [];
        let cur = ""; let inQ = false;
        for (let i = 0; i < line.length; i += 1) {
          const ch = line[i];
          if (ch === '"' && (i === 0 || line[i - 1] !== '\\')) {
            inQ = !inQ;
          } else if (ch === ',' && !inQ) {
            cells.push(cur); cur = "";
          } else {
            cur += ch;
          }
        }
        cells.push(cur);
        return cells.map(c => c.replace(/^"|"$/g,"").trim());
      };
      const header = parseLine(lines[0]);
      for (let i = 1; i < lines.length; i += 1) {
        const row = parseLine(lines[i]);
        const obj = {};
        header.forEach((h, idx) => obj[h] = row[idx] ?? "");
        out.push(obj);
      }
      return out;
    },

    loadMembersCSV(text) {
      try {
        const rows = this.csvToRows(text || "");
        return rows;
      } catch { return []; }
    },
  }
};
</script>

<style scoped>
.deployment-shell{max-width:1200px;margin:0 auto;padding:1rem 0}
.topbar{display:flex;justify-content:space-between;align-items:flex-start;gap:1rem;margin-bottom:1rem}
.topbar .title{font-size:1.2rem;font-weight:800;letter-spacing:.08em;color:#bfe3ff}
.topbar .subtitle{font-size:.9rem;opacity:.85}
.planner-grid{display:grid;grid-template-columns:320px 1fr;gap:1rem}
.panel{background:rgba(0,0,0,.3);border:1px solid rgba(255,255,255,.08);border-radius:14px;overflow:hidden}
.panel-header{padding:.9rem 1rem;border-bottom:1px solid rgba(255,255,255,.08)}
.panel-header .h{font-weight:800;letter-spacing:.08em}
.panel-body{padding:1rem}
.panel-footer{padding:.8rem 1rem;border-top:1px solid rgba(255,255,255,.08)}
.unit-list{padding:.5rem}
.unit-item{padding:.7rem .8rem;border-radius:12px;cursor:pointer;border:1px solid transparent}
.unit-item:hover{background:rgba(255,255,255,.05)}
.unit-item.active{border-color:rgba(191,227,255,.35);background:rgba(191,227,255,.08)}
.unit-title{font-weight:800}
.unit-meta{display:flex;align-items:center;gap:.5rem;font-size:.85rem;opacity:.85}
.divider{width:1px;height:12px;background:rgba(255,255,255,.2);display:inline-block}
.warn{background:rgba(255,120,120,.12);border:1px solid rgba(255,120,120,.22);padding:.6rem .8rem;border-radius:12px;margin:.6rem 0}
.toolbar{display:flex;justify-content:space-between;align-items:center;gap:1rem;margin-bottom:.6rem}
.stats-row{display:flex;align-items:center;gap:.6rem;margin:.3rem 0 .9rem 0}
.pts.big{font-weight:800}
.pts.big.over{color:#ffb7b7}
.group-card{border:1px solid rgba(255,255,255,.08);border-radius:14px;overflow:hidden}
.squad-modal-meta{display:flex;justify-content:space-between;align-items:center;padding:1rem;border-bottom:1px solid rgba(255,255,255,.08)}
.squad-modal-meta.invalid{background:rgba(255,120,120,.08)}
.squad-tag{width:56px;height:56px;border-radius:14px;display:flex;align-items:center;justify-content:center;background:rgba(191,227,255,.08);border:1px solid rgba(191,227,255,.2)}
.squad-tag span{font-weight:900}
.squad-members-grid{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:.8rem;padding:1rem}
@media (max-width: 900px){.planner-grid{grid-template-columns:1fr}.squad-members-grid{grid-template-columns:1fr}}
.member-card{border:1px solid rgba(255,255,255,.08);border-radius:14px;overflow:hidden;background:rgba(0,0,0,.22)}
.member-card.vacant{opacity:.9}
.member-card.closed{opacity:.55}
.member-header{display:flex;gap:.7rem;align-items:flex-start;padding:.8rem;border-bottom:1px solid rgba(255,255,255,.08)}
.member-header-text h3{margin:0;font-weight:900;letter-spacing:.06em}
.rank-line{margin:.2rem 0 0 0;opacity:.85;font-size:.85rem;display:flex;gap:.6rem;flex-wrap:wrap}
.rank-icon{width:26px;height:26px;object-fit:contain;margin-right:.35rem}
.rank-icon.small{width:20px;height:20px}
.name-line{display:flex;align-items:center;gap:.35rem}
.member-body{display:flex;gap:1rem;padding:.8rem}
.member-column{flex:1}
.detail-line{margin:.35rem 0}
.role-accent{font-weight:800}
.cert-list{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:.35rem .6rem;margin-top:.4rem}
.cert-row{display:flex;align-items:center;gap:.4rem;font-size:.85rem;opacity:.92}
.cert-checkbox{width:14px;height:14px;border-radius:4px;border:1px solid rgba(255,255,255,.25);display:inline-flex;align-items:center;justify-content:center}
.cert-checkbox.checked{border-color:rgba(191,227,255,.65)}
.checkbox-dot{width:8px;height:8px;border-radius:3px;background:rgba(191,227,255,.75)}
.member-footer{display:flex;align-items:center;justify-content:space-between;gap:.6rem;padding:.7rem .8rem;border-top:1px solid rgba(255,255,255,.08);font-size:.8rem;opacity:.9}
.cert-none{opacity:.75}
.loadout-row{margin:.6rem 0 0 0}
.primary-label{display:block;font-size:.8rem;opacity:.85;margin-bottom:.25rem}
.loadout-select{width:100%;padding:.45rem .5rem;border-radius:10px;border:1px solid rgba(255,255,255,.12);background:rgba(0,0,0,.25);color:#e8f6ff}
.btn{border-radius:12px;border:1px solid rgba(255,255,255,.12);background:rgba(255,255,255,.08);color:#e8f6ff;padding:.5rem .8rem;cursor:pointer}
.btn:hover{background:rgba(255,255,255,.12)}
.btn.primary{background:rgba(191,227,255,.18);border-color:rgba(191,227,255,.3)}
.btn.ghost{background:transparent}
.btn.small{padding:.35rem .6rem;border-radius:10px;font-size:.85rem}
.btn.xsmall{padding:.25rem .45rem;border-radius:10px;font-size:.78rem}
.picker-overlay,.squad-overlay{position:fixed;inset:0;background:rgba(0,0,0,.6);display:flex;align-items:center;justify-content:center;padding:1rem;z-index:999}
.picker-modal,.squad-modal{width:min(960px,100%);max-height:90vh;overflow:hidden;background:rgba(10,12,16,.95);border:1px solid rgba(255,255,255,.12);border-radius:16px}
.picker-header{display:flex;justify-content:space-between;align-items:center;padding:1rem;border-bottom:1px solid rgba(255,255,255,.08)}
.picker-title{font-weight:900;letter-spacing:.06em}
.picker-close,.squad-close{background:transparent;border:0;color:#e8f6ff;font-size:1.2rem;cursor:pointer}
.picker-controls{display:flex;gap:.6rem;align-items:center;flex-wrap:wrap;padding:.8rem 1rem;border-bottom:1px solid rgba(255,255,255,.08)}
.picker-search{flex:1;min-width:220px;padding:.5rem .6rem;border-radius:10px;border:1px solid rgba(255,255,255,.12);background:rgba(0,0,0,.25);color:#e8f6ff}
.picker-check{font-size:.85rem;opacity:.9;display:flex;align-items:center;gap:.35rem}
.picker-sort{padding:.5rem .6rem;border-radius:10px;border:1px solid rgba(255,255,255,.12);background:rgba(0,0,0,.25);color:#e8f6ff}
.picker-list{padding:1rem;max-height:55vh;overflow:auto}
.pick-row{display:flex;gap:.8rem;align-items:center;padding:.6rem .7rem;border-radius:12px;border:1px solid rgba(255,255,255,.08);margin-bottom:.55rem}
.pick-row.assigned{border-color:rgba(191,227,255,.28);background:rgba(191,227,255,.06)}
.pick-row .row-left{display:flex;align-items:center;gap:.5rem;min-width:220px}
.pick-row .row-mid{flex:1;display:flex;flex-direction:column;gap:.2rem}
.pick-row .row-right{display:flex;justify-content:flex-end}
.chip{display:inline-block;padding:.2rem .45rem;border-radius:999px;background:rgba(255,255,255,.08);font-size:.78rem}
.chip.muted{opacity:.8}
.picker-footer{display:flex;justify-content:space-between;align-items:center;padding:.8rem 1rem;border-top:1px solid rgba(255,255,255,.08)}
.overview-grid.two-by-two{display:grid;grid-template-columns:repeat(2,minmax(0,1fr));gap:1rem;padding:1rem}
@media (max-width: 900px){.overview-grid.two-by-two{grid-template-columns:1fr}}
.overview-card{border:1px solid rgba(255,255,255,.08);border-radius:14px;overflow:hidden}
.overview-head{display:flex;justify-content:space-between;align-items:flex-start;padding:.8rem;border-bottom:1px solid rgba(255,255,255,.08)}
.overview-body{padding:.8rem}
.slots-compact{display:flex;flex-direction:column;gap:.45rem}
.slot-row{display:flex;justify-content:space-between;gap:.8rem;padding:.45rem .55rem;border-radius:12px;border:1px solid rgba(255,255,255,.06);background:rgba(0,0,0,.18)}
.slot-left{display:flex;flex-direction:column;gap:.15rem}
.slot-name{font-weight:800}
.slot-id{font-size:.78rem;opacity:.8}
.slot-right{display:flex;align-items:center;gap:.5rem;font-size:.85rem;opacity:.9}
.overview-empty{opacity:.7}
.compact-header{display:flex;justify-content:space-between;align-items:center;padding:1rem;border-bottom:1px solid rgba(255,255,255,.08)}
.muted{opacity:.85}
.small{font-size:.85rem}
.actions-row{display:flex;justify-content:space-between;align-items:center;padding:1rem;border-top:1px solid rgba(255,255,255,.08)}
/* fireteam header row inside the grid */
.fireteam-row{grid-column:1 / -1;display:flex;align-items:center;justify-content:space-between;gap:.6rem;padding:.45rem .65rem;border-radius:12px;border:1px solid rgba(191,227,255,.18);background:rgba(191,227,255,.06)}
.fireteam-title{font-weight:900;letter-spacing:.08em}
.fireteam-count{opacity:.85;font-size:.85rem}
.fireteam-divider{grid-column:1 / -1;height:1px;background:rgba(255,255,255,.08);margin:.2rem 0}
.section-header{display:flex;gap:.6rem;align-items:center}
.section-header img{width:22px;height:22px}
.section-header h1{margin:0;font-weight:900;letter-spacing:.08em;font-size:.9rem;color:#bfe3ff}
</style>
