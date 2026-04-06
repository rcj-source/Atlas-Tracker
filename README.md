import React, { useMemo, useState } from "react";
import { motion, AnimatePresence } from "framer-motion";
import {
  Wallet,
  Bell,
  ArrowUpRight,
  ArrowDownRight,
  RefreshCcw,
  Search,
  Plus,
  Shield,
  Eye,
  Activity,
  ChevronRight,
  Home,
  BarChart3,
  Check,
  Trash2,
} from "lucide-react";
import {
  ResponsiveContainer,
  PieChart as RePieChart,
  Pie,
  Cell,
  AreaChart,
  Area,
  CartesianGrid,
  XAxis,
  YAxis,
  Tooltip,
  BarChart,
  Bar,
} from "recharts";
import { Card, CardContent, CardHeader, CardTitle, CardDescription } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Badge } from "@/components/ui/badge";
import { Progress } from "@/components/ui/progress";

const emptyTrend = [
  { t: "00h", value: 0 },
  { t: "12h", value: 0 },
  { t: "24h", value: 0 },
  { t: "36h", value: 0 },
  { t: "48h", value: 0 },
];

const emptyDistribution = [
  { name: "No assets", value: 100 },
];

const emptyChainDistribution = [
  { name: "Bitcoin", value: 0 },
  { name: "Ethereum", value: 0 },
  { name: "Solana", value: 0 },
];

const colors = ["#ffffff", "#52525b", "#71717a", "#a1a1aa"];
const chainColors = ["#ffffff", "#a1a1aa", "#52525b"];

type ChainType = "Solana" | "Ethereum" | "Bitcoin";

type Watcher = {
  alias: string;
  address: string;
  chain: ChainType;
  change: number;
  lastAction: string;
  updated: string;
  rawData?: any;
};

function StatCard({ title, value, subtitle, icon: Icon, positive = true }: any) {
  return (
    <Card className="rounded-[28px] border-white/10 bg-white/5 shadow-2xl backdrop-blur-xl">
      <CardContent className="p-4">
        <div className="flex items-start justify-between gap-3">
          <div className="min-w-0">
            <p className="text-xs text-zinc-400">{title}</p>
            <div className="mt-2 text-xl font-semibold tracking-tight text-white sm:text-2xl">{value}</div>
            <div className={`mt-2 flex items-center gap-1 text-xs ${positive ? "text-emerald-400" : "text-red-400"}`}>
              {positive ? <ArrowUpRight className="h-3.5 w-3.5" /> : <ArrowDownRight className="h-3.5 w-3.5" />}
              <span className="truncate">{subtitle}</span>
            </div>
          </div>
          <div className="rounded-2xl border border-white/10 bg-white/5 p-3">
            <Icon className="h-5 w-5 text-white" />
          </div>
        </div>
      </CardContent>
    </Card>
  );
}

function NavPill({ label, active, onClick, icon: Icon }: any) {
  return (
    <button
      type="button"
      onClick={onClick}
      className={`flex min-w-0 flex-1 items-center justify-center gap-2 rounded-2xl px-3 py-3 text-xs font-medium transition ${
        active ? "bg-white text-black" : "bg-white/5 text-zinc-300 hover:bg-white/10 hover:text-white"
      }`}
    >
      <Icon className="h-4 w-4" />
      <span className="truncate">{label}</span>
    </button>
  );
}

function EmptyState({ title, description, actionLabel, onAction }: { title: string; description: string; actionLabel?: string; onAction?: () => void }) {
  return (
    <Card className="rounded-[28px] border border-dashed border-white/10 bg-white/[0.03]">
      <CardContent className="flex flex-col items-center justify-center p-8 text-center">
        <div className="flex h-14 w-14 items-center justify-center rounded-3xl border border-white/10 bg-white/5">
          <Wallet className="h-6 w-6 text-white" />
        </div>
        <div className="mt-4 text-lg font-semibold">{title}</div>
        <div className="mt-2 max-w-sm text-sm text-zinc-400">{description}</div>
        {actionLabel && onAction ? (
          <Button onClick={onAction} className="mt-5 rounded-2xl bg-white text-black hover:bg-zinc-200">
            <Plus className="mr-2 h-4 w-4" />
            {actionLabel}
          </Button>
        ) : null}
      </CardContent>
    </Card>
  );
}

