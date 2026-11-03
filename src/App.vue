<template>
  <div class="app">
    <header class="header">
      <div class="wrap header-in">
        <div class="brand">
          <img v-if="logoOk" class="logo-img" :src="logoSrc" alt="Logo D4T" @error="logoOk = false" />
          <div v-else class="logo">D4T</div>
          <div><strong>Impact Tracker</strong><small>Days For Travel</small></div>
        </div>
        <nav class="nav" aria-label="Menu utama">
          <button v-for="t in tabs" :key="t.id" :class="{ on: activeTab === t.id }" :aria-current="activeTab === t.id ? 'page' : undefined" @click="activeTab = t.id">{{ t.label }}</button>
        </nav>
      </div>
    </header>

    <section class="hero">
      <div class="wrap hero-in">
        <div>
          <h1>Perjalanan yang memulihkan diri, alam, dan komunitas.</h1>
          <p>Teknologi D4T mengukur emisi, memantau aliran material, dan mencatat aksi regeneratif setiap peserta, dari perencanaan sampai penanaman pohon.</p>
        </div>
        <dl class="kpis">
          <div v-for="k in kpi" :key="k.l"><dd>{{ k.v }}</dd><dt>{{ k.l }}</dt></div>
        </dl>
      </div>
    </section>

    <!-- MODAL PROYEK -->
    <div v-if="showModal" class="overlay" @click.self="showModal = false">
      <div class="modal" role="dialog" aria-modal="true" aria-labelledby="jm">
        <h3 id="jm">Tambah proyek baru</h3>
        <label class="field"><span>Nama program / trip</span><input v-model="newProject.nama" type="text" placeholder="Contoh: Garden Escape Lembang" /></label>
        <div class="grid2">
          <label class="field"><span>Jenis trip</span>
            <select v-model="newProject.tipe"><option>Trial Trip</option><option>Open Trip</option><option>Private Trip</option></select>
          </label>
          <label class="field"><span>Target peserta (pax)</span><input v-model.number="newProject.pax" type="number" min="1" /></label>
        </div>
        <p v-if="errProyek" class="error">{{ errProyek }}</p>
        <div class="btn-row end">
          <button class="btn btn-ghost" @click="showModal = false">Batal</button>
          <button class="btn btn-primary" @click="tambahProyek">Simpan proyek</button>
        </div>
      </div>
    </div>

    <datalist id="daftar-trip"><option v-for="p in daftarProyek" :key="p.id" :value="p.nama" /></datalist>

    <!-- OPERASIONAL -->
    <main v-if="activeTab === 'operasional'" class="wrap page">
      <div class="page-head"><h2>Alur operasional</h2><p>Satu proyek berjalan dari perencanaan sampai pencatatan dampak. Centang 5R saat trip berlangsung.</p></div>
      <div class="kanban">
        <section v-for="k in kolom" :key="k.tahap" class="col" :style="{ '--c': k.warna }">
          <h3><span class="leaf">{{ k.ikon }}</span>{{ k.judul }}<em>{{ proyekDi(k.tahap).length }}</em></h3>
          <article v-for="p in proyekDi(k.tahap)" :key="p.id" class="card-p">
            <div class="between"><h4>{{ p.nama }}</h4><span class="tag">{{ p.tipe }}</span></div>
            <p class="muted sm">Target {{ p.pax }} pax</p>
            <div v-if="k.tahap === 'Pelaksanaan'" class="checks">
              <label v-for="r in R5" :key="r.k"><input v-model="p.r[r.k]" type="checkbox" /> <b>{{ r.nama }}</b> {{ r.hint }}</label>
            </div>
            <button v-if="k.aksi" class="btn btn-soft btn-block" @click="k.tahap === 'Tracking' ? bukaKalkulator(p) : (p.tahap = k.berikut)">{{ k.aksi }}</button>
          </article>
          <p v-if="!proyekDi(k.tahap).length && k.tahap !== 'Perencanaan'" class="empty">{{ k.kosong }}</p>
          <button v-if="k.tahap === 'Perencanaan'" class="add" @click="showModal = true">+ Tambah proyek</button>
        </section>
      </div>
    </main>

    <!-- KALKULATOR -->
    <main v-else-if="activeTab === 'kalkulator'" class="wrap page calc">
      <div>
        <div class="page-head"><h2>Kalkulator karbon dan pohon offset</h2><p>Isi data program, lalu hitung emisi dan jumlah pohon yang perlu ditanam.</p></div>
        <div class="chips" role="tablist">
          <button v-for="j in jenis" :key="j.id" role="tab" :aria-selected="jenisAktif === j.id" class="chip" :class="{ on: jenisAktif === j.id }" @click="jenisAktif = j.id"><span>{{ j.ikon }}</span>{{ j.label }}</button>
        </div>
        <div class="card form">
          <div v-show="jenisAktif === 'program'" class="grid2">
            <label class="field full"><span>Nama program wisata</span><input v-model="form.nama" type="text" placeholder="Masukkan nama tur" /></label>
            <label class="field"><span>Jumlah peserta (pax)</span><input v-model.number="form.pax" type="number" min="1" /></label>
            <label class="field"><span>Biaya per pohon (Rp)</span><input v-model.number="form.hargaPohon" type="number" min="0" /></label>
          </div>
          <div v-show="jenisAktif === 'transport'" class="grid2">
            <label class="field"><span>Jarak udara (km)</span><input v-model.number="form.kmPesawat" type="number" min="0" /></label>
            <label class="field"><span>Jarak darat / bus (km)</span><input v-model.number="form.kmBus" type="number" min="0" /></label>
          </div>
          <div v-show="jenisAktif === 'akomodasi'" class="grid2">
            <label class="field"><span>Malam di hotel standar</span><input v-model.number="form.malamHotel" type="number" min="0" /></label>
            <label class="field"><span>Malam di eco-lodge</span><input v-model.number="form.malamEco" type="number" min="0" /></label>
          </div>
          <div v-show="jenisAktif === 'limbah'" class="grid2">
            <label class="field"><span>Plastik sekali pakai (pcs per pax)</span><input v-model.number="form.plastik" type="number" min="0" /></label>
            <label class="field"><span>Potensi sisa makanan (kg per pax)</span><input v-model.number="form.sisaMakanan" type="number" min="0" /></label>
            <label class="switch full">
              <input v-model="form.is5R" type="checkbox" /><i></i>
              <div><strong>Terapkan sirkularitas 5R</strong><small>Tolak plastik dan kurangi sisa makanan lewat porsi terukur</small></div>
            </label>
          </div>
          <p v-if="errHitung" class="error">{{ errHitung }}</p>
          <button class="btn btn-primary btn-block big" @click="kalkulasiDampak">Analisis dampak ekologi</button>
        </div>
      </div>

      <aside class="result-col">
        <div v-if="!hud.calculated" class="card placeholder"><div class="ph-ico">🌍</div><h3>Menunggu data</h3><p class="muted">Laporan emisi akan muncul di sini setelah analisis dijalankan.</p></div>
        <div v-else class="report">
          <div class="rp-top">
            <small>Laporan dampak</small><h3>{{ form.nama }}</h3>
            <p class="big-num">{{ hud.emisiBersih.toFixed(1) }} <span>kg CO₂e</span></p>
            <div class="score"><div class="bar"><i :style="{ width: hud.score + '%' }"></i></div><span>{{ skor }} ({{ hud.score }}/100)</span></div>
          </div>
          <div class="rp-sec">
            <h4>Sumber emisi</h4>
            <div v-for="b in rincian" :key="b.label" class="row-b">
              <div class="between"><span>{{ b.label }}</span><b>{{ b.nilai.toFixed(1) }} kg</b></div>
              <div class="bar sm"><i :style="{ width: b.persen + '%' }"></i></div>
            </div>
          </div>
          <div class="rp-sec">
            <h4>Efisiensi material</h4>
            <div class="between"><span>Plastik dihasilkan</span><b :class="{ cut: form.is5R }">{{ hud.limbah.plastikRaw }} pcs</b></div>
            <div class="between"><span>Sisa makanan</span><b>{{ (form.is5R ? hud.limbah.makananRaw * (1 - REDUKSI_MAKANAN) : hud.limbah.makananRaw).toFixed(1) }} kg</b></div>
          </div>
          <div class="rp-foot">
            <div><strong>🌳 {{ hud.pohon }}</strong><small>pohon ditanam</small></div>
            <div class="right"><strong>{{ formatRupiah(hud.biayaTotal) }}</strong><small>dana offset</small></div>
          </div>
        </div>
      </aside>
    </main>

    <!-- ALIRAN MATERIAL -->
    <main v-else-if="activeTab === 'material'" class="wrap page">
      <div class="page-head"><h2>Aliran material</h2><p>Pantau perjalanan alat guna ulang dari gudang ke peserta dan kembali, serta ke mana limbah trip berakhir.</p></div>
      <div class="two">
        <section class="card">
          <h3 class="h3">Reuse Kit: sirkulasi alat guna ulang</h3>
          <p class="muted sm">Setiap alat yang kembali menggantikan satu barang sekali pakai.</p>
          <div v-for="k in kit" :key="k.id" class="kit">
            <div class="between"><strong>{{ k.nama }}</strong><span class="tag">{{ tersedia(k) }} tersedia dari {{ k.total }}</span></div>
            <div class="kit-stats"><span>Dipinjam <b>{{ k.keluar }}</b></span><span>Sudah kembali <b>{{ k.uses }}</b></span><span>Hilang / rusak <b>{{ k.hilang }}</b></span></div>
            <div class="kit-act">
              <input v-model.number="k.n" type="number" min="1" aria-label="Jumlah" />
              <button class="btn btn-soft" @click="pinjam(k)">Pinjamkan</button>
              <button class="btn btn-soft" @click="kembali(k)">Kembali</button>
              <button class="btn btn-soft" @click="hilangkan(k)">Hilang</button>
              <button class="btn btn-danger" @click="hapusKit(k)">Hapus alat</button>
            </div>
          </div>
          <p v-if="errKit" class="error">{{ errKit }}</p>
          <div class="addrow">
            <input v-model="nk.nama" type="text" placeholder="Alat baru, mis. sedotan stainless" aria-label="Nama alat" />
            <input v-model.number="nk.total" type="number" min="1" aria-label="Jumlah stok" />
            <button class="btn btn-primary" @click="tambahKit">Tambah</button>
          </div>
        </section>

        <section class="card">
          <h3 class="h3">Log limbah per trip</h3>
          <p class="muted sm">Timbang limbah di akhir trip, lalu catat tujuannya.</p>
          <div class="addrow">
            <input v-model="wl.trip" list="daftar-trip" type="text" placeholder="Nama trip" aria-label="Nama trip" />
            <select v-model="wl.tujuan" aria-label="Tujuan"><option v-for="t in TUJUAN" :key="t.k" :value="t.k">{{ t.label }}</option></select>
            <input v-model.number="wl.kg" type="number" min="0" step="0.1" aria-label="Berat (kg)" />
            <button class="btn btn-primary" @click="catatLimbah">Catat</button>
          </div>
          <p v-if="errLimbah" class="error">{{ errLimbah }}</p>
          <div class="diversi"><strong>{{ pct(diversi) }}</strong><span>limbah terselamatkan dari TPA</span></div>
          <div v-for="t in TUJUAN" :key="t.k" class="row-b">
            <div class="between"><span>{{ t.label }}</span><b>{{ totalLimbah[t.k].toFixed(1) }} kg</b></div>
            <div class="bar sm"><i :style="{ width: (sumLimbah ? (totalLimbah[t.k] / sumLimbah) * 100 : 0) + '%' }"></i></div>
          </div>
          <p v-if="!logLimbah.length" class="empty">Belum ada limbah tercatat.</p>
          <ul v-else class="log">
            <li v-for="x in logLimbah.slice(0, 5)" :key="x.id">
              <span class="trip">{{ x.trip || 'Tanpa trip' }}</span><span class="muted">{{ labelTujuan(x.tujuan) }}</span><b>{{ x.kg }} kg</b>
              <button class="x" :aria-label="'Hapus catatan ' + (x.trip || 'tanpa trip')" @click="hapusLog(x.id)">✕</button>
            </li>
          </ul>
          <div v-if="logLimbah.length" class="between log-foot">
            <small class="muted">Menampilkan {{ Math.min(5, logLimbah.length) }} dari {{ logLimbah.length }} catatan</small>
            <button class="btn btn-danger" @click="hapusSemuaLog">Hapus semua log</button>
          </div>
        </section>
      </div>
    </main>

    <!-- IMPACT PASSPORT -->
    <main v-else-if="activeTab === 'passport'" class="wrap page">
      <div class="page-head"><h2>Impact Passport</h2><p>Setiap peserta mengumpulkan stempel dari aksi 5R selama trip. Pemandu menandai stempel di lapangan.</p></div>
      <div class="card addpass">
        <input v-model="np.nama" type="text" placeholder="Nama peserta" aria-label="Nama peserta" />
        <input v-model="np.trip" list="daftar-trip" type="text" placeholder="Trip (opsional)" aria-label="Trip" />
        <button class="btn btn-primary" @click="tambahPassport">Terbitkan passport</button>
      </div>
      <p v-if="errPass" class="error">{{ errPass }}</p>
      <p v-if="!passports.length" class="empty wide">Belum ada passport. Terbitkan satu untuk setiap peserta.</p>
      <div class="cards">
        <article v-for="p in passports" :key="p.id" class="passport">
          <header><div><h4>{{ p.nama }}</h4><small>{{ p.trip || 'Tanpa trip' }}</small></div><span class="tag lvl">{{ level(p) }}</span></header>
          <div class="stamps">
            <button v-for="s in STAMPS" :key="s.k" class="stamp" :class="{ earned: p.stamps[s.k] }" :aria-pressed="p.stamps[s.k]" :title="s.hint" @click="p.stamps[s.k] = !p.stamps[s.k]">
              <i>{{ p.stamps[s.k] ? '✓' : s.ikon }}</i><span>{{ s.nama }}</span>
            </button>
          </div>
          <div class="bar sm"><i :style="{ width: (jumlahStamp(p) / STAMPS.length) * 100 + '%' }"></i></div>
          <small class="muted">{{ jumlahStamp(p) }} dari {{ STAMPS.length }} stempel</small>
        </article>
      </div>
    </main>

    <!-- PORTOFOLIO -->
    <main v-else class="wrap page">
      <div class="page-head"><h2>Portofolio dampak</h2><p>Bukti bahwa teknologi D4T bekerja untuk praktik sirkular, bukan sekadar promosi.</p></div>
      <div class="two">
        <section v-for="b in bukti" :key="b.judul" class="card bukti">
          <h3 class="h3">{{ b.judul }}</h3>
          <p class="muted sm">{{ b.isi }}</p>
          <dl><div v-for="a in b.angka" :key="a.l"><dd>{{ a.v }}</dd><dt>{{ a.l }}</dt></div></dl>
        </section>
      </div>
      <h3 class="h3 gap">Riwayat kalkulasi</h3>
      <div class="cards">
        <p v-if="!historyData.length" class="empty wide">Belum ada riwayat. Hitung dampak sebuah program di tab Kalkulator.</p>
        <article v-for="(item, i) in historyData" :key="i" class="card pf">
          <header><h4>{{ item.nama }}</h4><span class="tag">{{ item.pax }} pax</span></header>
          <dl>
            <div><dt>Jejak emisi</dt><dd>{{ item.emisi }} kg</dd></div>
            <div><dt>Sirkularitas 5R</dt><dd :class="item.is5R ? 'ok' : 'off'">{{ item.is5R ? 'Aktif' : 'Nonaktif' }}</dd></div>
          </dl>
          <footer><span>🌳 {{ item.pohon }} pohon</span><b>{{ formatRupiah(item.totalBiaya) }}</b></footer>
        </article>
      </div>
    </main>

    <footer class="footer"><div class="wrap">© 2026 Days For Travel. To pioneer a sustainable future of tourism.</div></footer>
  </div>
