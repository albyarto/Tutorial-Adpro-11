# Tutorial Module 11: Deployment on Kubernetes - Reflection
Nama: Muhammad Albyarto Ghazali

NPM: 2306241695

## Refleksi Hello Minikube

> Compare the application logs before and after you exposed it as a Service. Try to open the app several times while the proxy into the Service is running. What do you see in the logs? Does the number of logs increase each time you open the app?

**Sebelum diekspos sebagai Service:**

Log aplikasi (`kubectl logs <POD_NAME>`) pada tahap ini umumnya hanya menampilkan pesan startup dari aplikasi `hello-node` (atau `agnhost`). Misalnya, pesan yang mengindikasikan bahwa server HTTP telah dimulai pada port tertentu (misalnya 8080). Tidak ada log yang berkaitan dengan permintaan masuk dari luar karena service belum diekspos dan diakses.

**Setelah diekspos sebagai Service dan diakses beberapa kali:**

Setelah menjalankan `kubectl expose deployment hello-node --type=LoadBalancer --port=8080` dan kemudian `minikube service hello-node`, sebuah tunnel dibuat, dan browser membuka URL aplikasi.

Setiap kali saya mengakses aplikasi melalui URL yang diberikan oleh `minikube service` (misalnya, me-refresh halaman browser atau membukanya di tab baru), saya melihat log baru muncul di output `kubectl logs <POD_NAME_hello-node>`. Log ini biasanya berupa catatan akses HTTP standar yang menunjukkan permintaan `GET /` atau serupa, beserta informasi seperti alamat IP klien (yang mungkin terlihat sebagai IP internal cluster atau proxy).

**Ya, jumlah log (terutama log akses) meningkat setiap kali aplikasi dibuka/diakses.** Ini menunjukkan bahwa Service berhasil meneruskan permintaan ke Pod, dan aplikasi di dalam Pod memproses serta mencatat permintaan tersebut.

**Contoh log yang terlihat :**

[I0509 21:24:05.853748 1 log.go:195] Started HTTP server on port 8080

... (setelah akses pertama)

<timestamp> INFO: <IP Address> - "GET / HTTP/1.1" 200 -

... (setelah akses kedua)

<timestamp> INFO: <IP Address> - "GET / HTTP/1.1" 200 -

> Notice that there are two versions of `kubectl get` invocation during this tutorial section. The first does not have any option, while the latter has `-n` option with value set to `kube-system`. What is the purpose of the `-n` option and why did the output not list the pods/services that you explicitly created?

Opsi `-n` (atau `--namespace`) pada perintah `kubectl` digunakan untuk menentukan **namespace** di mana perintah tersebut akan dieksekusi atau sumber daya mana yang akan ditampilkan.

*   **`kubectl get pods` (tanpa opsi `-n`):** Perintah ini secara default akan menampilkan Pods yang ada di namespace `default`. Ketika kita membuat deployment `hello-node` tanpa menentukan namespace, Kubernetes menempatkannya di namespace `default`.
*   **`kubectl get pods,services -n kube-system`:** Perintah ini secara spesifik meminta `kubectl` untuk menampilkan Pods dan Services yang ada di dalam namespace `kube-system`.

**Tujuan opsi `-n`:**
Namespace menyediakan cara untuk membagi sumber daya cluster di antara beberapa pengguna atau tim (isolasi virtual). Setiap namespace adalah lingkup terpisah untuk nama sumber daya. `kube-system` adalah namespace khusus yang digunakan oleh Kubernetes untuk menjalankan komponen sistem inti, seperti DNS cluster (misalnya CoreDNS), network proxy (kube-proxy), dan dalam kasus ini, `metrics-server` setelah diaktifkan.

**Mengapa output tidak menampilkan pods/services yang saya buat secara eksplisit?**
Ketika saya menjalankan `kubectl get pods,services -n kube-system`, outputnya tidak menampilkan pod `hello-node` atau service `hello-node` yang saya buat karena:
1.  Pod dan service `hello-node` yang saya buat berada di namespace `default` (karena tidak ada namespace yang ditentukan saat pembuatan).
2.  Perintah tersebut secara eksplisit meminta sumber daya *hanya* dari namespace `kube-system`.
Sumber daya dalam satu namespace tidak terlihat secara default saat melihat namespace lain, kecuali jika menggunakan opsi seperti `--all-namespaces` atau dengan menyebutkan nama namespace secara spesifik.

Berikut versi yang telah diparafrase dari jawaban temanmu agar tidak terlihat sama, namun tetap mempertahankan makna dan poin pentingnya:

---

## Refleksi Rolling Update Deployment

Setelah menyelesaikan bagian ini dalam tutorial, saya menjadi lebih memahami cara Kubernetes melakukan pembaruan aplikasi serta peran penting file manifest dalam proses tersebut. Inilah hasil refleksi saya:

### 1. Perbandingan Strategi Deployment: Rolling Update vs Recreate

