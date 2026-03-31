# Panduan Belajar & Pertanyaan Sidang — TURUN

## BAGIAN 1: APA YANG HARUS KAMU PELAJARI

---

### 1. Flutter & Dart (Frontend)

**Yang wajib paham:**
- **Widget Tree** — Semua di Flutter itu widget. Pahami bedanya `StatelessWidget` vs `StatefulWidget`. Kapan pakai yang mana?
  - `StatelessWidget`: UI yang ga berubah (misal: label, icon)
  - `StatefulWidget`: UI yang berubah-ubah (misal: form input, toggle, counter)
- **setState()** — Fungsi untuk trigger rebuild UI saat data berubah. Kalau ditanya "gimana UI-nya update?", jawabannya pasti lewat `setState()` atau state management.
- **State Management** — Kamu pakai apa? Provider? Riverpod? BLoC? Pahami kenapa pilih itu dan gimana flow datanya.
- **Navigator & Routing** — Gimana berpindah antar halaman. `Navigator.push()`, `Navigator.pop()`, atau pakai named routes.
- **Google Maps Plugin** — `google_maps_flutter` package. Pahami `GoogleMap` widget, `Marker`, `Polygon` (buat territory), `Polyline` (buat rute lari).
- **Geolocator** — Package untuk dapat lokasi GPS. Pahami `getCurrentPosition()`, `getPositionStream()` untuk real-time tracking.
- **Build & Run** — Cara `flutter run`, `flutter build apk`, bedanya debug vs release mode.
- **Lifecycle** — `initState()`, `dispose()`, `didChangeDependencies()`. Kapan stream GPS di-start dan di-stop.
- **async/await** — Flutter itu single-threaded. Semua operasi network/database pakai `Future` dan `async/await`. Pahami bedanya `Future` vs `Stream`.

**Buka dan pelajari file-file ini di project kamu:**
- `lib/main.dart` — Entry point, pahami struktur app-nya
- `lib/screens/` — Semua halaman UI
- `lib/services/` atau `lib/providers/` — Logic bisnis, koneksi ke Supabase
- `lib/models/` — Data model (User, Territory, Activity, dll)
- `pubspec.yaml` — Dependencies apa aja yang dipakai

---

### 2. Supabase (Backend)

**Yang wajib paham:**
- **Apa itu Supabase** — Open-source Firebase alternative, pakai PostgreSQL. Jawab: "Saya pilih Supabase karena pakai PostgreSQL yang support PostGIS untuk spatial data, dan ada fitur Auth, Realtime, dan Storage built-in."
- **Authentication** — Gimana user register dan login.
  ```dart
  // Register
  supabase.auth.signUp(email: email, password: password);
  // Login
  supabase.auth.signInWithPassword(email: email, password: password);
  // Google OAuth
  supabase.auth.signInWithOAuth(OAuthProvider.google);
  ```
- **Database Query** — Gimana fetch dan insert data.
  ```dart
  // Fetch data
  final data = await supabase.from('territories').select();
  // Insert data
  await supabase.from('activities').insert({'user_id': userId, 'distance': distance});
  // Update
  await supabase.from('territories').update({'owner_id': userId}).eq('id', territoryId);
  ```
- **Realtime Subscription** — Gimana dapet notifikasi real-time.
  ```dart
  supabase.from('territory_captures').stream(primaryKey: ['id']).listen((data) {
    // Update UI ketika ada territory yang direbut
  });
  ```
- **Row Level Security (RLS)** — Policy yang ngatur siapa boleh akses data apa. Contoh: user cuma bisa edit data miliknya sendiri.
- **Storage** — Upload dan akses file (foto profil, gambar landmark).
- **RPC (Remote Procedure Call)** — Cara panggil function PostgreSQL dari Flutter. Penting buat territory capture logic yang complex.

**Buka Supabase Dashboard kamu dan pelajari:**
- Table structure — tabel apa aja, kolom apa, relasi antar tabel
- RLS Policies — buka tiap tabel, liat policy-nya
- SQL Editor — coba jalanin query territory manual
- Edge Functions (kalau ada)
- Auth settings

---

### 3. PostGIS (Spatial Database)

**Yang wajib paham:**
- **Apa itu PostGIS** — Ekstensi PostgreSQL untuk data geospasial. Bisa simpan titik koordinat, garis, polygon, dan jalankan query spasial.
- **Data Type** — `geography` atau `geometry`. Territory boundary disimpan sebagai `POLYGON`, rute lari sebagai `LINESTRING`, lokasi landmark sebagai `POINT`.
- **Bedanya geography vs geometry** — `geography` menghitung jarak/area berdasarkan permukaan bumi (akurat tapi lambat). `geometry` menghitung pada bidang datar (cepat tapi kurang akurat untuk area luas). Untuk skala kota (Batam), `geometry` dengan SRID 4326 sudah cukup.
- **SRID 4326** — Spatial Reference ID untuk WGS84, sistem koordinat yang dipakai GPS. Longitude/Latitude.
- **Fungsi PostGIS yang kamu pakai:**
  - `ST_Intersects(geom1, geom2)` — Cek apakah dua geometri saling berpotongan. Dipakai untuk: "apakah rute lari user melewati territory ini?"
  - `ST_Contains(geom1, geom2)` — Cek apakah geom1 memuat geom2 sepenuhnya
  - `ST_Area(geom)` — Hitung luas area territory
  - `ST_DWithin(geom1, geom2, distance)` — Cek apakah dua geometri dalam jarak tertentu. Dipakai untuk: "apakah user dekat dengan landmark?"
  - `ST_MakePoint(longitude, latitude)` — Buat titik dari koordinat
  - `ST_SetSRID()` — Set sistem koordinat pada geometri
  - `ST_AsGeoJSON()` — Convert geometri ke format GeoJSON (untuk kirim ke Flutter)
  - `ST_GeomFromGeoJSON()` — Convert GeoJSON ke geometri (untuk simpan dari Flutter)
