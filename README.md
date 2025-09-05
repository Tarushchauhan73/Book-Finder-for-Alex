# Book-Finder-for-Alex
import React, { useEffect, useMemo, useRef, useState } from "react";
import {
  Search,
  BookOpen,
  Loader2,
  Heart,
  Info,
  X,
  ArrowLeft,
  ArrowRight,
  Filter,
  ExternalLink,
} from "lucide-react";

// --- Utilities
const COVER = (cover_i, size = "M") =>
  cover_i
    ? `https://covers.openlibrary.org/b/id/${cover_i}-${size}.jpg`
    : `https://placehold.co/300x450?text=No+Cover`;

const langOptions = [
  { code: "", name: "Any" },
  { code: "eng", name: "English" },
  { code: "hin", name: "Hindi" },
  { code: "ben", name: "Bengali" },
  { code: "fra", name: "French" },
  { code: "deu", name: "German" },
  { code: "spa", name: "Spanish" },
  { code: "ita", name: "Italian" },
  { code: "jpn", name: "Japanese" },
  { code: "rus", name: "Russian" },
  { code: "tam", name: "Tamil" },
  { code: "tel", name: "Telugu" },
];

const pageSizes = [20, 50, 100];

const SORTS = [
  { id: "relevance", label: "Relevance" },
  { id: "year_asc", label: "Year ↑" },
  { id: "year_desc", label: "Year ↓" },
  { id: "editions_desc", label: "Editions ↓" },
];

const SEARCH_TYPES = [
  { id: "title", label: "Title" },
  { id: "author", label: "Author" },
  { id: "subject", label: "Subject" },
  { id: "isbn", label: "ISBN" },
  { id: "all", label: "Everything" },
];

function useDebounced(value, delay) {
  const [v, setV] = useState(value);
  useEffect(() => {
    const t = setTimeout(() => setV(value), delay);
    return () => clearTimeout(t);
  }, [value, delay]);
  return v;
}

function classNames(...a) {
  return a.filter(Boolean).join(" ");
}

// --- Favorites persistence
const FAV_KEY = "bookfinder.favorites";
const loadFavs = () => {
  try {
    return JSON.parse(localStorage.getItem(FAV_KEY) || "[]");
  } catch {
    return [];
  }
};
const saveFavs = (ids) => localStorage.setItem(FAV_KEY, JSON.stringify(ids));

