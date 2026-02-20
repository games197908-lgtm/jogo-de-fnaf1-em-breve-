import React, { useState, useEffect, useRef } from 'react';
import { initializeApp, getApps } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  doc, 
  deleteDoc, 
  setDoc, 
  query,
  orderBy,
  limit
} from 'firebase/firestore';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  Lock, Trash2, CheckCircle, Megaphone, Camera, X, MessageSquare, ArrowRight, RefreshCw
} from 'lucide-react';

export default function App() {
  const [user, setUser] = useState(null);
  const [isReady, setIsReady] = useState(false);
  const [view, setView] = useState('public'); 
  const [loading, setLoading] = useState(false);
  const [sent, setSent] = useState(false);
  const [generatedCode, setGeneratedCode] = useState('');
  
  const [suggestions, setSuggestions] = useState([]); 
  const [systemNotice, setSystemNotice] = useState({ active: false, text: '' });
  
  const [name, setName] = useState('');
  const [suggestion, setSuggestion] = useState('');
  const [selectedImage, setSelectedImage] = useState(null);
  const [adminPass, setAdminPass] = useState('');
  const [newNoticeText, setNewNoticeText] = useState('');

  const fileInputRef = useRef(null);
  const ADMIN_PASSWORD = "admin";

  // Inicialização Segura
  useEffect(() => {
    let isMounted = true;
    
    const init = async () => {
      try {
        if (typeof __firebase_config === 'undefined') return;
        const firebaseConfig = JSON.parse(__firebase_config);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'sugestoes-canal-v2';

        const app = getApps().length === 0 ? initializeApp(firebaseConfig) : getApps()[0];
        const auth = getAuth(app);
        const db = getFirestore(app);

        // Tenta autenticar
        await signInAnonymously(auth);

        onAuthStateChanged(auth, (u) => {
          if (u && isMounted) {
            setUser(u);
            // Listeners
            onSnapshot(doc(db, 'artifacts', appId, 'public', 'settings', 'notice'), (s) => {
              if (s.exists()) setSystemNotice(s.data());
            });

            const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'suggestions'), orderBy('timestamp', 'desc'), limit(50));
            onSnapshot(q, (s) => {
              setSuggestions(s.docs.map(d => ({ id: d.id, ...d.data() })));
            });

            setIsReady(true);
          }
        });
      } catch (err) {
        console.error(err);
        // Se der erro, tenta libertar a tela após 5 segundos mesmo assim
        setTimeout(() => setIsReady(true), 5000);
      }
    };

    init();
    return () => { isMounted = false; };
  }, []);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!name.trim() || !suggestion.trim() || loading) return;
    setLoading(true);
    const code = Math.random().toString(36).substring(2, 7).toUpperCase();

    try {
      const db = getFirestore();
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'sugestoes-canal-v2';
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'suggestions'), {
        name: name.trim(),
        text: suggestion.trim(),
        code: code,
        photoUrl: selectedImage || `https://ui-avatars.com/api/?name=${encodeURIComponent(name)}&background=333&color=fff`,
        timestamp: Date.now()
      });
      setGeneratedCode(code);
      setSent(true);
      setName(''); setSuggestion(''); setSelectedImage(null);
    } catch (err) {
      alert("Erro ao enviar. Verifica a internet.");
    } finally {
      setLoading(false);
    }
  };

  // Tela de Carregamento com Botão de Emergência
  if (!isReady) return (
    <div className="fixed inset-0 bg-black flex flex-col items-center justify-center p-6 space-y-6">
      <div className="w-10 h-10 border-2 border-white/10 border-t-white rounded-full animate-spin"></div>
      <div className="text-center space-y-2">
        <p className="text-zinc-500 text-[10px] font-black uppercase tracking-widest">A ligar ao servidor...</p>
        <button 
          onClick={() => setIsReady(true)}
          className="text-white/20 text-[9px] uppercase font-bold underline"
        >
          Demorando muito? Carrega aqui
        </button>
      </div>
    </div>
  );

  return (
    <div className="min-h-screen bg-black text-white font-sans selection:bg-zinc-800 antialiased">
      
      {/* Aviso Global */}
      {systemNotice.active && view === 'public' && !sent && (
        <div className="bg-zinc-900 border-b border-zinc-800 px-6 py-4 flex items-center justify-center gap-3">
          <Megaphone size={14} className="text-zinc-400" />
          <p className="text-[10px] font-bold uppercase tracking-tight text-center">
            {systemNotice.text}
          </p>
        </div>
      )}

      <div className="max-w-md mx-auto px-6 py-12">
        
        {view === 'public' && !sent && (
          <div className="space-y-10 animate-in fade-in duration-500">
            <header className="text-center space-y-2">
              <h1 className="text-4xl font-black italic tracking-tighter uppercase leading-none">Sugestões</h1>
              <p className="text-[10px] uppercase tracking-[0.3em] text-zinc-600 font-bold">Ideias para o Canal</p>
            </header>

            <form onSubmit={handleSubmit} className="space-y-6">
              {/* Avatar */}
              <div className="flex flex-col items-center gap-3">
                <div 
                  onClick={() => fileInputRef.current.click()}
                  className="w-24 h-24 rounded-full bg-zinc-900 border-2 border-zinc-800 flex items-center justify-center overflow-hidden cursor-pointer"
                >
                  {selectedImage ? (
                    <img src={selectedImage} className="w-full h-full object-cover" />
                  ) : (
                    <Camera className="text-zinc-700" size={24} />
                  )}
                </div>
                <p className="text-[9px] font-black text-zinc-700 uppercase tracking-widest italic">Foto (Opcional)</p>
                <input type="file" ref={fileInputRef} className="hidden" accept="image/*" onChange={(e) => {
                  const file = e.target.files[0];
                  if (file) {
                    const reader = new FileReader();
                    reader.onload = () => setSelectedImage(reader.result);
                    reader.readAsDataURL(file);
                  }
                }} />
              </div>

              <div className="space-y-4">
                <div className="bg-zinc-900/50 rounded-2xl p-4 border border-zinc-800">
                  <input 
                    type="text" placeholder="Teu Nome" required value={name} onChange={e => setName(e.target.value)}
                    className="w-full bg-transparent outline-none font-bold text-lg placeholder:text-zinc-800" 
                  />
                </div>
                
                <div className="bg-zinc-900/50 rounded-2xl p-4 border border-zinc-800">
                  <textarea 
                    placeholder="Escreve a tua ideia aqui..." required value={suggestion} onChange={e => setSuggestion(e.target.value)}
                    className="w-full bg-transparent outline-none h-32 resize-none text-zinc-300 placeholder:text-zinc-800" 
                  />
                </div>

                <button 
                  disabled={loading}
                  className="w-full bg-zinc-100 text-black font-black py-5 rounded-2xl text-xs uppercase tracking-widest active:scale-[0.98] transition-all disabled:opacity-50 flex items-center justify-center gap-2"
                >
                  {loading ? <RefreshCw className="animate-spin" size={14}/> : "ENVIAR SUGESTÃO"}
                </button>
              </div>
            </form>

            <div className="pt-10 border-t border-zinc-900">
               <p className="text-[10px] font-black text-zinc-800 uppercase text-center mb-4 tracking-widest">Já tens um código ativo?</p>
               <div className="flex gap-2">
                  <div className="flex-1 bg-zinc-900/50 border border-zinc-800 rounded-xl p-3 text-center text-zinc-700 font-mono text-xs">CÓDIGO</div>
                  <button className="bg-white text-black px-6 rounded-xl text-[10px] font-black uppercase">Entrar</button>
               </div>
            </div>

            <button onClick={() => setView('login')} className="w-full text-zinc-900 text-[9px] font-black uppercase tracking-widest py-4">
              <Lock size={10} className="inline mr-1"/> Admin Painel
            </button>
          </div>
        )}

        {sent && (
          <div className="py-20 text-center space-y-8 animate-in zoom-in-95">
            <CheckCircle className="mx-auto text-white" size={48} />
            <div className="space-y-2">
              <h2 className="text-3xl font-black italic uppercase leading-none">Recebido!</h2>
              <p className="text-zinc-600 text-[10px] uppercase font-bold tracking-widest">A Nicole já recebeu a tua ideia.</p>
            </div>
            <div className="bg-zinc-900 border border-zinc-800 p-8 rounded-3xl inline-block px-12">
              <p className="text-[8px] font-black text-zinc-700 uppercase mb-3 tracking-widest">ID DA SUGESTÃO</p>
              <span className="text-4xl font-mono font-black tracking-widest text-white">{generatedCode}</span>
            </div>
            <button onClick={() => setSent(false)} className="block w-full text-zinc-600 text-[10px] font-black uppercase tracking-widest mt-12 underline">Voltar</button>
          </div>
        )}

        {view === 'login' && (
          <div className="py-20 text-center space-y-6">
            <h2 className="text-xl font-black italic uppercase">Restrito</h2>
            <input 
              type="password" placeholder="Senha" autoFocus
              className="w-full p-4 bg-zinc-900 border border-zinc-800 rounded-2xl text-center outline-none focus:border-white font-bold"
              onChange={e => setAdminPass(e.target.value)} 
            />
            <button 
              onClick={() => adminPass === ADMIN_PASSWORD ? setView('admin') : alert("Incorreto")}
              className="w-full bg-white text-black font-black py-4 rounded-2xl text-[10px] uppercase tracking-widest"
            >
              Entrar
            </button>
            <button onClick={() => setView('public')} className="text-zinc-600 text-[10px] uppercase font-bold block mx-auto">Cancelar</button>
          </div>
        )}

        {view === 'admin' && (
          <div className="space-y-10 pb-20">
            <header className="flex justify-between items-center">
              <h2 className="text-2xl font-black italic uppercase tracking-tighter">Mural Admin</h2>
              <button onClick={() => setView('public')} className="p-3 bg-zinc-900 rounded-xl"><X size={18}/></button>
            </header>

            <div className="bg-zinc-900 p-6 rounded-3xl border border-zinc-800 space-y-4">
              <p className="text-[10px] font-black uppercase text-zinc-500 italic">Aviso para Seguidores</p>
              <textarea 
                value={newNoticeText} onChange={e => setNewNoticeText(e.target.value)}
                className="w-full bg-black border border-zinc-800 p-4 rounded-xl text-xs font-bold outline-none h-24"
              />
              <button 
                onClick={async () => {
                   const db = getFirestore();
                   const appId = typeof __app_id !== 'undefined' ? __app_id : 'sugestoes-canal-v2';
                   await setDoc(doc(db, 'artifacts', appId, 'public', 'settings', 'notice'), {
                     active: !systemNotice.active,
                     text: newNoticeText
                   });
                }} 
                className={`w-full py-4 rounded-xl font-black text-[10px] uppercase tracking-widest ${systemNotice.active ? 'bg-red-900/30 text-red-500' : 'bg-white text-black'}`}
              >
                {systemNotice.active ? 'Desativar Aviso' : 'Ativar Aviso'}
              </button>
            </div>

            <div className="space-y-4">
              {suggestions.map(s => (
                <div key={s.id} className="p-5 bg-zinc-900/30 border border-zinc-800 rounded-3xl flex gap-4 items-start">
                  <img src={s.photoUrl} className="w-12 h-12 rounded-xl object-cover" />
                  <div className="flex-1">
                    <div className="flex items-center gap-2 mb-1">
                      <span className="font-black italic uppercase text-lg">{s.name}</span>
                      <span className="text-[8px] font-mono font-black text-zinc-600">{s.code}</span>
                    </div>
                    <p className="text-zinc-400 text-sm">{s.text}</p>
                  </div>
                  <button 
                    onClick={async () => {
                      if(confirm("Apagar?")) {
                        const appId = typeof __app_id !== 'undefined' ? __app_id : 'sugestoes-canal-v2';
                        await deleteDoc(doc(getFirestore(), 'artifacts', appId, 'public', 'data', 'suggestions', s.id));
                      }
                    }} 
                    className="text-zinc-800 hover:text-red-500 p-1"
                  >
                    <Trash2 size={16}/>
                  </button>
                </div>
              ))}
            </div>
          </div>
        )}

      </div>
    </div>
  );
}