- **Contoh query territory capture:**
  ```sql
  SELECT * FROM territories
  WHERE ST_Intersects(
    geo_boundary,
    ST_SetSRID(
      ST_MakeLine(ARRAY[ST_MakePoint(lon1,lat1), ST_MakePoint(lon2,lat2), ...]),
      4326
    )
  );
  ```

---

### 4. Google Maps & Territory Visualization (KRUSIAL)

**Ini bagian yang paling sering ditanya karena visual dan gampang di-challenge.**

**Yang wajib paham:**

**a) Setup Google Maps di Flutter:**
- API Key dari Google Cloud Console → enable "Maps SDK for Android" dan "Maps SDK for iOS"
- API Key ditaruh di `android/app/src/main/AndroidManifest.xml` (Android) dan `ios/Runner/AppDelegate.swift` (iOS)
- Package: `google_maps_flutter` di `pubspec.yaml`

**b) Render Territory (Polygon berwarna) di Peta:**
```dart
// Setiap territory punya boundary polygon + owner + warna
Set<Polygon> territoryPolygons = territories.map((territory) {
  return Polygon(
    polygonId: PolygonId(territory.id),
    points: territory.boundaryCoordinates.map((coord) =>
      LatLng(coord.latitude, coord.longitude)
    ).toList(),
    fillColor: Color(territory.ownerColor).withOpacity(0.3), // warna semi-transparan
    strokeColor: Color(territory.ownerColor),                  // border warna solid
    strokeWidth: 2,
  );
}).toSet();

// Di widget:
GoogleMap(
  polygons: territoryPolygons,
  polylines: runningRoutePolylines,
  markers: landmarkMarkers,
  ...
)
```

**c) Gimana Territory Punya Warna Berbeda per Owner:**
- Setiap user punya `profile_color` di tabel Users (disimpan sebagai hex string, misal "#FF5733")
- Saat fetch territories, JOIN dengan tabel Users untuk dapat warna owner
- Territory yang belum di-claim: warna abu-abu / default
- Territory yang sudah di-claim: warna sesuai owner-nya
- Pakai `fillColor` dengan opacity rendah (0.2-0.4) supaya peta di bawahnya masih keliatan
- `strokeColor` warna solid buat border biar keliatan jelas batasnya

**d) Gimana Data Territory Boundary Dibuat:**
- Territory boundary (polygon) bisa di-define secara manual (admin input koordinat)
- Atau generate grid otomatis berdasarkan area tertentu
- Disimpan di Supabase sebagai kolom `geo_boundary` bertipe PostGIS geometry/geography
- Saat fetch ke Flutter, data di-convert ke GeoJSON → parse jadi list `LatLng`

**e) Render Rute Lari (Polyline):**
```dart
Polyline(
  polylineId: PolylineId('running_route'),
  points: gpsCoordinates.map((c) => LatLng(c.lat, c.lng)).toList(),
  color: Colors.blue,
  width: 4,
)
```

**f) Render Landmark (Marker):**
```dart
Marker(
  markerId: MarkerId(landmark.id),
  position: LatLng(landmark.lat, landmark.lng),
  infoWindow: InfoWindow(title: landmark.name, snippet: landmark.description),
  icon: BitmapDescriptor.defaultMarkerWithHue(BitmapDescriptor.hueGreen),
)
```

**g) Camera & Viewport:**
- `CameraPosition` — set initial posisi peta (center di Batam)
- `GoogleMapController.animateCamera()` — gerakin kamera ke posisi tertentu
- `LatLngBounds` — buat ngatur zoom supaya semua territory keliatan

**h) Update Peta Real-time:**
- Saat user lari, setiap koordinat baru ditambah ke list → `setState()` → Polyline re-render
- Saat territory berubah owner (Realtime subscription), fetch ulang territories → update Polygon warna

---

### 5. Arsitektur & Flow Aplikasi

