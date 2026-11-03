# Mağaza Stok Uyarı Paneli (Skechers)

Mağaza sorumlusu sabah stok dosyasını panele yüklediği anda **"Sipariş vermem gereken ürün var mı, hangileri?"** sorusunun cevabını görür.

- Her ürünün stokunun **kaç gün yeteceğini** ve **tahmini bitiş tarihini** hesaplar.
- Ürünleri **Tükendi / Kritik / Dikkat / Kontrol et / Yeterli / Hareketsiz** olarak renklendirir.
- **"Ne yapmalı?"** kolonunda her ürün için ne zaman sipariş verileceğini yazar ("Bugün sipariş ver · ~3 gün stoksuz kalacak", "En geç 2 gün içinde sipariş ver (≈ 5 Kas)").
- Ürün adının yanında **ürün görselini** gösterir; görsel yoksa ya da açılamıyorsa kategori ikonu (👟 👕 🎒 🧦 📦) çıkar.
- **En acil ürün en üstte** olacak şekilde otomatik sıralar.
- En üstteki özet kutusunda "Bugün **X ürün** için sipariş vermelisin" yazar.

Tek dosya: `index.html` (HTML + CSS + JS). Backend, kurulum veya veritabanı yok.

---

## Çalıştırma

1. `index.html` dosyasını tarayıcıda açın (çift tıklamak yeterli).
2. Panel açılışta **örnek veriyle** gelir; üstte sarı **"ÖRNEK VERİ"** etiketi görünür.
3. **📂 Dosya yükle** düğmesiyle (ya da dosyayı sayfaya sürükleyip bırakarak) günün `.xlsx` veya `.csv` dosyasını yükleyin. Etiket **"Yüklenen dosya: ad · saat"** olur.
4. Doğru kolon adlarını bilmiyorsanız **⬇ Şablon indir (CSV)** ile örnek dosyayı indirip onu doldurun.

> Excel/CSV okuma kütüphanesi (SheetJS `xlsx@0.18.5`) cdnjs'ten yüklenir, bu yüzden dosya yüklemek için internet bağlantısı gerekir. Dosyanın kendisi **tarayıcıda işlenir, hiçbir sunucuya gönderilmez.**

