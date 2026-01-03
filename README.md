import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, collection, addDoc, updateDoc, deleteDoc, doc, 
  onSnapshot, query, orderBy, serverTimestamp 
} from 'firebase/firestore';
import { getAuth, signInAnonymously, onAuthStateChanged } from 'firebase/auth';
import { 
  Plus, Calendar, CheckCircle, AlertCircle, DollarSign, 
  User, Briefcase, FileText, Clock, Save, Trash2, Menu, X, 
  CreditCard, PieChart, Activity, Building2, HardHat
} from 'lucide-react';

// --- Firebase 配置 ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'construction-payment-v1';

// --- 狀態定義 ---
const STATUS_CONFIG = {
  draft: { label: '草稿', color: 'bg-gray-100 text-gray-700', icon: FileText },
  issue: { label: '缺資料', color: 'bg-red-100 text-red-700', icon: AlertCircle },
  planned: { label: '預計匯款', color: 'bg-yellow-100 text-yellow-800', icon: Clock },
  confirmed: { label: '會計已核', color: 'bg-blue-100 text-blue-700', icon: CheckCircle },
  paid: { label: '已匯款', color: 'bg-green-100 text-green-700', icon: DollarSign },
};

const VENDOR_TYPES = ['木工', '水電', '油漆', '泥作', '空調', '系統櫃', '地板', '清潔', '拆除', '其他'];

// --- 格式化工具 ---
const formatCurrency = (num) => new Intl.NumberFormat('zh-TW', { style: 'currency', currency: 'TWD', minimumFractionDigits: 0 }).format(num || 0);

