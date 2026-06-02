+++
title = 'Auto Blog Demo'
date = '2026-06-02T00:00:00Z'
draft = false
tags = ['demo','auto-blog']
cover = '/images/demo-cover.jpg'
+++

## Ringkasan

Ini adalah contoh blog post yang di-generate otomatis untuk memverifikasi pipeline YouTube → Markdown di blog kamu. File ini sudah siap di-repo lokal dan tinggal dipush ke GitHub lalu部署 via Netlify.

## Apa yang terjadi setelah kamu push

1. Kamu push repo  ke GitHub
2. Connect repo ke Netlify
3. Setiap kali kamu kirim link YouTube, agent akan:
   - fetch transkrip
   - generate blog post Markdown
   - simpan di 
   - bisa langsung commit + push

## Langkah berikutnya

Kirim satu link YouTube apapun, nanti gue jalankan alur penuh: transkrip → blog post → gambar → simpan → preview.