**Canlıya alma (opsiyonel):** `index.html` statik bir dosya olduğu için [Netlify Drop](https://app.netlify.com/drop)'a sürükle-bırak yapmak ya da GitHub Pages'e koymak yeterlidir.

### Dosya formatı

İlk satır başlık olmalıdır. Kolon adları büyük/küçük harf ve boşluk toleranslıdır; sıra önemsizdir, fazladan kolon olabilir.

| Kolon | Zorunlu | Tip | Örnek |
|---|---|---|---|
| `SKU` | ✔ | metin | `TSH-1042` |
| `ad` | ✔ | metin | `Basic Beyaz Tişört S` |
| `kategori` | ✔ | metin | `Tişört` |
| `mevcut_stok` | ✔ | sayı ≥ 0 | `15` |
| `gunluk_satis_ortalamasi` | ✔ | sayı ≥ 0 | `8` veya `2.5` |
| `tedarik_suresi_gun` | ✔ | sayı > 0 | `3` |
| `gorsel_url` | isteğe bağlı | `http(s)://` bağlantı | `https://…/urun.jpg` |

Örnek dosya (PO'nun örneği):

```csv
SKU,ad,kategori,mevcut_stok,gunluk_satis_ortalamasi,tedarik_suresi_gun,gorsel_url
TSH-1042,Basic Beyaz Tişört S,Tişört,15,8,3,
AYK-8801,Sneaker Beyaz 42,Ayakkabı,20,3,7,
CNT-3301,Sırt Çantası Siyah,Çanta,5,2,5,
```

- Ondalık ayırıcı **nokta** (`2.5`). `2,5` hatalı satır sayılır.
- CSV'de `,` veya `;` ayırıcı kullanılabilir; UTF-8 ve Türkçe Windows (Excel "CSV" kaydı) kodlaması okunur.
- Eksik kolon varsa dosya **reddedilir** ve hangi kolonun eksik olduğu yazılır; ekrandaki veri değişmez.
- Sayısal olmayan, negatif, boş değerli satırlar, `tedarik_suresi_gun` 0 olan satırlar ve tekrarlanan SKU'lar (ilki tutulur) **atlanır**; "N satır atlandı" uyarısında satır numarası ve sebebi listelenir.
- `gorsel_url` kolonu hiç yoksa dosya yine kabul edilir. Boş, `http(s)://` ile başlamayan ya da açılamayan bağlantı satırı atlatmaz; o satırda kategori ikonu görünür.

---

## İş kuralları

| # | Kural | Açıklama |
|---|---|---|
| IK-1 | Kalan gün | `mevcut_stok ÷ gunluk_satis_ortalamasi`, 1 ondalık. Durum kararı da bu yuvarlanmış değerle verilir (ekranda görülen ile kural tutarlı olsun diye). |
| IK-2 | Tahmini bitiş | bugün + kalan gün (ör. "≈ 6 Kas") |
| IK-3 | Durumlar (üstten alta öncelik) | ⚫ **Tükendi**: `mevcut_stok = 0` ve günlük satış > 0 · 🔴 **Kritik**: kalan gün ≤ tedarik süresi (şimdi sipariş verilse bile yetişmeyebilir) **veya** `mevcut_stok < 2` (adet alt sınırı, `MIN_STOK = 2` sabiti; yalnızca satış > 0 olan ürünlerde) · 🟡 **Dikkat**: tedarik süresi < kalan gün ≤ 2 × tedarik süresi · 🔵 **Kontrol et**: `mevcut_stok = 0` ve günlük satış = 0. `mevcut_stok` reyon stoğu olduğu için ürün reyonda yok; depoda bekliyor, hiç reyona çıkmamış ya da satıştan kalkmış olabilir. Satış 0 talep yokluğu sayılmaz. Görünür, Dikkat ile Yeterli arasında sıralanır, sipariş sayısına (IK-5) girmez, kendi sayacı ve filtre çipi var · 🟢 **Yeterli**: kalan gün > 2 × tedarik süresi · ⚪ **Hareketsiz**: günlük satış = 0 ve `mevcut_stok > 0` (reyonda duruyor ama satmıyor) |
| IK-4 | Sıralama | Önce durum önceliği. Tükendi içinde günlük satışı yüksek olan üstte (kaybedilen satış büyük). Diğerlerinde **pay = kalan gün − tedarik süresi** küçük olan üstte, eşitlikte kalan gün küçük olan üstte. |
| IK-5 | Özet kutusu | "Bugün **X ürün** için sipariş vermelisin" (X = Tükendi + Kritik) ve durum sayaçları. X = 0 ise yeşil "Bugün acil sipariş yok". |
| IK-6 | Beden | Her SKU (her beden) ayrı satır, gruplama yok. |
| IK-7 | Hareketsiz ürünler | Reyonda stok var ama satış 0. Varsayılan gizli; tablonun altında "X hareketsiz ürün · Göster/Gizle" anahtarı. |
| IK-8 | Doğrulama | Zorunlu kolon eksikse dosya reddedilir; hatalı satırlar atlanır ve listelenir (bkz. Dosya formatı). |
| IK-9 | Örnek veri | Açılışta örnek veri ve "ÖRNEK VERİ" etiketi; dosya yüklenince "Yüklenen dosya: ad · saat". Saklama yok. |
| IK-10 | Filtre ve arama | Durum çipleri, kategori listesi, SKU/ad arama. Sıralamayı değiştirmez; özet kutusu her zaman tüm dosyayı sayar. Yeni dosya yüklenince filtreler sıfırlanır. |
| IK-11 | Şablon indir | Doğru kolon adlarıyla (isteğe bağlı `gorsel_url` dahil) PO örneğini içeren CSV. |
| IK-12 | Ürün görseli | İsteğe bağlı `gorsel_url`. Ürün adının solunda ~40px görsel (`loading="lazy"`). Boş/geçersiz/kırık bağlantıda kategori ikonu: 👟 Ayakkabı, 👕 Tişört, 🎒 Çanta, 🧦 Çorap, 📦 diğer. Örnek veride görsel bağlantısı yok, ikon görünür. |
| IK-13 | "Ne yapmalı?" | Durum'un yanında, miktar önerisi yok. pay = kalan gün − tedarik süresi. ⚫ Tükendi → **Bugün sipariş ver** · "satış kaybediliyor · ~{tedarik} gün daha stoksuz" · 🔴 Kritik, pay < 0 → **Bugün sipariş ver** · "~{⌈−pay⌉} gün stoksuz kalacak" · 🔴 Kritik, pay = 0 → **Bugün sipariş ver** · "son gün, mal stok biterken gelir" · 🔴 Kritik yalnızca adet kuralıyla (pay > 0) → **Bugün sipariş ver** · "reyonda {stok} adet kaldı (alt sınır 2)" · 🔵 Kontrol et → "Reyonda yok, depoyu kontrol et" · "satış verisi yok, reyona çıkmamış olabilir" (mavi, kalın değil) · 🟡 Dikkat → "En geç {⌊pay⌋} gün içinde sipariş ver (≈ tarih)", ⌊pay⌋ = 0 ise "En geç bugün sipariş ver" · 🟢 Yeterli → "Sipariş gerekmez" (soluk) · ⚪ Hareketsiz → "—". Kritik/Tükendi metni kalın ve kırmızı. |

---

## Kabul kriteri → Panelde karşılığı

| Kabul kriteri | Panelde karşılığı |
|---|---|
| Ürün listesi bir tabloda görünsün | Ana tablo: durum, "Ne yapmalı?" (IK-13), SKU, ürün görseli/kategori ikonu + ad (IK-12), kategori, stok, günlük satış, tedarik, kalan gün, tahmini bitiş |
| Her ürünün stokta ne kadar süre kalacağı hesaplansın | "Kalan gün" + "Tahmini bitiş" kolonları (IK-1, IK-2) |
| Kritik ürünler görsel olarak öne çıksın | Renkli durum rozeti + satır rengi (IK-3). Kritik = kalan gün ≤ tedarik süresi **veya** stok < 2 adet |
| Sıralama otomatik, en acil üstte | IK-4: durum önceliği + pay sıralaması, kullanıcı müdahalesi yok |
| Sabah "sipariş vermem gereken var mı" hemen anlaşılsın | Özet kutusu (IK-5) + satır bazında "Ne yapmalı?" kolonu (IK-13) |

## PO'nun söylediği → Karar

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
| Ürün görselleri iyi olur, `gorsel_url` eklensin; linkler ürün veritabanından gelecek | İsteğe bağlı `gorsel_url`, boş/kırık linkte kategori ikonu |
| "Bugün sipariş ver / ~X gün stoksuz / En geç X gün içinde" ifadeleri net; kritikte aciliyet, dikkatte zaman aralığı önemli | "Ne yapmalı?" kolonu (IK-13); miktar yok |
| Panelin öneriyi göstermesi yeterli, sipariş mevcut kanaldan verilir, entegrasyon gerekmez | Panel yalnızca öneri gösterir; entegrasyon kapsam dışı |
| `mevcut_stok` reyon stoğudur; reyon/depo ayrımı veride yok, şimdilik mevcut veriyle devam | Stok 0 + satış 0 → "Kontrol et" (reyonda yok, talep yokluğu sayılmaz); stok > 0 + satış 0 → Hareketsiz; reyon/depo ayrımı kapsam dışı + bilinen sınırlama |
| Stoğu 2 adetin altına düşen ürün de kritik sayılsın; hem gün hem adet kriteri | Kritik = kalan gün ≤ tedarik **veya** stok < 2 |
| Login ekranı şu aşamada gerekli değil; basit ve hızlı erişim daha önemli | Giriş ekranı yok (kapsam dışı: kullanıcı girişi). Veri tarayıcıda işlendiği için girişsiz kullanım stok verisini dışarı açmaz |
| Aynı modelin bedenlerini yan yana gösteren görünüm pratik ve faydalı; model kodu/beden ayrı alınabilirse Excel'e eklenmeli, ama sonraki aşamada | Kapsam dışı (sonraki aşama): model bazında beden kırılımı görünümü; model_kodu + beden kolonu gerektirir |

---

## Kapsam dışı (sonraki iterasyonlar)

- Sipariş **miktarı** önerisi
- Sipariş listesini CSV olarak dışa aktarma (depoya göndermek için; PO istemedi, aday özellik)
- Sipariş talebinin bir sisteme iletilmesi / entegrasyon (PO: sipariş mevcut kanaldan verilir, öneri yeterli)
- Reyon/depo stok ayrımı. Veride yok; PO bu veriye erişimin faydalı olacağını söyledi, sonraki iterasyon adayı
- **Yoldaki sipariş** takibi / "sipariş verildi" işareti
- Çoklu şube
- Verinin saklanması, geçmiş/loglama, trend karşılaştırması
- Beden gruplama (model bazında toplam stok)
- Model bazında beden kırılımı görünümü (ör. 'Dry Fit Koşu Tişörtü Siyah: S ⚫ · M 🔴 · L 🟢 · XL 🟡'). PO faydalı buldu, sonraki aşama. Excel'e model_kodu ve beden kolonları gerekir; mevcut satır görünümü varsayılan kalır.
- POS/ERP entegrasyonu, kullanıcı girişi
- Satış ortalamasının hesaplanması (dosyadaki `gunluk_satis_ortalamasi` olduğu gibi kabul edilir; mevsimsellik/kampanya etkisi yok)

## Bilinen sınırlamalar

- **Sipariş verilen ürün, mal gelene kadar her sabah kırmızı görünmeye devam eder.** Panel yoldaki siparişi bilmez.
- `mevcut_stok` reyon stoğudur; panel depo stoğunu bilmez.
- Veri saklanmaz: sayfa yenilenince örnek veriye döner, dosyanın tekrar yüklenmesi gerekir.
- Dosya okuma kütüphanesi CDN'den geldiği için yükleme için internet gerekir (örnek veri internetsiz de görünür).
- "Bugün" ve tahmini bitiş tarihi, panelin açıldığı bilgisayarın tarihine göre hesaplanır.
- `.xlsx` dosyasında yalnızca **ilk sayfa** okunur.
- Görsel linkleri mağazanın ürün veritabanından, veri sahibi tarafından sağlanır; örnek veride görsel yoktur, kategori ikonu görünür.

---

## Kontrol listesi (teslim öncesi)

- [x] Açılışta örnek veri dolu ve "ÖRNEK VERİ" etiketi görünüyor
- [x] Logo başlıkta düzgün (kırpılmış, bozulmamış)
- [x] 5 kabul kriterinin her biri panelde görülebiliyor
- [x] Tükendi > Kritik > Dikkat > Yeterli sırası doğru, hareketsizler gizli
- [x] PO'nun 3 satırlık CSV'si yüklenince: üçü de Kritik (1.9 ≤ 3, 6.7 ≤ 7, 2.5 ≤ 5)
- [x] Eksik kolonlu / hatalı satırlı dosyada anlaşılır mesaj
- [x] .xlsx dosyası da yükleniyor
- [x] "Ne yapmalı?" kolonu: örnek veride Arch Fit 38 → "Bugün sipariş ver · ~3 gün stoksuz"; Dikkat satırlarında "En geç X gün içinde" + tarih
- [x] Adet kuralı: AYK-9100-45 (stok 1, kalan 20 gün) Kritik ve "reyonda 1 adet kaldı (alt sınır 2)"
- [x] TSH-4000-XL "Kontrol et" olarak görünür, sipariş sayısına girmez; özet "15 ürün"
- [x] Dry Fit Koşu Tişörtü Siyah S/M/L/XL dört ayrı satırda, farklı durumlarda
- [x] Beden gerektiren her modelde tam seri: Kadın ayakkabı 36–41, Erkek 40/41–45(46), Tişört S–XL
- [x] `gorsel_url` olmayan dosya (PO'nun 3 satırı) hatasız yükleniyor; dolu/boş/kırık URL'de görsel veya ikon doğru
- [x] README'de iki tablo + kapsam dışı + bilinen sınırlama + çalıştırma talimatı

Kontroller headless Chrome ile yapıldı: örnek veri, PO CSV'si, `.xlsx`, eksik kolonlu dosya, hatalı satırlı dosya, `;` ayırıcılı CSV, Windows-1254 kodlu CSV, `gorsel_url`'siz dosya, dolu/boş/geçersiz/kırık `gorsel_url`, şablon indir → geri yükle, filtreler ve 390px telefon genişliği.