</template>

<script setup>
import { ref, reactive, computed } from 'vue'

const activeTab = ref('operasional')
const showModal = ref(false)
const logoOk = ref(true)
const logoSrc = '/logo-d4t.svg' // file ada di folder public/
const jenisAktif = ref('program')
const errProyek = ref(''), errHitung = ref(''), errKit = ref(''), errLimbah = ref(''), errPass = ref('')

const tabs = [
  { id: 'operasional', label: 'Operasional' }, { id: 'kalkulator', label: 'Kalkulator' },
  { id: 'material', label: 'Aliran Material' }, { id: 'passport', label: 'Impact Passport' },
  { id: 'riwayat', label: 'Portofolio' }
]
const jenis = [
  { id: 'program', ikon: '📝', label: 'Program' }, { id: 'transport', ikon: '🚌', label: 'Transportasi' },
  { id: 'akomodasi', ikon: '🏨', label: 'Akomodasi' }, { id: 'limbah', ikon: '♻️', label: 'Limbah' }
]
const R5 = [
  { k: 'refuse', nama: 'Refuse', hint: 'tolak plastik sekali pakai', ikon: '🚫' },
  { k: 'reduce', nama: 'Reduce', hint: 'minimalkan sampah', ikon: '➖' },
  { k: 'reuse', nama: 'Reuse', hint: 'pakai Reuse Kit', ikon: '🔁' },
  { k: 'recycle', nama: 'Recycle', hint: 'pilah dan setor sampah', ikon: '♻️' },
  { k: 'regenerate', nama: 'Regenerate', hint: 'ikut menanam pohon', ikon: '🌱' }
]
const STAMPS = R5.map((r) => ({ ...r, hint: r.hint[0].toUpperCase() + r.hint.slice(1) }))
const kolom = [
  { tahap: 'Perencanaan', judul: 'Perencanaan', ikon: '🗺️', warna: '#7a8f89', aksi: 'Lanjut ke pemasaran', berikut: 'Pemasaran' },
  { tahap: 'Pemasaran', judul: 'Pemasaran', ikon: '📢', warna: '#3b82b6', aksi: 'Mulai pelaksanaan trip', berikut: 'Pelaksanaan', kosong: 'Belum ada proyek di tahap ini.' },
  { tahap: 'Pelaksanaan', judul: 'Pelaksanaan 5R', ikon: '♻️', warna: '#c98a14', aksi: 'Selesai dan catat dampak', berikut: 'Tracking', kosong: 'Belum ada trip berjalan.' },
  { tahap: 'Tracking', judul: 'Impact tracking', ikon: '🍃', warna: '#0b7a62', aksi: 'Buka kalkulator', kosong: 'Belum ada data untuk dihitung.' }
]
const pct = (v) => (v === null ? '–' : v + '%')

