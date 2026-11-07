# V-Navlungo — Otomatik İade Lojistiği ve Kargo Takip Botu

**Shopify'daki iade taleplerini alıp kargo platformu Navlungo'da iade kodu üreten, iadeyi Shopify'da onaylayıp takip numarasını yazan ve iade kargolarının teslimini izleyen; para ve operasyon kaybına yol açabilecek "çift kod" hatalarına karşı katmanlı korumalarla tasarlanmış bot.**

![.NET Framework](https://img.shields.io/badge/.NET_Framework-4.7.2-512BD4?logo=dotnet&logoColor=white)
![Shopify GraphQL](https://img.shields.io/badge/Shopify-GraphQL_Admin_API-95BF47?logo=shopify&logoColor=white)
![Playwright](https://img.shields.io/badge/Playwright-Chromium-2EAD33?logo=playwright&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?logo=sqlite&logoColor=white)
![SignalR](https://img.shields.io/badge/SignalR-Telegram_k%C3%B6pr%C3%BCs%C3%BC-blue)
![Kaynak](https://img.shields.io/badge/kaynak_kodu-%C3%B6zel_(ticari)-lightgrey)

> **30 saniyede:**
> - **Problem:** Her iade için operatör iki sistem arasında gidip geliyordu: Shopify'da talebi bul → Navlungo'da sipariş ara, iade kodu oluştur, takip numarasını kopyala → Shopify'da onayla ve numarayı yaz. Yavaş, tekrarlı ve hataya açık.
> - **Çözüm:** Shopify tarafı resmî GraphQL API ile, API'si olmayan Navlungo tarafı Playwright tarayıcı otomasyonu ile uçtan uca otomatikleştirildi; teslim takibi ve Telegram'dan uzaktan komut eklendi.
> - **Öne çıkan:** Aynı iade için iki kez kargo kodu üretilmesini imkânsız kılan **iki aşamalı oluşturma + kalıcı iade kodu defteri + tekil kilit** tasarımı.

<p align="center">
  <img src="assets/hero.gif" alt="Bekleyen Shopify iadeleri sırayla işleniyor: Navlungo kodu alınıyor, Shopify iadesi tamamlanıyor" width="900">
</p>

<details>
<summary><b>English summary</b></summary>

V-Navlungo automates e-commerce return logistics between Shopify (official GraphQL Admin API via ShopifySharp) and Navlungo, a shipping platform with no public API (Playwright automation with cookie sessions). It fetches pending return requests, creates the return shipping code in Navlungo, approves the return in Shopify and attaches the tracking number, then tracks return shipments in SQLite and reports deliveries through a SignalR → Telegram bridge that also accepts remote commands. The core design goal is never creating a duplicate return code: a two-phase create (retryable preparation, single-shot creation), a persistent return-code ledger written *before* clicking "create", and a single exclusive browser lock. Source is private.
</details>

---

## Problem

İade süreci, birbirine bağlı olmayan iki sistemde elle yürüyordu. Hata maliyeti asimetrikti:

- Bir iade **atlanırsa** → müşteri bekler, şikâyet eder (geri alınabilir).
- Bir iade için **iki kez kod üretilirse** → gereksiz kargo bedeli, karışan takip numaraları, yanlış numaranın müşteriye gitmesi (geri alınması zor).

Bu yüzden hedef sadece "otomatikleştirmek" değil, **belirsizlik anında güvenli tarafta kalan** bir otomasyon kurmaktı.

## Çözüm

### Akış A — Otomatik iade onayı

```mermaid
sequenceDiagram
    participant B as V-Navlungo
    participant S as Shopify (GraphQL)
    participant L as İade kodu defteri (SQLite)
    participant N as Navlungo (Playwright)
    B->>S: return_status:return_requested siparişleri (sayfalı)
    loop her sipariş
        B->>L: Kayıt var mı?
        alt kod zaten alınmış
            B->>S: yalnız onay + takip no adımını tekrarla
        else kayıt yok
            B->>N: Aşama 1 · ara, tam eşleşen tek satırı seç, modal hazırla (≤2 deneme)
            B->>L: "Oluşturuluyor" yaz (tıklamadan ÖNCE)
            B->>N: Aşama 2 · Oluştur (TEK ATIŞ)
            B->>N: Yeni İADE satırını oku (salt-okunur)
            B->>L: "KodAlındı"
            B->>S: returnApproveRequest + reverseDeliveryCreateWithShipping
            B->>L: "ShopifyTamam"
        end
    end
    B-->>B: Rapor → Telegram
```

### Akış B — Kargo teslim takibi
Aktif iade gönderileri periyodik olarak kazınır (AngleSharp ile tablo ayrıştırma), SQLite'a yazılır; listeden düşen gönderi "teslim edildi" olarak bildirilir.

## Mühendislik kararları

**Çift kod koruması (katmanlı savunma):**
1. **İki aşamalı oluşturma.** Hazırlık adımları güvenle tekrarlanabilir; "Oluştur"a basıldıktan sonra **hiçbir hata için tekrar oluşturulmaz** — yalnızca salt-okunur kod okuma denenir. Okunamazsa sipariş "manuel kontrol"e kilitlenir.
2. **Kalıcı iade kodu defteri.** Tıklamadan *önce* `%ProgramData%` altındaki SQLite deftere yazılır; durum makinesi: `Oluşturuluyor → KodAlındı → ShopifyTamam` / `ManuelKontrol`. Uygulama çökse bile yeniden başlatmada aynı iade için kod üretilmez. Eski sürüm verileri her açılışta tekrarsız göç ettirilir.
3. **Tam eşleşme kuralı.** Arama sonucunda sipariş numarası *tam* eşleşen ve türü iade olmayan **tek** satır yoksa işlem yapılmaz.
4. **Doğru kodu okuma.** Oluşturmadan önce mevcut iade satırlarının anlık görüntüsü alınır; sonra görüntüde olmayan ve göndericisi siparişin alıcısıyla eşleşen **tek** yeni satır okunur. Aynı anda panelden başka biri kod üretse bile yanlış numara Shopify'a yazılmaz.
5. **Tekil kilit.** Tek bir Chromium/oturum sahibi (`BrowserSessionHost`) tüm işleri `SemaphoreSlim(1,1)` ile sıraya koyar; paralel döngüler çift kod üretemez.

**Teslim çıkarımında yanlış pozitife karşı korumalar:** gönderi ancak *art arda iki taramada* görünmezse teslim sayılır; hiç satır okunamazsa ya da liste sayfa sınırına dayanırsa o turda çıkarım yapılmaz; tek turda anormal çok düşüş olursa uyarı; yeniden görünen gönderi için "DÜZELTME" bildirimi.

**Gerçek dünya dersleri:**
- Panel sürekli arka plan isteği attığı için `networkidle` beklemesi asla tamamlanmıyordu (60 sn timeout döngüleri) → `DOMContentLoaded` + kritik öğenin görünürlüğü desenine geçildi.
- Bilinçli olarak ekranlı tarayıcı; pencere CDP ile ekran dışına taşınıp Win32 ile görev çubuğu/Alt+Tab'dan gizlenir. Oturum 4 dakikada bir yoklanıp storage state diske kaydedilir.

**Uzaktan yönetim:** SignalR tabanlı ortak bildirim altyapısı üzerinden Telegram'dan *Kazımayı Başlat / İadeyi Başlat / Teslim Edilenleri Listele / Botu Durdur* komutları.

## Teknoloji yığını

| Alan | Teknolojiler |
|---|---|
| Dil / Platform | C#, .NET Framework 4.7.2, WinForms + DevExpress |
| Shopify | GraphQL Admin API, ShopifySharp |
| Tarayıcı otomasyonu | Microsoft Playwright (Chromium), CDP, Win32 API |
| HTML ayrıştırma | AngleSharp |
| Veri | SQLite |
| Bildirim / komut | SignalR istemcisi → Telegram köprüsü |

## Projenin evrimi

V-Navlungo, daha önce yazdığım [Shopifylungo](https://github.com/furkan-karapinar/Shopifylungo-showcase)'nun yeniden tasarımıdır. İlk sürüm Shopify yönetim panelini de tarayıcıyla kazıyordu; V-Navlungo'da Shopify tarafı **resmî GraphQL API'sine** taşındı, tarayıcı otomasyonu yalnızca API'si olmayan Navlungo'ya bırakıldı ve çift kod korumaları eklendi.

## Ekran görüntüleri

| | |
|---|---|
| ![İade](assets/returns.png) **İade sekmesi** — bekleyen talepler ve sonuçlar | ![Kargo takip](assets/tracking.png) **Kargo takip** |
| ![Defter](assets/ledger.png) **İade kodu kayıtları** — `ShopifyTamam` / `KodAlindi` / `ManuelKontrol` durumları | |

> Görsellerdeki tüm sipariş, müşteri ve takip bilgileri kurgusaldır. Uygulama izole bir demo kopyasında, Navlungo ve Shopify'a bağlanmadan sahte veriyle çalıştırılmıştır; GIF'teki iade turu gerçek defter (SQLite) koduyla yazılan bir simülasyondur.

## Rolüm

Tek geliştirici: süreç analizi, iki sistemin entegrasyonu, hata senaryolarının modellenmesi, güvenlik korumaları ve operasyonel izleme.

## Kaynak kodu hakkında

Bu depo bir **vaka çalışmasıdır**; kod ticari bir işletmeye ait olduğu için paylaşılmamaktadır. Görüşmede canlı demo ve kod incelemesi yapabilirim.

📫 **İletişim:** [GitHub profilim](https://github.com/furkan-karapinar)
