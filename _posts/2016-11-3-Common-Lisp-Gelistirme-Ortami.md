---
layout: post
title: Common Lisp Geliştirme Ortamı
---

Yakın zamanda [Peter Seibel](https://twitter.com/peterseibel)'ın meşhur kitabı [Practical Common Lisp](http://www.gigamonkeys.com/book/)'i okumaya başladım. Kitap web üzerinden çevrimiçi olarak okunabiliyor. Buraya da bu süreçte öğrendiklerimle ilgili basit notlar yazmak istiyorum.

Lisp ya da Common Lisp ile ilgili fazla bir şey söylemeden doğrudan geliştirme ortamının kurulumuna geçeceğim. İşlemtim sistemi olarak Ubuntu 16.04 versiyonunu kullanacağım. Genelde bütün eğitsellerde anlatılan Common Lisp geliştirme ortamı şu 3 bileşenden oluşuyor: bir Common Lisp gerçekleştirmesi (implementation), Emacs ve Slime. 

Commom Lisp aslında ANSI(American National Standards Institute) tarafından standartlaştırılmış bir standart olduğu için (biraz garip bir cümle oldu ama anlaşıldı umarım) bir çok farklı kurum/şirket/topluluk vs. tarafından gerçekleştirmeleri mevcut. [Şu](http://www.cliki.net/Common+Lisp+implementation) sayfada bir liste bulabilirsiniz. Ben bu eğitselde [SBCL](http://www.sbcl.org/) kullanacağım. SBCL kurmak için aşağıdaki komutu çalıştırabilirsiniz:

```
$ sudo apt-get install sbcl
```

Emacs hakkında çok fazla bir şey söylemeye gerek yok diye düşünüyorum. Kuruyoruz:

```
$ sudo apt-get install emacs
```

[Slime](https://common-lisp.net/project/slime/), Common Lisp ile Emacs üzerinde geliştirme yapmamızı kolaylaştıran bir Emacs modu. Slime kurmak için birden fazla yol var aslında. Kendi sayfalarında da anlattıkları [Melpa](https://melpa.org) üzerinden kurulum bunlardan birisi. Ben daha kolay olduğunu düşündüğüm ikinci yöntemi kullanacağım. Slime Ubuntu depolarında
mevcut. Dolayısı ile diğer paketleri kurduğumuz gibi Slime'ı da kurabiliriz:

```
$ sudo apt-get install slime
```

Şimdi ise Emacs üzerinden Slime ve SBCL kullanabilmek için ufak bir ayar yapmamız gerekecek. Bunun için istediğiniz bir metin editörü ile `~/.emacs` dosyasını açıyoruz. İçesirine aşağıdaki satırları ekliyoruz:

```
 ;; Set your lisp system and, optionally, some contribs
 (setq inferior-lisp-program "/opt/sbcl/bin/sbcl")
 (setq slime-contribs '(slime-fancy))
```

Kaydedip kapattıyoruz. Böylece kurulumumuzu tamamlamış oldu. Şimdi Emacs'i başlatıyoruz (ya da yeniden başlatıyoruz). 

```
$ emacs -nw
```

Editörümüz açıldıktan sonra `M-x slime`'ı tuşluyoruz. İlk çalıştırma biraz zaman alabilir. Bişeylerin derlendiğini vs görebilirsiniz. Derleme işlemleri bittiğinde aşağıdaki gibi bir pencere ile sizi bekliyor olacak:

```
; SLIME
CL-USER> 
```

Ve Common Lisp RPEL'imiz hazır. Burada:

```
CL-USER> (+ 5 6)
11
```

gibi basit fonksiyoları çağırabilir ya da kendi fonksiyonlarınızı tanımlayabilirsiniz:

```
CL-USER> (defun adder (x)
           (lambda (y) (+ x y)))
ADDER
CL-USER> (funcall (adder 2) 6)
8
```

Happy hacking!

