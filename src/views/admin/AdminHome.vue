<!-- src/views/admin/AdminHome.vue -->
<template>
  <div
    class="windows-grid content-container"
    :class="{ animate: animateView }"
    :style="{ 'animation-delay': animationDelay }"
  >
    <!-- LEFT -->
    <section class="section-container left-window">
      <div class="header-shell">
        <div class="section-header simple-admin-plate admin-plate--clipped">
          <img src="/icons/protocol.svg" alt="" />
          <h1>ADMIN</h1>
        </div>
        <div class="admin-plate-connector" aria-hidden="true"></div>
      </div>

      <div class="section-content-container">
        <div v-if="isAuthed" class="rail">
          <button
            class="rail-card"
            :class="{ active: activeKey === 'promotions' }"
            @click="activeKey = 'promotions'"
          >
            <div class="rail-card-head">
              <img src="/icons/protocol.svg" class="rail-icon" alt="" />
              <div class="rail-title">Promotions Overview</div>
            </div>
            <div class="rail-card-body">
              <div class="rail-line">
                <span class="label">Eligible now</span>
                <span class="pill ok">{{ eligibleNowCount }}</span>
              </div>
              <div class="rail-line">
                <span class="label">Imminent (≤3)</span>
                <span class="pill warn">{{ imminentCount }}</span>
              </div>
              <div class="rail-foot">Click to open</div>
            </div>
          </button>
        </div>
        <div v-else class="muted">Staff only.</div>
      </div>
    </section>

    <!-- RIGHT -->
    <section class="section-container right-window">
      <div class="header-shell">
        <div class="section-header clipped-medium-backward-pilot right-header">
          <img src="/icons/protocol.svg" alt="" />
          <h1>PROMOTIONS OVERVIEW</h1>
        </div>
        <div class="rhombus-back">&nbsp;</div>
      </div>

      <div class="section-content-container right-content">
        <div v-if="!isAuthed" class="muted">Staff only.</div>

        <div v-else class="promotions-panel">
          <div class="filters">
            <div class="row">
              <label class="control">
                <span>Search</span>
                <input type="text" v-model="search" placeholder="Name, rank, squad" />
              </label>

              <label class="control">
                <span>Squad</span>
                <select v-model="selectedSquad">
                  <option value="__ALL__">All squads</option>
                  <option v-for="s in squads" :key="s" :value="s">{{ s }}</option>
                </select>
              </label>

              <label class="control">
                <span>Sort by</span>
                <select v-model="sortKey">
                  <option value="rank">Rank (high→low)</option>
                  <option value="ops">Ops attended</option>
                  <option value="progress">Progress to next rank</option>
                  <option value="name">Name (A→Z)</option>
                </select>
              </label>

              <label class="control chk">
                <input type="checkbox" v-model="onlyPromotable" />
                <span>Promotable only</span>
              </label>
            </div>
          </div>

          <div class="chips">
            <span class="chip">Total: {{ promotionsTable.length }}</span>
            <span class="chip ok">Eligible now: {{ eligibleNowCount }}</span>
            <span class="chip warn">Imminent (≤3): {{ imminentCount }}</span>
          </div>

          <div class="table-shell">
            <div class="tr head grid6">
              <span class="th name">Name</span>
              <span class="th rank">Rank</span>
              <span class="th squad">Squad</span>
              <span class="th ops">Ops</span>
              <span class="th next">Next Rank</span>
              <span class="th prog">Progress</span>
            </div>

            <div class="rows-scroll">
              <div v-if="promotionsTable.length === 0" class="empty">
                No matching results.
              </div>

              <div
                v-for="(row, i) in promotionsTable"
                :key="row.id || row.name + i"
                class="tr grid6"
              >
                <span class="td name">{{ row.name }}</span>
                <span class="td rank">{{ row.rank || "—" }}</span>
                <span class="td squad">{{ row.squad || "—" }}</span>
                <span class="td ops">
                  <span v-if="isFiniteNum(row.ops)">{{ row.ops }}</span>
                  <span v-else class="muted">N/A</span>
                </span>
                <span class="td next">
                  <span v-if="row.nextRank">{{ row.nextRank }} <small v-if="row.nextAt">({{ row.nextAt }})</small></span>
                  <span v-else class="muted">—</span>
                </span>
                <span class="td prog">
                  <div class="bar" :class="{ done: row.opsToNext === 0 && row.nextRank }">
                    <div class="fill" :style="{ width: (row.progress ?? 0) + '%' }"></div>
                  </div>
                </span>
              </div>
            </div>
          </div>

          <div class="muted footnote">
            Source: Members CSV for roster; Troop Status CSV to exclude Discharged.
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { isAdmin } from "@/utils/adminAuth";

