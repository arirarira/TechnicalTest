# Jawaban Technical Test - Quality Assurance
**Nama:** Ari Ramadhan  
**Posisi:** QA Engineer / Automation QA  

---

## Soal 1 – Fundamental QA dan Quality Judgment

### 1. Testing Scope Decision
Dalam menghadapi rilis yang ketat (7 hari) dengan keterbatasan resources (1 QA) dan kondisi environment staging yang tidak stabil, saya akan membagi ruang lingkup pengujian (testing scope) sebagai berikut:

*   **Fitur yang Akan Diuji Secara Mendalam (In-Depth Testing):**
    *   **Login & Logout:** Ini adalah gerbang utama aplikasi B2C. Jika alur login gagal, seluruh pengguna tidak dapat bertransaksi, yang akan langsung berdampak fatal pada bisnis. Logout juga penting untuk memastikan session management aman dan data pribadi pengguna terlindungi (terutama di perangkat publik).
    *   **Reset Password:** Ini adalah alur kritis keamanan (security-critical). Jika fungsi reset password bermasalah (misalnya token tidak kedaluwarsa atau verifikasi email gagal), ini akan mengakibatkan keluhan massal ke Customer Support (CS) dan membuka celah keamanan yang sangat berisiko tinggi (*account takeover*).
    *   *Skenario Pengujian:* Saya akan menguji happy path, negative path (input tidak valid, format salah, token kadaluarsa), session persistence, dan response error handling.
*   **Fitur yang Akan Diuji Secara Minimal (Minimal Testing / Smoke Testing):**
    *   **Edit Profile:** Sesuai dengan arahan Product Manager (PM) bahwa fitur utama harus berjalan dan bug kecil bisa menyusul. Edit Profile adalah fitur personalisasi sekunder yang tidak langsung menghentikan proses transaksi utama pengguna (core business).
    *   *Skenario Pengujian:* Pengujian hanya akan mencakup *happy path* dasar (mengubah data profil standar seperti nama, lalu memastikan data tersebut berhasil tersimpan di database dan tampil kembali di UI). Edge cases yang rumit (seperti upload file gambar dengan ukuran sangat besar, jenis format file aneh, atau handling karakter khusus/unicode ekstrem) akan ditunda.

### 2. Prioritization Logic (1 Hari Testing Efektif)
Jika saya hanya memiliki waktu 1 hari efektif untuk melakukan testing sebelum rilis, saya akan menggunakan pendekatan **Risk-Based Testing (RBT)** dengan urutan prioritas sebagai berikut:

1.  **Prioritas 1: Login & Session Handling (Happy Path + Validasi Dasar)**
    *   *Alasan:* Verifikasi awal agar pengguna bisa masuk ke dalam sistem. Tanpa ini, alur pengujian lainnya tidak bisa dijalankan.
2.  **Prioritas 2: Reset Password (Happy Path)**
    *   *Alasan:* Menjamin pengguna yang lupa password tetap memiliki akses mandiri untuk masuk kembali ke akun mereka tanpa membebani tim CS di hari pertama rilis.
3.  **Prioritas 3: Edit Profile (Happy Path)**
    *   *Alasan:* Melakukan verifikasi cepat bahwa input field dasar berfungsi dan data ter-update di database tanpa menghasilkan error server (5xx).
4.  **Prioritas 4: Smoke Test / Regression Test Cepat pada Fitur Transaksi Utama (Core Flow)**
    *   *Alasan:* Karena env dev/staging sering tidak sinkron dan regresi sering muncul, saya harus memastikan perubahan pada login/profile ini tidak merusak alur transaksi utama B2C yang sudah berjalan stabil sebelumnya (*regression safety net*).

### 3. Handling Conflict dengan Developer
Jika terjadi perbedaan pendapat di mana developer merasa suatu perilaku/bug adalah *expected behavior* sementara saya menilainya berdampak buruk bagi user (*bad UX*), langkah konkret yang saya lakukan adalah:

1.  **Kumpulkan Bukti Secara Objektif (Data-Driven):** Saya akan mendokumentasikan bug tersebut secara detail lengkap dengan screen recording, screenshot, log error console browser, serta langkah-langkah reproduksi yang jelas.
2.  **Jelaskan Menggunakan Perspektif Pengguna (UX Impact):** Saya akan menjelaskan dampak nyata terhadap pengguna akhir dan bisnis.  
    *Contoh:* "Jika tombol simpan tidak dinonaktifkan (*disabled*) setelah diklik sekali, user berpotensi melakukan klik berkali-kali (*double submit*), yang dapat menyebabkan transaksi ganda atau pengisian data duplikat di database."