export default function AtlasTrackerV1() {
  const WORKER_BASE_URL = "https://bitter-sea-c49b.ramirezricardocontacto.workers.dev";
  const [selectedChain, setSelectedChain] = useState("All");
  const [query, setQuery] = useState("");
  const [activeScreen, setActiveScreen] = useState("dashboard");
  const [watchers, setWatchers] = useState<Watcher[]>([]);
  const [showSaved, setShowSaved] = useState(false);
  const [formError, setFormError] = useState("");
  const [newWatcher, setNewWatcher] = useState<{ alias: string; address: string; chain: ChainType }>({ alias: "", address: "", chain: "Solana" });

  const hasWallets = watchers.length > 0;

  const filteredWatchers = useMemo(() => {
    return watchers.filter((wallet) => {
      const chainMatch = selectedChain === "All" || wallet.chain === selectedChain;
      const search = query.trim().toLowerCase();
      const queryMatch = !search || wallet.alias.toLowerCase().includes(search) || wallet.address.toLowerCase().includes(search);
      return chainMatch && queryMatch;
    });
  }, [watchers, query, selectedChain]);

  const trackedChains = useMemo(() => {
    const chains = new Set(watchers.map((wallet) => wallet.chain));
    return Array.from(chains);
  }, [watchers]);

  const validateAddress = (address: string, chain: ChainType) => {
    const trimmed = address.trim();
    if (chain === "Ethereum") return /^0x[a-fA-F0-9]{40}$/.test(trimmed);
    if (chain === "Bitcoin") return /^(bc1|[13])[a-zA-HJ-NP-Z0-9]{20,74}$/.test(trimmed);
    return /^[1-9A-HJ-NP-Za-km-z]{32,44}$/.test(trimmed);
  };

  const fetchWalletData = async (address: string, chain: ChainType) => {
    let endpoint = "";

    if (chain === "Solana") endpoint = `/solana?address=${encodeURIComponent(address)}`;
    if (chain === "Ethereum") endpoint = `/ethereum?address=${encodeURIComponent(address)}`;
    if (chain === "Bitcoin") endpoint = `/bitcoin?address=${encodeURIComponent(address)}`;

    const response = await fetch(`${WORKER_BASE_URL}${endpoint}`);
    const data = await response.json();

    if (!response.ok || data?.success === false) {
      throw new Error(data?.error || "No se pudo consultar la wallet en el worker.");
    }

    return data;
  };

  const saveWatcher = async () => {
    const alias = newWatcher.alias.trim();
    const address = newWatcher.address.trim();

    if (!alias || !address) {
      setFormError("Alias y wallet address son obligatorios.");
      return;
    }

    if (!validateAddress(address, newWatcher.chain)) {
      setFormError(`La dirección no parece válida para ${newWatcher.chain}.`);
      return;
    }

    const duplicated = watchers.some((wallet) => wallet.address.toLowerCase() === address.toLowerCase() && wallet.chain === newWatcher.chain);
    if (duplicated) {
      setFormError("Esa wallet ya está siendo trackeada.");
      return;
    }

    try {
      setFormError("");
      const blockchainData = await fetchWalletData(address, newWatcher.chain);

      setWatchers((prev) => [
        {
          alias,
          address,
          chain: newWatcher.chain,
          change: 0,
          lastAction: "Fetched from blockchain",
          updated: "Just now",
          rawData: blockchainData,
        },
        ...prev,
      ]);

      setNewWatcher({ alias: "", address: "", chain: "Solana" });
      setShowSaved(true);
      setTimeout(() => setShowSaved(false), 2200);
    } catch (error: any) {
      setFormError(error?.message || "No se pudo guardar y consultar la wallet.");
    }
  };

  const deleteWatcher = (address: string, chain: ChainType) => {
    setWatchers((prev) => prev.filter((wallet) => !(wallet.address === address && wallet.chain === chain)));
  };

  const renderDashboard = () => (
    <div className="space-y-5">
      <div className="grid gap-4 sm:grid-cols-2 xl:grid-cols-4">
        <StatCard title="Tracked Wallets" value={String(watchers.length)} subtitle={hasWallets ? `${trackedChains.length} chains selected` : "No wallets added yet"} icon={Eye} positive={hasWallets} />
        <StatCard title="Portfolio Value" value={hasWallets ? "Pending sync" : "$0"} subtitle={hasWallets ? "Waiting for first snapshot" : "Add a wallet to begin tracking"} icon={Wallet} positive={hasWallets} />
        <StatCard title="12h Changes" value={hasWallets ? "Pending" : "—"} subtitle={hasWallets ? "Will appear after sync" : "No activity without wallets"} icon={Activity} positive={hasWallets} />
        <StatCard title="Chains Active" value={String(trackedChains.length)} subtitle={hasWallets ? trackedChains.join(", ") : "No chains selected yet"} icon={Shield} positive={trackedChains.length > 0} />
      </div>

      {!hasWallets ? (
        <EmptyState
          title="No estás trackeando ninguna wallet"
          description="El dashboard debe arrancar vacío. Agrega una wallet de Solana, Ethereum o Bitcoin y después aparecerán snapshots, distribución y cambios de 12 horas."
          actionLabel="Agregar primera wallet"
          onAction={() => setActiveScreen("watchlist")}
        />
      ) : (
        <>
          <div className="grid gap-5 xl:grid-cols-[1.45fr_1fr]">
            <Card className="rounded-[28px] border-white/10 bg-white/5">
              <CardHeader className="pb-2">
                <CardTitle>Portfolio trend</CardTitle>
                <CardDescription>Preview visual until real backend snapshots are connected</CardDescription>
              </CardHeader>
              <CardContent className="h-[260px] px-2 sm:h-[320px] sm:px-6">
                <ResponsiveContainer width="100%" height="100%">
                  <AreaChart data={emptyTrend}>
                    <defs>
                      <linearGradient id="fillCurve" x1="0" y1="0" x2="0" y2="1">
                        <stop offset="5%" stopColor="#ffffff" stopOpacity={0.18} />
                        <stop offset="95%" stopColor="#ffffff" stopOpacity={0} />
                      </linearGradient>
                    </defs>
                    <CartesianGrid stroke="rgba(255,255,255,0.08)" vertical={false} />
                    <XAxis dataKey="t" stroke="#71717a" tickLine={false} axisLine={false} />
                    <YAxis hide />
                    <Tooltip contentStyle={{ background: "rgba(0,0,0,0.92)", border: "1px solid rgba(255,255,255,0.1)", borderRadius: 16 }} />
                    <Area type="monotone" dataKey="value" stroke="#ffffff" strokeWidth={2.2} fill="url(#fillCurve)" />
                  </AreaChart>
                </ResponsiveContainer>
              </CardContent>
            </Card>

            <Card className="rounded-[28px] border-white/10 bg-white/5">
              <CardHeader className="pb-2">
                <CardTitle>Asset allocation</CardTitle>
                <CardDescription>Will populate after first real sync</CardDescription>
              </CardHeader>
              <CardContent className="h-[260px] px-2 sm:h-[320px] sm:px-6">
                <ResponsiveContainer width="100%" height="100%">
                  <RePieChart>
                    <Pie data={emptyDistribution} dataKey="value" nameKey="name" innerRadius={45} outerRadius={86} paddingAngle={0}>
                      {emptyDistribution.map((entry, index) => (
                        <Cell key={entry.name} fill={colors[index % colors.length]} />
                      ))}
                    </Pie>
                    <Tooltip contentStyle={{ background: "rgba(0,0,0,0.92)", border: "1px solid rgba(255,255,255,0.1)", borderRadius: 16 }} />
                  </RePieChart>
                </ResponsiveContainer>
              </CardContent>
            </Card>
          </div>

          <div className="grid gap-5 xl:grid-cols-3">
            <Card className="rounded-[28px] border-white/10 bg-white/5">
              <CardHeader className="pb-2">
                <CardTitle>Chain exposure</CardTitle>
                <CardDescription>Placeholder view until balances are loaded</CardDescription>
              </CardHeader>
              <CardContent className="h-[220px] px-2 sm:h-[260px] sm:px-6">
                <ResponsiveContainer width="100%" height="100%">
                  <BarChart data={emptyChainDistribution}>
                    <CartesianGrid stroke="rgba(255,255,255,0.08)" vertical={false} />
                    <XAxis dataKey="name" stroke="#71717a" tickLine={false} axisLine={false} fontSize={12} />
                    <YAxis hide />
                    <Tooltip contentStyle={{ background: "rgba(0,0,0,0.92)", border: "1px solid rgba(255,255,255,0.1)", borderRadius: 16 }} />
                    <Bar dataKey="value" radius={[10, 10, 0, 0]}>
                      {emptyChainDistribution.map((entry, index) => (
                        <Cell key={entry.name} fill={chainColors[index % chainColors.length]} />
                      ))}
                    </Bar>
                  </BarChart>
                </ResponsiveContainer>
              </CardContent>
            </Card>

            <Card className="rounded-[28px] border-white/10 bg-white/5 xl:col-span-2">
              <CardHeader className="pb-2">
                <CardTitle>Tracked wallets</CardTitle>
                <CardDescription>These are the only wallets currently registered by the user</CardDescription>
              </CardHeader>
              <CardContent className="space-y-3">
                {watchers.map((wallet) => (
                  <div key={`${wallet.address}-${wallet.chain}`} className="flex items-start gap-3 rounded-2xl border border-white/10 bg-black/30 p-4">
                    <div className="mt-0.5 rounded-full bg-white/10 p-2">
                      <Wallet className="h-4 w-4 text-white" />
                    </div>
                    <div className="min-w-0 flex-1">
                      <div className="flex items-center gap-2">
                        <div className="font-medium">{wallet.alias}</div>
                        <Badge className="rounded-xl border-white/10 bg-white/10 text-white">{wallet.chain}</Badge>
                      </div>
                      <div className="mt-1 truncate text-sm text-zinc-400">{wallet.address}</div>
                      <div className="mt-1 text-sm text-zinc-500">{wallet.lastAction}
                  {wallet.rawData ? (
                    <div className="mt-2 break-all text-xs text-zinc-500">Worker conectado · {wallet.rawData.chain || wallet.chain}</div>
                  ) : null}</div>
                    </div>
                    <Button type="button" variant="outline" onClick={() => deleteWatcher(wallet.address, wallet.chain)} className="rounded-2xl border-white/10 bg-white/5 text-white hover:bg-white/10">
                      <Trash2 className="h-4 w-4" />
                    </Button>
                  </div>
                ))}
              </CardContent>
            </Card>
          </div>
        </>
      )}
    </div>
  );

  const renderPortfolio = () => (
    <div className="space-y-5">
      {!hasWallets ? (
        <EmptyState
          title="Aún no hay portfolio"
          description="La sección de portfolio debe permanecer vacía hasta que el usuario agregue wallets y exista una primera sincronización real."
          actionLabel="Agregar wallet"
          onAction={() => setActiveScreen("watchlist")}
        />
      ) : (
        <>
          <Card className="rounded-[28px] border-white/10 bg-white/5">
            <CardHeader className="gap-4">
              <div>
                <CardTitle>Holdings breakdown</CardTitle>
                <CardDescription>Preview mode. No real token balances loaded yet.</CardDescription>
              </div>
              <div className="flex gap-2 overflow-x-auto pb-1">
                {["All", "Solana", "Ethereum", "Bitcoin"].map((chain) => (
                  <Button
                    type="button"
                    key={chain}
                    variant="outline"
                    onClick={() => setSelectedChain(chain)}
                    className={`shrink-0 rounded-2xl border-white/10 ${
                      selectedChain === chain ? "bg-white text-black hover:bg-zinc-200" : "bg-white/5 text-white hover:bg-white/10"
                    }`}
                  >
                    {chain}
                  </Button>
                ))}
              </div>
            </CardHeader>
          </Card>

          <EmptyState title="Sin balances cargados" description="Las wallets ya fueron agregadas, pero esta V1 todavía no consulta balances on-chain. Cuando conectemos backend y APIs aquí aparecerán holdings, distribución y cambios." />
        </>
      )}
    </div>
  );

  const renderWatchlist = () => (
    <div className="space-y-5">
      <Card className="rounded-[28px] border-white/10 bg-white/5">
        <CardHeader>
          <CardTitle>Add wallet watcher</CardTitle>
          <CardDescription>Register a public address for 12h snapshots</CardDescription>
        </CardHeader>
        <CardContent className="space-y-4">
          <Input
            placeholder="Alias"
            value={newWatcher.alias}
            onChange={(e) => {
              setFormError("");
              setNewWatcher((prev) => ({ ...prev, alias: e.target.value }));
            }}
            className="h-11 rounded-2xl border-white/10 bg-white/5 text-white placeholder:text-zinc-500"
          />
          <Input
            placeholder="Public wallet address"
            value={newWatcher.address}
            onChange={(e) => {
              setFormError("");
              setNewWatcher((prev) => ({ ...prev, address: e.target.value }));
            }}
            className="h-11 rounded-2xl border-white/10 bg-white/5 text-white placeholder:text-zinc-500"
          />
          <div className="grid grid-cols-3 gap-2">
            {(["Solana", "Ethereum", "Bitcoin"] as ChainType[]).map((chain) => (
              <Button
                type="button"
                key={chain}
                variant="outline"
                onClick={() => {
                  setFormError("");
                  setNewWatcher((prev) => ({ ...prev, chain }));
                }}
                className={`rounded-2xl border-white/10 ${
                  newWatcher.chain === chain ? "bg-white text-black hover:bg-zinc-200" : "bg-white/5 text-white hover:bg-white/10"
                }`}
              >
                {chain}
              </Button>
            ))}
          </div>
          <Button type="button" onClick={saveWatcher} className="w-full rounded-2xl bg-white text-black hover:bg-zinc-200">
            <Plus className="mr-2 h-4 w-4" />
            Save watcher
          </Button>
          {formError ? <div className="rounded-2xl border border-red-500/20 bg-red-500/10 p-3 text-sm text-red-300">{formError}</div> : null}
          <AnimatePresence>
            {showSaved && (
              <motion.div
                initial={{ opacity: 0, y: 8 }}
                animate={{ opacity: 1, y: 0 }}
                exit={{ opacity: 0, y: 8 }}
                className="flex items-center gap-2 rounded-2xl border border-emerald-500/20 bg-emerald-500/10 p-3 text-sm text-emerald-300"
              >
                <Check className="h-4 w-4" />
                Wallet guardada correctamente.
              </motion.div>
            )}
          </AnimatePresence>
        </CardContent>
      </Card>

      {!hasWallets ? (
        <EmptyState title="No wallets tracked" description="Todavía no hay ninguna wallet en la watchlist. Agrega la primera arriba y aparecerá aquí abajo." />
      ) : (
        <div className="space-y-4">
          {filteredWatchers.map((wallet) => (
            <Card key={`${wallet.alias}-${wallet.address}`} className="rounded-[28px] border-white/10 bg-white/5">
              <CardContent className="p-4">
                <div className="flex items-start justify-between gap-3">
                  <div className="min-w-0">
                    <div className="flex items-center gap-2">
                      <div className="font-medium">{wallet.alias}</div>
                      <Badge className="rounded-xl border-white/10 bg-white/10 text-white">{wallet.chain}</Badge>
                    </div>
                    <div className="mt-1 truncate text-sm text-zinc-400">{wallet.address}</div>
                  </div>
                  <div className="flex items-center gap-2">
                    <Button type="button" variant="outline" className="rounded-2xl border-white/10 bg-white/5 text-white hover:bg-white/10">
                      <ChevronRight className="h-4 w-4" />
                    </Button>
                    <Button type="button" variant="outline" onClick={() => deleteWatcher(wallet.address, wallet.chain)} className="rounded-2xl border-white/10 bg-white/5 text-white hover:bg-white/10">
                      <Trash2 className="h-4 w-4" />
                    </Button>
                  </div>
                </div>

                <div className="mt-4 grid grid-cols-2 gap-3 text-sm">
                  <div className="rounded-2xl border border-white/10 bg-black/30 p-3">
                    <div className="text-zinc-400">12h change</div>
                    <div className={wallet.change >= 0 ? "mt-1 font-medium text-emerald-400" : "mt-1 font-medium text-red-400"}>
                      {wallet.change >= 0 ? "+" : ""}
                      {wallet.change}%
                    </div>
                  </div>
                  <div className="rounded-2xl border border-white/10 bg-black/30 p-3">
                    <div className="text-zinc-400">Updated</div>
                    <div className="mt-1 font-medium">{wallet.updated}</div>
                  </div>
                </div>

                <div className="mt-3 rounded-2xl border border-white/10 bg-black/30 p-3 text-sm text-zinc-300">
                  {wallet.lastAction}
                </div>
              </CardContent>
            </Card>
          ))}
        </div>
      )}
    </div>
  );

  const renderSummary = () => (
    <div className="space-y-5">
      <Card className="rounded-[28px] border-white/10 bg-white/5">
        <CardHeader>
          <CardTitle>Next snapshot</CardTitle>
          <CardDescription>Portfolio and watched wallets refresh every 12 hours</CardDescription>
        </CardHeader>
        <CardContent>
          <div className="text-4xl font-semibold tracking-tight">04:21:17</div>
          <Button type="button" className="mt-4 w-full rounded-2xl bg-white text-black hover:bg-zinc-200">
            <RefreshCcw className="mr-2 h-4 w-4" />
            Force preview refresh
          </Button>
        </CardContent>
      </Card>

      {!hasWallets ? (
        <EmptyState title="No hay resumen aún" description="Los resúmenes de 12 horas aparecerán cuando exista al menos una wallet agregada y haya una sincronización real." />
      ) : (
        <Card className="rounded-[28px] border-white/10 bg-white/5">
          <CardHeader>
            <CardTitle>Protocol notes</CardTitle>
            <CardDescription>Scope locked for V1</CardDescription>
          </CardHeader>
          <CardContent className="space-y-3 text-sm text-zinc-300">
            <div className="rounded-2xl border border-white/10 bg-black/30 p-4">Read-only wallet tracking</div>
            <div className="rounded-2xl border border-white/10 bg-black/30 p-4">Chains: Solana, Ethereum, Bitcoin</div>
            <div className="rounded-2xl border border-white/10 bg-black/30 p-4">Refresh cadence: every 12 hours</div>
            <div className="rounded-2xl border border-white/10 bg-black/30 p-4">No swaps, no custody, no signatures</div>
          </CardContent>
        </Card>
      )}
    </div>
  );

  return (
    <div className="min-h-screen bg-black text-white">
      <div className="absolute inset-0 bg-[radial-gradient(circle_at_top_right,rgba(255,255,255,0.08),transparent_28%),radial-gradient(circle_at_bottom_left,rgba(255,255,255,0.06),transparent_22%)]" />

      <div className="relative mx-auto flex min-h-screen w-full max-w-md flex-col px-4 pb-28 pt-4 sm:max-w-2xl sm:px-6 lg:max-w-6xl lg:px-8">
        <header className="sticky top-0 z-20 -mx-4 border-b border-white/10 bg-black/80 px-4 pb-4 pt-2 backdrop-blur-xl sm:-mx-6 sm:px-6 lg:mx-0 lg:rounded-b-3xl lg:border lg:bg-white/[0.04] lg:px-5 lg:py-5">
          <div className="flex items-center justify-between gap-3">
            <div className="flex items-center gap-3">
              <div className="flex h-11 w-11 items-center justify-center rounded-2xl bg-white text-black shadow-lg">
                <Wallet className="h-5 w-5" />
              </div>
              <div>
                <div className="text-base font-semibold tracking-tight">Atlas Tracker</div>
                <div className="text-xs text-zinc-400">Mobile V1 · Multichain watcher</div>
              </div>
            </div>
            <Button
              type="button"
              variant="outline"
              onClick={() => setActiveScreen("watchlist")}
              className="h-11 rounded-2xl border-white/10 bg-white/5 text-white hover:bg-white/10"
            >
              <Plus className="mr-2 h-4 w-4" />
              Add
            </Button>
          </div>

          <div className="mt-4 relative">
            <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-zinc-500" />
            <Input
              value={query}
              onChange={(e) => setQuery(e.target.value)}
              placeholder="Search wallet alias or address"
              className="h-11 rounded-2xl border-white/10 bg-white/5 pl-10 text-white placeholder:text-zinc-500"
            />
          </div>
        </header>

        <main className="mt-5 flex-1">
          <AnimatePresence mode="wait">
            <motion.div
              key={activeScreen}
              initial={{ opacity: 0, y: 10 }}
              animate={{ opacity: 1, y: 0 }}
              exit={{ opacity: 0, y: -10 }}
              transition={{ duration: 0.18 }}
            >
              {activeScreen === "dashboard" && renderDashboard()}
              {activeScreen === "portfolio" && renderPortfolio()}
              {activeScreen === "watchlist" && renderWatchlist()}
              {activeScreen === "summary" && renderSummary()}
            </motion.div>
          </AnimatePresence>
        </main>

        <nav className="fixed bottom-0 left-0 right-0 z-30 border-t border-white/10 bg-black/90 p-3 backdrop-blur-xl lg:left-1/2 lg:max-w-md lg:-translate-x-1/2 lg:rounded-t-3xl lg:border-x">
          <div className="mx-auto flex w-full max-w-md gap-2">
            <NavPill label="Home" icon={Home} active={activeScreen === "dashboard"} onClick={() => setActiveScreen("dashboard")} />
            <NavPill label="Portfolio" icon={BarChart3} active={activeScreen === "portfolio"} onClick={() => setActiveScreen("portfolio")} />
            <NavPill label="Watchlist" icon={Eye} active={activeScreen === "watchlist"} onClick={() => setActiveScreen("watchlist")} />
            <NavPill label="12h" icon={Bell} active={activeScreen === "summary"} onClick={() => setActiveScreen("summary")} />
          </div>
        </nav>
      </div>
    </div>
  );
}
