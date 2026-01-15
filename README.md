# juansebm.github.io
SEGUIMIENTO OBC
[app.js](https://github.com/user-attachments/files/24655335/app.js)
/* CuentaRegresiva Pro - PWA local (sin servidor)
   - CRUD de eventos en localStorage
   - Conteo regresivo en vivo
   - Notificaciones web opcionales (requiere permiso y HTTPS si se publica)
*/

const $ = (id) => document.getElementById(id);

const STORAGE_KEY = "crp.events.v1";
const NOTIFIED_KEY = "crp.notified.v1"; // { "<eventId>|<minutes>": true }

const PRIORITIES = [
  { value: "alta", label: "Alta", rank: 3 },
  { value: "media", label: "Media", rank: 2 },
  { value: "baja", label: "Baja", rank: 1 },
];

const DEFAULT_ALERT_MINUTES = [60 * 24 * 30, 60 * 24 * 7, 60 * 48, 60 * 24]; // 30d,7d,48h,24h

function uid() {
  // id corto estable
  return "ev_" + Math.random().toString(36).slice(2, 10) + "_" + Date.now().toString(36);
}

function nowMs() {
  return Date.now();
}

function loadEvents() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    const items = raw ? JSON.parse(raw) : [];
    if (!Array.isArray(items)) return [];
    return items;
  } catch {
    return [];
  }
}

function saveEvents(events) {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(events));
}

function loadNotified() {
  try {
    const raw = localStorage.getItem(NOTIFIED_KEY);
    const obj = raw ? JSON.parse(raw) : {};
    return obj && typeof obj === "object" ? obj : {};
  } catch {
    return {};
  }
}

function saveNotified(obj) {
  localStorage.setItem(NOTIFIED_KEY, JSON.stringify(obj));
}

function parseEndMs(datetimeLocalValue) {
  // datetime-local devuelve "YYYY-MM-DDTHH:mm"
  // Esto se interpreta como hora local.
  const d = new Date(datetimeLocalValue);
  const t = d.getTime();
  return Number.isFinite(t) ? t : null;
}

function fmt2(n) {
  return String(n).padStart(2, "0");
}

