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