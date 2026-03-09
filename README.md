import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, collection, doc, onSnapshot, setDoc, addDoc, deleteDoc, query } from 'firebase/firestore';
import { Plus, Trash2, Save, Calculator, Ruler, Settings2, Scissors, ChevronRight, Home, CheckCircle2, Link, Copy, ExternalLink, BookOpen, Sparkles } from 'lucide-react';

// --- Firebase 配置 ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'sewing-scaler-default';

// --- 常數定義 ---
const COMPONENT_TYPES = [
  { id: 'both', label: '全縮放', desc: '寬高皆隨袋體變動' },
  { id: 'width', label: '僅寬度', desc: '寬度變動，高度固定' },
  { id: 'height', label: '僅高度', desc: '高度變動，寬度固定' },
  { id: 'fixed', label: '固定', desc: '尺寸完全固定' },
];

export default function App() {
  const [user, setUser] = useState(null);
  const [templates, setTemplates] = useState([]);
  const [activeTab, setActiveTab] = useState('list'); // 'list', 'edit', 'calculate'
  const [currentTemplate, setCurrentTemplate] = useState(null);
  const [loading, setLoading] = useState(true);
  const [copyFeedback, setCopyFeedback] = useState(null);

  const [scaleInput, setScaleInput] = useState({ addW: 0, addH: 0 });

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth Error", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
      setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'templates'));
    const unsubscribe = onSnapshot(q, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setTemplates(data);
    }, (err) => console.error("Firestore Error", err));
    return () => unsubscribe();
  }, [user]);

  const saveTemplate = async (templateData) => {
    if (!user) return;
    const docRef = templateData.id 
      ? doc(db, 'artifacts', appId, 'public', 'data', 'templates', templateData.id)
      : doc(collection(db, 'artifacts', appId, 'public', 'data', 'templates'));
    
    await setDoc(docRef, {
      ...templateData,
      updatedAt: Date.now()
    }, { merge: true });
    
    setActiveTab('list');
  };

  const deleteTemplate = async (id) => {
    if (!window.confirm("確定要刪除這個模板嗎？")) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'templates', id));
  };

  const startNewTemplate = () => {
    setCurrentTemplate({
      name: '新作品',
      baseWidth: 14,
      baseHeight: 11.5,
      seamAllowance: 1.0, 
      components: [
        { id: Date.now(), name: '表布', baseW: 14, baseH: 11.5, type: 'both', quantity: 2, includeSA: true },
        { id: Date.now() + 1, name: '拉鍊', baseW: 13, baseH: 1, type: 'width', quantity: 1, includeSA: false },
      ],
      links: []
    });
    setActiveTab('edit');
  };

  const handleCopyLink = (url, id) => {
    const el = document.createElement('textarea');
    el.value = url;
    document.body.appendChild(el);
    el.select();
    document.execCommand('copy');
    document.body.removeChild(el);
    
    setCopyFeedback(id);
    setTimeout(() => setCopyFeedback(null), 2000);
  };

  const calculateFinalSize = (comp, addW, addH, globalSA) => {
    let baseW = Number(comp.baseW);
    let baseH = Number(comp.baseH);
    const sa = comp.includeSA ? Number(globalSA) : 0;

    if (comp.type === 'both') {
      baseW += Number(addW);
      baseH += Number(addH);
    } else if (comp.type === 'width') {
      baseW += Number(addW);
    } else if (comp.type === 'height') {
      baseH += Number(addH);
    }

    const finalW = baseW + (2 * sa);
    const finalH = baseH + (2 * sa);

    return { w: finalW.toFixed(1), h: finalH.toFixed(1) };
  };

  if (loading) return <div className="flex items-center justify-center h-screen bg-[#f7fdff] text-sky-400 font-bold animate-pulse">
    <div className="flex flex-col items-center gap-3">
      <div className="w-12 h-12 border-4 border-sky-100 border-t-sky-400 rounded-full animate-spin"></div>
      載入中...
    </div>
  </div>;

  return (
    <div className="min-h-screen bg-[#f7fdff] text-black font-sans pb-24">
      {/* 導覽列 - 超淡海鹽藍 */}
      <header className="bg-white/80 backdrop-blur-md border-b border-sky-50 sticky top-0 z-30 p-4 shadow-[0_2px_15px_rgba(224,242,254,0.4)]">
        <div className="max-w-2xl mx-auto flex justify-between items-center">
          <div className="flex items-center gap-2 cursor-pointer" onClick={() => setActiveTab('list')}>
            <div className="bg-gradient-to-br from-sky-300 to-cyan-400 p-2 rounded-2xl text-white shadow-lg shadow-sky-100">
              <Scissors size={22} />
            </div>
            <h1 className="text-xl font-black tracking-tight text-black">裁縫小助手</h1>
          </div>
          {activeTab === 'list' && (
            <button 
              onClick={startNewTemplate}
              className="bg-sky-400 hover:bg-sky-500 text-white px-5 py-2.5 rounded-full text-sm font-bold flex items-center gap-1.5 transition-all shadow-md shadow-sky-50 active:scale-95"
            >
              <Plus size={18} /> 新增模板
            </button>
          )}
        </div>
      </header>

      <main className="max-w-2xl mx-auto p-4">
        {/* 列表頁 */}
        {activeTab === 'list' && (
          <div className="grid gap-5">
            {templates.length === 0 ? (
              <div className="text-center py-24 bg-white/40 rounded-[2.5rem] border-2 border-dashed border-sky-100">
                <Sparkles size={48} className="mx-auto text-sky-100 mb-4" />
                <p className="text-sky-300 font-medium italic">點擊下方按鈕開始記錄你的創意</p>
                <button onClick={startNewTemplate} className="text-sky-500 text-sm font-bold mt-2 hover:underline">建立第一個作品</button>
              </div>
            ) : (
              templates.map(t => (
                <div key={t.id} className="bg-white p-6 rounded-[2rem] shadow-[0_8px_30px_rgb(0,0,0,0.02)] border border-sky-50 hover:border-sky-200 transition-all group relative overflow-hidden">
                  <div className="absolute top-0 right-0 w-32 h-32 bg-sky-50/30 rounded-full -mr-16 -mt-16 opacity-40 group-hover:scale-110 transition-transform"></div>
                  <div className="relative z-10 flex justify-between items-center">
                    <div>
                      <h3 className="text-lg font-black text-black">{t.name}</h3>
                      <div className="flex gap-3 mt-1.5 text-sm font-bold">
                        <span className="flex items-center gap-1 bg-sky-50/50 text-sky-600 px-2 py-0.5 rounded-lg">{t.baseWidth} × {t.baseHeight} cm</span>
                        <span className="flex items-center gap-1 text-cyan-500">縫份 {t.seamAllowance || 0}cm</span>
                      </div>
                    </div>
                    <div className="flex gap-2">
                      <button 
                        onClick={() => { setCurrentTemplate(t); setScaleInput({ addW: 0, addH: 0 }); setActiveTab('calculate'); }}
                        className="p-3 text-sky-400 bg-sky-50/50 rounded-2xl hover:bg-sky-400 hover:text-white transition-all"
                        title="計算縮放"
                      >
                        <Calculator size={20} />
                      </button>
                      <button 
                        onClick={() => { setCurrentTemplate(t); setActiveTab('edit'); }}
                        className="p-3 text-slate-400 bg-slate-50 rounded-2xl hover:bg-slate-200 hover:text-black transition-all"
                      >
                        <Settings2 size={20} />
                      </button>
                      <button 
                        onClick={() => deleteTemplate(t.id)}
                        className="p-3 text-rose-300 bg-rose-50/30 rounded-2xl hover:bg-rose-500 hover:text-white transition-all"
                      >
                        <Trash2 size={20} />
                      </button>
                    </div>
                  </div>
                </div>
              ))
            )}
          </div>
        )}

        {/* 編輯頁 */}
        {activeTab === 'edit' && currentTemplate && (
          <div className="space-y-6 animate-in fade-in slide-in-from-bottom-4 duration-500">
            <div className="bg-white p-8 rounded-[2.5rem] shadow-sm border border-sky-50 space-y-6">
              <h2 className="text-xl font-black flex items-center gap-2 text-black"><Settings2 size={24} className="text-sky-400"/> 基礎設定</h2>
              <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                <div className="md:col-span-2">
                  <label className="block text-xs font-black text-black/40 mb-2 uppercase tracking-widest">作品名稱</label>
                  <input 
                    type="text" 
                    value={currentTemplate.name} 
                    onChange={(e) => setCurrentTemplate({...currentTemplate, name: e.target.value})}
                    placeholder="輸入作品名稱..."
                    className="w-full bg-sky-50/30 border-2 border-transparent focus:border-sky-100 rounded-2xl p-4 transition-all outline-none font-black text-black placeholder:text-sky-200"
                  />
                </div>
                <div>
                  <label className="block text-xs font-black text-black/40 mb-2 uppercase tracking-widest">基準尺寸 (寬 × 高 cm)</label>
                  <div className="flex items-center gap-3">
                    <input 
                      type="number" 
                      value={currentTemplate.baseWidth} 
                      onChange={(e) => setCurrentTemplate({...currentTemplate, baseWidth: parseFloat(e.target.value)})}
                      className="w-full bg-sky-50/30 border-2 border-transparent focus:border-sky-100 rounded-2xl p-4 text-center font-black"
                    />
                    <span className="text-sky-100 font-black">×</span>
                    <input 
                      type="number" 
                      value={currentTemplate.baseHeight} 
                      onChange={(e) => setCurrentTemplate({...currentTemplate, baseHeight: parseFloat(e.target.value)})}
                      className="w-full bg-sky-50/30 border-2 border-transparent focus:border-sky-100 rounded-2xl p-4 text-center font-black"
                    />
                  </div>
                </div>
                <div>
                  <label className="block text-xs font-black text-black/40 mb-2 uppercase tracking-widest">預設縫份 (cm)</label>
                  <input 
                    type="number" 
                    step="0.1"
                    value={currentTemplate.seamAllowance} 
                    onChange={(e) => setCurrentTemplate({...currentTemplate, seamAllowance: parseFloat(e.target.value)})}
                    className="w-full bg-cyan-50/50 border-2 border-cyan-100 rounded-2xl p-4 font-black text-black text-center"
                  />
                </div>
              </div>
            </div>

            {/* 教學網址區塊 */}
            <div className="bg-white p-8 rounded-[2.5rem] shadow-sm border border-sky-50 space-y-5">
              <div className="flex justify-between items-center">
                <h2 className="text-xl font-black flex items-center gap-2 text-black"><Link size={24} className="text-sky-400"/> 教學與連結</h2>
                <button 
                  onClick={() => setCurrentTemplate({
                    ...currentTemplate, 
                    links: [...(currentTemplate.links || []), { id: Date.now(), title: '', url: '' }]
                  })}
                  className="bg-sky-50 text-sky-500 px-4 py-2 rounded-xl text-xs font-black hover:bg-sky-100 transition-all border border-sky-100"
                >
                  + 新增連結
                </button>
              </div>
              
              <div className="space-y-4">
                {currentTemplate.links?.length > 0 ? (
                  currentTemplate.links.map((link, idx) => (
                    <div key={link.id} className="flex gap-4 items-start p-5 bg-sky-50/20 rounded-3xl border border-sky-50 group">
                      <div className="flex-1 space-y-3">
                        <input 
                          type="text" 
                          placeholder="教學標題"
                          value={link.title}
                          onChange={(e) => {
                            const newLinks = [...currentTemplate.links];
                            newLinks[idx].title = e.target.value;
                            setCurrentTemplate({...currentTemplate, links: newLinks});
                          }}
                          className="w-full bg-white border-transparent focus:border-sky-100 rounded-xl p-3 text-sm font-black outline-none shadow-sm text-black"
                        />
                        <input 
                          type="text" 
                          placeholder="URL"
                          value={link.url}
                          onChange={(e) => {
                            const newLinks = [...currentTemplate.links];
                            newLinks[idx].url = e.target.value;
                            setCurrentTemplate({...currentTemplate, links: newLinks});
                          }}
                          className="w-full bg-white/50 border-transparent focus:border-sky-100 rounded-xl p-3 text-xs text-black/60 outline-none"
                        />
                      </div>
                      <button 
                        onClick={() => {
                          const newLinks = currentTemplate.links.filter((_, i) => i !== idx);
                          setCurrentTemplate({...currentTemplate, links: newLinks});
                        }}
                        className="p-2 text-sky-200 hover:text-rose-500 transition-colors"
                      >
                        <Trash2 size={20} />
                      </button>
                    </div>
                  ))
                ) : (
                  <p className="text-sm text-sky-200 text-center py-6 italic font-bold tracking-tight">尚未儲存教學連結</p>
                )}
              </div>
            </div>

            <div className="space-y-5">
              <div className="flex justify-between items-center px-4">
                <h2 className="text-xl font-black flex items-center gap-2 text-black"><Ruler size={24} className="text-sky-400"/> 零件配置</h2>
                <button 
                  onClick={() => setCurrentTemplate({
                    ...currentTemplate, 
                    components: [...currentTemplate.components, { id: Date.now(), name: '新零件', baseW: 10, baseH: 10, type: 'both', quantity: 1, includeSA: true }]
                  })}
                  className="bg-sky-400 text-white px-5 py-2.5 rounded-2xl text-sm font-black flex items-center gap-1.5 shadow-lg shadow-sky-50 hover:bg-sky-500 transition-all"
                >
                  <Plus size={18} /> 新增零件
                </button>
              </div>

              {currentTemplate.components.map((comp, idx) => (
                <div key={comp.id} className="bg-white p-6 rounded-[2rem] shadow-sm border border-sky-50 space-y-5">
                  <div className="flex justify-between items-center">
                    <input 
                      type="text" 
                      value={comp.name} 
                      onChange={(e) => {
                        const newComps = [...currentTemplate.components];
                        newComps[idx].name = e.target.value;
                        setCurrentTemplate({...currentTemplate, components: newComps});
                      }}
                      className="font-black text-black bg-transparent border-b-2 border-transparent focus:border-sky-100 outline-none text-xl"
                    />
                    <div className="flex items-center gap-5">
                      <label className="flex items-center gap-2 cursor-pointer">
                         <span className={`text-xs font-black uppercase tracking-tighter ${comp.includeSA ? 'text-black' : 'text-slate-300'}`}>計算縫份</span>
                         <div 
                           onClick={() => {
                             const newComps = [...currentTemplate.components];
                             newComps[idx].includeSA = !newComps[idx].includeSA;
                             setCurrentTemplate({...currentTemplate, components: newComps});
                           }}
                           className={`w-12 h-6 rounded-full relative transition-all duration-300 ${comp.includeSA ? 'bg-sky-400' : 'bg-slate-200'}`}
                         >
                           <div className={`absolute top-1 w-4 h-4 bg-white rounded-full shadow-sm transition-all duration-300 ${comp.includeSA ? 'left-7' : 'left-1'}`} />
                         </div>
                      </label>
                      <button 
                        onClick={() => {
                          const newComps = currentTemplate.components.filter((_, i) => i !== idx);
                          setCurrentTemplate({...currentTemplate, components: newComps});
                        }}
                        className="text-slate-200 hover:text-rose-500 transition-colors"
                      >
                        <Trash2 size={20} />
                      </button>
                    </div>
                  </div>
                  
                  <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
                    <div className="space-y-1.5">
                      <label className="text-[10px] font-black text-black/30 uppercase tracking-[0.1em]">原始寬</label>
                      <input 
                        type="number" 
                        value={comp.baseW} 
                        onChange={(e) => {
                          const newComps = [...currentTemplate.components];
                          newComps[idx].baseW = parseFloat(e.target.value);
                          setCurrentTemplate({...currentTemplate, components: newComps});
                        }}
                        className="w-full bg-sky-50/30 border-none rounded-xl p-3 font-black text-black text-center"
                      />
                    </div>
                    <div className="space-y-1.5">
                      <label className="text-[10px] font-black text-black/30 uppercase tracking-[0.1em]">原始高</label>
                      <input 
                        type="number" 
                        value={comp.baseH} 
                        onChange={(e) => {
                          const newComps = [...currentTemplate.components];
                          newComps[idx].baseH = parseFloat(e.target.value);
                          setCurrentTemplate({...currentTemplate, components: newComps});
                        }}
                        className="w-full bg-sky-50/30 border-none rounded-xl p-3 font-black text-black text-center"
                      />
                    </div>
                    <div className="space-y-1.5">
                      <label className="text-[10px] font-black text-black/30 uppercase tracking-[0.1em]">數量</label>
                      <input 
                        type="number" 
                        value={comp.quantity} 
                        onChange={(e) => {
                          const newComps = [...currentTemplate.components];
                          newComps[idx].quantity = parseInt(e.target.value);
                          setCurrentTemplate({...currentTemplate, components: newComps});
                        }}
                        className="w-full bg-sky-50/30 border-none rounded-xl p-3 font-black text-black text-center"
                      />
                    </div>
                    <div className="space-y-1.5">
                      <label className="text-[10px] font-black text-black/30 uppercase tracking-[0.1em]">規則</label>
                      <select 
                        value={comp.type} 
                        onChange={(e) => {
                          const newComps = [...currentTemplate.components];
                          newComps[idx].type = e.target.value;
                          setCurrentTemplate({...currentTemplate, components: newComps});
                        }}
                        className="w-full bg-sky-50/30 border-none rounded-xl p-3 font-black text-black text-sm appearance-none text-center"
                      >
                        {COMPONENT_TYPES.map(t => <option key={t.id} value={t.id}>{t.label}</option>)}
                      </select>
                    </div>
                  </div>
                </div>
              ))}
            </div>

            <div className="flex gap-4 pt-6">
              <button 
                onClick={() => saveTemplate(currentTemplate)}
                className="flex-1 bg-sky-400 text-white font-black py-5 rounded-[2rem] flex items-center justify-center gap-2 shadow-xl shadow-sky-50 active:scale-[0.98] transition-all"
              >
                <Save size={24} /> 儲存模板
              </button>
              <button 
                onClick={() => setActiveTab('list')}
                className="px-10 bg-slate-100 text-slate-500 font-black py-5 rounded-[2rem] hover:bg-slate-200 transition-all"
              >
                取消
              </button>
            </div>
          </div>
        )}

        {/* 計算頁 */}
        {activeTab === 'calculate' && currentTemplate && (
          <div className="space-y-8 animate-in fade-in zoom-in-95 duration-500">
            <div className="bg-gradient-to-br from-sky-300 via-sky-400 to-cyan-400 text-white p-8 rounded-[3rem] shadow-2xl space-y-8 relative overflow-hidden">
              <div className="absolute -right-10 -top-10 w-48 h-48 bg-white/10 rounded-full blur-2xl"></div>
              
              <div className="flex justify-between items-start relative z-10">
                <div>
                  <h2 className="text-3xl font-black tracking-tight text-white">{currentTemplate.name}</h2>
                  <div className="flex gap-2 mt-2">
                    <span className="bg-white/20 backdrop-blur-sm px-3 py-1 rounded-full text-[10px] font-black uppercase tracking-widest text-white">基準: {currentTemplate.baseWidth}x{currentTemplate.baseHeight}</span>
                    <span className="bg-white text-sky-500 px-3 py-1 rounded-full text-[10px] font-black uppercase tracking-widest shadow-lg">縫份: {currentTemplate.seamAllowance}cm</span>
                  </div>
                </div>
              </div>
              
              <div className="grid grid-cols-2 gap-6 relative z-10">
                <div className="space-y-2.5">
                  <label className="text-[10px] font-black opacity-90 uppercase tracking-[0.3em] ml-2 text-white">寬度增加 (W)</label>
                  <div className="relative group">
                    <input 
                      type="number" 
                      value={scaleInput.addW} 
                      onChange={(e) => setScaleInput({...scaleInput, addW: e.target.value})}
                      className="w-full bg-white/20 backdrop-blur-md border-2 border-white/30 rounded-[2rem] p-5 text-white placeholder-white/50 focus:bg-white/30 transition-all outline-none text-3xl font-black shadow-inner"
                    />
                    <div className="absolute right-6 top-1/2 -translate-y-1/2 opacity-70 text-sm font-black">cm</div>
                  </div>
                </div>
                <div className="space-y-2.5">
                  <label className="text-[10px] font-black opacity-90 uppercase tracking-[0.3em] ml-2 text-white">高度增加 (H)</label>
                  <div className="relative group">
                    <input 
                      type="number" 
                      value={scaleInput.addH} 
                      onChange={(e) => setScaleInput({...scaleInput, addH: e.target.value})}
                      className="w-full bg-white/20 backdrop-blur-md border-2 border-white/30 rounded-[2rem] p-5 text-white placeholder-white/50 focus:bg-white/30 transition-all outline-none text-3xl font-black shadow-inner"
                    />
                    <div className="absolute right-6 top-1/2 -translate-y-1/2 opacity-70 text-sm font-black">cm</div>
                  </div>
                </div>
              </div>
            </div>

            {/* 教學連結顯示 */}
            {currentTemplate.links?.length > 0 && (
              <div className="space-y-4">
                 <h3 className="font-black text-black flex items-center gap-2 px-4 text-lg"><BookOpen size={22} className="text-sky-400"/> 教學資源</h3>
                 <div className="grid gap-3">
                    {currentTemplate.links.map(link => (
                      <div key={link.id} className="bg-white p-5 rounded-[1.8rem] border border-sky-50 flex items-center justify-between group shadow-sm hover:shadow-md transition-all">
                        <div className="flex-1 mr-4 overflow-hidden">
                          <h4 className="font-black text-black text-sm truncate">{link.title || '無標題'}</h4>
                          <p className="text-[10px] text-sky-300 font-bold truncate mt-0.5">{link.url || '---'}</p>
                        </div>
                        <button 
                          onClick={() => handleCopyLink(link.url, link.id)}
                          className={`flex items-center gap-2 px-5 py-2.5 rounded-2xl text-[10px] font-black transition-all active:scale-95 ${copyFeedback === link.id ? 'bg-emerald-500 text-white shadow-lg' : 'bg-sky-50 text-black hover:bg-sky-400 hover:text-white border border-sky-100'}`}
                        >
                          {copyFeedback === link.id ? (
                            <><CheckCircle2 size={16}/> 已複製</>
                          ) : (
                            <><Copy size={16}/> 複製網址</>
                          )}
                        </button>
                      </div>
                    ))}
                 </div>
              </div>
            )}

            <div className="space-y-5">
              <div className="flex items-center justify-between px-4">
                <h3 className="font-black text-black flex items-center gap-2 text-lg"><Scissors size={22} className="text-sky-400"/> 裁切清單</h3>
                <div className="flex items-center gap-2 text-[10px] font-black text-sky-400 bg-white border border-sky-50 px-4 py-2 rounded-full shadow-sm">
                  <Sparkles size={14}/> 自動計算縫份
                </div>
              </div>
              
              {currentTemplate.components.map(comp => {
                const final = calculateFinalSize(comp, scaleInput.addW, scaleInput.addH, currentTemplate.seamAllowance);
                const isModified = comp.type !== 'fixed' && (scaleInput.addW != 0 || scaleInput.addH != 0);
                
                return (
                  <div key={comp.id} className={`bg-white p-6 rounded-[2.2rem] border transition-all flex items-center gap-6 ${isModified ? 'border-sky-100 shadow-xl shadow-sky-50 ring-2 ring-sky-50' : 'border-sky-50 shadow-sm'}`}>
                    <div className={`w-16 h-16 rounded-[1.2rem] flex flex-col items-center justify-center font-black transition-all ${isModified ? 'bg-sky-400 text-white shadow-lg shadow-sky-100' : 'bg-slate-50 text-slate-200'}`}>
                      <span className="text-[10px] opacity-70 mb-0.5">數量</span>
                      <span className="text-2xl">{comp.quantity}</span>
                    </div>
                    <div className="flex-1">
                      <div className="flex justify-between items-center mb-1.5">
                        <span className="font-black text-black text-lg tracking-tight">{comp.name}</span>
                        {comp.includeSA ? (
                          <span className="text-[9px] text-white font-black uppercase tracking-widest bg-cyan-400 px-2.5 py-1 rounded-full shadow-sm">含縫份</span>
                        ) : (
                          <span className="text-[9px] text-slate-300 font-black uppercase tracking-widest bg-slate-50 px-2.5 py-1 rounded-full">淨尺寸</span>
                        )}
                      </div>
                      <div className="flex items-baseline gap-2">
                        <span className={`text-3xl font-black tracking-tighter ${isModified || comp.includeSA ? 'text-black' : 'text-slate-500'}`}>
                          {final.w} <span className="text-xl font-normal text-sky-200 mx-0.5">×</span> {final.h}
                        </span>
                        <span className="text-xs font-black text-sky-200 uppercase tracking-widest">cm</span>
                      </div>
                    </div>
                  </div>
                );
              })}
            </div>

            <button 
              onClick={() => setActiveTab('list')}
              className="w-full py-5 text-sky-300 font-black border-2 border-white bg-white/50 rounded-[2rem] hover:bg-white hover:text-sky-500 hover:border-sky-100 transition-all shadow-sm active:scale-95 mb-8"
            >
              返回模板列表
            </button>
          </div>
        )}
      </main>

      {/* 底部導航 - 懸浮式氣泡 */}
      <nav className="fixed bottom-6 left-4 right-4 bg-white/80 backdrop-blur-xl border border-sky-50 p-3 flex justify-around items-center max-w-2xl mx-auto z-40 rounded-[2.5rem] shadow-[0_10px_30px_rgba(224,242,254,0.6)]">
        <button 
          onClick={() => setActiveTab('list')}
          className={`flex flex-col items-center py-2 px-8 rounded-2xl transition-all duration-300 ${activeTab === 'list' ? 'text-black bg-sky-50 font-black shadow-inner' : 'text-slate-300 font-bold hover:text-sky-200'}`}
        >
          <Home size={22} className={activeTab === 'list' ? 'mb-0.5 scale-110' : 'mb-1'}/>
          <span className="text-[10px] tracking-tight">作品列表</span>
        </button>
        <button 
          onClick={startNewTemplate}
          className={`flex flex-col items-center py-2 px-8 rounded-2xl transition-all duration-300 ${activeTab === 'edit' ? 'text-black bg-sky-50 font-black shadow-inner' : 'text-slate-300 font-bold hover:text-sky-200'}`}
        >
          <Plus size={22} className={activeTab === 'edit' ? 'mb-0.5 scale-110' : 'mb-1'}/>
          <span className="text-[10px] tracking-tight">新增模板</span>
        </button>
        <button 
          onClick={() => { if(currentTemplate) setActiveTab('calculate'); }}
          className={`flex flex-col items-center py-2 px-8 rounded-2xl transition-all duration-300 ${activeTab === 'calculate' ? 'text-black bg-sky-50 font-black shadow-inner' : 'text-slate-300 font-bold hover:text-sky-200'} ${!currentTemplate ? 'opacity-20 pointer-events-none' : ''}`}
        >
          <Calculator size={22} className={activeTab === 'calculate' ? 'mb-0.5 scale-110' : 'mb-1'}/>
          <span className="text-[10px] tracking-tight">尺寸計算</span>
        </button>
      </nav>
    </div>
  );
}
