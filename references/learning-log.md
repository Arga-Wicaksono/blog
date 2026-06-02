
### zeroday-cloud-pgp-poc-2026
- **Date**: 2026-06-02
- **Source**: https://youtu.be/xmhxPZvUtXU
- **Topic tags**: postgresql, pgcrypto, pwn, heap, arbitrary-write, rce
- **What I learned**:
  1. `pgp_sym_decrypt` bisa melanggar database encoding karena percaya format byte PGP tanpa verifikasi ulang setelah dekripsi.
  2. `pg_utf_mblen` yang hanya membaca byte pertama dapat dipaksa melaporkan panjang UTF-8 lebih besar sehingga melompat buffer.
  3. Dari *dirty string* bisa laju ke arbitrary write, lalu RCE via manipulasi free pointer pada AllocSet allocator.
- **Pitfalls / edge cases**:
  - Splitting post-auth capability terlalu dini ("no CREATE") bisa terlewatkan meski masih bisa memuat ekstensi trusted.
  - Patch tidak perlu memperketat allocator performansi karena dampak too high; lebih efektif menambahkan validasi encoding setelah dekripsi.
- **Reusable pattern / command / snippet**:
  ```bash
  # Audit trusted extensions surface
  psql -c "SELECT * FROM pg_available_extensions WHERE trusted;"
  ```
- **Confidence**: medium

### dark-web-ai-osint-pattern-2026
- **Date**: 2026-06-02
- **Source**: https://youtu.be/_KzObeom88Y
- **Topic tags**: dark-web, osint, threat-research, tor, ai-tooling
- **What I learned**:
  1. Sinyal threat intel valid di dark web jarang dan sangat berisiko tertutup oleh derau besar.
  2. Pendekatan yang lebih stabil adalah memakai AI sebagai selektor dan perencana, bukan sebagai akses awal.
  3. Guardrail operasional dan persona ethics harus ditentukan sebelum akses, bukan setelah insiden.
- **Pitfalls / edge cases**:
  - Kontinuitas tor circuit rapuh sulit dijadikan batch automation tanpa circuit break.
  - Kepercayaan komunitas threat actor butuh minggu, tidak saat pertama.
- **Reusable pattern / command / snippet**:
  ```text
  Researcher stack: VPN + Tor + isolasi persona + AI triage + Markdown notes
  ```
- **Confidence**: medium
