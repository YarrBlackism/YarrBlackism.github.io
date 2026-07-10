<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>Gu Master — Cultivation Task Tracker</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@500;700&family=Spectral:ital,wght@0,300;0,400;0,600;1,300&display=swap" rel="stylesheet">
<script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react/18.3.1/umd/react.production.min.js"></script>
<script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.3.1/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.24.7/babel.min.js"></script>
<style>
  html, body { margin: 0; padding: 0; background: #0C110E; }
</style>
</head>
<body>
<div id="root"></div>

<script type="text/babel" data-presets="react">
const { useState, useEffect, useRef } = React;

/* ============================================================
   GU MASTER — a Reverend Insanity-inspired productivity tracker
   Progress is saved to localStorage on every change, so it
   persists whenever you exit and reopen the app.
   ============================================================ */

const STORAGE_KEY = "gu_master_state";

const RANKS = [
  { rank: 1, essenceName: "Green Copper", color: "#7FB08A", glow: "rgba(127,176,138,0.35)", capacity: 100 },
  { rank: 2, essenceName: "Red Steel", color: "#C9614F", glow: "rgba(201,97,79,0.35)", capacity: 180 },
  { rank: 3, essenceName: "White Silver", color: "#C7D0DC", glow: "rgba(199,208,220,0.35)", capacity: 300 },
  { rank: 4, essenceName: "Yellow Gold", color: "#D9A441", glow: "rgba(217,164,65,0.4)", capacity: 480 },
  { rank: 5, essenceName: "Purple Crystal", color: "#9D7BD8", glow: "rgba(157,123,216,0.4)", capacity: 720 },
];
const STAGES = ["Initial", "Middle", "Upper", "Peak"];

const IMMORTAL_RANKS = [
  { rank: 6, essenceName: "Green Grape", color: "#93C24E", glow: "rgba(147,194,78,0.4)", aperture: "Blessed Land", beadCap: 25, tribsNeeded: 3 },
  { rank: 7, essenceName: "Red Date", color: "#C34A33", glow: "rgba(195,74,51,0.4)", aperture: "Blessed Land", beadCap: 40, tribsNeeded: 3 },
  { rank: 8, essenceName: "White Litchi", color: "#EAE4D3", glow: "rgba(234,228,211,0.4)", aperture: "Grotto-Heaven", beadCap: 60, tribsNeeded: 4 },
  { rank: 9, essenceName: "Yellow Apricot", color: "#E9B44C", glow: "rgba(233,180,76,0.45)", aperture: "Grotto-Heaven", beadCap: 90, tribsNeeded: 5 },
];
const VENERABLE = { color: "#F4E8C8", glow: "rgba(244,232,200,0.55)" };

const PATHS = [
  { id: "strength", glyph: "力", name: "Strength", hint: "Physical work, exercise, chores" },
  { id: "wisdom", glyph: "智", name: "Wisdom", hint: "Study, reading, coursework" },
  { id: "blood", glyph: "血", name: "Blood", hint: "Hard grinds, painful obligations" },
  { id: "flame", glyph: "炎", name: "Flame", hint: "Urgent, deadline-driven tasks" },
  { id: "water", glyph: "水", name: "Water", hint: "Routine maintenance, admin" },
  { id: "time", glyph: "时", name: "Time", hint: "Planning, scheduling, long-term" },
  { id: "dream", glyph: "梦", name: "Dream", hint: "Creative projects, side quests" },
  { id: "sword", glyph: "剑", name: "Sword", hint: "Direct goals, one decisive cut" },
];

const RANK_ESSENCE = { 1: 10, 2: 22, 3: 40, 4: 65, 5: 100 };
const RANK_BEADS = { 1: 1, 2: 2, 3: 4, 4: 7, 5: 12 };

const todayKey = () => new Date().toISOString().slice(0, 10);

const DEFAULT_STATE = {
  essence: 0,
  stageIndex: 0,
  readyToAscend: false,
  immortal: null,
  venerable: false,
  tasks: [],
  refinedCount: 0,
  tribulationsSurvived: 0,
  streak: 0,
  lastRefineDay: null,
};

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return DEFAULT_STATE;
    const merged = { ...DEFAULT_STATE, ...JSON.parse(raw) };
    if (!merged.immortal && merged.stageIndex >= 20) {
      merged.immortal = { rank: 6, beads: 0, tribs: 0 };
      merged.stageIndex = 19;
      merged.essence = 0;
      merged.readyToAscend = false;
    }
    return merged;
  } catch (e) {
    return DEFAULT_STATE;
  }
}

