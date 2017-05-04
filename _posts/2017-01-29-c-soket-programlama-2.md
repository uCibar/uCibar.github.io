---
title: Temel Soket Fonksiyonları (C Soket Programlama - 2)
layout: post
summary: C ile chat uygulaması. Soketler, multithreading, client, server... Bir MSN
  Messenger kolay yetişmiyor.
categories: c-lang
---

Tekrar merhaba, uzun bir sınav döneminden sonra **soketler** ve **C** ile olan maceramıza geri dönebiliriz. Bu yazıda temel soket fonksiyonlarını anlatacağım ve sonrasında basit bir **echo server** yazarız.

## Fonksiyonlar

Soket fonksiyonlarının bulunduğu kütüphaneler sistemden sisteme değişiklik gösterebilir diyorlar, mesela ben **arpa/inet.h** kütüphanesini kullanıyorum ve bütün soket fonksiyonlarını ve structları içeriyor. Ama internetteki çeşitli örneklerde **sys/socket.h** , **netinet/in.h** gibi başka kütüphanelere de rastlayabilirsiniz. Açıkçası aralarındaki farkı, hangisinin kullanılması gerektiğini veya programın taşınabilir olması için hepsi kullanılmalı mı gibi sorulara yanıtım yok, bu konu benimde kafamı karıştırdı diyebilirim, ben şimdilik **arpa/inet.h** kütüphanesini kullanmaya devam edeceğim.

### Socket() - Önce bir soket oluşturalım
{% highlight c %}
int socket(int domain, int type, int protocol);
{% endhighlight %}
**socket()** fonksiyonu geriye bir soket dosya tanımlayıcı döndürür, bu dosya tanımlayıcı aslında **integer** bir değerdir ve **socket()** fonksiyonu bize bir soket oluşturur diyebiliriz, Diğer soket fonksiyonlarımızda bu **integer** değeri (dosya tanımlayıcımızı) kullanırız ve o fonksiyonlar hangi soket için işlem göreceklerini bilirler.

Peki **socket()** fonsiyonunun aldığı argümanlar? hemen bahsedelim:
* **int domain:** bu argüman tıpkı **sockaddr_in** struct'ında olduğu gibi adres ailesini belirtir. **AF_INET** olabilir.

*  **int type:** bu argüman ise oluşturacağımız soketin ne tür bir soket olacağını belirtir. Aşağıdaki değerleri alabilir:
     1. **TCP** için; **SOCK_STREAM**
     2.  **UDP** için **SOCK_DGRAM**


* **int protocol:** bu argümanın değerini **0 (sıfır)** yapıyoruz. Neden mi? bilmem, bu konuda pek bir araştırma yapmadım, biraz ezbere kaçtım diyebilirim. Bu konuları öğrendiğim yerlerde hep **0** yapıyorlardı ve nedenini onlarda pek fazla açıklamıyordu :) Ama anladığım kadarı ile **int type** argümanında belirlediğimiz değere göre bir protokol belirlenmesi gerekiyor ve **protocol'ü sıfır** yaparsak fonksiyonumuz protokolü otomatik olarak kendisi belirliyor. Yani ben böyle anladım :)

Tabi ki, yukarıda bahsettiğim argümanların aldığı değerler bunlarla sınırlı değil, bir çok adres ailesi ve protokol için farklı değerler var, ben sadece **TCP/IP** protokolü için uygun olanlardan bahsettim.

Şimdi aşağıdaki kod parçacığına bakalım:

{% highlight c %}
int main_socket;
main_socket = socket(AF_INET, SOCK_STREAM, 0);
{% endhighlight %}

Bu kod parçacığında; **socket()** fonksiyonu ile **IPv4 - AF_INET** adres ailesini kullanan bir **TCP - SOCK_STREAM** soket oluşturduk ve **main_socket** değişkenine, oluşturduğumuz soketin dosya tanımlayıcısını atadık. Artık soketimiz ile yapacağımız işlemlere **main_socket** ile erişeceğiz.

