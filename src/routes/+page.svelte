<!-- src/routes/+page.svelte -->

<script>
  import { onMount } from 'svelte';
  import { getDashboardData, getDrilldownData, getFilterOptions, getHeadcountData } from '$lib/api.js';
  import Filters from '$lib/components/Filters.svelte';
  import KpiCard from '$lib/components/KpiCard.svelte';
  import InsightCard from '$lib/components/InsightCard.svelte';
  import LineChart from '$lib/components/LineChart.svelte';
  import BarChart from '$lib/components/BarChart.svelte';

  // ================================================================
  // Constants (CORRECTED to match the backend API response keys)
  // ================================================================
  const KPI_MAP = {
    total_hires: 'Total Hires',
    avg_time_to_fill: 'Avg. Time to Fill',
    avg_cost_per_hire: 'Avg. Cost Per Hire',
    ijp_adherence_rate: 'IJP Adherence',
    build_buy_rate: 'Build Rate',
    diversity_hire_rate: 'Diversity Rate',
  };
  const LABEL_TO_KEY = Object.fromEntries(Object.entries(KPI_MAP).map(([k, v]) => [v, k]));
  const KPI_LABELS = Object.values(KPI_MAP);

  // ================================================================
  // State Management
  // ================================================================
  
  // -- Filter State --
  let businessUnits = ['All Units'];
  let functions = ['All Functions'];
  let selectedBU = 'All Units';
  let selectedFunction = 'All Functions';
  let dateRange = { start: '2025-01-01', end: '2025-12-31' };

  // -- View & Loading State --
  let view = 'main'; // 'main' or 'drilldown'
  let selectedKpiLabel = null;
  let loading = true;
  let error = null;
  
  // -- Data State --
  let dashboardData = { kpiData: {}, insightData: { insights: [] } };
  let drilldownData = null;
  let allHeadcountData = [];
  let headcountSummary = { total: 0, available: 0, gap: 0 };

  // ================================================================
  // Data Fetching
  // ================================================================

  onMount(async () => {
    try {
      const [options, headcount] = await Promise.all([
        getFilterOptions(),
        getHeadcountData()
      ]);
      
      businessUnits = ['All Units', ...options.businessGroups];
      functions = ['All Functions', ...options.functions];
      allHeadcountData = headcount;

    } catch (e) {
      error = "Failed to load initial page data. Please refresh.";
      console.error(e);
    }
  });

  // Reactive block for fetching main dashboard data
  $: {
    (async () => {
      loading = true;
      error = null;
      try {
        const filters = {
          business_group: selectedBU === 'All Units' ? '' : selectedBU,
          function: selectedFunction === 'All Functions' ? '' : selectedFunction,
          start_date: dateRange.start,
          end_date: dateRange.end,
        };
        dashboardData = await getDashboardData(filters);
      } catch (e) {
        error = e.message;
        console.error(e);
      } finally {
        loading = false;
      }
    })();
  }

  // Reactive block for calculating headcount summary
  $: {
    if (allHeadcountData.length > 0) {
      const sumRows = (rows) =>
        rows.reduce(
          (acc, r) => ({
            total: acc.total + (r.total_headcount || 0),
            available: acc.available + (r.available_headcount || 0),
            gap: acc.gap + (r.gap || 0),
          }), { total: 0, available: 0, gap: 0 }
        );

      const isAllUnits = selectedBU === 'All Units';
      const isAllFunc = selectedFunction === 'All Functions';

      let result = { total: 0, available: 0, gap: 0 };

      if (isAllUnits && isAllFunc) {
        const overallRows = allHeadcountData.filter(d => d.function === 'Overall' && d.business_group !== 'All Units');
        result = sumRows(overallRows);
      } else if (isAllUnits && !isAllFunc) {
        const filteredRows = allHeadcountData.filter(d => d.function === selectedFunction && d.business_group !== 'All Units');
        result = sumRows(filteredRows);
      } else if (!isAllUnits && isAllFunc) {
        const row = allHeadcountData.find(d => d.business_group === selectedBU && d.function === 'Overall');
        if(row) result = { total: row.total_headcount, available: row.available_headcount, gap: row.gap };
      } else {
        const row = allHeadcountData.find(d => d.business_group === selectedBU && d.function === selectedFunction);
        if(row) result = { total: row.total_headcount, available: row.available_headcount, gap: row.gap };
      }
      headcountSummary = result;
    }
  }

  // Reactive block specifically for fetching DRILLDOWN data.
  $: {
    if (view === 'drilldown' && selectedKpiLabel) {
      (async () => {
        loading = true;
        error = null;
        drilldownData = null; 

        const mainDashboardKey = LABEL_TO_KEY[selectedKpiLabel];
    
        if (!mainDashboardKey) {
            error = `Could not find a valid key for KPI: "${selectedKpiLabel}"`;
            loading = false;
            return;
        }

        const drilldownKey = mainDashboardKey
          .replace('avg_', '')
          .replace('build_buy_rate', 'build_rate')
          .replace('diversity_hire_rate', 'diversity_rate');
        
        try {
          const filters = {
            business_group: selectedBU === 'All Units' ? '' : selectedBU,
            function: selectedFunction === 'All Functions' ? '' : selectedFunction,
          };

          drilldownData = await getDrilldownData(drilldownKey, filters);
        } catch(e) {
          error = e.message;
          console.error(e);
        } finally {
          loading = false;
        }
      })();
    }
  }

  // ================================================================
  // Handlers & Formatters
  // ================================================================
  
  function openDrilldown(event) {
    const kpiLabel = event.detail.label;
    selectedKpiLabel = kpiLabel;
    view = 'drilldown';
  }

  function goBackToMain() {
    view = 'main';
    selectedKpiLabel = null;
    drilldownData = null;
    error = null;
  }

  // Generic formatter for numbers without special units (like headcount)
  const fmt = (v) => v === null || v === undefined || !Number.isFinite(+v) ? '—' : v.toLocaleString();

  // MODIFIED: Updated formatter for KPI values with correct units and rounding
  function formatKpiValue(key, value) {
    if (value === null || value === undefined || !Number.isFinite(+value)) return '—';

    switch (key) {
      case 'avg_time_to_fill':
        return `${Math.round(value)} days`;
      case 'avg_cost_per_hire':
        // MODIFIED: Shows the full number with commas, not 'k' format.
        return `₹${Math.round(value).toLocaleString()}`;
      case 'ijp_adherence_rate':
      case 'build_buy_rate':
      case 'diversity_hire_rate':
        return `${Math.round(value)}%`;
      case 'total_hires':
      default:
        return value.toLocaleString();
    }
  }