function formatCountdown(ms) {
  const totalSeconds = Math.max(0, Math.floor(ms / 1000));
  const days = Math.floor(totalSeconds / 86400);
  const hours = Math.floor((totalSeconds % 86400) / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = totalSeconds % 60;
  return { days, hours, minutes, seconds, text: `${days}d ${fmt2(hours)}:${fmt2(minutes)}:${fmt2(seconds)}` };
}

function formatDateTime(ms) {
  const d = new Date(ms);
  // Formato amigable en español (sin depender de Intl options raras)
  const yyyy = d.getFullYear();
  const mm = fmt2(d.getMonth() + 1);
  const dd = fmt2(d.getDate());
  const hh = fmt2(d.getHours());
  const mi = fmt2(d.getMinutes());
  return `${yyyy}-${mm}-${dd} ${hh}:${mi}`;
}

function relativeBadge(endMs) {
  const diff = endMs - nowMs();
  const days = diff / (1000 * 60 * 60 * 24);
  if (diff <= 0) return { cls: "danger", label: "Vencido" };
  if (days <= 5) return { cls: "danger", label: "Crítico" };
  if (days <= 15) return { cls: "warn", label: "Próximo" };
  return { cls: "ok", label: "A tiempo" };
}

function priorityInfo(value) {
  return PRIORITIES.find(p => p.value === value) || PRIORITIES[1];
}

function toast(msg) {
  const el = $("toast");
  el.textContent = msg;
  el.style.display = "block";
  clearTimeout(toast._t);
  toast._t = setTimeout(() => { el.style.display = "none"; }, 2500);
}

// --- UI state
let events = loadEvents();
let editingId = null;

// --- Elements
const listEl = $("list");
const emptyEl = $("empty");

const modalOverlay = $("modal");
const form = $("form");
const modalTitle = $("modalTitle");
const modalHint = $("modalHint");

const inpId = $("editId");
const inpName = $("name");
const inpCategory = $("category");
const inpPriority = $("priority");
const inpEnd = $("end");
const inpNotes = $("notes");

const btnAdd = $("btnAdd");
const btnAddEmpty = $("btnAddEmpty");
const btnClose = $("btnClose");
const btnCancel = $("btnCancel");
const btnSave = $("btnSave");
const btnDelete = $("btnDelete");

const inpSearch = $("search");
const selSort = $("sort");
const selFilter = $("filter");

const alertChips = $("alertChips");
const btnEnableNotifications = $("btnEnableNotifications");

const tplItem = $("tplItem");

// --- Modal helpers
function openModal(mode, eventObj = null) {
  modalOverlay.style.display = "flex";
  document.body.style.overflow = "hidden";

  if (mode === "create") {
    editingId = null;
    inpId.value = "";
    modalTitle.textContent = "Nuevo evento";
    modalHint.innerHTML = `Tip: Puedes crear eventos de proyectos, entregables o cierres. <span class="kbd">Esc</span> para cerrar.`;
    btnDelete.style.display = "none";

    inpName.value = "";
    inpCategory.value = "Proyecto";
    inpPriority.value = "media";
    // por defecto: mañana 17:00
    const d = new Date();
    d.setDate(d.getDate() + 1);
    d.setHours(17, 0, 0, 0);
    inpEnd.value = toDatetimeLocal(d);
    inpNotes.value = "";

    renderAlertChips(DEFAULT_ALERT_MINUTES, DEFAULT_ALERT_MINUTES.slice());
    form.dataset.alerts = JSON.stringify(DEFAULT_ALERT_MINUTES.slice());
  } else if (mode === "edit" && eventObj) {
    editingId = eventObj.id;
    inpId.value = eventObj.id;
    modalTitle.textContent = "Editar evento";
    modalHint.textContent = "Edita y guarda. Las alertas se disparan solo una vez por umbral.";
    btnDelete.style.display = "inline-block";

    inpName.value = eventObj.name;
    inpCategory.value = eventObj.category;
    inpPriority.value = eventObj.priority;
    inpEnd.value = toDatetimeLocal(new Date(eventObj.endMs));
    inpNotes.value = eventObj.notes || "";

    const alerts = Array.isArray(eventObj.alertMinutes) && eventObj.alertMinutes.length
      ? eventObj.alertMinutes
      : DEFAULT_ALERT_MINUTES.slice();
    renderAlertChips(alerts, alerts.slice());
    form.dataset.alerts = JSON.stringify(alerts.slice());
  }

  setTimeout(() => inpName.focus(), 50);
}

function closeModal() {
  modalOverlay.style.display = "none";
  document.body.style.overflow = "";
}

function toDatetimeLocal(dateObj) {
  // YYYY-MM-DDTHH:mm
  const yyyy = dateObj.getFullYear();
  const mm = fmt2(dateObj.getMonth() + 1);
  const dd = fmt2(dateObj.getDate());
  const hh = fmt2(dateObj.getHours());
  const mi = fmt2(dateObj.getMinutes());
  return `${yyyy}-${mm}-${dd}T${hh}:${mi}`;
}

// --- Alerts UI
function minutesToLabel(m) {
  if (m % (60 * 24) === 0) {
    const d = m / (60 * 24);
    return `${d} día${d === 1 ? "" : "s"}`;
  }
  if (m % 60 === 0) {
    const h = m / 60;
    return `${h} h`;
  }
  return `${m} min`;
}

function renderAlertChips(all, selected) {
  alertChips.innerHTML = "";
  const sel = new Set(selected);
  all.sort((a, b) => b - a).forEach((m) => {
    const b = document.createElement("button");
    b.type = "button";
    b.className = "btn ghost";
    b.style.borderRadius = "999px";
    b.style.padding = "6px 10px";
    b.style.borderColor = sel.has(m) ? "rgba(56,189,248,.45)" : "var(--border)";
    b.style.color = sel.has(m) ? "var(--accent)" : "var(--muted)";
    b.textContent = minutesToLabel(m);
    b.addEventListener("click", () => {
      if (sel.has(m)) sel.delete(m); else sel.add(m);
      const next = Array.from(sel).sort((a, b) => b - a);
      form.dataset.alerts = JSON.stringify(next);
      renderAlertChips(all, next);
    });
    alertChips.appendChild(b);
  });

  // chip para agregar umbral personalizado
  const add = document.createElement("button");
  add.type = "button";
  add.className = "btn ghost";
  add.style.borderRadius = "999px";
  add.style.padding = "6px 10px";
  add.textContent = "+ umbral";
  add.addEventListener("click", () => {
    const v = prompt("Escribe el umbral en minutos (ej: 90) o en horas (ej: 3h) o días (ej: 2d):");
    if (!v) return;
    let m = null;
    const s = String(v).trim().toLowerCase();
    if (/^\d+$/.test(s)) m = parseInt(s, 10);
    else if (/^\d+\s*h$/.test(s)) m = parseInt(s, 10) * 60;
    else if (/^\d+\s*d$/.test(s)) m = parseInt(s, 10) * 60 * 24;
    if (!m || m <= 0) { toast("Umbral inválido."); return; }

    const cur = new Set(JSON.parse(form.dataset.alerts || "[]"));
    cur.add(m);

    const allSet = new Set([...all, m]);
    const allNext = Array.from(allSet);
    const selectedNext = Array.from(cur).sort((a, b) => b - a);

    form.dataset.alerts = JSON.stringify(selectedNext);
    renderAlertChips(allNext, selectedNext);
  });
  alertChips.appendChild(add);
}

// --- Render list
function getFilteredSorted() {
  const q = (inpSearch.value || "").trim().toLowerCase();
  const cat = selFilter.value;

  let items = events.slice();

  if (q) {
    items = items.filter(e =>
      (e.name || "").toLowerCase().includes(q) ||
      (e.category || "").toLowerCase().includes(q) ||
      (e.notes || "").toLowerCase().includes(q)
    );
  }

  if (cat !== "all") {
    items = items.filter(e => e.category === cat);
  }

  const sort = selSort.value;
  if (sort === "soon") {
    items.sort((a, b) => a.endMs - b.endMs);
  } else if (sort === "priority") {
    items.sort((a, b) => {
      const pa = priorityInfo(a.priority).rank;
      const pb = priorityInfo(b.priority).rank;
      if (pb !== pa) return pb - pa;
      return a.endMs - b.endMs;
    });
  } else if (sort === "name") {
    items.sort((a, b) => (a.name || "").localeCompare(b.name || "", "es"));
  }

  return items;
}

function render() {
  // construir filtro de categorías dinámico
  const cats = Array.from(new Set(events.map(e => e.category).filter(Boolean))).sort((a, b) => a.localeCompare(b, "es"));
  const current = selFilter.value;
  selFilter.innerHTML = `<option value="all">Todas</option>` + cats.map(c => `<option value="${escapeHtml(c)}">${escapeHtml(c)}</option>`).join("");
  if ([...cats, "all"].includes(current)) selFilter.value = current;

  const items = getFilteredSorted();
  listEl.innerHTML = "";

  if (!items.length) {
    emptyEl.style.display = "block";
    listEl.style.display = "none";
  } else {
    emptyEl.style.display = "none";
    listEl.style.display = "grid";
  }

  for (const e of items) {
    const node = tplItem.content.firstElementChild.cloneNode(true);

    node.querySelector("[data-title]").textContent = e.name;
    node.querySelector("[data-notes]").textContent = e.notes ? e.notes : "";
    node.querySelector("[data-notes]").style.display = e.notes ? "block" : "none";

    const badge = relativeBadge(e.endMs);
    const badgeEl = node.querySelector("[data-badge]");
    badgeEl.textContent = badge.label;
    badgeEl.classList.add(badge.cls);

    node.querySelector("[data-cat]").textContent = e.category;
    node.querySelector("[data-pri]").textContent = `Prioridad: ${capitalize(e.priority)}`;
    node.querySelector("[data-end]").textContent = `Cierra: ${formatDateTime(e.endMs)}`;

    // timer
    node.querySelector("[data-timer]").textContent = formatCountdown(e.endMs - nowMs()).text;

    // botones
    node.querySelector("[data-edit]").addEventListener("click", () => openModal("edit", e));
    node.querySelector("[data-dup]").addEventListener("click", () => duplicateEvent(e.id));
    node.querySelector("[data-del]").addEventListener("click", () => deleteEvent(e.id));

    listEl.appendChild(node);
  }
}

function escapeHtml(s) {
  return String(s).replace(/[&<>"']/g, (c) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    "\"": "&quot;",
    "'": "&#39;"
  }[c]));
}

