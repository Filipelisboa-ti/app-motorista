import React, { useState, useEffect, useCallback } from "react";
import { Home, Target, TrendingUp, History, Plus, Trash2, Fuel, Route, Clock, Wallet, ChevronRight, X, Loader2, Settings2 } from "lucide-react";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, BarChart, Bar } from "recharts";

// ---------- palette ----------
// C1: #DD6B35 (laranja principal, cabeçalho)  C2: #F0A03C (laranja-âmbar, menu inferior)
// C3: #3FAF9E (verde-água, positivo/secundário)  C4: #D6503F (vermelho, negativo)
// bg: #F6F2EC  card: #FFFFFF  border: #ECE3D8  text: #34302B  muted: #A79C8D

const todayStr = () => new Date().toLocaleDateString("sv-SE");
const brl = (n) => (isFinite(n) ? n : 0).toLocaleString("pt-BR", { style: "currency", currency: "BRL" });
const num = (v) => Number(String(v).replace(",", ".")) || 0;
const uid = () => Math.random().toString(36).slice(2, 9);

function startOfWeek(dateStr) {
  const d = new Date(dateStr + "T00:00:00");
  const day = d.getDay();
  const diff = day === 0 ? -6 : 1 - day;
  d.setDate(d.getDate() + diff);
  return d.toLocaleDateString("sv-SE");
}
function addDays(dateStr, n) {
  const d = new Date(dateStr + "T00:00:00");
  d.setDate(d.getDate() + n);
  return d.toLocaleDateString("sv-SE");
}
function custosFixosTotal(metas) {
  return Object.values(metas.custosFixos || {}).reduce((s, v) => s + (Number(v) || 0), 0);
}
function dayTotals(entry, metas) {
  if (!entry) return { ganhos: 0, combustivel: 0, custosFixos: 0, extras: 0, gastos: 0, lucro: 0, litros: 0, porKm: 0, porHora: 0 };
  const ganhos = (entry.ganhos || []).reduce((s, g) => s + num(g.valor), 0);
  const km = num(entry.km);
  const horas = num(entry.horas);
  const consumo = num(metas.consumo) || 1;
  const litros = km / consumo;
  const combustivel = litros * num(entry.precoLitro);
  const custosFixos = custosFixosTotal(metas);
  const extras = (entry.extras || []).reduce((s, g) => s + num(g.valor), 0);
  const gastos = combustivel + custosFixos + extras;
  const lucro = ganhos - gastos;
  return { ganhos, combustivel, custosFixos, extras, gastos, lucro, litros, porKm: km > 0 ? lucro / km : 0, porHora: horas > 0 ? lucro / horas : 0 };
}

const DEFAULT_DATA = {
  dias: {},
  metas: {
    kmSemanal: 520,
    faturamentoSemanal: 1100,
    consumo: 7.5,
    custosFixos: { manutencao: 30, financiamento: 38, seguro: 7, documentacao: 6, amortizacao: 15 },
  },
};
const CUSTO_LABELS = { manutencao: "manutenção", financiamento: "financiamento", seguro: "seguro", documentacao: "documentação", amortizacao: "amortização" };

