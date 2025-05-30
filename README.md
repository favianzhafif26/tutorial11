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

