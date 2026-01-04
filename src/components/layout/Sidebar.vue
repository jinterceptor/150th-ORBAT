<!-- src/components/layout/Sidebar.vue -->
<template>
  <div class="sidebar-page">
    <section class="sidebar-layout">
      <o-sidebar
        id="sidebar"
        position="static"
        :animate="animate"
        :mobile="mobile"
        :expand-on-hover="expandOnHover"
        :reduce="reduce"
        open
      >
        <router-link class="clipped-bottom-right" to="/status" @click.native="playBrowse">
          <img src="/icons/orbital.svg" />
          <span>Status</span>
        </router-link>

        <router-link class="clipped-bottom-right" to="/roster" @click.native="playBrowse">
          <img src="/icons/license.svg" />
          <span>Roster</span>
        </router-link>

        <!-- NEW: Deployment (Officer/Staff only) -->
        <router-link
          v-if="isOfficerOrStaff"
          class="clipped-bottom-right"
          to="/deployment"
          @click.native="playBrowse"
        >
          <img src="/icons/deployable.svg" />
          <span>Deployment</span>
        </router-link>

        <router-link
          v-if="isOfficerOrStaff"
          class="clipped-bottom-right"
          to="/admin"
          @click.native="playBrowse"
        >
          <img src="/icons/protocol.svg" />
          <span>Admin</span>
        </router-link>
      </o-sidebar>
    </section>
  </div>
</template>

<script>
import { useAdminAuth } from "@/composables/useAdminAuth";
let browseAudio;

export default {
  name: "Sidebar",
  props: { animate: { type: Boolean, required: true } },
  setup() {
    const { isOfficerOrStaff } = useAdminAuth();
    return { isOfficerOrStaff };
  },
  data() {
    return { expandOnHover: false, mobile: "reduced", reduce: false };
  },
  mounted() {
    browseAudio = new Audio("/sound/Orbat Main Menu Browse.ogg");
    browseAudio.volume = 0.6;
  },
  methods: {
    playBrowse() {
      if (!browseAudio) return;
      try { browseAudio.currentTime = 0; browseAudio.play().catch(() => {}); } catch {}
    },
  },
};
</script>

<style scoped>

/* Override Oruga default sidebar backgrounds (some themes apply green panels) */
#sidebar :deep(.o-sidebar__wrapper),
#sidebar :deep(.o-sidebar__content),
#sidebar :deep(.o-sidebar__container),
#sidebar :deep(.o-sidebar__items){
  background: transparent !important;
}

/* Paint the actual rail on the outermost Oruga node */
#sidebar{
  background: linear-gradient(180deg, rgba(8,14,20,0.90), rgba(3,6,10,0.94)) !important;
}


/* =========================
   UNSC TERMINAL SIDEBAR THEME
   Visual-only: no logic changes
   ========================= */

.sidebar-page{
  position: relative;
}

/* Ensure the sidebar area reads like a terminal rail */
.sidebar-layout{
  height: 100%;
}

/* Target Oruga sidebar container */
#sidebar{
  position: relative;

  /* Sidebar width (give labels room + big icons) */
  width: 320px;
  min-width: 320px;

  border-right: 1px solid rgba(170,220,255,0.14);
  background: linear-gradient(180deg, rgba(8,14,20,0.90), rgba(3,6,10,0.94));
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.06) inset,
    0 0 28px rgba(120,180,255,0.08);
  overflow: hidden;
}
/* Scanlines */
#sidebar::before{
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
#sidebar::after{
  content:"";
  position:absolute;
  inset:-30%;
  pointer-events:none;
  background: radial-gradient(circle at 30% 30%, rgba(120,180,255,0.06), transparent 55%);
  opacity: .85;
  z-index: 0;
}

/* Keep links above effects */
#sidebar :deep(a),
#sidebar :deep(.o-sidebar__content),
#sidebar :deep(.o-sidebar__items){
  position: relative;
  z-index: 1;
}


/* Ensure Oruga containers respect width */
#sidebar :deep(.o-sidebar__content),
#sidebar :deep(.o-sidebar__items){
  width: 100%;
}

/* Link styling */
#sidebar :deep(a){
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 14px;
  margin: 10px 12px;
  border-radius: 14px;
  min-height: 72px; /* matches icon size */
  min-width: 0;

  border: 1px solid rgba(170,220,255,0.14);
  background: rgba(0,0,0,0.18);
  color: rgba(190,230,255,0.92);
  text-decoration: none;

  letter-spacing: 0.12em;
  text-transform: uppercase;
  font-size: 12px;

  box-shadow: 0 0 0 1px rgba(170,220,255,0.04) inset;
  transition: border-color .12s ease, box-shadow .12s ease, transform .12s ease, background .12s ease;
}


#sidebar :deep(a:hover){
  border-color: rgba(170,220,255,0.55);
  box-shadow:
    0 0 0 1px rgba(170,220,255,0.08) inset,
    0 0 22px rgba(120,180,255,0.10);
  transform: translateY(-1px);
}

#sidebar :deep(a.router-link-active){
  border-color: rgba(120,255,190,0.48);
  box-shadow:
    0 0 0 1px rgba(120,255,190,0.10) inset,
    0 0 24px rgba(120,255,190,0.10);
}

/* Icons */
#sidebar :deep(a img){
  width: 56px !important;
  height: 56px !important;
  max-width: none !important;
  max-height: none !important;
  flex: 0 0 56px;
  opacity: .95;
  filter: drop-shadow(0 0 14px rgba(120,180,255,0.16));
}


/* Label */
#sidebar :deep(a span){
  min-width: 0;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: normal;
}


/* Keep your clipped class from affecting the new "terminal tile" shape */
#sidebar :deep(.clipped-bottom-right){
  clip-path: none !important;
}

/* Mobile reduced mode often collapses text; keep icons crisp */

/* Active indicator bar (Oruga) -> terminal cyan instead of green */
#sidebar :deep(.o-sidebar__item.is-active::before),
#sidebar :deep(.o-sidebar__item--active::before),
#sidebar :deep(.o-sidebar__item--active > a::before),
#sidebar :deep(.router-link-active::before){
  background: rgba(170,220,255,0.55) !important;
  box-shadow: 0 0 16px rgba(120,180,255,0.16) !important;
}

/* Larger icons (closer to original) */
#sidebar :deep(a img){
  width: 56px;
  height: 56px;
  opacity: .95;
  filter: drop-shadow(0 0 14px rgba(120,180,255,0.16));
}



/* Prevent long labels (e.g., Deployment) from clipping */
#sidebar :deep(a){
  min-width: 0;
  padding-right: 14px;
}
#sidebar :deep(a span){
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}


@media (max-width: 860px){
  #sidebar{ width: 260px; min-width: 260px; }
  #sidebar :deep(a){ margin: 8px 10px; padding: 10px 12px; min-height: 64px; }
  #sidebar :deep(a img){ width: 44px !important; height: 44px !important; flex: 0 0 44px; }
}


</style>