function mortalInfo(stageIndex) {
  const clamped = Math.min(stageIndex, 19);
  return { rankInfo: RANKS[Math.floor(clamped / 4)], stage: STAGES[clamped % 4] };
}
const immortalInfo = (rank) => IMMORTAL_RANKS[rank - 6];

function GuMasterTracker() {
  const [state, setState] = useState(loadState);
  const [name, setName] = useState("");
  const [path, setPath] = useState("wisdom");
  const [rank, setRank] = useState(1);
  const [toast, setToast] = useState(null);
  const [pulse, setPulse] = useState(false);
  const toastTimer = useRef(null);
  const fileRef = useRef(null);

  // ---- autosave on every change ----
  useEffect(() => {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
    } catch (e) {
      console.error("Aperture failed to seal (save error)", e);
    }
  }, [state]);

  const isImmortal = !!state.immortal;
  const m = mortalInfo(state.stageIndex);
  const im = isImmortal ? immortalInfo(state.immortal.rank) : null;

  const accent = state.venerable ? VENERABLE.color : isImmortal ? im.color : m.rankInfo.color;
  const glow = state.venerable ? VENERABLE.glow : isImmortal ? im.glow : m.rankInfo.glow;

  const pct = state.venerable
    ? 100
    : isImmortal
      ? Math.min(100, Math.round((state.immortal.beads / im.beadCap) * 100))
      : Math.min(100, Math.round((state.essence / m.rankInfo.capacity) * 100));

  const title = state.venerable
    ? "Venerable"
    : isImmortal
      ? `Rank ${state.immortal.rank} Gu Immortal`
      : `Rank ${m.rankInfo.rank} ${m.stage} Stage`;

  const showToast = (msg) => {
    setToast(msg);
    clearTimeout(toastTimer.current);
    toastTimer.current = setTimeout(() => setToast(null), 3600);
  };

  const addGu = () => {
    const trimmed = name.trim();
    if (!trimmed) return;
    const gu = { id: Date.now() + Math.random().toString(16).slice(2), name: trimmed, path, rank, created: Date.now() };
    setState((s) => ({ ...s, tasks: [gu, ...s.tasks] }));
    setName("");
  };

  const bumpStreak = (s) => {
    const today = todayKey();
    if (s.lastRefineDay === today) return s.streak;
    const yesterday = new Date(Date.now() - 86400000).toISOString().slice(0, 10);
    return s.lastRefineDay === yesterday ? s.streak + 1 : 1;
  };

  const refineGu = (gu) => {
    setPulse(true);
    setTimeout(() => setPulse(false), 900);

    setState((s) => {
      const base = {
        ...s,
        streak: bumpStreak(s),
        lastRefineDay: todayKey(),
        refinedCount: s.refinedCount + 1,
        tasks: s.tasks.filter((t) => t.id !== gu.id),
      };

      /* ---------- Immortal cultivation: beads & tribulations ---------- */
      if (s.immortal && !s.venerable) {
        let { rank, beads, tribs } = s.immortal;
        let tribsSurvived = s.tribulationsSurvived;
        beads += RANK_BEADS[gu.rank];
        let info = immortalInfo(rank);
        let msg = `Your ${info.aperture.toLowerCase()} produced ${RANK_BEADS[gu.rank]} bead${RANK_BEADS[gu.rank] > 1 ? "s" : ""} of ${info.essenceName.toLowerCase()} immortal essence.`;
        let venerable = false;

        while (beads >= info.beadCap) {
          beads -= info.beadCap;
          tribs += 1;
          tribsSurvived += 1;
          if (tribs >= info.tribsNeeded) {
            tribs = 0;
            if (rank >= 9) {
              venerable = true;
              msg = "The final tribulation parts like a curtain. Heaven's will bows — you are Venerable.";
              break;
            }
            rank += 1;
            info = immortalInfo(rank);
            msg = `Breakthrough! Rank ${rank} Gu Immortal — your aperture now yields ${info.essenceName.toLowerCase()} immortal essence.`;
          } else {
            msg = `A heavenly tribulation descends… and you endure. (${tribs}/${info.tribsNeeded} toward Rank ${rank + 1})`;
          }
        }
        showToast(msg);
        return { ...base, immortal: { rank, beads, tribs }, tribulationsSurvived: tribsSurvived, venerable: venerable || s.venerable };
      }

      if (s.venerable) {
        showToast("A Venerable still keeps a task list. Eternity is long.");
        return base;
      }

      /* ---------- Mortal cultivation: primeval essence & stages ---------- */
      let essence = s.essence + RANK_ESSENCE[gu.rank];
      let stageIndex = s.stageIndex;
      let readyToAscend = s.readyToAscend;
      let advanced = false;

      while (essence >= RANKS[Math.floor(stageIndex / 4)].capacity) {
        if (stageIndex >= 19) {
          essence = RANKS[4].capacity;
          readyToAscend = true;
          break;
        }
        essence -= RANKS[Math.floor(stageIndex / 4)].capacity;
        stageIndex += 1;
        advanced = true;
      }

      if (readyToAscend && !s.readyToAscend) {
        showToast("Your aperture brims at Rank 5 Peak. The three qi of heaven, earth, and human stir — ascension awaits.");
      } else if (advanced) {
        const c = mortalInfo(stageIndex);
        showToast(`Breakthrough! Rank ${c.rankInfo.rank} ${c.stage} Stage — essence turns ${c.rankInfo.essenceName.toLowerCase()}.`);
      } else {
        showToast(`Refinement succeeded. +${RANK_ESSENCE[gu.rank]} primeval essence.`);
      }
      return { ...base, essence, stageIndex, readyToAscend };
    });
  };

  const attemptAscension = () => {
    setPulse(true);
    setTimeout(() => setPulse(false), 900);
    setState((s) => ({
      ...s,
      immortal: { rank: 6, beads: 0, tribs: 0 },
      essence: 0,
      readyToAscend: false,
    }));
    showToast("Your mortal aperture shatters. Heaven qi and earth qi pour in, balanced by your human accumulation — it reforms as a Blessed Land. You are a Gu Immortal.");
  };

  const discardGu = (gu) => {
    setState((s) => ({ ...s, tasks: s.tasks.filter((t) => t.id !== gu.id) }));
    showToast(`The ${gu.name} Gu perished unrefined.`);
  };

  const resetAll = () => {
    if (!window.confirm("Cripple your cultivation and start over? All progress is lost.")) return;
    setState(DEFAULT_STATE);
    showToast("Your aperture has been crippled. Begin again from mortal flesh.");
  };

  // ---- export / import save (soul backup) ----
  const exportSave = () => {
    const blob = new Blob([JSON.stringify(state, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `gu-master-save-${todayKey()}.json`;
    a.click();
    URL.revokeObjectURL(url);
    showToast("Soul backup created. Guard it well.");
  };

  const importSave = (e) => {
    const file = e.target.files && e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const parsed = JSON.parse(reader.result);
        if (typeof parsed !== "object" || parsed === null || typeof parsed.stageIndex !== "number") {
          throw new Error("bad save");
        }
        setState({ ...DEFAULT_STATE, ...parsed });
        showToast("Soul backup restored. Your cultivation returns.");
      } catch (err) {
        showToast("That file is not a valid save. The heavens reject it.");
      }
    };
    reader.readAsText(file);
    e.target.value = "";
  };

  const apertureName = state.venerable ? "Sovereign Aperture" : isImmortal ? im.aperture : "Mortal Aperture";

  return (
    <div className="gm-root" style={{ "--accent": accent, "--glow": glow }}>
      <style>{`
        .gm-root {
          min-height: 100vh; background: radial-gradient(1200px 800px at 50% -10%, #131B16 0%, #0C110E 60%);
          color: #D8DFD6; font-family: 'Spectral', Georgia, serif; padding: 28px 16px 80px; box-sizing: border-box;
        }
        .gm-wrap { max-width: 640px; margin: 0 auto; }
        .gm-eyebrow { font-family: 'Cinzel', serif; letter-spacing: 0.35em; font-size: 11px; color: #7E8B80; text-transform: uppercase; text-align: center; }
        .gm-title { font-family: 'Cinzel', serif; font-weight: 700; font-size: clamp(26px, 6vw, 38px); text-align: center; margin: 6px 0 2px; color: var(--accent); text-shadow: 0 0 24px var(--glow); transition: color .6s, text-shadow .6s; }
        .gm-sub { text-align: center; color: #93A096; font-size: 14px; font-style: italic; margin-bottom: 26px; }
        .gm-panel { background: rgba(21,29,25,0.85); border: 1px solid #263229; border-radius: 14px; padding: 18px; margin-bottom: 18px; }
        .gm-aperture-wrap { display:flex; flex-direction: column; align-items: center; gap: 10px; padding: 26px 18px; }
        .gm-stats { display:flex; gap: 22px; justify-content:center; flex-wrap: wrap; color:#93A096; font-size:13px; }
        .gm-stats b { color:#D8DFD6; font-weight: 600; }
        .gm-label { font-family:'Cinzel',serif; font-size: 11px; letter-spacing: .2em; text-transform: uppercase; color:#7E8B80; margin-bottom:8px; display:block; }
        .gm-input, .gm-select { width:100%; background:#0F1512; border:1px solid #2C3A31; color:#D8DFD6; border-radius:8px; padding:10px 12px; font-family:inherit; font-size:15px; outline:none; box-sizing: border-box; }
        .gm-input:focus, .gm-select:focus { border-color: var(--accent); box-shadow: 0 0 0 2px var(--glow); }
        .gm-row { display:flex; gap:10px; margin-top:10px; flex-wrap: wrap; }
        .gm-row > * { flex:1; min-width: 140px; }
        .gm-btn { background: var(--accent); color:#0C110E; border:none; border-radius:8px; padding:11px 16px; font-family:'Cinzel',serif; font-weight:700; letter-spacing:.08em; cursor:pointer; font-size:13px; transition: transform .1s, box-shadow .2s; }
        .gm-btn:hover { box-shadow: 0 0 18px var(--glow); }
        .gm-btn:active { transform: scale(.97); }
        .gm-btn.ghost { background: transparent; color:#93A096; border:1px solid #2C3A31; font-weight:500; }
        .gm-btn.ghost:hover { color:#D8DFD6; box-shadow:none; border-color:#465548; }
        .gm-btn.ascend { animation: gm-beckon 2.2s ease-in-out infinite; }
        @keyframes gm-beckon { 0%,100% { box-shadow: 0 0 8px var(--glow); } 50% { box-shadow: 0 0 30px var(--glow); } }
        .gm-task { display:flex; align-items:center; gap:12px; padding:12px 4px; border-bottom:1px solid #1E2823; }
        .gm-task:last-child { border-bottom:none; }
        .gm-glyph { font-size:22px; width:44px; height:44px; border-radius:10px; display:flex; align-items:center; justify-content:center; background:#0F1512; border:1px solid #2C3A31; color:var(--accent); flex-shrink:0; }
        .gm-task-name { font-size:15.5px; }
        .gm-task-meta { font-size:12px; color:#7E8B80; margin-top:2px; }
        .gm-task-actions { margin-left:auto; display:flex; gap:8px; flex-shrink:0; }
        .gm-mini { border:1px solid #2C3A31; background:transparent; color:#93A096; border-radius:7px; padding:7px 12px; cursor:pointer; font-family:'Cinzel',serif; font-size:11px; letter-spacing:.06em; }
        .gm-mini.refine { border-color: var(--accent); color: var(--accent); }
        .gm-mini.refine:hover { background: var(--accent); color:#0C110E; box-shadow:0 0 14px var(--glow); }
        .gm-mini:hover { color:#D8DFD6; }
        .gm-empty { text-align:center; color:#7E8B80; font-style:italic; padding:26px 10px; font-size:14px; }
        .gm-toast { position:fixed; bottom:22px; left:50%; transform:translateX(-50%); background:#151D19; border:1px solid var(--accent); color:#D8DFD6; padding:12px 20px; border-radius:10px; font-size:14px; box-shadow:0 6px 30px rgba(0,0,0,.5), 0 0 20px var(--glow); animation: gm-rise .3s ease; max-width: 90vw; text-align:center; z-index:50; }
        @keyframes gm-rise { from { opacity:0; transform:translate(-50%, 12px);} to { opacity:1; transform:translate(-50%,0);} }
        @keyframes gm-wave { from { transform: translateX(0); } to { transform: translateX(-200px); } }
        @keyframes gm-drift { from { transform: translateX(0); } to { transform: translateX(-120px); } }
        @keyframes gm-pulse { 0% { filter: drop-shadow(0 0 0px var(--glow)); } 40% { filter: drop-shadow(0 0 26px var(--glow)); } 100% { filter: drop-shadow(0 0 6px var(--glow)); } }
        .gm-aperture { animation: ${pulse ? "gm-pulse .9s ease" : "none"}; filter: drop-shadow(0 0 8px var(--glow)); }
        @media (prefers-reduced-motion: reduce) { .gm-aperture *, .gm-toast, .gm-btn.ascend { animation: none !important; } }
        .gm-tribs { display:flex; gap:8px; justify-content:center; }
        .gm-trib-dot { width:12px; height:12px; border-radius:50%; border:1.5px solid var(--accent); opacity:.45; }
        .gm-trib-dot.passed { background: var(--accent); opacity:1; box-shadow:0 0 8px var(--glow); }
        .gm-save-row { display:flex; gap:10px; justify-content:center; flex-wrap:wrap; margin-top:14px; }
        .gm-footer { text-align:center; color:#5C685E; font-size:12px; font-style:italic; margin-top:30px; }
      `}</style>

      <div className="gm-wrap">
        <div className="gm-eyebrow">{apertureName} · Cultivation Record</div>
        <h1 className="gm-title">{title}</h1>
        <div className="gm-sub">
          {state.venerable
            ? "Beneath the heavens, an invincible existence. The task list, however, remains."
            : isImmortal
              ? "Calamities and tribulations forge Gu Immortals. Each task endured is heaven defied."
              : "A Gu Master relies on no one. Refine your tasks, fill your aperture."}
        </div>

        {/* ---- Aperture / Blessed Land ---- */}
        <div className="gm-panel gm-aperture-wrap">
          <ApertureOrb pct={pct} accent={accent} world={isImmortal || state.venerable} venerable={state.venerable} />
          <div style={{ fontFamily: "'Cinzel',serif", fontSize: 13, letterSpacing: ".12em", color: "var(--accent)" }}>
            {state.venerable ? "Beyond Essence" : isImmortal ? `${im.essenceName} Immortal Essence` : `${m.rankInfo.essenceName} Essence`}
          </div>

          {state.venerable ? (
            <div style={{ color: "#93A096", fontSize: 13 }}>All tribulations conquered.</div>
          ) : isImmortal ? (
            <React.Fragment>
              <div style={{ color: "#93A096", fontSize: 13 }}>
                {state.immortal.beads} / {im.beadCap} beads until the next tribulation
              </div>
              <div className="gm-tribs" title="Tribulations survived at this rank">
                {Array.from({ length: im.tribsNeeded }).map((_, i) => (
                  <div key={i} className={"gm-trib-dot " + (i < state.immortal.tribs ? "passed" : "")} />
                ))}
              </div>
              <div style={{ color: "#7E8B80", fontSize: 12 }}>
                Survive {im.tribsNeeded} tribulations to reach {state.immortal.rank >= 9 ? "Venerable" : `Rank ${state.immortal.rank + 1}`}
              </div>
            </React.Fragment>
          ) : (
            <div style={{ color: "#93A096", fontSize: 13 }}>
              {state.essence} / {m.rankInfo.capacity} essence to next stage
            </div>
          )}

          {state.readyToAscend && !isImmortal && (
            <button className="gm-btn ascend" onClick={attemptAscension}>
              Undergo Immortal Ascension
            </button>
          )}

          <div className="gm-stats">
            <span><b>{state.refinedCount}</b> Gu refined</span>
            <span><b>{state.tribulationsSurvived}</b> tribulations survived</span>
            <span><b>{state.streak}</b> day streak</span>
            <span><b>{state.tasks.length}</b> Gu awaiting</span>
          </div>
        </div>

        {/* ---- Add Gu ---- */}
        <div className="gm-panel">
          <span className="gm-label">Capture a new Gu (add task)</span>
          <input
            className="gm-input"
            placeholder="Name the task…  e.g. Finish HTB Web Attacks module"
            value={name}
            onChange={(e) => setName(e.target.value)}
            onKeyDown={(e) => e.key === "Enter" && addGu()}
          />
          <div className="gm-row">
            <div>
              <span className="gm-label" style={{ marginTop: 10 }}>Path</span>
              <select className="gm-select" value={path} onChange={(e) => setPath(e.target.value)}>
                {PATHS.map((p) => (
                  <option key={p.id} value={p.id}>{p.glyph} {p.name} — {p.hint}</option>
                ))}
              </select>
            </div>
            <div>
              <span className="gm-label" style={{ marginTop: 10 }}>Gu Rank (difficulty)</span>
              <select className="gm-select" value={rank} onChange={(e) => setRank(Number(e.target.value))}>
                {[1, 2, 3, 4, 5].map((r) => (
                  <option key={r} value={r}>
                    {isImmortal
                      ? `Rank ${r} — ${RANK_BEADS[r]} bead${RANK_BEADS[r] > 1 ? "s" : ""}`
                      : `Rank ${r} — ${RANK_ESSENCE[r]} essence`}
                  </option>
                ))}
              </select>
            </div>
          </div>
          <div className="gm-row">
            <button className="gm-btn" onClick={addGu}>Capture Gu</button>
          </div>
        </div>

        {/* ---- Task list ---- */}
        <div className="gm-panel">
          <span className="gm-label">Gu awaiting refinement</span>
          {state.tasks.length === 0 ? (
            <div className="gm-empty">
              {isImmortal
                ? "Your blessed land lies fallow. Capture a Gu — even immortals must tend their estates."
                : "Your aperture is quiet. Capture a Gu above — even a Rank 1 Moonlight-grade errand feeds cultivation."}
            </div>
          ) : (
            state.tasks.map((gu) => {
              const p = PATHS.find((x) => x.id === gu.path) || PATHS[0];
              return (
                <div className="gm-task" key={gu.id}>
                  <div className="gm-glyph" title={p.name + " Path"}>{p.glyph}</div>
                  <div>
                    <div className="gm-task-name">{gu.name}</div>
                    <div className="gm-task-meta">
                      {p.name} Path · Rank {gu.rank} · {isImmortal ? `+${RANK_BEADS[gu.rank]} beads` : `+${RANK_ESSENCE[gu.rank]} essence`}
                    </div>
                  </div>
                  <div className="gm-task-actions">
                    <button className="gm-mini refine" onClick={() => refineGu(gu)}>Refine</button>
                    <button className="gm-mini" onClick={() => discardGu(gu)} title="Delete task">Discard</button>
                  </div>
                </div>
              );
            })
          )}
        </div>

        {/* ---- Save management ---- */}
        <div className="gm-save-row">
          <button className="gm-btn ghost" onClick={exportSave}>Export save (soul backup)</button>
          <button className="gm-btn ghost" onClick={() => fileRef.current && fileRef.current.click()}>Import save</button>
          <button className="gm-btn ghost" onClick={resetAll}>Cripple cultivation (reset)</button>
          <input ref={fileRef} type="file" accept="application/json,.json" style={{ display: "none" }} onChange={importSave} />
        </div>

        <div className="gm-footer">
          {isImmortal
            ? "Even a grotto-heaven withers when its master idles. Refine daily."
            : "Refine daily, lest your streak-Gu starve. Fate is unfair — schedule accordingly."}
          <br />Progress is saved automatically in this browser.
        </div>
      </div>

      {toast && <div className="gm-toast">{toast}</div>}
    </div>
  );
}

