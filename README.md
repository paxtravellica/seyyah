<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="utf-8" />
    <title>Seyyah - Tarihi Rotalar ve Künyeler</title>
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #f0f2f5; margin: 0; padding: 20px; }
        .header-box { max-width: 1300px; margin: 0 auto 20px auto; padding: 15px; background: #fff; border-radius: 10px; text-align: center; box-shadow: 0 2px 10px rgba(0,0,0,0.05); }
        h2 { color: #2c3e50; margin: 0; }
        
        .main-container { display: flex; max-width: 1300px; margin: 0 auto; height: 750px; background: #fff; border-radius: 15px; overflow: hidden; box-shadow: 0 10px 30px rgba(0,0,0,0.1); }
        
        /* Harita Alanı */
        #map { flex: 2.5; background: #e5e3df; border-right: 1px solid #eee; }
        
        /* Sağ Panel (Künye Alanı) */
        .sidebar { flex: 1; padding: 25px; display: flex; flex-direction: column; background: #ffffff; overflow-y: auto; }
        h3 { color: #34495e; border-bottom: 2px solid #3498db; padding-bottom: 10px; margin-top: 0; }
        
        /* Seçim Kutusu */
        select { width: 100%; padding: 12px; margin-bottom: 20px; border-radius: 8px; border: 1px solid #dcdde1; font-size: 16px; background-color: #f9f9f9; cursor: pointer; transition: 0.3s; }
        select:hover { border-color: #3498db; }

        /* Künye Sütunları/Kartları */
        .kunye-row { margin-bottom: 15px; padding: 12px; background: #f8f9fa; border-radius: 8px; border-left: 5px solid #3498db; }
        .kunye-label { display: block; font-size: 11px; text-transform: uppercase; color: #7f8c8d; font-weight: bold; letter-spacing: 1px; margin-bottom: 4px; }
        .kunye-value { font-size: 15px; color: #2c3e50; font-weight: 500; }

        /* Rota Listesi */
        .rota-section { margin-top: 10px; border-top: 1px solid #eee; padding-top: 15px; }
        .rota-title { font-weight: bold; font-size: 14px; margin-bottom: 10px; display: block; color: #2c3e50; }
        .rota-item { display: flex; justify-content: space-between; font-size: 13px; padding: 6px 0; border-bottom: 1px dashed #eee; color: #555; }
        .rota-date { color: #3498db; font-weight: bold; }
    </style>
</head>
<body>

    <div class="header-box">
        <h2>Dijital Seyyah Arşivi</h2>
    </div>
    
    <div class="main-container">
        <div id="map"></div>
        
        <div class="sidebar">
            <h3>Seyyah Seçimi</h3>
            <select id="seyyahSelect" onchange="haritayiGuncelle()">
                <option value="roger">Roger Bodenham</option>
                <option value="locke">John Locke</option>
                <option value="foxe">John Foxe</option>
            </select>

            <div id="kunyeGosterim">
                </div>
        </div>
    </div>

    <script>
        var map = L.map('map').setView([35, 20], 4);
        L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/NatGeo_World_Map/MapServer/tile/{z}/{y}/{x}').addTo(map);

        var seyyahKatalogu = {
            "roger": {
                isim: "Roger Bodenham", renk: "#27ae60", meslek: "Deniz Kaptanı", amac: "Ticaret",
                tarihAraligi: "1550 - 1551",
                rota: [
                    { sehir: "Gravesend", tarih: "1550", koor: [51.44, 0.37] },
                    { sehir: "Cebelitarık", tarih: "1551", koor: [36.14, -5.35] },
                    { sehir: "Mayorka", tarih: "1551", koor: [39.69, 3.01] },
                    { sehir: "Santorini", tarih: "1551", koor: [36.4, 25.4] },
                    { sehir: "Londra", tarih: "1551", koor: [51.50, -0.12] }
                ]
            },
            "locke": {
                isim: "John Locke", renk: "#2980b9", meslek: "Gemi Kaptanı", amac: "Hac (Pilgrimage)",
                tarihAraligi: "1553 - 1554",
                rota: [
                    { sehir: "Cadiz", tarih: "26 Mart 1553", koor: [36.52, -6.28] },
                    { sehir: "Venedik", tarih: "Mayıs 1553", koor: [45.44, 12.31] },
                    { sehir: "Limasol", tarih: "14 Ağustos 1553", koor: [34.67, 33.04] },
                    { sehir: "Kudüs", tarih: "Eylül 1553", koor: [31.76, 35.21] },
                    { sehir: "Venedik", tarih: "02 Ocak 1554", koor: [45.44, 12.31] }
                ]
            },
            "foxe": {
                isim: "John Foxe", renk: "#e67e22", meslek: "Topçu (Master Gunner)", amac: "Esaret (Captivity)",
                tarihAraligi: "1563",
                rota: [
                    { sehir: "İskenderiye", tarih: "1563", koor: [31.20, 29.91] }
                ]
            }
        };

        var aktifCizgi = null;
        var aktifNoktalar = L.layerGroup().addTo(map);

        function haritayiGuncelle() {
            var val = document.getElementById("seyyahSelect").value;
            var s = seyyahKatalogu[val];

            if (aktifCizgi) map.removeLayer(aktifCizgi);
            aktifNoktalar.clearLayers();

            // Sütunlu/Kartlı Künye Tasarımı
            var kunyeHTML = `
                <div class="kunye-row">
                    <span class="kunye-label">Seyyah İsmi</span>
                    <span class="kunye-value">${s.isim}</span>
                </div>
                <div class="kunye-row" style="border-left-color: #9b59b6;">
                    <span class="kunye-label">Meslek</span>
                    <span class="kunye-value">${s.meslek}</span>
                </div>
                <div class="kunye-row" style="border-left-color: #e67e22;">
                    <span class="kunye-label">Seyahat Amacı</span>
                    <span class="kunye-value">${s.amac}</span>
                </div>
                <div class="kunye-row" style="border-left-color: #2ecc71;">
                    <span class="kunye-label">Seyahat Tarihi</span>
                    <span class="kunye-value">${s.tarihAraligi}</span>
                </div>
                
                <div class="rota-section">
                    <span class="rota-title">Gidilen Duraklar</span>
                    ${s.rota.map(d => `
                        <div class="rota-item">
                            <span>${d.sehir}</span>
                            <span class="rota-date">${d.tarih}</span>
                        </div>
                    `).join('')}
                </div>
            `;
            document.getElementById("kunyeGosterim").innerHTML = kunyeHTML;

            var pts = s.rota.map(p => p.koor);
            if(pts.length > 1) {
                aktifCizgi = L.polyline(pts, {color: s.renk, weight: 5, opacity: 0.7}).addTo(map);
                map.fitBounds(aktifCizgi.getBounds(), {padding: [50, 50]});
            } else {
                map.setView(pts[0], 6);
            }

            s.rota.forEach(function(d) {
                L.circleMarker(d.koor, {radius: 8, color: '#fff', weight: 2, fillColor: s.renk, fillOpacity: 1})
                    .addTo(aktifNoktalar)
                    .bindPopup(`<b>${d.sehir}</b><br>${d.tarih}`);
            });
        }

        haritayiGuncelle();
    </script>
</body>
</html>
