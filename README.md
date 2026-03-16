import { useState, useRef } from "react";
import * as XLSX from "xlsx";

var SKIP_TABS = ["sales","stock","abc curve","forecast","control","safety stock","wallet","sortimento","resumo","dashboard","config","parametros","capa","indice"];
var norm = function(s){return (s||"").toLowerCase().trim().normalize("NFD").replace(/[\u0300-\u036f]/g,"");};
var pNum = function(v){if(v==null||v==="")return 0;if(typeof v==="number")return v;return parseFloat(String(v).replace(/[R$\s]/g,"").replace(/\./g,"").replace(",","."))||0;};
var PERIOD = 15;

function parseSheet(ws){
  var range=XLSX.utils.decode_range(ws["!ref"]||"A1");
  var r1=[],r2=[];
  for(var c=0;c<=range.e.c;c++){
    var c1=ws[XLSX.utils.encode_cell({r:0,c})],c2=ws[XLSX.utils.encode_cell({r:1,c})];
    r1.push(c1?String(c1.v||"").trim():"");
    r2.push(c2?String(c2.v||"").trim():"");
  }
  var groups=r1.slice();
  for(var i=1;i<groups.length;i++) if(!groups[i]) groups[i]=groups[i-1];
  var stockCols=[],salesCols=[];
  var skuCol=null,descCol=null,curvaCol=null,catCol=null,estTotalCol=null;
  r2.forEach(function(h,i){
    if(!h)return;var n=norm(h),g=norm(groups[i]);
    if(!skuCol&&(n.includes("sku")||n==="codigo")){skuCol=h;return;}
    if(!descCol&&n.includes("descri")){descCol=h;return;}
    if(!curvaCol&&n.includes("curva")){curvaCol=h;return;}
    if(!catCol&&n.includes("categor")){catCol=h;return;}
    if(!estTotalCol&&n.includes("estoque total")){estTotalCol=h;return;}
    if(n.includes("empresa")||n.includes("custo")||n.includes("preco")||n.includes("valor")||n.includes("margem"))return;
    if(g.includes("estoque")&&!n.includes("total")&&!n.includes("custo")) stockCols.push(h);
    else if(g.includes("venda")||g.includes("vendas")||g.includes("faturamento")||g.includes("origem")) salesCols.push(h);
  });
  var data=[];
  for(var r=2;r<=range.e.r;r++){
    var obj={};var has=false;
    r2.forEach(function(h,c){if(!h)return;var cell=ws[XLSX.utils.encode_cell({r:r,c:c})];obj[h]=cell?(cell.v!=null?cell.v:""):"";if(cell&&cell.v!=null&&cell.v!=="")has=true;});
    if(has&&obj[skuCol]) data.push(obj);
  }
  return {data:data,skuCol:skuCol,descCol:descCol,curvaCol:curvaCol,catCol:catCol,estTotalCol:estTotalCol,stockCols:stockCols,salesCols:salesCols};
}

function parseBaseData(ws){
  var json=XLSX.utils.sheet_to_json(ws,{defval:""});
  if(!json.length) return {};
  var headers=Object.keys(json[0]);
  var skuCol=headers.find(function(h){var n=norm(h);return n.includes("sku")||n==="codigo"||n==="cod";})||headers[0];
  var multCol=headers.find(function(h){var n=norm(h);return n.includes("multiplo")||n.includes("múltiplo")||n.includes("mult")||n.includes("cx")||n.includes("caixa")||n.includes("embalagem")||n.includes("pack")||n.includes("un/cx")||n.includes("qtd cx");});
  var map={};
  if(skuCol&&multCol){
    json.forEach(function(row){
      var sku=String(row[skuCol]||"").trim();
      var mult=pNum(row[multCol]);
      if(sku&&mult>0) map[norm(sku)]={mult:mult};
    });
  }
  return map;
}

function autoMap(sales,stock){
  var map={};
  var tokens=function(s){return norm(s).split(/[\s\-–_,.:;/\\|()]+/).filter(function(t){return t.length>=2;});};
  var RK=["es","sp","rj","mg","pr","sc","ba","go","ce","pe","rs","df"];
  sales.forEach(function(sc){
    var sT=tokens(sc);var best=null,bs=-1;
    stock.forEach(function(stk){
      var tT=tokens(stk);var score=0;
      RK.forEach(function(r){if(sT.some(function(t){return t===r;})&&tT.some(function(t){return t===r;}))score+=10;});
      sT.forEach(function(st){if(st.length<3||RK.includes(st))return;tT.forEach(function(tt){if(tt.length<3||RK.includes(tt))return;if(st===tt)score+=4;else if(st.includes(tt)||tt.includes(st))score+=2;});});
      if(score>bs){bs=score;best=stk;}
    });
    if(best&&bs>0) map[sc]=best;
  });
  return map;
}