// ---------- flat ring progress ----------
function Ring({ value, target, label, color = "#DD6B35", format = brl, size = 92 }) {
  const pct = target > 0 ? Math.min(value / target, 1) : 0;
  const r = (size - 12) / 2;
  const c = 2 * Math.PI * r;
  return (
    <div className="flex flex-col items-center gap-2">
      <div className="relative" style={{ width: size, height: size }}>
        <svg width={size} height={size} className="-rotate-90">
          <circle cx={size / 2} cy={size / 2} r={r} fill="none" stroke="#ECE3D8" strokeWidth="9" />
          <circle
            cx={size / 2}
            cy={size / 2}
            r={r}
            fill="none"
            stroke={color}
            strokeWidth="9"
            strokeLinecap="round"
            strokeDasharray={`${pct * c} ${c}`}
            style={{ transition: "stroke-dasharray .5s ease" }}
          />
function ProgressBar({ label, value, target, format = brl, color = "#3FAF9E" }) {
  const pct = target > 0 ? Math.min((value / target) * 100, 100) : 0;
  return (
    <div>
      <div className="flex justify-between text-xs text-[#8A8073] mb-1">
        <span>{label} <span className="text-[#C2B7A8]">(meta {format(target)})</span></span>
        <span className="font-mono font-semibold" style={{ color }}>{pct.toFixed(0)}%</span>
      </div>
      <div className="h-2 rounded-full bg-[#ECE3D8] overflow-hidden">
        <div className="h-full rounded-full transition-all" style={{ width: `${pct}%`, background: color }} />
      </div>
    </div>
  );
}

function Card({ children, className = "" }) {
  return <div className={`bg-white border border-[#ECE3D8] rounded-2xl shadow-sm shadow-[#DD6B35]/5 p-4 ${className}`}>{children}</div>;
}

function LineEditor({ title, icon: Icon, rows, onChange, placeholder, color, nameField }) {
  const update = (id, field, val) => onChange(rows.map((r) => (r.id === id ? { ...r, [field]: val } : r)));
  const remove = (id) => onChange(rows.filter((r) => r.id !== id));
  const add = () => onChange([...rows, { id: uid(), [nameField]: "", valor: "" }]);

  return (
    <Card>
      <div className="flex items-center justify-between mb-3">
        <div className="flex items-center gap-2 text-sm font-semibold text-[#34302B]">
          <Icon size={16} style={{ color }} />
          {title}
        </div>
        <button onClick={add} className="flex items-center gap-1 text-xs px-2.5 py-1.5 rounded-full text-white active:scale-95 transition" style={{ background: color }}>
          <Plus size={13} /> adicionar
        </button>
      </div>
      {rows.length === 0 && <div className="text-xs text-[#C2B7A8] py-2">nenhum registro ainda</div>}
      <div className="flex flex-col gap-2">
        {rows.map((r) => (
          <div key={r.id} className="flex items-center gap-2">
            <input
              value={r[nameField]}
              onChange={(e) => update(r.id, nameField, e.target.value)}
              placeholder={placeholder}
              className="flex-1 bg-[#FAF6F0] border border-[#ECE3D8] rounded-lg px-2.5 py-2 text-sm text-[#34302B] placeholder-[#C2B7A8] outline-none focus:border-[#DD6B35]"
            />
            <input
              value={r.valor}
              onChange={(e) => update(r.id, "valor", e.target.value.replace(/[^0-9.,]/g, ""))}
              inputMode="decimal"
              placeholder="R$"
              className="w-24 bg-[#FAF6F0] border border-[#ECE3D8] rounded-lg px-2.5 py-2 text-sm text-right font-mono text-[#34302B] placeholder-[#C2B7A8] outline-none focus:border-[#DD6B35]"
            />
            <button onClick={() => remove(r.id)} className="text-[#C2B7A8] active:text-[#D6503F] p-1">
              <X size={16} />
            </button>
          </div>
        ))}
      </div>
    </Card>
  );
}

function weekTotals(data, date) {
  const weekStart = startOfWeek(date);
  const weekDates = Array.from({ length: 7 }, (_, i) => addDays(weekStart, i));
  return weekDates.reduce(
    (acc, d) => {
      const t = dayTotals(data.dias[d], data.metas);
      acc.km += num(data.dias[d]?.km);
      acc.faturamento += t.ganhos;
      acc.lucro += t.lucro;
      return acc;
    },
    { km: 0, faturamento: 0, lucro: 0 }
  );
}

// ---------- Hoje ----------
function HojeTab({ data, saveDay }) {
  const date = todayStr();
  const existing = data.dias[date];
  const metas = data.metas;
  const [ganhos, setGanhos] = useState(existing?.ganhos || []);
  const [extras, setExtras] = useState(existing?.extras || []);
  const [km, setKm] = useState(existing?.km ?? "");
  const [horas, setHoras] = useState(existing?.horas ?? "");
  const [precoLitro, setPrecoLitro] = useState(existing?.precoLitro ?? "");
  const [saved, setSaved] = useState(false);

  useEffect(() => {
    setGanhos(existing?.ganhos || []);
    setExtras(existing?.extras || []);
    setKm(existing?.km ?? "");
    setHoras(existing?.horas ?? "");
    setPrecoLitro(existing?.precoLitro ?? "");
  }, [date]); // eslint-disable-line

  const totals = dayTotals({ ganhos, extras, km, horas, precoLitro }, metas);
  const week = weekTotals(data, date);

  const handleSave = () => {
    saveDay(date, { ganhos, extras, km, horas, precoLitro });
    setSaved(true);
    setTimeout(() => setSaved(false), 1500);
  };

  return (
    <div className="flex flex-col gap-4 pb-4">
      <div className="text-center pt-1">
        <div className="text-[11px] uppercase tracking-wider text-[#A79C8D]">turno de hoje</div>
        <div className="text-sm text-[#34302B] font-semibold capitalize">
          {new Date(date + "T00:00:00").toLocaleDateString("pt-BR", { weekday: "long", day: "2-digit", month: "long" })}
        </div>
      </div>

      <div className="grid grid-cols-2 gap-3">
        <Card className="!p-3">
          <div className="flex items-center gap-1.5 text-[11px] text-[#A79C8D] mb-1"><Route size={13} /> km rodados</div>
          <input value={km} onChange={(e) => setKm(e.target.value.replace(/[^0-9.,]/g, ""))} inputMode="decimal" placeholder="0" className="w-full bg-transparent text-lg font-mono font-semibold text-[#34302B] outline-none" />
        </Card>
        <Card className="!p-3">
          <div className="flex items-center gap-1.5 text-[11px] text-[#A79C8D] mb-1"><Clock size={13} /> horas na rua</div>
          <input value={horas} onChange={(e) => setHoras(e.target.value.replace(/[^0-9.,]/g, ""))} inputMode="decimal" placeholder="0" className="w-full bg-transparent text-lg font-mono font-semibold text-[#34302B] outline-none" />
        </Card>
      </div>

      <Card>
        <div className="flex items-center gap-1.5 text-[11px] text-[#A79C8D] mb-1"><Fuel size={13} /> preço do litro hoje</div>
        <input value={precoLitro} onChange={(e) => setPrecoLitro(e.target.value.replace(/[^0-9.,]/g, ""))} inputMode="decimal" placeholder="R$ 0,00" className="w-full bg-transparent text-lg font-mono font-semibold text-[#34302B] outline-none" />
        <div className="text-[11px] text-[#C2B7A8] mt-1">
          {num(km) > 0 && num(precoLitro) > 0 ? `${totals.litros.toFixed(1)} L a ${num(metas.consumo)} km/L → combustível: ${brl(totals.combustivel)}` : `consumo fixo considerado: ${num(metas.consumo)} km/L`}
        </div>
      </Card>
<LineEditor title="Ganhos" icon={Wallet} rows={ganhos} onChange={setGanhos} placeholder="app (ex: Uber)" color="#3FAF9E" nameField="app" />
      <LineEditor title="Outros gastos do dia" icon={Trash2} rows={extras} onChange={setExtras} placeholder="ex: pedágio, lavagem" color="#D6503F" nameField="tipo" />

      <Card>
        <div className="text-xs text-[#A79C8D] mb-2 flex items-center gap-1.5"><Settings2 size={13} /> custos fixos do dia ({brl(custosFixosTotal(metas))})</div>
        <div className="flex justify-between text-xs text-[#6E6459] py-0.5"><span>combustível</span><span className="font-mono">{brl(totals.combustivel)}</span></div>
        <div className="flex justify-between text-xs text-[#6E6459] py-0.5"><span>fixos (manut./financ./seguro/doc./amort.)</span><span className="font-mono">{brl(totals.custosFixos)}</span></div>
        <div className="flex justify-between text-xs text-[#6E6459] py-0.5"><span>outros gastos</span><span className="font-mono">{brl(totals.extras)}</span></div>
        <div className="flex justify-between text-sm text-[#34302B] font-semibold pt-2 mt-1 border-t border-[#ECE3D8]"><span>saldo do dia</span><span className="font-mono" style={{ color: totals.lucro >= 0 ? "#3FAF9E" : "#D6503F" }}>{brl(totals.lucro)}</span></div>
      </Card>

      <Card>
        <div className="text-xs text-[#A79C8D] mb-3 flex items-center gap-1.5"><Target size={13} /> semana em andamento</div>
        <div className="flex flex-col gap-3">
          <ProgressBar label="km" value={week.km} target={metas.kmSemanal} format={(v) => `${v.toFixed(0)} km`} color="#F0A03C" />
          <ProgressBar label="faturamento" value={week.faturamento} target={metas.faturamentoSemanal} color="#3FAF9E" />
        </div>
      </Card>

      <button onClick={handleSave} className="w-full py-3 rounded-xl font-semibold text-sm text-white active:scale-[0.98] transition" style={{ background: saved ? "#3FAF9E" : "#DD6B35" }}>
        {saved ? "salvo" : "salvar turno de hoje"}
      </button>
    </div>
  );
}

// ---------- Metas ----------
function MetasTab({ data, saveMetas }) {
  const [kmSemanal, setKmSemanal] = useState(data.metas.kmSemanal);
  const [faturamentoSemanal, setFaturamentoSemanal] = useState(data.metas.faturamentoSemanal);
  const [consumo, setConsumo] = useState(data.metas.consumo);
  const [custos, setCustos] = useState(data.metas.custosFixos);

  const date = todayStr();
  const week = weekTotals(data, date);
  const totalFixosDia = Object.values(custos).reduce((s, v) => s + (Number(v) || 0), 0);

  const commit = () =>
    saveMetas({
      kmSemanal: num(kmSemanal),
      faturamentoSemanal: num(faturamentoSemanal),
      consumo: num(consumo) || 1,
      custosFixos: Object.fromEntries(Object.entries(custos).map(([k, v]) => [k, num(v)])),
    });

  return (
    <div className="flex flex-col gap-5 pb-4">
      <div className="flex justify-center gap-4">
        <Ring value={week.km} target={data.metas.kmSemanal} label="km na semana" color="#F0A03C" format={(v) => `${v.toFixed(0)} km`} />
        <Ring value={week.faturamento} target={data.metas.faturamentoSemanal} label="faturamento" color="#3FAF9E" />
      </div>

      <Card>
        <div className="text-sm font-semibold text-[#34302B] flex items-center gap-2 mb-3"><Target size={16} className="text-[#3FAF9E]" /> metas semanais</div>
        <div className="flex flex-col gap-3">
          <div className="flex items-center justify-between">
            <span className="text-sm text-[#6E6459]">km rodados</span>
            <input value={kmSemanal} onChange={(e) => setKmSemanal(e.target.value.replace(/[^0-9.,]/g, ""))} inputMode="decimal" className="w-28 bg-[#FAF6F0] border border-[#ECE3D8] rounded-lg px-2.5 py-1.5 text-sm text-right font-mono text-[#34302B] outline-none focus:border-[#DD6B35]" />
          </div>
          <div className="flex items-center justify-between">
            <span className="text-sm text-[#6E6459]">faturamento (R$)</span>
            <input value={faturamentoSemanal} onChange={(e) => setFaturamentoSemanal(e.target.value.replace(/[^0-9.,]/g, ""))} inputMode="decimal" className="w-28 bg-[#FAF6F0] border border-[#ECE3D8] rounded-lg px-2.5 py-1.5 text-sm text-right font-mono text-[#34302B] outline-none focus:border-[#DD6B35]" />
          </div>
          <div className="flex items-center justify-between">
            <span className="text-sm text-[#6E6459]">consumo do veículo (km/L)</span>
            <input value={consumo} onChange={(e) => setConsumo(e.target.value.replace(/[^0-9.,]/g, ""))} inputMode="decimal" className="w-28 bg-[#FAF6F0] border border-[#ECE3D8] rounded-lg px-2.5 py-1.5 text-sm text-right font-mono text-[#34302B] outline-none focus:border-[#DD6B35]" />
          </div>
        </div>
      </Card>

      <Card>
        <div className="text-sm font-semibold text-[#34302B] flex items-center gap-2 mb-3">
          <Settings2 size={16} className="text-[#DD6B35]" /> custos fixos por dia
          <span className="ml-auto font-mono text-xs text-[#DD6B35]">{brl(totalFixosDia)}</span>
        </div>
        <div className="flex flex-col gap-3">
          {Object.entries(custos).map(([key, val]) => (
            <div key={key} className="flex items-center justify-between">
              <span className="text-sm text-[#6E6459] capitalize">{CUSTO_LABELS[key] || key}</span>
              <input
                value={val}
                onChange={(e) => setCustos({ ...custos, [key]: e.target.value.replace(/[^0-9.,]/g, "") })}
                inputMode="decimal"
                className="w-28 bg-[#FAF6F0] border border-[#ECE3D8] rounded-lg px-2.5 py-1.5 text-sm text-right font-mono text-[#34302B] outline-none focus:border-[#DD6B35]"
              />
            </div>
          ))}
        </div>
      </Card>

      <button onClick={commit} className="w-full py-3 rounded-xl text-sm font-semibold text-white active:scale-[0.98] transition" style={{ background: "#3FAF9E" }}>
        salvar metas
      </button>
    </div>
  );
}

// ---------- Relatórios ----------
function RelatoriosTab({ data }) {
  const days = Object.entries(data.dias)
    .sort(([a], [b]) => a.localeCompare(b))
    .slice(-14)
    .map(([d, e]) => {
      const t = dayTotals(e, data.metas);
      return { data: d.slice(5).split("-").reverse().join("/"), lucro: Number(t.lucro.toFixed(2)), ganhos: Number(t.ganhos.toFixed(2)) };
    });

  const all = Object.values(data.dias).map((e) => dayTotals(e, data.metas));
  const totalLucro = all.reduce((s, t) => s + t.lucro, 0);
  const kmTotal = Object.values(data.dias).reduce((s, e) => s + num(e.km), 0);
  const horasTotal = Object.values(data.dias).reduce((s, e) => s + num(e.horas), 0);
  const mediaKm = kmTotal > 0 ? totalLucro / kmTotal : 0;
  const mediaHora = horasTotal > 0 ? totalLucro / horasTotal : 0;

  return (
    <div className="flex flex-col gap-4 pb-4">
      <Card>
        <div className="text-xs text-[#A79C8D] mb-2 flex items-center gap-1.5"><TrendingUp size={14} className="text-[#DD6B35]" /> saldo líquido — últimos 14 dias</div>
        <div style={{ width: "100%", height: 160 }}>
          <ResponsiveContainer>
            <LineChart data={days} margin={{ top: 5, right: 8, left: -20, bottom: 0 }}>
              <CartesianGrid stroke="#ECE3D8" vertical={false} />
              <XAxis dataKey="data" stroke="#C2B7A8" tick={{ fontSize: 10 }} />
              <YAxis stroke="#C2B7A8" tick={{ fontSize: 10 }} />
              <Tooltip contentStyle={{ background: "#FFFFFF", border: "1px solid #ECE3D8", borderRadius: 8, fontSize: 12 }} labelStyle={{ color: "#34302B" }} formatter={(v) => brl(v)} />
              <Line type="monotone" dataKey="lucro" stroke="#DD6B35" strokeWidth={2.5} dot={{ r: 2 }} />
            </LineChart>
          </ResponsiveContainer>
        </div>
      </Card>

      <Card>
        <div className="text-xs text-[#A79C8D] mb-2 flex items-center gap-1.5"><Wallet size={14} className="text-[#3FAF9E]" /> faturamento por dia</div>
        <div style={{ width: "100%", height: 140 }}>
          <ResponsiveContainer>
            <BarChart data={days} margin={{ top: 5, right: 8, left: -20, bottom: 0 }}>
              <CartesianGrid stroke="#ECE3D8" vertical={false} />
              <XAxis dataKey="data" stroke="#C2B7A8" tick={{ fontSize: 10 }} />
              <YAxis stroke="#C2B7A8" tick={{ fontSize: 10 }} />
              <Tooltip contentStyle={{ background: "#FFFFFF", border: "1px solid #ECE3D8", borderRadius: 8, fontSize: 12 }} labelStyle={{ color: "#34302B" }} formatter={(v) => brl(v)} />
              <Bar dataKey="ganhos" fill="#3FAF9E" radius={[4, 4, 0, 0]} />
            </BarChart>
          </ResponsiveContainer>
        </div>
      </Card>

      <div className="grid grid-cols-3 gap-2 text-center">
        <Card className="!py-3"><div className="text-[10px] text-[#A79C8D]">saldo total</div><div className="font-mono text-sm font-semibold text-[#34302B]">{brl(totalLucro)}</div></Card>
        <Card className="!py-3"><div className="text-[10px] text-[#A79C8D]">média R$/km</div><div className="font-mono text-sm font-semibold text-[#34302B]">{brl(mediaKm)}</div></Card>
        <Card className="!py-3"><div className="text-[10px] text-[#A79C8D]">média R$/hora</div><div className="font-mono text-sm font-semibold text-[#34302B]">{brl(mediaHora)}</div></Card>
      </div>
    </div>
  );
}

// ---------- Histórico ----------
function HistoricoTab({ data, deleteDay }) {
  const [open, setOpen] = useState(null);
  const days = Object.entries(data.dias).sort(([a], [b]) => b.localeCompare(a));

  if (days.length === 0) return <div className="text-center text-sm text-[#A79C8D] py-10">nenhum turno registrado ainda</div>;

  return (
    <div className="flex flex-col gap-2 pb-4">
      {days.map(([d, e]) => {
        const t = dayTotals(e, data.metas);
        const isOpen = open === d;
        return (
          <div key={d} className="bg-white border border-[#ECE3D8] rounded-2xl overflow-hidden shadow-sm shadow-[#DD6B35]/5">
            <button onClick={() => setOpen(isOpen ? null : d)} className="w-full flex items-center justify-between px-4 py-3">
              <div className="text-left">
                <div className="text-sm text-[#34302B] font-medium capitalize">{new Date(d + "T00:00:00").toLocaleDateString("pt-BR", { day: "2-digit", month: "short", weekday: "short" })}</div>
                <div className="text-[11px] text-[#A79C8D]">{e.km || 0} km · {e.horas || 0} h</div>
              </div>
              <div className="flex items-center gap-2">
                <span className="font-mono text-sm font-semibold" style={{ color: t.lucro >= 0 ? "#3FAF9E" : "#D6503F" }}>{brl(t.lucro)}</span>
                <ChevronRight size={16} className={`text-[#C2B7A8] transition ${isOpen ? "rotate-90" : ""}`} />
              </div>
            </button>
            {isOpen && (
              <div className="px-4 pb-4 flex flex-col gap-2 border-t border-[#ECE3D8] pt-3">
                <div className="flex justify-between text-xs text-[#6E6459]"><span>ganhos</span><span className="font-mono">{brl(t.ganhos)}</span></div>
                <div className="flex justify-between text-xs text-[#6E6459]"><span>combustível</span><span className="font-mono">{brl(t.combustivel)}</span></div>
                <div className="flex justify-between text-xs text-[#6E6459]"><span>custos fixos</span><span className="font-mono">{brl(t.custosFixos)}</span></div>
                <div className="flex justify-between text-xs text-[#6E6459]"><span>outros gastos</span><span className="font-mono">{brl(t.extras)}</span></div>
                <div className="flex justify-between text-xs text-[#6E6459]"><span>R$/km · R$/hora</span><span className="font-mono">{brl(t.porKm)} · {brl(t.porHora)}</span></div>
                <button onClick={() => { deleteDay(d); setOpen(null); }} className="mt-1 flex items-center justify-center gap-1.5 text-xs text-[#D6503F] py-2 rounded-lg bg-[#FAF6F0]">
                  <Trash2 size={13} /> apagar turno
                </button>
              </div>
            )}
          </div>
        );
      })}
    </div>
  );
}

// ---------- App ----------
export default function App() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [tab, setTab] = useState("hoje");
  const [error, setError] = useState(null);

  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get("app-data", false);
        const parsed = res ? JSON.parse(res.value) : DEFAULT_DATA;
        setData({ ...DEFAULT_DATA, ...parsed, metas: { ...DEFAULT_DATA.metas, ...parsed.metas, custosFixos: { ...DEFAULT_DATA.metas.custosFixos, ...(parsed.metas?.custosFixos || {}) } } });
      } catch {
        setData(DEFAULT_DATA);
      } finally {
        setLoading(false);
      }
    })();
  }, []);

  const persist = useCallback(async (next) => {
    setData(next);
    try {
      await window.storage.set("app-data", JSON.stringify(next), false);
    } catch {
      setError("não consegui salvar agora — os dados ficam só nesta sessão");
      setTimeout(() => setError(null), 3000);
    }
  }, []);

  const saveDay = (date, entry) => persist({ ...data, dias: { ...data.dias, [date]: entry } });
  const deleteDay = (date) => {
    const dias = { ...data.dias };
    delete dias[date];
    persist({ ...data, dias });
  };
  const saveMetas = (metas) => persist({ ...data, metas });

  if (loading || !data) {
    return (
      <div className="h-screen bg-[#F6F2EC] flex items-center justify-center">
        <Loader2 className="animate-spin text-[#DD6B35]" size={28} />
      </div>
    );
  }

  const tabs = [
    { id: "hoje", label: "Hoje", icon: Home },
    { id: "metas", label: "Metas", icon: Target },
    { id: "relatorios", label: "Relatórios", icon: TrendingUp },
    { id: "historico", label: "Histórico", icon: History },
  ];

  return (
    <div className="h-screen overflow-hidden bg-[#F6F2EC] flex flex-col" style={{ fontFamily: "system-ui, -apple-system, sans-serif" }}>
      <style>{`input::placeholder { color: #D8CDBE; } * { -webkit-tap-highlight-color: transparent; }`}</style>

      <div className="px-4 pt-5 pb-4 flex-none w-full" style={{ background: "#DD6B35" }}>
        <div className="max-w-md w-full mx-auto">
          <div className="text-[11px] uppercase tracking-[0.15em] text-white/75">painel do motorista</div>
          <div className="text-lg font-bold text-white">seu negócio, em números</div>
        </div>
      </div>

      <div className="flex-1 min-h-0 overflow-y-auto px-4 pt-4 max-w-md w-full mx-auto">
        {tab === "hoje" && <HojeTab data={data} saveDay={saveDay} />}
        {tab === "metas" && <MetasTab data={data} saveMetas={saveMetas} />}
        {tab === "relatorios" && <RelatoriosTab data={data} />}
        {tab === "historico" && <HistoricoTab data={data} deleteDay={deleteDay} />}
      </div>

      {error && <div className="mx-4 mb-2 text-xs text-center text-[#D6503F] bg-white border border-[#D6503F]/30 rounded-lg py-2 flex-none">{error}</div>}

      <div className="flex justify-around py-2.5 flex-none w-full" style={{ background: "#F0A03C" }}>
        <div className="flex justify-around w-full max-w-md mx-auto">
          {tabs.map((t) => {
            const Icon = t.icon;
            const active = tab === t.id;
            return (
              <button key={t.id} onClick={() => setTab(t.id)} className="flex flex-col items-center gap-1 px-3 py-1 transition">
                <div className={`flex items-center justify-center w-9 h-9 rounded-full transition ${active ? "bg-white" : ""}`}>
                  <Icon size={18} color={active ? "#DD6B35" : "#FFFFFF"} strokeWidth={active ? 2.5 : 2} />
                </div>
                <span className={`text-[10px] font-medium ${active ? "text-white" : "text-white/70"}`}>{t.label}</span>
              </button>
            );
          })}
        </div>
      </div>
    </div>
  );
}
        </svg>
        <div className="absolute inset-0 flex items-center justify-center">
          <span className="text-xs font-bold" style={{ color }}>{Math.round(pct * 100)}%</span>
        </div>
      </div>
      <div className="text-center">
        <div className="font-mono text-sm font-semibold text-[#34302B]">{format(value)}</div>
        <div className="text-[10px] text-[#A79C8D]">{label} · meta {format(target)}</div>
      </div>
    </div>
  );
}
