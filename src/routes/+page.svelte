<script lang="ts">
  let query = $state("");
  let loading = $state(false);
  let error = $state("");
  let last_p_id: number = $state(0);
  let results: Record<string, any>[] = $state([]);
  let is_fully_loaded = $state(false);
  const columns: string[] = ["Katastrálne územie", "Poradové číslo", "LV", "Meno neznámeho vlastníka"];
  const columnKeys: string[] = ["kataster", "poradove_cislo", "lv", "vlastnik"];
  const widths: number[] = [200, 120, 80, 480];
  const PORADOVE_CISLO_COL_INDEX = 1;
  const LV_COL_INDEX = 2;
  const MIN_QUERY_LENGTH = 3;
  let searchTimeout: ReturnType<typeof setTimeout> | null = null;

  async function load(last_id?: number) {
    let local_result = [];
    loading = true;
    error = "";
    try {
      const res = await fetch("https://db.cifko.sk/search", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ partial_name: query, last_p_id: last_id }),
      });

      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      const data = await res.json();
      if (data?.status !== "ok") throw new Error(data?.error ?? "Neznáma chyba servera");
      last_p_id = data.last_p_id;

      local_result = data.results;
      is_fully_loaded = local_result.length !== data.page_size;
    } catch (e: any) {
      error = e?.message ?? String(e);
      local_result = [];
    } finally {
      loading = false;
    }
    return local_result;
  }

  async function search() {
    const trimmed = query?.trim() ?? "";
    if (trimmed.length < MIN_QUERY_LENGTH) {
      results = [];
      return;
    }
    results = await load();
  }

  async function next() {
    if (loading) return;
    const newResults = await load(last_p_id);
    results = [...results, ...newResults];
  }

  function renderCell(row: any, col: string, i: number) {
    if (row[col] !== undefined) return row[col];
    if (typeof row === "object" && row !== null) return Object.values(row)[i] ?? "-";
    return String(row);
  }

  function startSearch() {
    if (searchTimeout) clearTimeout(searchTimeout);
    const trimmed = query?.trim() ?? "";

    if (trimmed.length < MIN_QUERY_LENGTH) {
      results = [];
      error = "";
      return;
    }

    searchTimeout = setTimeout(() => {
      search();
    }, 250);
  }
</script>

