# MVP Planı: Mağaza Stok Uyarı Paneli (Skechers)

**Hedef:** Sabah gelen mağaza sorumlusu panele dosyayı yüklediği anda "sipariş vermem gereken ürün var mı, hangileri?" sorusunun cevabını görsün.
**Süre:** 2 saat · **Çıktı:** `index.html` (tek dosya) + `README.md`

---

## 1. Teknik tercih

| Konu | Karar | Gerekçe |
|---|---|---|
| Yapı | Tek dosya `index.html` (HTML + CSS + JS içinde) | Backend gerekmiyor: veri her sabah yükleniyor, saklama/loglama istenmedi |
| Dosya okuma | SheetJS `xlsx@0.18.5` (cdnjs), hem .xlsx hem .csv | Tek kütüphane iki formatı da okuyor |
| Logo | `Case_Gorseller/Skechers-Logo-768x432.png` → base64 olarak HTML'e gömülü | Tek dosya her yerde logolu açılsın. Görselde boşluk çok, CSS ile `height` + `object-fit: cover` kırpılır |
| Renk | Skechers laciverti (~`#0B2A5B`) başlık/vurgu rengi | Logo ile uyumlu |
| Gizlilik | Dosya tarayıcıda işlenir, hiçbir sunucuya gitmez | Canlıya çıkılsa bile stok verisi dışarı çıkmaz |
| Canlı (opsiyonel) | Netlify Drop veya GitHub Pages (statik) | Tek dosyayı sürükle-bırak ile yayına almak yeterli |

---

## 2. İş kuralları