export default function App() {
  const [user, setUser] = useState(null);
  const [currentView, setCurrentView] = useState('dashboard');
  const [sidebarOpen, setSidebarOpen] = useState(false);
  const [isLoading, setIsLoading] = useState(true);

  // 資料狀態
  const [projects, setProjects] = useState([]);
  const [vendors, setVendors] = useState([]);
  const [payments, setPayments] = useState([]);

  // 1. 初始化身分驗證 (匿名登入) - 必須先完成登入才能進行後續操作
  useEffect(() => {
    const initAuth = async () => {
      try {
        // 確保身分驗證完成
        await signInAnonymously(auth);
      } catch (error) {
        console.error("Auth Error:", error);
      }
    };
    initAuth();
    
    // 監聽登入狀態變化
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      if (currentUser) {
        setUser(currentUser);
      }
    });
    return () => unsubscribe();
  }, []);

  // 2. 實時監聽 Firestore 資料 - 嚴格檢查 user 是否存在
  useEffect(() => {
    // 關鍵修正：若 user 尚未登入完成，不可執行查詢，否則會報 Missing Permissions 錯誤
    if (!user) return;

    let unsubs = [];

    try {
      // 案場資料
      const qProjects = query(collection(db, 'artifacts', appId, 'public', 'data', 'projects'), orderBy('createdAt', 'desc'));
      const unsubP = onSnapshot(qProjects, 
        (s) => setProjects(s.docs.map(d => ({id: d.id, ...d.data()}))), 
        (e) => console.error("Projects Listener Error:", e)
      );
      unsubs.push(unsubP);

      // 工班資料
      const qVendors = query(collection(db, 'artifacts', appId, 'public', 'data', 'vendors'), orderBy('createdAt', 'desc'));
      const unsubV = onSnapshot(qVendors, 
        (s) => setVendors(s.docs.map(d => ({id: d.id, ...d.data()}))), 
        (e) => console.error("Vendors Listener Error:", e)
      );
      unsubs.push(unsubV);

      // 請款資料
      const qPayments = query(collection(db, 'artifacts', appId, 'public', 'data', 'payments'), orderBy('expectedDate', 'desc'));
      const unsubPay = onSnapshot(qPayments, 
        (s) => {
          setPayments(s.docs.map(d => ({id: d.id, ...d.data()})));
          setIsLoading(false);
        }, 
        (e) => {
          console.error("Payments Listener Error:", e);
          setIsLoading(false);
        }
      );
      unsubs.push(unsubPay);
    } catch (err) {
      console.error("Setup Snapshot Error:", err);
    }

    return () => unsubs.forEach(unsub => unsub());
  }, [user]);

  // 3. 資料庫操作函數
  const handleSavePayment = async (p) => {
    if(!user) return;
    const { id, ...data } = p;
    try {
      if (id) {
        await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'payments', id), { ...data, updatedAt: serverTimestamp() });
      } else {
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'payments'), { ...data, createdAt: serverTimestamp(), updatedAt: serverTimestamp() });
      }
    } catch (e) { console.error("Save Payment Error:", e); }
  };

  const handleDeletePayment = async (id) => {
    if(!user || !window.confirm('確定要刪除這筆請款單嗎？')) return;
    try {
      await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'payments', id));
    } catch (e) { console.error("Delete Payment Error:", e); }
  };

  const handleSaveProject = async (p) => {
    if(!user) return;
    const { id, ...data } = p;
    try {
      if (id) {
        await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'projects', id), data);
      } else {
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'projects'), { ...data, createdAt: serverTimestamp() });
      }
    } catch (e) { console.error("Save Project Error:", e); }
  };

  const handleSaveVendor = async (v) => {
    if(!user) return;
    const { id, ...data } = v;
    try {
      if (id) {
        await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'vendors', id), data);
      } else {
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'vendors'), { ...data, createdAt: serverTimestamp() });
      }
    } catch (e) { console.error("Save Vendor Error:", e); }
  };

  const renderContent = () => {
    if (!user || isLoading) return <div className="flex h-full items-center justify-center text-gray-400">連線雲端資料庫中...</div>;
    switch (currentView) {
      case 'dashboard': return <Dashboard payments={payments} projects={projects} setView={setCurrentView} />;
      case 'payments': return <PaymentView payments={payments} projects={projects} vendors={vendors} onSave={handleSavePayment} onDelete={handleDeletePayment} />;
      case 'vendors': return <VendorView vendors={vendors} onSave={handleSaveVendor} />;
      case 'projects': return <ProjectView projects={projects} onSave={handleSaveProject} />;
      default: return <Dashboard />;
    }
  };

  return (
    <div className="flex h-screen bg-gray-50 font-sans text-slate-800">
      {/* 響應式側邊欄 */}
      <aside className={`fixed inset-y-0 left-0 z-50 w-64 bg-slate-900 text-white shadow-xl transform transition-transform duration-300 ease-in-out ${sidebarOpen ? 'translate-x-0' : '-translate-x-full'} md:relative md:translate-x-0`}>
        <div className="p-6 border-b border-slate-800 flex justify-between items-center">
          <h1 className="text-xl font-bold text-white flex items-center gap-2">工班請款系統</h1>
          <button onClick={() => setSidebarOpen(false)} className="md:hidden text-gray-400 p-2"><X size={24} /></button>
        </div>
        <nav className="p-4 space-y-1">
          <NavButton active={currentView === 'dashboard'} onClick={() => { setCurrentView('dashboard'); setSidebarOpen(false); }} icon={Activity}>營運總覽</NavButton>
          <NavButton active={currentView === 'payments'} onClick={() => { setCurrentView('payments'); setSidebarOpen(false); }} icon={DollarSign}>請款單管理</NavButton>
          <NavButton active={currentView === 'vendors'} onClick={() => { setCurrentView('vendors'); setSidebarOpen(false); }} icon={HardHat}>工班清冊</NavButton>
          <NavButton active={currentView === 'projects'} onClick={() => { setCurrentView('projects'); setSidebarOpen(false); }} icon={Building2}>案場清單</NavButton>
        </nav>
      </aside>

      {/* 主內容區 */}
      <main className="flex-1 overflow-y-auto flex flex-col">
        <header className="bg-white shadow-sm p-4 flex justify-between items-center sticky top-0 z-10 border-b md:px-8">
          <div className="flex items-center gap-4">
            <button onClick={() => setSidebarOpen(true)} className="md:hidden text-gray-600 p-2 -ml-2"><Menu size={24} /></button>
            <h2 className="text-lg font-bold text-gray-800">
              {currentView === 'dashboard' ? '營運總覽' : currentView === 'payments' ? '請款單管理' : currentView === 'vendors' ? '工班管理' : '案場管理'}
            </h2>
          </div>
          <div className="text-[10px] text-gray-400 bg-gray-50 px-2 py-1 rounded">連線 ID: {user?.uid?.slice(0, 6) || 'Connecting...'}</div>
        </header>
        <div className="p-4 md:p-8 max-w-7xl mx-auto w-full">{renderContent()}</div>
      </main>
    </div>
  );
}

