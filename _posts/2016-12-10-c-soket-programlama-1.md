---
title: Soketler, Soketlerimiz! (C Soket Programlama)
layout: post
summary: C ile chat uygulaması. Soketler, multithreading, client, server... Bir MSN
  Messenger kolay yetişmiyor.
categories: C
thumbnail: cogs
tags:
- socket
- soket
- C
- network
- multithreading
---

Data Communıcation Systems dersinde, hocamızın verdiği chat uygulaması ödevi üzerine kolları sıvadım ve açtım atomu. Kötü geçen bir vize ve bu ödevi yaparsak eklenecek 15 puan, üstüne bir de soketler, network gibi eğlenceli konuların geçtiği bir ödev, ne güzel değil mi? Nedense okulda verilen ödevleri genelde sevmişimdir. Merak etmeyin "hocam ödev vardı?" diyen öğrencilerden değilim :) Sadece okuldaki ezberci eğitimden biraz sıyrılıp, kendimizi geliştirdiğimiz ve birşeyler ürettiğimiz ödevler hoşuma gidiyor.

Neyse, konumuza dönecek olursak; hocamız ödevi C, Java veya C# da yapmamızı istedi. Javaya pek aşina değilim, C# da onun laciverti zaten, bende C ile yapmaya karar verdim, low level candır :)