3.  **Gunakan Referensi Standar / PRD:** Merujuk pada Product Requirement Document (PRD) awal atau prinsip kegunaan standar (*Usability Heuristics*).
4.  **Melibatkan Product Manager (PM) sebagai Penentu:** Jika tetap buntu, saya akan mendiskusikan temuan ini dalam forum bersama PM. Saya akan memaparkan risiko dari sisi kualitas produk, sementara PM yang akan mengambil keputusan akhir dari sisi prioritas bisnis dan produk.

### 4. Quality Risk Awareness
Dua risiko terbesar jika rilis tetap dilakukan pada kondisi ini adalah:

1.  **Risiko 1: Kebocoran Sesi Pengguna atau Celah Keamanan (Session Leak & Security Vulnerability)**
    *   *Deskripsi:* Akibat env tidak sinkron dan waktu pengujian terbatas, ada risiko token session tidak ter-invalidate dengan benar saat logout atau token reset password bisa ditebak/digunakan kembali.
    *   *Keputusan:* **Eskalasi.** Risiko keamanan data pribadi pengguna B2C adalah hal kritikal yang tidak dapat ditoleransi (*zero-tolerance issue*). Saya akan mengekskalasi hal ini kepada Tech Lead dan PM untuk menunda rilis hingga fungsi otentikasi benar-benar aman.
2.  **Risiko 2: Bug Kosmetik / Validasi Minor pada Fitur Edit Profile**
    *   *Deskripsi:* Misalnya tampilan foto profil yang sedikit bergeser di perangkat seluler tertentu, atau tidak adanya validasi format ukuran file foto profil yang ramah pengguna.
    *   *Keputusan:* **Diterima (Accept dengan Mitigasi).** Risiko ini dapat diterima untuk menjaga timeline rilis 7 hari. Mitigasinya adalah dengan mencatat semua isu minor tersebut di Jira sebagai backlog hotfix untuk segera diperbaiki setelah rilis utama selesai.

---

## Soal 2 – Automation UI dan Test Engineering Judgment

### Part A – Analisa Automation Existing
Tiga penyebab utama automation UI menjadi flaky berdasarkan pengalaman nyata di lapangan:

1.  **Waiting Strategy yang Tidak Tepat (Hardcoded Waits):**
    *   Menggunakan static wait seperti `cy.wait(3000)` atau thread sleep. Hal ini sering gagal di environment CI karena performa server/network runner CI cenderung berfluktuasi. Jika respons server sedang lambat melebihi batas waktu tunggu statis, pengujian akan langsung gagal secara acak.
2.  **Ketergantungan Data / State Pollution (Data Dependency):**
    *   Test case menggunakan data test yang sama berulang kali secara paralel tanpa adanya isolasi data, atau test case baru sangat bergantung pada state/output dari test case sebelumnya. Jika salah satu test case di awal gagal, test case berikutnya akan gagal secara beruntun (*cascade failure*).
3.  **Selector yang Tidak Stabil / Dinamis (Fragile Locators):**
    *   Menggunakan selektor CSS bawaan framework styling (seperti `.css-1y8md2` atau `.MuiButton-root`) atau XPath absolut yang sangat panjang. Ketika front-end developer mengubah tata letak minor atau melakukan rebuild library UI, selektor tersebut akan langsung patah.

*Penyebab yang paling sering saya temui:*  
**Waiting Strategy dan Data Dependency.** Di environment CI/CD, resource machine (CPU/RAM) sering kali mengalami *throttling*, sehingga rendering UI dan response API berjalan jauh lebih lambat daripada saat dijalankan secara local. Jika test script tidak menggunakan penantian berbasis kondisi (misalnya menunggu intercept API selesai atau menunggu elemen memiliki atribut tertentu), test akan menjadi sangat flaky di CI. Masalah ini diperparah jika tidak ada skema isolasi data test (seperti database seeding per test suite atau penggunaan dynamic data generator).

---

### Part B – Design Automation Test (Opsi 2 – Real Automation Code)
Mengikuti standar dan gaya penulisan (*code style*) dari project yang pernah saya buat, di bawah ini adalah implementasi Automation UI menggunakan **Cypress** dengan arsitektur **Page Object Model (POM)** yang bersih dan maintainable.

