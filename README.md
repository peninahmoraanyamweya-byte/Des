import React, { useState, useEffect } from 'react';

// --- PRO THEME CONFIG ---
const COLORS = {
  bg: '#0B0E11',
  surface: '#1E2329',
  border: '#2B3139',
  primary: '#00C087',
  secondary: '#F6465D',
  text: '#EAECEF',
  muted: '#848E9C',
  accent: '#F0B90B'
};

export default function FullTradingTerminal() {
  // --- STATE MANAGEMENT ---
  const [prices, setPrices] = useState({ bitcoin: 0, ethereum: 0, solana: 0 });
  const [activeAsset, setActiveAsset] = useState('bitcoin');
  const [wallet, setWallet] = useState({ cash: 10000, bitcoin: 0, ethereum: 0, solana: 0 });
  const [history, setHistory] = useState([]);
  const [view, setView] = useState('trade'); // 'trade' or 'deposit'
  const [depositAmount, setDepositAmount] = useState('');

  // --- LIVE DATA FEED ---
  useEffect(() => {
    const ws = new WebSocket('wss://ws.coincap.io/prices?assets=bitcoin,ethereum,solana');
    ws.onmessage = (e) => {
      const data = JSON.parse(e.data);
      setPrices(prev => ({ ...prev, ...data }));
    };
    return () => ws.close();
  }, []);

  // --- TRADING LOGIC ---
  const handleTrade = (type) => {
    const amountUSD = 500;
    const currentPrice = prices[activeAsset];
    if (type === 'BUY' && wallet.cash >= amountUSD) {
      setWallet(prev => ({
        ...prev,
        cash: prev.cash - amountUSD,
        [activeAsset]: prev[activeAsset] + (amountUSD / currentPrice)
      }));
      logHistory('BUY', activeAsset, amountUSD);
    } else if (type === 'SELL' && wallet[activeAsset] > 0) {
      const sellQty = wallet[activeAsset] * 0.5;
      setWallet(prev => ({
        ...prev,
        cash: prev.cash + (sellQty * currentPrice),
        [activeAsset]: prev[activeAsset] - sellQty
      }));
      logHistory('SELL', activeAsset, sellQty * currentPrice);
    }
  };

  const logHistory = (type, asset, val) => {
    const entry = { id: Date.now(), type, asset, val: val.toFixed(2), time: new Date().toLocaleTimeString() };
    setHistory(prev => [entry, ...prev]);
  };

  const triggerMpesa = () => {
    if (!depositAmount) return alert("Enter amount");
    alert(`STK Push sent to phone for KES ${depositAmount}`);
    setWallet(prev => ({ ...prev, cash: prev.cash + parseFloat(depositAmount) / 130 })); // Approx USD conversion
    setDepositAmount('');
    setView('trade');
  };

  // --- CALCULATIONS ---
  const portfolioValue = wallet.cash + (wallet.bitcoin * prices.bitcoin) + (wallet.ethereum * prices.ethereum) + (wallet.solana * prices.solana);

  return (
    <div style={styles.app}>
      {/* 1. TOP NAV */}
      <nav style={styles.nav}>
        <div style={styles.logo}>BINANCE <span style={{color: COLORS.primary}}>PRO</span></div>
        <div style={styles.navLinks}>
          <span onClick={() => setView('trade')} style={styles.clickable}>Market</span>
          <span onClick={() => setView('deposit')} style={styles.clickable}>Deposit</span>
        </div>
        <div style={styles.walletHeader}>
          Portfolio: <span style={{color: COLORS.primary}}>${portfolioValue.toLocaleString(undefined, {maximumFractionDigits: 2})}</span>
        </div>
      </nav>

      <div style={styles.layout}>
        {/* 2. LEFT: WATCHLIST */}
        <aside style={styles.sidebarLeft}>
          <h4 style={styles.sectionTitle}>Markets</h4>
          {Object.keys(prices).map(asset => (
            <div key={asset} onClick={() => setActiveAsset(asset)} style={{...styles.marketRow, borderLeft: activeAsset === asset ? `3px solid ${COLORS.accent}` : 'none'}}>
              <span>{asset.toUpperCase()}</span>
              <span style={{color: COLORS.primary}}>${parseFloat(prices[asset] || 0).toLocaleString()}</span>
            </div>
          ))}
        </aside>

        {/* 3. CENTER: CHART OR DEPOSIT */}
        <main style={styles.mainContent}>
          {view === 'trade' ? (
            <>
              <div style={styles.chartHeader}>
                <h2>{activeAsset.toUpperCase()} / USDT</h2>
                <h1 style={{color: COLORS.primary}}>${parseFloat(prices[activeAsset] || 0).toLocaleString()}</h1>
              </div>
              <div style={styles.iframeWrapper}>
                <iframe
                  key={activeAsset}
                  title="chart"
                  src={`https://s.tradingview.com/widgetembed/?symbol=BINANCE:${activeAsset.toUpperCase()}USDT&interval=D&theme=dark`}
                  style={styles.iframe}
                />
              </div>
            </>
          ) : (
            <div style={styles.depositContainer}>
              <h2>Fund Account via M-Pesa</h2>
              <input 
                style={styles.input} 
                placeholder="Amount in KES" 
                value={depositAmount} 
                onChange={(e) => setDepositAmount(e.target.value)}
              />
              <button onClick={triggerMpesa} style={styles.actionBtn}>Send STK Push</button>
            </div>
          )}
        </main>

        {/* 4. RIGHT: TRADING & HISTORY */}
        <aside style={styles.sidebarRight}>
          <div style={styles.tradeBox}>
            <h4 style={styles.sectionTitle}>Execution</h4>
            <div style={styles.walletStats}>
              <p>Available Cash: ${wallet.cash.toFixed(2)}</p>
              <p>Holding: {wallet[activeAsset].toFixed(4)} {activeAsset.substring(0,3).toUpperCase()}</p>
            </div>
            <button onClick={() => handleTrade('BUY')} style={styles.buyBtn}>BUY $500</button>
            <button onClick={() => handleTrade('SELL')} style={styles.sellBtn}>SELL 50%</button>
          </div>

          <div style={styles.historyBox}>
            <h4 style={styles.sectionTitle}>Recent Activity</h4>
            {history.slice(0, 10).map(item => (
              <div key={item.id} style={styles.historyItem}>
                <span style={{color: item.type === 'BUY' ? COLORS.primary : COLORS.secondary}}>{item.type}</span>
                <span>{item.val}</span>
                <span style={{fontSize: '10px', color: COLORS.muted}}>{item.time}</span>
              </div>
            ))}
          </div>
        </aside>
      </div>
    </div>
  );
}

