# Reflection on Hello Minikube

## 1. Compare the application logs before and after you exposed it as a Service. Try to open the app several times while the proxy into the Service is running. What do you see in the logs? Does the number of logs increase each time you open the app?

Sebelum mengekspos aplikasi `hello-node` sebagai Service, saya menyadari bahwa log dari pod-nya hanya menampilkan pesan inisialisasi dari container `agnhost`, seperti konfirmasi bahwa server `netexec` telah aktif dan siap menerima koneksi pada port yang ditentukan. Aktivitasnya terlihat sangat minim karena belum ada interaksi eksternal. Namun, situasi berubah sepenuhnya setelah saya menjalankan perintah:  
```bash
kubectl expose deployment hello-node --type=LoadBalancer --port=8080
```  
dan mengakses aplikasi beberapa kali melalui:  
```bash
minikube service hello-node
```  
Setiap kali saya membuka atau me-refresh aplikasi di browser, log pod langsung mencatat permintaan HTTP baru yang masuk. Jelas terlihat bahwa jumlah baris log meningkat signifikan seiring interaksi yang saya lakukan. Hal ini membuktikan bahwa Service berfungsi dengan baik dan berhasil meneruskan permintaan eksternal ke pod yang berjalan.  

## 2. Notice that there are two versions of `kubectl get` invocation during this tutorial section. The first does not have any option, while the latter has `-n` option with value set to `kube-system`. What is the purpose of the `-n` option and why did the output not list the pods/services that you explicitly created?

Dari pengalaman saya, opsi `-n` dalam perintah `kubectl get` berfungsi untuk menentukan namespace target tempat Kubernetes harus mencari sumber daya. Seperti folder dalam sebuah sistem file, namespace bertindak sebagai pengelompok logis sumber daya di dalam cluster.  
 
Ketika saya menjalankan perintah seperti:  
```bash
kubectl get deployments,pods
```  
tanpa opsi `-n`, Kubernetes secara otomatis menargetkan namespace `default`. Inilah mengapa deployment dan pod `hello-node` saya muncul karena saya tidak menyertakan namespace khusus saat membuatnya, sehingga Kubernetes menempatkannya di namespace default.  

Saat saya mencoba:  
```bash
kubectl get pods,services -n kube-system
```  
sumber daya `hello-node` tidak muncul dalam hasil. Ini karena namespace `kube-system` dikhususkan untuk komponen inti Kubernetes (seperti `coredns`, `metrics-server`, atau `kube-proxy`), bukan untuk aplikasi yang di-deploy pengguna.  

### Key Takeaways  
1. Namespace = Batas Logis: Memisahkan sumber daya aplikasi pengguna (`default`) dari infrastruktur sistem (`kube-system`).  
2. Opsi `-n` sebagai Penunjuk Target: Tanpa opsi ini, `kubectl` selalu beroperasi di namespace `default`.  
3. Isolasi yang Jelas: Sumber daya di namespace berbeda tidak saling terlihat kecuali jika diakses secara eksplisit.  

---

# Reflection on Rolling Update & Kubernetes Manifest File

## 1. What is the difference between Rolling Update and Recreate deployment strategy?

### Rolling Update (Default Strategy)
- Cara Kerja:  
  - Memperbarui Pod secara bertahap (satu per satu).  
  - Pod versi baru (*new replica*) dibuat terlebih dahulu sebelum Pod lama (old replica) diterminate.  
- Keuntungan:  
  - Zero Downtime: Aplikasi tetap tersedia selama proses update karena selalu ada Pod yang berjalan.  
  - Graceful Transition: Cocok untuk aplikasi yang harus highly available (misal: layanan produksi).  
- Contoh Use Case:  
  - Update versi minor/major aplikasi web yang mendukung backwards compatibility.  

### Recreate Strategy 
- Cara Kerja:  
  - Menghapus semua Pod lama terlebih dahulu, baru kemudian membuat Pod versi baru.  
- Risiko:  
  - Downtime: Aplikasi tidak tersedia sementara hingga Pod baru siap.  
- Kapan Digunakan?  
  - Jika versi baru tidak kompatibel untuk berjalan bersamaan dengan versi lama (misal: perubahan skema database).  
  - Untuk aplikasi non-kritis yang bisa toleransi interruption (misal: batch job).  



