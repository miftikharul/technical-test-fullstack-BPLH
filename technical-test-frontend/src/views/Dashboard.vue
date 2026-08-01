<template>
  <div style="padding: 30px; max-width: 1200px; margin: 0 auto; font-family: sans-serif;">
    <!-- Header -->
    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; border-bottom: 2px solid #eee; padding-bottom: 15px;">
      <h2>Dashboard Sistem Dokumen (Role: {{ userRole }})</h2>
      <button @click="logout" style="padding: 8px 16px; background: #dc3545; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold;">Logout</button>
    </div>

  
    <div style="display: flex; gap: 15px; margin-bottom: 30px;">
      <div v-for="(val, key) in stats" :key="key" style="flex: 1; padding: 15px; background: #f8f9fa; border-radius: 8px; text-align: center; border: 1px solid #e0e0e0;">
        <h4 style="text-transform: uppercase; margin: 0; color: #6c757d; font-size: 11px;">{{ key }}</h4>
        <p style="font-size: 24px; font-weight: bold; margin: 8px 0 0 0;">{{ val }}</p>
      </div>
    </div>

    <div v-if="userRole === 'pemohon'" style="margin-bottom: 30px; padding: 20px; background: #eef6ff; border: 1px solid #b6d4fe; border-radius: 8px;">
      <h3 style="margin-top: 0; color: #052c65;">Buat Permohonan Dokumen Baru</h3>
      <form @submit.prevent="submitForm">
        <div style="margin-bottom: 10px;">
          <label style="font-weight: bold; display: block;">Judul Permohonan:</label>
          <input v-model="form.title" type="text" required style="width: 100%; padding: 8px; box-sizing: border-box;" placeholder="Judul permohonan...">
        </div>
        <div style="margin-bottom: 10px;">
          <label style="font-weight: bold; display: block;">Deskripsi:</label>
          <textarea v-model="form.description" rows="2" style="width: 100%; padding: 8px; box-sizing: border-box;"></textarea>
        </div>
        <div style="margin-bottom: 15px;">
          <label style="font-weight: bold; display: block;">Unggah Dokumen (PDF/Docx):</label>
          <input type="file" @change="handleFileUpload" style="margin-top: 5px;">
        </div>
        <button type="submit" style="padding: 10px 20px; background: #0d6efd; color: white; border: none; border-radius: 4px; cursor: pointer; font-weight: bold;">Kirim Permohonan</button>
      </form>
    </div>

    <h3>Daftar Permohonan Dokumen</h3>
    <table style="width: 100%; border-collapse: collapse; background: white; border: 1px solid #ddd;">
      <thead>
        <tr style="background: #0d6efd; color: white; text-align: left;">
          <th style="padding: 10px; border: 1px solid #ddd;">ID</th>
          <th style="padding: 10px; border: 1px solid #ddd;">Judul</th>
          <th style="padding: 10px; border: 1px solid #ddd;">Pemohon</th>
          <th style="padding: 10px; border: 1px solid #ddd;">Status</th>
          <th style="padding: 10px; border: 1px solid #ddd;">Aksi</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="item in submissions" :key="item.id">
          <td style="padding: 10px; border: 1px solid #ddd;">#{{ item.id }}</td>
          <td style="padding: 10px; border: 1px solid #ddd;">{{ item.title }}</td>
          <td style="padding: 10px; border: 1px solid #ddd;">{{ item.user?.name || 'User' }}</td>
          <td style="padding: 10px; border: 1px solid #ddd;">
            <span style="padding: 4px 8px; border-radius: 4px; background: #198754; color: white; font-size: 11px; text-transform: uppercase; font-weight: bold;">{{ item.status }}</span>
          </td>
          <td style="padding: 10px; border: 1px solid #ddd;">
            <!-- Penilai Actions -->
            <div v-if="userRole === 'penilai' && item.status === 'submitted'" style="display: flex; gap: 5px;">
              <button @click="reviewDoc(item.id, 'approved')" style="background: #198754; color: white; border: none; padding: 5px 8px; cursor: pointer; border-radius: 3px;">Approve</button>
              <button @click="reviewDoc(item.id, 'revision')" style="background: #ffc107; color: black; border: none; padding: 5px 8px; cursor: pointer; border-radius: 3px;">Revisi</button>
              <button @click="reviewDoc(item.id, 'rejected')" style="background: #dc3545; color: white; border: none; padding: 5px 8px; cursor: pointer; border-radius: 3px;">Reject</button>
            </div>
            <button v-if="item.histories && item.histories.length > 0" @click="showHistory(item)" style="background: #6c757d; color: white; border: none; padding: 5px 8px; cursor: pointer; border-radius: 3px; margin-left: 5px;">Riwayat</button>
          </td>
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
const selectedFile = ref(null);
const router = useRouter();
const form = ref({ title: '', description: '' });

const logout = () => { localStorage.clear(); router.push('/login'); };

const handleFileUpload = (e) => { selectedFile.value = e.target.files[0]; };

const fetchData = async () => {
  try {
    const resStats = await api.get('/dashboard');
    stats.value = resStats.data;
    const resSubmissions = await api.get('/submissions');
    submissions.value = resSubmissions.data.data || [];
  } catch (err) { if (err.response?.status === 401) logout(); }
};

const submitForm = async () => {
  const formData = new FormData();
  formData.append('title', form.value.title);
  formData.append('description', form.value.description);
  formData.append('action', 'submitted');
  if (selectedFile.value) formData.append('file', selectedFile.value);

  await api.post('/submissions', formData, { headers: { 'Content-Type': 'multipart/form-data' } });
  form.value = { title: '', description: '' };
  selectedFile.value = null;
  fetchData();
};

const reviewDoc = async (id, status) => {
  const notes = prompt(`Masukkan Catatan Penilaian (${status}):`, 'Telah di-review');
  if (notes === null) return;
  await api.post(`/submissions/${id}/review`, { status, notes });
  fetchData();
};

const showHistory = (item) => {
  const historyText = item.histories.map(h => `- Status: ${h.status} | Catatan: ${h.notes || '-'}`).join('\n');
  alert(`Riwayat Penilaian #${item.id}:\n\n${historyText}`);
};

onMounted(() => {
  const user = JSON.parse(localStorage.getItem('user') || '{}');
  userRole.value = user.roles?.[0] || 'pemohon';
  fetchData();
});
</script>