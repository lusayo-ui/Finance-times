import React, { useState, useEffect, useCallback } from 'react';
import { 
  Search, 
  RefreshCw, 
  ExternalLink, 
  Clock, 
  Filter
} from 'lucide-react';

const COLORS = {
  paper: '#fff1e5', // FT Newsprint
  ink: '#333333',
  accent: '#990f3d',
  border: '#d9cfc3'
};

const FEEDS = [
  { name: 'Nation Business', url: 'https://mwnation.com/category/tie-business/business-news/feed' },
  { name: 'Malawi 24 Business', url: 'https://malawi24.com/category/business/feed' },
  { name: 'Nyasa Business', url: 'https://www.nyasatimes.com/category/business/feed' }
];

const FINANCE_KEYWORDS = [
  'economy', 'finance', 'kwacha', 'inflation', 'gdp', 'bank', 'rbm', 'market', 
  'trade', 'investment', 'business', 'revenue', 'tax', 'debt', 'budget', 
  'interest', 'stock', 'mse', 'fiscal', 'monetary', 'price', 'commodity',
  'mkw', 'procurement', 'world bank', 'imf', 'admarc', 'escom', 'mwk'
];

export default function App() {
  const [articles, setArticles] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [searchTerm, setSearchTerm] = useState('');
  const [lastRefreshed, setLastRefreshed] = useState(null);

  const fetchNews = useCallback(async (isAutoRefresh = false) => {
    if (!isAutoRefresh) setLoading(true);
    setError(null);
    let allArticles = [];

    try {
      // Fetch feeds sequentially with cache-busting
      for (const feed of FEEDS) {
        try {
          const timestamp = new Date().getTime();
          const apiUrl = `https://api.rss2json.com/v1/api.json?rss_url=${encodeURIComponent(feed.url)}&t=${timestamp}`;
          
          const response = await fetch(apiUrl);
          if (!response.ok) continue;
          
          const data = await response.json();
          if (data.status === 'ok' && data.items) {
            const items = data.items.map(item => ({
              id: item.guid || item.link,
              title: item.title,
              link: item.link,
              description: item.description?.replace(/<[^>]*>?/gm, '').replace(/&nbsp;/g, ' '),
              published: item.pubDate,
              source: feed.name,
              content: (item.title + " " + (item.description || "")).toLowerCase()
            }));
            allArticles = [...allArticles, ...items];
          }
        } catch (err) {
          console.error(`Feed Error: ${feed.name}`, err);
        }
      }
      
      const filtered = allArticles.filter(article => 
        FINANCE_KEYWORDS.some(keyword => article.content.includes(keyword))
      );

      // Sort by newest
      filtered.sort((a, b) => new Date(b.published).getTime() - new Date(a.published).getTime());
      
      // Strict unique check by title
      const unique = Array.from(new Map(filtered.map(item => [item.title, item])).values());
      
      setArticles(unique);
      setLastRefreshed(new Date());
      
      if (unique.length === 0 && allArticles.length > 0) {
        setError("No business news matches currently found in active feeds.");
      }
    } catch (err) {
      setError("Connectivity issue with news servers.");
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    // Initial fetch
    fetchNews();

    // Set up 1-hour interval (3,600,000 milliseconds)
    const intervalId = setInterval(() => {
      console.log('Performing scheduled 1-hour auto-refresh...');
      fetchNews(true);
    }, 3600000);

    return () => clearInterval(intervalId);
  }, [fetchNews]);

  const formatDate = (dateString) => {
    try {
      const date = new Date(dateString.replace(/-/g, '/'));
      return date.toLocaleDateString('en-GB', { 
        day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit'
      });
    } catch (e) { return dateString; }
  };

  const displayArticles = articles.filter(a => 
    a.title.toLowerCase().includes(searchTerm.toLowerCase()) ||
    a.source.toLowerCase().includes(searchTerm.toLowerCase())
  );

  return (
    <div className="min-h-screen" style={{ backgroundColor: COLORS.paper, color: COLORS.ink, fontFamily: 'serif' }}>
      
      {/* Concise Header */}
      <header className="border-b border-black py-4 px-4 sticky top-0 z-50 shadow-sm" style={{ backgroundColor: COLORS.paper }}>
        <div className="max-w-5xl mx-auto flex flex-col sm:flex-row justify-between items-center gap-4">
          <div className="flex items-center gap-4">
            <h1 className="text-2xl font-black uppercase tracking-tighter border-b-2 border-black">
              Malawi Financial Times
            </h1>
            <button 
              onClick={() => fetchNews()}
              className="p-1.5 hover:bg-black/5 rounded-full transition-colors relative"
              title="Manual Refresh"
            >
              <RefreshCw size={18} className={loading ? 'animate-spin' : ''} />
              {loading && !articles.length === 0 && (
                <span className="absolute -top-1 -right-1 flex h-2 w-2">
                  <span className="animate-ping absolute inline-flex h-full w-full rounded-full bg-red-400 opacity-75"></span>
                  <span className="relative inline-flex rounded-full h-2 w-2 bg-red-500"></span>
                </span>
              )}
            </button>
          </div>

          <div className="relative w-full sm:w-64 font-sans">
            <Search size={16} className="absolute left-3 top-2.5 text-black/40" />
            <input 
              type="text" 
              placeholder="Search news..." 
              value={searchTerm}
              onChange={(e) => setSearchTerm(e.target.value)}
              className="w-full bg-white/50 border border-black/10 rounded pl-10 pr-4 py-1.5 text-sm outline-none focus:bg-white transition-all"
            />
          </div>
        </div>
      </header>

      <main className="max-w-5xl mx-auto px-4 py-6">
        {/* Error Messaging */}
        {error && (
          <div className="mb-6 p-3 bg-red-50 border border-red-200 text-red-800 text-sm flex gap-2 items-center rounded">
            <Filter size={16} /> {error}
          </div>
        )}

        {/* Content List */}
        <div className="space-y-0 divide-y divide-black/10">
          {loading && articles.length === 0 ? (
            [1,2,3,4,5].map(i => (
              <div key={i} className="py-8 animate-pulse">
                <div className="h-3 bg-black/5 w-24 mb-3"></div>
                <div className="h-6 bg-black/5 w-3/4 mb-2"></div>
                <div className="h-4 bg-black/5 w-1/2"></div>
              </div>
            ))
          ) : (
            displayArticles.map((article) => (
              <a 
                key={article.id} 
                href={article.link} 
                target="_blank" 
                rel="noreferrer"
                className="group block py-6 hover:bg-black/[0.02] transition-colors"
              >
                <div className="flex justify-between items-start gap-6">
                  <div className="flex-grow">
                    <div className="flex items-center gap-3 mb-2 font-sans font-bold text-[10px] uppercase tracking-widest text-red-800">
                      <span>{article.source}</span>
                      <span className="text-black/30 flex items-center gap-1">
                        <Clock size={10} /> {formatDate(article.published)}
                      </span>
                    </div>
                    <h2 className="text-xl md:text-2xl font-bold leading-tight group-hover:underline decoration-red-800 underline-offset-4 mb-2">
                      {article.title}
                    </h2>
                    <p className="text-sm text-slate-600 line-clamp-2 leading-relaxed italic">
                      {article.description}
                    </p>
                  </div>
                  <div className="shrink-0 pt-1 text-black/20 group-hover:text-red-800 transition-colors">
                    <ExternalLink size={18} />
                  </div>
                </div>
              </a>
            ))
          )}
        </div>

        {!loading && displayArticles.length === 0 && !error && (
          <div className="py-20 text-center border-2 border-dashed border-black/10 rounded">
            <p className="font-sans text-sm font-bold uppercase opacity-40">No matching financial records found.</p>
          </div>
        )}
      </main>

      <footer className="max-w-5xl mx-auto px-4 py-12 border-t border-black/10 text-center font-sans">
        <p className="text-[10px] font-black uppercase tracking-widest opacity-40 mb-2">
          Source: Verified Malawi RSS Infrastructure
        </p>
        {lastRefreshed && (
          <p className="text-[9px] opacity-30 uppercase tracking-tighter">
            Intelligence Cycle Status: Auto-Refresh Active (Every 60m) • Last Synced: {lastRefreshed.toLocaleTimeString()}
          </p>
        )}
      </footer>
    </div>
  );
}
