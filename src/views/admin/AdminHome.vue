<!-- src/views/admin/AdminHome.vue -->
<template>
  <div class="admin-home windows-grid">
    <!-- LEFT: nav / summary -->
    <section class="section-container left-window">
      <div class="header-shell">
        <div class="section-header simple-admin-plate admin-plate--clipped">
        <span class="window-icons" aria-hidden="true">
          <span class="proto-shell"><img src="/icons/protocol.svg" alt=""></span>
        </span>
        <h1>ADMIN</h1>
        </div>
        <div class="admin-plate-connector" aria-hidden="true"></div>
      </div>

      <div class="section-content-container left-content">
        <div v-if="!isAuthed" class="muted pad">Staff only.</div>

        <div v-else class="rail">
          <button class="rail-card active" type="button">
            <div class="rail-card-head">
<div class="rail-title">Promotions Overview</div>
            </div>

            <div class="rail-card-body">
              <div class="rail-line">
                <span class="label">Roster</span>
                <span class="pill">{{ filteredMembers.length }}</span>
              </div>
              <div class="rail-line">
                <span class="label">Eligible now</span>
                <span class="pill ok">{{ eligibleNowCount }}</span>
              </div>
              <div class="rail-line">
                <span class="label">Imminent (≤3)</span>
                <span class="pill warn">{{ imminentCount }}</span>
              </div>
              <div class="rail-foot">Discharged excluded via Troop Status sheet.</div>
            </div>
          </button>

          <div class="rail-hint muted">
            Data sources:<br />
            <span class="mono">Members CSV</span> (roster) + <span class="mono">RefData CSV</span> (Troop Status)
          </div>
        </div>
      </div>
    </section>

    <!-- RIGHT: Promotions table -->
    <section class="section-container right-window">
      <div class="header-shell">
        <div class="section-header clipped-medium-backward-pilot right-header">
        <span class="window-icons" aria-hidden="true">
          <span class="proto-shell"><img src="/icons/protocol.svg" alt=""></span>
        </span>

