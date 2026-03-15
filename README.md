import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, updateDoc, doc, deleteDoc, setDoc, increment } from 'firebase/firestore';
import { 
  AlertCircle, CheckCircle2, Clock, User, PlusCircle, Trash2, 
  ShieldAlert, Filter, Monitor, Search, 
  ThumbsUp, MessageSquare, Lock, Unlock, ChevronRight, Tag,
  Image as ImageIcon, X, Eye, Users, ChevronDown
} from 'lucide-react';

// Конфигурация Firebase (заполняется автоматически в среде исполнения)
const firebaseConfigStr = typeof __firebase_config !== 'undefined' ? __firebase_config : '{}';
let app, auth, db;
const appId = typeof __app_id !== 'undefined' ? __app_id : 'vcws-bug-tracker-v1';

try {
  if (firebaseConfigStr !== '{}') {
    const firebaseConfig = JSON.parse(firebaseConfigStr);
    app = initializeApp(firebaseConfig);
    auth = getAuth(app);
    db = getFirestore(app);
  }
} catch (e) {
  console.error("Ошибка инициализации Firebase:", e);
}

export default function App() {
  // Состояния приложения
  const [user, setUser] = useState(null);
  const [bugs, setBugs] = useState([]);
  const [filter, setFilter] = useState('open'); // По умолчанию открытые
  const [searchQuery, setSearchQuery] = useState('');
  const [isCloudActive, setIsCloudActive] = useState(false);

  // Админ-панель
  const [isAdmin, setIsAdmin] = useState(false);
  const [showAdminLogin, setShowAdminLogin] = useState(false);
  const [adminPin, setAdminPin] = useState('');

  // Поля формы
  const [title, setTitle] = useState('');
  const [desc, setDesc] = useState('');
  const [robloxName, setRobloxName] = useState('');
  const [category, setCategory] = useState('Gameplay');
  const [proofImage, setProofImage] = useState(null);
  const [siteViews, setSiteViews] = useState(0);

  // Состояния UI
  const [replyText, setReplyText] = useState({});
  const [openChats, setOpenChats] = useState({});

  // Иконка дракона (белая, уменьшенная)
  const DragonFace = ({ size = 20, className = "" }) => (
    <svg width={size} height={size} viewBox="0 0 24 24" fill="none" stroke="white" strokeWidth="2.5" strokeLinecap="round" strokeLinejoin="round" className={className}>
      <path d="M7 3l-2 5 3 2" />
      <path d="M17 3l2 5-3 2" />
      <path d="M5 10l-3 4 4 2-1 5 6-2 1 4 1-4 6 2-1-5 4-2-3-4" />
      <path d="M8 14l2-1" />
      <path d="M16 14l-2-1" />
      <path d="M10 18h4" />
    </svg>
  );

  // Локальное хранилище (если облако недоступно)
  const saveLocalBugs = (newBugs) => {
    setBugs(newBugs);
    localStorage.setItem('vcws_bugs', JSON.stringify(newBugs));
  };

  // Авторизация
  useEffect(() => {
    if (auth) {
      const initAuth = async () => {
        try {
          if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
            await signInWithCustomToken(auth, __initial_auth_token);
          } else {
            await signInAnonymously(auth);
          }
        } catch (e) { console.error("Ошибка авторизации", e); }
      };
      initAuth();
      const unsub = onAuthStateChanged(auth, setUser);
      return () => unsub();
    }
  }, []);

  // Загрузка данных
  useEffect(() => {
    if (db && user) {
      setIsCloudActive(true);
      
      // Просмотры сайта
      const statsRef = doc(db, 'artifacts', appId, 'public', 'data', 'site_stats', 'main');
      if (!sessionStorage.getItem('vcws_viewed')) {
        setDoc(statsRef, { views: increment(1) }, { merge: true });
        sessionStorage.setItem('vcws_viewed', 'true');
      }
      onSnapshot(statsRef, (snap) => {
        if (snap.exists()) setSiteViews(snap.data().views || 0);
      });

      // Баги
      const bugsRef = collection(db, 'artifacts', appId, 'public', 'data', 'roblox_bugs_v2');
      const unsub = onSnapshot(bugsRef, (snap) => {
        const list = [];
        snap.forEach(d => list.push({ id: d.id, ...d.data() }));
        list.sort((a, b) => b.timestamp - a.timestamp);
        setBugs(list);
      });
      return () => unsub();
    } else {
      // Режим без интернета
      const local = localStorage.getItem('vcws_bugs');
      if (local) setBugs(JSON.parse(local));
      const v = localStorage.getItem('vcws_views') || "0";
      setSiteViews(parseInt(v));
    }
  }, [user]);

  // Обработка фото
  const handlePhoto = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (ev) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        const MAX = 800;
        let scale = 1;
        if (img.width > MAX) scale = MAX / img.width;
        canvas.width = img.width * scale;
        canvas.height = img.height * scale;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        setProofImage(canvas.toDataURL('image/jpeg', 0.6));
      };
      img.src = ev.target.result;
    };
    reader.readAsDataURL(file);
  };

  // Отправка бага
  const sendBug = async (e) => {
    e.preventDefault();
    if (!title.trim() || !desc.trim() || !robloxName.trim()) return;

    const bugData = {
      title: title.trim(),
      description: desc.trim(),
      author: robloxName.trim(),
      device: "PC",
      category,
      proofImage,
      status: 'open',
      timestamp: Date.now(),
      upvotes: 0,
      views: 0,
      votedUsers: [],
      replies: [],
      userId: user?.uid || 'anon'
    };

    if (db && user) {
      await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'roblox_bugs_v2'), bugData);
    } else {
      const local = [{ ...bugData, id: Date.now().toString() }, ...bugs];
      saveLocalBugs(local);
    }

    setTitle(''); setDesc(''); setRobloxName(''); setProofImage(null);
  };

  // Просмотр карточки
  const viewBug = async (id) => {
    if (!id || sessionStorage.getItem(`v_${id}`)) return;
    sessionStorage.setItem(`v_${id}`, 'true');
    if (db) {
      updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'roblox_bugs_v2', id), { views: increment(1) });
    } else {
      saveLocalBugs(bugs.map(b => b.id === id ? {...b, views: (b.views||0)+1} : b));
    }
  };

  // Лайк
  const vote = async (bug) => {
    const uid = user?.uid || 'local';
    const voted = bug.votedUsers?.includes(uid);
    let count = bug.upvotes || 0;
    let users = bug.votedUsers || [];

    if (voted) {
      count--;
      users = users.filter(i => i !== uid);
    } else {
      count++;
      users.push(uid);
    }

    if (db) {
      updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'roblox_bugs_v2', bug.id), { upvotes: count, votedUsers: users });
    } else {
      saveLocalBugs(bugs.map(b => b.id === bug.id ? {...b, upvotes: count, votedUsers: users} : b));
    }
  };

  // Сообщение в чат
  const sendReply = async (id, current) => {
    const txt = replyText[id];
    if (!txt?.trim()) return;

    const msg = {
      text: txt.trim(),
      author: isAdmin ? '👨‍💻 Разработчик' : 'Игрок',
      isAdminReply: isAdmin,
      timestamp: Date.now()
    };

    const next = [...(current || []), msg];
    if (db) {
      updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'roblox_bugs_v2', id), { replies: next });
    } else {
      saveLocalBugs(bugs.map(b => b.id === id ? {...b, replies: next} : b));
    }
    setReplyText({...replyText, [id]: ''});
  };

  // Админ: Решить/Удалить
  const fix = async (id, status) => {
    const s = status === 'open' ? 'resolved' : 'open';
    if (db) updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'roblox_bugs_v2', id), { status: s });
    else saveLocalBugs(bugs.map(b => b.id === id ? {...b, status: s} : b));
  };

  const remove = async (id) => {
    if (db) deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'roblox_bugs_v2', id));
    else saveLocalBugs(bugs.filter(b => b.id !== id));
  };

  // Фильтрация поиска
  const visibleBugs = bugs.filter(b => {
    const f = filter === 'all' ? true : b.status === filter;
    const s = b.title.toLowerCase().includes(searchQuery.toLowerCase()) || b.author.toLowerCase().includes(searchQuery.toLowerCase());
    return f && s;
  });

  return (
    <div className={`min-h-screen animated-bg text-gray-200 font-sans selection:bg-cyan-500/30 transition-all duration-700 ${isAdmin ? 'admin-mode' : ''}`}>
      <style>{`
        @keyframes gradientBG { 0% { background-position: 0% 50%; } 50% { background-position: 100% 50%; } 100% { background-position: 0% 50%; } }
        .animated-bg { background: linear-gradient(-45deg, #0a0a0c, #1a1625, #0a0a0c, #0f172a); background-size: 400% 400%; animation: gradientBG 15s ease infinite; }
        .admin-mode { background: linear-gradient(-45deg, #020617, #083344, #042f2e, #020617); background-size: 400% 400%; }
        
        .glass { background: rgba(17, 18, 20, 0.75); backdrop-filter: blur(16px); border: 1px solid rgba(255, 255, 255, 0.05); }
        .admin-mode .glass { background: rgba(10, 20, 35, 0.4); backdrop-filter: blur(24px) saturate(160%); border: 1px solid rgba(6, 182, 212, 0.2); border-top-color: rgba(6, 182, 212, 0.5); }
        
        .bg-grid { position: absolute; inset: -100%; width: 300%; height: 300%; pointer-events: none; }
        .grid-layer { position: absolute; inset: 0; background-size: 50px 50px; background-image: linear-gradient(to right, rgba(59, 130, 246, 0.05) 1px, transparent 1px), linear-gradient(to bottom, rgba(59, 130, 246, 0.05) 1px, transparent 1px); animation: pulse 10s infinite alternate; }
        .admin-mode .grid-layer { background-image: linear-gradient(to right, rgba(6, 182, 212, 0.15) 1px, transparent 1px), linear-gradient(to bottom, rgba(6, 182, 212, 0.15) 1px, transparent 1px); }
        @keyframes pulse { 0% { transform: scale(1); opacity: 0.3; } 100% { transform: scale(1.1) translate(1%, 1%); opacity: 0.6; } }
        
        .card-anim { animation: cardIn 0.5s cubic-bezier(0.2, 1, 0.3, 1) forwards; opacity: 0; }
        @keyframes cardIn { from { transform: translateY(20px); opacity: 0; } to { transform: translateY(0); opacity: 1; } }
      `}</style>

      {/* ФОН */}
      <div className="fixed inset-0 overflow-hidden z-0">
        <div className="grid-layer"></div>
        {isAdmin && <div className="absolute top-1/4 left-1/4 w-96 h-96 bg-cyan-500/10 blur-[120px] rounded-full animate-pulse"></div>}
      </div>

      {/* ШАПКА */}
      <header className="glass sticky top-0 z-50 border-b shadow-2xl">
        <div className="max-w-7xl mx-auto px-4 py-4 flex flex-col md:flex-row items-center justify-between gap-4">
          <div className="flex items-center gap-4">
            <div className={`p-2.5 rounded-2xl transition-all duration-500 ${isAdmin ? 'bg-cyan-500 shadow-[0_0_20px_rgba(6,182,212,0.6)]' : 'bg-blue-600 shadow-lg'}`}>
              <DragonFace size={22} />
            </div>
            <div>
              <h1 className="text-2xl font-black text-white tracking-tighter italic">VCWS</h1>
              <div className="flex items-center gap-2 text-[10px] font-bold uppercase tracking-widest text-gray-500">
                {isCloudActive ? <span className="text-emerald-500">● Online</span> : <span>○ Local</span>}
                {isAdmin && <span className="text-cyan-400 border border-cyan-500/30 px-1 rounded ml-2">Admin Mode</span>}
              </div>
            </div>
          </div>

          <div className="hidden lg:flex gap-6">
            <div className="text-center"><p className="text-[10px] text-gray-500 font-bold uppercase">Всего</p><p className="text-lg font-black">{bugs.length}</p></div>
            <div className="text-center"><p className="text-[10px] text-gray-500 font-bold uppercase">Посетители</p><p className="text-lg font-black text-blue-400">{siteViews}</p></div>
          </div>

          <div className="flex items-center gap-3 w-full md:w-auto">
            <div className="relative flex-1 md:w-64">
              <Search size={16} className="absolute left-3 top-1/2 -translate-y-1/2 text-gray-500" />
              <input 
                type="text" placeholder="Поиск по архиву..." value={searchQuery} onChange={(e) => setSearchQuery(e.target.value)}
                className="w-full pl-10 pr-4 py-2 bg-black/40 border border-white/10 rounded-xl text-sm focus:border-cyan-500 outline-none transition-all"
              />
            </div>
            {isAdmin ? (
              <button onClick={() => setIsAdmin(false)} className="px-4 py-2 bg-red-500/20 text-red-400 border border-red-500/30 rounded-xl font-bold text-xs hover:bg-red-500/30 transition-all">ВЫЙТИ</button>
            ) : (
              <button onClick={() => setShowAdminLogin(!showAdminLogin)} className="p-2.5 bg-white/5 border border-white/10 rounded-xl text-gray-400 hover:text-white transition-all"><Lock size={18}/></button>
            )}
          </div>
        </div>

        {/* ПИН-КОД */}
        {showAdminLogin && !isAdmin && (
          <div className="absolute top-full right-4 mt-2 p-5 glass rounded-2xl shadow-2xl w-72 card-anim">
            <p className="text-xs font-bold text-gray-400 mb-3 uppercase tracking-wider">Доступ Разработчика</p>
            <form onSubmit={(e) => {
              e.preventDefault();
              if (adminPin === '20030092331223473') { setIsAdmin(true); setShowAdminLogin(false); setAdminPin(''); }
              else alert('PIN неверный');
            }} className="flex gap-2">
              <input 
                type="password" value={adminPin} onChange={(e) => setAdminPin(e.target.value)} placeholder="••••"
                className="flex-1 bg-black/60 border border-white/10 rounded-lg text-center tracking-[1em] py-2 outline-none focus:border-cyan-500"
              />
              <button type="submit" className="px-4 bg-cyan-600 text-white rounded-lg font-bold">→</button>
            </form>
          </div>
        )}
      </header>

      <main className="max-w-7xl mx-auto px-4 py-8 grid grid-cols-1 lg:grid-cols-12 gap-8 relative z-10">
        
        {/* ФОРМА (Слева) */}
        <div className="lg:col-span-4">
          <div className="glass p-6 rounded-3xl border border-white/5 sticky top-28 shadow-2xl">
            <h2 className="text-xl font-black text-white mb-6 flex items-center gap-3">
              <PlusCircle size={24} className="text-blue-500" /> СОЗДАТЬ РЕПОРТ
            </h2>
            <form onSubmit={sendBug} className="space-y-4">
              <div className="space-y-1">
                <label className="text-[10px] font-bold text-gray-500 uppercase tracking-widest ml-1">Никнейм Roblox</label>
                <input type="text" value={robloxName} onChange={(e) => setRobloxName(e.target.value)} placeholder="Username" className="w-full px-4 py-3 bg-black/40 border border-white/10 rounded-2xl text-sm focus:border-blue-500 outline-none transition-all" required />
              </div>

              <div className="grid grid-cols-2 gap-3">
                <div className="space-y-1">
                  <label className="text-[10px] font-bold text-gray-500 uppercase tracking-widest ml-1">Железо</label>
                  <select className="w-full px-4 py-3 bg-black/40 border border-white/10 rounded-2xl text-sm opacity-50 cursor-not-allowed" disabled><option>PC</option></select>
                </div>
                <div className="space-y-1">
                  <label className="text-[10px] font-bold text-gray-500 uppercase tracking-widest ml-1">Тип бага</label>
                  <select value={category} onChange={(e) => setCategory(e.target.value)} className="w-full px-4 py-3 bg-black/40 border border-white/10 rounded-2xl text-sm focus:border-blue-500 outline-none">
                    <option value="Gameplay">Геймплей</option><option value="Map">Карта</option><option value="UI">Интерфейс</option><option value="Exploit">Читеры</option>
                  </select>
                </div>
              </div>

              <div className="space-y-1">
                <label className="text-[10px] font-bold text-gray-500 uppercase tracking-widest ml-1">Что случилось?</label>
                <input type="text" value={title} onChange={(e) => setTitle(e.target.value)} placeholder="Напр: Пропал инвентарь" className="w-full px-4 py-3 bg-black/40 border border-white/10 rounded-2xl text-sm focus:border-blue-500 outline-none" required />
              </div>

              <div className="space-y-1">
                <label className="text-[10px] font-bold text-gray-500 uppercase tracking-widest ml-1">Подробности</label>
                <textarea value={desc} onChange={(e) => setDesc(e.target.value)} placeholder="Опишите шаги..." className="w-full px-4 py-3 bg-black/40 border border-white/10 rounded-2xl text-sm focus:border-blue-500 outline-none min-h-[120px]" required />
              </div>

              <div className="pt-2">
                {proofImage ? (
                  <div className="relative rounded-2xl overflow-hidden border border-blue-500/50 card-anim">
                    <img src={proofImage} className="w-full h-32 object-cover" />
                    <button type="button" onClick={() => setProofImage(null)} className="absolute top-2 right-2 p-1.5 bg-red-500/80 text-white rounded-lg"><X size={16} /></button>
                  </div>
                ) : (
                  <label className="flex flex-col items-center justify-center h-24 bg-black/40 border-2 border-dashed border-white/10 rounded-2xl cursor-pointer hover:bg-white/5 hover:border-blue-500/50 transition-all">
                    <ImageIcon size={28} className="text-gray-600 mb-1" />
                    <span className="text-[10px] font-bold text-gray-500 uppercase tracking-widest">Добавить пруф</span>
                    <input type="file" accept="image/*" className="hidden" onChange={handlePhoto} />
                  </label>
                )}
              </div>

              <button type="submit" className="w-full py-4 bg-gradient-to-r from-blue-600 to-indigo-700 hover:from-blue-500 hover:to-indigo-600 text-white font-black rounded-2xl shadow-xl transition-all active:scale-95 uppercase tracking-widest">
                ОПУБЛИКОВАТЬ
              </button>
            </form>
          </div>
        </div>

        {/* ЛЕНТА (Справа) */}
        <div className="lg:col-span-8 space-y-6">
          <div className="flex gap-4 border-b border-white/5 pb-4">
            <button onClick={() => setFilter('open')} className={`px-6 py-2.5 rounded-xl font-bold text-sm transition-all ${filter === 'open' ? 'bg-blue-600 text-white shadow-lg' : 'text-gray-500 hover:text-white'}`}>🔥 АКТУАЛЬНЫЕ</button>
            <button onClick={() => setFilter('resolved')} className={`px-6 py-2.5 rounded-xl font-bold text-sm transition-all ${filter === 'resolved' ? 'bg-emerald-600 text-white shadow-lg' : 'text-gray-500 hover:text-white'}`}>✅ ИСПРАВЛЕННЫЕ</button>
          </div>

          <div className="space-y-4">
            {visibleBugs.length === 0 ? (
              <div className="text-center py-32 glass rounded-3xl border border-dashed border-white/10">
                <CheckCircle2 size={64} className="mx-auto text-white/5 mb-4" />
                <p className="text-gray-500 font-bold uppercase tracking-widest">Тут пока пусто</p>
              </div>
            ) : (
              visibleBugs.map((bug, idx) => (
                <div key={bug.id} onMouseEnter={() => viewBug(bug.id)} className={`glass rounded-3xl border overflow-hidden transition-all duration-500 card-anim ${bug.status === 'resolved' ? 'opacity-60 grayscale-[0.5]' : 'hover:border-blue-500/30 hover:shadow-2xl shadow-blue-900/10'}`} style={{ animationDelay: `${idx * 0.05}s` }}>
                  <div className={`h-1.5 w-full ${bug.status === 'resolved' ? 'bg-emerald-500' : 'bg-red-600'}`}></div>
                  
                  <div className="p-6">
                    <div className="flex flex-col sm:flex-row gap-6">
                      {/* Лайки */}
                      <div className="flex sm:flex-col items-center gap-2 p-2 bg-black/40 rounded-2xl border border-white/5 shrink-0 h-fit">
                        <button onClick={() => vote(bug)} className={`p-2 rounded-xl transition-all ${bug.votedUsers?.includes(user?.uid||'local') ? 'text-blue-400 bg-blue-500/10' : 'text-gray-500 hover:text-white'}`}>
                          <ThumbsUp size={20} fill={bug.votedUsers?.includes(user?.uid||'local') ? 'currentColor' : 'none'} />
                        </button>
                        <span className="text-lg font-black text-white">{bug.upvotes || 0}</span>
                      </div>

                      <div className="flex-1 min-w-0">
                        {/* Бейджи */}
                        <div className="flex flex-wrap items-center gap-3 mb-3">
                          <span className="flex items-center gap-1.5 px-3 py-1 bg-white/5 border border-white/10 rounded-full text-[10px] font-black text-gray-300 uppercase tracking-wider"><User size={12}/> {bug.author}</span>
                          <span className={`px-3 py-1 rounded-full text-[10px] font-black uppercase tracking-wider border ${getCategoryColor(bug.category)}`}>{bug.category}</span>
                          <span className="flex items-center gap-1.5 px-3 py-1 bg-blue-500/10 border border-blue-500/20 rounded-full text-[10px] font-black text-blue-400 uppercase tracking-wider"><Eye size={12}/> {bug.views || 0}</span>
                          <span className="text-[10px] text-gray-600 ml-auto font-bold uppercase tracking-widest italic">{new Date(bug.timestamp).toLocaleString('ru-RU', {day:'numeric', month:'short', hour:'2-digit', minute:'2-digit'})}</span>
                        </div>

                        <h3 className={`text-xl font-black mb-2 ${bug.status === 'resolved' ? 'text-gray-500 line-through' : 'text-white'}`}>{bug.title}</h3>
                        <p className="text-gray-400 text-sm leading-relaxed mb-6 italic">"{bug.description}"</p>

                        {bug.proofImage && (
                          <div className="mb-6 group relative rounded-2xl overflow-hidden border border-white/10 w-fit">
                            <img src={bug.proofImage} className="max-h-72 w-auto object-contain cursor-zoom-in group-hover:scale-105 transition-transform duration-500" onClick={() => window.open(bug.proofImage, '_blank')} />
                          </div>
                        )}

                        {/* ЧАТ */}
                        <div className="pt-6 border-t border-white/5">
                          <button onClick={() => setOpenChats({...openChats, [bug.id]: !openChats[bug.id]})} className="flex items-center justify-between w-full text-[10px] font-black uppercase tracking-[0.2em] text-gray-500 hover:text-white transition-colors group">
                            <span className="flex items-center gap-2"><MessageSquare size={14} className="group-hover:animate-bounce"/> Обсуждение ({(bug.replies || []).length})</span>
                            <div className={`transition-transform duration-300 ${openChats[bug.id] ? 'rotate-180' : ''}`}><ChevronDown size={14}/></div>
                          </button>

                          <div className={`grid transition-all duration-500 ease-in-out ${openChats[bug.id] ? 'grid-rows-[1fr] mt-5 opacity-100' : 'grid-rows-[0fr] mt-0 opacity-0 overflow-hidden'}`}>
                            <div className="overflow-hidden space-y-3">
                              <div className="max-h-64 overflow-y-auto pr-2 space-y-3 scroll-smooth">
                                {(bug.replies || []).map((r, i) => (
                                  <div key={i} className={`p-4 rounded-2xl border transition-all ${r.isAdminReply ? 'bg-cyan-500/10 border-cyan-500/30 shadow-[0_0_15px_rgba(6,182,212,0.1)]' : 'bg-white/5 border-white/5 hover:bg-white/10'}`}>
                                    <div className="flex justify-between items-center mb-1.5">
                                      <span className={`text-[10px] font-black uppercase tracking-wider ${r.isAdminReply ? 'text-cyan-400' : 'text-blue-500'}`}>{r.author}</span>
                                      <span className="text-[10px] text-gray-600 font-bold italic">{new Date(r.timestamp).toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'})}</span>
                                    </div>
                                    <p className="text-sm text-gray-200">{r.text}</p>
                                  </div>
                                ))}
                              </div>

                              {bug.status === 'open' ? (
                                <div className="flex gap-2 pt-2">
                                  <input 
                                    type="text" placeholder="Написать сообщение..." value={replyText[bug.id] || ''} onChange={(e) => setReplyText({...replyText, [bug.id]: e.target.value})}
                                    onKeyPress={(e) => e.key === 'Enter' && sendReply(bug.id, bug.replies)}
                                    className="flex-1 px-4 py-3 bg-black/60 border border-white/10 rounded-2xl text-sm focus:border-blue-600 outline-none transition-all"
                                  />
                                  <button onClick={() => sendReply(bug.id, bug.replies)} className="px-5 bg-blue-600 hover:bg-blue-500 text-white rounded-2xl transition-all shadow-lg active:scale-95"><ChevronRight size={20}/></button>
                                </div>
                              ) : <div className="text-center py-4 bg-black/30 rounded-2xl border border-white/5 text-[10px] font-black text-gray-600 uppercase tracking-widest">Обсуждение закрыто: Баг Исправлен</div>}
                            </div>
                          </div>
                        </div>
                      </div>

                      {/* АДМИН ПАНЕЛЬ (Dev Panel) */}
                      {isAdmin && (
                        <div className="flex flex-col gap-3 p-4 bg-cyan-950/20 border border-cyan-500/40 rounded-3xl shadow-2xl h-fit w-full sm:w-auto card-anim">
                          <div className="flex items-center gap-2 mb-1 justify-center sm:justify-start">
                            <ShieldAlert size={14} className="text-cyan-400 animate-pulse" />
                            <span className="text-[10px] font-black text-cyan-400 uppercase tracking-widest">Dev Panel</span>
                          </div>
                          <button onClick={() => fix(bug.id, bug.status)} className={`flex-1 sm:flex-none p-3.5 rounded-2xl border transition-all active:scale-90 flex justify-center items-center shadow-lg ${bug.status === 'open' ? 'bg-emerald-500/20 text-emerald-400 border-emerald-500/40 hover:bg-emerald-500/30' : 'bg-amber-500/20 text-amber-400 border-amber-500/40 hover:bg-amber-500/30'}`}>
                            {bug.status === 'open' ? <CheckCircle2 size={22}/> : <Clock size={22}/>}
                          </button>
                          <button onClick={() => remove(bug.id)} className="flex-1 sm:flex-none p-3.5 bg-red-500/20 text-red-400 border border-red-500/40 rounded-2xl hover:bg-red-500/30 transition-all active:scale-90 shadow-lg flex justify-center items-center">
                            <Trash2 size={22}/>
                          </button>
                        </div>
                      )}
                    </div>
                  </div>
                </div>
              ))
            )}
          </div>
        </div>
      </main>
    </div>
  );
}