// --- KANBAN ---
const daftarProyek = ref([])
const newProject = reactive({ nama: '', pax: 1, tipe: 'Trial Trip' })
const proyekDi = (t) => daftarProyek.value.filter((p) => p.tahap === t)
const tambahProyek = () => {
  if (!newProject.nama.trim()) { errProyek.value = 'Nama proyek harus diisi.'; return }
  daftarProyek.value.push({
    id: Date.now(), nama: newProject.nama.trim(), pax: newProject.pax > 0 ? newProject.pax : 1,
    tipe: newProject.tipe, tahap: 'Perencanaan',
    r: { refuse: false, reduce: false, reuse: false, recycle: false, regenerate: false }
  })
  Object.assign(newProject, { nama: '', pax: 1, tipe: 'Trial Trip' })
  errProyek.value = ''; showModal.value = false
}
const bukaKalkulator = (p) => {
  form.nama = p.nama; form.pax = p.pax
  form.is5R = Object.values(p.r).every(Boolean)
  hud.calculated = false; jenisAktif.value = 'transport'; activeTab.value = 'kalkulator'
}

// --- KALKULATOR ---
const EF = { BUS: 0.03, PESAWAT: 0.2, HOTEL: 25.0, ECOLODGE: 5.0, POHON: 25.0, PLASTIK: 0.5, SISA_MAKANAN: 2.5 }
const REDUKSI_MAKANAN = 0.5 // pengurangan sisa makanan saat 5R aktif
const form = reactive({ nama: '', pax: 1, hargaPohon: 25000, kmBus: 0, kmPesawat: 0, malamHotel: 0, malamEco: 0, plastik: 0, sisaMakanan: 0, is5R: false })
const hud = reactive({ calculated: false, emisiBersih: 0, pohon: 0, biayaTotal: 0, score: 0, emTransport: 0, emAkomodasi: 0, emLimbah: 0, limbah: { plastikRaw: 0, makananRaw: 0 } })
const historyData = ref([])
const formatRupiah = (v) => new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', minimumFractionDigits: 0 }).format(v || 0)

