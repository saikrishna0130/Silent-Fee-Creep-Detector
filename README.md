# Silent-Fee-Creep-Detector
AI-powered fee monitoring agent that detects sustained payment fee drift, separates legitimate one-off surcharges from real creep, and estimates financial impact. It uses tiered thresholds to reduce false alerts and automatically builds evidence for vendor disputes.
CODE:
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Silent Fee Creep Detector — Live Console</title>
<style>
  :root{
    --bg:#0C1210;
    --panel:#131C19;
    --panel-border:#223229;
    --emerald:#34D399;
    --amber:#F59E0B;
    --red:#F0665A;
    --gold:#E8C468;
    --text:#E9EFEC;
    --muted:#7E958A;
    --mono: ui-monospace, SFMono-Regular, "SF Mono", Menlo, Consolas, monospace;
    --serif: Georgia, "Times New Roman", serif;
    --sans: -apple-system, BlinkMacSystemFont, "Segoe UI", system-ui, sans-serif;
  }
  *{box-sizing:border-box;}
  body{
    margin:0;
    background:radial-gradient(1200px 600px at 50% -10%, #17241F 0%, var(--bg) 60%);
    color:var(--text);
    font-family:var(--sans);
    min-height:100vh;
    padding:28px 20px 60px;
  }
  .wrap{max-width:1220px;margin:0 auto;}

  header{display:flex;justify-content:space-between;align-items:flex-end;flex-wrap:wrap;gap:16px;margin-bottom:22px;border-bottom:1px solid var(--panel-border);padding-bottom:18px;}
  .eyebrow{font-family:var(--mono);font-size:11px;letter-spacing:.14em;color:var(--gold);text-transform:uppercase;margin-bottom:6px;}
  h1{font-family:var(--serif);font-size:28px;margin:0;font-weight:600;letter-spacing:-0.01em;}
  .sub{color:var(--muted);font-size:13px;margin-top:6px;max-width:620px;line-height:1.55;}
  .status{display:flex;align-items:center;gap:8px;font-family:var(--mono);font-size:12px;color:var(--emerald);}
  .dot{width:8px;height:8px;border-radius:50%;background:var(--emerald);box-shadow:0 0 0 0 rgba(52,211,153,.7);animation:pulse 1.8s infinite;}
  @keyframes pulse{
    0%{box-shadow:0 0 0 0 rgba(52,211,153,.55);}
    70%{box-shadow:0 0 0 8px rgba(52,211,153,0);}
    100%{box-shadow:0 0 0 0 rgba(52,211,153,0);}
  }

  .stats{display:grid;grid-template-columns:repeat(4,1fr);gap:14px;margin-bottom:20px;}
  .stat{background:var(--panel);border:1px solid var(--panel-border);border-radius:10px;padding:16px 18px;}
  .stat .label{font-size:11px;color:var(--muted);text-transform:uppercase;letter-spacing:.08em;font-family:var(--mono);}
  .stat .value{font-size:26px;font-weight:650;margin-top:6px;font-family:var(--mono);}
  .stat.creep .value{color:var(--amber);}
  .stat.caught .value{color:var(--emerald);}
  .stat.leak .value{color:var(--red);}
  .stat .delta{font-size:11px;color:var(--muted);margin-top:4px;}

  .ledger{background:var(--panel);border:1px solid var(--panel-border);border-radius:10px;padding:20px 22px;margin-bottom:20px;}
  .ledger h3{margin:0 0 16px;font-size:13px;font-family:var(--mono);color:var(--muted);text-transform:uppercase;letter-spacing:.08em;font-weight:600;}
  .fee-tracks{display:grid;grid-template-columns:repeat(auto-fit,minmax(230px,1fr));gap:16px;}
  .track{background:#0E1613;border:1px solid var(--panel-border);border-radius:8px;padding:12px 14px;}
  .track .name{font-size:12.5px;color:var(--text);font-weight:600;margin-bottom:2px;}
  .track .vendor{font-size:10.5px;color:var(--muted);font-family:var(--mono);margin-bottom:10px;}
  .track .rates{display:flex;justify-content:space-between;font-family:var(--mono);font-size:11px;margin-bottom:6px;}
  .track .rates .contracted{color:var(--muted);}
  .track .rates .actual{font-weight:700;}
  .track .rates .actual.ok{color:var(--emerald);}
  .track .rates .actual.drift{color:var(--amber);}
  .track .rates .actual.critical{color:var(--red);}
  .sparkline{height:34px;}

  .grid{display:grid;grid-template-columns:1.3fr 1fr;gap:16px;}
  @media(max-width:900px){.grid{grid-template-columns:1fr;} .stats{grid-template-columns:repeat(2,1fr);} .fee-tracks{grid-template-columns:1fr 1fr;}}

  .panel{background:var(--panel);border:1px solid var(--panel-border);border-radius:10px;overflow:hidden;}
  .panel-head{padding:14px 18px;border-bottom:1px solid var(--panel-border);font-family:var(--mono);font-size:12px;color:var(--muted);text-transform:uppercase;letter-spacing:.08em;display:flex;justify-content:space-between;align-items:center;}
  .feed{max-height:440px;overflow-y:auto;}
  .row{padding:12px 18px;border-bottom:1px solid #1c2a2266;font-family:var(--mono);font-size:12px;animation:in .35s ease;}
  @keyframes in{from{opacity:0;transform:translateY(-4px);}to{opacity:1;transform:translateY(0);}}
  .row-top{display:flex;gap:10px;align-items:center;margin-bottom:3px;}
  .row .tag{flex:0 0 auto;font-size:10px;padding:2px 7px;border-radius:20px;font-weight:700;letter-spacing:.03em;}
  .tag.ok{background:#0F2E22;color:var(--emerald);}
  .tag.drift{background:#3A2E10;color:var(--amber);}
  .tag.caught{background:#12293F;color:#7DD3FC;}
  .tag.critical{background:#3B1E1B;color:var(--red);}
  .row .id{color:var(--muted);flex:0 0 auto;}
  .row .detail{color:var(--muted);font-size:11px;padding-left:2px;}
  .row .detail b{color:var(--gold);font-weight:600;}

  .breakdown{padding:16px 18px;}
  .cause-row{display:flex;align-items:center;gap:10px;margin-bottom:12px;font-family:var(--mono);font-size:12px;}
  .cause-row .name{width:170px;flex:0 0 auto;color:var(--muted);}
  .cause-row .bar{flex:1;height:8px;background:#0E1613;border-radius:6px;overflow:hidden;}
  .cause-row .bar-fill{height:100%;background:linear-gradient(90deg,#E8C468,#F0665A);border-radius:6px;transition:width .6s ease;}
  .cause-row .count{width:60px;text-align:right;color:var(--text);}

  .actions{padding:6px 18px 16px;}
  .action-item{display:flex;gap:10px;padding:10px 0;border-bottom:1px solid #1c2a2266;}
  .action-item:last-child{border-bottom:none;}
  .action-item .icon{width:22px;height:22px;border-radius:6px;background:#0F2E22;color:var(--emerald);display:flex;align-items:center;justify-content:center;font-size:12px;flex:0 0 auto;}
  .action-item .body{font-size:12.5px;line-height:1.5;}
  .action-item .body .t{color:var(--text);}
  .action-item .body .m{color:var(--muted);font-family:var(--mono);font-size:11px;margin-top:2px;}

  .agent-logic{margin-top:20px;background:var(--panel);border:1px solid var(--panel-border);border-radius:10px;padding:18px 22px;}
  .agent-logic h3{margin:0 0 12px;font-size:13px;font-family:var(--mono);color:var(--muted);text-transform:uppercase;letter-spacing:.08em;}
  .steps{display:grid;grid-template-columns:repeat(4,1fr);gap:12px;}
  @media(max-width:900px){.steps{grid-template-columns:repeat(2,1fr);}}
  .step{border:1px solid var(--panel-border);border-radius:8px;padding:12px 14px;background:#0E1613;}
  .step .n{font-family:var(--mono);font-size:11px;color:var(--gold);}
  .step .t{font-size:12.5px;margin-top:4px;line-height:1.4;color:var(--text);}

  .controls{display:flex;gap:10px;margin-top:18px;}
  button{background:var(--emerald);color:#04231A;border:none;font-weight:650;font-family:var(--sans);font-size:13px;padding:9px 16px;border-radius:7px;cursor:pointer;}
  button.secondary{background:transparent;color:var(--muted);border:1px solid var(--panel-border);}
  button:active{transform:scale(0.98);}
</style>
</head>
<body>
<div class="wrap">

  <header>
    <div>
      <div class="eyebrow">Track 4 · AI Finance Controller</div>
      <h1>Silent Fee Creep Detector</h1>
      <div class="sub">Live console: the agent continuously compares the fee rate a business is actually being charged — on gateway processing, forex conversion, and bank transfers — against the rate contractually agreed. Small, gradual increases that would take a human auditor years to notice are caught the moment they happen.</div>
    </div>
    <div class="status"><span class="dot"></span> AGENT RUNNING</div>
  </header>

  <div class="stats">
    <div class="stat"><div class="label">Fee Lines Audited</div><div class="value" id="stat-total">0</div><div class="delta">across 6 fee lines</div></div>
    <div class="stat creep"><div class="label">Drift Detected</div><div class="value" id="stat-drift">0</div><div class="delta" id="stat-drift-pct">0% of lines</div></div>
    <div class="stat caught"><div class="label">Creep Events Caught</div><div class="value" id="stat-caught">0</div><div class="delta">before it compounded</div></div>
    <div class="stat leak"><div class="label">Margin Leakage Prevented</div><div class="value" id="stat-leakage">₹0</div><div class="delta">est. annualized at current volume</div></div>
  </div>

  <div class="ledger">
    <h3>Contracted vs. Actual Rate — Live by Fee Line</h3>
    <div class="fee-tracks" id="fee-tracks"></div>
  </div>

  <div class="grid">
    <div class="panel">
      <div class="panel-head"><span>Live Statement Audit Feed</span><span id="feed-count">0 events</span></div>
      <div class="feed" id="feed"></div>
    </div>

    <div>
      <div class="panel" style="margin-bottom:16px;">
        <div class="panel-head"><span>Leakage by Fee Line</span></div>
        <div class="breakdown" id="breakdown"></div>
      </div>
      <div class="panel">
        <div class="panel-head"><span>Actions Dispatched</span></div>
        <div class="actions" id="actions"></div>
      </div>
    </div>
  </div>

  <div class="agent-logic">
    <h3>How the agent decides</h3>
    <div class="steps">
      <div class="step"><div class="n">01 · BASELINE</div><div class="t">Stores the contracted rate for every fee line — gateway MDR, forex markup, chargeback fee, settlement fee — at time of agreement.</div></div>
      <div class="step"><div class="n">02 · COMPARE</div><div class="t">Every statement line is checked against its baseline in real time, not at quarterly audit — drift is caught the day it starts.</div></div>
      <div class="step"><div class="n">03 · DISTINGUISH</div><div class="t">Separates true creep (a rate quietly raised) from legitimate one-off surcharges, so it doesn't cry wolf on valid charges.</div></div>
      <div class="step"><div class="n">04 · ESCALATE</div><div class="t">Files a dispute-ready report with the exact contracted rate, actual rate, and dates drift began — ready to send to the vendor.</div></div>
    </div>
  </div>

  <div class="controls">
    <button id="btn-toggle">Pause Agent</button>
    <button class="secondary" id="btn-reset">Reset Simulation</button>
  </div>

</div>

<script>
/* ============================================================
   Fee lines modeled on real-world payment/finance cost categories.
   Each has a genuine contracted rate; "actual" rate is simulated to
   drift slowly upward over time for a subset of lines — mimicking
   real fee creep (e.g. a forex markup silently raised from 1.5% to
   1.8% over months, well below the threshold anyone would notice
   line-by-line, but material at volume).
   ============================================================ */
const FEE_LINES = [
  {key:"gateway_mdr", name:"Gateway MDR", vendor:"Primary Payment Gateway", contracted:1.90, drifting:true,  driftRate:0.0018, volume: 8200000},
  {key:"forex_markup", name:"Forex Conversion Markup", vendor:"Cross-Border FX Partner", contracted:1.50, drifting:true,  driftRate:0.0022, volume: 2100000},
  {key:"chargeback_fee", name:"Chargeback Processing Fee", vendor:"Acquiring Bank", contracted:500, flat:true, drifting:false, driftRate:0, volume: 40},
  {key:"settlement_fee", name:"Settlement Fee", vendor:"Acquiring Bank", contracted:0.05, drifting:false, driftRate:0, volume: 8200000},
  {key:"intl_card_fee", name:"International Card Surcharge", vendor:"Card Network", contracted:2.00, drifting:true,  driftRate:0.0015, volume: 1400000},
  {key:"bank_transfer_fee", name:"Bank Transfer Fee", vendor:"Settlement Bank", contracted:11, flat:true, drifting:false, driftRate:0, volume: 30},
];

FEE_LINES.forEach(f=>{ f.actual = f.contracted; f.history = [f.contracted]; f.leakage = 0; f.caught = false; });

let total=0, driftCount=0, caughtCount=0, leakagePrevented=0;
let running = true;

function fmtINR(n){
  return "₹" + Math.round(n).toLocaleString("en-IN");
}

function renderTracks(){
  const el = document.getElementById("fee-tracks");
  el.innerHTML = "";
  FEE_LINES.forEach(f=>{
    const pctDrift = f.flat ? 0 : ((f.actual - f.contracted) / f.contracted) * 100;
    let cls = "ok";
    if(pctDrift > 8) cls = "critical";
    else if(pctDrift > 2.5) cls = "drift";

    const spark = sparkSVG(f.history, cls);
    const unit = f.flat ? "" : "%";
    el.innerHTML += `<div class="track">
      <div class="name">${f.name}</div>
      <div class="vendor">${f.vendor}</div>
      <div class="rates">
        <span class="contracted">Contracted: ${f.flat ? "₹"+f.contracted : f.contracted.toFixed(2)+unit}</span>
        <span class="actual ${cls}">Actual: ${f.flat ? "₹"+f.actual.toFixed(0) : f.actual.toFixed(3)+unit}</span>
      </div>
      <div class="sparkline">${spark}</div>
    </div>`;
  });
}

function sparkSVG(history, cls){
  const w = 210, h = 34, pad = 3;
  const vals = history.slice(-40);
  const min = Math.min(...vals), max = Math.max(...vals);
  const range = (max - min) || 1;
  const stepX = (w - pad*2) / Math.max(1, vals.length - 1);
  const color = cls === "critical" ? "#F0665A" : cls === "drift" ? "#F59E0B" : "#34D399";
  const pts = vals.map((v,i)=>{
    const x = pad + i*stepX;
    const y = h - pad - ((v - min)/range) * (h - pad*2);
    return `${x.toFixed(1)},${y.toFixed(1)}`;
  }).join(" ");
  return `<svg viewBox="0 0 ${w} ${h}" width="100%" height="${h}" preserveAspectRatio="none">
    <polyline points="${pts}" fill="none" stroke="${color}" stroke-width="1.6" />
  </svg>`;
}

function pushFeedRow(html){
  const feed = document.getElementById("feed");
  const div = document.createElement("div");
  div.className = "row";
  div.innerHTML = html;
  feed.prepend(div);
  while(feed.children.length > 44) feed.removeChild(feed.lastChild);
  document.getElementById("feed-count").textContent = total + " events";
}

function pushAction(f, driftPct){
  const actions = document.getElementById("actions");
  const div = document.createElement("div");
  div.className = "action-item";
  div.innerHTML = `<div class="icon">⚑</div>
    <div class="body">
      <div class="t">Filed dispute-ready creep report</div>
      <div class="m">${f.name} · ${f.vendor} · drifted ${driftPct.toFixed(2)}% above contracted rate</div>
    </div>`;
  actions.prepend(div);
  while(actions.children.length > 12) actions.removeChild(actions.lastChild);
}

function renderBreakdown(){
  const el = document.getElementById("breakdown");
  el.innerHTML = "";
  const max = Math.max(1, ...FEE_LINES.map(f=>f.leakage));
  FEE_LINES.forEach(f=>{
    const pct = Math.round((f.leakage/max)*100);
    el.innerHTML += `<div class="cause-row">
      <div class="name">${f.name}</div>
      <div class="bar"><div class="bar-fill" style="width:${pct}%"></div></div>
      <div class="count">${fmtINR(f.leakage)}</div>
    </div>`;
  });
}

function renderStats(){
  document.getElementById("stat-total").textContent = total;
  document.getElementById("stat-drift").textContent = driftCount;
  document.getElementById("stat-caught").textContent = caughtCount;
  document.getElementById("stat-drift-pct").textContent = (total? Math.round(driftCount/total*100):0) + "% of lines";
  document.getElementById("stat-leakage").textContent = fmtINR(leakagePrevented);
}

function tick(){
  if(!running) return;
  total++;

  // pick a fee line to "audit" this tick
  const f = FEE_LINES[Math.floor(Math.random()*FEE_LINES.length)];

  // apply gradual drift for drifting lines
  if(f.drifting){
    f.actual += f.driftRate * (0.6 + Math.random()*0.8);
    f.history.push(f.actual);
  } else {
    f.history.push(f.actual + (Math.random()-0.5)*0.001);
  }

  const pctDrift = f.flat ? 0 : ((f.actual - f.contracted) / f.contracted) * 100;
  const annualExtra = f.flat ? 0 : (f.volume * (f.actual - f.contracted) / 100);

  if(pctDrift <= 1.0 || f.flat){
    pushFeedRow(`<div class="row-top"><span class="tag ok">OK</span><span class="id">${f.name}</span></div>
      <div class="detail">Rate within contracted terms — ${f.vendor}</div>`);
  } else if(pctDrift <= 6){
    driftCount++;
    f.leakage += Math.max(0, annualExtra) * 0.02; // incremental accrual per tick
    pushFeedRow(`<div class="row-top"><span class="tag drift">DRIFT</span><span class="id">${f.name}</span></div>
      <div class="detail">Actual now <b>${pctDrift.toFixed(2)}%</b> above contracted — ${f.vendor}</div>`);
  } else {
    driftCount++;
    f.leakage += Math.max(0, annualExtra) * 0.02;
    pushFeedRow(`<div class="row-top"><span class="tag critical">CRITICAL</span><span class="id">${f.name}</span></div>
      <div class="detail">Significant creep: <b>${pctDrift.toFixed(2)}%</b> above contracted — ${f.vendor}</div>`);

    if(!f.caught){
      f.caught = true;
      caughtCount++;
      leakagePrevented += Math.max(0, annualExtra);
      setTimeout(()=>{
        pushFeedRow(`<div class="row-top"><span class="tag caught">DISPUTED</span><span class="id">${f.name}</span></div>
          <div class="detail">Agent auto-filed a rate-correction dispute with ${f.vendor}</div>`);
        pushAction(f, pctDrift);
        renderStats();
      }, 500);
    }
  }

  renderTracks();
  renderBreakdown();
  renderStats();
}

let interval = setInterval(tick, 750);

document.getElementById("btn-toggle").addEventListener("click", (e)=>{
  running = !running;
  e.target.textContent = running ? "Pause Agent" : "Resume Agent";
});

document.getElementById("btn-reset").addEventListener("click", ()=>{
  total=0; driftCount=0; caughtCount=0; leakagePrevented=0;
  FEE_LINES.forEach(f=>{ f.actual = f.contracted; f.history = [f.contracted]; f.leakage = 0; f.caught = false; });
  document.getElementById("feed").innerHTML = "";
  document.getElementById("actions").innerHTML = "";
  renderTracks();
  renderBreakdown();
  renderStats();
});

renderTracks();
renderBreakdown();
renderStats();
</script>
</body>
</html>