**Pahami flow ini:**
1. **User register/login** → Supabase Auth → dapat session token (JWT)
2. **App load** → Fetch semua territories dari Supabase → Convert geo_boundary ke list LatLng → Render Polygon di Google Maps dengan warna owner
3. **User mulai lari** → Geolocator stream GPS coordinates → simpan ke list → render Polyline di map real-time
4. **User selesai lari** → Koordinat dikirim ke Supabase → PostGIS cek `ST_Intersects` dengan territory boundaries → kalau overlap, update territory owner → update warna territory di peta
5. **Notifikasi Under Attack** → Supabase Realtime subscription detect perubahan di `territory_captures` → trigger notifikasi ke owner lama → update peta owner baru
6. **Leaderboard** → Query `SUM(ST_Area(geo_boundary))` per user → sort descending
7. **Landmark** → User create landmark → simpan POINT ke Supabase → tampil sebagai Marker di peta semua user

---

### 6. UCD & SUS (Metodologi)

**User-Centered Design:**
- 4 fase: Understand → Specify → Design → Evaluate
- Iteratif — bisa balik ke fase sebelumnya
- Standar: ISO 9241-210
- Bedakan dengan Waterfall: UCD fokus ke user, iteratif. Waterfall linear, sequential.
- Bedakan dengan Design Thinking: DT lebih broad (empathize, define, ideate, prototype, test). UCD lebih spesifik ke software usability.

**System Usability Scale:**
- Dibuat oleh John Brooke (1986)
- 10 pernyataan, skala Likert 1-5
- Ganjil = positif (1,3,5,7,9), Genap = negatif (2,4,6,8,10)
- Rumus: Ganjil (skor - 1), Genap (5 - skor), total × 2.5
- Range 0-100
- Skor kamu: 76,0 → Grade B → Good
- Rata-rata global: 68
- Pahami kenapa P3 (kemudahan) tinggi dan P10 (learning curve) rendah

---

## BAGIAN 2: PERTANYAAN YANG KEMUNGKINAN DITANYA

---

### Kategori A: Flutter & Frontend

**A1. "Kenapa pilih Flutter, bukan React Native atau native Android?"**
> Flutter satu codebase untuk Android dan iOS, performa mendekati native karena compile ke native code lewat Dart AOT compilation (bukan bridge kayak React Native), hot reload bikin development cepat, dan widget system-nya lengkap buat UI yang custom.

**A2. "Jelaskan perbedaan StatelessWidget dan StatefulWidget. Di project kamu, mana yang pakai Stateful?"**
> StatelessWidget untuk UI yang ga berubah, StatefulWidget untuk UI yang dinamis. Di TURUN, halaman running session pakai StatefulWidget karena data GPS terus update, timer berjalan, dan peta harus re-render. Halaman about atau splash screen pakai Stateless.

**A3. "State management apa yang kamu pakai? Kenapa?"**
> (Jawab sesuai yang kamu pakai — Provider/Riverpod/BLoC/GetX. Pahami minimal: di mana state disimpan, gimana widget listen ke perubahan, gimana trigger update.)

**A4. "Gimana cara tracking GPS real-time di Flutter?"**
> Pakai package `geolocator`. Saya panggil `Geolocator.getPositionStream()` yang return Stream of Position. Setiap ada posisi baru, saya tambahkan ke list koordinat dan update Polyline di Google Maps widget lewat setState.

**A5. "Gimana cara handle kalau GPS-nya ga akurat atau sinyal lemah?"**
> Ada parameter `desiredAccuracy` di Geolocator yang saya set ke `LocationAccuracy.high`. Selain itu, bisa filter titik yang akurasinya di bawah threshold tertentu (misal > 20 meter accuracy, skip titik itu).

**A6. "Kalau user minimize app, tracking tetap jalan ga?"**
> (Jawab sesuai implementasi — pakai background service atau tidak. Kalau pakai, sebutkan package-nya. Kalau belum, jujur aja dan bilang itu jadi saran pengembangan.)

**A7. "Apa itu Hot Reload dan Hot Restart? Bedanya apa?"**
> Hot Reload: inject perubahan kode ke running app tanpa kehilangan state. Hot Restart: restart app dari awal, state hilang. Hot Reload lebih cepat tapi ga bisa handle perubahan structural kayak tambah parameter di constructor.

**A8. "Jelaskan lifecycle StatefulWidget."**
> `createState()` → `initState()` (dipanggil sekali, setup awal) → `build()` (render UI) → `didUpdateWidget()` (saat parent rebuild) → `dispose()` (cleanup, cancel stream, timer). Di TURUN, saya start GPS stream di `initState()` dan cancel di `dispose()`.

**A9. "Apa itu BuildContext? Untuk apa?"**
> BuildContext itu referensi ke posisi widget di widget tree. Dipakai untuk akses theme, navigator, media query, dan inherited widget (kayak Provider). Setiap widget punya context sendiri.

**A10. "Bedanya `Future` dan `Stream` di Dart?"**
> Future itu single async result — satu kali selesai. Contoh: fetch data dari API. Stream itu continuous — bisa emit banyak value over time. Contoh: GPS position updates, Supabase realtime subscription.

---

### Kategori B: Supabase & Backend

**B1. "Kenapa Supabase bukan Firebase?"**
> Supabase pakai PostgreSQL yang open-source dan support PostGIS untuk spatial queries. Firebase pakai NoSQL (Firestore) yang ga cocok untuk geospatial data. Selain itu, Supabase lebih fleksibel karena bisa jalankan raw SQL query dan ada Row Level Security bawaan.