const kalkulasiDampak = () => {
  if (!form.nama.trim()) { errHitung.value = 'Nama program wisata wajib diisi di tab Program.'; jenisAktif.value = 'program'; return }
  errHitung.value = ''
  const pax = form.pax > 0 ? form.pax : 1
  const emTransport = form.kmBus * EF.BUS * pax + form.kmPesawat * EF.PESAWAT * pax
  const emAkomodasi = form.malamHotel * EF.HOTEL + form.malamEco * EF.ECOLODGE
  const beratPlastik = form.plastik * 0.02 * pax
  const beratMakanan = form.sisaMakanan * pax
  const emLimbah = form.is5R
    ? beratMakanan * (1 - REDUKSI_MAKANAN) * EF.SISA_MAKANAN
    : beratPlastik * EF.PLASTIK + beratMakanan * EF.SISA_MAKANAN
  const emBersih = emTransport + emAkomodasi + emLimbah
  const pohon = Math.ceil(emBersih / EF.POHON)
  const totalBiaya = pohon * form.hargaPohon
  Object.assign(hud, { emisiBersih: emBersih, pohon, biayaTotal: totalBiaya, emTransport, emAkomodasi, emLimbah })
  hud.limbah.plastikRaw = form.plastik * pax
  hud.limbah.makananRaw = beratMakanan
  let s = 100 - Math.min((emBersih / pax / 100) * 100, 100)
  if (form.is5R) s = Math.min(s + 15, 100)
  hud.score = Math.round(s)
  hud.calculated = true
  historyData.value.push({ nama: form.nama, pax, emisi: emBersih.toFixed(1), is5R: form.is5R, pohon, totalBiaya })
}
const skor = computed(() => (hud.score >= 70 ? 'Emisi rendah' : hud.score >= 40 ? 'Emisi sedang' : 'Emisi tinggi'))
const rincian = computed(() => {
  const t = hud.emisiBersih || 1
  return [['Transportasi', hud.emTransport], ['Akomodasi', hud.emAkomodasi], ['Limbah', hud.emLimbah]]
    .map(([label, nilai]) => ({ label, nilai, persen: Math.round((nilai / t) * 100) }))
})

