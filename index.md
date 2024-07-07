# sp_Blitz

Bu dökümantasyon, SQL Server veritabanında indekslerin analiz edilmesi ve optimize edilmesi için sp_Blitz aracını kullanarak nasıl yapılacağını açıklamaktadır. Bu dökümantasyonda, sp_Blitz'in belirli uyarı tipleri ve bu uyarılara karşı alınması gereken aksiyonlar detaylı bir şekilde açıklanacaktır.

## Gereksinimler 
- SQL Server 2012 veya üstü
- SQL Server Management Studio (SSMS)
- sp_Blitz script'i ya da scriptleri*

## İndirme

Öncelikle, sp_Blitz script'ini indirip SQL Server'a yükleyin.

```bash
GitHub: https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit
zip: https://downloads.brentozar.com/FirstResponderKit.zip
```
## sp_Blitz Script'ini Çalıştırma

İndirdiğiniz dosyada, FirstResponderKit, göreceğiniz script'lerden kullanmak istediğinizi açın.

- Bu aşamada yapılması gereken Connection Security adımında mandotory ayarını optional yapmaktır. Aksi takdirde sertifika hatasıyla karşılaşılması olasıdır.

- Bağlantı başarılı bir şekilde gerçekleştikten sonra yapılması gereken ekrana gelen script'i execute etmektir. Uzun bir query olacaktır, endişe edilecek bir durum gerektirmiyor.

- Yeni bir sorgu penceresi açın ve çalıştırmak istediğiniz sorguyu execute edin.

## İndeksleri Belirleme
Tablodaki mevcut indeksleri belirlemek için aşağıdaki sorguyu çalıştırın

```sql 
SELECT 
    t.name AS TableName, 
    i.name AS IndexName, 
    i.index_id AS IndexID, 
    ic.index_column_id AS IndexColumnID, 
    col.name AS ColumnName
FROM 
    sys.indexes i
INNER JOIN 
    sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
INNER JOIN 
    sys.columns col ON ic.object_id = col.object_id AND ic.column_id = col.column_id
INNER JOIN 
    sys.tables t ON i.object_id = t.object_id
WHERE 
    t.name = '' -- buraya tablo adını girin.
ORDER BY 
    t.name, i.index_id, ic.index_column_id;
```

## İndeks Kullanımını Analiz Etme
İndekslerin kullanım durumunu analiz etmek için aşağıdaki sorguyu çalıştırın
```sql 
SELECT 
    OBJECT_NAME(s.object_id) AS TableName,
    i.name AS IndexName,
    i.index_id AS IndexID,
    s.user_seeks,
    s.user_scans,
    s.user_lookups,
    s.user_updates
FROM 
    sys.dm_db_index_usage_stats s
INNER JOIN 
    sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id
WHERE 
    OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
    AND s.database_id = DB_ID('') -- Veri Tabanı adını girin.
    AND OBJECT_NAME(s.object_id) = '' -- tablo adını girin 
ORDER BY 
    s.user_seeks DESC;
```

## Gereksiz İndeksleri Kaldırılması - Yeniden Yapılandırılması
 Bu bölümde ele alınmayacaktır.

## Önemli Uyarılar
sp_Blitz tarafından raporlanan bazı önemli uyarı tipleri aşağıda listelenmiştir.

* High Priority Issues
* Performance Issues
* Security Issues
* Best Practices

## Alınması Gereken Aksiyonlar
sp_Blitz tarafından raporlanan bazı önemli uyarı tipleri ve bu uyarılara karşı alınması gereken aksiyonlar aşağıda listelenmiştir.

#### 1- High Priority Issues

* Missing Backups : Veritabanınızın yedeklenmediğini veya son yedeklemenin çok eski olduğunu gösterir.\
  
  Yapılması Gerekenler:

    Veritabanının düzenli olarak yedeklendiğinden emin olun.\
    Yedeklemerinizi gözden geçirin ve otomatik yedekleme talimatı oluşturun.

* Long Running Queries : Çok uzun süren sorgular bulunduğunu gösterir.\

   Yapılması Gerekenler:

    Uzun süren sorguları optimize edin.
    Sorguların indeks kullanımını kontrol edin ve gerekirse yeni indeksler oluşturun.

#### 2- Performance Issues

* Missing Indexes: Veritabanınızda eksik indeksler olduğunu gösterir.

  Yapılması Gerekenler:

    sp_Blitz'in önerdiği eksik indeksleri oluşturun.\
    İndeks oluştururken, sorgu performansını artıracak şekilde tasarlayın.

* High CPU Usage: SQL Server'ın yüksek CPU kullanımına sahip olduğunu gösterir.

   Yapılması Gerekenler:

    CPU kullanımını artıran sorguları belirleyin ve optimize edin.\
    Gereksiz işlemleri ve uzun süren sorguları minimize edin.

#### 3- Security Issues
  * Open Permissions: Veritabanınızda gereksiz geniş izinler verildiğini gösterir.

    Yapılması Gerekenler:

     Kullanıcı izinlerini gözden geçirin ve sadece gerekli olan izinleri verin.\
     Gereksiz geniş yetkileri kaldırın.

  * Weak Passwords: Zayıf şifreler kullanıldığını gösterir.

    Yapılması Gerekenler:

    Kullanıcı şifre politikalarını güçlendirin.\
    Şifreleri periyodik olarak değiştirmelerini sağlayın.

#### 4- Best Practices
  * Auto-Growth Settings: Veritabanı otomatik büyüme ayarlarının optimal olmadığını gösterir.

    Yapılması Gerekenler:

      Otomatik büyüme ayarlarını optimize edin (örneğin, yüzde yerine sabit MB değerleri kullanın).\
      Veri ve log dosyalarının büyüme oranlarını uygun şekilde ayarlayın.

  * Database Maintenance: Veritabanı bakım işlemlerinin düzgün yapılmadığını gösterir.

    Yapılması Gerekenler:

    Veritabanı bakım planlarını gözden geçirin ve düzenli bakım yapıldığından emin olun.\
    İndeks yeniden düzenleme, istatistik güncelleme ve bütünlük kontrollerini periyodik olarak gerçekleştirin.


## Sonuç 

Bu dökümantasyon, SQL Server'da sp_Blitz kullanarak indekslerin nasıl analiz edileceğini ve optimize edileceğini açıklamaktadır. Ayrıca, sp_Blitz'in belirli uyarı tipleri ve bu uyarılara karşı alınması gereken aksiyonlar detaylı bir şekilde açıklanmıştır. 

## Kaynaklar

   - https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-usage-stats-transact-sql\
   - https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-indexes-transact-sql\
   - https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit