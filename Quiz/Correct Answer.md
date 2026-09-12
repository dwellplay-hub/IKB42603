**1. Which deployment model combines private and public cloud?**

* **Jawapan Betul:** Hybrid Cloud


* **Penjelasan:** Model ini menggabungkan infrastruktur awan persendirian (*private*) dan awan awam (*public*) untuk fleksibiliti serta keselamatan data.

---

**2. If an access key is compromised, what should be done first?**

* **Jawapan Betul:** Deactivate or rotate the key


* **Penjelasan:** Menyahaktifkan atau menukar (*rotate*) kunci akses dengan serta-merta adalah tindakan paling penting bagi menghalang pencerobohan lanjut.

---

**3. Which deployment model provides the MOST control?**

* **Jawapan Betul:** Private Cloud


* **Penjelasan:** Awan persendirian diuruskan khas untuk satu organisasi sahaja, memberikan kawalan fizikal, seni bina, dan keselamatan yang paling menyeluruh.

---

**4. LocalStack is used because it:**

* **Jawapan Betul:** Emulates AWS cloud services locally (Simulates AWS environment)


* **Penjelasan:** LocalStack membolehkan kita menguji servis awan AWS seperti S3, IAM, dan KMS secara setempat di dalam komputer tanpa kos akaun sebenar.



---

**5. Which security principle gives users only the permissions required to perform their tasks?**

* **Jawapan Betul:** Principle of Least Privilege


* **Penjelasan:** Prinsip keselamatan ini memastikan pengguna atau entiti hanya diberikan kebenaran minimum yang diperlukan untuk menyelesaikan tugasan mereka.



---

**6. Which characteristic allows cloud resources to automatically grow or shrink?**

* **Jawapan Betul:** Rapid Elasticity


* **Penjelasan:** Keupayaan sumber awan untuk membesar (*scale up/out*) atau mengecut (*scale in/down*) secara dinamik mengikut beban kerja dipanggil *Rapid Elasticity*.

---

**7. A Kubernetes cluster consists of:**

* **Jawapan Betul:** Multiple nodes (Control Plane and Worker Nodes)


* **Penjelasan:** Kluster Kubernetes mengandungi nod utama (*control plane*) dan beberapa nod pekerja (*worker nodes*) yang menjalankan aplikasi.



---

**8. Which AWS identity has unlimited privileges?**

* **Jawapan Betul:** Root User


* **Penjelasan:** Pengguna utama (*root user*) yang dicipta semasa akaun AWS dibuka mempunyai akses penuh tanpa had kepada setiap sumber dan servis.



---

**9. Which command lists Kubernetes nodes?**

* **Jawapan Betul:** `kubectl get nodes`

* **Penjelasan:** Perintah rasmi Kubernetes CLI untuk melihat status semua nod yang ada di dalam kluster.



---

**10. Which service model requires customers to manage the operating system?**

* **Jawapan Betul:** IaaS (Infrastructure as a Service)


* **Penjelasan:** Di bawah model IaaS, pembekal awan menyediakan perkakasan dan rangkaian, manakala pengguna bertanggungjawab memasang, mengkonfigurasi, dan menampal sistem operasi (OS).

---

**11. Which IAM identity is normally used as a temporary identity?**

* **Jawapan Betul:** IAM Role


* **Penjelasan:** *IAM Role* digunakan untuk memberikan kebenaran dengan kredensial keselamatan sementara (*temporary security credentials*), bukannya identiti kekal seperti *IAM User*.



---

**12. Docker is mainly used to:**

* **Jawapan Betul:** Run containers


* **Penjelasan:** Docker ialah platform yang direka khas untuk mencipta, mengedarkan, dan menjalankan aplikasi di dalam kontena yang terpencil.



---

**13. What does ARN stand for?**

* **Jawapan Betul:** Amazon Resource Name


* **Penjelasan:** Format pengenal pasti piawai yang digunakan untuk merujuk mana-mana sumber AWS secara unik di seluruh dunia.



---

**14. Which endpoint is commonly used with LocalStack?**

* **Jawapan Betul:** `http://localhost:4566`

* **Penjelasan:** Port lalai (port 4566) yang bertindak sebagai gerbang API tunggal bagi semua servis yang diemulasikan oleh LocalStack.



---

**15. Which account should never have access keys created for routine use?**

* **Jawapan Betul:** Root User


