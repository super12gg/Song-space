<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Karaoke Party Pool - คลังเพลงคาราโอเกะสาธารณะ</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link href="https://fonts.googleapis.com/css2?family=Prompt:wght@300;400;500;600;700&display=swap" rel="stylesheet">
  <style>
    body { font-family: 'Prompt', sans-serif; background: #0f172a; color: #f8fafc; }
    .glass { background: rgba(30, 41, 59, 0.75); backdrop-filter: blur(12px); border: 1px solid rgba(255,255,255,0.1); }
  </style>
</head>
<body class="min-h-screen p-4 md:p-8">
  <div class="max-w-3xl mx-auto space-y-6">
    
    <!-- Header -->
    <div class="text-center space-y-2">
      <h1 class="text-3xl md:text-4xl font-bold bg-gradient-to-r from-pink-500 via-purple-400 to-indigo-400 bg-clip-text text-transparent">
        🎤 Karaoke Party Pool
      </h1>
      <p class="text-slate-400 text-sm">เพิ่มเพลง ค้นหาทุกแพลตฟอร์ม และสุ่มคิวร้องร่วมกันแบบเรียลไทม์</p>
      <div id="statusBadge" class="inline-block text-xs px-3 py-1 rounded-full bg-amber-500/20 text-amber-300">
        กำลังเชื่อมต่อฐานข้อมูล...
      </div>
    </div>

    <!-- Randomizer Section -->
    <div class="glass rounded-2xl p-6 text-center space-y-4 shadow-xl">
      <h2 class="text-lg font-semibold text-purple-300">🎲 สุ่มเพลงขึ้นร้อง</h2>
      <div id="randomDisplay" class="py-6 px-4 rounded-xl bg-slate-900/80 border border-slate-700 min-h-[110px] flex flex-col items-center justify-center">
        <p class="text-slate-400 text-sm">กดปุ่มด้านล่างเพื่อสุ่มเพลงจากคลัง</p>
      </div>
      <div id="randomActions" class="flex flex-wrap justify-center gap-3">
        <button onclick="pickRandomSong()" class="px-6 py-3 rounded-xl bg-gradient-to-r from-pink-600 to-purple-600 hover:from-pink-500 hover:to-purple-500 font-semibold shadow-lg transition transform active:scale-95">
          🎲 สุ่มเพลงเลย!
        </button>
      </div>
    </div>

    <!-- Add & Search Song Form -->
    <div class="glass rounded-2xl p-6 space-y-4 shadow-xl">
      <h2 class="text-lg font-semibold text-indigo-300">➕ เพิ่มเพลง / ค้นหาจากทุกแพลตฟอร์ม</h2>
      <div class="grid grid-cols-1 md:grid-cols-2 gap-3">
        <input id="songTitle" type="text" placeholder="ชื่อเพลง (เช่น ทรงอย่างแบด)" class="w-full px-4 py-3 rounded-xl bg-slate-900 border border-slate-700 focus:border-purple-500 focus:outline-none text-white" />
        <input id="songArtist" type="text" placeholder="ชื่อศิลปิน (เช่น Paper Planes)" class="w-full px-4 py-3 rounded-xl bg-slate-900 border border-slate-700 focus:border-purple-500 focus:outline-none text-white" />
      </div>
      <div class="flex flex-wrap gap-3">
        <button onclick="searchPlatforms()" class="flex-1 py-3 px-4 rounded-xl bg-slate-700 hover:bg-slate-600 font-medium transition">
          🔍 ค้นหาจากทุกแพลตฟอร์ม
        </button>
        <button onclick="addSongDirect()" class="flex-1 py-3 px-4 rounded-xl bg-emerald-600 hover:bg-emerald-500 font-semibold transition">
          💾 บันทึกลงคลังทันที
        </button>
      </div>

      <!-- Platform Search Results -->
      <div id="searchResults" class="hidden pt-3 border-t border-slate-700 space-y-3"></div>
    </div>

    <!-- Song Pool List -->
    <div class="glass rounded-2xl p-6 space-y-4 shadow-xl">
      <div class="flex justify-between items-center">
        <h2 class="text-lg font-semibold text-pink-300">🎶 คลังเพลงที่รอคิว (<span id="songCount">0</span>)</h2>
      </div>
      <div id="songList" class="space-y-3">
        <p class="text-center text-slate-500 py-6 text-sm">ยังไม่มีเพลงในคลัง เพิ่มเพลงแรกได้เลย!</p>
      </div>
    </div>

  </div>

  <!-- Firebase SDK -->
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
    import { getDatabase, ref, push, onValue, remove } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-database.js";

    // ⚠️ นำลิงก์ Realtime Database ของคุณมาวางแทนที่ตรงนี้ (ต้องมี / ปิดท้ายหรือไม่ก็ได้)
    const DATABASE_URL = "https://kara-4b83d-default-rtdb.asia-southeast1.firebasedatabase.app/";

    const statusBadge = document.getElementById('statusBadge');
    let songs = [];
    let currentPickedId = null;

    // ตรวจสอบว่าใส่ URL หรือยัง
    if (DATABASE_URL.includes("YOUR-PROJECT-ID")) {
      statusBadge.textContent = "⚠️ กรุณาใส่ลิงก์ Firebase DATABASE_URL ในโค้ดก่อนใช้งาน";
      statusBadge.className = "inline-block text-xs px-3 py-1 rounded-full bg-rose-500/20 text-rose-300";
    } else {
      const app = initializeApp({ databaseURL: DATABASE_URL });
      const db = getDatabase(app);
      const songsRef = ref(db, 'karaoke_songs');

      onValue(songsRef, (snapshot) => {
        const data = snapshot.val();
        songs = data ? Object.entries(data).map(([id, val]) => ({ id, ...val })) : [];
        statusBadge.textContent = "🟢 เชื่อมต่อออนไลน์แล้ว (ซิงก์ทุกคน)";
        statusBadge.className = "inline-block text-xs px-3 py-1 rounded-full bg-emerald-500/20 text-emerald-300";
        renderSongs();
      }, (error) => {
        statusBadge.textContent = "❌ เชื่อมต่อไม่สำเร็จ: " + error.message;
        statusBadge.className = "inline-block text-xs px-3 py-1 rounded-full bg-rose-500/20 text-rose-300";
      });

      window.saveSongToDB = (title, artist) => {
        if (!title.trim()) return alert("กรุณากรอกชื่อเพลงก่อนครับ");
        push(songsRef, {
          title: title.trim(),
          artist: artist.trim() || "ไม่ระบุศิลปิน",
          createdAt: Date.now()
        });
        document.getElementById('songTitle').value = '';
        document.getElementById('songArtist').value = '';
        document.getElementById('searchResults').classList.add('hidden');
      };

      window.markAsSung = (id) => {
        remove(ref(db, `karaoke_songs/${id}`));
        if (currentPickedId === id) {
          currentPickedId = null;
          document.getElementById('randomDisplay').innerHTML = `<p class="text-emerald-400 font-medium">✅ ลบเพลงที่ร้องแล้วออกจากคลังเรียบร้อย!</p>`;
          document.getElementById('randomActions').innerHTML = `
            <button onclick="pickRandomSong()" class="px-6 py-3 rounded-xl bg-gradient-to-r from-pink-600 to-purple-600 hover:from-pink-500 hover:to-purple-500 font-semibold shadow-lg transition">
              🎲 สุ่มเพลงถัดไป!
            </button>`;
        }
      };
    }

    window.addSongDirect = () => {
      const title = document.getElementById('songTitle').value;
      const artist = document.getElementById('songArtist').value;
      if (window.saveSongToDB) window.saveSongToDB(title, artist);
    };

    window.searchPlatforms = () => {
      const title = document.getElementById('songTitle').value.trim();
      const artist = document.getElementById('songArtist').value.trim();
      if (!title) return alert("พิมพ์ชื่อเพลงก่อนกดค้นหาครับ");

      const query = encodeURIComponent(`${title} ${artist}`.trim());
      const karaokeQuery = encodeURIComponent(`${title} ${artist} karaoke`.trim());
      const safeTitle = title.replace(/'/g, "\\'");
      const safeArtist = artist.replace(/'/g, "\\'");

      const resultsDiv = document.getElementById('searchResults');
      resultsDiv.classList.remove('hidden');
      resultsDiv.innerHTML = `
        <div class="p-4 rounded-xl bg-slate-900/90 border border-slate-700 space-y-3">
          <div class="flex flex-col sm:flex-row sm:items-center justify-between gap-2">
            <div>
              <p class="font-semibold text-white text-base">${title}</p>
              <p class="text-xs text-slate-400">${artist || 'ไม่ระบุศิลปิน'}</p>
            </div>
            <button onclick="saveSongToDB('${safeTitle}', '${safeArtist}')" class="px-4 py-2 rounded-lg bg-emerald-600 hover:bg-emerald-500 text-sm font-semibold whitespace-nowrap">
              ➕ บันทึกเพลงนี้ลงคลัง
            </button>
          </div>
          <div class="grid grid-cols-2 sm:grid-cols-4 gap-2 pt-2 text-xs text-center">
            <a href="https://www.youtube.com/results?search_query=${karaokeQuery}" target="_blank" class="py-2 px-2 rounded-lg bg-red-600/20 text-red-300 border border-red-500/30 hover:bg-red-600/30">▶️ YouTube Karaoke</a>
            <a href="https://open.spotify.com/search/${query}" target="_blank" class="py-2 px-2 rounded-lg bg-green-600/20 text-green-300 border border-green-500/30 hover:bg-green-600/30">🎧 Spotify</a>
            <a href="https://music.apple.com/th/search?term=${query}" target="_blank" class="py-2 px-2 rounded-lg bg-pink-600/20 text-pink-300 border border-pink-500/30 hover:bg-pink-600/30">🎵 Apple Music</a>
            <a href="https://www.joox.com/th/search/${query}" target="_blank" class="py-2 px-2 rounded-lg bg-amber-600/20 text-amber-300 border border-amber-500/30 hover:bg-amber-600/30">🎤 JOOX</a>
          </div>
        </div>
      `;
    };

    window.pickRandomSong = () => {
      if (songs.length === 0) return alert("ยังไม่มีเพลงในคลังให้สุ่มครับ!");
      const display = document.getElementById('randomDisplay');
      const actions = document.getElementById('randomActions');
      let count = 0;

      const interval = setInterval(() => {
        const temp = songs[Math.floor(Math.random() * songs.length)];
        display.innerHTML = `<p class="text-xl font-bold text-purple-300 animate-pulse">🎵 ${temp.title}</p><p class="text-xs text-slate-400">${temp.artist}</p>`;
        count++;
        if (count > 12) {
          clearInterval(interval);
          const chosen = songs[Math.floor(Math.random() * songs.length)];
          currentPickedId = chosen.id;
          const ytLink = `https://www.youtube.com/results?search_query=${encodeURIComponent(chosen.title + ' ' + chosen.artist + ' karaoke')}`;
          
          display.innerHTML = `
            <span class="text-xs uppercase tracking-wider text-pink-400 font-semibold mb-1">🎉 เพลงที่สุ่มได้คือ</span>
            <p class="text-2xl font-bold text-white">${chosen.title}</p>
            <p class="text-sm text-slate-300">${chosen.artist}</p>
          `;
          actions.innerHTML = `
            <a href="${ytLink}" target="_blank" class="px-5 py-3 rounded-xl bg-red-600 hover:bg-red-500 font-semibold shadow-lg transition">
              ▶️ เปิดร้องบน YouTube
            </a>
            <button onclick="markAsSung('${chosen.id}')" class="px-5 py-3 rounded-xl bg-emerald-600 hover:bg-emerald-500 font-semibold shadow-lg transition">
              ✅ ร้องแล้ว (ลบออกจากคลัง)
            </button>
            <button onclick="pickRandomSong()" class="px-4 py-3 rounded-xl bg-slate-700 hover:bg-slate-600 font-medium transition">
              🔄 สุ่มใหม่
            </button>
          `;
        }
      }, 70);
    };

    function renderSongs() {
      const list = document.getElementById('songList');
      document.getElementById('songCount').textContent = songs.length;
      if (songs.length === 0) {
        list.innerHTML = `<p class="text-center text-slate-500 py-6 text-sm">ยังไม่มีเพลงในคลัง เพิ่มเพลงแรกได้เลย!</p>`;
        return;
      }
      list.innerHTML = songs.map((s, idx) => {
        const ytLink = `https://www.youtube.com/results?search_query=${encodeURIComponent(s.title + ' ' + s.artist + ' karaoke')}`;
        return `
          <div class="flex items-center justify-between p-3.5 rounded-xl bg-slate-900/70 border border-slate-800 hover:border-slate-700 transition gap-3">
            <div class="min-w-0">
              <p class="font-medium text-white truncate"><span class="text-purple-400 mr-2">#${idx + 1}</span>${s.title}</p>
              <p class="text-xs text-slate-400 truncate">${s.artist}</p>
            </div>
            <div class="flex items-center gap-2 shrink-0">
              <a href="${ytLink}" target="_blank" class="px-3 py-1.5 rounded-lg bg-red-600/20 text-red-300 hover:bg-red-600/30 text-xs">▶️ ร้อง</a>
              <button onclick="markAsSung('${s.id}')" class="px-3 py-1.5 rounded-lg bg-slate-800 hover:bg-emerald-600/30 text-slate-300 hover:text-emerald-300 text-xs transition">
                ✅ ร้องแล้ว
              </button>
            </div>
          </div>
        `;
      }).join('');
    }
  </script>
</body>
</html>
