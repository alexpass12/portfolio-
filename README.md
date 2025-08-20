import React, { useEffect, useMemo, useState } from "react";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, BarChart, Bar, Legend, PieChart, Pie, Cell } from "recharts";
import { Plus, Download, Upload, RefreshCw, Trash2, TrendingUp, DollarSign, PiggyBank, Car, Shield, Coins } from "lucide-react";

/**
 * Weekly Portfolio Live Dashboard
 * - Tracks WellsTrade positions (stocks + ETFs), 529 plan, car loan, and cash
 * - Lets you add weekly snapshots and see charts over time
 * - Data persists to localStorage (no sign-in needed)
 * - Import/Export JSON to back up your data
 *
 * Preloaded with your 08/20/2025 snapshot — edit anything and click “Save snapshot”.
 */

const STORAGE_KEY = "jp_weekly_dashboard_v1";

// ---- Types ----
/** @typedef {{ symbol: string, name: string, type: "Stock"|"ETF", shares: number, costBasis: number, lastPrice: number, marketValue: number, todaysChange: number, unrealized: number, estIncome: number }} Position */
/** @typedef {{ date: string, wellstradeTotal: number, wellstradeUnrealized: number, wellstradeIncome: number, stocksTotal: number, etfsTotal: number, cash: number, notes?: string }} WellsTradeSnapshot */
/** @typedef {{ date: string, total: number, principal: number, earnings: number }} TRoweSnapshot */
/** @typedef {{ date: string, balance: number, payoffAmount?: number, rateAPR?: number, paymentDue?: number }} CarLoanSnapshot */
/** @typedef {{ date: string, cashChecking: number, cashSavings: number }} CashSnapshot */
/** @typedef {{ positions: Position[], wellsSnapshots: WellsTradeSnapshot[], troweSnapshots: TRoweSnapshot[], carSnapshots: CarLoanSnapshot[], cashSnapshots: CashSnapshot[] }} DashboardData */

