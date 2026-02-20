import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  onSnapshot, 
  doc, 
  deleteDoc, 
  setDoc, 
  query,
  limit
} from 'firebase/firestore';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  Lock, Trash2, LogOut, Upload, Send, CheckCircle, User, Megaphone, Clock, MessageSquare, Copy 
} from 'lucide-react';

// --- CONFIGURAÇÃO FIREBASE ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'sugestoes-permanentes-v1';

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
  const [newNoticeText, setNewNoticeText] = useState('Estamos a ter muitas sugestões! Podemos demorar um pouco a responder no vídeo.');
  
  const ADMIN_PASSWORD = "admin";
  const fileInputRef = useRef(null);

  // 1. GESTÃO DE AUTENTICAÇÃO (Evita Tela Branca)
  useEffect(() => {
    const performAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Erro Auth:", err);
      }
    };

    performAuth();

    const unsubscribe = onAuthStateChanged(auth, (u) => {
      if (u) {
        setUser(u);
        setIsReady(true); // Só liberta a tela quando houver user
      }
    });

    return () => unsubscribe();
  }, []);

  // 2. GESTÃO DE DADOS (Firestore)
  useEffect(() => {
    if (!user || !isReady) return;

    // Aviso Global
    const noticeDoc = doc(db, 'artifacts', appId, 'public', 'settings', 'notice');
    const unsubNotice = onSnapshot(noticeDoc, (docSnap) => {
      if (docSnap.exists()) {
        setSystemNotice(docSnap.data());
      }
    }, (err) => console.log("Erro ao ler aviso:", err));

    // Sugestões
    const sugCol = collection(db, 'artifacts', appId, 'public', 'data', 'suggestions');
    const unsubSug = onSnapshot(sugCol, (snapshot) => {
      const docs = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      setSuggestions(docs.sort((a, b) => (b.timestamp || 0) - (a.timestamp || 0)));
    }, (err) => console.log("Erro ao ler sugestões:", err));

    return () => {
      unsubNotice();
      unsubSug();
    };
  }, [user, isReady]);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!name.trim() || !suggestion.trim() || loading || !user) return;

    setLoading(true);
    const code = Math.random().toString(36).substring(2, 7).toUpperCase();

    try {
      const colRef = collection(db, 'artifacts', appId, 'public', 'data', 'suggestions');
      await addDoc(colRef, {
        name: name.trim(),
        text: suggestion.trim(),
        code: code,
        photoUrl: selectedImage || `https://ui-avatars.com/api/?name=${encodeURIComponent(name)}&background=random&color=fff`,
        timestamp: Date.now(),
        creatorUid: user.uid
      });
      
      setGeneratedCode(code);
      setSent(true);
      setName(''); setSuggestion(''); setSelectedImage(null);
    } catch (err) {
      console.error("Erro ao enviar:", err);
    } finally {
      setLoading(false);
    }
  };

  const toggleNotice = async () => {
    if (!user) return;
    try {
      const noticeDoc = doc(db, 'artifacts', appId, 'public', 'settings', 'notice');
      await setDoc(noticeDoc, {
        active: !systemNotice.active,
        text: newNoticeText
      });
    } catch (err) {
      console.error("Erro ao alternar aviso:", err);
    }
  };

  // TELA DE CARREGAMENTO (Previne a Tela Branca Pura)
  if (!isReady) {
    return (
      <div className="min-h-screen bg-black flex flex-col items-center justify-center gap-4">
        <div className="w-10 h-10 border-4 border-zinc-800 border-t-white rounded-full animate-spin"></div>
        <p className="text-zinc-600 text-[10px] font-bold uppercase tracking-widest">A ligar ao servidor...</p>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-black text-white font-sans selection:bg-zinc-800 antialiased">
      
      {/* AVISO DO ADMIN */}
      {systemNotice.active && view === 'public' && (
        <div className="bg-amber-500 text-black px-6 py-4 flex items-center justify-center gap-3 animate-in slide-in-from-top duration-700">
          <Clock size={18} className="shrink-0" />
          <p className="text-[11px] font-black uppercase tracking-tight text-center leading-tight">
            {systemNotice.text}
          </p>
        </div>
      )}

      {/* INTERFACE PÚBLICA */}
      {view === 'public' && !sent && (
        <div className="max-w-md mx-auto px-6 py-12 animate-in fade-in duration-500">
          <header className="mb-12 text-center">
            <h1 className="text-6xl font-black italic tracking-tighter">SUGESTÕES</h1>
            <p className="text-[10px] uppercase tracking-[0.4em] text-zinc-600 font-bold mt-2">Nicole & Família</p>
          </header>

          <form onSubmit={handleSubmit} className="space-y-4">
            <div className="flex flex-col items-center mb-8 gap-2">
              <div 
                onClick={() => fileInputRef.current?.click()} 
                className="w-24 h-24 rounded-[2.5rem] bg-zinc-900 border-2 border-zinc-800 flex items-center justify-center cursor-pointer overflow-hidden transition-all hover:border-zinc-500 group"
              >
                {selectedImage ? (
                  <img src={selectedImage} className="w-full h-full object-cover" alt="Avatar" />
                ) : (
                  <div className="flex flex-col items-center gap-1 opacity-20">
                    <User size={24} />
                    <span className="text-[8px] font-black uppercase tracking-tighter">Foto</span>
                  </div>
                )}
              </div>
              <span className="text-[9px] font-bold text-zinc-700 uppercase tracking-widest italic">Opcional</span>
              <input type="file" ref={fileInputRef} className="hidden" accept="image/*" onChange={(e) => {
                const file = e.target.files[0];
                if (file) {
                  const r = new FileReader();
                  r.onload = () => setSelectedImage(r.result);
                  r.readAsDataURL(file);
                }
              }} />
            </div>

            <input 
              type="text" 
              placeholder="Teu Nome" 
              required 
              value={name} 
              onChange={e => setName(e.target.value)}
              className="w-full p-5 bg-zinc-900 border border-zinc-800 rounded-3xl outline-none focus:border-white font-bold transition-all placeholder:text-zinc-700" 
            />
            
            <textarea 
              placeholder="Escreve aqui a tua sugestão..." 
              required 
              value={suggestion} 
              onChange={e => setSuggestion(e.target.value)}
              className="w-full p-5 bg-zinc-900 border border-zinc-800 rounded-3xl outline-none focus:border-white h-40 resize-none font-medium transition-all placeholder:text-zinc-700" 
            />

            <button 
              type="submit"
              disabled={loading} 
              className="w-full bg-white text-black font-black py-6 rounded-3xl transition-all uppercase tracking-widest text-xs flex items-center justify-center gap-3 active:scale-95 disabled:opacity-50"
            >
              {loading ? "A GUARDAR..." : "ENVIAR SUGESTÃO"}
            </button>
          </form>

          <button onClick={() => setView('admin-login')} className="mt-20 w-full text-zinc-900 text-[10px] font-bold uppercase flex items-center justify-center gap-2 hover:text-zinc-600">
            <Lock size={12} /> Gerir Mural
          </button>
        </div>
      )}

      {/* SUCESSO */}
      {sent && (
        <div className="max-w-md mx-auto px-6 py-24 text-center animate-in zoom-in-95 duration-300">
          <div className="bg-zinc-950 p-10 rounded-[3rem] border border-zinc-900 shadow-2xl">
            <CheckCircle className="mx-auto mb-6 text-emerald-500" size={60} />
            <h2 className="font-black text-3xl mb-2 tracking-tighter italic uppercase">Enviada!</h2>
            <p className="text-zinc-600 text-[10px] mb-8 uppercase font-bold tracking-widest">
              A Nicole já recebeu a tua ideia.<br/>Fica atento aos vídeos!
            </p>
            <div className="bg-black p-6 rounded-3xl border border-zinc-800 mb-8">
              <p className="text-[9px] font-black text-zinc-700 uppercase mb-3 tracking-tighter">ID DA SUGESTÃO</p>
              <span className="text-3xl font-mono font-black text-white tracking-widest">{generatedCode}</span>
            </div>
            <button onClick={() => setSent(false)} className="w-full bg-white text-black font-black py-5 rounded-2xl text-xs uppercase tracking-widest">
              VOLTAR
            </button>
          </div>
        </div>
      )}

      {/* LOGIN ADMIN */}
      {view === 'admin-login' && (
        <div className="max-w-sm mx-auto pt-40 px-6 animate-in fade-in">
           <div className="bg-zinc-900 p-10 rounded-[3rem] border border-zinc-800 text-center">
            <h2 className="font-black mb-8 italic uppercase text-xl">LOGIN</h2>
            <input 
              type="password" 
              autoFocus 
              className="w-full p-5 bg-black rounded-2xl mb-6 text-center border border-zinc-800 outline-none font-bold"
              onChange={e => setAdminPass(e.target.value)} 
              placeholder="Senha" 
            />
            <button 
              onClick={() => adminPass === ADMIN_PASSWORD ? setView('dashboard') : alert("Palavra-passe errada")}
              className="w-full py-4 bg-white text-black rounded-2xl font-black text-xs uppercase active:scale-95 transition-all"
            >
              Entrar
            </button>
            <button onClick={() => setView('public')} className="mt-6 text-zinc-600 text-[10px] uppercase font-bold">Cancelar</button>
          </div>
        </div>
      )}

      {/* DASHBOARD ADMIN */}
      {view === 'dashboard' && (
        <div className="max-w-5xl mx-auto px-6 py-12 animate-in fade-in">
          <header className="flex justify-between items-center mb-12">
            <div>
              <h1 className="text-5xl font-black italic tracking-tighter uppercase">Mural</h1>
              <p className="text-zinc-600 text-[10px] font-bold uppercase tracking-widest mt-1">Tens {suggestions.length} ideias salvas</p>
            </div>
            <button onClick={() => setView('public')} className="p-4 bg-zinc-900 rounded-2xl hover:bg-zinc-800"><LogOut size={20}/></button>
          </header>

          <div className="mb-12 p-8 bg-zinc-950 border border-zinc-900 rounded-[2.5rem]">
             <div className="flex items-center gap-3 mb-6">
                <Megaphone size={20} className="text-amber-500" />
                <h3 className="font-black uppercase text-sm italic tracking-tight">Publicar Aviso na Home</h3>
             </div>
             <textarea 
               value={newNoticeText} 
               onChange={e => setNewNoticeText(e.target.value)}
               className="w-full bg-black border border-zinc-800 p-5 rounded-2xl text-xs font-bold text-zinc-400 outline-none mb-4 h-24"
             />
             <button 
               onClick={toggleNotice} 
               className={`w-full py-5 rounded-2xl font-black text-[10px] uppercase tracking-widest transition-all ${systemNotice.active ? 'bg-red-500/10 text-red-500 border border-red-500/30' : 'bg-emerald-500 text-black'}`}
             >
               {systemNotice.active ? 'RETIRAR AVISO DO AR' : 'COLOCAR AVISO NO AR'}
             </button>
          </div>
          
          <div className="grid gap-6">
            {suggestions.map(s => (
              <div key={s.id} className="p-8 rounded-[3.5rem] border border-zinc-900 bg-zinc-950 flex flex-col md:flex-row gap-6 items-start hover:border-zinc-700 transition-all group">
                <img src={s.photoUrl} className="w-16 h-16 rounded-[1.5rem] object-cover border border-zinc-800 shadow-xl" alt="S" />
                <div className="flex-1 min-w-0">
                  <div className="flex items-center gap-3 mb-4">
                    <h3 className="font-black text-2xl italic tracking-tight">{s.name}</h3>
                    <span className="font-mono text-[10px] text-amber-500 font-black px-3 py-1 bg-black rounded-full border border-zinc-800">{s.code}</span>
                  </div>
                  <div className="text-zinc-300 text-lg leading-relaxed p-6 bg-black/50 rounded-[2rem] border border-white/5 font-medium">
                    {s.text}
                  </div>
                  <div className="mt-4 flex items-center gap-2 text-zinc-700 font-bold text-[9px] uppercase tracking-widest">
                    <Clock size={10} /> {new Date(s.timestamp).toLocaleDateString('pt-PT')} às {new Date(s.timestamp).toLocaleTimeString('pt-PT', {hour: '2-digit', minute:'2-digit'})}
                  </div>
                </div>
                <button 
                  onClick={async () => { if(confirm("Apagar esta sugestão?")) await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'suggestions', s.id)); }} 
                  className="p-5 text-zinc-800 hover:text-red-500 bg-zinc-900 rounded-3xl hover:bg-zinc-800 transition-all self-center md:self-start"
                >
                  <Trash2 size={24}/>
                </button>
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}