Tamam,  C bilgim fena değildir fakat soket programlama hakkında pek bir bilgim yok, bende klasik olarak Google'a "C soket programlama" yazdım.(evet türkçe yazdım) Çıkan sonuçlar pek tatmin etmedi,  detaylı bir kaynak bulamadım. Yabancı kaynaklara yöneldiğimde, o muhteşem kılavuzu, [beej'in ağ programlama kılavuzunu](http://www.belgeler.org/bgnet/bgnet.html) buldum. [belgeler.org](http://www.belgeler.org/) da kılavuzu türkçeleştirmiş sağolsun. Bu seride anlatacağım şeylerin neredeyse %80'ini bu kılavuzdan öğrendim diyebilirim.

Anlatmaya başlamadan önce belirtmek istediğim son şey ise; C ile soket programlarken Unix ve Windows sistemlerde işlerin farklı yürüdüğü. Ayrıca işin içine ileride "multi-threading" kavramı girince bu fark iyice açılıyor. ben unix üzerinden anlatacağım, şimdiden belirteyim.

İşin teorisini yani **ağ teorisini** geçiyorum,  tcp/ip, katmanlar, veri paketlemesi vs. bunları, önerdiğim kılavuzdan öğrenebilirsiniz. Ben işin programlama kısmını anlatmaya çalışacağım.

# Ufak Adımlar
Bir soket açmak istediğimizde çeşitli bilgilere ihtiyaç duyarız, tahmin edebileceğimiz gibi bunlar; port, ip adresi, ipv4 mü veya v6 mı gibi bilgiler. C'de bu bilgileri tamı tamına üç struct'ta tutuyoruz. Evet, nedense tek bir struct yerine üç adet struct'a ihtiyacımız var. Bunlardan bir tanesi diğer ikisinin birleşiminden oluşuyor diyebilirim, yani bizim asıl struct'ımızı oluşturuyor.

ve karşınızda, Struct'larımız:

{% highlight c %}
struct sockaddr {
    unsigned short    sa_family;    // adres ailesi, ileride değineceğiz.
    char              sa_data[14];  // ip adresi  ve port'u birlikte barındıran kısım.
};
{% endhighlight %}

**sockaddr** struct'ının ip ve port'u birlikte barındırması ve char türünde olması, sıkıntılar çıkaracağından programcılar başka bir struct daha tasarlamışlar. Bu struct'ın ismi **sockaddr_in**. Peki **sockaddr'yi** niye tasarlamışlar? Hiçbir fikrim yok.

{% highlight c %}
struct sockaddr_in {
    short int          sin_family;  // Adres ailesi, merak etmeyin ileride değineceğiz :)
    unsigned short int sin_port;    // Port numarası.
    struct in_addr     sin_addr;    // ip adresi.
    unsigned char      sin_zero[8]; // sockaddr ile aynı boyda olması için 8 byte null değer.
};
{% endhighlight %}

Gördüğünüz gibi **sockaddr_in** struct'ımız, kardeşi **sockaddr'ye** göre daha derli toplu. Port ve ip adresi ayrı ayrı tutuluyor ve port kısmı olması gerektiği gibi **int** türünde.

Burada belirtmem gereken en önemli kısım, tüm olayın **sockaddr** struct'ı ile gerçekleştiği. Yani çağıracağımız ve kullanacağımız bütün soket fonksiyonları **sockaddr** türünde bir veri veya adres bekliyor. Yapıyı ona göre kurmuşlar zamanında. Ama daha derli toplu, ip ve port'un ayrı ayrı tutulduğu bir struct daha kullanışlı değil mi? Cevap vereyim; evet, öyle! Bu yüzden **sockaddr_in** türünde bir yapı ortaya çıkmış.

Ama? bütün soket fonksiyonları **sockaddr** türünde veri beklemiyor mu? **sockaddr_in** struct'ını nasıl kullanacağız? Bunun cevabı **sockaddr** ve **sockaddr_in** struct'larının aynı boyutta olması ve böylelikle bu iki yapıyı birbirine dönüştürebilmemiz. İşte **sockaddr_in** struct'ındaki **sin_zero** değişkeni burada önem kazanıyor. Boyutun **sockaddr** struct'ı ile aynı olması için **sockaddr_in**'deki son 8 byte'ı sıfır ile dolduruyoruz.

Son olarak **sockaddr_in** struct'ının içerdiği **sin_addr** değişkenine değinelim; kendisi ip adresini tutan değişken ve **in_addr** türünde. Peki bu **in_addr** struct'ı nereden çıktı? İşte bahsetmemiz geren son struct bu.

{% highlight c %}
struct in_addr {
    unsigned long s_addr; // 32-bit yani 4 byte
};
{% endhighlight %}

Bu struct'a değinmeden önce [Endian'dan](https://tr.wikipedia.org/wiki/Endian) bahsetmem lazım. İşlemciler byte'ları saklarken önemli byte'ın solda veya sağda olmasına göre ikiye ayrılır. Önemli byte'ın solda bulunduğu sıralamaya "Big-Endian", sağda bulunduğu sıralamaya "Little-Endian" denir. Bu seride kullanacağımız TCP/IP protokolü Big-Endian kullanır. Yani TCP/IP paketi gönderileceği zaman önce en değerli bitler gönderilir veya alınır. Bizimde bu kurala uyup göndereceğimiz paketleri Big-Endian şeklinde düzenlememiz gerekir.

Düzenlememiz gereken en temel bilgiler IP adresi ve port numarasıdır. Bu değerleri Big-Endian'a çeviren fonksiyonlara ileride değineceğim, biz **in_addr** struct'ımıza geri dönelim; IP adreslerini bir değişkende tutmak istediğimizde bu değişkenimizin **String** türünde olması gerekiyor, C için konuşacak olursak bir **Char** dizisi olmalı. Sonuçta IP adresleri "nokta" ve "sayılardan" oluşan bir yapıda ve mesela doğrudan **integer** türünde bir değişkene atayamayız IP adresini.

Ayrıca Bir TCP/IP paketinin IP header'ının "kaynak adres" ve "hedef adres" kısımları 32 bit'ten oluşur, yani 4 byte. Tahmin edebileceğiniz gibi **String** veya **Char** dizisi türündeki bir IP adresi 4 byte'tan daha büyük boyutlarda. Bu yüzden IP adreslerini 4 byte boyutunda ve Big-Endian sıralamasında bir **integer'a** dönüştürmemiz gerekiyor. Bu dönüştürme işlemlerini ileride yapacağız ve merak etmeyin bu dönüştürmeyi yapmak, anlatmaktan daha kolay :) (C'de bunun için hazır fonksiyonlar var). Ve işte dönüştürdüğümüz bu IP adresini **in_addr** struct'ında tutuyoruz.

**in_addr** struct'ı görüldüğü gibi sadece **unsigned long**  türünde bir **integer** barındırıyor. Kafanızda şu soru oluşabilir; " E peki sadece **unsigned long** türünde bir **integer** ise neden bu türde bir değişkeni kendimiz oluşturmuyoruz? İlla **in_addr** struct'ını mı kullanmak zorundayız? " Evet zorundasınız! Çünkü bu sistemi kurgulayan programcılar zamanında bu yapıya göre kurgulamışlar, ve IP adresini göndereceğiniz herhangi bir soket fonksiyonu sizden **unsigned long** türünden bir **integer** değil, **in_addr** türünden bir değişken bekliyor. İşte bu yüzden ve malesef **in_addr** struct'ını kullanmak zorundayız. Asıl struct'ımız olan **sockaddr_in** struct'ının IP adresini tutan değişkeninin **in_addr** türünde olmasının nedeni de işte bu yüzden, Umarım anlatabilmişimdir :)

Bu üç struct kafanızı karıştırmış olabilir, ileri de örnekler yaptığımızda daha rahat kavrayacağınıza inanıyorum. Struct'ları bitirmeden önce özetlersek; bir soket oluşturacağımız zaman IP adresi, port vb. bilgileri tutacağımız asıl struct **sockaddr_in** struct'ıdır. Ama zamanında kurulan yapı **sockaddr_in** struct'ını tanımadığı için diğer iki struct'ı bir şekilde **sockaddr_in** ile birlikte kullanıyoruz.

# IP, Port ve Big-Endian Dünyası
Evet yukarıda Big-Endian ve değerli bitler ile ilgili bir sohbet gerçekleştirdik. Şimdi IP adresimizi ve port numaramızı Big-Endian dünyasına ekleyelim. Veri türlerini dönüştürme işlemine daha ayrıntılı göz atmak isterseniz tavsiye ettiğim kılavuzda ayrıntılı bilgiye ulaşabilirsiniz ([buradan](http://www.belgeler.org/bgnet/bgnet_structs-convert.html) ve [buradan](http://www.belgeler.org/bgnet/bgnet_structs-ipaddr.html)), Ben sadece yaptığım chat uygulamasında kullandığım 3 fonksiyondan bahsedeceğim.

Dönüştürebileceğiniz iki tür var. "short" ve "long". Port numaramız short ve ip adresimiz long türünde ve dönüşümleri buna göre yapmamız gerekiyor.

"Hocam? kullandığım işlemci mimarisi zaten Big-Endian sıralaması kullanıyor. Yine de bu dönüşümleri yapmalımıyım?" Eğer yazdığınız programın taşınabilir olmasını istiyorsanız, Evet! Aksi taktirde yazdığınız program başka bir bilgisayar da çalışmayabilir.

### htons() Fonksiyonu
**htons()** fonksiyonu "host" + "to" + "network"  + "short" kelimelerinin baş harflerinden oluşan( "to" hariç :) ), Little-Endian'ı, Big-Endian'a "short" türünde dönüştüren bir fonksiyon. Bu fonksiyona benzer **htonl()** fonksiyonu da mevcut ve dikkat edin sonunda "l" harfi olduğu için kendileri "long" türünde bir dönüşüm yapıyor. Bu arada bahsetmediğim bir kavram ise; "host" ve "network".

"Little-Endian" byte sıralaması eşittir "Host" ve "Big-Endian" bayt sıralaması eşittir "Network". Yani fonksiyonumuzun isimlendirilmesi gayet mantıklı :) Mesela **ntohs()** diye bir fonksiyonda var, ne iş yaptığını siz bulun :) Kısacası; **htons()** fonksiyonunu port numarasını dönüştürmek için kullanıyoruz.