// ---- Initial data (from your 08/20/2025 messages) ----
const initialPositions /** @type {Position[]} */ = [
  // Stocks
  { symbol: "CRBG", name: "COREBRIDGE FINANCIAL INC", type: "Stock", shares: 5.073, costBasis: 30.70, lastPrice: 34.20, marketValue: 173.50, todaysChange: 0.36, unrealized: 17.74, estIncome: 4.87 },
  { symbol: "UNH", name: "UNITEDHEALTH GROUP", type: "Stock", shares: 0.53455, costBasis: 285.04, lastPrice: 299.84, marketValue: 160.28, todaysChange: -2.35, unrealized: 7.91, estIncome: 4.73 },
  { symbol: "FAST", name: "FASTENAL CO", type: "Stock", shares: 3, costBasis: 48.12, lastPrice: 49.54, marketValue: 148.62, todaysChange: -1.08, unrealized: 4.27, estIncome: 2.64 },
  { symbol: "ZURVY", name: "ZURICH INSURANCE GROUP", type: "Stock", shares: 4, costBasis: 35.20, lastPrice: 37.08, marketValue: 148.32, todaysChange: 2.80, unrealized: 7.52, estIncome: 5.65 },
  { symbol: "KRMN", name: "KARMAN HOLDINGS INC", type: "Stock", shares: 2, costBasis: 52.96, lastPrice: 50.76, marketValue: 101.52, todaysChange: 1.78, unrealized: -4.40, estIncome: 0 },
  { symbol: "KO", name: "COCA-COLA COMPANY", type: "Stock", shares: 1.30274, costBasis: 69.09, lastPrice: 70.70, marketValue: 92.10, todaysChange: 0.74, unrealized: 2.10, estIncome: 2.66 },
  { symbol: "WFC", name: "WELLS FARGO & CO NEW", type: "Stock", shares: 1, costBasis: 80.72, lastPrice: 78.16, marketValue: 78.16, todaysChange: 0.63, unrealized: -2.56, estIncome: 1.80 },
  { symbol: "VWAGY", name: "VOLKSWAGEN AG ADR", type: "Stock", shares: 5, costBasis: 11.62, lastPrice: 12.02, marketValue: 60.10, todaysChange: 0.55, unrealized: 2.02, estIncome: 2.27 },
  { symbol: "NVDA", name: "NVIDIA CORP", type: "Stock", shares: 0.34165, costBasis: 178.55, lastPrice: 175.40, marketValue: 59.93, todaysChange: -0.08, unrealized: -1.07, estIncome: 0.01 },
  { symbol: "NYMT", name: "NEW YORK MTG TR INC", type: "Stock", shares: 8.175, costBasis: 6.02, lastPrice: 7.04, marketValue: 57.55, todaysChange: 0.98, unrealized: 8.34, estIncome: 6.54 },
  { symbol: "VZ", name: "VERIZON COMMUNICATIONS", type: "Stock", shares: 1.032, costBasis: 39.52, lastPrice: 45.06, marketValue: 46.50, todaysChange: 0.12, unrealized: 5.72, estIncome: 2.80 },
  // ETFs
  { symbol: "QQQM", name: "INVESCO NASDAQ 100 ETF", type: "ETF", shares: 2.86419, costBasis: 190.51, lastPrice: 232.97, marketValue: 667.27, todaysChange: -3.98, unrealized: 121.62, estIncome: 3.56 },
  { symbol: "VOO", name: "VANGUARD INDX S&P500 ETF", type: "ETF", shares: 1.05157, costBasis: 577.23, lastPrice: 586.58, marketValue: 616.83, todaysChange: -1.63, unrealized: 9.83, estIncome: 7.29 },
  { symbol: "SPMO", name: "INVESCO S&P 500 MOM ETF", type: "ETF", shares: 2, costBasis: 114.92, lastPrice: 115.52, marketValue: 231.04, todaysChange: -0.34, unrealized: 1.21, estIncome: 1.31 },
  { symbol: "AAAU", name: "GOLDMAN SACHS PHYS GOLD ETF", type: "ETF", shares: 4.31728, costBasis: 26.41, lastPrice: 33.07, marketValue: 142.77, todaysChange: 1.42, unrealized: 28.77, estIncome: 0 },
  { symbol: "IBIT", name: "ISHARES BITCOIN TR ETF", type: "ETF", shares: 2, costBasis: 67.20, lastPrice: 64.90, marketValue: 129.80, todaysChange: 1.42, unrealized: -4.60, estIncome: 0 },
  { symbol: "IAU", name: "ISHARES GOLD TR ETF", type: "ETF", shares: 2, costBasis: 62.98, lastPrice: 63.13, marketValue: 126.26, todaysChange: 1.28, unrealized: 0.31, estIncome: 0 },
  { symbol: "FSZ", name: "FIRST TR SWITZERLAND ETF", type: "ETF", shares: 1, costBasis: 78.97, lastPrice: 79.59, marketValue: 79.59, todaysChange: 0.62, unrealized: 0.62, estIncome: 1.28 },
  { symbol: "SCHD", name: "SCHWAB US DIVIDEND ETF", type: "ETF", shares: 2.05073, costBasis: 26.15, lastPrice: 27.51, marketValue: 56.42, todaysChange: 0.12, unrealized: 2.80, estIncome: 2.10 },
  { symbol: "EWL", name: "ISHARES MSCI SWITZ ETF", type: "ETF", shares: 1, costBasis: 54.37, lastPrice: 55.52, marketValue: 55.52, todaysChange: 0.62, unrealized: 1.15, estIncome: 1.02 },
  { symbol: "VTI", name: "VANGUARD TOTAL STK MKT ETF", type: "ETF", shares: 0.11471, costBasis: 313.83, lastPrice: 313.84, marketValue: 36.00, todaysChange: -0.10, unrealized: 0.00, estIncome: 0.43 },
];

const initialData /** @type {DashboardData} */ = {
  positions: initialPositions,
  wellsSnapshots: [
    {
      date: "2025-08-20",
      wellstradeTotal: 3268.74,
      wellstradeUnrealized: 209.30,
      wellstradeIncome: 50.96,
      stocksTotal: 1126.58,
      etfsTotal: 2141.50,
      cash: 0.66,
      notes: "Close +0.12%. UNH red; gold/IBIT green; NYMT +1.7%."
    },
  ],
  troweSnapshots: [
    { date: "2025-08-19", total: 5402.29, principal: 5011.00, earnings: 391.29 },
    { date: "2025-08-18", total: 5423.39, principal: 5011.00, earnings: 412.39 },
  ],
  carSnapshots: [
    { date: "2025-08-15", balance: 15924.90, payoffAmount: 15972.28, rateAPR: 7.24, paymentDue: 347.00 },
  ],
  cashSnapshots: [
    { date: "2025-08-18", cashChecking: 90.79, cashSavings: 80.40 },
  ],
};

