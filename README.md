<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>🔍 DataLens — Analyse de données</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/PapaParse/5.4.1/papaparse.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/recharts/2.12.7/Recharts.min.js"></script>
<style>
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Helvetica,sans-serif;background:#f8f9fc;color:#111;min-height:100vh}
select,button,input{font-family:inherit}
::-webkit-scrollbar{width:6px;height:6px}
::-webkit-scrollbar-thumb{background:#c7d2fe;border-radius:3px}
::-webkit-scrollbar-track{background:transparent}
.tab-btn{background:none;border:none;cursor:pointer;padding:9px 14px;font-size:13px;white-space:nowrap;transition:all .12s;border-bottom:2.5px solid transparent;color:#6b7280}
.tab-btn.active{font-weight:600;color:#6366f1;border-bottom-color:#6366f1}
.card{background:#fff;border:1px solid rgba(0,0,0,.09);border-radius:11px;padding:13px 15px;box-shadow:0 1px 3px rgba(0,0,0,.04)}
.meta-card{background:#fff;border:1px solid rgba(0,0,0,.09);border-radius:10px;padding:9px 16px;text-align:center;min-width:80px;box-shadow:0 1px 3px rgba(0,0,0,.04)}
table{width:100%;border-collapse:collapse;font-size:12.5px}
th{padding:9px 12px;text-align:left;font-weight:500;white-space:nowrap;background:#1e1b4b;color:#fff}
td{padding:7px 12px;border-bottom:1px solid rgba(0,0,0,.07);white-space:nowrap}
tr:nth-child(even) td{background:#f9fafb}
.upload-zone{border:2px dashed rgba(0,0,0,.2);border-radius:16px;padding:72px 24px;text-align:center;cursor:pointer;transition:all .18s;background:#fff}
.upload-zone.drag,.upload-zone:hover{border-color:#6366f1;background:#eef2ff}
.btn-primary{background:#6366f1;color:#fff;border:none;border-radius:9px;padding:10px 22px;cursor:pointer;font-size:13.5px;font-weight:600;transition:opacity .15s}
.btn-primary:hover{opacity:.88}
.btn-primary:disabled{opacity:.55;cursor:not-allowed}
.btn-green{background:#10b981;color:#fff;border:none;border-radius:8px;padding:7px 15px;cursor:pointer;font-size:13px;font-weight:600;transition:opacity .15s}
.btn-green:hover{opacity:.88}
.btn-ghost{background:rgba(255,255,255,.12);color:#fff;border:1px solid rgba(255,255,255,.22);border-radius:8px;padding:7px 15px;cursor:pointer;font-size:13px;font-weight:500;transition:all .15s}
.btn-ghost:hover{background:rgba(255,255,255,.2)}
select.std{padding:6px 12px;border-radius:8px;border:1px solid rgba(0,0,0,.18);background:#fff;font-size:13px;color:#111;cursor:pointer}
.anom-banner{background:#fef2f2;border:1px solid #fecaca;border-radius:9px;padding:12px 16px;color:#b91c1c;font-weight:500;font-size:13.5px;margin-bottom:14px}
.ai-box{background:#eef2ff;border:1px solid #c7d2fe;border-radius:11px;padding:20px 24px;line-height:1.85;font-size:14px;color:#1e1b4b;white-space:pre-wrap}
.corr-row{display:flex;align-items:center;gap:12px;background:#fff;border:1px solid rgba(0,0,0,.07);border-radius:8px;padding:10px 14px;margin-bottom:6px}
.corr-bar-track{width:90px;height:6px;background:rgba(0,0,0,.1);border-radius:4px;flex-shrink:0;overflow:hidden}
.stat-chip{background:#f3f4f6;border-radius:8px;padding:10px 14px;text-align:center}
.section-title{font-size:14px;font-weight:600;margin-bottom:12px;color:#374151}
</style>
</head>
<body>
<div id="root"></div>
<script>
const {createElement:h,useState,useRef,useCallback}=React
const {BarChart,Bar,XAxis,YAxis,CartesianGrid,Tooltip,ResponsiveContainer,Cell}=Recharts
const PAL=['#6366f1','#10b981','#f59e0b','#ef4444','#8b5cf6','#06b6d4','#ec4899','#14b8a6','#f97316','#84cc16']
const IND='#6366f1'
 
const fmtN=(v,d=3)=>v==null?'—':(+v).toLocaleString('fr-FR',{maximumFractionDigits:d,minimumFractionDigits:d})
 
function calcStats(vals){
  const nums=vals.filter(v=>typeof v==='number'&&isFinite(v))
  if(!nums.length)return null
  const s=[...nums].sort((a,b)=>a-b),n=nums.length
  const mean=nums.reduce((a,v)=>a+v,0)/n
  const med=n%2?s[Math.floor(n/2)]:(s[n/2-1]+s[n/2])/2
  const std=Math.sqrt(nums.reduce((a,v)=>a+(v-mean)**2,0)/n)
  const q1=s[Math.floor(n*.25)],q3=s[Math.floor(n*.75)],iqr=q3-q1
  const skew=std?nums.reduce((a,v)=>a+((v-mean)/std)**3,0)/n:0
  return{mean,med,std,min:s[0],max:s[n-1],q1,q3,iqr,skew,n}
}
 
function pearson(x,y){
  const n=Math.min(x.length,y.length);if(n<2)return 0
  const mx=x.reduce((a,v)=>a+v,0)/n,my=y.reduce((a,v)=>a+v,0)/n
  const num=x.reduce((a,v,i)=>a+(v-mx)*(y[i]-my),0)
  const den=Math.sqrt(x.reduce((a,v)=>a+(v-mx)**2,0)*y.reduce((a,v)=>a+(v-my)**2,0))
  return den?+(num/den).toFixed(3):0
}
 
function colType(vals){
  const ok=vals.filter(v=>v!=null&&v!=='')
  if(!ok.length)return'text'
  const n=ok.filter(v=>isFinite(Number(String(v).replace(/[\s]/g,'').replace(',','.')))).length
  return n/ok.length>.85?'number':'text'
}
 
function makeHist(vals,bins=22){
  const nums=vals.filter(v=>typeof v==='number'&&isFinite(v))
  if(!nums.length)return[]
  const mn=Math.min(...nums),mx=Math.max(...nums),w=(mx-mn)/bins||1
  const b=Array.from({length:bins},(_,i)=>({x:+(mn+i*w).toFixed(3),n:0}))
  nums.forEach(v=>{b[Math.min(Math.floor((v-mn)/w),bins-1)].n++})
  return b
}
 
function parseFile(file,onOk,onErr){
  const ext=file.name.split('.').pop().toLowerCase()
  const process=raw=>{
    if(!raw.length)return onErr('Fichier vide')
    const keys=Object.keys(raw[0])
    const cols=keys.map(name=>({name,type:colType(raw.map(r=>r[name]))}))
    const rows=raw.map(r=>{
      const out={}
      cols.forEach(({name,type})=>{
        if(type==='number'){const v=parseFloat(String(r[name]||'').replace(/\s/g,'').replace(',','.'));out[name]=isFinite(v)?v:null}
        else out[name]=r[name]==null?'':String(r[name])
      })
      return out
    })
    onOk({cols,rows})
  }
  if(['csv','tsv'].includes(ext)){
    Papa.parse(file,{header:true,skipEmptyLines:true,delimiter:ext==='tsv'?'\t':undefined,complete:r=>process(r.data),error:e=>onErr(e.message)})
  }else if(['xlsx','xls'].includes(ext)){
    const rd=new FileReader()
    rd.onload=e=>{try{const wb=XLSX.read(e.target.result,{type:'binary'});process(XLSX.utils.sheet_to_json(wb.Sheets[wb.SheetNames[0]]))}catch(e){onErr(e.message)}}
    rd.onerror=()=>onErr('Erreur de lecture')
    rd.readAsBinaryString(file)
  }else onErr('Format non supporté. Utilise CSV, TSV ou Excel (.xlsx/.xls)')
}
 
function runAnalysis(df){
  const numC=df.cols.filter(c=>c.type==='number').map(c=>c.name)
  const catC=df.cols.filter(c=>c.type==='text').map(c=>c.name)
  const numSt=numC.map(col=>({col,...calcStats(df.rows.map(r=>r[col]))})).filter(s=>s&&s.n)
  const corr=numC.length>=2?numC.map((a,i)=>numC.map((b,j)=>{if(i===j)return 1;const p=df.rows.filter(r=>r[a]!=null&&r[b]!=null);return pearson(p.map(r=>r[a]),p.map(r=>r[b]))})):[]
  const catF={}
  catC.forEach(col=>{const f={};df.rows.forEach(r=>{const v=String(r[col]||'');f[v]=(f[v]||0)+1});catF[col]=Object.entries(f).sort((a,b)=>b[1]-a[1]).slice(0,15).map(([label,count])=>({label,count}))})
  const anom=new Set()
  numSt.forEach(({col,q1,q3,iqr})=>{const lo=q1-1.5*iqr,hi=q3+1.5*iqr;df.rows.forEach((r,i)=>{if(r[col]!=null&&(r[col]<lo||r[col]>hi))anom.add(i)})})
  return{numC,catC,numSt,corr,catF,anom}
}
 
function doExport(df,an){
  const wb=XLSX.utils.book_new()
  XLSX.utils.book_append_sheet(wb,XLSX.utils.aoa_to_sheet([["DataLens — Rapport d'analyse"],["Généré le",new Date().toLocaleString('fr-FR')],[],["Lignes",df.rows.length],["Colonnes",df.cols.length],["Colonnes numériques",an.numC.length],["Anomalies IQR",an.anom.size]]),'Synthèse')
  XLSX.utils.book_append_sheet(wb,XLSX.utils.json_to_sheet(df.rows),'Données')
  if(an.numSt.length)XLSX.utils.book_append_sheet(wb,XLSX.utils.json_to_sheet(an.numSt.map(s=>({Colonne:s.col,Moyenne:+s.mean.toFixed(4),Médiane:+s.med.toFixed(4),'Écart-type':+s.std.toFixed(4),Min:s.min,Max:s.max,Q1:+s.q1.toFixed(4),Q3:+s.q3.toFixed(4),IQR:+s.iqr.toFixed(4),Skewness:+s.skew.toFixed(4)}))),'KPIs')
  if(an.corr.length)XLSX.utils.book_append_sheet(wb,XLSX.utils.aoa_to_sheet([['', ...an.numC],...an.corr.map((r,i)=>[an.numC[i],...r])]),'Corrélations')
  if(an.anom.size)XLSX.utils.book_append_sheet(wb,XLSX.utils.json_to_sheet(df.rows.filter((_,i)=>an.anom.has(i))),'Anomalies')
  XLSX.writeFile(wb,'datalens_rapport.xlsx')
}
 
function Heatmap({matrix,labels}){
  const n=labels.length,cell=Math.max(34,Math.min(60,Math.floor(440/n))),lw=92,lh=46,W=lw+cell*n,H=lh+cell*n
  const col=v=>v>=0?`rgba(99,102,241,${Math.abs(v)*.85+.1})`:`rgba(239,68,68,${Math.abs(v)*.85+.1})`
  return h('svg',{width:'100%',viewBox:`0 0 ${W} ${H}`,style:{maxWidth:W}},
    labels.map((l,i)=>h('text',{key:'r'+i,x:lw-5,y:lh+i*cell+cell/2,textAnchor:'end',fontSize:Math.min(11,cell*.36),fill:'#6b7280',dominantBaseline:'central'},l.length>13?l.slice(0,13)+'…':l)),
    labels.map((l,j)=>h('text',{key:'c'+j,x:lw+j*cell+cell/2,y:lh-6,textAnchor:'middle',fontSize:Math.min(11,cell*.36),fill:'#6b7280',transform:`rotate(-35,${lw+j*cell+cell/2},${lh-6})`},l.length>11?l.slice(0,11)+'…':l)),
    matrix.map((row,i)=>row.map((v,j)=>h('g',{key:`${i}-${j}`},
      h('rect',{x:lw+j*cell,y:lh+i*cell,width:cell-2,height:cell-2,rx:3,fill:col(v)}),
      cell>40&&h('text',{x:lw+j*cell+cell/2,y:lh+i*cell+cell/2,textAnchor:'middle',dominantBaseline:'central',fontSize:Math.min(10,cell*.29),fill:Math.abs(v)>.55?'#fff':'#1e1b4b'},v.toFixed(2))
    )))
  )
}
 
function App(){
  const [df,setDf]=useState(null),[an,setAn]=useState(null),[tab,setTab]=useState('kpis')
  const [busy,setBusy]=useState(false),[err,setErr]=useState(''),[drag,setDrag]=useState(false)
  const [nSel,setNSel]=useState(null),[cSel,setCSel]=useState(null)
  const [aiTxt,setAiTxt]=useState(''),[aiLoad,setAiLoad]=useState(false)
  const ref=useRef()
 
  const load=useCallback(file=>{
    if(!file)return
    setBusy(true);setErr('');setDf(null);setAn(null);setAiTxt('')
    parseFile(file,data=>{
      const a=runAnalysis(data);setDf(data);setAn(a)
      setNSel(data.cols.find(c=>c.type==='number')?.name||null)
      setCSel(data.cols.find(c=>c.type==='text')?.name||null)
      setTab('kpis');setBusy(false)
    },msg=>{setErr(msg);setBusy(false)})
  },[])
 
  const onDrop=e=>{e.preventDefault();setDrag(false);load(e.dataTransfer.files[0])}
 
  async function genAI(){
    if(!an||!df)return
    setAiLoad(true);setAiTxt('')
    try{
      const pairs=[]
      an.numC.forEach((a,i)=>an.numC.forEach((b,j)=>{if(j>i)pairs.push({a,b,r:an.corr[i][j]})}))
      const payload={rows:df.rows.length,cols:df.cols.length,numCols:an.numC,catCols:an.catC,anomalies:an.anom.size,
        kpis:an.numSt.map(s=>({col:s.col,mean:+s.mean.toFixed(3),std:+s.std.toFixed(3),min:s.min,max:s.max,skew:+s.skew.toFixed(3)})),
        topCorr:pairs.sort((x,y)=>Math.abs(y.r)-Math.abs(x.r)).slice(0,5)}
      const res=await fetch('https://api.anthropic.com/v1/messages',{method:'POST',headers:{'Content-Type':'application/json'},
        body:JSON.stringify({model:'claude-sonnet-4-20250514',max_tokens:1000,
          system:"Tu es expert data analyst. Génère une synthèse (~8 phrases) en français du dataset : tendances, corrélations notables, anomalies, 2-3 recommandations concrètes. Sois direct, utilise les chiffres.",
          messages:[{role:'user',content:JSON.stringify(payload)}]})})
      const d=await res.json()
      setAiTxt(d.content?.[0]?.text||'Analyse indisponible.')
    }catch{setAiTxt('Erreur de connexion. Vérifie ta connexion internet.')}
    setAiLoad(false)
  }
 
  const TABS=df?[
    {id:'kpis',l:'🏁 KPIs'},{id:'dist',l:'📊 Distributions'},{id:'corr',l:'🔗 Corrélations'},
    {id:'cat',l:'🗂️ Catégories'},{id:'anom',l:`⚠️ Anomalies${an?.anom.size?` (${an.anom.size})`:''}`},
    {id:'data',l:'📋 Données'},{id:'ai',l:'🤖 Analyse IA'}
  ]:[]
 
  return h('div',{style:{minHeight:'100vh',background:'#f8f9fc'}},
    // ── Header
    h('div',{style:{background:'#1e1b4b',padding:'15px 24px',display:'flex',alignItems:'center',justifyContent:'space-between',gap:10,flexWrap:'wrap',boxShadow:'0 2px 12px rgba(0,0,0,.18)'}},
      h('div',null,
        h('div',{style:{color:'#fff',fontWeight:700,fontSize:20,letterSpacing:'-.02em'}},'🔍 DataLens'),
        h('div',{style:{color:'rgba(255,255,255,.42)',fontSize:11.5,marginTop:1}},'Import · Analyse · Dashboard · Export Excel')
      ),
      h('div',{style:{display:'flex',gap:8,flexWrap:'wrap'}},
        h('button',{className:'btn-ghost',onClick:()=>ref.current?.click()},`📂 ${df?'Changer de fichier':'Importer un fichier'}`),
        df&&an&&h('button',{className:'btn-green',onClick:()=>doExport(df,an)},'⬇️ Export Excel')
      )
    ),
    h('input',{ref,type:'file',accept:'.csv,.tsv,.xlsx,.xls',style:{display:'none'},onChange:e=>load(e.target.files[0])}),
 
    h('div',{style:{padding:'22px 24px',maxWidth:1200,margin:'0 auto'}},
 
      // ── Upload
      !df&&!busy&&h('div',{
        className:`upload-zone${drag?' drag':''}`,
        onDrop,onDragOver:e=>{e.preventDefault();setDrag(true)},onDragLeave:()=>setDrag(false),
        onClick:()=>ref.current?.click()
      },
        h('div',{style:{fontSize:52}},'📊'),
        h('p',{style:{fontWeight:700,fontSize:20,margin:'16px 0 7px',color:'#1e1b4b'}},'Dépose ton fichier ici'),
        h('p',{style:{color:'#6b7280',fontSize:14}},'CSV · TSV · Excel (.xlsx / .xls) — jusqu\'à 50 MB'),
        h('p',{style:{color:'#a5b4fc',fontSize:13,marginTop:14}},'ou clique pour parcourir'),
        err&&h('div',{style:{marginTop:18,background:'#fef2f2',border:'1px solid #fecaca',borderRadius:8,padding:'10px 16px',color:'#b91c1c',fontSize:13}},'❌ '+err)
      ),
 
      // ── Loading
      busy&&h('div',{style:{textAlign:'center',padding:'80px 0'}},
        h('div',{style:{fontSize:40,display:'inline-block',animation:'spin 1.2s linear infinite'}},'⏳'),
        h('p',{style:{fontWeight:500,marginTop:16,color:'#374151',fontSize:15}},'Chargement et analyse en cours…'),
        h('style',null,'@keyframes spin{to{transform:rotate(360deg)}}')
      ),
 
      // ── Dashboard
      df&&an&&h('div',null,
 
        // Meta strip
        h('div',{style:{display:'flex',gap:10,marginBottom:20,flexWrap:'wrap'}},
          [{l:'Lignes',v:df.rows.length.toLocaleString('fr-FR'),c:IND},{l:'Colonnes',v:df.cols.length,c:'#0891b2'},
           {l:'Numériques',v:an.numC.length,c:'#059669'},{l:'Catégorielles',v:an.catC.length,c:'#7c3aed'},
           {l:'Anomalies',v:an.anom.size,c:an.anom.size?'#dc2626':'#059669'}]
          .map(m=>h('div',{key:m.l,className:'meta-card'},
            h('div',{style:{fontSize:21,fontWeight:700,color:m.c}},m.v),
            h('div',{style:{fontSize:11,color:'#6b7280',marginTop:2}},m.l)
          ))
        ),
 
        // Tabs
        h('div',{style:{background:'#fff',borderRadius:10,boxShadow:'0 1px 4px rgba(0,0,0,.07)',marginBottom:18}},
          h('div',{style:{display:'flex',gap:0,borderBottom:'1px solid rgba(0,0,0,.08)',overflowX:'auto',padding:'0 8px'}},
            TABS.map(t=>h('button',{key:t.id,className:`tab-btn${tab===t.id?' active':''}`,onClick:()=>setTab(t.id)},t.l))
          ),
 
          h('div',{style:{padding:'20px'}},
 
            // ── KPIs
            tab==='kpis'&&(!an.numSt.length?h('p',{style:{color:'#6b7280'}},'Aucune colonne numérique détectée.'):h('div',null,
              h('div',{style:{display:'grid',gridTemplateColumns:'repeat(auto-fill,minmax(168px,1fr))',gap:12,marginBottom:24}},
                an.numSt.map((s,i)=>h('div',{key:s.col,className:'card',style:{borderLeft:`4px solid ${PAL[i%PAL.length]}`}},
                  h('div',{style:{fontSize:11,color:'#6b7280',textTransform:'uppercase',letterSpacing:'.06em',marginBottom:4,overflow:'hidden',textOverflow:'ellipsis',whiteSpace:'nowrap'}},s.col),
                  h('div',{style:{fontSize:24,fontWeight:700,color:PAL[i%PAL.length]}},fmtN(s.mean,2)),
                  h('div',{style:{fontSize:11.5,color:'#6b7280',marginTop:3}},'σ = '+fmtN(s.std,2)+' · mdn '+fmtN(s.med,2))
                ))
              ),
              h('div',{className:'section-title'},'Tableau statistique complet'),
              h('div',{style:{overflowX:'auto',borderRadius:8,border:'1px solid rgba(0,0,0,.07)'}},
                h('table',null,
                  h('thead',null,h('tr',null,['Colonne','Moy.','Méd.','σ','Min','Max','Q1','Q3','IQR','Skew','n'].map(c=>h('th',{key:c},c)))),
                  h('tbody',null,an.numSt.map((s,i)=>h('tr',{key:s.col},
                    h('td',{style:{fontWeight:600,color:'#1e1b4b'}},s.col),
                    ...[s.mean,s.med,s.std,s.min,s.max,s.q1,s.q3,s.iqr,s.skew].map((v,j)=>h('td',{key:j},fmtN(v))),
                    h('td',null,s.n.toLocaleString('fr-FR'))
                  )))
                )
              )
            )),
 
            // ── Distributions
            tab==='dist'&&(!an.numC.length?h('p',{style:{color:'#6b7280'}},'Aucune colonne numérique.'):h('div',null,
              h('div',{style:{display:'flex',alignItems:'center',gap:10,marginBottom:18,flexWrap:'wrap'}},
                h('span',{style:{fontSize:13,fontWeight:500}},'Colonne :'),
                h('select',{className:'std',value:nSel||'',onChange:e=>setNSel(e.target.value)},
                  an.numC.map(c=>h('option',{key:c},c))
                )
              ),
              nSel&&(()=>{
                const s=an.numSt.find(x=>x.col===nSel),bins=makeHist(df.rows.map(r=>r[nSel]))
                return h('div',null,
                  h('div',{style:{display:'grid',gridTemplateColumns:'repeat(auto-fill,minmax(115px,1fr))',gap:10,marginBottom:20}},
                    s&&[['Moyenne',s.mean],['Médiane',s.med],['σ',s.std],['Min',s.min],['Max',s.max],['IQR',s.iqr],['Skewness',s.skew]].map(([l,v])=>
                      h('div',{key:l,className:'stat-chip'},
                        h('div',{style:{fontSize:18,fontWeight:700,color:IND}},fmtN(v,2)),
                        h('div',{style:{fontSize:11,color:'#6b7280',marginTop:2}},l)
                      )
                    )
                  ),
                  h('div',{className:'section-title'},'Histogramme de distribution'),
                  h(ResponsiveContainer,{width:'100%',height:230},
                    h(BarChart,{data:bins,margin:{top:4,right:8,left:0,bottom:0}},
                      h(CartesianGrid,{strokeDasharray:'3 3',stroke:'rgba(0,0,0,.07)'}),
                      h(XAxis,{dataKey:'x',tick:{fontSize:10},tickFormatter:v=>Number(v).toLocaleString('fr-FR',{maximumFractionDigits:2})}),
                      h(YAxis,{tick:{fontSize:10}}),
                      h(Tooltip,{formatter:v=>[v,'Fréquence']}),
                      h(Bar,{dataKey:'n',fill:IND,radius:[3,3,0,0]})
                    )
                  )
                )
              })()
            )),
 
            // ── Corrélations
            tab==='corr'&&(an.numC.length<2?h('p',{style:{color:'#6b7280'}},'Il faut au moins 2 colonnes numériques pour calculer les corrélations.'):h('div',null,
              h('div',{className:'section-title'},'Heatmap des corrélations (Pearson)'),
              h('div',{style:{overflowX:'auto',marginBottom:24}},h(Heatmap,{matrix:an.corr,labels:an.numC})),
              h('div',{className:'section-title'},'Top corrélations'),
              (()=>{
                const p=[]
                an.numC.forEach((a,i)=>an.numC.forEach((b,j)=>{if(j>i)p.push({a,b,r:an.corr[i][j]})}))
                return p.sort((x,y)=>Math.abs(y.r)-Math.abs(x.r)).slice(0,10).map(({a,b,r})=>
                  h('div',{key:`${a}-${b}`,className:'corr-row'},
                    h('span',{style:{flex:1,fontSize:13,fontWeight:500,overflow:'hidden',textOverflow:'ellipsis',whiteSpace:'nowrap'}},`${a} × ${b}`),
                    h('div',{className:'corr-bar-track'},h('div',{style:{width:`${Math.abs(r)*100}%`,height:'100%',background:r>0?IND:'#ef4444',borderRadius:4}})),
                    h('span',{style:{fontWeight:700,color:Math.abs(r)>.5?(r>0?IND:'#ef4444'):'#9ca3af',minWidth:52,textAlign:'right',fontSize:13}},`${r>0?'+':''}${r.toFixed(3)}`)
                  )
                )
              })()
            )),
 
            // ── Catégories
            tab==='cat'&&(!an.catC.length?h('p',{style:{color:'#6b7280'}},'Aucune colonne catégorielle détectée.'):h('div',null,
              h('div',{style:{display:'flex',alignItems:'center',gap:10,marginBottom:18,flexWrap:'wrap'}},
                h('span',{style:{fontSize:13,fontWeight:500}},'Colonne :'),
                h('select',{className:'std',value:cSel||'',onChange:e=>setCSel(e.target.value)},
                  an.catC.map(c=>h('option',{key:c},c))
                )
              ),
              cSel&&an.catF[cSel]&&h(ResponsiveContainer,{width:'100%',height:Math.min(440,an.catF[cSel].length*30+40)},
                h(BarChart,{data:an.catF[cSel],layout:'vertical',margin:{top:4,right:30,left:115,bottom:4}},
                  h(CartesianGrid,{strokeDasharray:'3 3',stroke:'rgba(0,0,0,.07)'}),
                  h(XAxis,{type:'number',tick:{fontSize:11}}),
                  h(YAxis,{type:'category',dataKey:'label',tick:{fontSize:11},width:115}),
                  h(Tooltip),
                  h(Bar,{dataKey:'count',name:'Occurrences',radius:[0,3,3,0]},
                    an.catF[cSel].map((_,i)=>h(Cell,{key:i,fill:PAL[i%PAL.length]}))
                  )
                )
              )
            )),
 
            // ── Anomalies
            tab==='anom'&&(!an.anom.size?
              h('div',{style:{textAlign:'center',padding:'50px 0'}},
                h('div',{style:{fontSize:48}},'✅'),
                h('p',{style:{fontWeight:600,marginTop:14,fontSize:16,color:'#1e1b4b'}},'Aucune anomalie détectée'),
                h('p',{style:{color:'#6b7280',fontSize:13,marginTop:6}},'Méthode IQR × 1.5 appliquée sur toutes les colonnes numériques')
              ):h('div',null,
                h('div',{className:'anom-banner'},`⚠️ ${an.anom.size} ligne${an.anom.size>1?'s':''} anormale${an.anom.size>1?'s':''} détectée${an.anom.size>1?'s':''} — méthode IQR ± 1.5 par colonne`),
                h('div',{style:{overflowX:'auto',maxHeight:420,overflowY:'auto',borderRadius:8,border:'1px solid rgba(0,0,0,.08)'}},
                  h('table',null,
                    h('thead',null,h('tr',null,['#',...df.cols.slice(0,10).map(c=>c.name)].map(hd=>h('th',{key:hd,style:{background:'#dc2626'}},hd)))),
                    h('tbody',null,df.rows.filter((_,i)=>an.anom.has(i)).slice(0,200).map((row,ri)=>
                      h('tr',{key:ri},
                        h('td',{style:{color:'#9ca3af'}},ri+1),
                        df.cols.slice(0,10).map(c=>h('td',{key:c.name},row[c.name]==null?'—':String(row[c.name]).slice(0,32)))
                      )
                    ))
                  )
                )
              )
            ),
 
            // ── Données
            tab==='data'&&h('div',null,
              h('div',{style:{overflowX:'auto',maxHeight:480,overflowY:'auto',borderRadius:8,border:'1px solid rgba(0,0,0,.08)'}},
                h('table',null,
                  h('thead',null,h('tr',null,df.cols.map(c=>h('th',{key:c.name},c.name,' ',h('span',{style:{opacity:.38,fontSize:9.5}},c.type==='number'?'🔢':'🔤'))))),
                  h('tbody',null,df.rows.slice(0,300).map((row,i)=>h('tr',{key:i},
                    df.cols.map(c=>h('td',{key:c.name},row[c.name]==null?h('span',{style:{color:'#d1d5db'}},'—'):String(row[c.name]).slice(0,50)))
                  )))
                )
              ),
              df.rows.length>300&&h('p',{style:{color:'#9ca3af',fontSize:12,textAlign:'center',marginTop:10}},`300 lignes affichées sur ${df.rows.length.toLocaleString('fr-FR')} au total`)
            ),
 
            // ── IA
            tab==='ai'&&h('div',null,
              h('p',{style:{color:'#6b7280',fontSize:14,marginBottom:18,lineHeight:1.6}},'Génère une synthèse en langage naturel de tes données grâce à Claude (connexion internet requise).'),
              h('button',{className:'btn-primary',onClick:genAI,disabled:aiLoad},aiLoad?'⏳ Analyse en cours…':'🤖 Générer la synthèse IA'),
              aiTxt&&h('div',{className:'ai-box',style:{marginTop:20}},
                h('strong',{style:{display:'block',marginBottom:10,color:IND,fontSize:15}},'📊 Synthèse IA'),
                aiTxt
              )
            )
 
          ) // end tab content
        ) // end tab card
      ) // end dashboard
    ) // end container
  )
}
 
ReactDOM.createRoot(document.getElementById('root')).render(h(App))
</script>
</body>
</html>
