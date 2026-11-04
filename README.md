# mini-farm
Farming game

<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#477b36">
<title>Mini Farm 🌱</title>
<style>
*{box-sizing:border-box}
body{margin:0;padding:16px;background:#e8f4dc;color:#263b20;
font-family:system-ui,sans-serif}
h1{text-align:center;color:#386b2d}
.stats{display:flex;justify-content:space-around;gap:8px;
flex-wrap:wrap;background:white;padding:14px;border-radius:14px}
#farm{display:grid;grid-template-columns:repeat(2,1fr);gap:12px;
max-width:650px;margin:20px auto}
.field{min-height:105px;border:3px solid #79532e;border-radius:12px;
background:#75a94d;color:white;font-size:17px;font-weight:bold}
.field.selected{outline:4px solid #e4b82e}
.field.tilled{background:#986b43}
.field.planted{background:#b3d58b;color:#253c1d}
.panel{max-width:650px;margin:15px auto;background:white;
padding:16px;border-radius:16px}
button,select{font-size:17px;padding:13px;border:0;border-radius:10px}
.action{width:100%;margin:5px 0;background:#477b36;color:white}
.action:disabled{opacity:.4}
.secondary{background:#e5eddf;color:#263b20}
.progress{height:12px;background:#ddd;border-radius:10px;overflow:hidden}
.bar{height:100%;background:#68a84b;width:0%}
.small{font-size:14px;color:#63745b}
</style>
</head>
<body>
<h1>🌱 Meine Mini Farm</h1>

<div class="stats">
 <strong>🪙 <span id="coins">30</span></strong>
 <strong>🌾 <span id="harvests">0</span> Ernten</strong>
 <strong>🚜 <span id="ready">0</span> Äcker</strong>
</div>

<div id="farm"></div>

<div class="panel">
 <h2 id="selected">Feld 1</h2>
 <p id="status">Wiese – noch nicht gepflügt.</p>
 <div class="progress"><div class="bar" id="progress"></div></div>
 <p class="small" id="timer"></p>

 <h3>🌱 Samen auswählen</h3>
 <select id="crop">
  <option value="carrot">🥕 Karotte – 2 Münzen</option>
  <option value="corn">🌽 Mais – 5 Münzen</option>
  <option value="berry">🍓 Erdbeere – 8 Münzen</option>
 </select>

 <button class="action" onclick="plow()">🚜 Pflügen (30 Sekunden)</button>
 <button class="action" onclick="plant()">🌱 Pflanzen</button>
 <button class="action" onclick="fertilize()">⚡ Dünger (2 Münzen)</button>
 <button class="action" onclick="harvest()">🌾 Ernten</button>
 <button class="action secondary" onclick="clearField()">Feld zurücksetzen</button>
 <button class="action secondary" onclick="update()">⏱ Zeit aktualisieren</button>
 <button class="action secondary" onclick="shareGame()">📤 Spiel teilen</button>
</div>

<script>
const crops={
 carrot:{name:"Karotte",emoji:"🥕",price:2,reward:5,time:60000},
 corn:{name:"Mais",emoji:"🌽",price:5,reward:12,time:120000},
 berry:{name:"Erdbeere",emoji:"🍓",price:8,reward:18,time:180000}
};

let saved;
try{saved=JSON.parse(localStorage.getItem("miniFarmSave"))}catch(e){}
let coins=saved?.coins??30;
let harvests=saved?.harvests??0;
let selected=0;
let fields=saved?.fields??Array.from({length:10},()=>({
 tilled:false,plowing:null,plant:null
}));

function save(){
 localStorage.setItem("miniFarmSave",JSON.stringify({
 coins,harvests,fields
 }));
}
function remaining(start,duration){
 return Math.max(0,Math.ceil((duration-(Date.now()-start))/1000));
}
function update(){
 const now=Date.now();
 fields.forEach(f=>{
  if(f.plowing && now-f.plowing>=30000){
   f.plowing=null;f.tilled=true;
  }
 });
 render();
 save();
}
function render(){
 document.getElementById("coins").textContent=coins;
 document.getElementById("harvests").textContent=harvests;
 document.getElementById("ready").textContent=
 fields.filter(f=>f.tilled).length;

 const farm=document.getElementById("farm");
 farm.innerHTML="";
 fields.forEach((f,i)=>{
  let label="🌿 Wiese";
  if(f.plowing)label="🚜 "+remaining(f.plowing,30000)+" s";
  else if(f.plant){
   const c=crops[f.plant.type];
   label=c.emoji+" "+c.name;
  }else if(f.tilled)label="🟫 Acker";

  const b=document.createElement("button");
  b.className="field"+(i===selected?" selected":"")+
    (f.plant?" planted":f.tilled?" tilled":"");
  b.textContent="Feld "+(i+1)+"\n"+label;
  b.style.whiteSpace="pre-line";
  b.onclick=()=>{selected=i;render()};
  farm.appendChild(b);
 });

 const f=fields[selected];
 document.getElementById("selected").textContent="Feld "+(selected+1);
 let status="Wiese – noch nicht gepflügt.";
 let progress=0,timer="";

 if(f.plowing){
  const elapsed=Date.now()-f.plowing;
  status="🚜 Pflügen läuft!";
  progress=Math.min(100,elapsed/30000*100);
  timer="Noch "+remaining(f.plowing,30000)+" Sekunden";
 }else if(f.plant){
  const c=crops[f.plant.type];
  const elapsed=Date.now()-f.plant.started;
  const left=remaining(f.plant.started,f.plant.duration);
  status=c.emoji+" "+c.name+(left===0?" – Ernte bereit!":" wächst");
  progress=Math.min(100,elapsed/f.plant.duration*100);
  timer=left===0?"Du kannst jetzt ernten!":"Noch "+left+" Sekunden";
 }else if(f.tilled){
  status="🟫 Gepflügter Acker – bereit zum Pflanzen.";
 }
 document.getElementById("status").textContent=status;
 document.getElementById("timer").textContent=timer;
 document.getElementById("progress").style.width=progress+"%";
}

function plow(){
 const f=fields[selected];
 if(f.tilled||f.plowing||f.plant)return;
 f.plowing=Date.now();
 save();render();
}
function plant(){
 const f=fields[selected];
 if(!f.tilled||f.plant||f.plowing)return;
 const type=document.getElementById("crop").value;
 const c=crops[type];
 if(coins<c.price){alert("Du hast nicht genug Münzen!");return}
 coins-=c.price;
 f.plant={type,started:Date.now(),duration:c.time,boosted:false};
 save();render();
}
function fertilize(){
 const f=fields[selected];
 if(!f.plant||f.plant.boosted)return;
 if(coins<2){alert("Du hast nicht genug Münzen!");return}
 coins-=2;
 f.plant.duration=Math.ceil(f.plant.duration*.75);
 f.plant.boosted=true;
 save();render();
}
function harvest(){
 const f=fields[selected];
 if(!f.plant)return;
 if(Date.now()-f.plant.started<f.plant.duration){
  alert("Die Pflanze ist noch nicht fertig!");return;
 }
 coins+=crops[f.plant.type].reward;
 harvests++;
 f.plant=null;
 save();render();
}
function clearField(){
 if(!confirm("Dieses Feld wirklich zurücksetzen?"))return;
 fields[selected]={tilled:false,plowing:null,plant:null};
 save();render();
}
async function shareGame(){
 const url=location.href;
 if(navigator.share){
  try{await navigator.share({title:"Meine Mini Farm",text:"Spiel meine Farm!",url})}
  catch(e){}
 }else{
  try{await navigator.clipboard.writeText(url);alert("Link kopiert!")}
  catch(e){alert("Kopiere diesen Link: "+url)}
 }
}
render();
setInterval(update,1000);
</script>
</body>
</html>