| # | Kural |
|---|---|
| IK-1 | **Kalan gün** = `mevcut_stok ÷ gunluk_satis_ortalamasi`, 1 ondalıkla gösterilir |
| IK-2 | **Tahmini bitiş tarihi** = bugün + kalan gün (ör. "≈ 6 Kas") |
| IK-3 | **Durumlar** (üstten alta öncelik): |
| | ⚫ **Tükendi**: `mevcut_stok = 0` ve `gunluk_satis > 0` |
| | 🔴 **Kritik**: kalan gün ≤ `tedarik_suresi_gun` (şimdi sipariş verilse bile yetişmeyebilir) **veya** `mevcut_stok < 2` (adet alt sınırı, `MIN_STOK = 2` sabiti; satış > 0 olan ürünlerde) |
| | 🟡 **Dikkat**: tedarik süresi < kalan gün ≤ **2 × tedarik süresi** |
| | 🔵 **Kontrol et**: `mevcut_stok = 0` ve `gunluk_satis = 0`. `mevcut_stok` reyon stoğu olduğu için bu ürün reyonda yok; depoda bekliyor, hiç reyona çıkmamış ya da satıştan kalkmış olabilir. Satış 0'ı talep yokluğu saymıyoruz. **Görünür**, Dikkat ile Yeterli arasında sıralanır, sipariş sayısına (IK-5) girmez |
| | 🟢 **Yeterli**: kalan gün > 2 × tedarik süresi |
| | ⚪ **Hareketsiz**: `gunluk_satis = 0` ve `mevcut_stok > 0` (reyonda duruyor ama satmıyor) → varsayılan gizli |
| IK-4 | **Sıralama otomatik**: önce durum önceliği. Grup içinde: Tükendi → günlük satışı yüksek olan üstte (kaybedilen satış büyük). Diğerleri → **pay = kalan gün − tedarik süresi** küçük olan üstte, eşitlikte kalan gün küçük olan üstte |
| IK-5 | **Özet kutusu** (tablonun üstünde): "Bugün **X ürün** için sipariş vermelisin" (X = Tükendi + Kritik). Yanında durum sayaçları. X = 0 ise yeşil "Bugün acil sipariş yok" |
| IK-6 | **Beden**: her SKU (her beden) ayrı satır. Gruplama yok |
| IK-7 | **Hareketsiz ürünler**: tablonun altında "X hareketsiz ürün, göster/gizle" anahtarı |
| IK-8 | **Doğrulama**: zorunlu kolonlar `SKU, ad, kategori, mevcut_stok, gunluk_satis_ortalamasi, tedarik_suresi_gun` (büyük/küçük harf ve boşluk toleranslı). Eksik kolon → yükleme reddedilir, hangi kolonun eksik olduğu yazılır. Sayısal olmayan, negatif ya da boş değerli satırlar, `tedarik_suresi_gun ≤ 0` olan satırlar ve tekrarlanan SKU'lar (ilki tutulur) atlanır; "N satır atlandı" uyarısı ve satır listesi gösterilir. Ondalık ayırıcı nokta |
| IK-9 | **Örnek veri**: panel açılışta örnek veriyle dolu gelir, üstte belirgin **"ÖRNEK VERİ"** etiketi olur. Dosya yüklenince etiket "Yüklenen dosya: ad · saat" olur. Saklama yok, sayfa yenilenince örnek veriye döner |
| IK-10 | **Filtre ve arama** (kendi kararım, düşük maliyetli): durum çipleri, kategori açılır listesi, SKU/ad arama kutusu. Filtreler sıralamayı bozmaz |
| IK-11 | **Şablon indir**: doğru kolon adlarıyla boş/örnek CSV indirme bağlantısı (yanlış kolon adı hatasını önler). Şablonda isteğe bağlı `gorsel_url` kolonu da bulunur |
| IK-12 | **Ürün görseli**: isteğe bağlı `gorsel_url` kolonu (zorunlu kolonlar değişmez, kolon hiç olmayan dosyalar da kabul edilir). Ad kolonunun solunda ~40px küçük görsel gösterilir (`loading="lazy"`). Değer boşsa, `http(s)://` ile başlamıyorsa ya da görsel yüklenemezse (`onerror`) kategori ikonu gösterilir (👟 Ayakkabı, 👕 Tişört, 🎒 Çanta, 🧦 Çorap, 📦 diğer). Geçersiz URL satırı atlatmaz. **Örnek veride gerçek görsel bağlantısı yok**, kategori ikonları kullanılır (telif ve kırık link riski). Gerçek URL'ler mağazanın ürün veritabanından gelir |
| IK-13 | **"Ne yapmalı?" kolonu** (Durum'un yanında, miktar yok). `pay = kalan gün − tedarik süresi`: |
| | ⚫ Tükendi → **"Bugün sipariş ver"**, alt satır: "satış kaybediliyor · ~{tedarik} gün daha stoksuz" |
| | 🔴 Kritik, pay < 0 → **"Bugün sipariş ver"**, alt satır: "~{ceil(−pay)} gün stoksuz kalacak" |
| | 🔴 Kritik, pay = 0 → **"Bugün sipariş ver"**, alt satır: "son gün, mal stok biterken gelir" |
| | 🔴 Kritik yalnızca adet kuralıyla (pay > 0) → **"Bugün sipariş ver"**, alt satır: "reyonda {stok} adet kaldı (alt sınır 2)" |
| | 🔵 Kontrol et → **"Reyonda yok, depoyu kontrol et"**, alt satır: "satış verisi yok, reyona çıkmamış olabilir" (mavi, kalın değil) |
| | 🟡 Dikkat → **"En geç {floor(pay)} gün içinde sipariş ver"** + "(≈ tarih)", tarih = bugün + floor(pay). floor(pay) = 0 ise "En geç bugün sipariş ver" |
| | 🟢 Yeterli → "Sipariş gerekmez" (soluk) · ⚪ Hareketsiz → "—" |
| | Kritik/Tükendi metni kalın ve kırmızı (aciliyet vurgusu). Sıralama değişmez (IK-4 zaten pay'e göre) |

---

## 3. Örnek veri (~25 satır)

- PO'nun verdiği 3 satır aynen yer alır (`TSH-1042`, `AYK-8801`, `CNT-3301`).
- Skechers ürün gamına benzer satırlar: sneaker/koşu ayakkabısı (farklı numaralar), tişört (S/M/L), çanta, çorap.
- **Her durumdan en az 2 satır olur**: Tükendi, Kritik, Dikkat, Yeterli, Hareketsiz. Ayrıca aynı modelin bir bedeni kritik, diğeri yeterli olan bir örnek (IK-6'yı göstermek için).
- Adet kuralını göstermek için 1 satır eklenir: `AYK-9100-45, Max Cushioning Erkek Sneaker 45, Ayakkabı, 1, 0.05, 7` → kalan 20 gün (gün kuralına göre Yeterli) ama stok < 2 olduğu için **Kritik**.
- Mevcut `TSH-4000-XL` (stok 0, satış 0) **Kontrol et** durumuna geçer.
- Dry Fit Koşu Tişörtü Siyah'ın tüm bedenleri yer alır (bedenler farklı durumda): `TSH-2201-M, Dry Fit Koşu Tişörtü Siyah M, Tişört, 6, 2.5, 4` (Kritik) · `TSH-2201-L, …L, Tişört, 18, 2, 4` (Yeterli) · `TSH-2201-XL, …XL, Tişört, 10, 1.5, 4` (Dikkat).
- **Tam beden serileri** (beden gerektiren her modelde tüm bedenler; PO'nun kendi 3 satırı olduğu gibi kalır): Kadın ayakkabı 36–41, Erkek ayakkabı 40/41–45(46), Tişört S–XL. Eklenen 24 satır:

```
SKU,ad,kategori,mevcut_stok,gunluk_satis_ortalamasi,tedarik_suresi_gun   → beklenen durum
AYK-2320-36,D'Lites Kadın Sneaker Beyaz 36,Ayakkabı,12,0.4,5            → Yeterli
AYK-2320-37,D'Lites Kadın Sneaker Beyaz 37,Ayakkabı,7,0.8,5             → Dikkat
AYK-5510-36,Arch Fit Kadın Yürüyüş 36,Ayakkabı,10,0.5,6                 → Yeterli
AYK-5510-37,Arch Fit Kadın Yürüyüş 37,Ayakkabı,9,1.2,6                  → Dikkat
AYK-5510-41,Arch Fit Kadın Yürüyüş 41,Ayakkabı,15,0.6,6                 → Yeterli
AYK-7720-40,Go Run Koşu Ayakkabısı Mavi 40,Ayakkabı,20,0.5,7            → Yeterli
AYK-7720-43,Go Run Koşu Ayakkabısı Mavi 43,Ayakkabı,4,1.4,7             → Kritik
AYK-7720-44,Go Run Koşu Ayakkabısı Mavi 44,Ayakkabı,16,0.9,7            → Yeterli
AYK-7720-45,Go Run Koşu Ayakkabısı Mavi 45,Ayakkabı,0,0.4,7             → Tükendi
AYK-9100-40,Max Cushioning Erkek Sneaker 40,Ayakkabı,14,0.6,7           → Yeterli
AYK-9100-41,Max Cushioning Erkek Sneaker 41,Ayakkabı,11,1,7             → Dikkat
AYK-9100-42,Max Cushioning Erkek Sneaker 42,Ayakkabı,8,1.6,7            → Kritik
AYK-9100-43,Max Cushioning Erkek Sneaker 43,Ayakkabı,22,1.5,7           → Yeterli
AYK-9100-44,Max Cushioning Erkek Sneaker 44,Ayakkabı,9,0.7,7            → Dikkat
AYK-9001-41,Uno Erkek Sneaker Beyaz 41,Ayakkabı,15,0.7,7                → Yeterli
AYK-9001-42,Uno Erkek Sneaker Beyaz 42,Ayakkabı,10,1.3,7                → Dikkat
AYK-9001-43,Uno Erkek Sneaker Beyaz 43,Ayakkabı,18,1.2,7                → Yeterli
AYK-9001-44,Uno Erkek Sneaker Beyaz 44,Ayakkabı,0,0,7                   → Kontrol et (yeni beden, reyona çıkmamış)
AYK-9001-45,Uno Erkek Sneaker Beyaz 45,Ayakkabı,8,0.4,7                 → Yeterli
TSH-3105-S,Oversize Logo Tişört Lacivert S,Tişört,14,1.5,3              → Yeterli
TSH-3105-XL,Oversize Logo Tişört Lacivert XL,Tişört,3,1.2,3             → Kritik
TSH-4000-S,Retro Logo Tişört Gri S,Tişört,9,0.8,4                       → Yeterli
TSH-4000-M,Retro Logo Tişört Gri M,Tişört,5,1.1,4                       → Dikkat
TSH-4000-L,Retro Logo Tişört Gri L,Tişört,0,1,4                         → Tükendi
```

- Beklenen sayılar (51 satır): Tükendi 5 · Kritik 10 · Dikkat 12 · Kontrol et 2 · Yeterli 20 · Hareketsiz 2 → özet "Bugün **15 ürün** için sipariş vermelisin".

---

## 4. Fazlar (2 saat)

| Faz | Süre | İçerik | Bitti sayılır |
|---|---|---|---|
| F0 İskelet | 10 dk | `index.html`, başlık + logo, "ÖRNEK VERİ" etiketi, örnek veri dizisi | Sayfa açılıyor, logo ve etiket görünüyor |
| F1 Hesap motoru | 25 dk | IK-1…IK-4, IK-6 | Örnek veride her satırın durumu ve sırası doğru |
| F2 Tablo + görsel | 30 dk | Renkli satırlar/rozetler, özet kutusu (IK-5), hareketsiz anahtarı (IK-7) | Açılışta ilk görülen şey "X ürün için sipariş vermelisin" |
| F3 Yükleme | 25 dk | .xlsx/.csv yükleme, doğrulama ve hata mesajları (IK-8), etiket değişimi (IK-9), şablon indir (IK-11) | PO'nun 3 satırlık örneği CSV olarak yüklenince doğru gösteriliyor; hatalı dosya anlaşılır mesaj veriyor |
| F4 Filtre + görsel | 20 dk | IK-12 ürün görseli, IK-10 filtre (zaman kalırsa) | `gorsel_url` dolu satırda görsel, boş/kırık satırda kategori ikonu; durum/kategori/arama çalışıyor |
| F5 README + kontrol | 20 dk | README tabloları, kabul kriteri kontrolü, (opsiyonel) canlıya çıkış | Aşağıdaki kontrol listesi tamam |

**Zaman sıkışırsa kesilecek sıra:** IK-10 → IK-11 → IK-2 → IK-12.

---

## 5. Kapsam dışı (sonraki iterasyonlar)

- Sipariş **miktarı** önerisi (PO: "sonraki iterasyonda")
- Sipariş listesini CSV olarak dışa aktarma (depoya göndermek için; PO istemedi, aday özellik)
- Sipariş talebinin bir sisteme iletilmesi / entegrasyon (PO: sipariş mevcut kanaldan verilir, öneri yeterli)
- Reyon/depo stok ayrımı. Veride yok; PO bu veriye erişimin faydalı olacağını söyledi, sonraki iterasyon adayı
- **Yoldaki sipariş** takibi / "sipariş verildi" işareti. **Bilinen sınırlama:** sipariş verilen ürün, mal gelene kadar her sabah kırmızı görünmeye devam eder
- Çoklu şube
- Verinin saklanması, geçmiş/loglama, trend karşılaştırması
- Beden gruplama (model bazında toplam stok)
- **Model bazında beden kırılımı görünümü** (ör. "Dry Fit Koşu Tişörtü Siyah: S ⚫ · M 🔴 · L 🟢 · XL 🟡"). PO faydalı buldu, sonraki aşama. Excel'e `model_kodu` ve `beden` kolonları gerekir; mevcut satır görünümü varsayılan kalır
- POS/ERP entegrasyonu, kullanıcı girişi
- Satış ortalamasının hesaplanması (dosyadaki `gunluk_satis_ortalamasi` olduğu gibi kabul edilir; mevsimsellik/kampanya etkisi yok)

---

## 6. README'de olacak tablolar

### Kabul kriteri → Panelde karşılığı
| Kabul kriteri | Panelde karşılığı |
|---|---|
| Ürün listesi bir tabloda görünsün | Ana tablo: SKU, ad, kategori, stok, günlük satış, tedarik, kalan gün, tahmini bitiş, durum |
| Her ürünün stokta ne kadar süre kalacağı hesaplansın | "Kalan gün" + "Tahmini bitiş" kolonları (IK-1, IK-2) |
| Kritik ürünler görsel olarak öne çıksın | Renkli durum rozeti + satır rengi (IK-3) |
| Sıralama otomatik, en acil üstte | IK-4: durum önceliği + pay sıralaması, kullanıcı müdahalesi yok |
| Sabah "sipariş vermem gereken var mı" hemen anlaşılsın | Özet kutusu (IK-5) |

### PO'nun söylediği → Karar
| PO'nun söylediği | Karar |
|---|---|
| Kritik = kalan gün ≤ tedarik süresi; dikkat 1.5–2 kat olabilir, karar sizde | Kritik ≤ tedarik; Dikkat ≤ 2 × tedarik |
| Tedarik 3–7 gün, ürüne göre değişir, `tedarik_suresi_gun` alanı var | Hesap ürün bazında bu alanla yapılır |
| Her sabah .xlsx/.csv yükleme yeterli | Tek dosya HTML, tarayıcıda dosya yükleme |
| Sipariş miktarı önerisi şimdilik gerekmez | Kapsam dışı |
| Stok 0 / satış 0 olanları göstermeyelim → (öneri sonrası) Tükendi üstte, hareketsizler gizli olsun | Tükendi en üstte; hareketsizler gizli + "göster" anahtarı |
| Kolon adları: SKU, ad, kategori, mevcut_stok, gunluk_satis_ortalamasi, tedarik_suresi_gun; ondalık nokta | Doğrulama bu adlarla; şablon indirme |
| Yoldaki sipariş bilgisi yok, şimdilik gerek yok | Kapsam dışı + bilinen sınırlama notu |
| Tek mağaza, tek dosya | Şube kolonu/filtresi yok |
| Veri hatırlansın/loglansın mı → hayır, her açılışta yükleme | Saklama yok; açılışta örnek veri |
| Her beden ayrı satır | Her SKU ayrı satır |
| Skechers çalışması, logo kullanılacak | Başlıkta Skechers logosu, lacivert tema |
| Ürün görselleri iyi olur, `gorsel_url` kolonu eklensin; bağlantılar ürün veritabanından gelecek | İsteğe bağlı `gorsel_url` kolonu; boş/kırık bağlantıda kategori ikonu; örnek veride ikon |
| "Bugün sipariş ver / ~X gün stoksuz / En geç X gün içinde" ifadeleri net; kritikte aciliyet, dikkatte zaman aralığı önemli | "Ne yapmalı?" kolonu (IK-13); miktar yok |
| Panelin öneriyi göstermesi yeterli, sipariş mevcut kanaldan verilir, entegrasyon gerekmez | Panel yalnızca öneri gösterir; entegrasyon kapsam dışı |
| `mevcut_stok` reyon stoğudur; reyon/depo ayrımı veride yok, şimdilik mevcut veriyle devam | Stok 0 + satış 0 → "Kontrol et" (reyonda yok, talep yokluğu sayılmaz); stok > 0 + satış 0 → Hareketsiz; reyon/depo ayrımı kapsam dışı + bilinen sınırlama |
| Stoğu 2 adetin altına düşen ürün de kritik sayılsın; hem gün hem adet kriteri | Kritik = kalan gün ≤ tedarik **veya** stok < 2 |
| Aynı modelin bedenlerini yan yana gösteren görünüm pratik ve faydalı; model kodu/beden ayrı alınabilirse Excel'e eklenmeli; ama sonraki aşamada | Kapsam dışı (sonraki aşama): model bazında beden kırılımı görünümü, `model_kodu` + `beden` kolonu gerektirir |
| Login ekranı şu aşamada gerekli değil; basit ve hızlı erişim daha önemli | Giriş ekranı yok (kapsam dışı: kullanıcı girişi). Veri tarayıcıda işlendiği için girişsiz kullanım stok verisini dışarı açmaz |

---

## 7. Teslim öncesi kontrol listesi

- [ ] Açılışta örnek veri dolu ve "ÖRNEK VERİ" etiketi görünüyor
- [ ] Logo başlıkta düzgün (kırpılmış, bozulmamış)
- [ ] 5 kabul kriterinin her biri panelde görülebiliyor
- [ ] Tükendi > Kritik > Dikkat > Yeterli sırası doğru, hareketsizler gizli
- [ ] PO'nun 3 satırlık CSV'si yüklenince: üçü de Kritik (1.9 ≤ 3, 6.7 ≤ 7, 2.5 ≤ 5)
- [ ] Eksik kolonlu / hatalı satırlı dosyada anlaşılır mesaj
- [ ] .xlsx dosyası da yükleniyor
- [ ] "Ne yapmalı?" kolonu: örnek veride Arch Fit 38 → "Bugün sipariş ver · ~3 gün stoksuz"; Dikkat satırlarında "En geç X gün içinde" + tarih
- [ ] Adet kuralı: AYK-9100-45 (stok 1, kalan 20 gün) Kritik ve "reyonda 1 adet kaldı (alt sınır 2)"
- [ ] TSH-4000-XL "Kontrol et" olarak görünür, sipariş sayısına girmez; özet "15 ürün"
- [ ] Beden gerektiren her modelde tam seri: Kadın ayakkabı 36–41, Erkek 40/41–45(46), Tişört S–XL
- [ ] Dry Fit Koşu Tişörtü Siyah S/M/L/XL dört ayrı satırda, farklı durumlarda (Tükendi/Kritik/Dikkat/Yeterli)
- [ ] `gorsel_url` olmayan dosya (PO'nun 3 satırı) hatasız yükleniyor; dolu/boş/kırık URL'de görsel veya ikon doğru
- [ ] README'de iki tablo + kapsam dışı + bilinen sınırlama + çalıştırma talimatı