**B2. "Jelaskan Row Level Security yang kamu implementasi."**
> RLS itu policy di level database yang ngatur akses data per-row. Contoh: user cuma bisa update profil sendiri. Policy-nya ditulis di SQL: `CREATE POLICY "Users can update own profile" ON users FOR UPDATE USING (auth.uid() = user_id)`. Jadi walau user coba query data orang lain, database otomatis block.

**B3. "Gimana proses autentikasi bekerja? Jelaskan step by step."**
> User register dengan email/password → Supabase hash password dan simpan di auth.users → Supabase generate JWT token → token disimpan di device (secure storage) → setiap request ke database, token dikirim di Authorization header → Supabase verify JWT, extract user_id → RLS policy di-apply berdasarkan user_id itu.

**B4. "Apa itu JWT? Isinya apa?"**
> JSON Web Token. Ada 3 bagian: Header (algoritma), Payload (data user: user_id, email, expiry time), Signature (verifikasi integritas). Supabase generate ini saat login, dan Flutter simpan untuk autentikasi subsequent requests.

**B5. "Gimana Realtime subscription bekerja untuk notifikasi Under Attack?"**
> Supabase Realtime pakai WebSocket connection yang persistent. Saya subscribe ke perubahan di tabel territory_captures. Setiap ada INSERT atau UPDATE, Supabase broadcast event ke semua connected client. Di Flutter, saya listen stream itu, cek apakah territory yang berubah milik current user, kalau iya tampilkan notifikasi.

**B6. "Data apa aja yang disimpan di database? Jelaskan struktur tabel dan relasinya."**
> Users (profil, poin, tier), Activities (riwayat lari: jarak, durasi, pace), Activity Coordinates (titik GPS per aktivitas, relasi many-to-one ke Activities), Territories (wilayah dengan geo_boundary polygon, owner_id FK ke Users), Territory Captures (riwayat penguasaan: siapa capture kapan), Goals (target harian user), Landmarks (titik landmark dengan posisi POINT).

**B7. "Gimana handle concurrent territory capture? Kalau 2 orang claim bersamaan?"**
> Pakai database transaction dengan row-level locking. Di Supabase, saya buat RPC function (stored procedure) yang: BEGIN transaction → SELECT territory FOR UPDATE (lock row) → cek siapa yang duluan → UPDATE owner → COMMIT. Yang kedua akan wait sampai lock dilepas, lalu dapat territory yang udah berubah owner.

**B8. "Kalau Supabase down, app-nya gimana?"**
> Saat ini belum ada offline-first mode. Kalau Supabase down, fitur yang butuh network (login, fetch territory, save run) ga jalan. Saran pengembangan: implementasi local caching pakai Hive atau drift (SQLite) untuk cache data territory dan riwayat lari, lalu sync saat koneksi kembali.

**B9. "Gimana handle kalau user jahat manipulasi data? Misal fake GPS?"**
> Di level database, RLS mencegah user mengubah data orang lain. Untuk fake GPS, itu memang limitasi — bisa diatasi dengan server-side validation (cek kecepatan antar titik, kalau terlalu cepat = suspicious), tapi ini belum diimplementasi dan jadi saran pengembangan.

**B10. "Supabase itu hosted di mana? Data privacy gimana?"**
> Supabase Cloud di-host di AWS. Untuk project ini saya pakai free tier. Data disimpan di region yang dipilih saat create project. Kalau concern privacy, bisa self-host Supabase karena open-source.

---

### Kategori C: PostGIS & Geospasial

**C1. "Apa itu PostGIS dan kenapa kamu butuh itu?"**
> PostGIS itu ekstensi PostgreSQL untuk data geospasial. Saya butuh karena territory capture perlu ngecek apakah rute lari pengguna melewati area territory tertentu — itu butuh spatial query yang efisien. PostgreSQL biasa ga bisa hitung intersection antara polygon dan line.

**C2. "Jelaskan gimana territory capture secara teknis bekerja. Step by step."**
> 1) User selesai lari, Flutter kirim array koordinat GPS ke Supabase. 2) Di Supabase, koordinat di-convert jadi LINESTRING pakai ST_MakeLine. 3) Query jalankan ST_Intersects antara LINESTRING rute dan POLYGON geo_boundary setiap territory. 4) Territory yang intersect = user melewati situ. 5) Update owner_id territory ke user yang baru capture. 6) Insert record di territory_captures untuk history. 7) Realtime broadcast ke semua client.

**C3. "Apa bedanya ST_Intersects dan ST_Contains?"**
> `ST_Intersects` — TRUE kalau ada bagian yang overlap walau cuma sedikit. `ST_Contains` — TRUE kalau geometry pertama memuat geometry kedua sepenuhnya. Untuk territory capture saya pakai Intersects karena user cukup melewati sebagian territory, ga perlu seluruhnya.

