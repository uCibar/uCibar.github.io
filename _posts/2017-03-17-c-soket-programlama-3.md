---
title: Haydi Squash Oynalayım (C Soket Programlama - 3)
layout: post
summary: C ile chat uygulaması. Soketler, multithreading, client, server... Bir MSN
  Messenger kolay yetişmiyor.
categories: c-lang
---

Peki **echo server** nedir? Öncelikle terimi türkçeye çevirelim; **yankı sunucusu.** Yani, bir sunucumuz var ve amacı yankı yapmak. Ama neyin yankısı? Tabi ki istemciden gönderdiğimiz mesajın yankısı.

Yazıya başlarken aklıma gelen muhteşem(!) analojimden de bahsedeyim, zira yazının başlığını da bu analoji oluşturuyor :)  **Squash** yani **duvar tenisini** duymuş olabilirsiniz, tenis topunu duvara gönderdiğimiz, sonuç olarak duvarında topu bize geri gönderdiği bir spor. İşte **echo server da** tam olarak bu işi yapıyor. Kendisi topun fırlatıldığı bir duvar!  Tabi **squash da** tenis topu bize değil rakibimize geri dönüyor ama olsun, bence **echo server** ile benzeşiyorlar :)

Sözün özü, bir istemciden gönderilen mesajın aynısını istemciye geri gönderen sunuculara **echo server** diyoruz.

**Why the f\*ck?** demiş olabilirsiniz, Mesajın kendisini bize geri gönderen bir sunucuya neden ihtiyacımız olsun ki?

**Echo server'lar**; soket/network programlamamın **Hello World'ü** gibidir. Bir dilde, sistemde vs. **echo server** yazabilen,nasıl çalıştığını anlayabilen bir kişi; o dil ve/veya sistem de soket/network programlama temelini kavramıştır. Yani **echo server'ın** bir amacı yok,  sadece temelleri öğrenmelik basit bir **server/client** uygulaması...

**Echo server'ı** yazmadan önce fonksiyonlarımıza göz atalım...