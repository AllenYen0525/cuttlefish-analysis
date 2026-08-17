<!DOCTYPE html>
<html lang="zh-TW">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>烏賊實驗數據圖表對比與篩選系統</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
    body { display: flex; flex-direction: column; height: 100vh; background-color: #f4f6f8; color: #333; }
    
    /* 頂部導覽列與篩選器 */
    header { background: #ffffff; padding: 16px 24px; border-bottom: 1px solid #e1e4e8; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
    h1 { font-size: 1.25rem; margin-bottom: 12px; color: #24292e; }
    .filter-container { display: flex; flex-wrap: wrap; gap: 12px; align-items: center; }
    .filter-group { display: flex; flex-direction: column; gap: 4px; }
    .filter-group label { font-size: 0.75rem; font-weight: bold; color: #586069; text-transform: uppercase; }
    select { padding: 6px 12px; border: 1px solid #d1d5da; border-radius: 6px; background-color: #fff; font-size: 0.9rem; }
    
    /* 按鈕與操作區 */
    .action-bar { margin-left: auto; display: flex; align-items: center; gap: 12px; }
    .btn { padding: 8px 16px; background-color: #2da44e; color: white; border: none; border-radius: 6px; font-weight: bold; cursor: pointer; }
    .btn:disabled { background-color: #94d3a2; cursor: not-allowed; }
    
    /* 主要內容展示區 */
    main { display: flex; flex: 1; overflow: hidden; }
    .gallery-section { flex: 1; padding: 20px; overflow-y: auto; }
    .chart-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 16px; }
    .chart-card { background: white; border: 2px solid #e1e4e8; border-radius: 8px; padding: 12px; display: flex; flex-direction: column; cursor: pointer; transition: all 0.2s; }
    .chart-card.selected { border-color: #0969da; background-color: #f0f7ff; }
    .chart-card img { width: 100%; height: 180px; object-fit: cover; border-radius: 4px; background: #eee; }
    .chart-info { margin-top: 10px; font-size: 0.85rem; }
    .tag-badge { display: inline-block; background: #eef1f5; color: #444; padding: 2px 6px; border-radius: 4px; font-size: 0.75rem; margin: 2px; }

    /* 全螢幕圖表對比視窗 */
    .compare-section { display: none; width: 100%; height: 100%; position: fixed; top: 0; left: 0; background: rgba(0,0,0,0.85); z-index: 100; padding: 20px; flex-direction: column; }
    .compare-section.active { display: flex; }
    .compare-header { display: flex; justify-content: space-between; align-items: center; color: white; margin-bottom: 12px; }
    .compare-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 16px; flex: 1; overflow-y: auto; }
    .compare-item { background: white; padding: 12px; border-radius: 8px; display: flex; flex-direction: column; }
    .compare-item img { width: 100%; height: auto; max-height: 70vh; object-fit: contain; }
  </style>
</head>
<body>

  <header>
    <h1>實驗數據圖表對比與篩選系統</h1>
    <div class="filter-container" id="filterContainer">
      <!-- 篩選選單由 JavaScript 讀取 data.json 後自動生成 -->
      <div class="action-bar">
        <span id="selectedCount">已選擇 0 張圖表</span>
        <button id="compareBtn" class="btn" disabled onclick="toggleCompareView(true)">開始對比</button>
      </div>
    </div>
  </header>

  <main>
    <section class="gallery-section">
      <div class="chart-grid" id="chartGrid">載入資料中...</div>
    </section>
  </main>

  <!-- 並列對比面板 -->
  <div class="compare-section" id="compareSection">
    <div class="compare-header">
      <h2>圖表橫向對比視窗</h2>
      <button class="btn" style="background:#cf222e;" onclick="toggleCompareView(false)">關閉對比</button>
    </div>
    <div class="compare-grid" id="compareGrid"></div>
  </div>

  <script>
    let chartsData = [];
    let selectedChartIds = new Set();
    let currentFilters = {};

    // 1. 從伺服器讀取 data.json 檔案
    async function loadData() {
      try {
        const response = await fetch('./data.json');
        if (!response.ok) throw new Error('讀取 data.json 失敗');
        chartsData = await response.json();
        initFilters();
        renderCharts();
      } catch (error) {
        document.getElementById('chartGrid').innerHTML = `
          <div style="color: #cf222e; padding: 20px;">
            <h3>資料載入失敗！</h3>
            <p>請確認儲存庫根目錄下是否有建立 <b>data.json</b> 且格式正確。</p>
          </div>`;
      }
    }

    // 2. 自動分析所有標籤，生成下拉篩選選單
    function initFilters() {
      const filterContainer = document.getElementById('filterContainer');
      const actionBar = filterContainer.querySelector('.action-bar');
      
      const tagCategories = new Set();
      chartsData.forEach(chart => {
        if (chart.tags) {
          Object.keys(chart.tags).forEach(cat => tagCategories.add(cat));
        }
      });

      tagCategories.forEach(category => {
        const options = new Set(chartsData.map(c => c.tags ? c.tags[category] : null).filter(Boolean));

        const groupDiv = document.createElement('div');
        groupDiv.className = 'filter-group';
        
        const label = document.createElement('label');
        label.innerText = category;
        
        const select = document.createElement('select');
        select.innerHTML = `<option value="">全部 (${category})</option>` + 
          Array.from(options).map(opt => `<option value="${opt}">${opt}</option>`).join('');
        
        select.addEventListener('change', (e) => {
          currentFilters[category] = e.target.value;
          renderCharts();
        });

        groupDiv.appendChild(label);
        groupDiv.appendChild(select);
        filterContainer.insertBefore(groupDiv, actionBar);
      });
    }

    // 3. 根據選取條件過濾並渲染圖表網格
    function renderCharts() {
      const grid = document.getElementById('chartGrid');
      grid.innerHTML = '';

      const filteredCharts = chartsData.filter(chart => {
        return Object.keys(currentFilters).every(cat => {
          return !currentFilters[cat] || (chart.tags && chart.tags[cat] === currentFilters[cat]);
        });
      });

      if (filteredCharts.length === 0) {
        grid.innerHTML = '<p style="padding: 20px; color: #6e7681;">沒有符合條件的圖表。</p>';
        return;
      }

      filteredCharts.forEach(chart => {
        const isSelected = selectedChartIds.has(chart.id);
        const card = document.createElement('div');
        card.className = `chart-card ${isSelected ? 'selected' : ''}`;
        card.onclick = () => toggleSelectChart(chart.id);

        const tagsHtml = chart.tags ? Object.entries(chart.tags)
          .map(([k, v]) => `<span class="tag-badge"><b>${k}:</b> ${v}</span>`).join('') : '';

        card.innerHTML = `
          <img src="${chart.file}" alt="${chart.title}" onerror="this.src='https://via.placeholder.com/300x200?text=圖片路徑錯誤'">
          <div class="chart-info">
            <strong>${chart.title}</strong>
            <div style="margin-top:6px;">${tagsHtml}</div>
          </div>
        `;
        grid.appendChild(card);
      });
    }

    // 4. 切換圖表勾選狀態
    function toggleSelectChart(id) {
      if (selectedChartIds.has(id)) {
        selectedChartIds.delete(id);
      } else {
        selectedChartIds.add(id);
      }
      document.getElementById('selectedCount').innerText = `已選擇 ${selectedChartIds.size} 張圖表`;
      document.getElementById('compareBtn').disabled = selectedChartIds.size < 2;
      renderCharts();
    }

    // 5. 控制並列對比面板視窗
    function toggleCompareView(show) {
      const compareSection = document.getElementById('compareSection');
      const compareGrid = document.getElementById('compareGrid');
      
      if (show) {
        compareGrid.innerHTML = '';
        chartsData.filter(c => selectedChartIds.has(c.id)).forEach(chart => {
          const item = document.createElement('div');
          item.className = 'compare-item';
          item.innerHTML = `
            <h3 style="margin-bottom:8px;">${chart.title}</h3>
            <img src="${chart.file}" />
          `;
          compareGrid.appendChild(item);
        });
        compareSection.classList.add('active');
      } else {
        compareSection.classList.remove('active');
      }
    }

    // 頁面載入完成後自動執行
    window.onload = loadData;
  </script>
</body>
</html>