* **Penjelasan:** Akaun *root* tidak sepatutnya mempunyai *access keys* aktif untuk tugasan biasa kerana tahap risiko keselamatan yang terlalu tinggi sekiranya terdedah.



---

**16. Which is NOT an essential characteristic of cloud computing?**

* **Jawapan Betul:** Manual Provisioning


* **Penjelasan:** Ciri asas awan mengikut takrifan NIST ialah *On-demand Self-service* (automatik), bukannya penyediaan berasaskan manual.



---

**17. Google Docs is an example of:**

* **Jawapan Betul:** SaaS (Software as a Service)


* **Penjelasan:** Aplikasi siap pakai berasaskan web yang boleh digunakan terus oleh pengguna akhir tanpa perlu menguruskan infrastruktur atau kod di belakang tabir.

---

**18. Which service model provides virtual machines?**

* **Jawapan Betul:** IaaS (Infrastructure as a Service)


* **Penjelasan:** IaaS membekalkan sumber asas seperti mesin maya (*virtual machines*, cth. AWS EC2), storan, dan rangkaian kepada pengguna.



---

**19. Which IAM component contains permissions?**

* **Jawapan Betul:** IAM Policy


* **Penjelasan:** Dokumen berformat JSON yang mentakrifkan kebenaran khusus (*Effect, Action, Resource*) untuk membenarkan atau menyekat operasi.



---

**20. For easier permission management, policies should preferably be attached to:**

* **Jawapan Betul:** IAM Groups


* **Penjelasan:** Melekatkan dasar pada kumpulan (*group*) membolehkan pengurusan hak akses secara berpusat mengikut peranan atau jabatan.



---

**21. In the ARN `arn:aws:s3:::my-bucket`, which component represents the AWS service?**

* **Jawapan Betul:** `s3`

* **Penjelasan:** Berdasarkan sintaks `arn:partition:service:region:account-id:resource`, bahagian ketiga (`s3`) merujuk secara terus kepada jenis servis AWS.



---

**22. A node is:**

* **Jawapan Betul:** A worker machine (virtual or physical)


* **Penjelasan:** Mesin fizikal atau mesin maya di dalam kluster Kubernetes yang bertanggungjawab menjalankan beban kerja kontena (*Pods*).



---

**23. Which AWS-managed policy provides full administrative access?**

* **Jawapan Betul:** AdministratorAccess


* **Penjelasan:** Polisi terurus yang memberi kebenaran tanpa had ke atas setiap servis dan tindakan di dalam akaun AWS.



---

**24. Which tool creates a local Kubernetes cluster?**

* **Jawapan Betul:** kind (Kubernetes in Docker)


* **Penjelasan:** Alat pembangunan ringan yang membolehkan kluster Kubernetes dijalankan secara setempat di dalam kontena Docker.



---

**25. Access keys are mainly used for:**

* **Jawapan Betul:** Programmatic access / AWS CLI and SDK authentication


* **Penjelasan:** Digunakan untuk mengesahkan identiti panggilan API melalui persekitaran baris arahan (CLI) atau kod perisian (SDK).



---

**26. Which ARN component identifies the AWS account that owns the resource?**

* **Jawapan Betul:** Account ID


* **Penjelasan:** Nombor ID 12-digit dalam format ARN (`arn:aws:service:region:account-id:resource`) menandakan akaun pemilik sumber berkenaan.

---

**27. Cloud computing refers to:**

* **Jawapan Betul:** Delivering computing resources over the Internet


* **Penjelasan:** Penghantaran perkhidmatan pengkomputeran—termasuk pelayan, storan, pangkalan data, dan perisian—melalui sambungan internet.



---

**28. The smallest deployable unit in Kubernetes is:**

* **Jawapan Betul:** Pod


* **Penjelasan:** Pod ialah unit pelaksanaan asas terkecil dalam Kubernetes yang menempatkan satu atau beberapa kontena yang berkongsi storan dan rangkaian.



---

**29. A collection of IAM users is called:**

* **Jawapan Betul:** IAM Group


* **Penjelasan:** Entiti pengurusan yang menggabungkan beberapa *IAM User* di bawah satu kumpulan dengan kebenaran polisi yang sama.



---

**30. Which AWS CLI command verifies the current identity?**

* **Jawapan Betul:** `aws sts get-caller-identity`

* **Penjelasan:** Perintah CLI AWS Security Token Service (STS) untuk melihat butiran User ID, Account ID, dan ARN bagi kredensial yang sedang aktif.