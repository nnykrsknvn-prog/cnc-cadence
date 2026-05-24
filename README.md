# cnc-cadence
<!DOCTYPE html>
<html lang="pt">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>CNC Cadence</title>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;600;700;800&family=DM+Mono:wght@400;600&display=swap" rel="stylesheet"/>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.5/babel.min.js"></script>
<style>
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
  body { background: #F0F2F4; font-family: 'DM Sans', sans-serif; color: #1C2128; -webkit-font-smoothing: antialiased; }
  input[type=time]::-webkit-calendar-picker-indicator { opacity: 0.4; }
  input[type=number]::-webkit-inner-spin-button { -webkit-appearance: none; }
  ::-webkit-scrollbar { width: 6px; } ::-webkit-scrollbar-track { background: #F0F2F4; } ::-webkit-scrollbar-thumb { background: #C4C9D0; border-radius: 3px; }
</style>
</head>
<body>
<div id="root"></div>
<script type="text/babel">
const { useState, useEffect, useRef } = React;
const ACCENTS = ["#E85A2A","#2A8FE8","#2ABA6E","#C44FD8","#D4A827"];
function pad(n){return String(n).padStart(2,"0");}
function getNow(){const d=new Date();return d.getHours()*60+d.getMinutes()+d.getSeconds()/60;}
function getNowFull(){const d=new Date();return `${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`;}
let uid=1;

function ImageUpload({value,onChange,label}){
  const ref=useRef();
  function handleFile(e){
    const f=e.target.files[0];if(!f)return;
    const r=new FileReader();r.onload=ev=>onChange(ev.target.result);r.readAsDataURL(f);
  }
  return(
    <div style={{display:"flex",flexDirection:"column",gap:5}}>
      <span style={{fontSize:9,letterSpacing:1,color:"#9BA3AE",textTransform:"uppercase",fontWeight:600}}>{label}</span>
      <div onClick={()=>ref.current.click()} style={{width:"100%",aspectRatio:"1/1",borderRadius:8,border:`1.5px dashed ${value?"#3D445199":"#D0D4D8"}`,background:value?"#F5F6F7":"#FAFAFA",display:"flex",flexDirection:"column",alignItems:"center",justifyContent:"center",cursor:"pointer",overflow:"hidden",position:"relative"}}>
        {value
          ?<><img src={value} alt="" style={{width:"100%",height:"100%",objectFit:"cover"}}/><button onClick={e=>{e.stopPropagation();onChange(null);}} style={{position:"absolute",top:5,right:5,background:"rgba(0,0,0,0.45)",border:"none",borderRadius:"50%",color:"#fff",fontSize:10,width:20,height:20,cursor:"pointer",display:"flex",alignItems:"center",justifyContent:"center"}}>✕</button></>
          :<div style={{display:"flex",flexDirection:"column",alignItems:"center",gap:5}}><span style={{fontSize:22,opacity:0.35}}>📷</span><span style={{fontSize:9,color:"#9BA3AE",letterSpacing:0.5}}>CARREGAR</span></div>
        }
      </div>
      <input ref={ref} type="file" accept="image/*" style={{display:"none"}} onChange={handleFile}/>
    </div>
  );
}

function Ring({pct,accent,size=88}){
  const r=size/2-8,circ=2*Math.PI*r;
  return(
    <svg width={size} height={size} style={{flexShrink:0}}>
      <circle cx={size/2} cy={size/2} r={r} fill="none" stroke="#E8EAEC" strokeWidth={7}/>
      <circle cx={size/2} cy={size/2} r={r} fill="none" stroke={accent} strokeWidth={7}
        strokeDasharray={`${circ*Math.min(pct,1)} ${circ}`} strokeLinecap="round"
        transform={`rotate(-90 ${size/2} ${size/2})`}/>
      <text x={size/2} y={size/2+5} textAnchor="middle" fill={accent} fontSize={14}
        fontFamily="'DM Mono',monospace" fontWeight="600">{Math.round(pct*100)}%</text>
    </svg>
  );
}

function Pill({label,value,accent}){
  return(
    <div style={{background:"#F5F6F7",display:"flex",flexDirection:"column",alignItems:"center",padding:"10px 6px",gap:3}}>
      <span style={{fontSize:15,fontWeight:700,fontFamily:"'DM Mono',monospace",letterSpacing:-0.5,color:accent}}>{value}</span>
      <span style={{fontSize:8,color:"#9BA3AE",textTransform:"uppercase",letterSpacing:0.8,textAlign:"center"}}>{label}</span>
    </div>
  );
}

function App(){
  const [machines,setMachines]=useState([]);
  const [shiftStart,setShiftStart]=useState("14:00");
  const [shiftEnd,setShiftEnd]=useState("22:00");
  const [now,setNow]=useState(getNow());
  const [clock,setClock]=useState(getNowFull());
  const [modal,setModal]=useState(false);
  const [editId,setEditId]=useState(null);
  const [form,setForm]=useState({machine:"",part:"",cycleMin:"",cycleSec:"",machineImg:null,partImg:null});

  useEffect(()=>{const t=setInterval(()=>{setNow(getNow());setClock(getNowFull());},1000);return()=>clearInterval(t);},[]);

  const startMin=(()=>{const[h,m]=shiftStart.split(":").map(Number);return h*60+m;})();
  const endMin=(()=>{const[h,m]=shiftEnd.split(":").map(Number);let e=h*60+m;if(e<=startMin)e+=1440;return e;})();
  const duration=endMin-startMin;
  const elapsed=Math.min(Math.max(now-startMin,0),duration);
  const shiftPct=duration>0?elapsed/duration:0;

  function calc(m){
    const cm=(parseFloat(m.cycleMin)||0)+(parseFloat(m.cycleSec)||0)/60;
    if(cm<=0)return{expected:0,total:0,pct:0,cm:0};
    const total=Math.floor(duration/cm),expected=Math.floor(elapsed/cm);
    return{expected,total,pct:
