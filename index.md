# sp_Blitz

Bu dökümantasyon, SQL Server veritabanında indekslerin analiz edilmesi ve optimize edilmesi için sp_Blitz aracını kullanarak nasıl yapılacağını açıklamaktadır.

## Gereksinimler

- SQL Server 2012 veya üstü

- SQL Server Management Studio (SSMS)

- sp_Blitz script'i ya da scriptleri

## İndirme

Öncelikle, sp_Blitz script'ini indirip SQL Server'a yükleyin.

- [GitHub üzerinden klonlayarak](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit)

- [zip olarak](https://downloads.brentozar.com/FirstResponderKit.zip)

## sp_Blitz Script'ini Çalıştırma

İndirdiğiniz dosyada, FirstResponderKit içinde göreceğiniz script'lerden kullanmak istediğinizi açın.

- Bu aşamada yapılması gereken Connection Security adımında mandatory ayarını optional yapmaktır. Aksi takdirde sertifika hatasıyla karşılaşılması olasıdır. Veri tabanı adını değiştirmeniz gerekmez. Sadece connection security ayarını mandatory yapın.

- Bağlantı başarılı bir şekilde gerçekleştikten sonra yapılması gereken ekrana gelen script'i execute etmektir. Uzun bir query olacaktır, endişe edilecek bir durum gerektirmiyor.

- Yeni bir sorgu penceresi açın ve çalıştırmak istediğiniz sorguyu execute edin.

- [Bütün sorgulara erişmek için -> readme.md içerisinde olacak](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit/tree/main#sp_blitz-overall-health-check)

## İndeksleri Belirleme

Tablodaki mevcut indeksleri belirlemek için aşağıdaki sorguyu çalıştırın:

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

İndekslerin kullanım durumunu analiz etmek için aşağıdaki sorguyu çalıştırın:

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

## Gereksiz İndekslerin Kaldırılması - Yeniden Yapılandırılması

Bu bölümde ele alınmayacaktır.

## Önemli Uyarılar

SQL Server'da sp_Blitz tarafından tespit edilen belirli indeks uyarıları ve bu uyarıların anlamları ile çözümleri aşağıda detaylandırılmıştır. Her bir uyarının ne anlama geldiğini, bu uyarılara karşı alınması gereken aksiyonları ve gerekli SQL sorgularını içermektedir.

## Alınması Gereken Aksiyonlar

### 1. Multiple Index Personalities: Duplicate Keys

#### Anlamı
Bu uyarı, aynı tablo üzerinde aynı anahtar sütunlarına sahip birden fazla indeks olduğunu gösterir. Bu durum, gereksiz indekslerin varlığına ve kaynakların israfına işaret eder.
### Çözüm

### SQL Sorguları
**İndeksleri Belirleme:**
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
   t.name = '' --tablo adı
ORDER BY
   t.name, i.index_id, ic.index_column_id;
```
**Gereksiz İndeksi Kaldırma:**
```sql
DROP INDEX IndexName ON TableName;
```
## 2. Multiple Index Personalities: Borderline Duplicate Keys
### Anlamı
Bu uyarı, benzer ancak tamamen aynı olmayan anahtar sütunlarına sahip birden fazla indeks olduğunu gösterir. Bu indeksler, gereksiz yere performans kaybına neden olabilir.
### Çözüm

### SQL Sorguları
**İndeksleri Belirleme ve Kullanım Analizi:**
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
   AND OBJECT_NAME(s.object_id) = '' -- tablo adı
ORDER BY
   s.user_seeks DESC;
```
**Gereksiz İndeksi Kaldırma:**
```sql
DROP INDEX IndexName ON TableName;
```
## 3. Indexaphobia: High Value Missing Index
### Anlamı
Bu uyarı, yüksek değere sahip eksik bir indeks olduğunu gösterir. Bu, belirli sorguların performansını ciddi şekilde artırabilecek bir indeksin eksik olduğunu belirtir.
### Çözüm

### SQL Sorguları
**Eksik İndeks Önerilerini Belirleme:**
```sql
SELECT
   migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS improvement_measure,
   OBJECT_NAME(mid.object_id) AS TableName,
   mid.equality_columns,
   mid.inequality_columns,
   mid.included_columns,
   migs.*
FROM
   sys.dm_db_missing_index_group_stats AS migs
INNER JOIN
   sys.dm_db_missing_index_groups AS mig ON migs.group_handle = mig.index_group_handle
INNER JOIN
   sys.dm_db_missing_index_details AS mid ON mig.index_handle = mid.index_handle
WHERE
   mid.database_id = DB_ID('') --DB adı
ORDER BY
   improvement_measure DESC;
```
**Eksik İndeksi Oluşturma:**
```sql
CREATE INDEX IndexName ON TableName (Column1, Column2) INCLUDE (Column3, Column4);
```
## 4. Aggressive Indexes & Under-Indexing: Total Lock Wait Time > 5 Minutes (Row + Page)
### Anlamı
Bu uyarı, toplam kilit bekleme süresinin 5 dakikayı aştığını gösterir. Bu durum, agresif indeksleme veya yetersiz indeksleme nedeniyle ortaya çıkabilir.
### Çözüm

### SQL Sorguları
**Kilit Bekleme Sürelerini Belirleme:**
```sql
SELECT
   request_session_id AS SPID,
   resource_type,
   resource_description,
   request_mode,
   request_status
FROM
   sys.dm_tran_locks
WHERE
   resource_type IN ('OBJECT', 'PAGE', 'RID', 'KEY')
ORDER BY
   request_session_id;
```
**Kilitlenmelere Neden Olan Sorguları Belirleme:**
```sql
SELECT
   blocking_session_id AS BlockingSessionID,
   session_id AS VictimSessionID,
   wait_type,
   wait_time,
   resource_description
FROM
   sys.dm_exec_requests
WHERE
   blocking_session_id <> 0;
```
**Kilitlenmelere Neden Olan İndeksi Kaldırma:**
```sql
DROP INDEX IndexName ON TableName;
```
## 5. Index Hoarder: NC Index with High Writes:Reads
### Anlamı
Bu uyarı, yüksek yazma işlemine sahip ancak okunma işlemleri nispeten düşük olan non-clustered bir indeksi gösterir. Bu indeksler, yazma performansını olumsuz etkileyebilir.
### Çözüm
Yüksek yazma ve düşük okuma oranına sahip indeksleri analiz edip gerekirse kaldırın veya optimize edin.
### SQL Sorguları
**İndeks Kullanım Analizi:**
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
   AND OBJECT_NAME(s.object_id) = 'YourTableName'
ORDER BY
   s.user_updates DESC;
```
**Gereksiz İndeksi Kaldırma:**
```sql
DROP INDEX IndexName ON TableName;
```
## 6. Index Hoarder: Unused NC Index with High Writes
### Anlamı
Bu uyarı, yüksek yazma işlemine sahip ancak kullanılmayan non-clustered bir indeksi gösterir. Bu indeksler, gereksiz kaynak tüketimine neden olur.
### Çözüm
Kullanılmayan ve yüksek yazma yüküne sahip indeksleri belirleyip kaldırın.
### SQL Sorguları
**İndeks Kullanım Analizi:**
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
   AND OBJECT_NAME(s.object_id) = '' --tablo adı
ORDER BY
   s.user_updates DESC;
```
**Gereksiz İndeksi Kaldırma:**
```sql
DROP INDEX IndexName ON TableName;
```
## Sonuç

Bu dökümantasyon, SQL Server'da sp_Blitz kullanarak indekslerin nasıl analiz edileceğini ve optimize edileceğini açıklamaktadır. Ayrıca, sp_Blitz'in belirli uyarı tipleri ve bu uyarılara karşı alınması gereken aksiyonlar detaylı bir şekilde açıklanmıştır.


 **Author: Birkan Cemil ABACI**
 
## Kaynaklar

- [sys.dm_db_index_usage_stats Transact-SQL](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-usage-stats-transact-sql)

- [sys.indexes Transact-SQL](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-indexes-transact-sql)

- [Brent Ozar's SQL Server First Responder Kit](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit)
   - BrentOzarULTD/SQL-Server-First-Responder-Kit: 
     - sp_Blitz, sp_BlitzCache, sp_BlitzFirst, sp_BlitzIndex, and other SQL Server scripts for health checks and performance tuning.
     - sp_Blitz, sp_BlitzCache, sp_BlitzFirst, sp_BlitzIndex, and other SQL Server scripts for health checks and performance tuning. - BrentOzarULTD/SQL-Server-First-Responder-Kit
 