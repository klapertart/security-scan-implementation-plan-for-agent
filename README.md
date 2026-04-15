# Security Scan Implementation Plan — Spring Boot

## Context

- OS: Windows PowerShell
- Service: Spring Boot (Maven)
- Scan order: Gitleaks → SonarQube → Trivy
- Executor: AI model (Gemini Flash atau sejenisnya)

---

## Prerequisites

Pastikan tools berikut sudah terinstall sebelum menjalankan plan ini:

- Git
- Java 17+
- Maven 3.6+
- Docker (untuk SonarQube)
- Gitleaks (via `winget install gitleaks`)
- Trivy (via `winget install AquaSecurity.Trivy`)

---

## Variables

Ganti semua nilai di bawah ini sesuai kebutuhan sebelum memberikan plan ini ke AI:

```
PROJECT_DIR     = C:\path\to\your\project
SONAR_URL       = http://localhost:9001
SONAR_LOGIN     = your_sonarqube_token
SONAR_PROJECT_KEY  = your-project-key
SONAR_PROJECT_NAME = Your Project Name
REPORT_DIR      = C:\path\to\report\output
```

---

## Step 1 — Gitleaks Scan

**Tujuan:** Memastikan tidak ada credential, API key, atau secret yang bocor di repository.

**Jalankan perintah berikut di PowerShell dari dalam folder project:**

```powershell
gitleaks detect --source . -v `
  --report-format json `
  --report-path $REPORT_DIR\gitleaks-report.json
```

**Kriteria lanjut ke step berikutnya:**
- Perintah selesai tanpa error
- Output menunjukkan `leaks found: 0`
- File `gitleaks-report.json` terbuat di `$REPORT_DIR`

**Jika ada temuan:**
- Baca file `gitleaks-report.json`
- Identifikasi file dan baris yang bermasalah
- Hapus atau pindahkan credential ke environment variable
- Jalankan ulang scan sampai tidak ada temuan
- Jangan lanjut ke step berikutnya sebelum bersih

---

## Step 2 — SonarQube Scan

**Tujuan:** Analisis kualitas kode dan keamanan statis (SAST) — mendeteksi bug, code smell, dan celah keamanan di source code.

### Step 2a — Pastikan SonarQube Berjalan

```powershell
curl http://localhost:9001/api/system/status
```

Response yang diharapkan:
```json
{"status":"UP"}
```

Jika SonarQube belum berjalan, jalankan docker compose terlebih dahulu:

```powershell
cd C:\path\to\sonarqube
docker compose up -d
```

Tunggu 60 detik lalu cek kembali status.

### Step 2b — Jalankan Scan

```powershell
cd $PROJECT_DIR

mvn clean verify sonar:sonar `
  "-Dsonar.host.url=http://localhost:9001" `
  "-Dsonar.projectKey=your-project-key" `
  "-Dsonar.projectName=Your Project Name" `
  "-Dsonar.login=your_sonarqube_token"
```

**Kriteria lanjut ke step berikutnya:**
- Output Maven menunjukkan `BUILD SUCCESS`
- Output menunjukkan `ANALYSIS SUCCESSFUL`
- Dashboard SonarQube bisa diakses di `http://localhost:9001/dashboard?id=your-project-key`

### Step 2c — Export Hasil Scan

```powershell
$SONAR_URL = "http://localhost:9001"
$SONAR_LOGIN = "your_sonarqube_token"
$PROJECT_KEY = "your-project-key"

curl -s `
  -u "${SONAR_LOGIN}:" `
  "${SONAR_URL}/api/issues/search?componentKeys=${PROJECT_KEY}&ps=500" `
  -o $REPORT_DIR\sonar-report.json

Write-Host "SonarQube report saved to $REPORT_DIR\sonar-report.json"
```

**Jika ada temuan BLOCKER atau CRITICAL:**
- Baca file `sonar-report.json`
- Identifikasi isu berdasarkan severity
- Perbaiki isu BLOCKER terlebih dahulu, lalu CRITICAL
- Jalankan ulang scan untuk verifikasi

---

## Step 3 — Trivy Scan

**Tujuan:** Mendeteksi celah keamanan (CVE) pada dependency/library yang digunakan di `pom.xml`.

### Step 3a — Jalankan Scan

```powershell
cd $PROJECT_DIR

trivy fs . `
  --format json `
  --output $REPORT_DIR\trivy-report.json
```

### Step 3b — Tampilkan Ringkasan di Terminal

```powershell
trivy fs . --severity CRITICAL,HIGH
```

**Kriteria selesai:**
- Perintah selesai tanpa error
- File `trivy-report.json` terbuat di `$REPORT_DIR`

**Jika ada temuan CRITICAL atau HIGH:**
- Baca kolom `Fixed Version` di output
- Update versi library di `pom.xml` ke versi yang sudah aman
- Jalankan ulang scan untuk verifikasi
- Jika `Fixed Version` kosong, catat sebagai known issue dan pantau terus

---

## Output Files

Setelah semua step selesai, folder `$REPORT_DIR` akan berisi:

```
report-output/
├── gitleaks-report.json    ← hasil scan secret
├── sonar-report.json       ← hasil scan SAST
└── trivy-report.json       ← hasil scan dependency CVE
```

---

## Completion Criteria

Semua step dianggap selesai jika:

- [ ] Gitleaks: `leaks found: 0`
- [ ] SonarQube: `BUILD SUCCESS` dan tidak ada isu BLOCKER
- [ ] Trivy: tidak ada temuan CRITICAL

---

## Notes for AI Executor

- Jalankan setiap step secara berurutan. Jangan lanjut ke step berikutnya jika step sebelumnya gagal.
- Jika ada perintah yang gagal, tampilkan pesan error lengkapnya dan hentikan eksekusi.
- Semua perintah dijalankan di Windows PowerShell.
- Variabel `$REPORT_DIR` harus sudah ada sebagai folder sebelum menjalankan perintah. Buat folder jika belum ada: `New-Item -ItemType Directory -Force -Path $REPORT_DIR`
- Jangan modifikasi source code tanpa konfirmasi dari developer.
- Laporkan ringkasan hasil setiap step sebelum melanjutkan ke step berikutnya.
