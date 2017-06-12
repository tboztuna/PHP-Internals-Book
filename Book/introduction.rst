Giriş
============

Bu kitap, birkaç PHP geliştiricisinin işbirliği ile PHP'nin iç işleyişinin nasıl çalıştığını anlatmak ve daha iyi
bir döküman elde etmek üzere hazırlanmıştır.

Bu kitabın 3 temel amacı vardır:

 * PHP'nin iç işleyişini belgelemek ve açıklamak.
 * Dilin eklentiler ile nasıl genişletilebileceğini belgelemek ve açıklamak.
 * PHP'yi geliştirmek için toplulukla nasıl etkileşim kuracağınızı belgelemek ve açıklamak.

Bu kitap öncelikle C programlama dilinde tecrübesi olan geliştiricileri hedef almaktadır. Ancak, C programlama
dilini hiç bilmeyen geliştiriciler için de anlaşılabilir olması için, bilgileri özetlemeye çalışacağız.

.. note:: Giriş bölümlerinden bazıları henüz yazılmadı (özellikle de temel eklenti geliştirme ve fonksiyon tanımlama 
   ile ilgili olanlar), bu sebeple PHP eklentisi geliştirme konusunda yeniyseniz, geri kalan giriş dersleri
   yayınlanana kadar beklemeniz gerekecek ya da bu konuyla ilgili
   `diğer kaynaklardan <https://wiki.php.net/internals/references>`_ faydalanarak başlayabilirsiniz.

Bu kitabın reposu şu adreste mevcuttur: GitHub_. Hata bildirimi ve geridönüşlerinizi 
şu adresten bildirebilirsiniz: `issue tracker`_.

.. _GitHub: https://github.com/phpinternalsbook/PHP-Internals-Book
.. _issue tracker: https://github.com/phpinternalsbook/PHP-Internals-Book/issues