#### 1. Page Object Class: `LocationPage.js` (Memperluas `BasePage` dengan `qase.step`)
File ini diletakkan di `cypress/pages/LocationPage.js`.

```javascript
import BasePage from "./BasePage";
import { qase } from "cypress-qase-reporter/mocha";

class LocationPage extends BasePage {
  // Definisi selector yang bersih menggunakan atribut name/test-id unik
  static inputLocationName = '[name="locationName"]';
  static inputLocationCode = '[name="locationCode"]';
  static dropdownLocationType = '#rhf-autocomplete-locationType';
  static switchActive = '#isActive-switch';
  static btnSave = 'button[type="submit"]';
  static toastMessage = '.MuiAlert-root .MuiAlert-message';
  
  // Method pembantu untuk membuat data dinamis yang unik
  generateLocationCode() {
    const randomNum = Math.floor(1000 + Math.random() * 9000);
    return `LOC-${randomNum}`;
  }

  enterLocationName(name) {
    qase.step("Input Nama Lokasi ke form", () => {
      super.typeText(LocationPage.inputLocationName, name);
    });
  }

  enterLocationCode(code) {
    qase.step("Input Kode Lokasi ke form", () => {
      super.typeText(LocationPage.inputLocationCode, code);
    });
  }

  selectLocationType(type) {
    qase.step(`Pilih Tipe Lokasi dari dropdown: ${type}`, () => {
      // Meniru style Material UI Autocomplete yang interaktif di parkee-cypress
      super.clickElement(LocationPage.dropdownLocationType);
      cy.get('.MuiAutocomplete-option')
        .contains(type)
        .should('be.visible')
        .click();
    });
  }

  toggleActiveSwitch(status) {
    qase.step(`Atur status keaktifan lokasi menjadi: ${status}`, () => {
      cy.get(LocationPage.switchActive).then(($el) => {
        const isChecked = $el.attr('value') === 'true' || $el.hasClass('Mui-checked');
        if (isChecked !== status) {
          super.clickElement(LocationPage.switchActive);
        }
      });
    });
  }

  clickSave() {
    qase.step("Klik tombol Simpan untuk mengirim form", () => {
      super.clickElement(LocationPage.btnSave);
    });
  }

  verifySuccessToast(expectedMessage) {
    qase.step(`Verifikasi Toast Message menampilkan teks: "${expectedMessage}"`, () => {
      super.shouldBeVisible(LocationPage.toastMessage);
      super.shouldHaveText(LocationPage.toastMessage, expectedMessage);
    });
  }
}

export default new LocationPage();
```

#### 2. Test Specification File: `createLocation.cy.js`
File ini diletakkan di `cypress/e2e/web_backoffice/location/createLocation.cy.js`.

```javascript
import { qase } from "cypress-qase-reporter/mocha";
import LocationPage from "../../../pages/LocationPage";

describe("Submit Form - Pendaftaran Lokasi Baru", () => {
  // Menggunakan URL imajiner admin console
  const urlPage = "https://imagine-web-admin.com/location/create";
  let locationData;

  before(() => {
    // Membaca test data fixture untuk kebutuhan form input
    cy.fixture("locationData").then((data) => {
      locationData = data;
    });
  });

  beforeEach(() => {
    // Menggunakan custom commands standard repo parkee-cypress untuk reset session & login
    cy.resetSession();
    cy.login();
    cy.visit(urlPage);

    // Waiting Strategy berbasis Network: Intercept API call sebelum interaksi UI dilakukan
    cy.intercept("POST", "**/api/v1/locations").as("createLocationRequest");
  });

  it("User berhasil submit form pembuatan lokasi baru dengan data valid", { tags: "Positive" }, () => {
    const uniqueCode = LocationPage.generateLocationCode();
    const locationName = `Lokasi Test Candi - ${uniqueCode}`;

    qase.step("Mengisi semua field wajib pada form lokasi", () => {
      LocationPage.enterLocationName(locationName);
      LocationPage.enterLocationCode(uniqueCode);
      LocationPage.selectLocationType(locationData.validPayload.type);
      LocationPage.toggleActiveSwitch(true);
    });

    qase.step("Submit form dan menunggu respons sukses dari server", () => {
      LocationPage.clickSave();

      // Conditional Wait: Menunggu response API selesai tanpa hardcoded sleep
      cy.wait("@createLocationRequest").then((interception) => {
        expect(interception.response.statusCode).to.eq(201);
        expect(interception.response.body.data.locationCode).to.eq(uniqueCode);
      });
    });

    qase.step("Verifikasi feedback berhasil pada halaman UI", () => {
      LocationPage.verifySuccessToast("Lokasi baru berhasil disimpan!");
      
      // Memastikan sistem melakukan redirect ke halaman list lokasi
      cy.url().should("include", "/location/list");
    });
  });

  it("User gagal submit form lokasi karena field nama lokasi kosong", { tags: "Negative" }, () => {
    qase.step("Mengisi field form kecuali Nama Lokasi", () => {
      const uniqueCode = LocationPage.generateLocationCode();
      LocationPage.enterLocationCode(uniqueCode);
      LocationPage.selectLocationType(locationData.validPayload.type);
    });

    qase.step("Mencoba melakukan submit form", () => {
      LocationPage.clickSave();
    });

    qase.step("Verifikasi validasi error muncul di UI dan request backend tidak terkirim", () => {
      // Validasi error message di UI input field
      cy.get('[name="locationName"]-helper-text')
        .should("be.visible")
        .should("contain.text", "Nama lokasi wajib diisi");

      // Memastikan HTTP POST request tidak terpicu (Client-side validation sukses memblokir)
      cy.get("@createLocationRequest.all").should("have.length", 0);
    });
  });
});
```