**C4. "Gimana cara simpan boundary territory di database?"**
> Kolom `geo_boundary` bertipe `geometry(Polygon, 4326)`. Value contoh: `ST_SetSRID(ST_GeomFromGeoJSON('{"type":"Polygon","coordinates":[[[104.05,1.13],[104.06,1.13],[104.06,1.14],[104.05,1.14],[104.05,1.13]]]}'), 4326)`. Titik pertama dan terakhir harus sama (closed polygon). SRID 4326 = WGS84 (sistem koordinat GPS).

**C5. "Apa itu SRID 4326? Kenapa pakai itu?"**
> SRID (Spatial Reference System Identifier) 4326 adalah WGS84 — sistem koordinat yang dipakai GPS di seluruh dunia. Latitude/Longitude. Saya pakai ini karena data dari GPS device (Geolocator) sudah dalam format WGS84.

**C6. "Gimana performa spatial query kalau territory-nya banyak?"**
> PostGIS pakai GiST (Generalized Search Tree) spatial index. Jadi ga scan semua territory — index memfilter berdasarkan bounding box dulu, baru cek intersection detail. Untuk skala kota Batam (puluhan-ratusan territory), performanya sangat cepat.

**C7. "Apa bedanya geometry dan geography di PostGIS?"**
> `geometry` — hitung di bidang datar (Cartesian), cepat tapi kurang akurat untuk jarak jauh. `geography` — hitung di permukaan bola bumi (geodetic), akurat tapi lebih lambat. Untuk skala kota (Batam), geometry sudah cukup akurat dan lebih performant.

**C8. "Gimana kalau mau extend ke kota lain? Apa yang perlu diubah?"**
> Tinggal tambah data territory baru dengan boundary polygon kota itu. Karena pakai PostGIS dengan SRID 4326 (global coordinate system), ga ada perubahan kode. Yang perlu: define territory boundaries kota baru dan insert ke database.

---

### Kategori D: Google Maps & Territory Visualization (KRUSIAL)

**D1. "Gimana territory bisa muncul di peta dengan warna berbeda-beda?"**
> Setiap territory punya owner, dan setiap owner punya profile_color. Saat fetch territories dari Supabase, saya JOIN dengan tabel users untuk dapat warna. Di Flutter, saya buat `Polygon` widget untuk setiap territory — `fillColor` diisi warna owner dengan opacity rendah (0.3) supaya peta masih keliatan, `strokeColor` warna solid buat border. Territory tanpa owner dikasih warna abu-abu default.

**D2. "Coba jelaskan kode yang render territory di peta. Gimana flow datanya?"**
> 1) Fetch data territory dari Supabase: `supabase.from('territories').select('*, users(profile_color)')`. 2) Parse geo_boundary dari GeoJSON jadi list LatLng. 3) Buat `Set<Polygon>` dari semua territory. 4) Pass ke `GoogleMap(polygons: ...)`. 5) Saat territory berubah owner (via Realtime), fetch ulang → rebuild Polygon set → `setState()` → peta update warna.

**D3. "Gimana convert data PostGIS boundary jadi Polygon di Google Maps Flutter?"**
> Data geo_boundary disimpan sebagai PostGIS geometry. Saat fetch, saya pakai `ST_AsGeoJSON(geo_boundary)` yang return JSON string berisi array koordinat. Di Flutter, parse JSON itu jadi `List<LatLng>`, lalu buat `Polygon(points: latLngList)`.

**D4. "Kenapa pakai Google Maps bukan Mapbox, OpenStreetMap, atau Leaflet?"**
> Google Maps paling familiar buat user Indonesia, akurasi peta di Indonesia paling bagus, official Flutter plugin (`google_maps_flutter`) well-maintained oleh Flutter team, dan dokumentasinya lengkap. Mapbox juga bagus tapi setup-nya lebih complex dan biaya untuk production lebih tinggi.

**D5. "Google Maps API itu gratis? Kalau banyak user gimana?"**
> Ada free quota: $200/bulan credit gratis dari Google (sekitar 28,000 map loads). Untuk skala tugas akhir dan testing ini cukup. Kalau scale ke production dengan ribuan user, perlu budgeting. Alternatif: switch ke Mapbox atau OpenStreetMap yang lebih murah.

**D6. "Gimana handle kalau territory boundaries-nya overlap satu sama lain?"**
> Idealnya territory boundaries ga boleh overlap — di-enforce saat admin define territory. Bisa pakai PostGIS `ST_Overlaps()` untuk validasi sebelum insert territory baru. Kalau ternyata overlap, user yang claim = dapet semua territory yang di-intersect rute lari-nya.

**D7. "Gimana cara user tau dia ada di territory mana saat lari?"**
> Saat running session, posisi GPS real-time dicek terhadap territory boundaries yang sudah di-load di Flutter. Bisa pakai point-in-polygon check secara lokal di Flutter (tanpa query ke server) untuk responsiveness. Territory yang sedang dilewati bisa di-highlight di peta.

**D8. "Kalau peta-nya zoom out jauh, semua territory keliatan ga? Gimana performa-nya?"**
> Semua Polygon tetap di-render. Untuk optimasi: bisa pakai clustering atau simplify polygon saat zoom out (kurangi jumlah titik). Tapi untuk skala Batam dengan puluhan territory, performa masih oke tanpa optimasi khusus.

