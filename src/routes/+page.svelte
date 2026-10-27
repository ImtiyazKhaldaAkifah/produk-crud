<script>
  import { onMount } from 'svelte';
  import { supabase } from '../lib/supabaseClient.js';

  // ===== STATE =====
  let produk = $state([]);
  let namaProduk = $state('');
  let harga = $state('');
  let stok = $state('');
  let editId = $state(null);
  let memuat = $state(true);
  let menyimpan = $state(false);
  let pesan = $state({ tipe: '', teks: '' });
  let galat = $state({ nama: '', harga: '', stok: '' });

  let timerPesan;

  // ===== FUNGSI BANTU =====
  function formatRupiah(angka) {
    return new Intl.NumberFormat('id-ID', {
      style: 'currency',
      currency: 'IDR',
      minimumFractionDigits: 0,
      maximumFractionDigits: 2
    }).format(angka);
  }

  function tampilkanPesan(tipe, teks) {
    pesan = { tipe, teks };
    clearTimeout(timerPesan);
    timerPesan = setTimeout(() => (pesan = { tipe: '', teks: '' }), 4000);
  }

  function kosong(nilai) {
    return nilai === null || nilai === undefined || String(nilai).trim() === '';
  }

  // ===== VALIDASI =====
  function validasi() {
    const hasil = { nama: '', harga: '', stok: '' };
    const nama = namaProduk.trim();

    if (nama === '') hasil.nama = 'Nama produk wajib diisi.';
    else if (nama.length > 100) hasil.nama = 'Nama produk maksimal 100 karakter.';

    if (kosong(harga)) hasil.harga = 'Harga wajib diisi.';
    else if (!Number.isFinite(Number(harga))) hasil.harga = 'Harga harus berupa angka.';
    else if (Number(harga) < 0) hasil.harga = 'Harga tidak boleh negatif.';

    if (kosong(stok)) hasil.stok = 'Stok wajib diisi.';
    else if (!Number.isInteger(Number(stok))) hasil.stok = 'Stok harus bilangan bulat.';
    else if (Number(stok) < 0) hasil.stok = 'Stok tidak boleh negatif.';
    else if (Number(stok) > 2147483647) hasil.stok = 'Stok terlalu besar.';

    galat = hasil;
    return !hasil.nama && !hasil.harga && !hasil.stok;
  }

  // ===== READ =====
  async function ambilProduk() {
    memuat = true;
    const { data, error } = await supabase
      .from('produk')
      .select('*')
      .order('id', { ascending: true });

    if (error) {
      console.error('Gagal memuat produk:', error);
      tampilkanPesan('error', 'Gagal memuat data: ' + error.message);
    } else {
      produk = data;
    }
    memuat = false;
  }

  onMount(ambilProduk);

  // ===== CREATE & UPDATE =====
  async function simpanProduk(event) {
    event.preventDefault();
    if (!validasi()) return;

    const dataProduk = {
      nama_produk: namaProduk.trim(),
      harga: Number(harga),
      stok: Number(stok)
    };

    menyimpan = true;
    let error;

    if (editId === null) {
      // CREATE
      ({ error } = await supabase.from('produk').insert(dataProduk));
      if (!error) tampilkanPesan('sukses', 'Produk berhasil ditambahkan.');
    } else {
      // UPDATE
      const { data, error: errUpdate } = await supabase
        .from('produk')
        .update(dataProduk)
        .eq('id', editId)
        .select();
      error = errUpdate;
      if (!error && data.length === 0) {
        error = { message: 'Tidak ada data yang diubah. Periksa policy UPDATE di Supabase.' };
      }
      if (!error) tampilkanPesan('sukses', 'Produk berhasil diperbarui.');
    }

    menyimpan = false;

    if (error) {
      console.error('Gagal menyimpan produk:', error);
      tampilkanPesan('error', 'Gagal menyimpan: ' + error.message);
      return;
    }

    resetForm();
    await ambilProduk();
  }

  // ===== EDIT =====
  function editProduk(p) {
    editId = p.id;
    namaProduk = p.nama_produk;
    harga = p.harga;
    stok = p.stok;
    galat = { nama: '', harga: '', stok: '' };
    window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function resetForm() {
    editId = null;
    namaProduk = '';
    harga = '';
    stok = '';
    galat = { nama: '', harga: '', stok: '' };
  }

  // ===== DELETE =====
  async function hapusProduk(p) {
    if (!confirm(`Yakin ingin menghapus "${p.nama_produk}"?`)) return;

    const { data, error } = await supabase
      .from('produk')
      .delete()
      .eq('id', p.id)
      .select();

    if (error) {
      console.error('Gagal menghapus produk:', error);
      tampilkanPesan('error', 'Gagal menghapus: ' + error.message);
      return;
    }
    if (data.length === 0) {
      tampilkanPesan('error', 'Tidak ada data yang dihapus. Periksa policy DELETE di Supabase.');
      return;
    }

    if (editId === p.id) resetForm();
    tampilkanPesan('sukses', 'Produk berhasil dihapus.');
    await ambilProduk();
  }
</script>

<main>
  <h1>Input Produk</h1>

  {#if pesan.teks}
    <div class="pesan {pesan.tipe}" role="status">{pesan.teks}</div>
  {/if}

  <!-- ===== FORM ===== -->
  <form class="kartu" onsubmit={simpanProduk} novalidate>
    <div class="grid-form">
      <div class="field">
        <label for="nama">Nama Produk</label>
        <input
          id="nama"
          type="text"
          placeholder="Contoh: Keyboard Mechanical"
          bind:value={namaProduk}
        />
        {#if galat.nama}<small class="galat">{galat.nama}</small>{/if}
      </div>

      <div class="field">
        <label for="harga">Harga</label>
        <input
          id="harga"
          type="number"
          min="0"
          step="any"
          placeholder="Contoh: 150000"
          bind:value={harga}
        />
        {#if galat.harga}<small class="galat">{galat.harga}</small>{/if}
      </div>

      <div class="field">
        <label for="stok">Stok</label>
        <input
          id="stok"
          type="number"
          min="0"
          step="1"
          placeholder="Contoh: 10"
          bind:value={stok}
        />
        {#if galat.stok}<small class="galat">{galat.stok}</small>{/if}
      </div>
    </div>

    <div class="aksi-form">
      <button type="submit" class="btn btn-utama" disabled={menyimpan}>
        {#if menyimpan}
          Menyimpan...
        {:else if editId === null}
          Simpan Produk
        {:else}
          Update Produk
        {/if}
      </button>
      {#if editId !== null}
        <button type="button" class="btn btn-batal" onclick={resetForm}>Batal</button>
      {/if}
    </div>
  </form>

  <!-- ===== TABEL ===== -->
  <h2>Daftar Produk (Database)</h2>

  <div class="kartu tabel-wrap">
    <table>
      <thead>
        <tr>
          <th>No</th>
          <th>Nama Produk</th>
          <th>Harga</th>
          <th>Stok</th>
          <th>Aksi</th>
        </tr>
      </thead>
      <tbody>
        {#if memuat}
          <tr><td colspan="5" class="tengah">Memuat data...</td></tr>
        {:else if produk.length === 0}
          <tr><td colspan="5" class="tengah">Belum ada data produk</td></tr>
        {:else}
          {#each produk as p, i (p.id)}
            <tr>
              <td>{i + 1}</td>
              <td>{p.nama_produk}</td>
              <td>{formatRupiah(p.harga)}</td>
              <td>{p.stok}</td>
              <td class="kolom-aksi">
                <button class="btn btn-edit" onclick={() => editProduk(p)}>Edit</button>
                <button class="btn btn-hapus" onclick={() => hapusProduk(p)}>Hapus</button>
              </td>
            </tr>
          {/each}
        {/if}
      </tbody>
    </table>
  </div>
</main>

<style>
  :global(body) {
    margin: 0;
    background: #0f172a;
    color: #e2e8f0;
    font-family: system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  }

  main {
    max-width: 900px;
    margin: 0 auto;
    padding: 24px 16px 48px;
  }

  h1 { margin: 0 0 16px; font-size: 1.8rem; }
  h2 { margin: 32px 0 12px; font-size: 1.3rem; }

  .kartu {
    background: #1e293b;
    border: 1px solid #334155;
    border-radius: 10px;
    padding: 20px;
  }

  .grid-form {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
  }
  .field { display: flex; flex-direction: column; gap: 6px; }
  label { font-size: 0.9rem; color: #cbd5e1; }

  input {
    padding: 10px 12px;
    border: 1px solid #475569;
    border-radius: 6px;
    background: #0f172a;
    color: #f8fafc;
    font-size: 1rem;
  }
  input:focus {
    outline: 2px solid #3b82f6;
    border-color: transparent;
  }

  .galat { color: #fca5a5; }
  .aksi-form { margin-top: 16px; display: flex; gap: 10px; }

  .btn {
    padding: 9px 16px;
    border: none;
    border-radius: 6px;
    font-size: 0.95rem;
    cursor: pointer;
    color: #fff;
  }
  .btn:disabled { opacity: 0.6; cursor: not-allowed; }
  .btn-utama { background: #2563eb; }
  .btn-utama:hover:not(:disabled) { background: #1d4ed8; }
  .btn-batal { background: #475569; }
  .btn-edit { background: #d97706; }
  .btn-hapus { background: #dc2626; }
  .btn-edit:hover { background: #b45309; }
  .btn-hapus:hover { background: #b91c1c; }

  .pesan {
    padding: 10px 14px;
    border-radius: 6px;
    margin-bottom: 16px;
  }
  .pesan.sukses { background: #14532d; color: #bbf7d0; border: 1px solid #166534; }
  .pesan.error  { background: #7f1d1d; color: #fecaca; border: 1px solid #991b1b; }

  .tabel-wrap { padding: 0; overflow-x: auto; }
  table { width: 100%; border-collapse: collapse; min-width: 560px; }
  th, td { padding: 12px 14px; text-align: left; border-bottom: 1px solid #334155; }
  th { background: #273449; font-size: 0.9rem; }
  tbody tr:last-child td { border-bottom: none; }
  tbody tr:hover { background: #243247; }
  .tengah { text-align: center; color: #94a3b8; padding: 24px; }
  .kolom-aksi { white-space: nowrap; display: flex; gap: 8px; }

  @media (max-width: 640px) {
    .grid-form { grid-template-columns: 1fr; }
    h1 { font-size: 1.5rem; }
  }
</style>