Unutmadan değineceğim son şey ise; aslında bu bütün soket fonksiyonları için geçerli diyebiliriz, **socket()** fonksiyonu herhangi bir hata durumunda geriye **-1** değerini döndürür ve hatayı **errno** değişkenine atar.

Soket/network programlar iken, aşağıdaki gibi bir hata denetimi yapmanız, saç baş yolmanıza engel olabilir :)

{% highlight c %}
if( (main_socket = socket(AF_INET, SOCK_STREAM, 0)) == -1 ){
    printf("<socket()> function: %s\n",strerror(errno));
    exit(1);
  }
{% endhighlight %}

Çeşitli hata yazdırma fonksiyonlarını kullanabilirsiniz, ben **strerror()** tercih ediyorum, Ayrıca **errno.h** kütüphanesini eklemeyi unutmayın :)

### bind() - Portlara salça olalım
{% highlight c %}
int bind(int sockfd, struct sockaddr *my_addr, int addrlen);
{% endhighlight %}
**bind()** fonksiyonu, oluşturduğumuz soketi seçtiğimiz port ile ilişkilendirir diyebiliriz. Bu sayede **sunucuya** bağlanacak **istemcimizin** erişeceği veya **sunucumuzun** dinleyeceği port görevine hazır olur.

Hemen aldığı parametrelerden bahsedelim:
* **int sockfd:** bu parametre **socket()** fonksiyonu ile oluşturduğumuz dosya tanımlayıcısıdır. Üstteki örnek için konuşacak olur isek; **main_socket** değişkenidir.

* **struct sockaddr *my_addr:**  Evet, hatırlarsanız bir önceki yazımızda structlardan bahsetmiştik ve sunucumuzun bilgilerini **sockaddr_in** struct'ında tutuyorduk. Bu parametre sunucu bilgilerimizi alır ve bilgilerimizin içerisindeki port numarasını ilişkilendirir. Ama dikkat etmemiz gereken bir nokta var; önceki yazıda da dediğim gibi soket fonksiyonları **sockaddr** türünden bir değer veya adres bekler, fakat bizim bilgilerimiz **sockaddr_in** türünde! Bilgilerimizi **bind()** fonksiyonuna göndermeden önce tür dönüşümünü yapmalıyız yoksa **bind()** fonksiyonu **sockaddr_in** türünden bir değeri anlamayacaktır.

* **int addrlen:** İkinci parametrede gönderdimiz bilgilerin(adresin) boyut değerini alan parametredir. bizim örneğimiz için **sizeof(sockaddr)**'dir.

Diğer soket fonksiyonlarında olduğu gibi **bind()** fonksiyonunda da hata denetimi yapmayı unutmuyoruz, kullanmak istediğimiz bir port hali hazırda kullanılıyor olabilir ve bu nedenle **bind()** fonksiyonu çalışmayabilir, hata denetimi yapmazsanız programınınız bu sebeple mi yoksa başka bir sebeple mi çalışmadığını asla öğrenemezsiniz :)

{% highlight c %}
    if( bind(sockfd, (sockaddr *)server_addr, sizeof(sockaddr)) == -1 ){
        printf("Error: <bind()> function: %s\n",strerror(errno));
        exit(1);
    }
{% endhighlight %}

Ve ayrıca; eğer **sockaddr_in** tipinde oluşturduğumuz server adres değişkenin **port** kısmını 0(sıfır) yaparsanız, sistem sizin için uygun, müsait bir port numarası seçer.

**bind()** fonksiyonu ile ilgili son olarak; eğer sunucu programı değilde istemci(client) programı yazıyorsanız **bind()** fonksiyonu ile hiç işiniz olmaz. Çünkü istemcinin işi sunucuya bağlanmaktır ve bir portu dinlemek zorunda değildir. Sadece bağlanacağı sunucunu hangi portu dinlediğini bilmesi yeterlidir.