export default {
  name: "AdminHome",
  props: {
    members: { type: Array, default: () => [] },
    orbat: { type: Array, default: () => [] },
    reserves: { type: Array, default: () => [] },
    attendance: { type: Array, default: () => [] },
  },
  data() {
    return {
      animateView: false,
      animationDelay: "0ms",
      activeKey: "promotions",

      // Same sheet used elsewhere: Troop List + Troop Status
      troopStatusCsvUrl:
        "https://docs.google.com/spreadsheets/d/e/2PACX-1vRq9fpYoWY_heQNfXegQ52zvOIGk-FCMML3kw2cX3M3s8blNRSH6XSRUdtTo7UXaJDDkg4bGQcl3jRP/pub?gid=107253735&single=true&output=csv",
      csvStatusIndex: Object.create(null), // nameKey -> normalized status
      csvTroopIndex: Object.create(null), // nameKey -> present in troop list

      search: "",
      selectedSquad: "__ALL__",
      sortKey: "rank",
      onlyPromotable: false,
    };
  },
  computed: {
    isAuthed() {
      return isAdmin();
    },

    nameKey() {
      return (name) =>
        String(name || "")
          .replace(/["'.]/g, "")
          .replace(/\s+/g, " ")
          .trim()
          .toUpperCase();
    },
    cleanMemberName() {
      return (name) => String(name || "").replace(/\s*[\(\[].*?[\)\]]\s*$/g, "").trim();
    },

    normalizeStatus() {
      const pretty = {
        ACTIVE: "Active",
        RESERVE: "Reserve",
        ELOA: "ELOA",
        OTHER: "Other",
        INACTIVE: "Inactive",
        DISCHARGED: "Discharged",
        UNKNOWN: "Unknown",
      };
      return (raw) => pretty[String(raw || "").trim().toUpperCase()] || "Unknown";
    },
    memberStatusOf() {
      return (m) => {
        const nk = this.nameKey(this.cleanMemberName(m?.name));
        return this.csvStatusIndex[nk] || "Unknown";
      };
    },
    isDischarged() {
      return (status) => String(status || "").trim().toLowerCase() === "discharged";
    },
    isInTroopList() {
      return (m) => {
        const nk = this.nameKey(this.cleanMemberName(m?.name));
        const hasCsv = Object.keys(this.csvTroopIndex).length > 0;
        return hasCsv ? !!this.csvTroopIndex[nk] : true;
      };
    },
    filteredMembers() {
      return (this.members || []).filter((m) => {
        const inList = this.isInTroopList(m);
        const st = this.memberStatusOf(m);
        return inList && !this.isDischarged(st);
      });
    },

    squads() {
      const set = new Set();
      for (const m of this.filteredMembers) {
        const squad = String(m?.squad || "").trim();
        if (squad) set.add(squad);
      }
      return Array.from(set).sort((a, b) => a.localeCompare(b, undefined, { sensitivity: "base" }));
    },

    promotionsTable() {
      const term = (this.search || "").trim().toLowerCase();
      const squadFilter = this.selectedSquad;
      const onlyProm = !!this.onlyPromotable;
      const rows = [];

      for (const m of this.filteredMembers) {
        const name = String(m?.name || "").trim();
        const rank = String(m?.rank || "").trim();
        const squad = String(m?.squad || "").trim();

        if (term) {
          const hay = [name, rank, squad].join(" ").toLowerCase();
          if (!hay.includes(term)) continue;
        }
        if (squadFilter !== "__ALL__" && squad !== squadFilter) continue;

        const rule = this.promotionLadderFor(rank);
        const ops = this.getOps(m);

        const nextRank = rule?.nextRank ?? null;
        const nextAt = rule?.nextAt ?? null;

        let progress = 0;
        let opsToNext = null;
        if (Number.isFinite(nextAt) && Number.isFinite(ops)) {
          progress = Math.min(100, Math.max(0, Math.round((ops / nextAt) * 100)));
          opsToNext = Math.max(0, nextAt - ops);
        }

        if (onlyProm && !(opsToNext === 0 && !!nextRank)) continue;

        rows.push({
          id: m.id,
          name,
          rank,
          squad,
          ops,
          nextRank,
          nextAt,
          progress,
          opsToNext,
        });
      }

      const rankVal = (r) => {
        const order = {
          GEN: 1,
          COL: 2,
          MAJ: 3,
          CPT: 4,
          "1LT": 5,
          "2LT": 6,
          CWO4: 7,
          CWO3: 8,
          CWO2: 9,
          WO: 10,
          GYSGT: 11,
          SSGT: 12,
          SGT: 13,
          CPL: 14,
          LCPL: 15,
          PFC: 16,
          PVT: 17,
        };
        const key = String(r || "").trim().toUpperCase().replace(/[.\s]/g, "");
        return order[key] || 999;
      };

      if (this.sortKey === "name") {
        rows.sort((a, b) => String(a.name).localeCompare(String(b.name), undefined, { sensitivity: "base" }));
      } else if (this.sortKey === "ops") {
        rows.sort((a, b) => (Number(b.ops) || 0) - (Number(a.ops) || 0));
      } else if (this.sortKey === "progress") {
        rows.sort((a, b) => (Number(b.progress) || 0) - (Number(a.progress) || 0));
      } else {
        rows.sort((a, b) => rankVal(a.rank) - rankVal(b.rank));
      }

      return rows;
    },

    eligibleNowCount() {
      return this.promotionsTable.filter((r) => r.opsToNext === 0 && !!r.nextRank).length;
    },
    imminentCount() {
      return this.promotionsTable.filter((r) => Number.isFinite(r.opsToNext) && r.opsToNext > 0 && r.opsToNext <= 3).length;
    },
  },
  created() {
    this.fetchTroopStatusCsv();
  },
  mounted() {
    this.animateView = true;
  },
  methods: {
    isFiniteNum(v) {
      return Number.isFinite(v);
    },

    promotionLadderFor(rank) {
      const r = String(rank || "").trim().toUpperCase().replace(/[.\s]/g, "");
      const ladder = {
        PVT: { nextAt: 3, nextRank: "PFC" },
        PFC: { nextAt: 6, nextRank: "LCPL" },
        LCPL: { nextAt: 10, nextRank: "CPL" },
        CPL: { nextAt: 15, nextRank: "SGT" },
        SGT: { nextAt: 22, nextRank: "SSGT" },
        SSGT: { nextAt: 30, nextRank: "GYSGT" },
      };
      return ladder[r] || null;
    },

    getOps(member) {
      const id = member?.id;
      const byId = (this.attendance || []).find((x) => String(x.id) === String(id));
      if (byId && Number.isFinite(Number(byId.ops))) return Number(byId.ops);

      const nk = this.nameKey(member?.name);
      const byName = (this.attendance || []).find((x) => this.nameKey(x.name) === nk);
      if (byName && Number.isFinite(Number(byName.ops))) return Number(byName.ops);

      const fallback = Number(member?.opsAttended);
      return Number.isFinite(fallback) ? fallback : null;
    },

    async fetchTroopStatusCsv() {
      try {
        const res = await fetch(this.troopStatusCsvUrl, { method: "GET" });
        const csvText = await res.text();
        const rows = this.parseCsv(csvText);
        if (!rows.length) return;

        const header = rows[0].map((h) => String(h || "").trim());
        const hdrLower = header.map((h) => h.toLowerCase().replace(/\s+/g, " ").trim());
        const nameIdx = hdrLower.findIndex((h) => h === "troop list");
        const statusIdx = hdrLower.findIndex((h) => h === "troop status");
        if (nameIdx === -1 || statusIdx === -1) return;

        const statusMap = Object.create(null);
        const troopMap = Object.create(null);
        for (let i = 1; i < rows.length; i++) {
          const r = rows[i];
          const rawName = String(r[nameIdx] || "").trim();
          if (!rawName) continue;
          const nk = this.nameKey(this.cleanMemberName(rawName));
          troopMap[nk] = true;
          statusMap[nk] = this.normalizeStatus(String(r[statusIdx] || "").trim());
        }
        this.csvStatusIndex = statusMap;
        this.csvTroopIndex = troopMap;
      } catch (e) {
        console.warn("CSV status load failed:", e);
      }
    },

    parseCsv(text) {
      const rows = [];
      let cur = [];
      let val = "";
      let inQ = false;
      for (let i = 0; i < text.length; i++) {
        const ch = text[i];
        if (inQ) {
          if (ch === '"') {
            if (text[i + 1] === '"') {
              val += '"';
              i += 1;
            } else {
              inQ = false;
            }
          } else {
            val += ch;
          }
        } else {
          if (ch === '"') inQ = true;
          else if (ch === ",") {
            cur.push(val);
            val = "";
          } else if (ch === "\n") {
            cur.push(val);
            rows.push(cur);
            cur = [];
            val = "";
          } else if (ch === "\r") {
            // ignore
          } else {
            val += ch;
          }
        }
      }
      cur.push(val);
      rows.push(cur);
      if (rows.length && rows[rows.length - 1].every((x) => String(x).length === 0)) rows.pop();
      return rows;
    },
  },
};
</script>

<style scoped>
.header-shell { height: 52px; overflow: hidden; }

/* Layout */
.windows-grid {
  display: grid;
  grid-template-columns: 380px minmax(1080px, 1fr);
  column-gap: 2.4rem;
  align-items: start;
  width: 100%;
}

.left-window { height: auto !important; }
.right-window { display: flex; flex-direction: column; max-height: 100vh; }

.right-content {
  flex: 1 1 auto;
  min-height: 0;
  overflow: hidden;
  padding: 0.6rem;
}

/* Left plate */
.simple-admin-plate {
  position: relative;
  z-index: 2;
  display: inline-flex;
  align-items: center;
  gap: 0.6rem;
  padding: 0 14px;
  height: 52px;
  width: 315px;
  background: #214d45;
}
.simple-admin-plate img { width: 20px; height: 20px; }
.simple-admin-plate h1 {
  margin: 0;
  line-height: 52px;
  font-weight: 700;
  letter-spacing: 0.18em;
  color: #0a0a0a;
  white-space: nowrap;
}
.admin-plate--clipped {
  clip-path: polygon(0 0, calc(100% - 22px) 0, 100% 26px, calc(100% - 22px) 52px, 0 52px);
}
.admin-plate-connector {
  display: inline-block;
  width: 14px;
  height: 14px;
  background: #214d45;
  transform: translateX(4px) translateY(19px) rotate(45deg);
}

.muted { color: rgba(158, 197, 230, 0.95); }
.empty { color: rgba(158, 197, 230, 0.95); padding: 0.8rem; text-align: center; }
.footnote { font-size: 12px; margin-top: 0.55rem; }

.promotions-panel {
  display: flex;
  flex-direction: column;
  gap: 0.6rem;
  height: 100%;
  min-height: 0;
}

.filters { border: 1px dashed rgba(30,144,255,0.35); border-radius: .35rem; padding: .5rem; }
.filters .row { display: grid; grid-template-columns: 1.2fr auto auto auto; gap: .6rem; align-items: end; }

.control { display: grid; gap: .2rem; }
.control span { font-size: .85rem; color: rgba(158, 197, 230, 0.95); }
.control input, .control select {
  background: rgba(0,10,30,0.35);
  border: 1px solid rgba(30,144,255,0.35);
  border-radius: .35rem;
  padding: .35rem .45rem;
  color: rgba(230, 243, 255, 0.95);
}
.control input::placeholder { color: rgba(170,199,230,0.95); }
.control input:focus, .control select:focus { outline: none; border-color: rgba(30,144,255,0.6); }
.control select option { background: rgba(5,20,40,0.98); color: rgba(230, 243, 255, 0.95); }
.control.chk { display: flex; align-items: center; gap: .45rem; padding-top: 1.25rem; }
.control.chk input[type="checkbox"] { width: 16px; height: 16px; accent-color: #78ffd0; }
.control.chk span { color: rgba(230, 243, 255, 0.95); font-size: .9rem; }

.chips { display: flex; gap: .45rem; flex-wrap: wrap; }
.chip { padding: .25rem .5rem; border-radius: 999px; background: rgba(0,10,30,0.25); border: 1px solid rgba(30,144,255,0.45); color: rgba(230, 243, 255, 0.95); }
.chip.ok { border-color: rgba(120,255,170,0.7); }
.chip.warn { border-color: rgba(255,190,80,0.7); }

/* Table: restore scroll */
.table-shell {
  flex: 1 1 auto;
  min-height: 0;
  border: 1px dashed rgba(30,144,255,0.35);
  border-radius: .35rem;
  background: rgba(0,10,30,0.18);
  display: flex;
  flex-direction: column;
  overflow-x: auto; /* horizontal if needed */
  overflow-y: hidden;
}

.grid6 {
  display: grid;
  grid-template-columns: 1.6fr .8fr 1fr .6fr .9fr 1.2fr;
  align-items: center;
  min-width: 980px; /* prevents clipping */
}

.tr.head {
  font-weight: 600;
  background: rgba(0,10,30,0.35);
  border-bottom: 1px dashed rgba(30,144,255,0.25);
  flex: 0 0 auto;
  position: sticky;
  top: 0;
  z-index: 2;
}

.rows-scroll {
  flex: 1 1 auto;
  min-height: 0;
  overflow: auto; /* vertical scroll restored */
}

.tr .th, .tr .td {
  padding: .4rem .5rem;
  color: rgba(230, 243, 255, 0.95);
  border-bottom: 1px dashed rgba(30,144,255,0.18);
}

.bar {
  height: 8px;
  background: rgba(0,10,30,0.35);
  border: 1px solid rgba(30,144,255,0.25);
  border-radius: 999px;
  position: relative;
  overflow: hidden;
}
.bar .fill {
  position: absolute;
  left: 0; top: 0; bottom: 0;
  width: 0%;
  transition: width .25s ease;
  background: rgba(120,200,255,0.6);
}
.bar.done .fill { background: rgba(120,255,170,0.7); }

/* Left rail */
.rail { display: grid; gap: .6rem; align-content: start; }
.rail-card {
  text-align: left;
  border: 1px solid rgba(30,144,255,0.35);
  background: rgba(0,10,30,0.35);
  border-radius: .5rem;
  padding: .6rem;
  cursor: pointer;
}
.rail-card.active { border-color: rgba(120,255,170,0.7); }
.rail-card-head { display: flex; align-items: center; gap: .5rem; margin-bottom: .35rem; }
.rail-icon { width: 20px; height: 20px; }
.rail-title, .rail-line .label { color: rgba(217, 235, 255, 0.95); }
.pill {
  font-size: .85rem;
  border: 1px solid rgba(30,144,255,0.45);
  border-radius: 999px;
  padding: .05rem .5rem;
  color: rgba(230, 243, 255, 0.95);
}
.rail-foot { margin-top: .25rem; font-size: .8rem; color: rgba(158, 197, 230, 0.95); }
</style>