### Contoh Konfigurasi di Manifest YAML  
```yaml
spec:
  strategy:
    type: RollingUpdate  # atau Recreate
    rollingUpdate:
      maxSurge: 1        # Jumlah Pod tambahan saat update
      maxUnavailable: 0  # Jumlah Pod yang boleh tidak tersedia
```

## 2. Try deploying the Spring Petclinic REST using Recreate deployment strategy and document your attempt.

### 1. Modifikasi Deployment Manifest
```yaml
spec:
  strategy:
    type: Recreate  # Strategi diubah dari RollingUpdate ke 
```
- Tujuan: Mengganti strategi default (`RollingUpdate`) dengan `Recreate` untuk memaksa Kubernetes menghapus semua Pod lama sebelum membuat yang baru.
- Catatan: Jika tidak ada perubahan image/konfigurasi, gunakan `kubectl rollout restart` untuk memicu recreate tanpa modifikasi YAML.

### 2. Membersihkan Deployment Lama (Opsional)
```bash
kubectl delete deployment spring-petclinic-rest
kubectl get pods --watch  # Verifikasi semua Pod terkait sudah terminated
```
- Alasan: Memastikan lingkungan bersih sebelum uji coba, terutama jika sebelumnya menggunakan `RollingUpdate`.
- Opsional: Langkah ini bisa dilewati jika ingin langsung mengupdate deployment yang sedang berjalan.

### 3. Menerapkan Perubahan
```bash
kubectl apply -f deployment-recreate.yaml
```

### 4. Observasi Perilaku Recreate
```bash
kubectl get pods -w  # Pantau real-time
```
Output yang Diharapkan:
1. Semua Pod lama masuk status `Terminating` (jika ada).
2. Downtime Period: Tidak ada Pod yang berjalan (`Running`) sampai proses termination selesai.
3. Pod baru mulai dibuat (`ContainerCreating` → `Running`).

### 5. Dokumentasi

![alt text](<Screenshot 2025-05-30 181652.png>)


## 3. Prepare different manifest files for executing Recreate deployment strategy

Sudah saya masukkan file duplikat dari `deployment.yml` yaitu `deployment-recreate.yml` yang diubah strategy-nya menjadi `Recreate`. 

## 4. What do you think are the benefits of using Kubernetes manifest files? Recall your experience in deploying the app manually and compare it to your experience when deploying the same app by applying the manifest files (i.e., invoking `kubectl apply -f` command) to the cluster.


### 1. Pendekatan Deklaratif  
- Cara Kerja:  
  ```yaml
  # deployment.yaml
  apiVersion: apps/v1
  kind: Deployment
  spec:
    replicas: 4  # "Saya ingin 4 pod"
  ```  
- Manfaat:  
  - Fokus pada "what" (keadaan akhir) bukan "how" (langkah manual).  
  - Kubernetes secara otomatis melakukan reconciliation untuk mencapai desired state.  

### 2. Integrasi dengan Version Control  
- Contoh Workflow Git:  
  ```bash
  git add deployment.yaml
  git commit -m "Add production-ready deployment config"
  git tag v1.0.0
  ```  
- Keuntungan:  
  - Audit trail perubahan konfigurasi.  
  - Rollback mudah dengan `git checkout <old-commit> && kubectl apply -f`.  

### 3. Konsistensi Lintas Lingkungan  
- Praktik Ideal:  
  - File base (e.g., `k8s/base/deployment.yaml`) yang sama di-reuse untuk:  
    ```bash
    kubectl apply -f ./k8s/base -n dev
    kubectl apply -f ./k8s/base -n prod --dry-run=client
    ```  

### 4. Sifat Idempoten  
- Illustrasi:  
  ```bash
  kubectl apply -f deployment.yaml  # First run: Creates resources
  kubectl apply -f deployment.yaml  # Second run: No changes if YAML unchanged
  ```  

### 5. Dokumentasi Otomatis  
- Contoh Readability:  
  ```yaml
  env:
    - name: DB_URL
      valueFrom: 
        configMapKeyRef:  # Jelas dependency ke ConfigMap
          name: app-config
  ```  

### 6. Dasar Otomatisasi CI/CD  
- Integrasi Pipeline:  
  ```yaml
  # .gitlab-ci.yml
  deploy:
    stage: deploy
    script:
      - kubectl apply -f k8s/ --namespace=$ENV
  ```  

### 7. Manajemen Kompleksitas  
- Solusi Organisasi:  
  - Kustomize: Override environment-specific values.  
  - Helm Charts: Package multi-resource dengan templating.