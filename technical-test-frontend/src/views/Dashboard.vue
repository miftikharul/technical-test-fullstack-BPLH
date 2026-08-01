<template>
  <div style="padding: 30px; max-width: 1100px; margin: 0 auto; font-family: sans-serif;">
    <!-- Header -->
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #eee; padding-bottom: 15px;">
      <h2>Dashboard Sistem Dokumen (Role: {{ userRole }})</h2>
      <button @click="logout" style="padding: 8px 16px; background: #dc3545; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold;">
        Logout
      </button>
    </div>

    <div style="display: flex; gap: 15px; margin-bottom: 30px;">
      <div v-for="(val, key) in stats" :key="key" style="flex: 1; padding: 15px; background: #f8f9fa; border-radius: 8px; text-align: center; border: 1px solid #e0e0e0;">
        <h4 style="text-transform: uppercase; margin: 0; color: #6c757d; font-size: 11px; letter-spacing: 1px;">{{ key }}</h4>
        <p style="font-size: 26px; font-weight: bold; margin: 8px 0 0 0; color: #333;">{{ val }}</p>
      </div>
    </div>

    <div v-if="userRole === 'pemohon'" style="margin-bottom: 30px; padding: 20px; background: #eef6ff; border: 1px solid #b6d4fe; border-radius: 8px;">
      <h3 style="margin-top: 0; color: #052c65;">Buat Permohonan Dokumen Baru</h3>
      <form @submit.prevent="submitForm">
        <div style="margin-bottom: 15px;">
          <label style="display: block; font-weight: bold; margin-bottom: 5px;">Judul Permohonan:</label>
          <input v-model="form.title" type="text" required style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box;" placeholder="Contoh: Pengajuan Anggaran Server Q3">
        </div>
        <div style="margin-bottom: 15px;">
          <label style="display: block; font-weight: bold; margin-bottom: 5px;">Deskripsi:</label>
          <textarea v-model="form.description" rows="3" style="width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box;" placeholder="Detail permohonan..."></textarea>
        </div>
        <div style="display: flex; gap: 10px;">
          <button type="button" @click="saveDraft" style="padding: 10px 20px; background: #6c757d; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold;">
            Simpan Draft
          </button>
          <button type="submit" style="padding: 10px 20px; background: #0d6efd; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold;">
            Kirim (Submit)
          </button>
        </div>
      </form>
    </div>

    <h3 style="margin-bottom: 15px;">Daftar Permohonan Dokumen</h3>
    <table style="width: 100%; border-collapse: collapse; background: white; border: 1px solid #ddd;">
      <thead>
        <tr style="background: #0d6efd; color: white; text-align: left;">
          <th style="padding: 12px; border: 1px solid #ddd;">ID</th>
          <th style="padding: 12px; border: 1px solid #ddd;">Judul Permohonan</th>
          <th style="padding: 12px; border: 1px solid #ddd;">Status</th>
          <th style="padding: 12px; border: 1px solid #ddd;">Aksi</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="item in submissions" :key="item.id">
          <td style="padding: 10px; border: 1px solid #ddd;">#{{ item.id }}</td>
          <td style="padding: 10px; border: 1px solid #ddd;">{{ item.title }}</td>
          <td style="padding: 10px; border: 1px solid #ddd;">
            <span style="padding: 4px 8px; border-radius: 4px; background: #198754; color: white; font-size: 12px; text-transform: uppercase; font-weight: bold;">
              {{ item.status }}
            </span>
          </td>
          <td style="padding: 10px; border: 1px solid #ddd;">
            <div v-if="userRole === 'penilai' && item.status === 'submitted'" style="display: flex; gap: 5px;">
              <button @click="reviewDoc(item.id, 'approved')" style="padding: 5px 10px; background: #198754; color: white; border: none; border-radius: 3px; cursor: pointer; font-size: 12px;">Approve</button>
              <button @click="reviewDoc(item.id, 'revision')" style="padding: 5px 10px; background: #ffc107; color: black; border: none; border-radius: 3px; cursor: pointer; font-size: 12px;">Revisi</button>
              <button @click="reviewDoc(item.id, 'rejected')" style="padding: 5px 10px; background: #dc3545; color: white; border: none; border-radius: 3px; cursor: pointer; font-size: 12px;">Reject</button>
            </div>
            <span v-else style="color: #888; font-size: 13px;">-</span>
          </td>
        </tr>
        <tr v-if="submissions.length === 0">
          <td colspan="4" style="text-align: center; padding: 20px; color: #888;">Belum ada data permohonan. Silakan buat baru di atas!</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import api from '../api';

const stats = ref({});
const submissions = ref([]);
const userRole = ref('');
const router = useRouter();
const form = ref({ title: '', description: '' });

const logout = () => {
  localStorage.clear();
  router.push('/login');
};

const fetchData = async () => {
  try {
    const resStats = await api.get('/dashboard');
    stats.value = resStats.data;

    const resSubmissions = await api.get('/submissions');
    submissions.value = resSubmissions.data.data || resSubmissions.data;
  } catch (err) {
    if (err.response?.status === 401) logout();
  }
};

const saveDraft = async () => {
  if (!form.value.title) return alert('Judul Wajib Diisi!');
  await api.post('/submissions', { ...form.value, action: 'draft' });
  form.value = { title: '', description: '' };
  fetchData();
};

const submitForm = async () => {
  if (!form.value.title) return alert('Judul Wajib Diisi!');
  await api.post('/submissions', { ...form.value, action: 'submitted' });
  form.value = { title: '', description: '' };
  fetchData();
};

const reviewDoc = async (id, status) => {
  const notes = prompt(`Masukkan Catatan Penilaian untuk status (${status}):`, 'Telah disetujui/di-review');
  if (notes === null) return;
  
  await api.post(`/submissions/${id}/review`, { status, notes });
  fetchData();
};

onMounted(() => {
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  userRole.value = user.roles?.[0] || 'pemohon';
  fetchData();
});
</script>