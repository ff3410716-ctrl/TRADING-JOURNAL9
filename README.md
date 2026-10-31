# TRADING-JOURNAL9

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SHIRO Trading Journal</title>
    
    <!-- Tailwind CSS for Premium Styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- React and Babel for JSX -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- Chart.js for Analytics -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

    <script>
        tailwind.config = {
            theme: {
                extend: {
                    colors: {
                        terminal: {
                            bg: '#020617',       // Deep Navy
                            card: '#0f172a',     // Slate 900
                            border: '#1e293b',   // Subtle Border
                            accent: '#fbbf24',   // Gold
                            profit: '#22c55e',   // Green
                            loss: '#ef4444',     // Red
                            neutral: '#94a3b8',  // Grey
                        }
                    }
                }
            }
        }
    </script>
    <style>
        body { background-color: #020617; color: white; font-family: 'Inter', sans-serif; }
        .custom-scroll::-webkit-scrollbar { width: 6px; }
        .custom-scroll::-webkit-scrollbar-thumb { background: #1e293b; border-radius: 10px; }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        // --- DATABASE SERVICE (IndexedDB) ---
        const DB_NAME = 'ShiroTradingDB';
        const STORE_NAME = 'trades';

        const initDB = () => {
            return new Promise((resolve, reject) => {
                const request = indexedDB.open(DB_NAME, 1);
                request.onupgradeneeded = (e) => {
                    const db = e.target.result;
                    if (!db.objectStoreNames.contains(STORE_NAME)) {
                        db.createObjectStore(STORE_NAME, { keyPath: 'id', autoIncrement: true });
                    }
                };
                request.onsuccess = () => resolve(request.result);
                request.onerror = () => reject('DB Error');
            });
        };

        const tradeService = {
            async addTrade(trade) {
                const db = await initDB();
                const tx = db.transaction(STORE_NAME, 'readwrite');
                tx.objectStore(STORE_NAME).add(trade);
                return tx.complete;
            },
            async getAllTrades() {
                const db = await initDB();
                return new Promise((resolve) => {
                    const tx = db.transaction(STORE_NAME, 'readonly');
                    const request = tx.objectStore(STORE_NAME).getAll();
                    request.onsuccess = () => resolve(request.result);
                });
            },
            async deleteTrade(id) {
                const db = await initDB();
                const tx = db.transaction(STORE_NAME, 'readwrite');
                tx.objectStore(STORE_NAME).delete(id);
                return tx.complete;
            }
        };

        // --- COMPONENTS ---

        const NavItem = ({ id, label, icon, active, setActive }) => (
            <button 
                onClick={() => setActive(id)}
                className={`flex items-center gap-3 p-3 rounded-lg transition-all w-full text-left ${active === id ? 'bg-terminal-accent text-black font-bold' : 'text-terminal-neutral hover:bg-terminal-border'}`}
            >
                <span className="text-lg">{icon}</span> {label}
            </button>
        );

        const StatCard = ({ label, value, color = 'text-white' }) => (
            <div className="bg-terminal-card p-4 rounded-xl border border-terminal-border shadow-lg">
                <p className="text-xs text-terminal-neutral uppercase tracking-wider">{label}</p>
                <p className={`text-2xl font-bold ${color}`}>{value}</p>
            </div>
        );

        const Dashboard = ({ trades }) => {
            const totalTrades = trades.length;
            const winningTrades = trades.filter(t => t.result === 'PROFIT').length;
            const losingTrades = trades.filter(t => t.result === 'LOSS').length;
            const winRate = totalTrades ? ((winningTrades / totalTrades) * 100).toFixed(1) : 0;
            const netPnL = trades.reduce((acc, t) => acc + parseFloat(t.pnl || 0), 0).toFixed(2);

            return (
                <div className="space-y-6 animate-fade-in">
                    <h2 className="text-2xl font-bold text-terminal-accent">Market Overview</h2>
                    <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
                        <StatCard label="Total Trades" value={totalTrades} />
                        <StatCard label="Net P/L" value={`$${netPnL}`} color={netPnL >= 0 ? 'text-terminal-profit' : 'text-terminal-loss'} />
                        <StatCard label="Win Rate" value={`${winRate}%`} color="text-terminal-accent" />
                        <StatCard label="Wins/Losses" value={`${winningTrades}W / ${losingTrades}L`} />
                    </div>
                    <div className="bg-terminal-card p-6 rounded-xl border border-terminal-border h-64 flex items-center justify-center text-terminal-neutral italic">
                        Chart.js Visualizations Render Here
                    </div>
                </div>
            );
        };

        const AddTrade = ({ onTradeAdded }) => {
            const [formData, setFormData] = useState({
                instrument: 'XAUUSD', direction: 'BUY', lotSize: '', entryPrice: '', exitPrice: '', result: 'PROFIT', pnl: '', strategy: '', session: 'New York', notes: ''
            });

            const getPKTDate = () => {
                return new Intl.DateTimeFormat('en-GB', {
                    timeZone: 'Asia/Karachi',
                    dateStyle: 'long', timeStyle: 'short', weekday: 'long',
                }).format(new Date());
            };

            const handleSubmit = async (e) => {
                e.preventDefault();
                await tradeService.addTrade({ ...formData, timestamp: getPKTDate(), id: Date.now() });
                onTradeAdded();
                alert("Trade recorded successfully, ONI!");
            };

            return (
                <form onSubmit={handleSubmit} className="max-w-2xl mx-auto space-y-6">
                    <h2 className="text-2xl font-bold text-terminal-accent">New Entry <span className="text-sm font-normal text-terminal-neutral block">{getPKTDate()}</span></h2>
                    
                    <div className="bg-terminal-card p-6 rounded-xl border border-terminal-border grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div>
                            <label className="block text-xs text-terminal-neutral mb-1">Instrument</label>
                            <select className="w-full bg-terminal-bg border border-terminal-border p-2 rounded" onChange={e => setFormData({...formData, instrument: e.target.value})}>
                                <option>XAUUSD</option><option>EURUSD</option><option>GBPUSD</option><option>BTCUSD</option><option>Other</option>
                            </select>
                        </div>
                        <div>
                            <label className="block text-xs text-terminal-neutral mb-1">Direction</label>
                            <div className="flex gap-2">
                                <button type="button" onClick={() => setFormData({...formData, direction: 'BUY'})} className={`flex-1 p-2 rounded ${formData.direction === 'BUY' ? 'bg-terminal-profit' : 'bg-terminal-bg border border-terminal-border'}`}>BUY</button>
                                <button type="button" onClick={() => setFormData({...formData, direction: 'SELL'})} className={`flex-1 p-2 rounded ${formData.direction === 'SELL' ? 'bg-terminal-loss' : 'bg-terminal-bg border border-terminal-border'}`}>SELL</button>
                            </div>
                        </div>
                    </div>

                    <div className="bg-terminal-card p-6 rounded-xl border border-terminal-border grid grid-cols-2 md:grid-cols-4 gap-4">
                        <Input label="Lot Size" onChange={v => setFormData({...formData, lotSize: v})} />
                        <Input label="Entry" onChange={v => setFormData({...formData, entryPrice: v})} />
                        <Input label="Exit" onChange={v => setFormData({...formData, exitPrice: v})} />
                        <Input label="P/L USD" onChange={v => setFormData({...formData, pnl: v})} />
                    </div>

                    <div className="bg-terminal-card p-6 rounded-xl border border-terminal-border">
                        <label className="block text-xs text-terminal-neutral mb-1">Trading Notes / Lessons</label>
                        <textarea className="w-full bg-terminal-bg border border-terminal-border p-2 rounded h-24" placeholder="What did you learn from this trade?" onChange={e => setFormData({...formData, notes: e.target.value})}></textarea>
                    </div>

                    <button type="submit" className="w-full bg-terminal-accent text-black font-bold p-4 rounded-xl hover:scale-[1.02] transition-transform">SAVE TRADE</button>
                </form>
            );
        };

        const Input = ({ label, onChange }) => (
            <div>
                <label className="block text-xs text-terminal-neutral mb-1">{label}</label>
                <input type="number" step="any" className="w-full bg-terminal-bg border border-terminal-border p-2 rounded" onChange={e => onChange(e.target.value)} />
            </div>
        );

        const TradeHistory = ({ trades, onTradeDeleted }) => (
            <div className="space-y-6">
                <h2 className="text-2xl font-bold text-terminal-accent">Trade History</h2>
                <div className="overflow-x-auto bg-terminal-card rounded-xl border border-terminal-border">
                    <table className="w-full text-left text-sm">
                        <thead className="bg-terminal-border text-terminal-neutral">
                            <tr>
                                <th className="p-4">Date</th>
                                <th className="p-4">Instrument</th>
                                <th className="p-4">Type</th>
                                <th className="p-4">P/L</th>
                                <th className="p-4">Action</th>
                            </tr>
                        </thead>
                        <tbody>
                            {trades.map(t => (
                                <tr key={t.id} className="border-t border-terminal-border hover:bg-terminal-bg">
                                    <td className="p-4 text-xs">{t.timestamp}</td>
                                    <td className="p-4 font-bold">{t.instrument}</td>
                                    <td className={`p-4 ${t.direction === 'BUY' ? 'text-terminal-profit' : 'text-terminal-loss'}`}>{t.direction}</td>
                                    <td className={`p-4 font-bold ${t.pnl >= 0 ? 'text-terminal-profit' : 'text-terminal-loss'}`}>${t.pnl}</td>
                                    <td className="p-4">
                                        <button onClick={() => tradeService.deleteTrade(t.id).then(onTradeDeleted)} className="text-terminal-loss hover:underline">Delete</button>
                                    </td>
                                </tr>
                            ))}
                            {trades.length === 0 && <tr><td colSpan="5" className="p-10 text-center text-terminal-neutral">No trades recorded yet.</td></tr>}
                        </tbody>
                    </table>
                </div>
            </div>
        );

        // --- MAIN APP ---

        const App = () => {
            const [activeTab, setActiveTab] = useState('dashboard');
            const [trades, setTrades] = useState([]);

            useEffect(() => { loadTrades(); }, []);
            const loadTrades = async () => {
                const data = await tradeService.getAllTrades();
                setTrades(data);
            };

            return (
                <div className="min-h-screen flex flex-col md:flex-row">
                    {/* Sidebar */}
                    <nav className="hidden md:flex w-64 bg-terminal-card border-r border-terminal-border flex-col p-6 gap-2 h-screen sticky top-0">
                        <h1 className="text-terminal-accent font-black text-2xl mb-10 tracking-tighter">SHIRO.AI</h1>
                        <NavItem id="dashboard" label="Dashboard" icon="📊" active={activeTab} setActive={setActiveTab} />
                        <NavItem id="add" label="Add Trade" icon="➕" active={activeTab} setActive={setActiveTab} />
                        <NavItem id="history" label="History" icon="📜" active={activeTab} setActive={setActiveTab} />
                        <NavItem id="settings" label="Settings" icon="⚙️" active={activeTab} setActive={setActiveTab} />
                    </nav>

                    {/* Main Content */}
                    <main className="flex-1 p-6 md:p-12 pb-24 overflow-y-auto custom-scroll">
                        {activeTab === 'dashboard' && <Dashboard trades={trades} />}
                        {activeTab === 'add' && <AddTrade onTradeAdded={loadTrades} />}
                        {activeTab === 'history' && <TradeHistory trades={trades} onTradeDeleted={loadTrades} />}
                        {activeTab === 'settings' && <div className="text-center p-20 text-terminal-neutral">Settings coming soon, ONI.</div>}
                    </main>

                    {/* Mobile Nav */}
                    <nav className="md:hidden fixed bottom-0 left-0 right-0 bg-terminal-card border-t border-terminal-border flex justify-around p-4">
                        <button onClick={() => setActiveTab('dashboard')} className={`text-2xl ${activeTab === 'dashboard' ? 'text-terminal-accent' : 'text-terminal-neutral'}`}>📊</button>
                        <button onClick={() => setActiveTab('add')} className={`text-2xl ${activeTab === 'add' ? 'text-terminal-accent' : 'text-terminal-neutral'}`}>➕</button>
