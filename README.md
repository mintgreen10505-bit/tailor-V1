<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>裁縫小助手 - 海鹽藍版</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Babel for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght@400;500;700;900&display=swap');
        body {
            font-family: 'Noto Sans TC', sans-serif;
            background-color: #f7fdff;
            color: black;
        }
        .bubble-shadow {
            box-shadow: 0 10px 30px rgba(224, 242, 254, 0.6);
        }
        .glass-nav {
            background: rgba(255, 255, 255, 0.8);
            backdrop-filter: blur(12px);
            -webkit-backdrop-filter: blur(12px);
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useMemo } = React;

        // 模擬 Firebase 或環境變數 (在 Canvas 環境中會由系統提供)
        const firebaseConfig = window.__firebase_config ? JSON.parse(window.__firebase_config) : {};
        const appId = window.__app_id || 'sewing-scaler-default';

        // 由於外部 CDN 載入 Lucide 可能較慢，定義一個簡單的圖示組件
        const Icon = ({ name, size = 24, className = "" }) => {
            return <i data-lucide={name} className={className} style={{ width: size, height: size }}></i>;
        };

        const COMPONENT_TYPES = [
            { id: 'both', label: '全縮放' },
            { id: 'width', label: '僅寬度' },
            { id: 'height', label: '僅高度' },
            { id: 'fixed', label: '固定' },
        ];

        function App() {
            const [activeTab, setActiveTab] = useState('list');
            const [templates, setTemplates] = useState([]);
            const [currentTemplate, setCurrentTemplate] = useState(null);
            const [scaleInput, setScaleInput] = useState({ addW: 0, addH: 0 });
            const [copyFeedback, setCopyFeedback] = useState(null);

            // 初始化時載入 Lucide 圖示
            useEffect(() => {
                if (window.lucide) window.lucide.createIcons();
            });

            // 模擬載入資料 (實際環境應對接 Firestore)
            useEffect(() => {
                const saved = localStorage.getItem('sewing_templates');
                if (saved) setTemplates(JSON.parse(saved));
            }, []);

            const saveToLocal = (data) => {
                setTemplates(data);
                localStorage.setItem('sewing_templates', JSON.stringify(data));
            };

            const handleSaveTemplate = (templateData) => {
                let newTemplates;
                if (templateData.id) {
                    newTemplates = templates.map(t => t.id === templateData.id ? templateData : t);
                } else {
                    newTemplates = [...templates, { ...templateData, id: Date.now() }];
                }
                saveToLocal(newTemplates);
                setActiveTab('list');
            };

            const handleDelete = (id) => {
                if (confirm("確定要刪除嗎？")) {
                    const newTemplates = templates.filter(t => t.id !== id);
                    saveToLocal(newTemplates);
                }
            };

            const startNew = () => {
                setCurrentTemplate({
                    name: '新作品',
                    baseWidth: 14,
                    baseHeight: 11.5,
                    seamAllowance: 1.0,
                    components: [
                        { id: 1, name: '表布', baseW: 14, baseH: 11.5, type: 'both', quantity: 2, includeSA: true },
                        { id: 2, name: '拉鍊', baseW: 13, baseH: 1, type: 'width', quantity: 1, includeSA: false }
                    ],
                    links: []
                });
                setActiveTab('edit');
            };

            const handleCopy = (url, id) => {
                const el = document.createElement('textarea');
                el.value = url;
                document.body.appendChild(el);
                el.select();
                document.execCommand('copy');
                document.body.removeChild(el);
                setCopyFeedback(id);
                setTimeout(() => setCopyFeedback(null), 2000);
            };

            const calculate = (comp, addW, addH, globalSA) => {
                let w = Number(comp.baseW);
                let h = Number(comp.baseH);
                const sa = comp.includeSA ? Number(globalSA) : 0;

                if (comp.type === 'both') {
                    w += Number(addW);
                    h += Number(addH);
                } else if (comp.type === 'width') {
                    w += Number(addW);
                } else if (comp.type === 'height') {
                    h += Number(addH);
                }

                return {
                    w: (w + 2 * sa).toFixed(1),
                    h: (h + 2 * sa).toFixed(1)
                };
            };

            return (
                <div className="min-h-screen pb-24">
                    {/* Header */}
                    <header className="bg-white/80 backdrop-blur-md border-b border-sky-50 sticky top-0 z-30 p-4 shadow-sm">
                        <div className="max-w-2xl mx-auto flex justify-between items-center">
                            <div className="flex items-center gap-2 cursor-pointer" onClick={() => setActiveTab('list')}>
                                <div className="bg-gradient-to-br from-sky-300 to-cyan-400 p-2 rounded-2xl text-white shadow-lg">
                                    <Icon name="scissors" size={20} />
                                </div>
                                <h1 className="text-xl font-black tracking-tight">裁縫小助手</h1>
                            </div>
                            {activeTab === 'list' && (
                                <button onClick={startNew} className="bg-sky-400 hover:bg-sky-500 text-white px-5 py-2.5 rounded-full text-sm font-bold flex items-center gap-1.5 shadow-md active:scale-95 transition-all">
                                    <Icon name="plus" size={18} /> 新增模板
                                </button>
                            )}
                        </div>
                    </header>

                    <main className="max-w-2xl mx-auto p-4">
                        {/* List View */}
                        {activeTab === 'list' && (
                            <div className="grid gap-5">
                                {templates.length === 0 ? (
                                    <div className="text-center py-24 bg-white/40 rounded-[2.5rem] border-2 border-dashed border-sky-100">
                                        <Icon name="sparkles" size={48} className="mx-auto text-sky-100 mb-4" />
                                        <p className="text-sky-300 font-medium italic">點擊下方按鈕開始記錄你的創意</p>
                                    </div>
                                ) : (
                                    templates.map(t => (
                                        <div key={t.id} className="bg-white p-6 rounded-[2rem] border border-sky-50 shadow-sm hover:border-sky-200 transition-all flex justify-between items-center group relative overflow-hidden">
                                            <div className="relative z-10">
                                                <h3 className="text-lg font-black">{t.name}</h3>
                                                <div className="flex gap-3 mt-1.5 text-sm font-bold">
                                                    <span className="bg-sky-50 text-sky-600 px-2 py-0.5 rounded-lg">{t.baseWidth}×{t.baseHeight}cm</span>
                                                    <span className="text-cyan-500">縫份 {t.seamAllowance}cm</span>
                                                </div>
                                            </div>
                                            <div className="flex gap-2 relative z-10">
                                                <button onClick={() => { setCurrentTemplate(t); setScaleInput({addW:0,addH:0}); setActiveTab('calculate'); }} className="p-3 text-sky-400 bg-sky-50 rounded-2xl hover:bg-sky-400 hover:text-white transition-all"><Icon name="calculator" size={20} /></button>
                                                <button onClick={() => { setCurrentTemplate(t); setActiveTab('edit'); }} className="p-3 text-slate-400 bg-slate-50 rounded-2xl hover:bg-slate-200 hover:text-black transition-all"><Icon name="settings-2" size={20} /></button>
                                                <button onClick={() => handleDelete(t.id)} className="p-3 text-rose-300 bg-rose-50 rounded-2xl hover:bg-rose-500 hover:text-white transition-all"><Icon name="trash-2" size={20} /></button>
                                            </div>
                                        </div>
                                    ))
                                )}
                            </div>
                        )}

                        {/* Edit View */}
                        {activeTab === 'edit' && currentTemplate && (
                            <div className="space-y-6">
                                <div className="bg-white p-8 rounded-[2.5rem] shadow-sm border border-sky-50 space-y-6">
                                    <h2 className="text-xl font-black flex items-center gap-2"><Icon name="settings-2" className="text-sky-400"/> 基礎設定</h2>
                                    <div className="space-y-4">
                                        <label className="block text-xs font-black text-black/40 uppercase tracking-widest">作品名稱</label>
                                        <input type="text" value={currentTemplate.name} onChange={e => setCurrentTemplate({...currentTemplate, name: e.target.value})} className="w-full bg-sky-50/30 border-2 border-transparent focus:border-sky-100 rounded-2xl p-4 outline-none font-black" />
                                        <div className="grid grid-cols-2 gap-4">
                                            <div>
                                                <label className="block text-xs font-black text-black/40 mb-2">基準尺寸 (寬×高)</label>
                                                <div className="flex items-center gap-2">
                                                    <input type="number" value={currentTemplate.baseWidth} onChange={e => setCurrentTemplate({...currentTemplate, baseWidth: e.target.value})} className="w-full bg-sky-50/30 rounded-2xl p-4 text-center font-black" />
                                                    <input type="number" value={currentTemplate.baseHeight} onChange={e => setCurrentTemplate({...currentTemplate, baseHeight: e.target.value})} className="w-full bg-sky-50/30 rounded-2xl p-4 text-center font-black" />
                                                </div>
                                            </div>
                                            <div>
                                                <label className="block text-xs font-black text-black/40 mb-2">預設縫份 (cm)</label>
                                                <input type="number" step="0.1" value={currentTemplate.seamAllowance} onChange={e => setCurrentTemplate({...currentTemplate, seamAllowance: e.target.value})} className="w-full bg-cyan-50 border-2 border-cyan-100 rounded-2xl p-4 text-center font-black text-cyan-600" />
                                            </div>
                                        </div>
                                    </div>
                                </div>

                                {/* 教學連結區塊 */}
                                <div className="bg-white p-8 rounded-[2.5rem] border border-sky-50 space-y-4">
                                    <div className="flex justify-between items-center">
                                        <h2 className="text-xl font-black flex items-center gap-2"><Icon name="link" className="text-sky-400"/> 教學連結</h2>
                                        <button onClick={() => setCurrentTemplate({...currentTemplate, links: [...(currentTemplate.links || []), {id: Date.now(), title:'', url:''}]})} className="text-xs font-black text-sky-500 bg-sky-50 px-4 py-2 rounded-xl">+ 新增連結</button>
                                    </div>
                                    {currentTemplate.links?.map((link, idx) => (
                                        <div key={link.id} className="flex gap-3 p-4 bg-sky-50/20 rounded-3xl items-start">
                                            <div className="flex-1 space-y-2">
                                                <input type="text" placeholder="教學標題" value={link.title} onChange={e => { const nl = [...currentTemplate.links]; nl[idx].title = e.target.value; setCurrentTemplate({...currentTemplate, links: nl}); }} className="w-full bg-white p-2 rounded-lg text-sm font-black outline-none" />
                                                <input type="text" placeholder="URL" value={link.url} onChange={e => { const nl = [...currentTemplate.links]; nl[idx].url = e.target.value; setCurrentTemplate({...currentTemplate, links: nl}); }} className="w-full bg-white/50 p-2 rounded-lg text-xs outline-none" />
                                            </div>
                                            <button onClick={() => setCurrentTemplate({...currentTemplate, links: currentTemplate.links.filter((_, i) => i !== idx)})} className="text-sky-200 hover:text-rose-500"><Icon name="trash-2" size={18} /></button>
                                        </div>
                                    ))}
                                </div>

                                {/* 零件清單 */}
                                <div className="space-y-4">
                                    <div className="flex justify-between items-center px-4">
                                        <h2 className="text-xl font-black flex items-center gap-2"><Icon name="ruler" className="text-sky-400"/> 零件配置</h2>
                                        <button onClick={() => setCurrentTemplate({...currentTemplate, components: [...currentTemplate.components, {id: Date.now(), name:'新零件', baseW:10, baseH:10, type:'both', quantity:1, includeSA:true}]})} className="bg-sky-400 text-white px-4 py-2 rounded-2xl font-black text-sm shadow-lg">+ 新增零件</button>
                                    </div>
                                    {currentTemplate.components.map((comp, idx) => (
                                        <div key={comp.id} className="bg-white p-6 rounded-[2rem] border border-sky-50 space-y-4">
                                            <div className="flex justify-between items-center">
                                                <input type="text" value={comp.name} onChange={e => { const nc = [...currentTemplate.components]; nc[idx].name = e.target.value; setCurrentTemplate({...currentTemplate, components: nc}); }} className="font-black text-xl bg-transparent border-b-2 border-transparent focus:border-sky-100 outline-none" />
                                                <label className="flex items-center gap-2 cursor-pointer">
                                                    <span className={`text-xs font-black ${comp.includeSA ? 'text-black':'text-slate-300'}`}>計算縫份</span>
                                                    <div onClick={() => { const nc = [...currentTemplate.components]; nc[idx].includeSA = !nc[idx].includeSA; setCurrentTemplate({...currentTemplate, components: nc}); }} className={`w-12 h-6 rounded-full relative transition-all ${comp.includeSA ? 'bg-sky-400':'bg-slate-200'}`}>
                                                        <div className={`absolute top-1 w-4 h-4 bg-white rounded-full shadow-sm transition-all ${comp.includeSA ? 'left-7':'left-1'}`} />
                                                    </div>
                                                </label>
                                            </div>
                                            <div className="grid grid-cols-4 gap-2">
                                                <input type="number" value={comp.baseW} onChange={e => { const nc = [...currentTemplate.components]; nc[idx].baseW = e.target.value; setCurrentTemplate({...currentTemplate, components: nc}); }} className="bg-sky-50/30 p-2 rounded-xl text-center font-black" />
                                                <input type="number" value={comp.baseH} onChange={e => { const nc = [...currentTemplate.components]; nc[idx].baseH = e.target.value; setCurrentTemplate({...currentTemplate, components: nc}); }} className="bg-sky-50/30 p-2 rounded-xl text-center font-black" />
                                                <input type="number" value={comp.quantity} onChange={e => { const nc = [...currentTemplate.components]; nc[idx].quantity = e.target.value; setCurrentTemplate({...currentTemplate, components: nc}); }} className="bg-sky-50/30 p-2 rounded-xl text-center font-black" />
                                                <select value={comp.type} onChange={e => { const nc = [...currentTemplate.components]; nc[idx].type = e.target.value; setCurrentTemplate({...currentTemplate, components: nc}); }} className="bg-sky-50/30 p-2 rounded-xl font-black text-xs">
                                                    {COMPONENT_TYPES.map(o => <option key={o.id} value={o.id}>{o.label}</option>)}
                                                </select>
                                            </div>
                                        </div>
                                    ))}
                                </div>

                                <div className="flex gap-4 pt-6">
                                    <button onClick={() => handleSaveTemplate(currentTemplate)} className="flex-1 bg-sky-400 text-white font-black py-5 rounded-[2rem] shadow-xl active:scale-95 transition-all">儲存模板</button>
                                    <button onClick={() => setActiveTab('list')} className="px-10 bg-slate-100 text-slate-500 font-black py-5 rounded-[2rem]">取消</button>
                                </div>
                            </div>
                        )}

                        {/* Calculate View */}
                        {activeTab === 'calculate' && currentTemplate && (
                            <div className="space-y-8 animate-in fade-in zoom-in-95 duration-500">
                                <div className="bg-gradient-to-br from-sky-300 via-sky-400 to-cyan-400 text-white p-8 rounded-[3rem] shadow-2xl space-y-8 relative overflow-hidden">
                                    <div className="absolute -right-10 -top-10 w-48 h-48 bg-white/10 rounded-full blur-2xl"></div>
                                    <h2 className="text-3xl font-black tracking-tight relative z-10">{currentTemplate.name}</h2>
                                    <div className="grid grid-cols-2 gap-6 relative z-10">
                                        <div className="space-y-2">
                                            <label className="text-[10px] font-black opacity-90 uppercase tracking-[0.3em] ml-2">寬度增加 (W)</label>
                                            <input type="number" value={scaleInput.addW} onChange={e => setScaleInput({...scaleInput, addW: e.target.value})} className="w-full bg-white/20 backdrop-blur-md border-2 border-white/30 rounded-[2rem] p-5 text-white font-black text-3xl outline-none" />
                                        </div>
                                        <div className="space-y-2">
                                            <label className="text-[10px] font-black opacity-90 uppercase tracking-[0.3em] ml-2">高度增加 (H)</label>
                                            <input type="number" value={scaleInput.addH} onChange={e => setScaleInput({...scaleInput, addH: e.target.value})} className="w-full bg-white/20 backdrop-blur-md border-2 border-white/30 rounded-[2rem] p-5 text-white font-black text-3xl outline-none" />
                                        </div>
                                    </div>
                                </div>

                                {/* Teaching Links in Calculate */}
                                {currentTemplate.links?.length > 0 && (
                                    <div className="space-y-3">
                                        <h3 className="font-black flex items-center gap-2 px-4 text-lg"><Icon name="book-open" className="text-sky-400"/> 教學資源</h3>
                                        <div className="grid gap-2">
                                            {currentTemplate.links.map(l => (
                                                <div key={l.id} className="bg-white p-5 rounded-[1.8rem] border border-sky-50 flex items-center justify-between shadow-sm">
                                                    <div className="flex-1 overflow-hidden mr-4">
                                                        <h4 className="font-black text-sm truncate">{l.title || '無標題'}</h4>
                                                        <p className="text-[10px] text-sky-300 font-bold truncate">{l.url}</p>
                                                    </div>
                                                    <button onClick={() => handleCopy(l.url, l.id)} className={`flex items-center gap-2 px-5 py-2.5 rounded-2xl text-[10px] font-black transition-all ${copyFeedback === l.id ? 'bg-emerald-500 text-white':'bg-sky-50 text-black border border-sky-100'}`}>
                                                        {copyFeedback === l.id ? '已複製' : '複製網址'}
                                                    </button>
                                                </div>
                                            ))}
                                        </div>
                                    </div>
                                )}

                                {/* Result List */}
                                <div className="space-y-4">
                                    <h3 className="font-black flex items-center gap-2 px-4 text-lg"><Icon name="scissors" className="text-sky-400"/> 裁切清單</h3>
                                    {currentTemplate.components.map(comp => {
                                        const res = calculate(comp, scaleInput.addW, scaleInput.addH, currentTemplate.seamAllowance);
                                        return (
                                            <div key={comp.id} className="bg-white p-6 rounded-[2.2rem] border border-sky-50 shadow-sm flex items-center gap-6">
                                                <div className="w-16 h-16 rounded-[1.2rem] bg-sky-400 text-white flex flex-col items-center justify-center font-black">
                                                    <span className="text-[10px] opacity-70">數量</span>
                                                    <span className="text-2xl">{comp.quantity}</span>
                                                </div>
                                                <div className="flex-1">
                                                    <div className="flex justify-between items-center mb-1">
                                                        <span className="font-black text-lg">{comp.name}</span>
                                                        <span className={`text-[9px] font-black uppercase px-2.5 py-1 rounded-full ${comp.includeSA ? 'bg-cyan-400 text-white':'bg-slate-100 text-slate-300'}`}>{comp.includeSA ? '含縫份':'淨尺寸'}</span>
                                                    </div>
                                                    <div className="flex items-baseline gap-1">
                                                        <span className="text-3xl font-black">{res.w} <span className="text-xl font-normal text-sky-200">×</span> {res.h}</span>
                                                        <span className="text-xs font-black text-sky-200 uppercase tracking-widest ml-1">cm</span>
                                                    </div>
                                                </div>
                                            </div>
                                        );
                                    })}
                                </div>

                                <button onClick={() => setActiveTab('list')} className="w-full py-5 text-sky-300 font-black bg-white/50 border-2 border-white rounded-[2rem] shadow-sm active:scale-95 mb-8">返回作品列表</button>
                            </div>
                        )}
                    </main>

                    {/* Bottom Nav */}
                    <nav className="fixed bottom-6 left-4 right-4 bg-white/80 backdrop-blur-xl border border-sky-50 p-3 flex justify-around items-center max-w-2xl mx-auto z-40 rounded-[2.5rem] bubble-shadow">
                        <button onClick={() => setActiveTab('list')} className={`flex flex-col items-center py-2 px-8 rounded-2xl transition-all ${activeTab === 'list' ? 'text-black bg-sky-50 font-black' : 'text-slate-300 font-bold'}`}>
                            <Icon name="home" size={22} className="mb-1" />
                            <span className="text-[10px]">作品列表</span>
                        </button>
                        <button onClick={startNew} className={`flex flex-col items-center py-2 px-8 rounded-2xl transition-all ${activeTab === 'edit' ? 'text-black bg-sky-50 font-black' : 'text-slate-300 font-bold'}`}>
                            <Icon name="plus" size={22} className="mb-1" />
                            <span className="text-[10px]">新增模板</span>
                        </button>
                        <button onClick={() => { if(currentTemplate) setActiveTab('calculate'); }} className={`flex flex-col items-center py-2 px-8 rounded-2xl transition-all ${activeTab === 'calculate' ? 'text-black bg-sky-50 font-black' : 'text-slate-300 font-bold'} ${!currentTemplate ? 'opacity-20 pointer-events-none':''}`}>
                            <Icon name="calculator" size={22} className="mb-1" />
                            <span className="text-[10px]">尺寸計算</span>
                        </button>
                    </nav>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
