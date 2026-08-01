<template>
  <div style="padding: 20px; max-width: 1000px; margin: 0 auto; font-family: sans-serif;">
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #eee; padding-bottom: 10px;">
      <h2>Dashboard Dokumen (Role: {{ userRole }})</h2>
      <button @click="logout" style="padding: 6px 12px; background: #dc3545; color: white; border: none; border-radius: 4px; cursor: pointer;">Logout</button>
    </div>

    <!-- Form Input (Khusus Pemohon) -->
    <div v-if="userRole === 'pemohon'" style="margin-bottom: 20px; padding: 15px; background: #eef6ff; border-radius: 6px;">
      <h3 style="margin-top:0;">{{ isEditing ? `Edit Permohonan #${editId}` : 'Buat Permohonan Baru' }}</h3>
      <form @submit.prevent>
        <input v-model="form.title" type="text" placeholder="Judul Permohonan" required style="width: 100%; padding: 8px; margin-bottom: 8px; box-sizing: border-box;" />
        <textarea v-model="form.description" placeholder="Deskripsi" rows="2" style="width: 100%; padding: 8px; margin-bottom: 8px; box-sizing: border-box;"></textarea>
        <input ref="fileInputRef" type="file" @change="e => selectedFile = e.target.files[0]" accept=".pdf,.doc,.docx,.jpg,.png" style="margin-bottom: 10px; display: block;" />
        <div style="display: flex; gap: 8px;">
          <button type="button" @click="submitForm('draft')" style="padding: 8px 12px; background: #6c757d; color: white; border: none; border-radius: 4px;">{{ isEditing ? 'Update Draft' : 'Simpan Draft' }}</button>
          <button type="button" @click="submitForm('submitted')" style="padding: 8px 12px; background: #0d6efd; color: white; border: none; border-radius: 4px;">{{ isEditing ? 'Update & Kirim' : 'Kirim' }}</button>
          <button v-if="isEditing" type="button" @click="resetForm" style="padding: 8px 12px; background: #dc3545; color: white; border: none; border-radius: 4px;">Batal</button>
        </div>
      </form>
    </div>

    <!-- Tabel Data -->
    <table style="width: 100%; border-collapse: collapse; border: 1px solid #ddd;">
      <thead>
        <tr style="background: #0d6efd; color: white; text-align: left;">
          <th style="padding: 8px;">ID</th><th style="padding: 8px;">Judul</th><th style="padding: 8px;">File</th><th style="padding: 8px;">Status</th><th style="padding: 8px;">Aksi</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="item in displayedSubmissions" :key="item.id" style="border-bottom: 1px solid #ddd;">
          <td style="padding: 8px;">#{{ item.id }}</td>
          <td style="padding: 8px;">{{ item.title }}</td>
          <td style="padding: 8px;">
            <a v-if="item.file_path" :href="`http://localhost:8000/storage/${item.file_path}`" target="_blank" style="color: #0d6efd; font-weight: bold;">📄 Lihat</a>
            <span v-else style="color: #888;">-</span>
          </td>
          <td style="padding: 8px;"><span style="padding: 3px 6px; background: #198754; color: white; border-radius: 3px; font-size: 11px;">{{ item.status }}</span></td>
          <td style="padding: 8px;">
            <div style="display: flex; gap: 4px; flex-wrap: wrap;">
              <button v-if="userRole === 'pemohon'" @click="startEdit(item)" style="background: #ffc107; border: none; padding: 4px 8px; border-radius: 3px;">Edit</button>
              <button @click="deleteItem(item.id)" style="background: #dc3545; color: white; border: none; padding: 4px 8px; border-radius: 3px;">Hapus</button>
              <template v-if="userRole === 'penilai' && item.status === 'submitted'">
                <button @click="reviewDoc(item.id, 'approved')" style="background: #198754; color: white; border: none; padding: 4px 6px; border-radius: 3px;">Approve</button>
                <button @click="reviewDoc(item.id, 'revision')" style="background: #fd7e14; color: white; border: none; padding: 4px 6px; border-radius: 3px;">Revisi</button>
                <button @click="reviewDoc(item.id, 'rejected')" style="background: #6c757d; color: white; border: none; padding: 4px 6px; border-radius: 3px;">Reject</button>
              </template>
              <button v-if="item.histories?.length" @click="alert(item.histories.map(h => `- ${h.status}: ${h.notes||'-'}`).join('\n'))" style="background: #0dcaf0; border: none; padding: 4px 6px; border-radius: 3px;">Riwayat</button>
            </div>
          </td>
        </tr>
        <tr v-if="!displayedSubmissions.length"><td colspan="5" style="text-align: center; padding: 15px; color: #666;">Belum ada data.</td></tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import api from '../api';