<div class="max-w-6xl mx-auto px-4 py-8 sm:px-6 lg:px-8">
  <div class="bg-white rounded-lg shadow-sm p-6 mb-8">
    <h1 class="mb-4 text-2xl font-semibold">Zoznam nezistených vlastníkov</h1>
    <div class="description">
      <div class="text-gray-600 space-y-3">
        <p>
          Slovenský pozemkový fond zverejňuje zoznam nezistených vlastníkov pôdy pre zvýšenie transparentnosti a ochrany
          vlastníckych práv. Údaje pochádzajú z katastra nehnuteľností a majú informatívny charakter.
        </p>
        <p>
          <strong>Ak identifikujete svojho zosnulého predka:</strong> Obráťte sa na notára alebo príslušný súd a overte,
          či prebehlo dedičské konanie. Ak áno, požiadajte súd o preverenie novoobjaveného majetku. Ak nie, požiadajte o
          dodatočné dedičské konanie.
        </p>
        <p class="text-sm">
          <i>
            Slovenský pozemkový fond poskytuje tieto údaje len ako informatívny prehľad a nevykonáva dedičské konania.
          </i>
        </p>
      </div>
    </div>
  </div>
  <div class="bg-amber-50 border border-amber-200 rounded-lg p-4 mb-6">
    <h2 class="text-1xl font-semibold text-amber-900 mb-2">Vyhlásenie</h2>
    <p class="text-sm text-amber-800">
      Táto stránka bola vytvorená vo voľnom čase zadarmo s cieľom uľahčiť vyhľadávanie v databáze nezistených
      vlastníkov, ktoré je na oficiálnej stránke SPF menej pohodlné. Autor stránky nenesie žiadnu zodpovednosť za
      správnosť, úplnosť alebo aktuálnosť zobrazených informácií. Pre overenie údajov vždy použite oficiálne zdroje.
    </p>
  </div>

  <div class="bg-white rounded-lg shadow-sm p-6 mb-6">
    <div class="relative">
      <svg
        xmlns="http://www.w3.org/2000/svg"
        width="24"
        height="24"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        stroke-width="2"
        stroke-linecap="round"
        stroke-linejoin="round"
        class="lucide lucide-search absolute left-3 top-1/2 -translate-y-1/2 text-gray-400 w-5 h-5"
        data-fg-d3bl25="0.8:2.17909:/src/app/App.tsx:216:13:7807:85:e:Search::::::B9rK"
        data-fgid-d3bl25=":rf:"><circle cx="11" cy="11" r="8"></circle><path d="m21 21-4.3-4.3"></path></svg
      >
      <input
        type="search"
        placeholder="Začnite písať meno vlastníka (min. 3 znaky)"
        class="w-full pl-10 pr-4 py-3 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
        bind:value={query}
        aria-label="search"
        oninput={startSearch}
      />
    </div>
    {#if query.trim().length > 0 && query.trim().length < MIN_QUERY_LENGTH}
      <p class="text-sm text-gray-500">
        Zadajte aspoň {MIN_QUERY_LENGTH} znaky pre vyhľadávanie
      </p>
    {/if}

    <div class="meta">
      {#if error}<div class="error">Chyba: {error}</div>{/if}
    </div>
  </div>
  {#if query.trim().length >= MIN_QUERY_LENGTH}
    <div class="relative min-h-[200px]">
      <div class="bg-white rounded-lg shadow-sm overflow-hidden">
        <div class="overflow-x-auto">
          {#if results.length}
            <table class="w-full table-fixed">
              <thead class="bg-gray-50 border-b border-gray-200">
                <tr>
                  {#each columns as col, i}
                    <th class="px-6 py-3 text-left font-semibold text-gray-900 relative" style="width: {widths[i]}px;"
                      >{col}</th
                    >
                  {/each}
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-200">
                {#each results as row}
                  <tr class="hover:bg-blue-50 transition-colors">
                    {#each columns as col, i}
                      <td class="px-6 py-4 text-gray-900 truncate">
                        {#if i == LV_COL_INDEX}
                          <a
                            href="https://kataster.skgeodesy.sk/Portal45/api/Bo/GeneratePrfPublic?prfNumber={row[columnKeys[LV_COL_INDEX]]}&cadastralUnitCode={row[columnKeys[PORADOVE_CISLO_COL_INDEX]]}&outputType=html"
                            class="flex w-full items-center justify-center rounded-md border border-slate-300 bg-slate-50 px-3 py-2 text-center text-sm font-medium text-slate-800 shadow-sm transition hover:border-slate-400 hover:bg-slate-100 hover:text-slate-900 focus:outline-none focus-visible:ring-2 focus-visible:ring-slate-400 focus-visible:ring-offset-1 active:bg-slate-200"
                          >
                            {row[columnKeys[i]]}
                          </a>
                        {:else}
                          {row[columnKeys[i]]}
                        {/if}
                      </td>
                    {/each}
                  </tr>
                {/each}
              </tbody>
            </table>
          {:else}
            <div class="bg-white rounded-lg shadow-sm p-12 text-center">
              <p class="text-gray-600">
                Neboli nájdené žiadne výsledky pre "{query}"
              </p>
            </div>
          {/if}
        </div>

        <div class="px-6 py-3 bg-gray-50 border-t border-gray-200 flex items-center justify-between flex-wrap gap-2">
          <p class="text-sm text-gray-600">
            {#if results.length == 1}
              Zobrazený 1 výsledok
            {:else if results.length >= 2 && results.length <= 4}
              Zobrazené {results.length} výsledky
            {:else}
              Zobrazených {results.length} výsledkov
            {/if}
          </p>
          {#if is_fully_loaded}
            <p class="text-sm text-gray-500">Načítané všetky výsledky</p>
          {:else}
            <button
              class="inline-flex items-center gap-2 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:bg-gray-400 disabled:cursor-not-allowed transition-colors"
              onclick={next}
              >Načítať viac<svg
                xmlns="http://www.w3.org/2000/svg"
                width="24"
                height="24"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
                class="lucide lucide-chevron-right w-4 h-4"><path d="m9 18 6-6-6-6"></path></svg
              ></button
            >
          {/if}
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  * {
    box-sizing: border-box;
  }
</style>