// --- REUSE KIT ---
const kit = ref([
  { id: 1, nama: 'Tumbler', total: 10, keluar: 0, uses: 0, hilang: 0, n: 1 },
  { id: 2, nama: 'Wadah makan', total: 10, keluar: 0, uses: 0, hilang: 0, n: 1 },
  { id: 3, nama: 'Tas kain', total: 10, keluar: 0, uses: 0, hilang: 0, n: 1 }
])
const nk = reactive({ nama: '', total: 10 })
const tersedia = (k) => k.total - k.keluar - k.hilang
const jml = (k) => Math.max(1, Math.floor(k.n) || 1)
const pinjam = (k) => {
  if (jml(k) > tersedia(k)) { errKit.value = `Stok ${k.nama} yang tersedia hanya ${tersedia(k)}.`; return }
  errKit.value = ''; k.keluar += jml(k)
}
const kembali = (k) => {
  if (jml(k) > k.keluar) { errKit.value = `${k.nama} yang sedang dipinjam hanya ${k.keluar}.`; return }
  errKit.value = ''; k.keluar -= jml(k); k.uses += jml(k)
}
const hilangkan = (k) => {
  if (jml(k) > k.keluar) { errKit.value = `${k.nama} yang sedang dipinjam hanya ${k.keluar}.`; return }
  errKit.value = ''; k.keluar -= jml(k); k.hilang += jml(k)
}
const tambahKit = () => {
  if (!nk.nama.trim() || !(nk.total > 0)) { errKit.value = 'Isi nama alat dan jumlah stok.'; return }
  kit.value.push({ id: Date.now(), nama: nk.nama.trim(), total: nk.total, keluar: 0, uses: 0, hilang: 0, n: 1 })
  nk.nama = ''; errKit.value = ''
}

const hapusKit = (k) => {
  if (!confirm(`Hapus "${k.nama}" dari daftar alat?`)) return
  kit.value = kit.value.filter((x) => x.id !== k.id); errKit.value = ''
}

// --- LOG LIMBAH ---
const TUJUAN = [{ k: 'daur', label: 'Didaur ulang' }, { k: 'guna', label: 'Digunakan ulang' }, { k: 'residu', label: 'Residu ke TPA' }]
const labelTujuan = (k) => TUJUAN.find((t) => t.k === k)?.label
const logLimbah = ref([])
const wl = reactive({ trip: '', tujuan: 'daur', kg: 0 })
const catatLimbah = () => {
  if (!(wl.kg > 0)) { errLimbah.value = 'Isi berat limbah lebih dari 0 kg.'; return }
  logLimbah.value.unshift({ id: Date.now(), trip: wl.trip.trim(), tujuan: wl.tujuan, kg: wl.kg })
  wl.kg = 0; errLimbah.value = ''
}
const hapusLog = (id) => { logLimbah.value = logLimbah.value.filter((x) => x.id !== id) }
const hapusSemuaLog = () => { if (confirm('Hapus semua catatan limbah?')) logLimbah.value = [] }
const totalLimbah = computed(() => {
  const t = { daur: 0, guna: 0, residu: 0 }
  logLimbah.value.forEach((x) => (t[x.tujuan] += x.kg))
  return t
})
const sumLimbah = computed(() => totalLimbah.value.daur + totalLimbah.value.guna + totalLimbah.value.residu)
const diversi = computed(() => (sumLimbah.value ? Math.round(((totalLimbah.value.daur + totalLimbah.value.guna) / sumLimbah.value) * 100) : null))

// --- IMPACT PASSPORT ---
const passports = ref([])
const np = reactive({ nama: '', trip: '' })
const tambahPassport = () => {
  if (!np.nama.trim()) { errPass.value = 'Nama peserta harus diisi.'; return }
  passports.value.push({ id: Date.now(), nama: np.nama.trim(), trip: np.trip.trim(), stamps: Object.fromEntries(STAMPS.map((s) => [s.k, false])) })
  np.nama = ''; errPass.value = ''
}
const jumlahStamp = (p) => STAMPS.filter((s) => p.stamps[s.k]).length
const level = (p) => (jumlahStamp(p) >= 4 ? 'Pohon' : jumlahStamp(p) >= 2 ? 'Tunas' : 'Benih')

// --- RINGKASAN ---
const plastikDihindari = computed(() => kit.value.reduce((a, k) => a + k.uses, 0))
const returnRate = computed(() => {
  const u = plastikDihindari.value, h = kit.value.reduce((a, k) => a + k.hilang, 0)
  return u + h ? Math.round((u / (u + h)) * 100) : null
})
const totalPohon = computed(() => historyData.value.reduce((a, x) => a + x.pohon, 0))
const kpi = computed(() => [
  { l: 'plastik sekali pakai dihindari', v: plastikDihindari.value },
  { l: 'alat guna ulang kembali', v: pct(returnRate.value) },
  { l: 'pohon target tanam', v: totalPohon.value }
])
const bukti = computed(() => [
  {
    judul: 'Fungsi jelas untuk praktik sirkular',
    isi: 'Tracker mengukur emisi tiap trip, Reuse Kit mencatat sirkulasi alat, dan Passport mencatat aksi 5R setiap peserta.',
    angka: [{ l: 'program terhitung emisinya', v: historyData.value.length }, { l: 'pohon target tanam', v: totalPohon.value }, { l: 'peserta ber-passport', v: passports.value.length }]
  },
  {
    judul: 'Efisiensi sumber daya dan pengurangan limbah',
    isi: 'Data aliran material menunjukkan berapa barang sekali pakai tergantikan dan berapa limbah yang tidak berakhir di TPA.',
    angka: [{ l: 'plastik sekali pakai dihindari', v: plastikDihindari.value }, { l: 'alat guna ulang kembali', v: pct(returnRate.value) }, { l: 'limbah terselamatkan dari TPA', v: pct(diversi.value) }]
  }
])
</script>