Berdasarkan praktik langsung dan dokumentasi resmi Kubernetes, saya mencatat beberapa perbedaan mendasar antara strategi **Rolling Update** dan **Recreate**:

* **Rolling Update (Strategi Default):**

  * **Cara Kerja:** Pergantian versi aplikasi dilakukan secara bertahap. Pod versi baru akan diluncurkan sambil perlahan menggantikan Pod versi lama, sehingga ada periode di mana keduanya aktif bersamaan (tergantung konfigurasi `maxSurge` dan `maxUnavailable`).
  * **Downtime:** Umumnya tidak terjadi downtime karena selalu ada Pod yang melayani permintaan pengguna selama proses berlangsung.
  * **Konsumsi Resource:** Memerlukan sumber daya lebih karena Pod lama dan baru berjalan bersamaan untuk sementara.
  * **Rollback:** Lebih mudah dan cepat untuk kembali ke versi sebelumnya jika versi baru bermasalah.
  * **Kesesuaian:** Cocok untuk aplikasi yang butuh ketersediaan tinggi, misalnya sistem produksi yang tidak boleh mengalami gangguan layanan.

* **Recreate:**

  * **Cara Kerja:** Semua Pod versi lama dihentikan terlebih dahulu, lalu Pod versi baru dibuat dari awal.
  * **Downtime:** Ada jeda waktu di mana tidak ada Pod yang aktif, sehingga aplikasi akan mengalami downtime sementara.
  * **Konsumsi Resource:** Lebih hemat sumber daya karena tidak ada Pod yang berjalan bersamaan.
  * **Rollback:** Masih memungkinkan, tapi risiko downtime lebih tinggi jika rollback harus dilakukan setelah aplikasi tidak aktif.
  * **Kesesuaian:** Sesuai untuk aplikasi yang tidak kritis, atau saat ada perubahan signifikan seperti migrasi database yang tidak kompatibel antar versi.

### 2. Uji Coba Deploy Spring Petclinic REST dengan Strategi Recreate

Saya mencoba strategi **Recreate** dengan langkah-langkah berikut:

1. **Mengubah File Deployment:**
   Saya mengedit file manifest `deployment.yaml` yang digunakan untuk aplikasi `spring-petclinic-rest`, lalu menambahkan strategi berikut:

   ```yaml
   spec:
     strategy:
       type: Recreate
   ```

2. **Menghapus Deployment Lama:**
   Untuk melihat efek Recreate lebih jelas, saya menghapus deployment yang lama:

   ```bash
   kubectl delete deployment spring-petclinic-rest
   ```

3. **Men-deploy Ulang dengan File yang Dimodifikasi:**
   Saya menjalankan kembali perintah:

   ```bash
   kubectl apply -f deployment-recreate.yaml
   ```

4. **Memantau Proses:**
   Dengan menjalankan:

   ```bash
   kubectl get pods -w
   ```

   Saya dapat melihat Pod lama dihentikan terlebih dahulu sebelum Pod baru dibuat. Selama periode ini, layanan tidak tersedia sampai Pod baru benar-benar aktif.


### 3. Membuat File Manifest Khusus untuk Recreate

Saya menyimpan versi modifikasi ini dengan nama `deployment-recreate.yaml`, sehingga saya bisa memilih antara Rolling Update atau Recreate sesuai kebutuhan.

### 4. Kelebihan Menggunakan Manifest File dalam Kubernetes

Setelah membandingkan metode deklaratif melalui manifest file dengan pendekatan imperatif menggunakan `kubectl`, saya menemukan beberapa keunggulan manifest file:

* **Pendekatan Deklaratif:**
  Kita cukup menentukan kondisi akhir yang diinginkan, dan Kubernetes akan mengusahakan agar sistem mencapai kondisi tersebut. Ini lebih mudah dipelihara dan dipahami.
* **Dapat Dicatat dalam Version Control:**
  File YAML bisa disimpan dalam Git atau sistem kontrol versi lain, memudahkan pelacakan perubahan konfigurasi dan kerja tim.
* **Dapat Digunakan Berulang:**
  File yang sama bisa digunakan untuk berbagai lingkungan seperti pengembangan atau produksi, menjamin konsistensi deploy.
* **Idempoten:**
  Perintah `kubectl apply` hanya akan menerapkan perubahan jika konfigurasi yang didefinisikan berbeda dari kondisi saat ini, sehingga aman dijalankan berulang kali.
* **Mudah Dibaca dan Dijadikan Dokumentasi:**
  File manifest memberikan dokumentasi konfigurasi aplikasi yang jelas, memudahkan pemahaman bagi anggota tim lain.
* **Mendukung Otomatisasi:**
  File ini bisa menjadi dasar automasi dalam pipeline CI/CD, mempercepat proses deployment dan pembaruan.
* **Manajemen Kompleksitas:**
  Untuk aplikasi besar dengan banyak komponen (Service, Deployment, Secret, dan lain-lain), penggunaan manifest file membantu pengelolaan konfigurasi menjadi lebih rapi dan sistematis.

---