function capitalize(s) {
  s = String(s || "");
  return s ? s[0].toUpperCase() + s.slice(1) : s;
}

// --- CRUD
function upsertEvent(data) {
  const idx = events.findIndex(e => e.id === data.id);
  if (idx >= 0) events[idx] = data;
  else events.unshift(data);
  saveEvents(events);
  render();
}

function deleteEvent(id) {
  const e = events.find(x => x.id === id);
  const ok = confirm(`¿Eliminar el evento "${e?.name || ""}"?`);
  if (!ok) return;
  events = events.filter(e => e.id !== id);
  saveEvents(events);
  render();
  toast("Evento eliminado.");
}

function duplicateEvent(id) {
  const e = events.find(x => x.id === id);
  if (!e) return;
  const copy = { ...e, id: uid(), name: `${e.name} (copia)` };
  events.unshift(copy);
  saveEvents(events);
  render();
  toast("Evento duplicado.");
}

// --- Notifications (best-effort)
async function ensureNotificationPermission() {
  if (!("Notification" in window)) {
    toast("Tu navegador no soporta notificaciones.");
    return false;
  }
  if (Notification.permission === "granted") return true;
  if (Notification.permission === "denied") {
    toast("Notificaciones bloqueadas (habilítalas en ajustes del navegador).");
    return false;
  }
  const res = await Notification.requestPermission();
  if (res === "granted") {
    toast("Notificaciones activadas.");
    return true;
  }
  toast("No se activaron las notificaciones.");
  return false;
}