**D9. "Gimana cara bikin route summary setelah lari? Yang ada jarak, durasi, dll?"**
> Jarak dihitung dari total jarak antar titik GPS berurutan (pakai rumus Haversine atau Geolocator.distanceBetween). Durasi = selisih waktu start dan stop. Pace = durasi / jarak. Kecepatan rata-rata = jarak / durasi. Semua dihitung di Flutter sebelum kirim ke Supabase.

**D10. "Gimana handle kalau user lari tapi GPS loncat-loncat (GPS drift)?"**
> Filter berdasarkan accuracy — kalau accuracy > threshold (misal 30m), skip titik itu. Bisa juga filter berdasarkan speed — kalau speed antar dua titik > threshold (misal 50 km/h untuk lari = impossible), skip titik itu. Ini mencegah garis rute yang zigzag ga jelas.

---

### Kategori E: Arsitektur & Design Pattern

**E1. "Gambarkan flow dari user mulai lari sampai territory ter-capture."**
> User tekan Start → Geolocator start stream GPS → setiap posisi baru: tambah ke list, update Polyline di map → user tekan Stop → hitung jarak/durasi/pace di Flutter → kirim array koordinat ke Supabase via RPC → PostGIS convert ke LINESTRING → ST_Intersects cek semua territory → territory yang match: update owner → insert ke territory_captures → Realtime broadcast → semua client update warna territory di peta → owner lama dapat notifikasi Under Attack.

**E2. "Arsitektur app kamu itu apa? MVC? MVVM? Clean Architecture?"**
> (Jawab sesuai yang kamu pakai. Kalau ga yakin, liat folder structure: kalau ada screens/services/models = layered architecture. Kalau ada features/domain/data = clean architecture. Kalau ada controllers = MVC.)

**E3. "Kenapa ga pakai microservices?"**
> Untuk skala tugas akhir, monolith sudah cukup. Supabase already handle auth, database, realtime, dan storage sebagai managed services. Microservices overkill untuk app dengan single team developer dan user base kecil.

**E4. "Gimana error handling di app kamu?"**
> Pakai try-catch di setiap async operation (network call, database query). Kalau error, tampilkan SnackBar atau Dialog ke user dengan pesan yang user-friendly. Di level Supabase, RLS otomatis return error kalau user ga punya akses.

**E5. "Gimana testing di project kamu?"**
> (Jawab jujur — kalau cuma SUS testing, bilang aja.) Untuk usability testing pakai SUS dengan 22 responden. Untuk unit testing / widget testing di Flutter, itu bagian yang bisa ditingkatkan sebagai saran pengembangan.

---

### Kategori F: UCD & SUS

**F1. "Jelaskan 4 fase UCD yang kamu lakukan, secara detail."**
> Understand: wawancara semi-terstruktur 4 calon pengguna (usia 17-35, pengguna app lari) → dapat insight: app monoton, cuma statistik, kurang interaktif, terasa kayak kewajiban. Specify: dari insight, define 17 functional requirements (FR-01 sampai FR-17) dan 6 non-functional requirements. Design: low-fi wireframe di Whimsical → high-fi UI di Stitch → implementasi di Flutter. Evaluate: usability testing 22 responden pakai SUS.

**F2. "Kenapa UCD bukan Design Thinking atau Waterfall?"**
> UCD itu standar ISO 9241-210, fokus ke iterasi berulang dengan user di pusat setiap keputusan design. Design Thinking lebih broad, cocok untuk eksplorasi masalah. Waterfall sequential, ga ada feedback loop dari user sampai akhir. UCD paling cocok karena saya butuh validasi dari user di setiap fase.

**F3. "Gimana cara hitung skor SUS? Hitungin dari satu responden sekarang."**
> WAJIB HAFAL. Misal responden jawab: P1=4, P2=2, P3=5, P4=1, P5=4, P6=2, P7=5, P8=1, P9=4, P10=3. Ganjil: (4-1)+(5-1)+(4-1)+(5-1)+(4-1) = 3+4+3+4+3 = 17. Genap: (5-2)+(5-1)+(5-2)+(5-1)+(5-3) = 3+4+3+4+2 = 16. Total = 17+16 = 33. SUS = 33 × 2.5 = 82.5.

**F4. "Skor SUS kamu 76, tapi P10 rendah. Itu masalah ga?"**
> P10 itu "Saya perlu belajar banyak hal sebelum menggunakan sistem ini" (negatif). Skor rendah artinya beberapa user merasa perlu belajar dulu — ini wajar untuk app baru dengan konsep unik (territory capture). Solusi: tambahkan onboarding tutorial dan tooltips di fitur-fitur kunci.

**F5. "Kenapa cuma 22 responden? Apa ga terlalu sedikit?"**
> Menurut Jeff Sauro (2011), 12-14 responden sudah cukup untuk SUS dengan margin of error ±10%. Nielsen Norman Group juga rekomendasikan 20+ untuk estimasi reliable. 22 responden sudah di atas kedua threshold itu.