### inet_addr() Fonksiyonu
Evet, yukarılarda bir yerlerde IP adreslerinin **String** türünde olduğunu ve **unsigned long** türünde bir **integer'a** dönüştürmemiz gerektiğini söylemiştim. İşte bu işi yapan fonksiyonumuz **inet_addr** fonksiyonu. IP adresimizi alıyor ve 4 byte'lık bir **integer'a** dönüştürüyor, bu sayede IP adresimizi IP Header'ına yerleştirebiliyoruz. Peki bu dönüştürdüğü **integer'ı** Big-Endian'a çevirecek miyiz? Normalde evet! Ama **inet_addr()** fonksiyonu o kadar yetenekli ki bunu bizim için yapıyor ve bizim dönüştürmemize gerek kalmıyor. Böyle bir dönüşüm yapmasaydı **htonl()** fonksiyonu ile dönüşümü kendimiz yapmalıydık.

Önemli bir diğer kısım ise; **inet_addr()** fonksiyonu, hata durumunda geriye **-1** döndürür. **sockaddr_in** struct'ında IP adresini tutan değişkenimiz, temelde **unsigned** türünde. Peki **unsigned** türünde bir değişkene **-1** değerini verirsek ne olur? "Derleyici hata verir!" diyenler yanılıyor :) **unsigned -1** eşittir **255.255.255.255**. Yani soketimiz için IP adresini yanlış belirlemiş oluruz. Sözün özü, **inet_addr()** fonksiyonunu kullanırken hata denetimi yapmayı unutmayın.