const submissions = ref([]), userRole = ref('pemohon'), selectedFile = ref(null), fileInputRef = ref(null);
const form = ref({ title: '', description: '' }), isEditing = ref(false), editId = ref(null), router = useRouter();

// Memastikan fleksibel membaca format Array maupun Pagination Objek Laravel
const displayedSubmissions = computed(() => {
  if (Array.isArray(submissions.value)) {
    return submissions.value;
  }
  return submissions.value?.data || [];
});

const logout = () => { localStorage.clear(); router.push('/login'); };
const resetForm = () => { form.value = { title: '', description: '' }; selectedFile.value = null; if (fileInputRef.value) fileInputRef.value.value = ''; isEditing.value = false; editId.value = null; };

// PERBAIKAN UTAMA DI SINI: Menyimpan objek paginator lengkap dari response Laravel
const fetchData = async () => {
  try {
    const res = await api.get(`/submissions?t=${Date.now()}`);
    submissions.value = res.data; // Simpan seluruh objek JSON dari backend
  } catch (e) { 
    if (e.response?.status === 401) logout(); 
  }
};

const startEdit = (item) => {
  isEditing.value = true; editId.value = item.id;
  form.value = { title: item.title, description: item.description || '' };
  window.scrollTo({ top: 0, behavior: 'smooth' });
};

const deleteItem = async (id) => {
  if (!confirm(`Hapus permohonan #${id}?`)) return;
  try {
    await api.delete(`/submissions/${id}`);
    fetchData();
  } catch (e) { alert(e.response?.data?.message || 'Gagal menghapus'); }
};

const submitForm = async (status) => {
  if (!form.value.title) return alert('Judul wajib diisi!');
  try {
    const fd = new FormData();
    fd.append('title', form.value.title);
    fd.append('description', form.value.description || '');
    fd.append('status', status);
    
    if (selectedFile.value) {
      fd.append('file', selectedFile.value);
    }

    const config = {
      headers: {
        'Content-Type': 'multipart/form-data',
      },
    };

    if (isEditing.value) {
      fd.append('_method', 'PUT');
      await api.post(`/submissions/${editId.value}`, fd, config);
    } else {
      await api.post('/submissions', fd, config);
    }

    alert(`Permohonan berhasil disimpan!`);
    resetForm();
    fetchData();
  } catch (e) {
    console.error('Submit Error:', e.response?.data);
    const errors = e.response?.data?.errors;
    if (errors) {
      alert(`Gagal Validasi Backend:\n${Object.values(errors).flat().join('\n')}`);
    } else {
      alert(e.response?.data?.message || 'Gagal menyimpan data.');
    }
  }
};

const reviewDoc = async (id, status) => {
  const notes = prompt(`Catatan Penilaian (${status}):`, 'Telah di-review');
  if (notes !== null) { await api.post(`/submissions/${id}/review`, { status, notes }); fetchData(); }
};

onMounted(() => {
  userRole.value = JSON.parse(localStorage.getItem('user') || '{}').roles?.[0] || 'pemohon';
  fetchData();
});
</script>