**F6. "SUS itu ukur usability aja. Gimana kamu ukur engagement?"**
> Selain SUS, saya juga lakukan survei gamifikasi terpisah yang tanya langsung: fitur mana yang paling meningkatkan motivasi lari. Hasilnya: territory capture 54,5%, leaderboard 18,2%, goals 13,6%. Ini mengukur engagement spesifik ke elemen gamifikasi.

**F7. "Agile Scrum kamu 8 sprint. Setiap sprint ngerjain apa?"**
> (Harus bisa jawab minimal garis besarnya.) Sprint 1-2: Setup project, auth, profil. Sprint 3-4: Running tracker, GPS integration. Sprint 5-6: Territory capture, maps, PostGIS. Sprint 7: Leaderboard, achievements, landmarks. Sprint 8: Polish, testing, bug fix. (Sesuaikan dengan timeline asli kamu.)

**F8. "Kamu bilang UCD iteratif. Tapi iterasinya berapa kali? Apa yang berubah setiap iterasi?"**
> (Jawab jujur sesuai proses kamu.) Minimal: wireframe → review → revisi wireframe → UI design → user feedback → revisi → implementasi → SUS testing → identifikasi perbaikan. Setiap iterasi ada perubahan berdasarkan feedback, misal: button placement, flow territory capture yang awalnya confusing, dll.

---

### Kategori G: Pertanyaan Tajam / Jebakan / Dosen Killer

**G1. "Kamu bilang pakai Agile. Tapi kamu develop sendiri. Agile itu buat tim. Gimana tuh?"**
> Agile bisa diadaptasi untuk solo developer. Saya tetap jalanin prinsip-prinsipnya: iterasi pendek (sprint), deliverable tiap sprint, retrospective (evaluasi apa yang bisa diperbaiki). Scrum-nya saya adaptasi tanpa daily standup, tapi tetap ada sprint planning dan review.

**G2. "Fitur territory capture ini konsepnya mirip Ingress atau Pokemon GO. Apa bedanya?"**
> Ingress dan Pokemon GO pakai augmented reality dan fokus ke collecting/battling. TURUN fokus ke running/exercise — territory di-capture lewat aktivitas lari, bukan sekedar visit lokasi. Konteks gamifikasi-nya berbeda: TURUN mendorong olahraga konsisten, bukan exploring location aja.

**G3. "22 responden dari mana? Apa mereka bias karena teman kamu semua?"**
> Responden dipilih berdasarkan kriteria: usia 17-35, pengguna smartphone aktif, pernah atau tertarik pakai app kebugaran. Tidak semua teman — ada yang dari komunitas lari dan mahasiswa jurusan lain. Tapi memang itu salah satu limitasi: belum diuji ke populasi yang lebih luas dan beragam.

**G4. "Kalau saya mau coba app ini sekarang, bisa?"**
> (Siapkan HP dengan app terinstall. Kalau ga bisa demo live, siapkan video recording demo.)

**G5. "Coba tunjukin di kode, di mana territory capture logic-nya."**
> (WAJIB tau lokasi file-nya. Buka folder services/repositories, tunjukin function yang call Supabase RPC untuk territory check. Tunjukin juga PostGIS function di SQL Editor Supabase.)

**G6. "Ganti warna territory ini jadi hijau, sekarang."**
> Buka file model Territory atau screen MapView. Cari dimana `fillColor` dan `strokeColor` di-set. Ganti value warna-nya. Run ulang app.

**G7. "Kalau 1000 orang pakai app ini bersamaan, server kuat ga?"**
> Supabase free tier ada limit concurrent connections. Untuk 1000 user simultan, perlu upgrade ke Pro plan. PostGIS spatial query tetap efisien karena GiST indexing. Bottleneck kemungkinan di Realtime connections. Solusi: upgrade plan, pakai connection pooling (PgBouncer), dan caching layer.

**G8. "Data GPS pengguna itu sensitif. Gimana kamu handle privacy-nya?"**
> Data lokasi disimpan di Supabase (AWS), di-protect dengan RLS — user lain ga bisa akses data aktivitas orang lain. Tapi memang belum ada enkripsi end-to-end atau opsi delete history. Itu masuk saran pengembangan: tambah fitur privacy settings dan data deletion.

**G9. "Kamu pakai AI buat develop. Apa kontribusi kamu sendiri?"**
> AI itu tools, sama kayak IDE atau Stack Overflow. Saya yang define arsitektur, tentukan fitur, buat keputusan teknis (pilih Supabase bukan Firebase, pakai PostGIS untuk spatial), design user flow, define requirements dari hasil wawancara, dan evaluasi hasilnya. AI bantu di implementasi kode, tapi keputusan engineering dan research tetap dari saya.

**G10. "Apa kelemahan terbesar dari app kamu?"**
> (Jujur dan tunjukkan self-awareness.) Belum ada offline mode — kalau ga ada internet, app ga fungsi. GPS battery drain cukup tinggi saat running session. Belum ada social features (chat, komunitas). Onboarding untuk fitur territory masih perlu diperbaiki (learning curve tinggi di SUS P10). Dan testing baru 22 responden di Batam — belum validated untuk kota lain.