<style scoped>
.app {
  overflow-x: clip;
  --jade: #0b4f3f; --jade-d: #073229; --teal: #8fd1c4; --sprout: #c9e4a0; --mist: #eef4f0;
  --line: #d7e4dd; --ink: #17302a; --mute: #5b726b; --warn: #b3261e;
  font-family: 'Plus Jakarta Sans', system-ui, -apple-system, 'Segoe UI', sans-serif;
  color: var(--ink); background: var(--mist); min-height: 100vh; line-height: 1.55;
}
.app *, .app *::before, .app *::after { box-sizing: border-box; }
.app h1, .app h2, .app h3, .app h4, .app p, .app dl, .app dd, .app dt, .app ul { margin: 0; padding: 0; }
.app button { font: inherit; cursor: pointer; }
.app :focus-visible { outline: 3px solid var(--teal); outline-offset: 2px; }
.wrap { max-width: 1160px; margin: 0 auto; padding: 0 24px; }
.page { padding-top: 40px; padding-bottom: 64px; }
.muted { color: var(--mute); } .sm { font-size: 13px; } .between { display: flex; justify-content: space-between; align-items: center; gap: 8px; }
.app h1, .page-head h2, .h3, .rp-top h3, .passport h4 { font-family: 'Fraunces', Georgia, 'Times New Roman', serif; }