function maybeNotify() {
  if (!("Notification" in window)) return;
  if (Notification.permission !== "granted") return;

  const notified = loadNotified();
  const t = nowMs();

  for (const e of events) {
    if (!e.endMs || e.endMs <= t) continue;
    const diffMin = Math.floor((e.endMs - t) / 60000);

    const thresholds = Array.isArray(e.alertMinutes) && e.alertMinutes.length ? e.alertMinutes : DEFAULT_ALERT_MINUTES;
    for (const th of thresholds) {
      if (diffMin <= th && diffMin >= th - 1) { // ventana de 1 minuto
        const key = `${e.id}|${th}`;
        if (notified[key]) continue;
        notified[key] = true;

        const pretty = minutesToLabel(th);
        const body = `Faltan ~${pretty} para: ${e.name}\nCierra: ${formatDateTime(e.endMs)}`;
        try {
          new Notification("⏳ Recordatorio", { body });
        } catch {
          // ignore
        }
      }
    }
  }

  saveNotified(notified);
}

// --- Live update timers
function tickTimers() {
  // actualiza timers visibles
  const nodes = listEl.querySelectorAll("[data-timer]");
  const items = getFilteredSorted();
  // ids in same order as rendered: update by walking cards
  const cards = listEl.children;
  for (let i = 0; i < cards.length; i++) {
    const endTextEl = cards[i].querySelector("[data-end]");
    const titleEl = cards[i].querySelector("[data-title]");
    // find matching event quickly by title+end? Better: store data-id on card
  }
  // Better: add data-id at render time and use lookup
  const lookup = new Map(events.map(e => [e.id, e]));
  for (const card of cards) {
    const id = card.getAttribute("data-id");
    if (!id) continue;
    const e = lookup.get(id);
    if (!e) continue;
    const timerEl = card.querySelector("[data-timer]");
    if (timerEl) timerEl.textContent = formatCountdown(e.endMs - nowMs()).text;

    const badge = relativeBadge(e.endMs);
    const badgeEl = card.querySelector("[data-badge]");
    if (badgeEl) {
      badgeEl.classList.remove("ok","warn","danger");
      badgeEl.classList.add(badge.cls);
      badgeEl.textContent = badge.label;
    }
  }
}