// --- STYLES ---
const styles = {
  app: { backgroundColor: COLORS.bg, height: '100vh', color: COLORS.text, fontFamily: 'Inter, sans-serif', display: 'flex', flexDirection: 'column' },
  nav: { height: '60px', borderBottom: `1px solid ${COLORS.border}`, display: 'flex', alignItems: 'center', padding: '0 20px', justifyContent: 'space-between' },
  logo: { fontWeight: 'bold', fontSize: '20px', letterSpacing: '1px' },
  navLinks: { display: 'flex', gap: '30px', fontSize: '14px', color: COLORS.muted },
  clickable: { cursor: 'pointer' },
  walletHeader: { backgroundColor: COLORS.surface, padding: '8px 15px', borderRadius: '8px', fontSize: '14px' },
  layout: { flex: 1, display: 'flex', overflow: 'hidden' },
  sidebarLeft: { width: '250px', borderRight: `1px solid ${COLORS.border}`, padding: '15px' },
  sidebarRight: { width: '300px', borderLeft: `1px solid ${COLORS.border}`, padding: '20px', display: 'flex', flexDirection: 'column', gap: '20px' },
  sectionTitle: { fontSize: '12px', color: COLORS.muted, marginBottom: '15px', textTransform: 'uppercase' },
  marketRow: { display: 'flex', justifyContent: 'space-between', padding: '12px', backgroundColor: COLORS.surface, marginBottom: '5px', cursor: 'pointer', fontSize: '14px' },
  mainContent: { flex: 1, display: 'flex', flexDirection: 'column', padding: '20px', backgroundColor: '#131722' },
  chartHeader: { marginBottom: '10px' },
  iframeWrapper: { flex: 1, borderRadius: '8px', overflow: 'hidden' },
  iframe: { width: '100%', height: '100%', border: 'none' },
  tradeBox: { backgroundColor: COLORS.surface, padding: '20px', borderRadius: '12px' },
  walletStats: { fontSize: '12px', marginBottom: '20px', color: COLORS.muted },
  buyBtn: { width: '100%', padding: '12px', backgroundColor: COLORS.primary, border: 'none', borderRadius: '4px', fontWeight: 'bold', cursor: 'pointer', marginBottom: '10px' },
  sellBtn: { width: '100%', padding: '12px', backgroundColor: COLORS.secondary, border: 'none', borderRadius: '4px', fontWeight: 'bold', cursor: 'pointer', color: 'white' },
  historyBox: { flex: 1 },
  historyItem: { display: 'flex', justifyContent: 'space-between', padding: '8px 0', borderBottom: `1px solid ${COLORS.border}`, fontSize: '12px' },
  depositContainer: { flex: 1, justifyContent: 'center', alignItems: 'center', display: 'flex', flexDirection: 'column' },
  input: { backgroundColor: COLORS.surface, border: `1px solid ${COLORS.border}`, padding: '15px', borderRadius: '8px', color: '#fff', width: '300px', marginBottom: '20px', fontSize: '18px' },
  actionBtn: { backgroundColor: COLORS.primary, padding: '15px 40px', borderRadius: '8px', border: 'none', fontWeight: 'bold', cursor: 'pointer' }
};