</script>

<div class="min-h-screen bg-gray-900 text-gray-300 p-6 font-sans">
  <div class="max-w-screen-xl mx-auto">
    <header class="mb-6">
      <h1 class="text-3xl font-bold text-white">Talent Dashboard</h1>
      <p class="text-gray-400 mt-1">Core hiring metrics and trends</p>
    </header>

    <Filters {businessUnits} {functions} bind:selectedBU bind:selectedFunction bind:dateRange />

    {#if error}
      <div class="bg-red-900/50 border border-red-700 text-red-300 p-4 rounded-lg mb-8">{error}</div>
    {/if}

    {#if view === 'main'}
      {#if loading && !dashboardData.kpiData.total_hires}
        <div class="text-center p-10">Loading Dashboard...</div>
      {:else}
        <section class="space-y-8">
          
          <div>
            <h2 class="text-xl font-semibold text-white mb-4">Headcount Overview</h2>
            <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
              <div class="bg-gradient-to-br from-blue-900/50 to-gray-900/50 p-6 rounded-lg shadow-lg border border-blue-700/50">
                <h3 class="text-sm font-semibold text-blue-300">Total Headcount</h3>
                <p class="text-3xl font-bold text-white mt-2">{fmt(headcountSummary.total)}</p>
              </div>
              <div class="bg-gradient-to-br from-green-900/50 to-gray-900/50 p-6 rounded-lg shadow-lg border border-green-700/50">
                <h3 class="text-sm font-semibold text-green-300">Available Headcount</h3>
                <p class="text-3xl font-bold text-white mt-2">{fmt(headcountSummary.available)}</p>
              </div>
              <div class="bg-gradient-to-br from-orange-900/50 to-gray-900/50 p-6 rounded-lg shadow-lg border border-orange-700/50">
                <h3 class="text-sm font-semibold text-orange-300">Open Positions (Gap)</h3>
                <p class="text-3xl font-bold text-white mt-2">{fmt(headcountSummary.gap)}</p>
              </div>
            </div>
          </div>

          <div>
            <h2 class="text-xl font-semibold text-white mb-4">Key Performance Indicators</h2>
            <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
              <!-- This section now uses the corrected formatKpiValue function -->
              {#each KPI_LABELS as label}
                {@const key = LABEL_TO_KEY[label]}
                <KpiCard
                  {label}
                  value={formatKpiValue(key, dashboardData.kpiData[key])}
                  on:kpiClick={openDrilldown}
                />
              {/each}
            </div>
          </div>

          <div>
            <h2 class="text-xl font-semibold text-white mb-4">AI-Driven Insights</h2>
            {#if !dashboardData.insightData || dashboardData.insightData.insights.length === 0}
              <div class="text-center text-gray-500 p-6">No insights for current selection.</div>
            {:else}
              <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                {#each dashboardData.insightData.insights as insight}
                  <InsightCard title={insight.title} description={insight.description} />
                {/each}
              </div>
            {/if}
          </div>
        </section>
      {/if}

    {:else if view === 'drilldown'}
      <div class="space-y-8">
        <div class="flex justify-between items-center">
          <h2 class="text-2xl font-bold text-white">
            Drilldown: <span class="text-blue-400">{selectedKpiLabel}</span>
          </h2>
          <button on:click={goBackToMain} class="px-4 py-2 text-sm bg-gray-700 hover:bg-gray-600 rounded-lg font-semibold">
            ← Back
          </button>
        </div>
        
        {#if loading}
          <div class="text-center p-10">Loading Drilldown...</div>
        {:else if drilldownData}
           <section class="bg-gray-800/20 p-4 rounded-lg border border-dashed border-gray-700">
             <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
               <div class="bg-gray-800/60 p-6 rounded-lg shadow-md">
                 <h4 class="font-semibold text-gray-200 mb-4">Month-wise Trend</h4>
                 <LineChart chartData={drilldownData.trend_chart_data} kpiName={selectedKpiLabel} />
               </div>
               <div class="bg-gray-800/60 p-6 rounded-lg shadow-md">
                 <h4 class="font-semibold text-gray-200 mb-4">Breakdown Comparison</h4>
                 <BarChart chartData={drilldownData.breakdown_chart_data} kpiName={selectedKpiLabel} />
               </div>
             </div>
           </section>

          <section>
            <h3 class="text-xl font-semibold text-white mb-4">AI-Driven Insights for {selectedKpiLabel}</h3>
             {#if !drilldownData.ai_insights || drilldownData.ai_insights.insights.length === 0}
                <div class="text-center text-gray-500 p-6">No insights for this KPI.</div>
             {:else}
                <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                  {#each drilldownData.ai_insights.insights as insight}
                    <InsightCard title={insight.title} description={insight.description} />
                  {/each}
                </div>
             {/if}
          </section>
        {/if}
      </div>
    {/if}
  </div>
</div>