**G11. "Kenapa nama-nya TURUN? Apa artinya?"**
> (Pastikan kamu punya jawaban yang jelas untuk ini.)

**G12. "Gamifikasi itu ada teori-nya. Framework apa yang kamu pakai? Octalysis? MDA?"**
> (Cek di Bab 2 kamu. Kalau pakai Octalysis, sebutin 8 core drives-nya dan mana yang relevan ke TURUN: Epic Meaning, Development & Accomplishment, Ownership & Possession. Kalau ga pakai framework spesifik, bilang kamu refer ke elemen gamifikasi umum: points, badges, leaderboard, challenges.)

**G13. "Kamu pakai Scrum tapi ga ada Scrum Master, ga ada Product Owner. Itu masih Scrum?"**
> Saya adaptasi Scrum untuk solo development. Roles digabung — saya jadi developer sekaligus product owner yang prioritize backlog. Dosen pembimbing berperan sebagai stakeholder yang kasih feedback tiap sprint review. Ini memang bukan pure Scrum, tapi prinsip-prinsipnya tetap dijalankan.

**G14. "Kalau diminta tambahin fitur running groups/komunitas, gimana arsitekturnya?"**
> Tambah tabel `groups` dan `group_members` di Supabase. Realtime subscription per group channel. Di Flutter, tambah screen Group List, Group Detail, Group Leaderboard. Bisa pakai Supabase Realtime Channels untuk group-specific updates.

**G15. "Gimana kamu pastiin skor SUS yang kamu hitung itu bener? Ada validasinya?"**
> Saya hitung pakai spreadsheet (Excel), double-check manual untuk beberapa responden. Rumus SUS sudah standar dan straightforward — ga ada ambiguitas dalam perhitungan. (Kalau dosen minta hitung ulang di tempat, HARUS BISA.)

---

## BAGIAN 3: CHECKLIST SEBELUM SIDANG

- [ ] Bisa jalanin `flutter run` tanpa error
- [ ] Bisa demo app di HP (GPS jalan, territory keliatan, bisa lari)
- [ ] Hafal struktur folder project — tau file mana untuk apa
- [ ] Bisa tunjukin kode territory capture logic (Flutter + SQL)
- [ ] Bisa tunjukin kode Google Maps Polygon rendering
- [ ] Bisa jelaskan setiap tabel di Supabase dan relasinya
- [ ] Hafal rumus SUS dan bisa hitung manual dari satu responden
- [ ] Bisa jelaskan flow territory capture end-to-end
- [ ] Bisa jelaskan minimal 5 fungsi PostGIS
- [ ] Bisa jelaskan kenapa pilih Flutter, Supabase, PostGIS (bukan alternatif)
- [ ] Bisa jelaskan 4 fase UCD dengan contoh konkret
- [ ] Bisa jelaskan setiap sprint di Agile secara garis besar
- [ ] Punya jawaban untuk "apa limitasi penelitian kamu"
- [ ] Punya jawaban untuk "apa kontribusi kamu vs AI"
- [ ] Punya jawaban untuk "apa saran pengembangan"
- [ ] Baca ulang Bab 1-5 tugas akhir, pahami SEMUA bagian
- [ ] Siapkan video demo sebagai backup kalau HP bermasalah

---

## BAGIAN 4: STRATEGI SIDANG

### Kalau Ga Bisa Jawab:
1. **Jangan panik, jangan diam.** Bilang: "Kalau tidak salah..." lalu coba jawab dengan logika.
2. **Kalau bener-bener ga tau:** "Untuk detail teknis yang sangat spesifik itu, saya perlu cek kembali. Tapi secara konsep, yang saya pahami adalah..."
3. **Arahkan ke yang kamu kuasai.** Misal ditanya internal PostGIS indexing, jawab: "Secara implementasi di project saya, yang saya lakukan adalah pakai ST_Intersects dan sudah tambah GiST index..."
4. **Jujur tapi profesional.** "Itu bagian yang masih perlu saya eksplorasi lebih dalam" > ngasal jawab ketahuan.

### Kalau Diminta Live Coding:
1. **Tenang.** Biasanya yang diminta itu hal simple: ganti warna, tambah field, fix minor bug.
2. **Buka file yang bener.** Ini kunci — kalau kamu tau struktur project, setengah masalah solved.
3. **Narasi sambil coding.** "Saya buka file screen map view... di sini saya cari bagian Polygon... ini dia fillColor-nya... saya ganti ke Colors.green..."
4. **Kalau stuck:** "Saya biasanya refer ke dokumentasi Flutter/Supabase untuk syntax yang spesifik, tapi secara logic, yang perlu dilakukan adalah..."

### Mindset:
- Dosen penguji mau liat kamu **paham konsep**, bukan hafal kode line by line
- Kalau ditanya "kenapa" → jawab dengan **reasoning**, bukan cuma "karena itu best practice"
- Kalau ditanya soal AI → **jangan defensif**. Tunjukin kamu paham apa yang AI tulis, itu yang penting
- **Preparation is 90% of confidence.** Baca file-file kunci di project, jalanin app, coba SQL query di Supabase dashboard