### connect() - Sunucuya bağlanma vakti geldi
{% highlight c %}
int connect(int sockfd, struct sockaddr *serv_addr, int addrlen); 
{% endhighlight %}

İsminden anlayabileceğiniz gibi **connect()** fonksiyonu, biryerlere bağlanmaktan sorumlu. Parametrelerini incelersek;

* **int sockfd:** tahmin edebileceğiniz gibi **socket()** foksiyonu ile oluşturduğumuz dosya tanımlayıcısıdır.

* **struct sockaddr *serv_addr:** bağlanmak istediğimiz adresi belirtiriz. Diğer fonksiyonlarda olduğu gibi burada da değişken türü **sockaddr**'dir ve tür dönüşümü yapmamız gerekmektedir.

* **int addrlen:** İkinci parametrede gönderdimiz bilgilerin(adresin) boyut değerini alan parametredir. bizim örneğimiz için **sizeof(sockaddr)**'dir.

Ve tabiki hata denetimi yapmayı unutmuyor, bir hata ile karşılaştığımızda saç baş yolmuyoruz.

{% highlight c %}
    if( connect(sockfd, (sockaddr *)server_addr, sizeof(sockaddr)) == -1 ){
        printf("Error: <connect()> function: %s\n",strerror(errno));
        exit(1);
    }
{% endhighlight %}

### listen() - Portu dinleyelim
{% highlight c %}
int listen(int sockfd, int backlog); 
{% endhighlight %}

Evet, sunucu programını yavaş yavaş yazıyoruz; **socket()** fonksiyonu ile bir soket oluşturduk, **bind()** fonksiyonu ile soketimizi ve dinlemek istediğimiz portu nişanladık ve sıra portu dinlemeye geldi! İşte **listen()** fonksiyonunun görevi bu! belirttiğimiz portu dinlemek. Hemen aldığı parametrelere bakalım:

* **sockfd:** **socket()** fonksiyonu ile oluşturduğumuz dosya tanımlayıcısı.

* **backlog:** Gelen çağrı kuyruğunda izin verilen bağlantı sayısını gösteriyor. Yani gelen bağlantı talepleri siz onları birazdan göreceğimiz **accept()** ile kabul edene dek bir kuyrukta bekler. İşte kuyruğumuzun sınırı **backlog**'tur.

Tahmin edin ne diyeceğim? Tabiki, hata denetimi yapmayı unutmuyoruz :)

### accept() - Bağlantıları kabul etme zamanı
{% highlight c %}
int accept(int sockfd, void *addr, int *addrlen);
{% endhighlight %}

Sıradan bir gün, **listen()** fonksiyonu ile portumuzu dinliyoruz. Bir süre sonra; Sessizliğin ortasında, kapı aralanıyor, oda ne? Küçük sevimli bir client içeri girmeye çalışıyor! Soğuk kanlılığımız ile **accept()** fonksiyonunu çağırıyoruz...

Evet, edebi yeteneğimi(!) bir kenara bırakırsak; **accept()** fonksiyonu, **listen()** ile dinlediğimiz bir porta bağlanmaya çalışan istemciyi kabul ettiğimiz fonksiyondur. **accept()** fonksiyonunun en önemli özelliği; Siz **accept()** işlevini çağırırsınız ve ona beklemekte olan çağrıyı kabul etmesini söylersiniz. O da size yepyeni bir soket dosya tanımlayıcısı döndürür sadece ve sadece söz konusu bağlantıya özel ve artık sunucuya bağlanan istemci ile yeni üretilen soket tanımlayıcısı üzerinden haberleşirsiniz.

**accept()** fonksiyonunun parametrelerine bakacak olursak;
* **int sockfd:** ana soket dosya tanımlayıcımızdır.

* **void *addr:** bağlanan istemcinin bilgilerini saklamak istediğimiz değişkenin adresi.

* **int *addrlen:** **sizeof(struct sockaddr_in)** değerini alır.