<h1>PROMOTIONS OVERVIEW</h1>
        </div>
        <div class="rhombus-back">&nbsp;</div>
      </div>

      <div class="section-content-container right-content">
        <div v-if="!isAuthed" class="muted pad">Staff only.</div>

        <div v-else class="promotions-panel">
          <div class="filters">
            <div class="row">
              <label class="control">
                <span>Search</span>
                <input v-model="search" type="text" placeholder="Name, rank, squad" />
              </label>

              <label class="control">
                <span>Squad</span>
                <select v-model="selectedSquad">
                  <option value="__ALL__">All squads</option>
                  <option v-for="s in squads" :key="s" :value="s">{{ s }}</option>
                </select>
              </label>

              <label class="control">
                <span>Sort</span>
                <select v-model="sortKey">
                  <option value="remaining">Ops remaining</option>
                  <option value="progress">Progress</option>
                  <option value="ops">Ops attended</option>
                  <option value="rank">Rank</option>
                  <option value="name">Name</option>
                </select>
              </label>

              <label class="control chk">
                <input v-model="onlyPromotable" type="checkbox" />
                <span>Eligible only</span>
              </label>
            </div>
          </div>

          <div class="chips">
            <span class="chip">Total: {{ promotionsTable.length }}</span>
            <span class="chip ok">Eligible: {{ eligibleNowCount }}</span>
            <span class="chip warn">≤3 ops: {{ imminentCount }}</span>
          </div>

          <div class="table-shell">
            <div class="tr head grid8">
              <span class="th name">Name</span>
              <span class="th rank">Rank</span>
              <span class="th status">Status</span>
              <span class="th squad">Squad</span>
              <span class="th ops">Ops</span>
              <span class="th next">Next</span>
              <span class="th need">Req</span>
              <span class="th rem">Rem</span>
            </div>

            <div class="rows-scroll">
              <div v-if="promotionsTable.length === 0" class="empty">
                No matching results.
              </div>

              <div
                v-for="(row, i) in promotionsTable"
                :key="row.id || row.name + i"
                class="tr grid8"
                :class="{ eligible: row.opsToNext === 0 && row.nextRank }"
              >
                <span class="td name">{{ row.name }}</span>
                <span class="td rank">{{ row.rank || "—" }}</span>
                <span class="td status"><span class="status-pill" :class="statusClass(row.status)">{{ row.status }}</span></span>
                <span class="td squad">{{ row.squad || "—" }}</span>
                <span class="td ops">
                  <span v-if="isFiniteNum(row.ops)">{{ row.ops }}</span>
                  <span v-else class="muted">N/A</span>
                </span>
                <span class="td next">
                  <span v-if="row.nextRank">{{ row.nextRank }}</span>
                  <span v-else class="muted">—</span>
                </span>
                <span class="td need">
                  <span v-if="isFiniteNum(row.nextAt)">{{ row.nextAt }}</span>
                  <span v-else class="muted">—</span>
                </span>
                <span class="td rem">
                  <span v-if="isFiniteNum(row.opsToNext)">{{ row.opsToNext }}</span>
                  <span v-else class="muted">—</span>
                </span>

                <div class="td prog span-all">
                  <div class="bar" :class="{ done: row.opsToNext === 0 && row.nextRank }">
                    <div class="fill" :style="{ width: (row.progress ?? 0) + '%' }"></div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="muted footnote">
            Notes: “Status” comes from RefData (Troop List / Troop Status). Discharged filtered out.
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
      search: "",
      selectedSquad: "__ALL__",
      sortKey: "remaining",
      onlyPromotable: false,

      // Same method as your deployment/status filtering: Troop List + Troop Status
      troopStatusCsvUrl:
        "https://docs.google.com/spreadsheets/d/e/2PACX-1vRq9fpYoWY_heQNfXegQ52zvOIGk-FCMML3kw2cX3M3s8blNRSH6XSRUdtTo7UXaJDDkg4bGQcl3jRP/pub?gid=107253735&single=true&output=csv",
      csvStatusIndex: Object.create(null),
      csvTroopIndex: Object.create(null),
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
        const status = this.memberStatusOf(m);
        return inList && !this.isDischarged(status);
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
      const rows = [];

      for (const m of this.filteredMembers) {
        const name = String(m?.name || "").trim();
        const rank = String(m?.rank || "").trim();
        const squad = String(m?.squad || "").trim();
        const status = this.memberStatusOf(m);

        if (term) {
          const hay = [name, rank, squad, status].join(" ").toLowerCase();
          if (!hay.includes(term)) continue;
        }
        if (squadFilter !== "__ALL__" && squad !== squadFilter) continue;

        const ladder = this.promotionLadderFor(rank);
        const ops = this.getOps(m);
        const nextRank = ladder?.nextRank ?? null;
        const nextAt = ladder?.nextAt ?? null;

        let progress = 0;
        let opsToNext = null;
        if (Number.isFinite(nextAt) && Number.isFinite(ops)) {
          progress = Math.min(100, Math.max(0, Math.round((ops / nextAt) * 100)));
          opsToNext = Math.max(0, nextAt - ops);
        }

        rows.push({
          id: m?.id,
          name,
          rank,
          squad,
          status,
          ops,
          nextRank,
          nextAt,
          progress,
          opsToNext,
        });
      }

      const safeNum = (v, fallback = 999999) => (Number.isFinite(Number(v)) ? Number(v) : fallback);
      const rankVal = (r) => {
        const order = {
          GEN: 1, COL: 2, MAJ: 3, CPT: 4, "1LT": 5, "2LT": 6,
          CWO4: 7, CWO3: 8, CWO2: 9, WO: 10,
          GYSGT: 11, SSGT: 12, SGT: 13, CPL: 14, LCPL: 15, PFC: 16, SPC: 17, PVT: 18,
          HM2: 19, HM3: 20, HN: 21, HA: 22,
          SPC2: 23, SPC3: 24, SPC4: 25
        };
        const key = String(r || "").trim().toUpperCase().replace(/[.\s]/g, "");
        return order[key] || 999;
      };

      if (this.onlyPromotable) {
        // eligible means: has a next rank + opsToNext computed and == 0
        rows.splice(
          0,
          rows.length,
          ...rows.filter((r) => r.nextRank && Number.isFinite(r.opsToNext) && r.opsToNext === 0)
        );
      }

      if (this.sortKey === "name") {
        rows.sort((a, b) => String(a.name).localeCompare(String(b.name), undefined, { sensitivity: "base" }));
      } else if (this.sortKey === "ops") {
        rows.sort((a, b) => safeNum(b.ops, -1) - safeNum(a.ops, -1));
      } else if (this.sortKey === "progress") {
        rows.sort((a, b) => safeNum(b.progress, -1) - safeNum(a.progress, -1));
      } else if (this.sortKey === "rank") {
        rows.sort((a, b) => rankVal(a.rank) - rankVal(b.rank));
      } else {
        // remaining (default)
        rows.sort((a, b) => safeNum(a.opsToNext) - safeNum(b.opsToNext) || rankVal(a.rank) - rankVal(b.rank));
      }

      return rows;
    },

    eligibleNowCount() {
      return this.promotionsTable.filter((r) => r.nextRank && Number.isFinite(r.opsToNext) && r.opsToNext === 0).length;
    },
    imminentCount() {
      return this.promotionsTable.filter((r) => Number.isFinite(r.opsToNext) && r.opsToNext > 0 && r.opsToNext <= 3).length;
    },
  },
  created() {
    this.fetchTroopStatusCsv();
  },
  methods: {
    isFiniteNum(v) {
      return Number.isFinite(v);
    },

    statusClass(status) {
      const s = String(status || "").toLowerCase();
      if (s === "active") return "ok";
      if (s === "reserve") return "warn";
      if (s === "eloa") return "eloa";
      if (s === "inactive") return "inactive";
      if (s === "other") return "other";
      if (s === "unknown") return "unknown";
      return "unknown";
    },

    promotionLadderFor(rank) {
      const key = String(rank || "").trim().toUpperCase().replace(/[.\s]/g, "");

      // Required OP Days are TOTAL ops needed to ATTAIN the rank.
      // Example: RCT→PVT when ops >= 2; PVT→PFC when ops >= 10; PFC→SPC when ops >= 20; etc.
      const ladders = {
        // Enlisted rifle track
        RCT: { nextAt: 2, nextRank: "PVT" },
        REC: { nextAt: 2, nextRank: "PVT" },
        RECRUIT: { nextAt: 2, nextRank: "PVT" },

        PVT: { nextAt: 10, nextRank: "PFC" },
        PV2: { nextAt: 10, nextRank: "PFC" },
        PFC: { nextAt: 20, nextRank: "SPC" },
        SPC: { nextAt: 30, nextRank: "SPC2" },
        SPC1: { nextAt: 30, nextRank: "SPC2" },
        SPC2: { nextAt: 40, nextRank: "SPC3" },
        SPC3: { nextAt: 50, nextRank: "SPC4" },

        // Corpsman track
        HA: { nextAt: 10, nextRank: "HN" },
        HN: { nextAt: 20, nextRank: "HM3" },
        HM3: { nextAt: 30, nextRank: "HM2" },

        // Warrant track
        CWO1: { nextAt: 10, nextRank: "CWO2" },
        WO: { nextAt: 10, nextRank: "CWO2" },
        CWO2: { nextAt: 20, nextRank: "CWO3" },
        CWO3: { nextAt: 30, nextRank: "CWO4" },
      };

      return ladders[key] || null;
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
/* =========================
   UNSC TERMINAL THEME PASS
   Visual-only: NO template/script changes.
   ========================= */

/* Window shells */
.admin-home .section-container{
  border-radius: 16px;
  border: 1px solid rgba(170, 220, 255, 0.22);
  background: linear-gradient(180deg, rgba(8, 14, 20, 0.92), rgba(3, 6, 10, 0.95));
  box-shadow:
    0 0 0 1px rgba(170, 220, 255, 0.06) inset,
    0 0 26px rgba(120, 180, 255, 0.10),
    0 0 110px rgba(0, 0, 0, 0.6);
  overflow: hidden;
  position: relative;
}

/* Scanlines + glow per window */
.admin-home .section-container::before{
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
.admin-home .section-container::after{
  content:"";
  position:absolute;
  inset:-20%;
  pointer-events:none;
  background: radial-gradient(circle at 30% 20%, rgba(120, 180, 255, 0.07), transparent 55%);
  opacity: .9;
  animation: adminFlicker 2.9s infinite;
  z-index: 0;
}
@keyframes adminFlicker{
  0%, 100% { transform: translate3d(0,0,0); opacity: .72; }
  12% { transform: translate3d(-1px, 1px, 0); opacity: .86; }
  24% { transform: translate3d(1px, -1px, 0); opacity: .70; }
  40% { transform: translate3d(0px, 2px, 0); opacity: .90; }
  65% { transform: translate3d(2px, 0px, 0); opacity: .78; }
}

/* Keep header/content above effects */
.admin-home .header-shell,
.admin-home .section-header,
.admin-home .section-content-container{
  position: relative;
  z-index: 1;
}

/* Header chrome bars */
.admin-home .header-shell{
  border-bottom: 1px solid rgba(170, 220, 255, 0.12);
  background: rgba(0,0,0,0.16);
}

/* Make both headers use terminal chrome (remove the green plate visuals) */
.admin-home .simple-admin-plate{
  background: transparent !important;
  width: auto !important;
  padding: 0 14px !important;
}
.admin-home .admin-plate-connector{
  background: rgba(170, 220, 255, 0.18) !important;
}
.admin-home .section-header.admin-plate--clipped{
  clip-path: none !important;
}
.admin-home .section-header.clipped-medium-backward-pilot{
  clip-path: none !important;
  background: transparent !important;
}
.admin-home .rhombus-back{ opacity: .15; filter: saturate(.9); }

/* Add "window dots" to both headers */
.admin-home .section-header::before{ content: none !important; display:none !important; }

/* Header typography */
.admin-home .section-header{
  padding: 12px 14px !important;
  gap: 10px !important;
  height: 52px;
}
.admin-home .section-header img{
  width: 18px;
  height: 18px;
  opacity: .9;
  filter: drop-shadow(0 0 10px rgba(120,180,255,0.18));
}
.admin-home .section-header h1{
  margin: 0 !important;
  line-height: 1 !important;
  font-size: 18px !important;
  font-weight: 700 !important;
  letter-spacing: 0.12em !important;
  color: rgba(190, 230, 255, 0.92) !important;
  text-transform: uppercase;
  white-space: nowrap;
}

/* Header window icons (protocol.svg is black, so give it a visible shell) */
.admin-home .window-icons{
  display: inline-flex;
  gap: 6px;
  margin-right: 10px;
  flex: 0 0 auto;
  align-items: center;
}
.admin-home .proto-shell{
  width: 18px;
  height: 18px;
  border-radius: 6px;
  display: grid;
  place-items: center;
  background: rgba(170,220,255,0.18);
  border: 1px solid rgba(170,220,255,0.22);
  box-shadow: 0 0 0 1px rgba(0,0,0,0.18) inset, 0 0 16px rgba(120,180,255,0.10);
}
.admin-home .proto-shell img{
  width: 12px;
  height: 12px;
  opacity: .92;
  filter: drop-shadow(0 0 8px rgba(120,180,255,0.14));
}


/* Left rail card -> terminal tile */
.admin-home .rail-card{
  border: 1px solid rgba(170,220,255,0.18) !important;
  background: rgba(0,0,0,0.22) !important;
  border-radius: 14px !important;
  box-shadow: 0 0 0 1px rgba(170,220,255,0.05) inset, 0 0 22px rgba(0,0,0,0.28);
}
.admin-home .rail-card.active{
  border-color: rgba(170,220,255,0.55) !important;
}
.admin-home .rail-title{ letter-spacing: .10em !important; text-transform: uppercase; }
.admin-home .pill{
  background: rgba(0,0,0,0.18);
  border-color: rgba(170,220,255,0.28) !important;
}
.admin-home .pill.ok{ border-color: rgba(120,255,190,0.55) !important; }
.admin-home .pill.warn{ border-color: rgba(255,190,80,0.7) !important; }

/* Filters + table chrome */
.admin-home .filters,
.admin-home .table-shell{
  border-color: rgba(170,220,255,0.18) !important;
  background: rgba(0,0,0,0.18) !important;
  border-radius: 14px !important;
}
.admin-home .tr.head{
  background: rgba(0,0,0,0.22) !important;
  border-bottom-color: rgba(170,220,255,0.16) !important;
}

/* Inputs/selects */
.admin-home .control input,
.admin-home .control select{
  background: rgba(5, 12, 20, 0.85) !important;
  border-color: rgba(170,220,255,0.22) !important;
  border-radius: 10px !important;
  color: rgba(226, 243, 255, 0.95) !important;
  box-shadow: 0 0 0 1px rgba(170,220,255,0.04) inset;
}
.admin-home .control input:focus,
.admin-home .control select:focus{
  outline: none;
  border-color: rgba(120,180,255,0.55) !important;
  box-shadow: 0 0 0 2px rgba(120,180,255,0.22);
}
.admin-home .control.chk input[type="checkbox"]{ accent-color: #78ffd0; }

/* Chips */
.admin-home .chip{
  background: rgba(0,0,0,0.18) !important;
  border-color: rgba(170,220,255,0.22) !important;
  border-radius: 999px;
}
.admin-home .chip.ok{ border-color: rgba(120,255,190,0.55) !important; }
.admin-home .chip.warn{ border-color: rgba(255,190,80,0.7) !important; }

/* Progress bar */
.admin-home .bar{
  background: rgba(0,0,0,0.24) !important;
  border-color: rgba(170,220,255,0.16) !important;
}
.admin-home .bar .fill{ background: rgba(120,200,255,0.55) !important; }
.admin-home .bar.done .fill{ background: rgba(120,255,190,0.65) !important; }

/* Status pills */
.admin-home .status-pill{
  border-color: rgba(170,220,255,0.22) !important;
  background: rgba(0,0,0,0.18) !important;
}

/* Scrollbars */
.admin-home .rows-scroll::-webkit-scrollbar{ width: 10px; height: 10px; }
.admin-home .rows-scroll::-webkit-scrollbar-thumb{ background: rgba(90,140,180,0.55); border-radius: 999px; }
.admin-home .rows-scroll::-webkit-scrollbar-track{ background: rgba(0,0,0,0.22); }


/* Override global fixed widths from _base.css for admin screens */
.windows-grid {
  display: grid;
  grid-template-columns: 420px minmax(1100px, 1fr);
  column-gap: 2.4rem;
  align-items: start;
  width: 100%;
}

/* Override global section sizing for this view (base.css sets fixed 393px width) */
.windows-grid > .section-container {
  width: auto !important;
  max-width: none !important;
  margin: 24px 24px !important;
  height: calc(100vh - 190px) !important;
  display: flex;
  flex-direction: column;
}

.left-window { min-width: 420px; }
.right-window { min-width: 0; }

.header-shell { height: 52px; overflow: hidden; }
.left-window > .section-content-container,
.right-window > .section-content-container { flex: 1 1 auto; min-height: 0; }

.left-content, .right-content { padding: 0.6rem; }

.pad { padding: 1rem; }

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
.mono { font-family: "Inconsolata", monospace; }

.rail { display: grid; gap: 0.8rem; align-content: start; }
.rail-card {
  text-align: left;
  border: 1px solid rgba(30,144,255,0.35);
  background: rgba(0,10,30,0.35);
  border-radius: 0.6rem;
  padding: 0.75rem;
}
.rail-card-head { display: flex; align-items: center; gap: 0.5rem; margin-bottom: 0.35rem; }
.rail-icon { width: 20px; height: 20px; }
.rail-title { color: rgba(217, 235, 255, 0.95); font-weight: 700; letter-spacing: 0.06em; }
.rail-line { display: flex; justify-content: space-between; align-items: center; margin-top: 0.35rem; }
.rail-line .label { color: rgba(217, 235, 255, 0.85); }
.rail-foot { margin-top: 0.5rem; font-size: 12px; color: rgba(158, 197, 230, 0.95); }
.rail-hint { font-size: 12px; line-height: 1.5; }

.pill {
  font-size: 0.85rem;
  border: 1px solid rgba(30,144,255,0.45);
  border-radius: 999px;
  padding: 0.05rem 0.55rem;
  color: rgba(230, 243, 255, 0.95);
}
.pill.ok { border-color: rgba(120,255,170,0.7); }
.pill.warn { border-color: rgba(255,190,80,0.7); }

.promotions-panel { display: flex; flex-direction: column; gap: .6rem; flex: 1 1 auto; min-height: 0; overflow: hidden; }

.filters { border: 1px dashed rgba(30,144,255,0.35); border-radius: 0.45rem; padding: 0.65rem; }
.filters .row { display: grid; grid-template-columns: 1.25fr auto auto auto; gap: 0.7rem; align-items: end; }

.control { display: grid; gap: 0.2rem; }
.control span { font-size: 0.85rem; color: rgba(158, 197, 230, 0.95); }
.control input, .control select {
  background: rgba(0,10,30,0.35);
  border: 1px solid rgba(30,144,255,0.35);
  border-radius: 0.45rem;
  padding: 0.45rem 0.55rem;
  color: rgba(230, 243, 255, 0.95);
}
.control input::placeholder { color: rgba(170,199,230,0.95); }
.control input:focus, .control select:focus { outline: none; border-color: rgba(30,144,255,0.6); }
.control select option { background: rgba(5,20,40,0.98); color: rgba(230, 243, 255, 0.95); }
.control.chk { display: flex; align-items: center; gap: 0.5rem; padding-top: 1.25rem; }
.control.chk input[type="checkbox"] { width: 16px; height: 16px; accent-color: #78ffd0; }
.control.chk span { color: rgba(230, 243, 255, 0.95); font-size: 0.9rem; }

.chips { display: flex; gap: 0.45rem; flex-wrap: wrap; }
.chip { padding: 0.25rem 0.55rem; border-radius: 999px; background: rgba(0,10,30,0.25); border: 1px solid rgba(30,144,255,0.45); color: rgba(230, 243, 255, 0.95); }
.chip.ok { border-color: rgba(120,255,170,0.7); }
.chip.warn { border-color: rgba(255,190,80,0.7); }

.table-shell { flex: 1 1 auto; min-height: 0; border: 1px dashed rgba(30,144,255,0.35); border-radius: .35rem; background: rgba(0,10,30,0.18); display: flex; flex-direction: column; overflow-x: auto; overflow-y: hidden; }

.grid8 {
  display: grid;
  grid-template-columns: 1.7fr 0.8fr 0.9fr 1.2fr 0.6fr 0.8fr 0.5fr 0.5fr;
  align-items: center;
  min-width: 1120px;
  column-gap: 0.1rem;
}
.span-all { grid-column: 1 / -1; }

.tr.head {
  font-weight: 700;
  background: rgba(0,10,30,0.35);
  border-bottom: 1px dashed rgba(30,144,255,0.25);
  flex: 0 0 auto;
  position: sticky;
  top: 0;
  z-index: 2;
}

.rows-scroll { flex: 1 1 auto; min-height: 0; overflow: auto; }

.tr .th, .tr .td {
  padding: 0.45rem 0.55rem;
  color: rgba(230, 243, 255, 0.95);
  border-bottom: 1px dashed rgba(30,144,255,0.18);
}
.tr.eligible .td { background: rgba(120,255,170,0.06); }

.status-pill {
  display: inline-flex;
  align-items: center;
  padding: 0.12rem 0.45rem;
  border-radius: 999px;
  border: 1px solid rgba(30,144,255,0.35);
  background: rgba(0,10,30,0.22);
  font-size: 12px;
  letter-spacing: 0.05em;
}
.status-pill.ok { border-color: rgba(120,255,170,0.7); }
.status-pill.warn { border-color: rgba(255,190,80,0.7); }
.status-pill.eloa { border-color: rgba(170,120,255,0.7); }
.status-pill.inactive { border-color: rgba(255,90,90,0.55); }
.status-pill.other, .status-pill.unknown { border-color: rgba(30,144,255,0.35); }

.bar {
  height: 8px;
  margin: 0.25rem 0.55rem 0.6rem;
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

.empty { padding: 1rem; text-align: center; color: rgba(158, 197, 230, 0.95); }
.footnote { font-size: 12px; margin-top: 0.35rem; }
</style>