function useLocalState(key, defaultValue) {
  const [state, setState] = useState(() => {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : defaultValue;
    } catch (e) {
      return defaultValue;
    }
  });
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(state));
  }, [key, state]);
  return [state, setState];
}

function currency(n) {
  return n?.toLocaleString(undefined, { style: "currency", currency: "USD", maximumFractionDigits: 2 });
}

function Section({ title, icon, children, right }) {
  return (
    <div className="bg-white rounded-2xl shadow p-5 border border-gray-100">
      <div className="flex items-center justify-between mb-4">
        <div className="flex items-center gap-2">
          {icon}
          <h2 className="text-xl font-semibold">{title}</h2>
        </div>
        {right}
      </div>
      {children}
    </div>
  );
}

export default function Dashboard() {
  const [data, setData] = useLocalState(STORAGE_KEY, initialData);
  const [editingPositions, setEditingPositions] = useState(false);

  const totals = useMemo(() => {
    const stocks = data.positions.filter(p => p.type === "Stock");
    const etfs = data.positions.filter(p => p.type === "ETF");
    const stocksTotal = stocks.reduce((s, p) => s + (p.marketValue || 0), 0);
    const etfsTotal = etfs.reduce((s, p) => s + (p.marketValue || 0), 0);
    const income = data.positions.reduce((s, p) => s + (p.estIncome || 0), 0);
    const unreal = data.positions.reduce((s, p) => s + (p.unrealized || 0), 0);
    return { stocksTotal, etfsTotal, income, unreal, grand: stocksTotal + etfsTotal };
  }, [data.positions]);

  // --- Handlers ---
  const saveSnapshot = () => {
    const today = new Date().toISOString().slice(0, 10);
    const snap = {
      date: today,
      wellstradeTotal: totals.grand + (data.wellsSnapshots.at(-1)?.cash ?? 0),
      wellstradeUnrealized: totals.unreal,
      wellstradeIncome: totals.income,
      stocksTotal: totals.stocksTotal,
      etfsTotal: totals.etfsTotal,
      cash: data.wellsSnapshots.at(-1)?.cash ?? 0.66,
      notes: currentNotes,
    };
    setData({ ...data, wellsSnapshots: [...data.wellsSnapshots, snap] });
    setCurrentNotes("");
  };

  const [currentNotes, setCurrentNotes] = useState("");

  const exportJSON = () => {
    const blob = new Blob([JSON.stringify(data, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `jp_dashboard_${new Date().toISOString().slice(0,10)}.json`;
    a.click();
    URL.revokeObjectURL(url);
  };

  const importJSON = (file) => {
    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const next = JSON.parse(e.target.result);
        setData(next);
      } catch (err) {
        alert("Invalid file");
      }
    };
    reader.readAsText(file);
  };

  const resetData = () => {
    if (confirm("Reset dashboard to initial 08/20/2025 data?")) setData(initialData);
  };

  const deleteLastSnapshot = () => {
    if (!data.wellsSnapshots.length) return;
    if (confirm("Delete last WellsTrade snapshot?")) {
      setData({ ...data, wellsSnapshots: data.wellsSnapshots.slice(0, -1) });
    }
  };

  const pieData = useMemo(() => {
    const byType = [
      { name: "Stocks", value: totals.stocksTotal },
      { name: "ETFs", value: totals.etfsTotal },
    ];
    return byType;
  }, [totals]);

  const positionRows = data.positions.map((p, i) => (
    <tr key={p.symbol} className="hover:bg-gray-50">
      <td className="px-3 py-2 font-medium">{p.symbol}</td>
      <td className="px-3 py-2 text-gray-600 hidden md:table-cell">{p.name}</td>
      <td className="px-3 py-2">{p.type}</td>
      <td className="px-3 py-2 text-right">{p.shares}</td>
      <td className="px-3 py-2 text-right">{currency(p.costBasis)}</td>
      {editingPositions ? (
        <td className="px-3 py-2 text-right"><input className="w-24 border rounded px-2 py-1" type="number" step="0.01" value={p.lastPrice}
          onChange={(e)=>{
            const lastPrice = parseFloat(e.target.value||"0");
            const next = [...data.positions];
            next[i] = { ...p, lastPrice, marketValue: +(lastPrice * p.shares).toFixed(2) };
            setData({ ...data, positions: next });
          }} /></td>
      ) : (
        <td className="px-3 py-2 text-right">{currency(p.lastPrice)}</td>
      )}
      <td className="px-3 py-2 text-right">{currency(p.marketValue)}</td>
      <td className={`px-3 py-2 text-right ${p.todaysChange>=0?"text-green-600":"text-red-600"}`}>{currency(p.todaysChange)}</td>
      <td className={`px-3 py-2 text-right ${p.unrealized>=0?"text-green-600":"text-red-600"}`}>{currency(p.unrealized)}</td>
      <td className="px-3 py-2 text-right">{currency(p.estIncome)}</td>
    </tr>
  ));

  const wellsChart = useMemo(() => data.wellsSnapshots.map(s => ({
    date: s.date,
    Total: s.wellstradeTotal,
    Stocks: s.stocksTotal,
    ETFs: s.etfsTotal,
  })), [data.wellsSnapshots]);

  const troweChart = useMemo(() => data.troweSnapshots.map(s => ({ date: s.date, Total: s.total, Earnings: s.earnings })), [data.troweSnapshots]);
  const carChart = useMemo(() => data.carSnapshots.map(s => ({ date: s.date, Balance: s.balance })), [data.carSnapshots]);

  const riskFlags = useMemo(() => {
    const flags = [];
    const nymt = data.positions.find(p=>p.symbol==="NYMT");
    if (nymt) flags.push({ label: "NYMT: high yield, high risk; monitor dividend stability", severity: "warn" });
    const ibit = data.positions.find(p=>p.symbol==="IBIT");
    if (ibit) flags.push({ label: "IBIT: crypto volatility can swing weekly P/L", severity: "info" });
    const gold = data.positions.find(p=>p.symbol==="AAAU") || data.positions.find(p=>p.symbol==="IAU");
    if (gold) flags.push({ label: "Gold: hedge, tends to move on USD & rates", severity: "info" });
    return flags;
  }, [data.positions]);

  return (
    <div className="min-h-screen bg-gray-100 py-6">
      <div className="max-w-7xl mx-auto px-4 space-y-6">
        <header className="flex items-center justify-between">
          <div className="flex items-center gap-3">
            <TrendingUp className="w-7 h-7" />
            <h1 className="text-2xl font-bold">Weekly Portfolio Dashboard</h1>
          </div>
          <div className="flex items-center gap-2">
            <button onClick={saveSnapshot} className="inline-flex items-center gap-2 px-3 py-2 rounded-xl bg-black text-white shadow"><Plus className="w-4 h-4"/>Save snapshot</button>
            <button onClick={exportJSON} className="inline-flex items-center gap-2 px-3 py-2 rounded-xl bg-white border"><Download className="w-4 h-4"/>Export</button>
            <label className="inline-flex items-center gap-2 px-3 py-2 rounded-xl bg-white border cursor-pointer">
              <Upload className="w-4 h-4"/>
              Import
              <input type="file" accept="application/json" className="hidden" onChange={(e)=>e.target.files&&importJSON(e.target.files[0])} />
            </label>
            <button onClick={resetData} className="inline-flex items-center gap-2 px-3 py-2 rounded-xl bg-white border"><RefreshCw className="w-4 h-4"/>Reset</button>
          </div>
        </header>

        {/* KPI Cards */}
        <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
          <div className="bg-white rounded-2xl p-4 shadow border">
            <div className="text-sm text-gray-500">WellsTrade Total</div>
            <div className="text-2xl font-semibold">{currency(totals.grand + (data.wellsSnapshots.at(-1)?.cash ?? 0))}</div>
          </div>
          <div className="bg-white rounded-2xl p-4 shadow border">
            <div className="text-sm text-gray-500">Unrealized Gain</div>
            <div className={`text-2xl font-semibold ${totals.unreal>=0?"text-green-600":"text-red-600"}`}>{currency(totals.unreal)}</div>
          </div>
          <div className="bg-white rounded-2xl p-4 shadow border">
            <div className="text-sm text-gray-500">Est. Annual Income</div>
            <div className="text-2xl font-semibold">{currency(totals.income)}</div>
          </div>
          <div className="bg-white rounded-2xl p-4 shadow border">
            <div className="text-sm text-gray-500">Cash</div>
            <div className="text-2xl font-semibold">{currency(data.wellsSnapshots.at(-1)?.cash ?? 0)}</div>
          </div>
        </div>

        {/* WellsTrade Charts */}
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-4">
          <Section title="WellsTrade — Total Over Time" icon={<DollarSign className="w-5 h-5"/>}>
            <div className="h-64">
              <ResponsiveContainer>
                <LineChart data={wellsChart} margin={{ top: 10, right: 10, left: 0, bottom: 0 }}>
                  <CartesianGrid strokeDasharray="3 3" />
                  <XAxis dataKey="date" />
                  <YAxis />
                  <Tooltip />
                  <Legend />
                  <Line type="monotone" dataKey="Total" stroke="#000000" dot />
                  <Line type="monotone" dataKey="Stocks" stroke="#8884d8" dot={false} />
                  <Line type="monotone" dataKey="ETFs" stroke="#82ca9d" dot={false} />
                </LineChart>
              </ResponsiveContainer>
            </div>
          </Section>

          <Section title="529 (T. Rowe) — Total & Earnings" icon={<PiggyBank className="w-5 h-5"/>}>
            <div className="h-64">
              <ResponsiveContainer>
                <BarChart data={troweChart}>
                  <CartesianGrid strokeDasharray="3 3" />
                  <XAxis dataKey="date" />
                  <YAxis />
                  <Tooltip />
                  <Legend />
                  <Bar dataKey="Total" fill="#8884d8" />
                  <Bar dataKey="Earnings" fill="#82ca9d" />
                </BarChart>
              </ResponsiveContainer>
            </div>
          </Section>

          <Section title="Car Loan — Balance" icon={<Car className="w-5 h-5"/>}>
            <div className="h-64">
              <ResponsiveContainer>
                <LineChart data={carChart}>
                  <CartesianGrid strokeDasharray="3 3" />
                  <XAxis dataKey="date" />
                  <YAxis />
                  <Tooltip />
                  <Legend />
                  <Line type="monotone" dataKey="Balance" stroke="#ff7300" dot />
                </LineChart>
              </ResponsiveContainer>
            </div>
          </Section>
        </div>

        {/* Allocation Pie & Positions Table */}
        <div className="grid grid-cols-1 lg:grid-cols-3 gap-4">
          <Section title="Allocation — Stocks vs ETFs" icon={<Shield className="w-5 h-5"/>}>
            <div className="h-64 flex items-center justify-center">
              <ResponsiveContainer>
                <PieChart>
                  <Pie data={pieData} dataKey="value" nameKey="name" outerRadius={90} label>
                    {pieData.map((entry, index) => (
                      <Cell key={`cell-${index}`} />
                    ))}
                  </Pie>
                  <Tooltip />
                  <Legend />
                </PieChart>
              </ResponsiveContainer>
            </div>
          </Section>

          <Section title="Positions (Editable Last Price)" icon={<Coins className="w-5 h-5"/>} right={
            <button onClick={()=>setEditingPositions(v=>!v)} className="text-sm px-3 py-1.5 rounded-lg border bg-white">
              {editingPositions?"Done":"Edit prices"}
            </button>
          }>
            <div className="overflow-x-auto">
              <table className="min-w-full text-sm">
                <thead>
                  <tr className="text-left text-gray-500 border-b">
                    <th className="px-3 py-2">Symbol</th>
                    <th className="px-3 py-2 hidden md:table-cell">Name</th>
                    <th className="px-3 py-2">Type</th>
                    <th className="px-3 py-2 text-right">Shares</th>
                    <th className="px-3 py-2 text-right">Cost</th>
                    <th className="px-3 py-2 text-right">Last</th>
                    <th className="px-3 py-2 text-right">Value</th>
                    <th className="px-3 py-2 text-right">Today</th>
                    <th className="px-3 py-2 text-right">Unreal.</th>
                    <th className="px-3 py-2 text-right">Est. Income</th>
                  </tr>
                </thead>
                <tbody>
                  {positionRows}
                </tbody>
              </table>
            </div>
          </Section>

          <Section title="Notes & Controls" icon={<TrendingUp className="w-5 h-5"/>} right={
            <button onClick={deleteLastSnapshot} className="inline-flex items-center gap-2 px-3 py-2 rounded-xl bg-white border text-red-600"><Trash2 className="w-4 h-4"/>Delete last</button>
          }>
            <div className="space-y-3">
              <div>
                <div className="text-xs text-gray-500 mb-1">Add note for this week (saved into snapshot)</div>
                <textarea value={currentNotes} onChange={e=>setCurrentNotes(e.target.value)} className="w-full min-h-[100px] border rounded-xl p-3" placeholder="e.g., UNH weak; gold & IBIT green; consider adding SCHD on dip"/>
              </div>
              <div className="text-xs text-gray-500">Risk flags</div>
              <ul className="list-disc ml-5 text-sm">
                {riskFlags.map((f,i)=> (
                  <li key={i} className={f.severity==="warn"?"text-amber-700":"text-gray-700"}>{f.label}</li>
                ))}
              </ul>
            </div>
          </Section>
        </div>

        {/* T. Rowe & Cash Editors */}
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
          <Section title="Update 529 (T. Rowe)" icon={<PiggyBank className="w-5 h-5"/>}>
            <div className="grid grid-cols-2 md:grid-cols-4 gap-3 items-end">
              <InputRow label="Date" id="trowe_date" type="date" defaultValue={new Date().toISOString().slice(0,10)} />
              <InputRow label="Total" id="trowe_total" type="number" step="0.01" />
              <InputRow label="Principal" id="trowe_principal" type="number" step="0.01" />
              <InputRow label="Earnings" id="trowe_earnings" type="number" step="0.01" />
              <button className="col-span-2 md:col-span-1 bg-black text-white rounded-xl px-3 py-2" onClick={()=>{
                const d = /** @type {HTMLInputElement} */(document.getElementById('trowe_date')).value;
                const total = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('trowe_total')).value||"0");
                const principal = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('trowe_principal')).value||"0");
                const earnings = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('trowe_earnings')).value||"0");
                setData({ ...data, troweSnapshots: [...data.troweSnapshots, { date: d, total, principal, earnings }] });
              }}>Add 529 snapshot</button>
            </div>
          </Section>

          <Section title="Update Car Loan" icon={<Car className="w-5 h-5"/>}>
            <div className="grid grid-cols-2 md:grid-cols-5 gap-3 items-end">
              <InputRow label="Date" id="car_date" type="date" defaultValue={new Date().toISOString().slice(0,10)} />
              <InputRow label="Balance" id="car_balance" type="number" step="0.01" />
              <InputRow label="Payoff" id="car_payoff" type="number" step="0.01" />
              <InputRow label="APR %" id="car_apr" type="number" step="0.01" />
              <InputRow label="Payment Due" id="car_due" type="number" step="0.01" />
              <button className="col-span-2 md:col-span-1 bg-black text-white rounded-xl px-3 py-2" onClick={()=>{
                const d = /** @type {HTMLInputElement} */(document.getElementById('car_date')).value;
                const balance = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('car_balance')).value||"0");
                const payoffAmount = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('car_payoff')).value||"0");
                const rateAPR = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('car_apr')).value||"0");
                const paymentDue = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('car_due')).value||"0");
                setData({ ...data, carSnapshots: [...data.carSnapshots, { date: d, balance, payoffAmount, rateAPR, paymentDue }] });
              }}>Add car snapshot</button>
            </div>
          </Section>
        </div>

        {/* Cash Editor */}
        <Section title="Update Cash (Checking + Savings)" icon={<DollarSign className="w-5 h-5"/>}>
          <div className="grid grid-cols-2 md:grid-cols-4 gap-3 items-end">
            <InputRow label="Date" id="cash_date" type="date" defaultValue={new Date().toISOString().slice(0,10)} />
            <InputRow label="Checking" id="cash_checking" type="number" step="0.01" />
            <InputRow label="Savings" id="cash_savings" type="number" step="0.01" />
            <button className="col-span-2 md:col-span-1 bg-black text-white rounded-xl px-3 py-2" onClick={()=>{
              const d = /** @type {HTMLInputElement} */(document.getElementById('cash_date')).value;
              const cashChecking = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('cash_checking')).value||"0");
              const cashSavings = parseFloat(/** @type {HTMLInputElement} */(document.getElementById('cash_savings')).value||"0");
              setData({ ...data, cashSnapshots: [...data.cashSnapshots, { date: d, cashChecking, cashSavings }] });
            }}>Add cash snapshot</button>
          </div>
        </Section>

        {/* Footer */}
        <div className="text-center text-xs text-gray-400 pb-6">Data is stored locally in your browser. Export regularly to back up.</div>
      </div>
    </div>
  );
}

function InputRow({ label, id, type="text", step, defaultValue }) {
  return (
    <label className="text-sm">
      <div className="text-gray-500 mb-1">{label}</div>
      <input id={id} type={type} step={step} defaultValue={defaultValue} className="w-full border rounded-xl px-3 py-2" />
    </label>
  );
}
