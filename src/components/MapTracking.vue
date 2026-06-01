<script setup lang="ts">
import { onMounted, ref, onBeforeUnmount } from 'vue';
import * as signalR from '@microsoft/signalr';
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

// Harita ve SignalR nesneleri için TypeScript tipleri
const map = ref<L.Map | null>(null);
const courierMarker = ref<L.Marker | null>(null);
const connection = ref<signalR.HubConnection | null>(null);
const isSimulating = ref<boolean>(false);
let simulationInterval: ReturnType<typeof setInterval> | null = null;

// İstanbul Kadıköy - Üsküdar hattında örnek kurye rotası
const mockRoute: [number, number][] = [
  [41.0026, 29.0270], [41.0035, 29.0260], [41.0050, 29.0255],
  [41.0070, 29.0250], [41.0095, 29.0242], [41.0115, 29.0230],
  [41.0135, 29.0220], [41.0155, 29.0210], [41.0180, 29.0205],
  [41.0200, 29.0190], [41.0225, 29.0160], [41.0240, 29.0153]
];
let currentRouteIndex = 0;

// TypeScript uyumlu Motor İkonu tanımı
const courierIcon = L.divIcon({
  html: `
    <div style="
      background-color: #ef4444; 
      border: 3px solid white; 
      border-radius: 50%; 
      width: 40px; 
      height: 40px; 
      display: flex; 
      justify-content: center; 
      align-items: center; 
      box-shadow: 0 4px 6px rgba(0,0,0,0.3);
    ">
      <!-- Şık bir kurye/motorcu emojisi ile kırılmayan hızlı çözüm -->
      <span style="font-size: 20px; line-height: 1;">🛵</span>
    </div>
  `,
  className: 'custom-courier-marker', // Leaflet'in kendi çirkin arka planını temizlemek için
  iconSize: [40, 40],
  iconAnchor: [20, 20], // İkonun tam merkezden haritaya oturması için
  popupAnchor: [0, -25]
});

onMounted(() => {
  // 1. Haritayı Başlat
  map.value = L.map('map').setView([41.0082, 29.0250], 14);

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap'
  }).addTo(map.value);

  // Rota çizgisini haritaya ekle
  L.polyline(mockRoute, { color: '#3b82f6', weight: 4, opacity: 0.5, dashArray: '5, 10' }).addTo(map.value);

  // 2. SignalR Bağlantısı (.NET Hub adresiniz)
  connection.value = new signalR.HubConnectionBuilder()
    .withUrl("http://localhost:5023/courierHub") // .NET portunuza göre güncelleyin
    .withAutomaticReconnect()
    .build();

  // 3. Backend'den gelen konum verisini dinle
  connection.value.on("ReceiveLocation", (courierId: string, lat: number, lng: number) => {
    const newLatLng = new L.LatLng(lat, lng);

    if (!map.value) return;

    if (!courierMarker.value) {
      // Kurye yoksa oluştur
      courierMarker.value = L.marker(newLatLng, { icon: courierIcon }).addTo(map.value)
        .bindPopup(`<b>Kurye Aktif</b><br>ID: ${courierId}`)
        .openPopup();
    } else {
      // Kurye varsa konumunu güncelle
      courierMarker.value.setLatLng(newLatLng);
    }

    // Haritayı kuryeye odakla
    map.value.panTo(newLatLng, { animate: true, duration: 1 });
  });

  // Bağlantıyı başlat
  connection.value.start()
    .then(() => console.log("SignalR Bağlantısı Başarılı! 🚀"))
    .catch(err => console.error("SignalR Hatası: ", err));
});

// 4. Simülasyon Yönetimi
const toggleSimulation = () => {
  if (isSimulating.value) {
    if (simulationInterval) clearInterval(simulationInterval);
    isSimulating.value = false;
  } else {
    isSimulating.value = true;
    simulationInterval = setInterval(() => {
      if (currentRouteIndex >= mockRoute.length) {
        currentRouteIndex = 0;
      }

      const [lat, lng] = mockRoute[currentRouteIndex];
      
      if (connection.value && connection.value.state === signalR.HubConnectionState.Connected) {
        connection.value.invoke("SendLocation", "Kurye-34", lat, lng)
          .catch(err => console.error("Konum gönderme hatası:", err));
      }

      currentRouteIndex++;
    }, 2000);
  }
};

onBeforeUnmount(() => {
  if (simulationInterval) clearInterval(simulationInterval);
});
</script>

<template>
  <div class="tracking-container">
    <div class="header">
      <h1>Canlı Kurye Takip Sistemi</h1>
      <button 
        @click="toggleSimulation" 
        :class="['sim-btn', isSimulating ? 'btn-stop' : 'btn-start']"
      >
        {{ isSimulating ? 'Simülasyonu Durdur 🛑' : 'Simülasyonu Başlat 🚀' }}
      </button>
    </div>
    
    <div id="map"></div>
  </div>
</template>

<style scoped>
.tracking-container {
  max-width: 1000px;
  margin: 40px auto;
  padding: 20px;
  font-family: sans-serif;
}
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}
.sim-btn {
  padding: 12px 24px;
  font-size: 16px;
  font-weight: bold;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
}
.btn-start { background-color: #10b981; color: white; }
.btn-stop { background-color: #ef4444; color: white; }
#map {
  width: 100%;
  height: 550px;
  border-radius: 12px;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  border: 2px solid #e2e8f0;
}
:deep(.custom-courier-marker) {
  background: transparent !important;
  border: none !important;
}
</style>