**accept()** fonksiyonuda diğerleri gibi hata durumunda **-1** döndürür.

### send() ve recv() - Birşey gönderme ve alma zamanı
{% highlight c %}
int send(int sockfd, const void *msg, int len, int flags);

int recv(int sockfd, void *buf, int len, unsigned int flags);
{% endhighlight %}

**accept()** fonksiyonu ile istemcimizi kabul ettiğimize göre, artık veri gönderip alabiliriz.

**send()** fonksiyonundan başlatacak olursak;
Tahmin edebileceğiniz gibi veri göndermek için kullandığımız fonksiyondur. Hemen aldığı parametrelere göz atalım;

* **int sockfd:** bu parametrede önemli bir nokta var; aldığı değer(soket tanımlayıcısı) **socket()** fonksiyonunun geriye döndürdüğü değer değildir. **accept()** fonksiyonunun geriye döndürdüğü o bağlantıya özel soket tanımlayıcısıdır.

* **const void *msg:** göndereceğimiz mesajın adresidir.

* **int len:** göndereceğimiz mesajın boyutudur.

* **int flags:** bu parametre ile ilgili pek bir bilgim yok, sanırım gönderime özel seçenekleri belirttiğimiz bir parametre. **0(sıfır)** olarak bırakabilirsiniz. Ayrıntılı bilgi isterseniz **send()** fonksiyonunun **man** sayfasına bakabilirsiniz.

**send()** fonksiyonunu kullanırken dikkat etmeniz gereken bir nokta var. **send()** fonksiyonu biraz nazlıdır.

**send()** fonksiyonu geriye; gönderdiği verinin boyutunu döndürür. Bu boyut sizin göndermek istediğiniz verinin boyutundan az olarabilir! Örnek vermek gerekirse; Siz 50 byte'lık bir veri göndermek için **send()** fonksiyonunu çalıştırdınız fakat **send()** fonksiyonu geriye 35 byte değerini döndürdü yani 35 byte gönderdiğini söyledi. E, peki geriye kalan 15 byte'lık veri nerede?

**send()** fonksiyonu Sizin göndermek istediğiniz verinin tamamını göndemeyebilir ve geriye "abi ben bu kadar gönderebildim" diyerek gönderdiği byte miktarını döndürür ve geriye kalan veriyi göndermek sizin sorumluluğunuzdadır. İyi haber ise; 1KB civarı boyuta kadar tek seferde gönderim yapabildiği söyleniyor.

**recv()** fonksiyonu, isminden de anlaşılacağı gibi veri almamızı sağlayan fonksiyondur. parametreleri **send()** ile benzerdir.

* **int sockfd:** **accept()** fonksiyonu ile gelen, o bağlantıya özel soket tanımlayıcısı.

* **void *buf:** gelen verinin yazılacağı tampon alanı.

* **int len:** gelen verinin boyutudur.

**recv()** fonksiyonu geriye gelen verinin boyutunu döndürür. Hata durumunda -1 döndürür ve eğer 0 değerini döndürüse bilin ki karşı taraf bağlantıyı koparmıştır.

### close() artık yatma vakti
{% highlight c %}
close(sockfd);
{% endhighlight %}
Evet tahmin edebileceğiniz gibi **close()** fonksiyonu bir soket bağlantısını sonlandırmak için kullanılıyor. Aldığı tek parametre; kapatmak istediğiniz soket tanımlayıcısı.


Temel soket fonksiyonlarından bahsettik. Daha öncede dediğim gibi; ben chat uygulamamda kullandığım foksiyonlardan ve durumlardan bahsedeceğim bu seride. Bu fonksiyonlar dışında bir çok soket fonksiyonu ve durum var. Daha ayrıntılı bilgi almak isterseniz; [beej'in ağ programlama kılavuzunu](http://www.belgeler.org/bgnet/bgnet.html) şiddetle tavsiye ederim.

Bir sonraki yazıda **echo server** yapmaya başlayabiliriz...