### inet_ntoa() Fonksiyonu
Bu fonksiyonumuz ise **unsigned long** türüne dönüştürülmüş IP adresimizi tekrar biz insanların anlayabileceği şekle,  **String** türüne dönüştürüyor. **inet_ntoa()** fonksiyonuna göndereceğimiz değer **in_addr** türünde olmalıdır.

### INADDR_ANY Sabiti
Yazdığınız programda, IP adresini elle değilde, otomatik olarak bilgisayarın IP adresi olarak belirlemek istiyorsanız kullanacağınız sabit **INADDR_ANY** sabitidir. **INADDR_ANY** sabitini Big-Endian sıralamasına dönüştürmenize gerek yoktur. Nedeni ise; bu sabitin değerinin aslında sıfıra eşit olması. Sıfırın Big,Little,Middle Endian'ı da sıfırdır :)
# Adres Ailesi
İleride bahsedeceğiz dediğim, **sockaddr_in** struct'ında bulunan **sin_family** değişkenimizden bahsetme zamanımız geldi. Nedir bu adres ailesi? IP adresi, internete veya bir ağa bağlı cihazlara verilen bir adres. TCP/IP paketleri gidecekleri cihazı bu adres sayesinde buluyor. Ayrıca IP adresi günümüzde en yaygın kullanılan adresleme yöntemidir. Ama çok sık kullanılmasalar da IP adresi dışında kullanılan adresleme yöntemleride var.

İşte adres ailesini belirmemizin nedeni bu. Ne tür bir adresleme protokolü kullanıldığını belirtiyoruz. Soket programlarken, illa TCP/IP protokolünü kullanacağız diye birşey yok. Bambaşka protokoller de kullanabiliriz ve bu protokollerin adresleme yöntemlerine ihtiyaç duyabiliriz. Ama tabi genelde internetin temelini oluşturan TCP/IP protokolünü kullanıyoruz :)

C de tanımlı birkaç adres ailesi:
* AF_INET    - IPv4 adres ailesini belirtir, bizimde kullanacağımız adres ailesi budur.
* AF_INET6 - IPv6'yı hepiniz duymuşsunuzdur, işte IPv6 adresleme için bu adres ailesini kullanıyoruz.
* AF_UNIX   -  Hakkında pek bir bilgiye sahip değilim, [buradan](https://en.wikipedia.org/wiki/Unix_domain_socket) okuyabilirsiniz.

Bunların dışında da birçok adres ailesi var, ama inanın ne oldukları ile ilgili hiç bir fikrim yok :)

## Örnekler
Evet çok konuştuk, Artık **sinaddr_in** struct'ımızı ve fonksiyonlarımızı kullanarak bir kaç örnek yapma vakti...

**sockaddr_in** struct'ının kullanımı;
{% highlight c %}
struct sockaddr_in server_addr; // sockaddr_in türünde, sunucunun adres bilgilerini tutan değişken.

server_addr.sin_family = AF_INET; //adres ailesinin IPv4 olduğunu belirtiyoruz.
server_addr.sin_port = htons(7878); //7878 portunu dinle. htons() ile Big-Endian'a çeviriyoruz.
server_addr.sin_addr.s_addr = INADDR_ANY; //Bilgisayarın IP adresini kullan, Elle IP belirtmiyoruz.

memset(&(server_addr.sin_zero), 0, 8); //sockaddr struct'ı ile aynı boyutta olması için 8 byte null değer.
{% endhighlight %}

Bilgisayarın IP adresini kullanmak yerine kendimiz bir IP adresi belirleyelim; Mesela programımız sadece Local'de çalışsın;
{% highlight c %}
server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");
{% endhighlight %}

Dediğimi yap, yaptığımı yapma kısmına hoş geldiniz :) Yukarıda **inet_addr()** fonksiyonunu kullanırken herhangi bir hata denetimi yapmadım. Eğer **inet_addr()** fonksiyonu hata verirse, geriye **-1** dönecek ve sunucu adresim **255.255.255.255** olacak. Siz bana bakmayın ve hata denetimi yapmayı unutmayın :)

Sunucumuza bir istemci bağlandı diyelim, IP adresini ekrana basmak istiyorsak;
{% highlight c %}
printf("%s \n", inet_ntoa(client.sin_addr) ); // sin_addr, in_addr türünde bir değişkendir.
{% endhighlight %}

Yazıyı burada sonlandırıyorum, ikinci yazıda temel soket fonksiyonlarından bahsederim ve ufak bir **echo server** yaparız...