function patchRenderDataId() {
  // ensure cards have data-id for tickTimers
  const cards = listEl.children;
  const items = getFilteredSorted();
  for (let i = 0; i < cards.length; i++) {
    cards[i].setAttribute("data-id", items[i]?.id || "");
  }
}

// Hook render to set data-id
const _render = render;
render = function() {
  _render();
  patchRenderDataId();
};

// --- Service worker
async function registerSW() {
  if ("serviceWorker" in navigator) {
    try {
      await navigator.serviceWorker.register("./service-worker.js");
    } catch {
      // ignore
    }
  }
}

// --- Events
btnAdd.addEventListener("click", () => openModal("create"));
btnAddEmpty.addEventListener("click", () => openModal("create"));
btnClose.addEventListener("click", closeModal);
btnCancel.addEventListener("click", closeModal);

modalOverlay.addEventListener("click", (e) => {
  if (e.target === modalOverlay) closeModal();
});

document.addEventListener("keydown", (e) => {
  if (e.key === "Escape") closeModal();
  if ((e.ctrlKey || e.metaKey) && e.key.toLowerCase() === "k") {
    e.preventDefault();
    inpSearch.focus();
  }
});

inpSearch.addEventListener("input", () => render());
selSort.addEventListener("change", () => render());
selFilter.addEventListener("change", () => render());

btnEnableNotifications.addEventListener("click", async () => {
  await ensureNotificationPermission();
});

btnDelete.addEventListener("click", () => {
  if (editingId) deleteEvent(editingId);
  closeModal();
});

form.addEventListener("submit", (e) => {
  e.preventDefault();

  const name = (inpName.value || "").trim();
  const category = (inpCategory.value || "").trim() || "General";
  const priority = inpPriority.value || "media";
  const endMs = parseEndMs(inpEnd.value);

  if (!name) { toast("Escribe un nombre."); inpName.focus(); return; }
  if (!endMs) { toast("Selecciona fecha y hora de finalización."); inpEnd.focus(); return; }

  const alerts = (() => {
    try {
      const v = JSON.parse(form.dataset.alerts || "[]");
      return Array.isArray(v) ? v.filter(n => Number.isFinite(n) && n > 0) : [];
    } catch {
      return [];
    }
  })();

  const eventObj = {
    id: editingId || uid(),
    name,
    category,
    priority,
    endMs,
    notes: (inpNotes.value || "").trim(),
    alertMinutes: alerts.length ? alerts : DEFAULT_ALERT_MINUTES.slice(),
    createdAt: editingId ? (events.find(x => x.id === editingId)?.createdAt || nowMs()) : nowMs(),
    updatedAt: nowMs(),
  };

  upsertEvent(eventObj);
  closeModal();
  toast(editingId ? "Evento actualizado." : "Evento creado.");
});

// --- Seed (si está vacío)
function seedIfEmpty() {
  if (events.length) return;
  const d1 = new Date(); d1.setDate(d1.getDate() + 10); d1.setHours(17,0,0,0);
  const d2 = new Date(); d2.setDate(d2.getDate() + 3); d2.setHours(12,0,0,0);
  events = [
    { id: uid(), name: "Cierre de Tramo / Legalización", category:"Proyecto", priority:"alta", endMs:d2.getTime(), notes:"Revisar soportes y consolidar evidencias.", alertMinutes: DEFAULT_ALERT_MINUTES.slice(), createdAt: nowMs(), updatedAt: nowMs() },
    { id: uid(), name: "Entrega de informe", category:"Académico", priority:"media", endMs:d1.getTime(), notes:"Ajustar conclusiones y anexos.", alertMinutes: DEFAULT_ALERT_MINUTES.slice(), createdAt: nowMs(), updatedAt: nowMs() },
  ];
  saveEvents(events);
}

// --- Loop
async function main() {
  seedIfEmpty();
  render();
  await registerSW();

  // tick UI each second
  setInterval(() => {
    tickTimers();
  }, 1000);

  // check notifications each 30s
  setInterval(() => {
    maybeNotify();
  }, 30000);

  // run once
  maybeNotify();
}

main();