function NavButton({ children, active, onClick, icon: Icon }) {
  return (
    <button onClick={onClick} className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl transition-all ${active ? 'bg-blue-600 text-white font-bold shadow-lg shadow-blue-900/20' : 'text-slate-400 hover:text-white hover:bg-slate-800'}`}>
      <Icon size={20} />
      {children}
    </button>
  );
}

// --- Dashboard ---
function Dashboard({ payments, projects, setView }) {
  const stats = useMemo(() => {
    const totalPending = payments.filter(p => p.status !== 'paid').reduce((s, p) => s + Number(p.amount || 0), 0);
    const paidThisMonth = payments.filter(p => p.status === 'paid').reduce((s, p) => s + Number(p.amount || 0), 0);
    return { totalPending, paidThisMonth };
  }, [payments]);

  return (
    <div className="space-y-6 animate-in fade-in duration-500">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div className="bg-white p-6 rounded-2xl shadow-sm border border-gray-100">
          <h3 className="text-sm font-bold text-gray-400 mb-1">未結請款總額</h3>
          <p className="text-3xl font-black text-slate-900">{formatCurrency(stats.totalPending)}</p>
        </div>
        <div className="bg-blue-600 p-6 rounded-2xl shadow-lg shadow-blue-200 text-white">
          <h3 className="text-sm font-bold opacity-80 mb-1">本月累計已付</h3>
          <p className="text-3xl font-black">{formatCurrency(stats.paidThisMonth)}</p>
        </div>
      </div>
      
      <div className="bg-white rounded-2xl shadow-sm border overflow-hidden">
        <div className="p-6 border-b flex justify-between items-center">
          <h3 className="font-bold text-gray-800">近期待匯項目</h3>
          <button onClick={() => setView('payments')} className="text-xs font-bold text-blue-600 hover:underline">查看完整列表</button>
        </div>
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead className="bg-gray-50 text-gray-400 font-bold uppercase text-[10px] tracking-wider border-b">
              <tr>
                <th className="px-6 py-4 text-left">日期</th>
                <th className="px-6 py-4 text-left">對象/案場</th>
                <th className="px-6 py-4 text-right">金額</th>
              </tr>
            </thead>
            <tbody className="divide-y divide-gray-100">
              {payments.filter(p => p.status !== 'paid').slice(0, 5).map(p => (
                <tr key={p.id} className="hover:bg-gray-50 transition-colors">
                  <td className="px-6 py-4 text-gray-500">{p.expectedDate}</td>
                  <td className="px-6 py-4">
                    <div className="font-bold text-slate-800">{p.vendorName}</div>
                    <div className="text-[10px] text-gray-400">{p.projectName}</div>
                  </td>
                  <td className="px-6 py-4 text-right font-bold text-blue-600">{formatCurrency(p.amount)}</td>
                </tr>
              ))}
              {payments.filter(p => p.status !== 'paid').length === 0 && (
                <tr>
                  <td colSpan="3" className="px-6 py-8 text-center text-gray-400 italic">目前無待匯款項</td>
                </tr>
              )}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  );
}

// --- Payment Management ---
function PaymentView({ payments, projects, vendors, onSave, onDelete }) {
  const [modalOpen, setModalOpen] = useState(false);
  const [editing, setEditing] = useState(null);

  const openModal = (p = null) => { setEditing(p); setModalOpen(true); };

  return (
    <div className="space-y-4 animate-in fade-in duration-500">
      <div className="flex justify-between items-center">
        <h3 className="font-bold text-xl">請款清單</h3>
        <button onClick={() => openModal()} className="bg-slate-900 text-white px-4 py-2 rounded-xl flex items-center gap-2 text-sm font-bold shadow-lg hover:bg-slate-800 transition-colors"><Plus size={18}/> 新增請款</button>
      </div>
      
      <div className="grid grid-cols-1 gap-3">
        {payments.map(p => (
          <div key={p.id} onClick={() => openModal(p)} className="bg-white p-4 rounded-xl shadow-sm border border-gray-100 flex items-center justify-between cursor-pointer hover:border-blue-300 hover:shadow-md transition-all">
            <div className="flex-1 min-w-0">
              <div className="flex items-center gap-2 mb-1">
                <span className={`px-2 py-0.5 rounded text-[10px] font-bold ${STATUS_CONFIG[p.status]?.color}`}>
                  {STATUS_CONFIG[p.status]?.label}
                </span>
                <span className="text-xs text-gray-400">{p.expectedDate}</span>
              </div>
              <h4 className="font-bold truncate text-slate-800">{p.vendorName}</h4>
              <p className="text-[10px] text-gray-400 truncate">{p.projectName} - {p.item}</p>
            </div>
            <div className="text-right ml-4">
              <div className="font-black text-slate-800">{formatCurrency(p.amount)}</div>
            </div>
          </div>
        ))}
        {payments.length === 0 && <div className="text-center py-20 text-gray-400 bg-white rounded-2xl border border-dashed">尚無請款記錄</div>}
      </div>

      {modalOpen && <PaymentModal item={editing} onClose={() => setModalOpen(false)} onSave={onSave} onDelete={onDelete} projects={projects} vendors={vendors} />}
    </div>
  );
}

function PaymentModal({ item, onClose, onSave, onDelete, projects, vendors }) {
  const [formData, setFormData] = useState({
    projectId: '', projectName: '', vendorId: '', vendorName: '',
    amount: '', expectedDate: '', item: '', status: 'draft',
    ...item
  });

  return (
    <div className="fixed inset-0 z-[60] flex items-center justify-center bg-slate-900/60 backdrop-blur-sm p-4 animate-in fade-in zoom-in duration-200">
      <div className="bg-white rounded-3xl shadow-2xl w-full max-w-md overflow-hidden flex flex-col max-h-[90vh]">
        <div className="p-6 border-b flex justify-between items-center">
          <h3 className="font-bold text-lg">{item ? '編輯請款' : '新增請款'}</h3>
          <button onClick={onClose} className="p-2 hover:bg-gray-100 rounded-full transition-colors"><X size={24}/></button>
        </div>
        <div className="p-6 overflow-y-auto space-y-4">
          <div>
            <label className="block text-xs font-bold text-gray-400 mb-1 uppercase tracking-wider">案場</label>
            <select className="w-full border-2 border-gray-100 p-3 rounded-xl bg-gray-50 focus:border-blue-500 outline-none transition-all" value={formData.projectId} onChange={e => {
              const p = projects.find(x => x.id === e.target.value);
              setFormData({...formData, projectId: e.target.value, projectName: p?.name || ''})
            }}>
              <option value="">選擇案場...</option>
              {projects.map(p => <option key={p.id} value={p.id}>{p.name}</option>)}
            </select>
          </div>
          <div>
            <label className="block text-xs font-bold text-gray-400 mb-1 uppercase tracking-wider">工班</label>
            <select className="w-full border-2 border-gray-100 p-3 rounded-xl bg-gray-50 focus:border-blue-500 outline-none transition-all" value={formData.vendorId} onChange={e => {
              const v = vendors.find(x => x.id === e.target.value);
              setFormData({...formData, vendorId: e.target.value, vendorName: v?.name || ''})
            }}>
              <option value="">選擇工班...</option>
              {vendors.map(v => <option key={v.id} value={v.id}>{v.name}</option>)}
            </select>
          </div>
          <div>
            <label className="block text-xs font-bold text-gray-400 mb-1 uppercase tracking-wider">項目內容</label>
            <input className="w-full border-2 border-gray-100 p-3 rounded-xl bg-gray-50 focus:border-blue-500 outline-none transition-all" placeholder="例如：木工第一期款" value={formData.item} onChange={e => setFormData({...formData, item: e.target.value})} />
          </div>
          <div className="grid grid-cols-2 gap-4">
            <div>
              <label className="block text-xs font-bold text-gray-400 mb-1 uppercase tracking-wider">金額</label>
              <input type="number" className="w-full border-2 border-gray-100 p-3 rounded-xl font-bold focus:border-blue-500 outline-none transition-all" value={formData.amount} onChange={e => setFormData({...formData, amount: e.target.value})} />
            </div>
            <div>
              <label className="block text-xs font-bold text-gray-400 mb-1 uppercase tracking-wider">預計日</label>
              <input type="date" className="w-full border-2 border-gray-100 p-3 rounded-xl focus:border-blue-500 outline-none transition-all" value={formData.expectedDate} onChange={e => setFormData({...formData, expectedDate: e.target.value})} />
            </div>
          </div>
          <div>
            <label className="block text-xs font-bold text-gray-400 mb-1 uppercase tracking-wider">狀態</label>
            <select className="w-full border-2 border-gray-100 p-3 rounded-xl bg-gray-50 font-bold focus:border-blue-500 outline-none transition-all" value={formData.status} onChange={e => setFormData({...formData, status: e.target.value})}>
              {Object.entries(STATUS_CONFIG).map(([key, val]) => <option key={key} value={key}>{val.label}</option>)}
            </select>
          </div>
        </div>
        <div className="p-6 bg-gray-50 border-t flex gap-3">
          {item && <button onClick={() => { handleDeletePayment(item.id); onClose(); }} className="p-3 text-red-500 hover:bg-red-50 rounded-xl transition-all"><Trash2 size={24}/></button>}
          <button onClick={onClose} className="flex-1 py-3 text-gray-500 font-bold">取消</button>
          <button onClick={() => { onSave(formData); onClose(); }} className="flex-1 py-3 bg-blue-600 text-white rounded-xl font-bold shadow-lg shadow-blue-200 hover:bg-blue-700 transition-all">儲存資料</button>
        </div>
      </div>
    </div>
  );
}

// --- Project & Vendor Management ---
function ProjectView({ projects, onSave }) {
  const [name, setName] = useState('');
  return (
    <div className="space-y-6 animate-in fade-in duration-500">
      <div className="bg-white p-6 rounded-2xl border shadow-sm">
        <h3 className="font-bold mb-4 text-gray-800">新增案場</h3>
        <div className="flex gap-2">
          <input className="flex-1 border-2 border-gray-100 p-3 rounded-xl bg-gray-50 focus:border-blue-500 outline-none transition-all" placeholder="輸入案場名稱..." value={name} onChange={e => setName(e.target.value)} />
          <button onClick={() => { if(name) { onSave({name}); setName(''); }}} className="bg-blue-600 text-white px-6 py-3 rounded-xl font-bold shadow-lg hover:bg-blue-700 transition-all">新增</button>
        </div>
      </div>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {projects.map(p => (
          <div key={p.id} className="bg-white p-5 rounded-2xl border border-gray-100 shadow-sm flex items-center justify-between hover:border-blue-200 transition-all">
            <span className="font-bold text-slate-700">{p.name}</span>
            <button onClick={() => { if(window.confirm('確定刪除案場？')) deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'projects', p.id)) }} className="text-gray-300 hover:text-red-500 p-2 transition-colors"><Trash2 size={18}/></button>
          </div>
        ))}
        {projects.length === 0 && <div className="col-span-full py-20 text-center text-gray-400 italic">目前無案場資料</div>}
      </div>
    </div>
  );
}

function VendorView({ vendors, onSave }) {
  const [vendorData, setVendorData] = useState({ name: '', type: '木工' });
  return (
    <div className="space-y-6 animate-in fade-in duration-500">
      <div className="bg-white p-6 rounded-2xl border shadow-sm">
        <h3 className="font-bold mb-4 text-gray-800">新增工班</h3>
        <div className="grid grid-cols-1 md:grid-cols-3 gap-3">
          <input className="border-2 border-gray-100 p-3 rounded-xl bg-gray-50 focus:border-blue-500 outline-none transition-all" placeholder="工班名稱..." value={vendorData.name} onChange={e => setVendorData({...vendorData, name: e.target.value})} />
          <select className="border-2 border-gray-100 p-3 rounded-xl bg-gray-50 focus:border-blue-500 outline-none transition-all font-bold" value={vendorData.type} onChange={e => setVendorData({...vendorData, type: e.target.value})}>
            {VENDOR_TYPES.map(t => <option key={t} value={t}>{t}</option>)}
          </select>
          <button onClick={() => { if(vendorData.name) { onSave(vendorData); setVendorData({name: '', type: '木工'}); }}} className="bg-blue-600 text-white py-3 rounded-xl font-bold shadow-lg hover:bg-blue-700 transition-all">新增工班</button>
        </div>
      </div>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {vendors.map(v => (
          <div key={v.id} className="bg-white p-5 rounded-2xl border border-gray-100 shadow-sm hover:border-blue-200 transition-all">
            <div className="flex justify-between items-start mb-2">
              <span className="font-black text-lg text-slate-800">{v.name}</span>
              <span className="text-[10px] bg-blue-50 px-2 py-0.5 rounded font-bold text-blue-600 uppercase tracking-tighter">{v.type}</span>
            </div>
            <div className="flex justify-end mt-4 border-t pt-2 border-gray-50">
              <button onClick={() => { if(window.confirm('確定刪除工班？')) deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'vendors', v.id)) }} className="text-gray-200 hover:text-red-500 p-2 transition-colors"><Trash2 size={18}/></button>
            </div>
          </div>
        ))}
        {vendors.length === 0 && <div className="col-span-full py-20 text-center text-gray-400 italic">目前無工班資料</div>}
      </div>
    </div>
  );
}