function waterFillWithLocks(channels, totalQty, boxMult, locks){
  // locks: { cdName: targetDays } - these get exactly what's needed, rest is water-filled
  var locked=[], free=[];
  var usedByLocked=0;

  channels.forEach(function(c){
    var rate=c.sales15d/PERIOD;
    var daysBefore=rate>0?c.stock/rate:Infinity;
    var base={cd:c.cd,salesCol:c.salesCol,stock:c.stock,sales15d:c.sales15d,rate:rate,daysBefore:daysBefore,daysAfter:daysBefore,send:0,sendRaw:0,boxes:null,loose:0,isLocked:false,targetDays:null};

    if(locks&&locks[c.cd]!=null&&locks[c.cd]!==""&&rate>0){
      var target=parseFloat(locks[c.cd]);
      var needed=Math.max(0,Math.round(target*rate-c.stock));
      base.send=needed;
      base.sendRaw=needed;
      base.isLocked=true;
      base.targetDays=target;
      base.daysAfter=rate>0?(c.stock+needed)/rate:Infinity;
      usedByLocked+=needed;
      locked.push(base);
    } else {
      free.push(base);
    }
  });

  var remaining=Math.max(0,totalQty-usedByLocked);

  // Water-fill on free channels
  var active=free.filter(function(c){return c.rate>0;});
  var zero=free.filter(function(c){return c.rate<=0;});

  if(active.length>0&&remaining>0){
    active.sort(function(a,b){return a.daysBefore-b.daysBefore;});
    var lvls=active.map(function(c){return c.daysBefore;});
    var rem=remaining;

    for(var i=0;i<active.length&&rem>0;i++){
      var cur=lvls[i],nxt=i+1<active.length?lvls[i+1]:Infinity;
      if(nxt<=cur) continue;
      var totalRate=0;
      for(var j=0;j<=i;j++) totalRate+=active[j].rate;
      var gap=nxt-cur;
      var needed2=Math.ceil(gap*totalRate);
      if(rem>=needed2&&nxt!==Infinity){
        for(var j2=0;j2<=i;j2++) lvls[j2]=nxt;
        rem-=needed2;
      } else {
        var extra=rem/totalRate;
        for(var j3=0;j3<=i;j3++) lvls[j3]+=extra;
        rem=0;
      }
    }
    if(rem>0){
      var tr=active.reduce(function(s,c){return s+c.rate;},0);
      if(tr>0){var ed=rem/tr;lvls.forEach(function(_,i){lvls[i]+=ed;});}
    }
    active.forEach(function(c,i){
      c.daysAfter=lvls[i];
      c.send=Math.round((c.daysAfter-c.daysBefore)*c.rate);
      c.sendRaw=c.send;
    });
    var totFree=active.reduce(function(s,c){return s+c.send;},0);
    var diff=remaining-totFree;
    if(diff!==0&&active.length){
      var mi=active.reduce(function(b,c,i,a){return c.rate>a[b].rate?i:b;},0);
      active[mi].send+=diff;
      active[mi].sendRaw=active[mi].send;
    }
  }

  var all=locked.concat(active).concat(zero);

  // Apply box multiple
  if(boxMult&&boxMult>1){
    var totalRounded=0;
    all.forEach(function(c){
      var rounded=Math.floor(c.send/boxMult)*boxMult;
      c.sendRaw=c.send;
      c.send=rounded;
      c.boxes=rounded/boxMult;
      totalRounded+=rounded;
    });
    var remainder=totalQty-totalRounded;
    var sorted=all.filter(function(c){return c.rate>0;}).sort(function(a,b){
      var dA=a.rate>0?(a.stock+a.send)/a.rate:Infinity;
      var dB=b.rate>0?(b.stock+b.send)/b.rate:Infinity;
      return dA-dB;
    });
    for(var si=0;si<sorted.length;si++){
      while(remainder>=boxMult){
        sorted[si].send+=boxMult;
        sorted[si].boxes=Math.floor(sorted[si].send/boxMult);
        remainder-=boxMult;
      }
      if(remainder<=0) break;
    }
    if(remainder>0&&sorted.length){
      sorted[0].send+=remainder;
      sorted[0].boxes=Math.floor(sorted[0].send/boxMult);
      sorted[0].loose=sorted[0].send%boxMult;
    }
    all.forEach(function(c){
      if(c.rate>0) c.daysAfter=(c.stock+c.send)/c.rate;
    });
  } else {
    all.forEach(function(c){c.boxes=null;c.loose=0;});
  }

  return all;
}