export default function BookFinderApp() {
  const [term, setTerm] = useState("");
  const [searchType, setSearchType] = useState("title");
  const [language, setLanguage] = useState("");
  const [minYear, setMinYear] = useState("");
  const [maxYear, setMaxYear] = useState("");
  const [sort, setSort] = useState("relevance");
  const [page, setPage] = useState(1);
  const [limit, setLimit] = useState(20);
  const [results, setResults] = useState([]);
  const [numFound, setNumFound] = useState(0);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");
  const [showOnlyFavs, setShowOnlyFavs] = useState(false);
  const [favorites, setFavorites] = useState(loadFavs());
  const [modalDoc, setModalDoc] = useState(null);
  const [details, setDetails] = useState({
    loading: false,
    data: null,
    editions: [],
  });

  const debouncedTerm = useDebounced(term, 400);
  const inputRef = useRef(null);

  // Keyboard shortcuts
  useEffect(() => {
    function onKey(e) {
      if (e.key === "/") {
        e.preventDefault();
        inputRef.current?.focus();
      }
      if (e.key.toLowerCase() === "f") {
        setShowOnlyFavs((s) => !s);
      }
    }
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, []);

  useEffect(() => {
    saveFavs(favorites);
  }, [favorites]);

  // Query builder
  const queryURL = useMemo(() => {
    const u = new URL("https://openlibrary.org/search.json");
    if (searchType === "isbn" && debouncedTerm.trim()) {
      u.searchParams.set("q", `isbn:${debouncedTerm.trim()}`);
    } else if (searchType === "all") {
      if (debouncedTerm.trim()) u.searchParams.set("q", debouncedTerm.trim());
    } else {
      if (debouncedTerm.trim())
        u.searchParams.set(searchType, debouncedTerm.trim());
    }
    if (language) u.searchParams.set("language", language);
    u.searchParams.set("page", String(page));
    u.searchParams.set("limit", String(limit));
    return u.toString();
  }, [debouncedTerm, searchType, language, page, limit]);

  // Fetch search
  useEffect(() => {
    let abort = new AbortController();
    async function run() {
      if (!debouncedTerm.trim() && !showOnlyFavs) {
        setResults([]);
        setNumFound(0);
        return;
      }
      if (showOnlyFavs) return; // client-side filtering when in favs view
      setLoading(true);
      setError("");
      try {
        const r = await fetch(queryURL, { signal: abort.signal });
        if (!r.ok) throw new Error(`Request failed: ${r.status}`);
        const j = await r.json();
        setNumFound(j.numFound || j.num_found || 0);
        setResults(j.docs || []);
      } catch (e) {
        if (e.name !== "AbortError") setError(e.message || String(e));
      } finally {
        setLoading(false);
      }
    }
    run();
    return () => abort.abort();
  }, [queryURL, showOnlyFavs]);

  // Client-side post-filters & sorting
  const processed = useMemo(() => {
    let list = results.slice();
    // Year filter
    const minY = parseInt(minYear, 10);
    const maxY = parseInt(maxYear, 10);
    if (!Number.isNaN(minY))
      list = list.filter((d) => (d.first_publish_year ?? -Infinity) >= minY);
    if (!Number.isNaN(maxY))
      list = list.filter((d) => (d.first_publish_year ?? Infinity) <= maxY);
    // Show only favorites
    if (showOnlyFavs) {
      list = list.filter((d) => favorites.includes(d.key));
    }
    // Sorting
    list.sort((a, b) => {
      if (sort === "relevance") return 0; // server gives relevance-ish order
      if (sort === "year_asc")
        return (a.first_publish_year ?? 1e9) - (b.first_publish_year ?? 1e9);
      if (sort === "year_desc")
        return (b.first_publish_year ?? -1) - (a.first_publish_year ?? -1);
      if (sort === "editions_desc")
        return (b.edition_count ?? 0) - (a.edition_count ?? 0);
      return 0;
    });
    return list;
  }, [results, minYear, maxYear, showOnlyFavs, favorites, sort]);

  const toggleFav = (doc) => {
    setFavorites((ids) =>
      ids.includes(doc.key)
        ? ids.filter((id) => id !== doc.key)
        : [...ids, doc.key]
    );
  };

  const onOpenDetails = async (doc) => {
    setModalDoc(doc);
    setDetails({ loading: true, data: null, editions: [] });
    try {
      const [workRes, edRes] = await Promise.all([
        fetch(`https://openlibrary.org${doc.key}.json`),
        fetch(`https://openlibrary.org${doc.key}/editions.json?limit=10`),
      ]);
      const data = workRes.ok ? await workRes.json() : null;
      const ed = edRes.ok ? await edRes.json() : { entries: [] };
      setDetails({ loading: false, data, editions: ed?.entries || [] });
    } catch (e) {
      setDetails({ loading: false, data: null, editions: [] });
    }
  };

  const clearFilters = () => {
    setLanguage("");
    setMinYear("");
    setMaxYear("");
    setSort("relevance");
  };

  const emptyState = (
    <div className="text-center py-20 text-gray-500">
      <BookOpen className="mx-auto mb-4" />
      <p className="text-lg">
        Search millions of books by title, author, subject, or ISBN.
      </p>
      <p className="text-sm">
        Tip: Press{" "}
        <kbd className="px-1 py-0.5 bg-gray-100 rounded border">/</kbd> to focus
        search. Press{" "}
        <kbd className="px-1 py-0.5 bg-gray-100 rounded border">f</kbd> to
        toggle favorites.
      </p>
    </div>
  );

  return (
    <div className="min-h-screen bg-gradient-to-b from-white to-gray-50 text-gray-900">
      {/* Header */}
      <header className="sticky top-0 z-30 backdrop-blur bg-white/80 border-b">
        <div className="max-w-7xl mx-auto px-4 py-3 flex items-center gap-3">
          <BookOpen className="w-7 h-7" />
          <h1 className="text-2xl font-bold">Book Finder for Alex</h1>
          <span className="ml-auto flex items-center gap-2">
            <button
              onClick={() => setShowOnlyFavs((s) => !s)}
              className={classNames(
                "inline-flex items-center gap-2 rounded-2xl px-3 py-1.5 border shadow-sm",
                showOnlyFavs ? "bg-rose-50 border-rose-200" : "bg-white"
              )}
              title="Show favorites (f)"
            >
              <Heart
                className={classNames(
                  "w-4 h-4",
                  showOnlyFavs ? "fill-rose-500 text-rose-500" : ""
                )}
              />
              <span className="text-sm">Favorites</span>
            </button>
          </span>
        </div>
      </header>

      {/* Controls */}
      <div className="max-w-7xl mx-auto px-4 py-4">
        <div className="grid grid-cols-1 md:grid-cols-12 gap-3 items-end">
          <div className="md:col-span-5">
            <label className="text-sm text-gray-600">Search</label>
            <div className="relative">
              <Search className="absolute left-3 top-1/2 -translate-y-1/2 w-5 h-5 text-gray-400" />
              <input
                ref={inputRef}
                value={term}
                onChange={(e) => {
                  setTerm(e.target.value);
                  setPage(1);
                }}
                onKeyDown={(e) => {
                  if (e.key === "Enter") setPage(1);
                }}
                placeholder="e.g., The Great Gatsby, Agatha Christie, machine learning, 9780140449266"
                className="w-full pl-10 pr-3 py-2 rounded-2xl border focus:ring-2 focus:ring-indigo-400 outline-none"
              />
            </div>
          </div>
          <div className="md:col-span-2">
            <label className="text-sm text-gray-600">Search by</label>
            <select
              value={searchType}
              onChange={(e) => {
                setSearchType(e.target.value);
                setPage(1);
              }}
              className="w-full rounded-2xl border px-3 py-2"
            >
              {SEARCH_TYPES.map((s) => (
                <option key={s.id} value={s.id}>
                  {s.label}
                </option>
              ))}
            </select>
          </div>
          <div className="md:col-span-2">
            <label className="text-sm text-gray-600">Language</label>
            <select
              value={language}
              onChange={(e) => {
                setLanguage(e.target.value);
                setPage(1);
              }}
              className="w-full rounded-2xl border px-3 py-2"
            >
              {langOptions.map((l) => (
                <option key={l.code || "any"} value={l.code}>
                  {l.name}
                </option>
              ))}
            </select>
          </div>
          <div className="md:col-span-3">
            <label className="text-sm text-gray-600 flex items-center gap-2">
              <Filter className="w-4 h-4" /> Year range
            </label>
            <div className="flex gap-2">
              <input
                value={minYear}
                onChange={(e) => setMinYear(e.target.value)}
                placeholder="Min"
                className="w-full rounded-2xl border px-3 py-2"
              />
              <input
                value={maxYear}
                onChange={(e) => setMaxYear(e.target.value)}
                placeholder="Max"
                className="w-full rounded-2xl border px-3 py-2"
              />
            </div>
          </div>
          <div className="md:col-span-2">
            <label className="text-sm text-gray-600">Sort</label>
            <select
              value={sort}
              onChange={(e) => setSort(e.target.value)}
              className="w-full rounded-2xl border px-3 py-2"
            >
              {SORTS.map((s) => (
                <option key={s.id} value={s.id}>
                  {s.label}
                </option>
              ))}
            </select>
          </div>
          <div className="md:col-span-12 flex gap-2 justify-between items-center">
            <div className="flex items-center gap-2">
              <button
                onClick={clearFilters}
                className="px-3 py-2 rounded-2xl border bg-white hover:bg-gray-50"
              >
                Clear filters
              </button>
              <span className="text-sm text-gray-500">
                {showOnlyFavs
                  ? `Showing ${processed.length} favorite result(s)`
                  : numFound
                  ? `${numFound.toLocaleString()} found`
                  : ""}
              </span>
            </div>
            <div className="flex items-center gap-3">
              <label className="text-sm text-gray-600">Per page</label>
              <select
                value={limit}
                onChange={(e) => {
                  setLimit(parseInt(e.target.value));
                  setPage(1);
                }}
                className="rounded-2xl border px-3 py-2"
              >
                {pageSizes.map((s) => (
                  <option key={s} value={s}>
                    {s}
                  </option>
                ))}
              </select>
            </div>
          </div>
        </div>
      </div>

      {/* Body */}
      <main className="max-w-7xl mx-auto px-4 pb-24">
        {error && (
          <div className="bg-rose-50 border border-rose-200 text-rose-600 px-4 py-3 rounded-2xl mb-4 flex items-center gap-2">
            <Info className="w-4 h-4" /> {error}
          </div>
        )}

        {!loading && processed.length === 0 && !error ? emptyState : null}

        {loading && (
          <div className="flex items-center gap-2 text-gray-600 py-10">
            <Loader2 className="w-5 h-5 animate-spin" /> Loading…
          </div>
        )}

        {!loading && processed.length > 0 && (
          <>
            <ul className="grid gap-4 grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5">
              {processed.map((doc) => (
                <li key={doc.key} className="group">
                  <div className="bg-white border rounded-2xl overflow-hidden shadow-sm hover:shadow-md transition">
                    <div className="relative aspect-[2/3] bg-gray-100">
                      <img
                        src={COVER(doc.cover_i, "M")}
                        alt={doc.title}
                        className="w-full h-full object-cover"
                      />
                      <button
                        onClick={() => toggleFav(doc)}
                        className="absolute top-2 right-2 p-1.5 rounded-full bg-white/90 border hover:bg-white"
                        title="Toggle favorite"
                      >
                        <Heart
                          className={classNames(
                            "w-4 h-4",
                            favorites.includes(doc.key)
                              ? "fill-rose-500 text-rose-500"
                              : "text-gray-500"
                          )}
                        />
                      </button>
                    </div>
                    <div className="p-3">
                      <h3 className="font-semibold line-clamp-2 min-h-[3.25rem]">
                        {doc.title}
                      </h3>
                      <p className="text-sm text-gray-600 line-clamp-1">
                        {(doc.author_name || []).join(", ")}
                      </p>
                      <div className="mt-2 flex items-center justify-between text-xs text-gray-500">
                        <span>{doc.first_publish_year || "—"}</span>
                        <span className="px-2 py-0.5 rounded-full bg-gray-100 border">
                          {doc.edition_count || 0} ed.
                        </span>
                      </div>
                      <div className="mt-2 flex items-center gap-2">
                        {doc.language?.slice(0, 2).map((l) => (
                          <span
                            key={l}
                            className="text-[10px] uppercase tracking-wide px-1.5 py-0.5 rounded border bg-gray-50"
                          >
                            {l}
                          </span>
                        ))}
                      </div>
                      <div className="mt-3 flex gap-2">
                        <button
                          onClick={() => onOpenDetails(doc)}
                          className="flex-1 px-3 py-1.5 rounded-xl border hover:bg-gray-50 text-sm"
                        >
                          Details
                        </button>
                        <a
                          href={`https://openlibrary.org${doc.key}`}
                          target="_blank"
                          rel="noreferrer"
                          className="px-3 py-1.5 rounded-xl border hover:bg-gray-50 text-sm inline-flex items-center gap-1"
                        >
                          <ExternalLink className="w-4 h-4" />
                        </a>
                      </div>
                    </div>
                  </div>
                </li>
              ))}
            </ul>

            {/* Pagination */}
            {!showOnlyFavs && (
              <div className="mt-6 flex items-center justify-center gap-3">
                <button
                  onClick={() => setPage((p) => Math.max(1, p - 1))}
                  disabled={page <= 1}
                  className="px-3 py-2 rounded-xl border bg-white disabled:opacity-50 inline-flex items-center gap-1"
                >
                  <ArrowLeft className="w-4 h-4" /> Prev
                </button>
                <span className="text-sm text-gray-600">Page {page}</span>
                <button
                  onClick={() => setPage((p) => p + 1)}
                  disabled={page * limit >= numFound}
                  className="px-3 py-2 rounded-xl border bg-white disabled:opacity-50 inline-flex items-center gap-1"
                >
                  Next <ArrowRight className="w-4 h-4" />
                </button>
              </div>
            )}
          </>
        )}
      </main>

      {/* Details Modal */}
      {modalDoc && (
        <div
          className="fixed inset-0 z-40 bg-black/50 flex items-end md:items-center justify-center p-0 md:p-6"
          onClick={() => setModalDoc(null)}
        >
          <div
            className="bg-white rounded-t-2xl md:rounded-2xl w-full md:max-w-3xl max-h-[90vh] overflow-auto"
            onClick={(e) => e.stopPropagation()}
          >
            <div className="p-4 border-b flex items-center justify-between sticky top-0 bg-white">
              <div className="flex items-center gap-3">
                <img
                  src={COVER(modalDoc.cover_i, "S")}
                  alt={modalDoc.title}
                  className="w-10 h-14 object-cover rounded"
                />
                <div>
                  <h3 className="font-semibold">{modalDoc.title}</h3>
                  <p className="text-sm text-gray-600">
                    {(modalDoc.author_name || []).join(", ")}
                  </p>
                </div>
              </div>
              <button
                onClick={() => setModalDoc(null)}
                className="p-2 rounded-xl border bg-white"
              >
                <X className="w-4 h-4" />
              </button>
            </div>
            <div className="p-4 space-y-4">
              {details.loading && (
                <div className="flex items-center gap-2 text-gray-600">
                  <Loader2 className="w-4 h-4 animate-spin" /> Loading details…
                </div>
              )}
              {!details.loading && (
                <>
                  {details.data?.description && (
                    <div>
                      <h4 className="font-semibold mb-1">Description</h4>
                      <p className="text-sm text-gray-700 whitespace-pre-line">
                        {typeof details.data.description === "string"
                          ? details.data.description
                          : details.data.description?.value}
                      </p>
                    </div>
                  )}
                  {details.data?.subjects &&
                    details.data.subjects.length > 0 && (
                      <div>
                        <h4 className="font-semibold mb-1">Subjects</h4>
                        <div className="flex flex-wrap gap-2">
                          {details.data.subjects.slice(0, 20).map((s) => (
                            <span
                              key={s}
                              className="text-xs px-2 py-0.5 rounded-full border bg-gray-50"
                            >
                              {s}
                            </span>
                          ))}
                        </div>
                      </div>
                    )}
                  <div>
                    <h4 className="font-semibold mb-1">Top editions</h4>
                    {details.editions.length === 0 ? (
                      <p className="text-sm text-gray-600">
                        No edition data available.
                      </p>
                    ) : (
                      <ul className="divide-y border rounded-2xl overflow-hidden">
                        {details.editions.map((ed) => (
                          <li
                            key={ed.key}
                            className="p-3 flex items-center gap-3"
                          >
                            <img
                              src={COVER(ed.covers?.[0], "S")}
                              alt="cover"
                              className="w-10 h-14 object-cover rounded"
                            />
                            <div className="flex-1">
                              <div className="font-medium text-sm">
                                {ed.title || modalDoc.title}
                              </div>
                              <div className="text-xs text-gray-600">
                                {ed.publish_date || ed.publish_year?.[0] || "—"}{" "}
                                •{" "}
                                {ed.languages
                                  ?.map((l) => l?.key?.split("/").pop())
                                  .join(", ")}
                              </div>
                            </div>
                            <a
                              href={`https://openlibrary.org${ed.key}`}
                              target="_blank"
                              rel="noreferrer"
                              className="text-sm px-2 py-1 rounded-lg border inline-flex items-center gap-1"
                            >
                              View <ExternalLink className="w-4 h-4" />
                            </a>
                          </li>
                        ))}
                      </ul>
                    )}
                  </div>
                </>
              )}
            </div>
          </div>
        </div>
      )}

      {/* Footer */}
      <footer className="text-center text-xs text-gray-500 py-6">
        Built for Alex • Data from Open Library • Client-side demo (no backend)
      </footer>
    </div>
  );
}
