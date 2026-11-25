     var ids = {}; sorted.forEach(function(n){ ids[n.id]=1; });
    Object.keys(existing).forEach(function(id){ if(!ids[id]) host.removeChild(existing[id]); });
    sorted.forEach(function(n, i){
      var c = existing[n.id];
      if(c && c.parentNode===host){ c._set(n); }
      else { c = noteCard(n); }
      if(host.children[i]!==c) host.insertBefore(c, host.children[i]||null);
    });
    $("notesEmpty").classList.toggle("hidden", sorted.length>0);
  }
  $("newNote").addEventListener("click", function(){
    var n = { id:uid(), title:"", body:"", ts:Date.now(), created:Date.now() };
    saveNote(n);
    setTimeout(function(){ var c = document.querySelector('[data-id="'+n.id+'"] input'); if(c) c.focus(); }, 50);
  });

  /* ---------- Dados ---------- */
  function upsert(arr, item){
    var i = arr.findIndex(function(x){ return x.id===item.id; });
    if(i>-1) arr[i] = Object.assign({}, arr[i], item); else arr.push(item);
  }
  function saveEntry(item){
    upsert(state.entries, item); renderMoney();
    if(STORE.write()) setStatus("Salvo");
  }
  function removeEntry(id){
    state.entries = state.entries.filter(function(x){ return x.id!==id; }); renderMoney();
    if(STORE.write()) setStatus("Salvo");
  }
  function saveNote(item){
    var cur = state.notes.find(function(x){ return x.id===item.id; });
    if(cur && cur.created && !item.created) item.created = cur.created;
    upsert(state.notes, item);
    if(!cur) renderNotes();
    if(STORE.write()) setStatus("Salvo");
  }
  function removeNote(id){
    clearTimeout(noteTimers[id]);
    state.notes = state.notes.filter(function(x){ return x.id!==id; }); renderNotes();
    if(STORE.write()) setStatus("Salvo");
  }

  /* ---------- Backup: exportar / importar ---------- */
  $("exportBtn").addEventListener("click", function(){
    var payload = { app:"Meu Caderno", version:1, exportedAt:new Date().toISOString(), entries:state.entries, notes:state.notes };
    var blob = new Blob([JSON.stringify(payload, null, 2)], { type:"application/json" });
    var url = URL.createObjectURL(blob);
    var a = el("a", { href:url, download:"meu-caderno-"+todayStr()+".json" });
    document.body.appendChild(a); a.click(); document.body.removeChild(a);
    setTimeout(function(){ URL.revokeObjectURL(url); }, 1000);
    setStatus("Cópia baixada");
  });
  $("importBtn").addEventListener("click", function(){ $("importFile").click(); });
  $("importFile").addEventListener("change", function(e){
    var file = e.target.files && e.target.files[0];
    if(!file) return;
    var reader = new FileReader();
    reader.onload = function(){
      var data;
      try{ data = JSON.parse(reader.result); }
      catch(err){ setStatus("Arquivo inválido"); e.target.value = ""; return; }
      var entries = cleanList(data.entries, cleanEntry);
      var notes = cleanList(data.notes, cleanNote);
      if(!entries.length && !notes.length){ setStatus("Nada para importar"); e.target.value = ""; return; }
      if(window.confirm("Substituir os dados atuais pelos do arquivo?\n\nOK = substituir\nCancelar = mesclar")){
        state.entries = entries; state.notes = notes; setStatus("Dados restaurados");
      } else {
        entries.forEach(function(it){ upsert(state.entries, it); });
        notes.forEach(function(it){ upsert(state.notes, it); });
        setStatus("Dados mesclados");
      }
      renderMoney(); renderNotes(); STORE.write();
      e.target.value = "";
    };
    reader.onerror = function(){ setStatus("Erro ao ler arquivo"); e.target.value = ""; };
    reader.readAsText(file);
  });

  /* ---------- Inicialização ---------- */
  var loaded = STORE.read();
  state.entries = loaded.entries;
  state.notes = loaded.notes;
  state.ready = true;
  renderMoney(); renderNotes();
  if(state.storageOk) setStatus("Salvo localmente");
  else showStorageError();

  /* Sincroniza entre abas/janelas abertas com o mesmo arquivo */
  window.addEventListener("storage", function(e){
    if(e.key!==STORE_KEY || !e.newValue) return;
    try{
      var data = JSON.parse(e.newValue);
      state.entries = cleanList(data.entries, cleanEntry);
      state.notes = cleanList(data.notes, cleanNote);
      renderMoney(); renderNotes(); setStatus("Sincronizado");
    }catch(err){}
  });

  /* Instalável como app (PWA) quando servido por HTTPS, ex.: GitHub Pages */
  if("serviceWorker" in navigator && location.protocol!=="file:"){
    window.addEventListener("load", function(){
      navigator.serviceWorker.register("service-worker.js").catch(function(){});
    });
  }
})();
</script>
</body>
</html>
 