export default function App(){
  var _s=useState("upload"),tab=_s[0],setTab=_s[1];
  var _f=useState(""),fileName=_f[0],setFileName=_f[1];
  var _f2=useState(""),file2Name=_f2[0],setFile2Name=_f2[1];
  var _l=useState(false),loading=_l[0],setLoading=_l[1];
  var _db=useState({}),db=_db[0],setDb=_db[1];
  var _bn=useState([]),brandNames=_bn[0],setBrandNames=_bn[1];
  var _bm=useState({}),brandMeta=_bm[0],setBrandMeta=_bm[1];
  var _mm=useState({}),multMap=_mm[0],setMultMap=_mm[1];
  var _mi=useState(""),multInfo=_mi[0],setMultInfo=_mi[1];
  var _it=useState([]),items=_it[0],setItems=_it[1];
  var _si=useState(""),skuInput=_si[0],setSkuInput=_si[1];
  var _qi=useState(""),qtyInput=_qi[0],setQtyInput=_qi[1];
  var _mo=useState(""),multOverride=_mo[0],setMultOverride=_mo[1];
  var _sr=useState([]),searchResults=_sr[0],setSearchResults=_sr[1];
  var _eb=useState(null),expBrand=_eb[0],setExpBrand=_eb[1];
  var _pm=useState(null),previewMatch=_pm[0],setPreviewMatch=_pm[1];
  var _pc=useState([]),previewChannels=_pc[0],setPreviewChannels=_pc[1];
  var _lk=useState({}),lockDays=_lk[0],setLockDays=_lk[1];
  var fileRef=useRef();
  var file2Ref=useRef();

  var handleFile=function(e,isBase){
    var file=e.target.files?e.target.files[0]:null;if(!file)return;
    setLoading(true);
    if(!isBase) setFileName(file.name); else setFile2Name(file.name);
    var reader=new FileReader();
    reader.onload=function(ev){
      try{
        var wb=XLSX.read(ev.target.result,{type:"array"});
        if(isBase){
          var combined={};var found=0;
          wb.SheetNames.forEach(function(name){
            if(norm(name).includes("base")){
              var result=parseBaseData(wb.Sheets[name]);
              if(result){Object.assign(combined,result);found+=Object.keys(result).length;}
            }
          });
          setMultMap(combined);setMultInfo(found+" SKUs com múltiplo");
        } else {
          var aDb={},aM={},bN=[];
          wb.SheetNames.forEach(function(name){
            if(SKIP_TABS.some(function(t){return norm(name).includes(t);}))return;
            var ws=wb.Sheets[name];if(!ws["!ref"])return;
            var p=parseSheet(ws);
            if(!p.skuCol||p.data.length<2||(!p.stockCols.length&&!p.salesCols.length))return;
            aDb[name]=p.data;
            aM[name]={data:p.data,skuCol:p.skuCol,descCol:p.descCol,curvaCol:p.curvaCol,catCol:p.catCol,estTotalCol:p.estTotalCol,stockCols:p.stockCols,salesCols:p.salesCols,mapping:autoMap(p.salesCols,p.stockCols)};
            bN.push(name);
          });
          setDb(aDb);setBrandMeta(aM);setBrandNames(bN);
        }
      }catch(err){alert("Erro: "+err.message);}
      setLoading(false);
    };
    reader.readAsArrayBuffer(file);e.target.value="";
  };

  var getMultiple=function(sku){
    var mo=parseInt(multOverride);
    if(mo>0) return mo;
    var entry=multMap[norm(sku)];
    return entry?entry.mult:1;
  };

  var buildChannels=function(row,meta){
    var cdGroups={};
    meta.salesCols.forEach(function(sc){
      var cd=meta.mapping[sc]||sc;
      if(!cdGroups[cd]) cdGroups[cd]={cd:cd,salesCols:[],stock:pNum(row[cd]),sales15d:0};
      cdGroups[cd].salesCols.push(sc);
      cdGroups[cd].sales15d+=pNum(row[sc]);
    });
    return Object.values(cdGroups).map(function(g){
      var rate=g.sales15d/PERIOD;
      var days=rate>0?g.stock/rate:Infinity;
      return {cd:g.cd,salesCol:g.salesCols.join(" + "),stock:g.stock,sales15d:g.sales15d,rate:rate,daysBefore:days};
    });
  };

  var searchSku=function(term){
    if(!term||term.length<2){setSearchResults([]);return;}
    var nt=norm(term);var found=[];
    brandNames.forEach(function(brand){
      var m=brandMeta[brand];
      (db[brand]||[]).forEach(function(row){
        var sku=String(row[m.skuCol]||""),desc=m.descCol?String(row[m.descCol]||""):"";
        if(norm(sku).includes(nt)||norm(desc).includes(nt)){
          var mult=multMap[norm(sku)];
          found.push({brand:brand,sku:sku,desc:desc,curva:m.curvaCol?row[m.curvaCol]:"",mult:mult?mult.mult:1,row:row,meta:m});
        }
      });
    });
    setSearchResults(found.slice(0,15));
  };

  var selectSku=function(r){
    setSkuInput(r.sku);
    setSearchResults([]);
    setMultOverride("");
    setPreviewMatch(r);
    var chs=buildChannels(r.row,r.meta);
    setPreviewChannels(chs);
    setLockDays({});
  };

  var doAdd=function(){
    if(!skuInput||!qtyInput)return;
    var totalQty=parseFloat(qtyInput)||0;if(!totalQty)return;
    var match=previewMatch;
    if(!match||norm(match.sku)!==norm(skuInput)){
      brandNames.forEach(function(brand){
        var m=brandMeta[brand];
        (db[brand]||[]).forEach(function(row){
          if(!match&&norm(String(row[m.skuCol]))===norm(skuInput))
            match={brand:brand,sku:String(row[m.skuCol]),desc:m.descCol?row[m.descCol]:"",curva:m.curvaCol?row[m.curvaCol]:"",row:row,meta:m};
        });
      });
    }
    if(!match){alert("SKU não encontrado.");return;}
    var boxMult=getMultiple(match.sku);
    var chs=buildChannels(match.row,match.meta);
    var result=waterFillWithLocks(chs,totalQty,boxMult,lockDays);
    setItems(function(p){return [{id:Date.now(),brand:match.brand,sku:match.sku,desc:match.desc,curva:match.curva,totalQty:totalQty,boxMult:boxMult,lockDays:Object.assign({},lockDays),channels:result}].concat(p);});
    setSkuInput("");setQtyInput("");setMultOverride("");setSearchResults([]);setPreviewMatch(null);setPreviewChannels([]);setLockDays({});
  };

  var removeItem=function(id){setItems(function(p){return p.filter(function(i){return i.id!==id;});});};

  var exportCSV=function(){
    var csv="Marca;SKU;Descrição;Curva;Qtd Total;Múltiplo;CD Destino;Canal;Est.Atual;Vendas 15d;Dias Antes;Travado;Enviar;Caixas;Avulsas;Dias Depois\n";
    items.forEach(function(r){r.channels.forEach(function(c){
      csv+='"'+r.brand+'";'+r.sku+';"'+r.desc+'";'+r.curva+";"+r.totalQty+";"+r.boxMult+';"'+c.cd+'";"'+c.salesCol+'";'+c.stock+";"+c.sales15d+";"+(c.daysBefore===Infinity?"":c.daysBefore.toFixed(1))+";"+(c.isLocked?c.targetDays+"d":"")+";"+c.send+";"+(c.boxes!=null?c.boxes:"")+";"+(c.loose||"")+";"+(c.daysAfter===Infinity?"":c.daysAfter.toFixed(1))+"\n";
    });});
    var blob=new Blob(["\uFEFF"+csv],{type:"text/csv;charset=utf-8;"});
    var url=URL.createObjectURL(blob);var a=document.createElement("a");a.href=url;a.download="divisao_nivelada.csv";a.click();URL.revokeObjectURL(url);
  };

  var S={
    app:{fontFamily:"system-ui,-apple-system,sans-serif",maxWidth:1100,margin:"0 auto",padding:16,color:"#1a1a2e"},
    hdr:{background:"linear-gradient(135deg,#0f0c29,#302b63,#24243e)",borderRadius:14,padding:"24px 28px",marginBottom:20,color:"#fff"},
    tabs:{display:"flex",gap:4,marginBottom:16,background:"#f0f0f5",borderRadius:10,padding:4},
    tab:function(a){return{flex:1,padding:"10px 14px",border:"none",borderRadius:8,cursor:"pointer",fontWeight:600,fontSize:13,background:a?"#fff":"transparent",color:a?"#302b63":"#666",boxShadow:a?"0 1px 4px rgba(0,0,0,.1)":"none"};},
    card:{background:"#fff",borderRadius:12,padding:20,marginBottom:14,border:"1px solid #e8e8f0",boxShadow:"0 1px 4px rgba(0,0,0,.04)"},
    btn:function(c){c=c||"#302b63";return{background:c,color:"#fff",border:"none",borderRadius:8,padding:"8px 18px",cursor:"pointer",fontWeight:600,fontSize:13,whiteSpace:"nowrap"};},
    btnSm:function(c){c=c||"#302b63";return{background:"transparent",color:c,border:"1px solid "+c,borderRadius:6,padding:"4px 12px",cursor:"pointer",fontSize:11,fontWeight:600};},
    inp:{border:"1px solid #ddd",borderRadius:8,padding:"10px 14px",fontSize:14,outline:"none",width:"100%",boxSizing:"border-box"},
    inpSm:{border:"1px solid #ddd",borderRadius:6,padding:"6px 8px",fontSize:13,outline:"none",width:60,textAlign:"center",boxSizing:"border-box"},
    badge:function(c){return{display:"inline-block",padding:"2px 10px",borderRadius:12,fontSize:11,fontWeight:600,background:c+"18",color:c};},
    chip:function(c){return{display:"inline-flex",alignItems:"center",gap:6,padding:"4px 12px",borderRadius:14,fontSize:12,fontWeight:600,background:c+"12",color:c,margin:3};},
    tbl:{width:"100%",borderCollapse:"collapse",fontSize:12},
    th:{textAlign:"left",padding:"8px 6px",borderBottom:"2px solid #e8e8f0",fontSize:10,fontWeight:700,color:"#555",textTransform:"uppercase"},
    td:{padding:"7px 6px",borderBottom:"1px solid #f0f0f5"},
    upload:{border:"3px dashed #ccc",borderRadius:16,padding:40,textAlign:"center",cursor:"pointer"},
    dd:{position:"absolute",top:"100%",left:0,right:0,background:"#fff",border:"1px solid #ddd",borderRadius:10,boxShadow:"0 8px 24px rgba(0,0,0,.12)",zIndex:10,maxHeight:280,overflow:"auto"},
    di:{padding:"10px 14px",cursor:"pointer",borderBottom:"1px solid #f5f5f5",display:"flex",justifyContent:"space-between",alignItems:"center",fontSize:13},
    lockBg:{background:"#fdf2f8",border:"1px solid #f0c6e0",borderRadius:10,padding:14,marginTop:12},
  };
  var cColor=function(c){return({"AA":"#e74c3c","A":"#e67e22","B":"#f1c40f","C":"#3498db"})[c]||"#888";};
  var dColor=function(d){if(d===Infinity)return"#888";if(d<7)return"#e74c3c";if(d<15)return"#e67e22";if(d<30)return"#f1c40f";return"#2ecc71";};
  var fD=function(d){return d===Infinity?"—":d.toFixed(1);};

  return(
    <div style={S.app}>
      <div style={S.hdr}>
        <h1 style={{margin:0,fontSize:20,fontWeight:700}}>📦 Divisão por CD — Nivelamento + Múltiplo + Trava de Dias</h1>
        <p style={{margin:"6px 0 0",opacity:.8,fontSize:13}}>Nivela dias de estoque · Respeita múltiplo/caixa · Permite travar dias em CDs específicos</p>
      </div>

      <div style={S.tabs}>
        {[["upload","📁 Planilhas"],["config","⚙️ Mapeamento"],["divide","➗ Dividir"]].map(function(x){
          return <button key={x[0]} style={S.tab(tab===x[0])} onClick={function(){setTab(x[0]);}}>{x[1]}</button>;
        })}
      </div>

      {tab==="upload"&&(
        <div>
          <div style={S.card}>
            <h3 style={{margin:"0 0 10px",fontSize:16}}>1. Planilha de Estoque & Vendas</h3>
            <p style={{fontSize:12,color:"#888",margin:"0 0 12px"}}>Excel com abas por marca — Linha 1 = grupo, Linha 2 = colunas</p>
            <div style={S.upload} onClick={function(){fileRef.current&&fileRef.current.click();}}>
              <input ref={fileRef} type="file" accept=".xlsx,.xls" onChange={function(e){handleFile(e,false);}} hidden/>
              {fileName?(
                <div>
                  <span style={{fontSize:30}}>✅</span>
                  <p style={{fontSize:14,fontWeight:700,color:"#302b63",margin:"8px 0 0"}}>{fileName}</p>
                  <p style={{fontSize:12,color:"#2ecc71"}}>{brandNames.length} marca(s): {brandNames.join(", ")}</p>
                </div>
              ):(
                <div>
                  <span style={{fontSize:40}}>📊</span>
                  <p style={{fontSize:15,fontWeight:600,color:"#302b63",margin:"8px 0 0"}}>Arraste ou clique</p>
                </div>
              )}
            </div>
          </div>
          <div style={S.card}>
            <h3 style={{margin:"0 0 10px",fontSize:16}}>2. Planilha de Múltiplos (opcional)</h3>
            <p style={{fontSize:12,color:"#888",margin:"0 0 12px"}}>Excel com abas "Base de Dados" contendo SKU e múltiplo por caixa</p>
            <div style={S.upload} onClick={function(){file2Ref.current&&file2Ref.current.click();}}>
              <input ref={file2Ref} type="file" accept=".xlsx,.xls" onChange={function(e){handleFile(e,true);}} hidden/>
              {file2Name?(
                <div>
                  <span style={{fontSize:30}}>✅</span>
                  <p style={{fontSize:14,fontWeight:700,color:"#302b63",margin:"8px 0 0"}}>{file2Name}</p>
                  <p style={{fontSize:12,color:"#2ecc71"}}>{multInfo}</p>
                </div>
              ):(
                <div>
                  <span style={{fontSize:40}}>📦</span>
                  <p style={{fontSize:15,fontWeight:600,color:"#302b63",margin:"8px 0 0"}}>Arraste ou clique</p>
                </div>
              )}
            </div>
          </div>
          {brandNames.length>0&&(
            <div style={{textAlign:"center",marginTop:8}}>
              <button onClick={function(){setTab("config");}} style={Object.assign({},S.btn(),{padding:"12px 32px",fontSize:15})}>Conferir Mapeamento →</button>
            </div>
          )}
        </div>
      )}

      {tab==="config"&&(
        <div>
          {!brandNames.length?(
            <div style={Object.assign({},S.card,{textAlign:"center",padding:40,color:"#888"})}>
              <p>Faça o upload primeiro.</p><button onClick={function(){setTab("upload");}} style={S.btn()}>Upload</button>
            </div>
          ):brandNames.map(function(name){
            var m=brandMeta[name];if(!m)return null;
            var isExp=expBrand===name,ok=m.salesCols.every(function(sc){return m.mapping[sc];});
            return(
              <div key={name} style={Object.assign({},S.card,{border:ok?"1px solid #e8e8f0":"2px solid #e67e22"})}>
                <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",cursor:"pointer"}} onClick={function(){setExpBrand(isExp?null:name);}}>
                  <div style={{display:"flex",gap:10,alignItems:"center"}}>
                    <strong style={{fontSize:16}}>{name}</strong>
                    <span style={S.badge("#3498db")}>{m.stockCols.length} CDs</span>
                    <span style={S.badge("#e67e22")}>{m.salesCols.length} canais</span>
                    {ok?<span style={S.badge("#2ecc71")}>✓</span>:<span style={S.badge("#e74c3c")}>⚠</span>}
                  </div>
                  <span style={{fontSize:16,color:"#888"}}>{isExp?"▲":"▼"}</span>
                </div>
                {isExp&&(
                  <div style={{marginTop:14}}>
                    <p style={{fontSize:12,color:"#888",margin:"0 0 10px"}}>Canal de venda → CD de estoque:</p>
                    {m.salesCols.map(function(sc){
                      return(
                        <div key={sc} style={{display:"flex",alignItems:"center",gap:8,padding:"6px 0",borderBottom:"1px solid #f5f5f5"}}>
                          <span style={Object.assign({},S.chip("#e67e22"),{flex:1,minWidth:0,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"})}>{sc}</span>
                          <span style={{fontSize:18,color:"#e67e22",fontWeight:800}}>→</span>
                          <select value={m.mapping[sc]||""} onChange={function(e){setBrandMeta(function(p){var u=Object.assign({},p);var b=Object.assign({},u[name]);var mp=Object.assign({},b.mapping);mp[sc]=e.target.value;b.mapping=mp;u[name]=b;return u;});}}
                            style={{border:"1px solid #ddd",borderRadius:8,padding:"6px 10px",fontSize:12,background:m.mapping[sc]?"#f0fff4":"#fff8e1",flex:1}}>
                            <option value="">Selecione...</option>
                            {m.stockCols.map(function(stk){return <option key={stk} value={stk}>{stk}</option>;})}
                          </select>
                        </div>
                      );
                    })}
                  </div>
                )}
              </div>
            );
          })}
          {brandNames.length>0&&(
            <div style={{textAlign:"center",marginTop:8}}>
              <button onClick={function(){setTab("divide");}} style={Object.assign({},S.btn(),{padding:"12px 32px",fontSize:15})}>Ir para Dividir →</button>
            </div>
          )}
        </div>
      )}

      {tab==="divide"&&(
        <div>
          {!brandNames.length?(
            <div style={Object.assign({},S.card,{textAlign:"center",padding:40,color:"#888"})}>
              <p>Faça o upload primeiro.</p><button onClick={function(){setTab("upload");}} style={S.btn()}>Upload</button>
            </div>
          ):(
            <div>
              <div style={S.card}>
                <div style={{display:"flex",gap:10,alignItems:"flex-end",flexWrap:"wrap"}}>
                  <div style={{flex:2,position:"relative",minWidth:200}}>
                    <label style={{fontSize:11,fontWeight:700,color:"#555",marginBottom:4,display:"block"}}>SKU ou descrição</label>
                    <input placeholder="Digite SKU ou nome..." value={skuInput}
                      onChange={function(e){setSkuInput(e.target.value);searchSku(e.target.value);}}
                      onBlur={function(){setTimeout(function(){setSearchResults([]);},200);}}
                      style={S.inp} autoFocus/>
                    {searchResults.length>0&&(
                      <div style={S.dd}>
                        {searchResults.map(function(r,i){
                          return(
                            <div key={i} style={Object.assign({},S.di,{background:i%2?"#fafafe":"#fff"})}
                              onMouseDown={function(){selectSku(r);}}>
                              <div style={{minWidth:0,overflow:"hidden"}}>
                                <strong style={{color:"#302b63"}}>{r.sku}</strong>
                                <span style={{marginLeft:8,fontSize:11,color:"#888"}}>{(r.desc||"").substring(0,35)}</span>
                              </div>
                              <div style={{display:"flex",gap:5,alignItems:"center",flexShrink:0}}>
                                {r.mult>1&&<span style={S.badge("#9b59b6")}>cx {r.mult}</span>}
                                {r.curva&&<span style={S.badge(cColor(r.curva))}>{r.curva}</span>}
                                <span style={S.badge("#302b63")}>{r.brand}</span>
                              </div>
                            </div>
                          );
                        })}
                      </div>
                    )}
                  </div>
                  <div style={{minWidth:120}}>
                    <label style={{fontSize:11,fontWeight:700,color:"#555",marginBottom:4,display:"block"}}>Quantidade</label>
                    <input placeholder="Ex: 500" type="number" value={qtyInput} onChange={function(e){setQtyInput(e.target.value);}}
                      onKeyDown={function(e){if(e.key==="Enter")doAdd();}} style={S.inp}/>
                  </div>
                  <div style={{minWidth:80}}>
                    <label style={{fontSize:11,fontWeight:700,color:"#555",marginBottom:4,display:"block"}}>Múlt./cx</label>
                    <input placeholder="Auto" type="number" value={multOverride} onChange={function(e){setMultOverride(e.target.value);}} style={Object.assign({},S.inp,{background:multOverride?"#f5f0ff":"#fff"})}/>
                  </div>
                  <button onClick={doAdd} style={Object.assign({},S.btn(),{padding:"10px 24px",height:46})}>+ Dividir</button>
                </div>

                {previewChannels.length>0&&(
                  <div style={S.lockBg}>
                    <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:10}}>
                      <div>
                        <strong style={{fontSize:13,color:"#9b2c6d"}}>🔒 Travar dias de estoque por CD</strong>
                        <p style={{margin:"2px 0 0",fontSize:11,color:"#888"}}>Defina quantos dias quer deixar num CD específico. O restante será nivelado entre os demais.</p>
                      </div>
                    </div>
                    <table style={Object.assign({},S.tbl,{fontSize:13})}>
                      <thead><tr>
                        <th style={S.th}>CD</th>
                        <th style={S.th}>Est. Atual</th>
                        <th style={S.th}>Venda 15d</th>
                        <th style={S.th}>Dias Atuais</th>
                        <th style={Object.assign({},S.th,{textAlign:"center"})}>🔒 Travar em (dias)</th>
                      </tr></thead>
                      <tbody>
                        {previewChannels.map(function(c,j){
                          return(
                            <tr key={j} style={{background:j%2?"#fdf2f8":"#fff"}}>
                              <td style={Object.assign({},S.td,{fontWeight:700,fontSize:12})}>{c.cd}</td>
                              <td style={S.td}>{c.stock.toLocaleString("pt-BR")}</td>
                              <td style={S.td}>{c.sales15d.toLocaleString("pt-BR")}</td>
                              <td style={S.td}>
                                <span style={{fontWeight:700,color:dColor(c.daysBefore)}}>{fD(c.daysBefore)}</span>
                                <span style={{fontSize:9,color:"#aaa"}}> dias</span>
                              </td>
                              <td style={Object.assign({},S.td,{textAlign:"center"})}>
                                <input type="number" placeholder="—" value={lockDays[c.cd]||""}
                                  onChange={function(e){setLockDays(function(p){var u=Object.assign({},p);if(e.target.value)u[c.cd]=e.target.value;else delete u[c.cd];return u;});}}
                                  style={Object.assign({},S.inpSm,{background:lockDays[c.cd]?"#fce4ec":"#fff",border:lockDays[c.cd]?"2px solid #e91e63":"1px solid #ddd"})}/>
                              </td>
                            </tr>
                          );
                        })}
                      </tbody>
                    </table>
                  </div>
                )}

                <div style={{marginTop:10,padding:"8px 12px",background:"#f8f8fc",borderRadius:8,fontSize:11,color:"#666"}}>
                  <b>Lógica:</b> CDs travados recebem exatamente o necessário para os dias definidos → o restante é nivelado entre os demais (water-fill) → arredonda ao múltiplo de caixa.
                  {Object.keys(multMap).length>0&&<span style={{marginLeft:8,color:"#9b59b6"}}> 📦 {Object.keys(multMap).length} SKUs com múltiplo</span>}
                </div>
              </div>

              {items.length>0&&(
                <div style={{display:"flex",justifyContent:"space-between",alignItems:"center",marginBottom:10}}>
                  <span style={{fontSize:13,color:"#888"}}>{items.length} divisão(ões)</span>
                  <div style={{display:"flex",gap:8}}>
                    <button onClick={function(){setItems([]);}} style={S.btnSm("#e74c3c")}>Limpar</button>
                    <button onClick={exportCSV} style={S.btn("#2ecc71")}>⬇ Exportar CSV</button>
                  </div>
                </div>
              )}

              {!items.length&&!previewChannels.length&&(
                <div style={Object.assign({},S.card,{textAlign:"center",padding:30,color:"#aaa"})}>
                  <p style={{fontSize:14}}>Digite um SKU, informe a quantidade e clique em Dividir.</p>
                </div>
              )}

              {items.map(function(item){
                var maxD=Math.max.apply(null,item.channels.filter(function(c){return c.daysAfter!==Infinity;}).map(function(c){return c.daysAfter;}).concat([1]));
                return(
                <div key={item.id} style={S.card}>
                  <div style={{display:"flex",justifyContent:"space-between",alignItems:"flex-start",marginBottom:12}}>
                    <div>
                      <div style={{display:"flex",gap:8,alignItems:"center",marginBottom:4}}>
                        <strong style={{fontSize:17}}>{item.sku}</strong>
                        <span style={S.badge("#302b63")}>{item.brand}</span>
                        {item.curva&&<span style={S.badge(cColor(item.curva))}>{item.curva}</span>}
                        {item.boxMult>1&&<span style={S.badge("#9b59b6")}>📦 {item.boxMult} un/cx</span>}
                      </div>
                      <p style={{margin:0,fontSize:12,color:"#666"}}>{item.desc}</p>
                    </div>
                    <div style={{textAlign:"right"}}>
                      <div style={{fontSize:13}}>Dividindo <b style={{fontSize:20,color:"#302b63"}}>{item.totalQty.toLocaleString("pt-BR")}</b> un.</div>
                      {item.boxMult>1&&<div style={{fontSize:11,color:"#9b59b6"}}>{Math.floor(item.totalQty/item.boxMult)} cx + {item.totalQty%item.boxMult} avulsas</div>}
                      <button onClick={function(){removeItem(item.id);}} style={Object.assign({},S.btnSm("#e74c3c"),{marginTop:6})}>Remover</button>
                    </div>
                  </div>

                  <table style={S.tbl}>
                    <thead><tr>
                      <th style={S.th}>CD Destino</th>
                      <th style={S.th}>Canal</th>
                      <th style={S.th}>Est.Atual</th>
                      <th style={S.th}>Venda 15d</th>
                      <th style={S.th}>Dias Antes</th>
                      <th style={Object.assign({},S.th,{textAlign:"center"})}>Enviar</th>
                      {item.boxMult>1&&<th style={Object.assign({},S.th,{textAlign:"center"})}>Caixas</th>}
                      <th style={S.th}>Dias Depois</th>
                      <th style={Object.assign({},S.th,{width:100})}>Nível</th>
                    </tr></thead>
                    <tbody>
                      {item.channels.map(function(c,j){
                        return(
                          <tr key={j} style={{background:c.isLocked?"#fdf2f8":j%2?"#fafafe":"#fff"}}>
                            <td style={Object.assign({},S.td,{fontWeight:700,fontSize:11})}>
                              {c.isLocked&&<span style={{color:"#e91e63",marginRight:4}}>🔒</span>}
                              {c.cd}
                            </td>
                            <td style={Object.assign({},S.td,{fontSize:10,color:"#888",maxWidth:140,overflow:"hidden",textOverflow:"ellipsis",whiteSpace:"nowrap"})} title={c.salesCol}>{c.salesCol}</td>
                            <td style={S.td}>{c.stock.toLocaleString("pt-BR")}</td>
                            <td style={S.td}>{c.sales15d.toLocaleString("pt-BR")}</td>
                            <td style={S.td}><span style={{fontWeight:700,color:dColor(c.daysBefore)}}>{fD(c.daysBefore)}</span><span style={{fontSize:9,color:"#aaa"}}> d</span></td>
                            <td style={Object.assign({},S.td,{textAlign:"center"})}>
                              {c.send>0?(
                                <div>
                                  <span style={{fontSize:18,fontWeight:800,color:c.isLocked?"#e91e63":"#302b63"}}>{c.send.toLocaleString("pt-BR")}</span>
                                  {c.loose>0&&<div style={{fontSize:9,color:"#9b59b6"}}>{c.loose} avulsa(s)</div>}
                                </div>
                              ):<span style={{color:"#ccc"}}>—</span>}
                            </td>
                            {item.boxMult>1&&(
                              <td style={Object.assign({},S.td,{textAlign:"center",fontSize:13,fontWeight:600,color:"#9b59b6"})}>
                                {c.send>0?(c.boxes!=null?c.boxes:"—"):"—"}
                                {c.loose>0&&<span style={{fontSize:9,color:"#888"}}> +{c.loose}</span>}
                              </td>
                            )}
                            <td style={S.td}>
                              <span style={{fontWeight:700,color:dColor(c.daysAfter)}}>{fD(c.daysAfter)}</span>
                              <span style={{fontSize:9,color:"#aaa"}}> d</span>
                              {c.isLocked&&<span style={{fontSize:9,color:"#e91e63",marginLeft:4}}>({c.targetDays}d alvo)</span>}
                            </td>
                            <td style={Object.assign({},S.td,{width:100})}>
                              {c.daysAfter!==Infinity&&(
                                <div style={{position:"relative",height:14,background:"#eee",borderRadius:7,overflow:"hidden"}}>
                                  <div style={{position:"absolute",left:0,top:0,height:"100%",width:Math.min(100,(c.daysBefore/maxD)*100)+"%",background:"#ddd",borderRadius:7}}/>
                                  <div style={{position:"absolute",left:0,top:0,height:"100%",width:Math.min(100,(c.daysAfter/maxD)*100)+"%",background:c.isLocked?"#e91e63":dColor(c.daysAfter),borderRadius:7,opacity:.7}}/>
                                </div>
                              )}
                            </td>
                          </tr>
                        );
                      })}
                    </tbody>
                    <tfoot><tr>
                      <td colSpan={4} style={Object.assign({},S.td,{textAlign:"right",fontWeight:700,color:"#555",borderTop:"2px solid #e8e8f0"})}>Total</td>
                      <td style={Object.assign({},S.td,{borderTop:"2px solid #e8e8f0"})}/>
                      <td style={Object.assign({},S.td,{textAlign:"center",fontWeight:800,fontSize:16,color:"#302b63",borderTop:"2px solid #e8e8f0"})}>
                        {item.channels.reduce(function(s,c){return s+c.send;},0).toLocaleString("pt-BR")}
                      </td>
                      {item.boxMult>1&&<td style={Object.assign({},S.td,{textAlign:"center",fontWeight:700,color:"#9b59b6",borderTop:"2px solid #e8e8f0"})}>
                        {item.channels.reduce(function(s,c){return s+(c.boxes||0);},0)}
                      </td>}
                      <td colSpan={2} style={Object.assign({},S.td,{borderTop:"2px solid #e8e8f0"})}/>
                    </tr></tfoot>
                  </table>

                  {item.channels.some(function(c){return c.sales15d<=0;})&&(
                    <div style={{marginTop:8,background:"#fff3cd",color:"#856404",padding:"6px 12px",borderRadius:8,fontSize:11}}>
                      ⚠️ Canais sem venda em 15d não recebem estoque.
                    </div>
                  )}
                </div>
              );})}
            </div>
          )}
        </div>
      )}
    </div>
  );
}