/* Aperture visualization: mortal liquid essence vs. immortal blessed land */
function ApertureOrb({ pct, accent, world, venerable }) {
  const size = 210;
  const r = 96;
  const cx = size / 2, cy = size / 2;
  const fillY = size - (pct / 100) * (size - 18) - 9;

  return (
    <svg className="gm-aperture" width={size} height={size} viewBox={`0 0 ${size} ${size}`} role="img"
      aria-label={world ? `Blessed land, ${pct}% to next tribulation` : `Aperture ${pct}% full`}>
      <defs>
        <clipPath id="apClip"><circle cx={cx} cy={cy} r={r} /></clipPath>
        <linearGradient id="essGrad" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%" stopColor={accent} stopOpacity="0.95" />
          <stop offset="100%" stopColor={accent} stopOpacity="0.55" />
        </linearGradient>
        <linearGradient id="skyGrad" x1="0" y1="0" x2="0" y2="1">
          <stop offset="0%" stopColor={accent} stopOpacity="0.30" />
          <stop offset="70%" stopColor="#0F1512" stopOpacity="1" />
        </linearGradient>
      </defs>

      <circle cx={cx} cy={cy} r={r + 5} fill="none" stroke="#2C3A31" strokeWidth="2" />
      {world && (
        <circle cx={cx} cy={cy} r={r + 5} fill="none" stroke={accent} strokeWidth="2.5"
          strokeDasharray={`${(pct / 100) * 2 * Math.PI * (r + 5)} ${2 * Math.PI * (r + 5)}`}
          strokeLinecap="round" transform={`rotate(-90 ${cx} ${cy})`} opacity="0.9" />
      )}
      <circle cx={cx} cy={cy} r={r} fill="#0F1512" stroke={accent} strokeWidth="2.5" strokeOpacity="0.8" />

      {world ? (
        <g clipPath="url(#apClip)">
          <rect x="0" y="0" width={size} height={size} fill="url(#skyGrad)" />
          <circle cx={cx + 34} cy={cy - 38} r={venerable ? 20 : 15} fill={accent} opacity="0.9">
            <animate attributeName="opacity" values="0.75;1;0.75" dur="5s" repeatCount="indefinite" />
          </circle>
          <g style={{ animation: "gm-drift 9s linear infinite" }}>
            <ellipse cx={cx - 30} cy={cy - 20} rx="42" ry="7" fill={accent} opacity="0.14" />
            <ellipse cx={cx + 90} cy={cy - 4} rx="55" ry="8" fill={accent} opacity="0.10" />
            <ellipse cx={cx + 210} cy={cy - 22} rx="42" ry="7" fill={accent} opacity="0.14" />
          </g>
          <path d={`M -10 ${cy + 44} q 45 -42 90 0 q 40 -34 80 0 q 30 -22 60 0 V ${size} H -10 Z`} fill={accent} opacity="0.28" />
          <path d={`M -10 ${cy + 66} q 60 -26 120 0 q 55 -20 110 0 V ${size} H -10 Z`} fill={accent} opacity="0.5" />
          <ellipse cx={cx - 26} cy={cy + 74} rx="34" ry="7" fill="#0F1512" opacity="0.55" />
        </g>
      ) : (
        <g clipPath="url(#apClip)">
          <g style={{ animation: "gm-wave 4s linear infinite" }}>
            <path
              d={`M -200 ${fillY}
                  q 25 -9 50 0 t 50 0 t 50 0 t 50 0 t 50 0 t 50 0 t 50 0 t 50 0 t 50 0 t 50 0 t 50 0 t 50 0
                  V ${size + 20} H -200 Z`}
              fill="url(#essGrad)"
            />
          </g>
        </g>
      )}

      <text x="50%" y={world ? "88%" : "50%"} textAnchor="middle" dominantBaseline="central"
        fontFamily="Cinzel, serif" fontWeight="700" fontSize={world ? 20 : 34}
        fill={world ? "#D8DFD6" : pct > 55 ? "#0C110E" : "#D8DFD6"}>
        {venerable ? "∞" : pct + "%"}
      </text>
    </svg>
  );
}

ReactDOM.createRoot(document.getElementById("root")).render(<GuMasterTracker />);
</script>
</body>
</html>