---

### Part C – Failure Investigation
Langkah-langkah investigasi sistematis saat test case gagal di CI tetapi selalu lulus di local:

1.  **Analisa Log Kegagalan CI & Media Artifacts:**
    *   Unduh video recording (`webp`/`mp4`) dan screenshot kegagalan test dari build pipeline CI. Ini membantu mendeteksi isu visual seperti modal tak terduga, loading overlay yang lambat hilang, atau *layout shifting*.
    *   Periksa log kegagalan konsol CI secara detail untuk melihat di step mana asersi mulai gagal, beserta pesan error aslinya (misal: `TimeoutError` menunggu elemen terlihat, atau respons API `504 Gateway Timeout`).
2.  **Cek Perbedaan Lingkungan (Environment Matrix):**
    *   Periksa spesifikasi hardware runner CI. Resource yang terbatas sering memperlambat pemrosesan web.
    *   Bandingkan data test. Pastikan data test yang digunakan di target environment CI tidak berbenturan atau dibersihkan oleh test case lain yang berjalan paralel.
    *   Bandingkan latency network dan performa API backend di staging env (tempat CI berjalan) dengan local environment.
3.  **Simulasi Kondisi CI di Local:**
    *   Jalankan test secara headless di local (misal: `cypress run --browser chrome`) karena terkadang headless rendering memiliki perilaku render engine berbeda dengan browser GUI.
    *   Aktifkan *network throttling* (misal ke profil Fast 3G) di Chrome DevTools untuk menirukan lambatnya koneksi di server CI.
    *   Jalankan test yang bermasalah berulang kali (looping 10-20 kali) di local untuk melihat apakah flakiness muncul di local.
4.  **Komunikasi dengan Developer:**
    *   Jika ditemukan kegagalan karena response backend lambat (timeout) atau perbedaan skema DB/API di environment staging CI, diskusikan langsung dengan Developer Backend.
    *   *Contoh komunikasi:* "Halo, test `createLocation` gagal di CI karena API POST `/api/v1/locations` merespons lebih dari 10 detik (HTTP 504) di server staging, sedangkan di local aman. Apakah ada perubahan query atau performa database staging sedang penuh?"
5.  **Kriteria Keputusan Tindak Lanjut (Decision Criteria):**
    *   **Fix (Perbaiki):** Jika kegagalan karena waiting strategy test script yang kurang asertif (misal butuh wait intercept API) atau selektor DOM yang rapuh. Test segera diperbaiki di sprint berjalan.
    *   **Disable / Quarantine (Karantina):** Jika bug memang nyata berasal dari aplikasi backend/frontend yang performanya lambat di staging, namun perbaikannya membutuhkan waktu lama oleh developer. Pengujian akan di-skip sementara (`it.skip`) agar tidak memblokir kelancaran CI pipeline tim, dan dibuatkan tiket Jira perbaikan yang di-link ke test case tersebut.
    *   **Delete (Hapus):** Jika fungsionalitas/fitur yang diuji sudah dihapus dari aplikasi (*deprecated*), atau test case tersebut redundant dengan test case lain yang lebih efisien dan andal.