/* header */
.header { background: var(--jade-d); position: sticky; top: 0; z-index: 50; }
.header-in { display: flex; align-items: center; justify-content: space-between; gap: 16px; height: 72px; }
.brand { display: flex; align-items: center; gap: 12px; }
.brand strong { display: block; color: #fff; font-size: 17px; line-height: 1.1; } .brand small { color: var(--teal); font-size: 12px; }
.logo-img { height: 46px; width: auto; }
.logo { width: 42px; height: 42px; border-radius: 50%; background: var(--jade); color: #fff; font-weight: 800; font-size: 14px; display: grid; place-items: center; }
.nav { display: flex; gap: 4px; }
.nav button { background: none; border: 0; color: #b9d8d0; padding: 9px 14px; border-radius: 999px; font-weight: 600; white-space: nowrap; }
.nav button:hover { color: #fff; } .nav button.on { background: var(--teal); color: var(--jade-d); }

/* hero */
.hero { background: radial-gradient(120% 140% at 85% -20%, #2c8a74 0%, transparent 55%), radial-gradient(90% 120% at 0% 110%, #14624f 0%, transparent 60%), var(--jade); color: #fff; }
.hero-in { display: grid; grid-template-columns: minmax(0, 1.4fr) minmax(0, 1fr); gap: 40px; align-items: center; padding-top: 48px; padding-bottom: 48px; }
.hero h1 { font-size: clamp(28px, 4vw, 44px); line-height: 1.12; font-weight: 700; letter-spacing: -0.015em; max-width: 18ch; }
.hero p { margin-top: 16px; max-width: 54ch; color: #d4ece6; font-size: 16px; }
.kpis { display: grid; gap: 10px; }
.kpis div { background: rgba(255, 255, 255, .09); border: 1px solid rgba(143, 209, 196, .25); border-radius: 22px 6px 22px 6px; padding: 12px 20px; display: flex; align-items: baseline; gap: 14px; }
.kpis dd { font-size: 28px; font-weight: 800; min-width: 3ch; color: var(--sprout); } .kpis dt { font-size: 14px; color: #d4ece6; }

/* common */
.page-head { margin-bottom: 28px; max-width: 62ch; }
.page-head h2 { font-size: 30px; font-weight: 700; color: var(--jade); } .page-head p { color: var(--mute); margin-top: 6px; }
.h3 { font-size: 20px; font-weight: 700; color: var(--jade); margin-bottom: 4px; } .gap { margin: 40px 0 16px; }
.card { background: #fff; border: 1px solid var(--line); border-radius: 28px 8px 28px 8px; padding: 24px; min-width: 0; }
.two { display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); gap: 24px; align-items: start; }
.cards { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 16px; }
.btn-row { display: flex; gap: 12px; } .btn-row.end { justify-content: flex-end; margin-top: 12px; }
.btn { border: 2px solid transparent; border-radius: 999px; padding: 10px 20px; font-weight: 700; transition: background .15s; }
.btn-primary { background: var(--jade); color: #fff; } .btn-primary:hover { background: var(--jade-d); }
.btn-ghost { background: var(--mist); color: var(--mute); }
.btn-soft { background: #fff; border-color: var(--line); color: var(--jade); font-size: 13px; padding: 8px 14px; } .btn-soft:hover { background: var(--mist); }
.btn-block { width: 100%; } .big { padding: 15px; font-size: 16px; margin-top: 8px; }
.tag { background: var(--mist); color: var(--mute); border-radius: 999px; padding: 2px 12px; font-size: 12px; font-weight: 700; white-space: nowrap; }
.error { color: var(--warn); font-size: 14px; font-weight: 600; margin-top: 8px; }
.empty { text-align: center; font-size: 13px; color: var(--mute); padding: 18px 8px; border: 1px dashed var(--line); border-radius: 14px; margin-top: 12px; }
.empty.wide { grid-column: 1 / -1; padding: 48px 16px; font-size: 16px; }
.field { display: grid; gap: 6px; font-size: 14px; font-weight: 600; } .field span { color: var(--mute); }
.app input[type='text'], .app input[type='number'], .app select { width: 100%; background: var(--mist); border: 2px solid transparent; border-radius: 12px; padding: 11px 13px; font: inherit; font-weight: 600; color: var(--ink); }
.app input:focus, .app select:focus { outline: none; border-color: var(--jade); background: #fff; }
.grid2 { display: grid; grid-template-columns: minmax(0, 1fr) minmax(0, 1fr); gap: 16px; } .full { grid-column: 1 / -1; }
.row-b { display: grid; gap: 6px; font-size: 14px; margin-top: 12px; }
.bar { height: 8px; border-radius: 999px; background: rgba(255, 255, 255, .2); overflow: hidden; } .bar i { display: block; height: 100%; background: var(--teal); border-radius: inherit; transition: width .4s; }
.bar.sm { background: var(--mist); height: 6px; } .bar.sm i { background: var(--jade); }

/* kanban */
.kanban { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 16px; align-items: start; }
.col { background: #fff; border: 1px solid var(--line); border-top: 4px solid var(--c); border-radius: 22px 6px 22px 6px; padding: 16px; display: grid; gap: 12px; }
.col h3 { font-size: 15px; font-weight: 800; display: flex; align-items: center; gap: 8px; }
.leaf { width: 30px; height: 30px; background: var(--mist); border-radius: 0 60% 0 60%; display: grid; place-items: center; font-size: 15px; }
.col h3 em { margin-left: auto; font-style: normal; font-size: 12px; background: var(--mist); color: var(--mute); border-radius: 999px; padding: 1px 9px; }
.card-p { background: var(--mist); border: 1px solid var(--line); border-radius: 14px; padding: 14px; display: grid; gap: 8px; } .card-p h4 { font-size: 15px; }
.checks { display: grid; gap: 6px; font-size: 12.5px; background: #fff; border-radius: 10px; padding: 10px; }
.checks label { display: flex; gap: 8px; align-items: flex-start; cursor: pointer; }
.add { border: 2px dashed var(--line); background: none; color: var(--mute); border-radius: 14px; padding: 12px; font-weight: 700; } .add:hover { background: var(--mist); color: var(--jade); }

/* calculator */
.calc { display: grid; grid-template-columns: minmax(0, 1.25fr) minmax(0, 1fr); gap: 32px; align-items: start; }
.chips { display: flex; flex-wrap: wrap; gap: 10px; margin-bottom: 16px; }
.chip { display: flex; align-items: center; gap: 8px; background: #fff; border: 2px solid var(--line); border-radius: 999px; padding: 9px 18px; font-weight: 700; color: var(--mute); }
.chip:hover { border-color: var(--teal); } .chip.on { border-color: var(--jade); background: var(--jade); color: #fff; }
.form { display: grid; gap: 16px; }
.switch { display: flex; align-items: center; gap: 14px; background: var(--mist); border-radius: 14px; padding: 14px 16px; cursor: pointer; }
.switch input { position: absolute; opacity: 0; }
.switch i { width: 44px; height: 24px; border-radius: 999px; background: var(--line); position: relative; flex: none; transition: background .2s; }
.switch i::after { content: ''; position: absolute; top: 3px; left: 3px; width: 18px; height: 18px; border-radius: 50%; background: #fff; transition: transform .2s; }
.switch input:checked + i { background: var(--jade); } .switch input:checked + i::after { transform: translateX(20px); }
.switch input:focus-visible + i { outline: 3px solid var(--teal); outline-offset: 2px; }
.switch strong { display: block; } .switch small { color: var(--mute); }
.result-col { position: sticky; top: 96px; }
.placeholder { text-align: center; padding: 48px 28px; border-style: dashed; } .ph-ico { font-size: 44px; margin-bottom: 8px; }
.report { background: #fff; border: 1px solid var(--line); border-radius: 28px 8px 28px 8px; overflow: hidden; }
.rp-top { background: var(--jade); color: #fff; padding: 24px; } .rp-top small { color: var(--teal); font-weight: 700; } .rp-top h3 { font-size: 22px; margin: 2px 0 10px; }
.big-num { font-size: 44px; font-weight: 800; line-height: 1; } .big-num span { font-size: 16px; font-weight: 600; color: #d4ece6; }
.score { margin-top: 16px; display: grid; gap: 6px; font-size: 14px; font-weight: 600; }
.rp-sec { padding: 20px 24px; border-bottom: 1px solid var(--line); display: grid; gap: 10px; font-size: 14px; } .rp-sec h4 { font-size: 14px; color: var(--mute); }
.cut { text-decoration: line-through; opacity: .6; }
.rp-foot { background: var(--mist); padding: 20px 24px; display: flex; justify-content: space-between; gap: 12px; }
.rp-foot strong { display: block; font-size: 22px; color: var(--jade); } .rp-foot small { color: var(--mute); } .right { text-align: right; }

/* material */
.kit { border-top: 1px solid var(--line); padding: 16px 0; display: grid; gap: 10px; }
.kit-stats { display: flex; gap: 16px; flex-wrap: wrap; font-size: 13px; color: var(--mute); } .kit-stats b { color: var(--ink); }
.kit-act { display: flex; gap: 8px; align-items: center; flex-wrap: wrap; } .kit-act input { width: 72px; flex: none; }
.addrow { display: flex; gap: 8px; margin-top: 16px; flex-wrap: wrap; } .addrow input[type='text'] { flex: 2 1 160px; width: auto; } .addrow input[type='number'], .addrow select { flex: 1 1 90px; width: auto; }
.diversi { margin: 20px 0 4px; display: flex; align-items: baseline; gap: 12px; } .diversi strong { font-size: 40px; font-weight: 800; color: var(--jade); } .diversi span { color: var(--mute); font-size: 14px; }
.log { list-style: none; margin-top: 16px; display: grid; gap: 6px; font-size: 13px; }
.log li { display: grid; grid-template-columns: minmax(0, 1fr) auto auto auto; gap: 10px; align-items: center; padding: 8px 12px; background: var(--mist); border-radius: 10px; }
.log .trip { overflow-wrap: anywhere; } .log-foot { margin-top: 12px; flex-wrap: wrap; }
.x { background: none; border: 0; color: var(--mute); width: 28px; height: 28px; border-radius: 50%; } .x:hover { background: #fde8e6; color: var(--warn); }
.btn-danger { background: #fff; border-color: #efc5c1; color: var(--warn); font-size: 13px; padding: 8px 14px; } .btn-danger:hover { background: #fde8e6; }

/* passport */
.addpass { display: flex; gap: 12px; flex-wrap: wrap; margin-bottom: 24px; } .addpass input { flex: 1 1 200px; width: auto; }
.passport { background: #fff; border: 2px solid var(--jade); border-radius: 28px 8px 28px 8px; padding: 20px; display: grid; gap: 14px; background-image: linear-gradient(var(--mist), var(--mist)); background-size: 100% 64px; background-repeat: no-repeat; }
.passport header { display: flex; justify-content: space-between; align-items: flex-start; gap: 8px; } .passport h4 { font-size: 19px; color: var(--jade); } .passport small { color: var(--mute); }
.lvl { background: var(--jade); color: #fff; }
.stamps { display: grid; grid-template-columns: repeat(5, minmax(0, 1fr)); gap: 4px; }
.stamp { background: none; border: 0; display: grid; justify-items: center; gap: 4px; padding: 0; font-size: 10px; min-width: 0; overflow-wrap: anywhere; font-weight: 700; color: var(--mute); }
.stamp i { font-style: normal; width: 46px; height: 46px; border-radius: 50%; border: 2px dashed #a9c2b8; display: grid; place-items: center; font-size: 18px; transition: transform .25s, background .2s; }
.stamp:hover i { border-color: var(--jade); }
.stamp.earned { color: var(--jade); } .stamp.earned i { background: var(--jade); color: var(--sprout); border: 2px solid var(--teal); outline: 2px solid var(--jade); outline-offset: 1px; transform: rotate(-8deg); font-weight: 800; }

/* portofolio */
.bukti dl { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 12px; margin-top: 18px; }
.bukti dl div { background: var(--mist); border-radius: 14px; padding: 12px; } .bukti dd { font-size: 26px; font-weight: 800; color: var(--jade); } .bukti dt { font-size: 12px; color: var(--mute); }
.pf { display: grid; gap: 16px; } .pf header { display: flex; justify-content: space-between; align-items: flex-start; gap: 8px; } .pf h4 { font-size: 18px; }
.pf dl { background: var(--mist); border-radius: 12px; padding: 14px; display: grid; gap: 8px; font-size: 14px; } .pf dl div { display: flex; justify-content: space-between; } .pf dt { color: var(--mute); } .pf dd { font-weight: 700; }
.ok { color: var(--jade); } .off { color: var(--mute); }
.pf footer { display: flex; justify-content: space-between; border-top: 1px solid var(--line); padding-top: 14px; font-weight: 700; } .pf footer span { color: var(--jade); }

/* modal + footer */
.overlay { position: fixed; inset: 0; background: rgba(7, 50, 41, .5); backdrop-filter: blur(3px); z-index: 100; display: grid; place-items: center; padding: 16px; }
.modal { background: #fff; border-radius: 28px 8px 28px 8px; padding: 28px; width: 100%; max-width: 460px; display: grid; gap: 14px; } .modal h3 { font-size: 22px; color: var(--jade); }
.footer { background: var(--jade-d); color: var(--teal); font-size: 13px; padding: 22px 0; }

@media (max-width: 1000px) {
  .kanban { grid-template-columns: repeat(2, minmax(0, 1fr)); } .calc, .hero-in, .two { grid-template-columns: minmax(0, 1fr); }
  .result-col { position: static; } .cards { grid-template-columns: repeat(2, minmax(0, 1fr)); }
}
@media (max-width: 640px) {
  .header-in { height: auto; flex-direction: column; align-items: stretch; padding-top: 12px; padding-bottom: 8px; } .nav { overflow-x: auto; }
  .kanban, .cards, .grid2, .bukti dl { grid-template-columns: minmax(0, 1fr); } .wrap { padding: 0 16px; }
  .page { padding-top: 28px; } .page-head h2 { font-size: 26px; } .card { padding: 18px; } .hero-in { padding-top: 32px; padding-bottom: 32px; gap: 24px; }
  .app input[type='text'], .app input[type='number'], .app select { font-size: 16px; }
  .btn { min-height: 44px; } .chip { padding: 9px 14px; } .kit-act input { width: 64px; } .kit-act .btn { flex: 1 1 auto; }
  .addrow > * { flex: 1 1 100% !important; } .addpass input { flex-basis: 100%; } .big-num { font-size: 38px; } .diversi strong { font-size: 34px; }
}
@media (prefers-reduced-motion: reduce) { .app * { transition: none !important; } }
</style>