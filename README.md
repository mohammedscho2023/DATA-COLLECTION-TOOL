<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
  <title>Offline Data Collector – All-in-One</title>
  <style>
    * {
      box-sizing: border-box;
      font-family: system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
    }
    body {
      max-width: 700px;
      margin: 0 auto;
      padding: 1rem;
      background: #f5f7fb;
    }
    .card {
      background: white;
      border-radius: 1rem;
      padding: 1.2rem;
      margin-bottom: 1rem;
      box-shadow: 0 2px 8px rgba(0,0,0,0.05);
    }
    h1 {
      font-size: 1.6rem;
      margin: 0 0 0.25rem 0;
    }
    .status {
      font-size: 0.85rem;
      color: #555;
      display: flex;
      gap: 1rem;
      align-items: center;
      flex-wrap: wrap;
    }
    .badge {
      background: #e9ecef;
      padding: 0.2rem 0.6rem;
      border-radius: 2rem;
      font-size: 0.75rem;
    }
    .badge.online { background: #d1fae5; color: #065f46; }
    .badge.offline { background: #fee2e2; color: #991b1b; }
    button {
      background: #2563eb;
      color: white;
      border: none;
      padding: 0.5rem 1rem;
      border-radius: 2rem;
      font-size: 0.9rem;
      cursor: pointer;
      font-weight: 500;
    }
    button.secondary {
      background: #6c757d;
    }
    button.sync-btn {
      background: #0d9488;
    }
    input, textarea {
      width: 100%;
      padding: 0.6rem;
      margin-top: 0.5rem;
      margin-bottom: 0.8rem;
      border: 1px solid #ccc;
      border-radius: 0.75rem;
      font-size: 1rem;
    }
    .record-item {
      background: #f8fafc;
      padding: 0.7rem;
      border-radius: 0.75rem;
      margin-bottom: 0.5rem;
      border-left: 4px solid #cbd5e1;
    }
    .record-item.pending {
      border-left-color: #f59e0b;
      background: #fffbeb;
    }
    .footer {
      text-align: center;
      font-size: 0.75rem;
      color: #6c757d;
      margin-top: 2rem;
    }
  </style>
</head>
<body>

<div class="card">
  <h1>📋 Data Collector</h1>
  <div class="status">
    <span id="networkStatus" class="badge offline">🔌 Offline</span>
    <span>📦 <span id="pendingCount">0</span> pending sync</span>
    <button id="manualSyncBtn" class="sync-btn">🔄 Sync now</button>
  </div>
</div>

<div class="card">
  <h3>➕ Add new record</h3>
  <input type="text" id="recordName" placeholder="Name / Title" autocomplete="off">
  <textarea id="recordDesc" rows="2" placeholder="Description (optional)"></textarea>
  <button id="saveBtn">💾 Save locally (offline-ready)</button>
</div>

<div class="card">
  <h3>📄 Your records</h3>
  <div id="recordsList">
    <div style="color: gray;">Loading...</div>
  </div>
</div>

<div class="footer">
  Offline-first · Syncs automatically when online
</div>

<script src="https://cdn.jsdelivr.net/npm/dexie@3.2.4/dist/dexie.js"></script>
<script>
  // ------------------------------------------------------------
  // 1. INIT DATABASE (IndexedDB)
  // ------------------------------------------------------------
  const db = new Dexie('DataCollectionDB');
  db.version(1).stores({
    records: '++id, name, description, timestamp, deviceId, synced'
  });

  function getDeviceId() {
    let id = localStorage.getItem('deviceId');
    if (!id) {
      id = 'dev_' + Math.random().toString(36).substring(2, 10);
      localStorage.setItem('deviceId', id);
    }
    return id;
  }

  async function addRecord(name, description) {
    if (!name.trim()) {
      alert('Please enter a name');
      return false;
    }
    await db.records.add({
      name: name.trim(),
      description: description.trim(),
      timestamp: Date.now(),
      deviceId: getDeviceId(),
      synced: false
    });
    return true;
  }

  async function getAllRecords() {
    return await db.records.orderBy('timestamp').reverse().toArray();
  }

  async function countUnsynced() {
    return await db.records.where('synced').equals(0).count();
  }

  // ------------------------------------------------------------
  // 2. SYNC ENGINE (push + pull) – REPLACE URLs WITH YOUR BACKEND
  // ------------------------------------------------------------
  // ⚠️ CHANGE THESE URLs to your real backend
  const API_BASE = 'https://your-backend.example.com/api';
  const PUSH_URL = `${API_BASE}/records`;
  const PULL_URL = `${API_BASE}/records?since=`;

  let lastSyncTime = parseInt(localStorage.getItem('lastSyncTime') || '0');

  async function pushToServer() {
    const unsynced = await db.records.where('synced').equals(0).toArray();
    if (unsynced.length === 0) return { pushed: 0 };

    let pushedCount = 0;
    for (const record of unsynced) {
      try {
        const { id, synced, ...payload } = record;
        const response = await fetch(PUSH_URL, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        });
        if (response.ok) {
          await db.records.update(record.id, { synced: true });
          pushedCount++;
        } else {
          console.warn('Server rejected record', await response.text());
        }
      } catch (err) {
        console.log('Push failed (network offline?)', err);
        break;
      }
    }
    return { pushed: pushedCount };
  }

  async function pullFromServer() {
    try {
      const url = PULL_URL + lastSyncTime;
      const response = await fetch(url);
      if (!response.ok) return { pulled: 0 };
      const remoteRecords = await response.json();

      let pulledCount = 0;
      for (const remote of remoteRecords) {
        const exists = await db.records.where('timestamp').equals(remote.timestamp)
          .and(r => r.deviceId === remote.deviceId).first();
        if (!exists) {
          await db.records.add({
            ...remote,
            synced: true
          });
          pulledCount++;
        }
      }

      if (remoteRecords.length > 0) {
        lastSyncTime = Date.now();
        localStorage.setItem('lastSyncTime', lastSyncTime);
      }
      return { pulled: pulledCount };
    } catch (err) {
      console.log('Pull failed (offline or server down)', err);
      return { pulled: 0 };
    }
  }

  // Full sync: push then pull
  async function performSync() {
    console.log('🔄 Syncing...');
    const pushResult = await pushToServer();
    const pullResult = await pullFromServer();
    console.log(`✅ Pushed ${pushResult.pushed}, pulled ${pullResult.pulled}`);
    await updateUI();
  }

  // ------------------------------------------------------------
  // 3. UI RENDERING
  // ------------------------------------------------------------
  async function updateUI() {
    const pending = await countUnsynced();
    document.getElementById('pendingCount').innerText = pending;

    const records = await getAllRecords();
    const container = document.getElementById('recordsList');
    if (records.length === 0) {
      container.innerHTML = '<div style="color:gray;">No records yet. Add one above 👆</div>';
      return;
    }

    let html = '';
    for (const rec of records) {
      const date = new Date(rec.timestamp).toLocaleString();
      const pendingClass = !rec.synced ? 'pending' : '';
      html += `
        <div class="record-item ${pendingClass}">
          <strong>${escapeHtml(rec.name)}</strong>
          ${!rec.synced ? '<span style="font-size:0.7rem; background:#fef3c7; padding:2px 6px; border-radius:20px; margin-left:8px;">⏳ pending</span>' : ''}
          <div style="font-size:0.8rem; color:#334155;">${escapeHtml(rec.description || '—')}</div>
          <div style="font-size:0.7rem; color:#64748b; margin-top:6px;">${date} · ${rec.deviceId}</div>
        </div>
      `;
    }
    container.innerHTML = html;
  }

  function escapeHtml(str) {
    if (!str) return '';
    return str.replace(/[&<>]/g, function(m) {
      if (m === '&') return '&amp;';
      if (m === '<') return '&lt;';
      if (m === '>') return '&gt;';
      return m;
    });
  }

  // ------------------------------------------------------------
  // 4. NETWORK STATUS & AUTO-SYNC
  // ------------------------------------------------------------
  function updateNetworkStatus() {
    const online = navigator.onLine;
    const statusSpan = document.getElementById('networkStatus');
    if (online) {
      statusSpan.innerText = '✅ Online';
      statusSpan.className = 'badge online';
      performSync();  // auto-sync when back online
    } else {
      statusSpan.innerText = '🔌 Offline';
      statusSpan.className = 'badge offline';
    }
  }

  window.addEventListener('online', updateNetworkStatus);
  window.addEventListener('offline', updateNetworkStatus);

  // ------------------------------------------------------------
  // 5. DYNAMIC SERVICE WORKER (Background Sync)
  // ------------------------------------------------------------
  (function registerDynamicServiceWorker() {
    if (!('serviceWorker' in navigator)) return;

    // Service worker code as a string (will be turned into a Blob URL)
    const swCode = `
      const SYNC_TAG = 'data-sync';
      self.addEventListener('install', event => {
        self.skipWaiting();
      });
      self.addEventListener('sync', event => {
        if (event.tag === SYNC_TAG) {
          event.waitUntil((async () => {
            console.log('Background sync triggered');
            const clients = await self.clients.matchAll({ type: 'window' });
            if (clients.length > 0) {
              clients[0].postMessage({ type: 'SYNC_TRIGGER' });
            } else {
              // Fallback: try calling a sync endpoint if you have one
              try {
                await fetch('/api/sync', { method: 'POST' });
              } catch (err) {}
            }
          })());
        }
      });
      self.addEventListener('message', event => {
        if (event.data && event.data.type === 'SYNC_COMPLETE') {
          console.log('Sync completed via main page');
        }
      });
    `;

    const blob = new Blob([swCode], { type: 'application/javascript' });
    const swUrl = URL.createObjectURL(blob);
    navigator.serviceWorker.register(swUrl).then(reg => {
      console.log('Dynamic Service Worker registered, background sync available');
      // Clean up the blob URL after registration (optional)
      setTimeout(() => URL.revokeObjectURL(swUrl), 1000);
    }).catch(err => console.log('SW registration failed:', err));
  })();

  // Listen for messages from the service worker to trigger sync
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.addEventListener('message', event => {
      if (event.data && event.data.type === 'SYNC_TRIGGER') {
        console.log('SW asked to sync – executing now');
        performSync();
        // Optionally reply back
        if (event.source) {
          event.source.postMessage({ type: 'SYNC_COMPLETE' });
        }
      }
    });
  }

  // Helper to request background sync after saving offline
  async function requestBackgroundSync() {
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      const reg = await navigator.serviceWorker.ready;
      await reg.sync.register('data-sync');
      console.log('Background sync requested');
    }
  }

  // ------------------------------------------------------------
  // 6. PWA MANIFEST (Dynamic)
  // ------------------------------------------------------------
  (function addDynamicManifest() {
    const manifestObj = {
      name: "Offline Data Collector",
      short_name: "DataColl",
      start_url: ".",
      display: "standalone",
      theme_color: "#2563eb",
      background_color: "#f5f7fb",
      icons: []
    };
    const manifestBlob = new Blob([JSON.stringify(manifestObj)], { type: 'application/json' });
    const manifestUrl = URL.createObjectURL(manifestBlob);
    const link = document.createElement('link');
    link.rel = 'manifest';
    link.href = manifestUrl;
    document.head.appendChild(link);
    // Revoke after load (optional)
    link.onload = () => URL.revokeObjectURL(manifestUrl);
  })();

  // ------------------------------------------------------------
  // 7. EVENT HANDLERS
  // ------------------------------------------------------------
  document.getElementById('saveBtn').onclick = async () => {
    const name = document.getElementById('recordName').value;
    const desc = document.getElementById('recordDesc').value;
    const success = await addRecord(name, desc);
    if (success) {
      document.getElementById('recordName').value = '';
      document.getElementById('recordDesc').value = '';
      await updateUI();

      if (navigator.onLine) {
        await performSync();
      } else {
        await requestBackgroundSync();
      }
    }
  };

  document.getElementById('manualSyncBtn').onclick = async () => {
    await performSync();
  };

  // Initial load
  updateUI();
  if (navigator.onLine) performSync();

  // Auto-refresh UI every 2 seconds
  setInterval(updateUI, 2000);
</script>
</body>
</html>
