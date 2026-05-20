# D365 Finance & Operations - Kapsamlı Metadata & X++ Referans Kılavuzu

> Her bölüm gerçek XML örnekleri ve gerçek X++ kodları içerir.
> Tarih: 2026-03-13

---

## İçindekiler

1. [AxClass - X++ Sınıf Tanımları](#axclass)
2. [AxTable - Tablo Tanımları](#axtable)
3. [AxTableExtension - Tablo Uzantıları](#axtableextension)
4. [AxForm - Form Tanımları](#axform)
5. [AxFormExtension - Form Uzantıları](#axformextension)
6. [AxEnum - Enumeration Tanımları](#axenum)
7. [AxEnumExtension - Enum Uzantıları](#axenumextension)
8. [AxEdt - Extended Data Type](#axedt)
9. [AxEdtExtension - EDT Uzantıları](#axedtextension)
10. [AxView - SQL View Tanımları](#axview)
11. [AxDataEntityView - Data Entity](#axdataentityview)
12. [AxDataEntityViewExtension - Entity Uzantıları](#axdataentityviewextension)
13. [AxQuery - Sorgu Tanımları](#axquery)
14. [AxQuerySimpleExtension - Sorgu Uzantıları](#axquerysimpleextension)
15. [AxMenuItemAction / Display / Output](#axmenuitem)
16. [AxReport - SSRS Rapor Tanımları](#axreport)
17. [AxSecurityPrivilege / Duty / Role](#security)
18. [AxMenuExtension - Menü Uzantıları](#axmenuextension)
19. [AxLabelFile - Label Dosyaları](#axlabelfile)
20. [Adlandırma Kuralları ve Konvansiyonlar](#naming)
21. [Metadata Klasör Yapısı](#folder)
22. [Genel D365 FO Gelistirme Desenleri](#genel-d365-fo-gelistirme-desenleri)
23. [Standart Tablo Alan Referans Dokumanlari](#standart-tablo-alan-referans-dokumanlari)

---

# AxClass - X++ Sinif Tanimlari


---

## XML Yapisi - Genel Sablon

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxClass xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
	<Name>SinifAdi</Name>
	<SourceCode>
		<Declaration><![CDATA[
public class SinifAdi
{
    // uye degiskenler
}
]]></Declaration>
		<Methods>
			<Method>
				<Name>methodAdi</Name>
				<Source><![CDATA[
    public void methodAdi()
    {
        // govde
    }

]]></Source>
			</Method>
		</Methods>
	</SourceCode>
</AxClass>
```

### Indentation Kurallari (KRITIK)

- `<AxClass>`, `<Name>`, `<SourceCode>` -> 0 tab (root)
- `<Declaration>`, `<Methods>` -> 1 tab
- `<Method>` -> 2 tab
- `<Name>` ve `<Source>` (Method icinde) -> 3 tab
- CDATA icindeki X++ kodu: class declaration 0 girintili, method govdesi 4 bosluk (ya da 1 tab) girintili
- Her `<Source>` CDATA blogu bos satirla kapanmadan once biter: `\n]]>`
- `<Declaration>` CDATA blogu da son `}` kapatilmadan sonra bos satir birakir

### CDATA Sonunda Bos Satir Kurali

**Declaration:**
```
<Declaration><![CDATA[
class SinifAdi
{
}
]]></Declaration>
```
Bos satir yok, direkt `]]>` ile kapanir.

**Method Source:**
```
<Source><![CDATA[
    public void run()
    {
    }

]]></Source>
```
Method CDATA'sinin sonunda her zaman bir bos satir (`\n`) vardir, sonra `]]>`.

---

## Class Tipleri ve Ornekleri

---

### 1. Normal Class (Is Mantigi Sinifi)


**Ogrenilen Kurallar:**
- `class` keyword'u `public` olmadan da kullanilabilir (access modifier opsiyonel)
- parm metodlari `return this;` ile fluent interface uygulayabilir (setter zincirleme icin)
- `RunStatic` static factory/entry method pattern - dis sistemler bu metodu cagirir
- Uye degiskenleri arasindan TAB hizalamalari kullanilabilir (gorsel duzeltme)
- Declaration CDATA icinde son `}` sonrasi bos satir var: `}\n]]>`

---

### 2. Batch Class (RunBaseBatch) - Basit


**Ogrenilen Kurallar:**
- `class` keyword'u (public modifier yok) - RunBaseBatch icin standart
- `#define.CurrentVersion(1)` ve `#define.CurrentList(version)` Declaration CDATA icerisinde tanimlanir
- `pack()` container donusunde `[#CurrentVersion, parmCompany]` seklinde
- `unpack()` basit durum: direkt `[version, parmCompany] = _packedClass;`
- `description()` static + `ClassDescription` donus tipi + label referansi
- `canRunInNewSession()` -> `protected boolean`
- `canGoBatchJournal()` -> `public boolean`
- `main()` metodu basinda `[SysEntryPointAttribute]` attribute
- `prompt()` dialog gosterir, `runOperation()` batch framework'u uzerinden calistirir

---

### 3. Batch Class (RunBaseBatch) - QueryRun ile Gelismis


**Ogrenilen Kurallar:**
- `internal final class` modifier kombinasyonu mumkun
- QueryRun ile batch: `pack()` -> `[queryRun.pack()]`, `unpack()` -> `queryRun = new QueryRun(packedQuery)`
- `new()` korunmali override: `protected void new()` + `super()` cagrisi
- `caption()` -> `public ClassDescription`, `description()` -> `public static ClassDescription` (her ikisi de kullanilabiliyor)
- `runsImpersonated()` -> `true` dondururse servis hesabi olarak calisir
- `canGoBatch()` -> `public boolean` (Batch Manager'dan tetikleme icin)
- `run()` icinde `#OCCRetryCount` macro + `super()` + try/catch (Deadlock + UpdateConflict)
- `retry` keyword'u Deadlock catch icinde kullanilir
- OCC (Optimistic Concurrency Control) icin `handleUpdateConflict()` yardimci metodu
- `showQueryValues()` ve `showQuerySelectButton()` -> dialog'da sorgu goster

---

### 4. Batch Class - internal final + BatchRetryable + #LOCALMACRO


**Ogrenilen Kurallar:**
- `implements BatchRetryable` -> `isRetryable()` metodu zorunludur
- `[Hookable(false)]` attribute + `public final boolean isRetryable()` -> CoC uzantisi engellenir
- `#LOCALMACRO.CurrentList ... #ENDMACRO` -> coklu parm degiskeni olan pack/unpack icin
- `pack()` -> `[#CurrentVersion, #CurrentList]` formatinda
- `unpack()` -> `switch (version)` ile versiyon kontrolu, `default: return false`
- `Main` (buyuk M) de gecerli - case-insensitive degil ama gorulen ornekte buyuk M kullanilmis
- Declaration'da tablo degiskeni: `smmParametersTable smmParametersTable;` (tip adi == degisken adi gecerli)

---

### 5. Controller Class (SrsReportRunController)


**Ogrenilen Kurallar:**
- `extends SrsReportRunController` ayri satira koyulabilir (uzun ise)
- `implements BatchRetryable` extends'in devami olarak
- Declaration CDATA'sinda `{}` bos body -> sadece satir sonuna `{` + bos satir + `}`
- `showDialog()` -> `protected boolean` -> dialog gostermemek icin `return false`
- `preRunModifyContract()` -> `protected void` -> controller calistirilmadan once contract doldurmak icin
- `this.parmArgs().caller() is FormRun` -> caller type kontrolu
- `formDataSourceStr(FormAdi, DataSourceAdi)` -> form datasource string intrinsic
- `this.parmReportContract().parmRdpContract()` -> contract nesnesine erisim
- `ssrsReportStr(RaporAdi, DesignAdi)` -> rapor + tasarim adi (iki parametre)
- `startOperation()` -> main()'de `runOperation()` degil `startOperation()` (SrsReportRunController icin)
- `controller.parmReportName(...)` satir sonu yok, bir alttaki satira devam ediyor

---

### 6. Contract Class (DataContract)


**Ogrenilen Kurallar:**
- `[DataContract]` class attribute -> Declaration CDATA'sinin ilk satiri
- Uye degisken tipi EDT kullanabilir: `LedgerJournalId journalNum`
- parm metodu uzerinde coklu attribute: `[DataMember,\n     SysOperationDisplayOrder('1'),\n     SysOperationControlVisibility(false)]`
- `SysOperationControlVisibility(false)` -> dialog'da bu alani gizler
- `SysOperationDisplayOrder('1')` -> dialog'da sira numarasi (string olarak)
- parm metodu donus tipi uye degiskenin tipi ile ayni

---

### 7. Contract Class - DataContractAttribute (DTO Pattern)


**Ogrenilen Kurallar:**
- `[DataContractAttribute]` (Attribute suffix'li versiyon) ile `[DataContract]` ikisi de gecerli
- `[DataMemberAttribute('SerializationKey')]` -> JSON/OData serializasyon icin key adi verilir
- `private str` -> uye degiskenler `private` keyword ile isaretlenebilir
- DTO (Data Transfer Object) pattern: class keyword basinda public yok
- Tum parm metodlari ayni pattern: `public str parmXxx(str _xxx = xxx)`
- Primitive `str` tipler icin EDT kullanmak zorunda degiliz

---

### 8. DP Class (SrsReportDataProviderPreProcessTempDB)


**Ogrenilen Kurallar:**
- Class'in class keyword'undan once attribute'lar gelir (her biri ayri satir)
- `[SRSReportQueryAttribute(queryStr(...))]` -> sorgu baglantisi
- `[SRSReportParameterAttribute(classStr(...))]` -> contract sinifi baglantisi
- `extends` ayri satira koyulabilir (uzun isimler icin)
- `private` uye degiskenler: `private LedgerJournalTable tmpLedgerJournalTable;`
- Dataset metodu: `[SRSReportDataSetAttribute(tableStr(...))]` attribute ile isaretlenir
- Dataset metodu `select tmpTablo; return tmpTablo;` seklinde
- `processReport()` -> `this.parmDataContract() as ContractClass` ile contract alinir
- `delete_from paymentTmp;` -> her rapor calistirisinda once temizle
- `paymentTmp.clear()` -> buffer temizle, sonra alan doldur, sonra `paymentTmp.insert()`
- Yorum satirlari (`// ----`) ile method gruplari bolunebilir
- `select firstOnly` (buyuk O ile) da gecerli (orn. `firstOnly` vs `firstonly1`)

---

### 9. Extension Class - Table CoC (Chain of Command)


**Ogrenilen Kurallar:**
- `[ExtensionOf(tableStr(SalesTable))]` -> tablo extension icin `tableStr()`
- `final class SinifAdi_Model_Extension` -> `public` yok, `final` zorunlu
- Extension class bos body: `{}`  veya `{\n}` (bos satir olmadan da olabiliyor)
- DataEventHandler ayni extension class icinde tanimlanabilir
- DataEventHandler `public static void` olmak zorunda
- `sender as SalesTable` -> tip donusumu
- `ttsBegin`/`ttsCommit` (buyuk B ve C) de gecerli (kucukle de olur: `ttsbegin`/`ttscommit`)
- `doUpdate()` -> event tetiklemeden guncelle
- modifiedField: `next modifiedField(_fieldId)` ONCE cagirilir (standart once, custom sonra)
- `insert(boolean _skipMarkup)` -> SalesTable'in insert imzasi parametreli
- `update()` -> bos parametre
- display metodu `public` modifier olmadan da tanimlanabilir

---

### 10. Extension Class - Gelismis modifiedField + switch Pattern


**Declaration:**
```x++
[ExtensionOf(tableStr(PurchLine))]
final class PurchLine_MyModel_Extension
{


}
```

**Ogrenilen Kurallar:**
- Declaration icinde bos satirlar olabilir (eksik/yanlis formatlama da derlenir)
- `modifiedField(FieldId _fieldId, boolean _userInput)` -> PurchLine'in iki parametreli imzasi
- `next modifiedField(_fieldId, _userInput)` -> her iki parametre aktarilir
- `switch (_fieldId)` ile `case fieldNum(PurchLine, AlanAdi):` pattern
- `update(boolean dropInvent, boolean updateOrderLineOfDeliverySchedule, boolean updatePurchTableDropShipStatus)` -> cok parametreli override
- `next update(dropInvent, updateOrderLineOfDeliverySchedule, updatePurchTableDropShipStatus)` -> tum parametreler aktarilir
- Buffer degisiklikleri `next update()` ONCESINDE yapilir, baska tablo islemleri SONRASINDA
- `insert()` override'da da ayni kural
- `[SysClientCacheDataMethodAttribute(true)]` -> display metodu icin cache attribute
- `this.orig().AlanAdi` -> kaydin orijinal degerine erisim (degisiklik kontrolu icin)
- `registeredInPurchUnit()` display metodunun CoC override'i: `real result = next registeredInPurchUnit();`
- `insert(boolean dropInvent, boolean findMarkup, ...)` - `next insert()` -> parametresiz next cagrisi gecerli (parametreler opsiyonel aktarilabilir)

---

### 11. Extension Class - Class CoC (SalesInvoiceDP)


**Ogrenilen Kurallar:**
- `[ExtensionOf(classStr(SalesInvoiceDP))]` -> class extension icin `classStr()`
- `protected void` metodlar da override edilebilir (sadece `public` degil)
- Uzun parametre listesi sonraki satira kirilebilir (4 bosluk girintili)
- `next` cagrisi de ayni sekilde uzun parametre listesiyle: `next methodName(p1, p2, p3, p4, p5, p6)`
- `salesInvoiceTmp` -> DP class'inin korunan uye alani, CoC icinden `this.` olmadan erisebilir
- `_custInvoiceTrans.salesLine()` -> relation metodu ile ilgili tabloya erisim

---

### 12. Extension Class - Sadece Static Main (SalesInvoiceController)


**Tam XML:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<AxClass xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
	<Name>SalesInvoiceController_MyModel_Extension</Name>
	<SourceCode>
		<Declaration><![CDATA[
[ExtensionOf(classStr(SalesInvoiceController))]
final class SalesInvoiceController_MyModel_Extension
{
}
]]></Declaration>
		<Methods>
			<Method>
				<Name>main</Name>
				<Source><![CDATA[
    public static void main(Args _args)
    {
        next main(_args);
    }

]]></Source>
			</Method>
		</Methods>
	</SourceCode>
</AxClass>
```

**Ogrenilen Kurallar:**
- `next main(_args)` -> static metodlarda da `next` kullanilir
- Sadece gecis noktasi icin minimal extension: bir metod, sadece `next` cagirisi
- `public static void main(Args _args)` -> static metod override'i

---

### 13. Extension Class - Form CoC


**Ogrenilen Kurallar:**
- `[ExtensionOf(formStr(EcoResProductCreate))]` -> form extension icin `formStr()`
- `this.updateLayout()` -> form layout'u yeniden hesapla
- `conIns(container, pozisyon, eleman)` -> container'a ekleme
- `conLen(container) + 1` -> container sonuna ekleme
- `FormControlEventHandler` form extension class icinde de tanimlanabilir (ayri EventHandler class icinde olmak zorunda degil)
- `ce.cancelSuperCall()` -> kucuk c ile (baska yerde `CancelSuperCall()` buyuk C de goruluyor)

---

### 14. Form DataSource Event Handler (Form Extension Class)


**Ogrenilen Kurallar:**
- `[FormDataSourceEventHandler(formDataSourceStr(FormAdi, DataSourceAdi), FormDataSourceEventType::Initialized)]`
- `FormDataSourceEventType::Initialized` -> datasource ilk kurulumunda
- `FormDataSourceEventType::Activated` -> satir secildigi / aktif oldugunda
- `sender.formRun()` -> FormRun nesnesine erisim
- `sender.cursor()` -> aktif kayda erisim
- `formRun.design().controlName(formControlStr(FormAdi, KontrolAdi))` -> isimle kontrol bulma
- `ctrl as FormRealControl` -> tip donusumu ile ozel kontrol metodlarina erisim
- `realCtrl.noOfDecimalsValue(4)` -> ondalik basamak sayisini runtime'da degistir
- `FormDropDialogButtonControl.enabled(boolean)` -> kontrol aktif/pasif yapma
- Declaration icinde bos satirlar ve yorumlar olabilir

---

### 15. Data Event Handler (Inserted) - Minimal


**Ogrenilen Kurallar:**
- `DataEventType::Inserted` (gecmis zaman - insert SONRASI)
- Tekli satir if: `if(kosul)\n    ifade;` (bos parantez yok)
- `sender` parametresi `Common` tipinde gelir, metod icinde cast edilebilir veya direkt kullanilabilir
- Extension class sadece DataEventHandler metodu icerirse CoC metodu olmak zorunda degil

---

### 16. Normal Class - CLR Interop + API Entegrasyonu


**Ogrenilen Kurallar:**
- `using NamspaceAdi;` -> extern DLL namespace import, Declaration CDATA'sinin en ustune
- `CLRObject` tip -> .NET nesnelerini tutmak icin
- `new CLRObject("System.Collections.Generic.List\`1[System.String]")` -> generic list olusturma
- Coklu degisken tek satir: `str url1, url2, url3;`
- TAB hizalama: `NoYes\t\t\t\t\toverrideRecords;` seklinde gorsel hizlama yapilabilir
- `protected void new(...)` -> constructoru korumak icin (sadece construct() ile nesne olusturulabilir)
- `public static ClassName construct(Common _common = null)` -> factory metodu

**CLR kullanimi ornegi:**
```x++
CLRObject exchangeRates = new CLRObject("...");
CLRObject enumerator = exchangeRates.GetEnumerator();
while (enumerator.MoveNext())
{
    exchangeRatesList += [enumerator.get_Current()];
}
// CLR property erisimleri: obj.get_PropertyName(), obj.set_PropertyName(value)
// CLR metod cagrisi: obj.MethodName(parametre)
```

---

### 17. Normal Class - final + Helper Pattern


**Ogrenilen Kurallar:**
- `public final class` -> kalitim engellenebilir (extends yapilamaz)
- `Common` tipindeki parametrelerde `_parmSourceLine.TableId` ile tablo tespiti yapilabilir
- `tableNum(SalesLine)` -> switch case'de tablo ID karsilastirmasi
- `_parmSourceLine as SalesLine` -> cast sonra initFrom metodu
- `tableId2Name(tableId)` -> tablo ID'den adi al
- `MarkupTrans::lastLineNum(tableId, recId)` -> son line numarasini bul
- Helper class: kalitim yok, uye degisken yok, sadece metodlar

---

### 18. Normal Class - HTTP/Flow Entegrasyonu


```x++
private void callPowerAutomateFlow()
{
    System.Net.HttpWebRequest   request;
    System.Net.HttpWebResponse  response;
    System.IO.StreamWriter      streamWriter;
    System.IO.StreamReader      streamReader;
    str                         jsonBody, responseText;
    System.IO.Stream            requestStream, responseStream;
    System.Text.Encoding        utf8;

    jsonBody = strFmt('{"confirmids":[%1]}', confirmIds);

    try
    {
        utf8            = System.Text.Encoding::get_UTF8();
        request         = System.Net.WebRequest::Create(flowUrl) as System.Net.HttpWebRequest;
        request.set_Method("POST");
        request.set_ContentType("application/json");
        request.set_Accept("application/json");

        requestStream   = request.GetRequestStream();
        streamWriter    = new System.IO.StreamWriter(requestStream, utf8);
        streamWriter.Write(jsonBody);
        streamWriter.Flush();
        streamWriter.Close();

        response        = request.GetResponse() as System.Net.HttpWebResponse;
        responseStream  = response.GetResponseStream();
        streamReader    = new System.IO.StreamReader(responseStream);
        responseText    = streamReader.ReadToEnd();

        info(strFmt("Response: %1", responseText));
    }
    catch (Exception::CLRError)
    {
        System.Exception ex = CLRInterop::getLastException();
        error(strFmt("Hata: %1", ex.ToString()));
    }
}
```

**Ogrenilen Kurallar:**
- `System.Net.HttpWebRequest`, `System.Net.HttpWebResponse` vs. -> .NET namespace'leri dogrudan kullanilabilir
- CLR property set: `request.set_Method("POST")`, `request.set_ContentType("...")`
- CLR property get: `System.Text.Encoding::get_UTF8()`
- `as System.Net.HttpWebRequest` -> CLR downcasting
- `Exception::CLRError` catch -> `CLRInterop::getLastException()` ile exception alinir
- `ex.ToString()` -> CLR string donusumu
- JSON body: `strFmt('{"key":[%1]}', value)` -> tek tirnak kullanilabilir

---

### 19. Normal Class - Vendor Invoice Olusturma (VendInvoiceInfo Pattern)


```x++
private void insertVendInvoiceHeader(PurchTable _purchTableInvoice, boolean _isProduction)
{
    VendInvoiceInfoTable    vendInvoiceInfoTable;
    NumberSeq               numberSeq;
    InvoiceId               purchInvoiceId;

    numberSeq      = NumberSeq::newGetNum(PurchParameters::numRefParmId());
    purchInvoiceId = numberSeq.num();

    vendInvoiceInfoTable.clear();
    vendInvoiceInfoTable.initValue();
    vendInvoiceInfoTable.initFromPurchTable(_purchTableInvoice);
    vendInvoiceInfoTable.Num                    = purchInvoiceId;
    vendInvoiceInfoTable.ParmId                 = purchInvoiceId;
    vendInvoiceInfoTable.TableRefId             = vendInvoiceInfoTable.ParmId;
    vendInvoiceInfoTable.VendInvoiceSaveStatus  = VendInvoiceSaveStatus::Pending;
    vendInvoiceInfoTable.DocumentOrigin         = DocumentOrigin::Manual;
    vendInvoiceInfoTable.TransDate              = systemDateGet();
    vendInvoiceInfoTable.ParmJobStatus          = ParmJobStatus::Waiting;

    if (vendInvoiceInfoTable.validateWrite())
    {
        vendInvoiceInfoTable.insert();
        // alt kayitlar...
    }
    else
    {
        numberSeq.abort();  // numara serisini geri birakmak icin
    }
}
```

**Ogrenilen Kurallar:**
- `NumberSeq::newGetNum(PurchParameters::numRefParmId())` -> ParmId numara serisi
- `numberSeq.num()` -> numara al
- `numberSeq.abort()` -> basarisiz olunca geri birak
- `clear()` -> buffer temizle
- `initValue()` -> varsayilan degerlerle doldur
- `initFromPurchTable()` -> kaynaktan kopyala
- `validateWrite()` -> insert'ten once dogrulama
- `ParmId = Num` ve `TableRefId = ParmId` -> standart kimlik zinciri

---

### 20. Extension Class - SalesFormletterParmData CoC (insertParmLine)


**Ogrenilen Kurallar:**
- `protected void` metodlarin da override edilebilecegi gosterildi
- `_parmLine.(fieldNum(SalesParmLine, DeliverNow))` -> dinamik alan erisimi (fieldNum ile)
- `_parmLine.data(salesParmLine)` -> Common buffer'a veri kopyala
- `salesParmLine.setLineAmount()` -> hesaplamali alan yenileme
- `next insertParmLine(_parmLine)` -> SONUNDA cagrilir (custom logic once, standart sonra)

---

## Kritik Kurallar

### XML Yapisi Kurallari

1. Root element: `<AxClass xmlns:i="http://www.w3.org/2001/XMLSchema-instance">`
2. `<Name>` icerigi dosya adinin `.xml` olmadan aynidir
3. TAB indentation: her icinedeki XML elemaninda bir TAB fazla
4. Her `<Source>` CDATA blogu son satirdan sonra bos satir birakir (`\n]]>`)
5. `<Declaration>` CDATA'da son `}` kapatildiktan sonra bos satir vardır
6. X++ method govdesi 4 bosluk (veya 1 TAB) girintili yazilir

### Class Declaration Kurallari

| Durum | Dogru Yazi |
|---|---|
| Normal class | `class SinifAdi` veya `public class SinifAdi` |
| Final class | `public final class SinifAdi` |
| Internal final | `internal final class SinifAdi` |
| Extends | `public class SinifAdi extends EbeveynSinif` |
| Implements | `... implements ArayuzAdi` (extends'in yanina) |
| Extension | `[ExtensionOf(tableStr/classStr/formStr(...))]` + `final class Adi_Model_Extension` |
| DataContract | `[DataContract]` veya `[DataContractAttribute]` + class |

### CoC Kurallari

- `next` cagrisi **tam olarak bir kez** yapilmalidir
- `next methodName()` -> CoC zincirini devam ettirir
- `next methodName(param1, param2)` -> parametreler aktarilir
- `modifiedField` gibi metodlarda `next` genellikle BASTA cagirilir (standart once)
- `insert`, `update` gibi metodlarda custom buffer degisiklikleri `next` ONCESINDE yapilir
- Baska tablo islemleri (MarkupTrans vb.) `next` SONRASINDA yapilir (deadlock riski)

### Attribute Siralamasi

Class uzerindeki attribute'lar class keyword'undan once gelir:
```x++
[SRSReportQueryAttribute(queryStr(...))]
[SRSReportParameterAttribute(classStr(...))]
public class DPClass extends SrsReportDataProviderPreProcessTempDB
```

Method uzerindeki coklu attribute virgul ve alt satir ile:
```x++
[DataMember,
 SysOperationDisplayOrder('1'),
 SysOperationControlVisibility(false)]
public ReturnType parmXxx(...)
```

---

## Sik Kullanilan X++ Desenleri

### parm Metodu Deseni
```x++
// Basit getter/setter
public DataType parmFieldName(DataType _fieldName = fieldName)
{
    fieldName = _fieldName;
    return fieldName;
}

// Fluent (zincirleme) - service class icin
public ClassName parmFieldName(DataType _fieldName = fieldName)
{
    fieldName = _fieldName;
    return this;
}
```

### construct() Factory
```x++
public static ClassName construct(Common _common = null)
{
    ClassName instance = new ClassName();
    instance.parmCommon(_common);
    return instance;
}
```

### Set ile Tekil Kayit Toplama
```x++
Set purchIdSet = new Set(Types::String);
// while select ile doldurun
purchIdSet.add(purchLine.PurchId);
// kullanim
SetEnumerator enumerator = purchIdSet.getEnumerator();
while (enumerator.moveNext())
{
    PurchId current = enumerator.current();
}
int count = purchIdSet.elements();
```

### Query Range Degeri String Olarak
```x++
QueryBuildRange range = qbds.addRange(fieldNum(PurchTable, PurchId));
range.value("PO-001,PO-002,PO-003");  // virgul ile coklu deger
```

### OCC Retry Count Pattern (Batch)
```x++
public void run()
{
    #OCCRetryCount
    super();

    try
    {
        // islemler
    }
    catch (Exception::Deadlock)
    {
        retry;
    }
    catch (Exception::UpdateConflict)
    {
        if (appl.ttsLevel() == 0)
        {
            if (xSession::currentRetryCount() >= #RetryNum)
            {
                throw Exception::UpdateConflictNotRecovered;
            }
            else
            {
                retry;
            }
        }
        else
        {
            throw Exception::UpdateConflict;
        }
    }
}
```

### validateWrite Kontrolu ile Insert
```x++
if (record.validateWrite())
{
    record.insert();
}
else
{
    numberSeq.abort();  // varsa numara serisini geri birak
}
```

### Display Metodu (Extension Class icinde)
```x++
// Table extension class icinde:
display ReturnType methodName()
{
    return this.iliskiliTablo().alan;
}

// Cache ile:
[SysClientCacheDataMethodAttribute(true)]
public display ReturnType methodName()
{
    return this.someCalculation();
}
```

### Lookup Override (FormControlEventHandler)
```x++
[FormControlEventHandler(formControlStr(FormAdi, KontrolAdi), FormControlEventType::Lookup)]
public static void KontrolAdi_OnLookup(FormControl sender, FormControlEventArgs e)
{
    SysTableLookup sysTableLookup = SysTableLookup::newParameters(tableNum(LookupTablosu), sender);

    sysTableLookup.addLookupfield(fieldNum(LookupTablosu, Alan1));
    sysTableLookup.addLookupfield(fieldNum(LookupTablosu, Alan2));

    sysTableLookup.performFormLookup();

    FormControlCancelableSuperEventArgs cancelArgs = e as FormControlCancelableSuperEventArgs;
    cancelArgs.CancelSuperCall();  // veya cancelArgs.cancelSuperCall()
}
```

### DataEventHandler - Inserting/Updating
```x++
[DataEventHandler(tableStr(TabloAdi), DataEventType::Inserted)]
public static void TabloAdi_onInserted(Common sender, DataEventArgs e)
{
    TabloAdi record = sender as TabloAdi;

    ttsBegin;
    record.selectForUpdate(true);  // ya da doUpdate
    record.AlanAdi = deger;
    record.doUpdate();
    ttsCommit;
}

[DataEventHandler(tableStr(TabloAdi), DataEventType::Updating)]
public static void SinifAdi_onUpdating(Common _sender, DataEventArgs _e)
{
    TabloAdi st = _sender;  // direkt cast, 'as' olmadan da gecerli
}
```

### FormDataSourceEventHandler
```x++
[FormDataSourceEventHandler(formDataSourceStr(FormAdi, DataSourceAdi), FormDataSourceEventType::Activated)]
public static void DataSource_OnActivated(FormDataSource sender, FormDataSourceEventArgs e)
{
    FormRun formRun = sender.formRun();
    TabloAdi record = sender.cursor() as TabloAdi;

    FormControl ctrl = formRun.design().controlName(formControlStr(FormAdi, KontrolAdi));
    FormRealControl realCtrl = ctrl as FormRealControl;

    if (realCtrl)
    {
        realCtrl.noOfDecimalsValue(4);
    }
}
```

---

## InventTrans Marking Deseni (Programatik Marking)

Satis ve satin alma hareketleri arasinda programatik marking yapmak icin `TmpInventTransMark` ve `InventTransMarkCollection` kullanilir.

### Temel Marking Metodu

```x++
public static void markInventoryTransaction(InventTransId _issueId, InventTransId _receiptId)
{
    InventTransId         issueInventTransId   = _issueId;
    InventTransId         receiptInventTransId = _receiptId;

    InventTransOriginId receiptInventTransOriginId = InventTransOrigin::findByInventTransId(receiptInventTransId).RecId;
    InventTrans         receiptInventTrans         = InventTrans::findByInventTransOrigin(receiptInventTransOriginId);

    InventTransOriginId issueInventTransOriginId = InventTransOrigin::findByInventTransId(issueInventTransId).RecId;
    InventTrans         issueInventTrans         = InventTrans::findByInventTransOrigin(issueInventTransOriginId);

    TmpInventTransMark tmpInventTransMark;
    InventTransMarkCollection collection = TmpInventTransMark::markingCollection(
        InventTransOrigin::find(receiptInventTransOriginId),
        receiptInventTrans.inventDim(),
        receiptInventTrans.Qty);

    collection.insertCollectionToTmpTable(tmpInventTransMark);

    select firstonly tmpInventTransMark
        where tmpInventTransMark.InventTransOrigin == issueInventTrans.InventTransOrigin
           && tmpInventTransMark.InventDimId       == issueInventTrans.InventDimId;

    if (tmpInventTransMark.RecId != 0)
    {
        Qty qtyToMark = issueInventTrans.Qty;

        tmpInventTransMark.QtyMarkNow =  qtyToMark;
        tmpInventTransMark.QtyRemain  -= tmpInventTransMark.QtyMarkNow;

        Map mapUpdated = new Map(Types::Int64, Types::Record);
        mapUpdated.insert(tmpInventTransMark.RecId, tmpInventTransMark);

        TmpInventTransMark::updateTmpMark(
            receiptInventTransOriginId,
            receiptInventTrans.inventDim(),
            -qtyToMark,
            mapUpdated.pack());
    }
}
```

### Onemli Notlar
- `_receiptId`: Alim tarafi (PurchLine) InventTransId
- `_issueId`: Cikis tarafi (SalesLine) InventTransId
- `TmpInventTransMark::markingCollection()` receipt tarafindan uygun marking adaylarini toplar
- `TmpInventTransMark::updateTmpMark()` marking islemini gerceklestirir
- `qtyToMark` issue tarafinin miktaridir (negatif deger)
- D365 standart marking API'si kullanilir, dogrudan `InventTrans.MarkingRefInventTransOrigin` guncellenmez

---

## Attribute Referansi

| Attribute | Kullanim | Orn. |
|---|---|---|
| `[ExtensionOf(tableStr(T))]` | Tablo extension class | `tableStr(SalesTable)` |
| `[ExtensionOf(classStr(C))]` | Class extension class | `classStr(SalesInvoiceDP)` |
| `[ExtensionOf(formStr(F))]` | Form extension class | `formStr(EcoResProductCreate)` |
| `[DataEventHandler(tableStr(T), DataEventType::X)]` | Tablo event handler | `DataEventType::Inserted` |
| `[FormControlEventHandler(formControlStr(F,C), FormControlEventType::X)]` | Form kontrol event | `FormControlEventType::Lookup` |
| `[FormDataSourceEventHandler(formDataSourceStr(F,DS), FormDataSourceEventType::X)]` | Form datasource event | `FormDataSourceEventType::Activated` |
| `[SysEntryPointAttribute]` | Giris noktasi | `main()` metodu uzerinde |
| `[DataContract]` | DataContract sinifi | Class uzerinde |
| `[DataContractAttribute]` | DataContract (Attribute suffix) | Class uzerinde (alternatif) |
| `[DataMember]` | DataContract uye | parm metodu uzerinde |
| `[DataMemberAttribute('Key')]` | DataContract uye + key adi | parm metodu uzerinde |
| `[SRSReportQueryAttribute(queryStr(Q))]` | DP sorgu baglantisi | DP class uzerinde |
| `[SRSReportParameterAttribute(classStr(C))]` | DP contract baglantisi | DP class uzerinde |
| `[SRSReportDataSetAttribute(tableStr(T))]` | DP dataset metodu | get* metodu uzerinde |
| `[SysOperationDisplayOrder('N')]` | Dialog siralama | parm metodu uzerinde |
| `[SysOperationControlVisibility(false)]` | Dialog gizleme | parm metodu uzerinde |
| `[Hookable(false)]` | CoC kapatma | `isRetryable()` gibi metodlar |
| `[SysClientCacheDataMethodAttribute(true)]` | Display metod cache | Display metodu uzerinde |

---

## Adlandirma Konvansiyonlari (Gercek Dosyalardan)

| Tip | Pattern | Ornek |
|---|---|---|

---

## Onemli Gozlemler

1. **`class` vs `public class`**: Normal class'larda `public` opsiyonel. Extension class'larda asla `public` yok.

2. **`final` modifier**: Extension class'lar ve bazi yardimci class'lar `final` kullanir.

3. **`internal` modifier**: Internal class'lar ayni modelden disariya acilmaz. `internal final class` kombinasyonu sik gorulur.

4. **Declaration bos body**: `{\n}` (iki satir) veya `{ }` (tek satir) hepsi gecerli.

5. **Method imza uyumu**: CoC'ta override edilen metodun imzasi kaynak ile tam eslesmek zorunda. `PurchLine.update(boolean, boolean, boolean)` override'inda parametreler aynen yazilir.

6. **`next` cagrisinda parametre**: `next update()` seklinde parametresiz cagri da X++'ta gecerli - parametreler opsiyonel gecilir.

7. **Yorum satirlari**: `//` ile tek satir, `/* */` ile blok yorum. DocComment (`/// <summary>`) da kullaniliyor.

8. **`#OCCRetryCount` macro**: RunBaseBatch.run() override'inda kullanilan hazir macro, `#RetryNum` ile birlikte.

9. **`ttsBegin`/`ttsCommit` case insensitivity**: Buyuk veya kucuk harf farki yok - proje icinde karisik kullanim goruldu.

10. **MultiSelectionContext**: Form'dan coklu secim: `_args.multiSelectionContext()`, `.getFirst()`, `.getNext()`.

---

# AxTable - Tablo Tanımları


---

## XML Yapısı - Genel Şablon


---

## Gerçek Tablo Örnekleri

### Örnek 15: Parametre Tablosu (AxTableFieldTime + initValue override)

**Öğrenilenler:**
- `<AxTableFieldTime>` tipi - `TimeOfDay` EDT ile kullanılır - X++'da `time` tipi
- `initValue()` override: kayıt oluşturulurken otomatik ID ataması
- Proje-ozel makro kullanımı (`#MyMacro` gibi)
- `ClusteredIndex` ve `PrimaryIndex` boş string olarak yazılabilir: `<PrimaryIndex></PrimaryIndex>`
- `index hint IdxAdi` - sorgu optimizasyonu için index ipucu
- `like` operatörü wildcard pattern için: `where oldTable.ParmID like newParm`
- `strDel`, `strLen`, `num2Str`, `str2Num` string manipulation fonksiyonları

---

## Alan Tipleri Tam Referansı

### i:type Değerleri ve Kullanım Kuralları

| i:type | X++ Tipi | Ek Zorunlu/Opsiyonel Özellikler | Gerçek Örnek |
|--------|----------|--------------------------------|--------------|
| `AxTableFieldString` | `str` | `<StringSize>` opsiyonel (varsayılan 10) | `CargoCompany`, `AttributeName` |
| `AxTableFieldReal` | `real` | - | `SalesExchangeRate`, `AmountAED`, `LineNumber` |
| `AxTableFieldEnum` | `enum` | `<EnumType>` zorunlu | `OverrideRecords`, `SalesStatus`, `AttributeType` |
| `AxTableFieldTime` | `time` | `TimeOfDay` EDT ile kullanılır | `BeginTime`, `EndTime` |

### KRITIK: EDT - Alan Tipi Uyum Kuralı

Gerçek projeden doğrulanan uyumlar:

| EDT | Baz Tip | Doğru Alan Tipi | Proje Örneği |
|-----|---------|-----------------|--------------|
| `NoYesId` | enum | `AxTableFieldEnum` | Tüm tablolarda yaygın |
| `TimeOfDay` | time | `AxTableFieldTime` | Parametre tablolari |

**DIKKAT - LineNumber EDT real tipidir:**
```xml
<!-- DOGRU: LineNumber EDT real bazlidir -->
<AxTableField xmlns="" i:type="AxTableFieldReal">
    <Name>LineNumber</Name>
    <ExtendedDataType>LineNumber</ExtendedDataType>
    <IgnoreEDTRelation>Yes</IgnoreEDTRelation>
</AxTableField>

<!-- YANLIS: Int ile kullanilamaz -->
<AxTableField xmlns="" i:type="AxTableFieldInt">
    <Name>LineNumber</Name>
    <ExtendedDataType>LineNumber</ExtendedDataType>
</AxTableField>
```

---

## İndeks Tipleri ve Kuralları

### AlternateKey=Yes (Unique Index)

```xml
<AxTableIndex>
    <Name>CargoCompanyIDx</Name>
    <AlternateKey>Yes</AlternateKey>
    <Fields>
        <AxTableIndexField>
            <DataField>CargoCompany</DataField>
        </AxTableIndexField>
    </Fields>
</AxTableIndex>
```

- Benzersizlik garantisi sağlar (UNIQUE index)
- `<ReplacementKey>CargoCompanyIDx</ReplacementKey>` ile tabloya bağlanır
- Birden fazla alan içerebilir (composite unique key)

### AllowDuplicates=Yes (Non-Unique Index)

```xml
<AxTableIndex>
    <Name>ItemIdIDx</Name>
    <AllowDuplicates>Yes</AllowDuplicates>
    <Fields>
        <AxTableIndexField>
            <DataField>ItemId</DataField>
        </AxTableIndexField>
    </Fields>
</AxTableIndex>
```

- Sadece performans ve arama için
- Benzersizlik garantisi yok

### Normal Index (ne AlternateKey ne AllowDuplicates)

```xml
<AxTableIndex>
    <Name>PrimaryIndex</Name>
    <Fields>
        <AxTableIndexField>
            <DataField>Identifier</DataField>
        </AxTableIndexField>
    </Fields>
</AxTableIndex>
```

- Varsayılan: unique olmayan (AllowDuplicates=No varsayılan)
- `AlternateKey` belirtilmemiş = No

### IsSystemGenerated Index (Staging tabloları)

```xml
<AxTableIndex>
    <Name>StagingIdx</Name>
    <AlternateKey>Yes</AlternateKey>
    <IsSystemGenerated>Yes</IsSystemGenerated>
    <Fields>
        <AxTableIndexField>
            <DataField>DefinitionGroup</DataField>
        </AxTableIndexField>
        <AxTableIndexField>
            <DataField>ExecutionId</DataField>
        </AxTableIndexField>
        <AxTableIndexField>
            <DataField>ItemId</DataField>
        </AxTableIndexField>
    </Fields>
</AxTableIndex>
```

### Index İçinde Sistem Alan Kullanımı

```xml
<!-- CreatedBy sistem alanı index içinde kullanılabilir -->
<AxTableIndex>
    <Name>ItemDimIDx</Name>
    <AllowDuplicates>Yes</AllowDuplicates>
    <Fields>
        <AxTableIndexField>
            <DataField>CreatedBy</DataField>
        </AxTableIndexField>
        <AxTableIndexField>
            <DataField>InventTransId</DataField>
        </AxTableIndexField>
    </Fields>
</AxTableIndex>
```

---

## Relation Tipleri ve Kuralları

### Standart Field Relation

```xml
<AxTableRelation>
    <Name>MarkupAutoLine</Name>
    <OnDelete>Restricted</OnDelete>
    <RelatedTable>MarkupAutoLine</RelatedTable>
    <Constraints>
        <AxTableRelationConstraint xmlns=""
            i:type="AxTableRelationConstraintField">
            <Name>MarkupCode</Name>
            <Field>MarkupCode</Field>
            <RelatedField>MarkupCode</RelatedField>
        </AxTableRelationConstraint>
    </Constraints>
</AxTableRelation>
```

### ForeignKey Relation (Extension'larda)


- `AxTableRelationForeignKey` i:type ile işaretlenir
- `<Index>RecId</Index>` bağlı tablonun hangi indeksini kullandığını belirtir

### RelatedFixed Constraint (Sabit değer koşulu)


- `AxTableRelationConstraintRelatedFixed`: ilişkili tablodaki alanın sabit değerine göre filtre
- `<ValueStr>` X++ enum değeri string olarak yazılır

### Validate=No Relation


- `<Validate>No</Validate>` - referential integrity kontrolü yapılmaz

### OnDelete Değerleri

| Değer | Davranış | Kullanım |
|-------|----------|----------|
| `Cascade` | Bağlı kayıtlar otomatik silinir | Parent-child ilişki |
| `Restricted` | Bağlı kayıt varken silme engellenir | Referans tabloları |
| (belirtilmemiş) | Kısıtlama yok | View ilişkileri |

---

## Field Groups - Zorunlu ve Opsiyonel

### Her tabloda bulunması gereken 5 standart grup:

```xml
<FieldGroups>
    <AxTableFieldGroup>
        <Name>AutoReport</Name>
        <Fields>
            <!-- Raporda gösterilecek alanlar -->
        </Fields>
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoLookup</Name>
        <Fields />  <!-- Genellikle boş -->
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoIdentification</Name>
        <AutoPopulate>Yes</AutoPopulate>
        <Fields>
            <!-- Tekil kimlik alanları (opsiyonel ama önerilen) -->
        </Fields>
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoSummary</Name>
        <Fields />  <!-- Genellikle boş -->
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoBrowse</Name>
        <Fields />  <!-- Genellikle boş -->
    </AxTableFieldGroup>
</FieldGroups>
```

### Özel gruplar:

| Grup Adı | Kullanım |
|----------|----------|
| `Grid` | Form grid kontrolü için field group |
| `Calculated` | Display metodlarını içeren grup |

### Grup Notları (Gerçek Projeden):

- `AutoReport`: her zaman ana alanlar içerir (TempDB hariç boş bırakılabilir)
- `AutoIdentification` + `<AutoPopulate>Yes</AutoPopulate>`: her zaman gerekli, alanlar opsiyonel
- `<IsSystemGenerated>Yes</IsSystemGenerated>`: staging tablolarında sistem grupları bu attribute ile işaretlenir

---

## Table Properties

### SaveDataPerCompany

| Değer | Anlam | Ne zaman kullanılır |
|-------|-------|---------------------|

**Kural:** `SaveDataPerCompany=No` olan tablolar genellikle:
- Marketplace, ürün öznitelik gibi şirket bağımsız veri
- Entegrasyon parametreleri (API URL, kullanıcı bilgileri)
- Global lookup/setup tabloları

### TableType

| Değer | Anlam | Özellikler |
|-------|-------|-----------|
| (belirtilmemiş) | Normal kalıcı tablo | Veritabanında fiziksel tablo |
| `TempDB` | Geçici veritabanı tablosu | SSRS raporu, session bazlı veri |
| (Staging) | DMF staging | `<TableGroup>Staging</TableGroup>` ile birlikte |

### AllowRowVersionChangeTracking

- `<AllowRowVersionChangeTracking>Yes</AllowRowVersionChangeTracking>`: satır versiyonu değişim takibi
- Performans sorgularında kullanılır
- Gerçek projede hemen her tabloda bulunur

### CacheLookup

- `<CacheLookup>EntireTable</CacheLookup>`: tüm tablo önbelleğe alınır
- Küçük ve sık okunan setup/lookup tablolarında kullanılır

### CreatedBy / ModifiedBy / CreatedDateTime / ModifiedDateTime

```xml
<CreatedBy>Yes</CreatedBy>
<CreatedDateTime>Yes</CreatedDateTime>
<ModifiedBy>Yes</ModifiedBy>
<ModifiedDateTime>Yes</ModifiedDateTime>
```

- Audit trail için - kim/ne zaman oluşturdu/düzenledi
- TempDB ve basit lookup tablolarında genellikle atlanır

### TitleField1 / TitleField2

```xml
<TitleField1>CargoCompany</TitleField1>
<TitleField2>CargoCompanyDescription</TitleField2>
```

- Form başlığında gösterilen alanlar
- Lookup tablolarında ilk alan genellikle `kod`, ikinci alan `açıklama`

### ReplacementKey

```xml
<ReplacementKey>CargoCompanyIDx</ReplacementKey>
```

- Tablonun natural key'i - AlternateKey=Yes olan indeks adı
- Form navigasyonunda ve FK ilişkilerinde kullanılır
- Her tabloda belirtilmesi önerilen

### ConfigurationKey

```xml
<ConfigurationKey>LogisticsBasic</ConfigurationKey>
```

- Staging tablolarında görülür
- Modül aktivasyon anahtarı

## Invent ile Baslayan Standart Tablo Aciklamalari

Bu bolum, standart system metadata altinda `AxTable/Invent*.xml` olarak bulunan tablo ailesinin hizli ozetidir.

**Detayli referanslar:**
- Alan bazli tam referans (569 tablo, 14800+ satir): [docs/MdDocuments/Invent_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Invent_Tablolari_Alan_Referansi_20260403.md)
- Cekirdek operasyon akis rehberi: [docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md](docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md)

- Kaynak surum: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Kapsam: 569 benzersiz standart tablo
- Not: `Invent*` wildcard'i nedeniyle `InventInventory*`, `InventCost*` ve `InventCDS*` gibi alt aileler de bu kapsama dahildir.

### Kategori Ozeti

| Kategori | Adet | Aciklama |
|---|---:|---|
| Core | 30 | Cekirdek stok ana verisi, on-hand, journal ve transfer hareketleri |
| Other | 230 | Maliyetleme, kalite, raporlama, data service ve destek tablolari |
| History | 13 | Gecmis/snapshot kopyalari |
| Tmp | 101 | Gecici isleme ve raporlama tamponlari |
| Localization | 49 | Ulke/bolge veya public sector genisletmeleri |
| Staging | 146 | DMF / entity staging aktarim tablolari |

### Cekirdek Tablo Ailesi

| Tablo | Aciklama |
|---|---|
| `InventTable` | Urunun sirket bazli stok, planlama, maliyet, depo ve tedarik ayarlari |
| `InventDim` | Site, depo, batch, serial ve varyant bazli stok boyutu kombinasyonlari |
| `InventSum` | Boyut bazinda on-hand miktar ozeti |
| `InventTrans` | Tum stok hareketlerinin fiziksel ve finansal hareket kaydi |
| `InventBatch` | Batch/lot ana verisi ve raf omru/izlenebilirlik bilgileri |
| `InventParameters` | Stok yonetimi modulu sistem parametreleri |
| `InventLocation` | Depo/warehouse tanimlari |
| `InventSite` | Site/tesis tanimlari |
| `InventItemGroup` | Item group bazli stok ve muhasebe davranis gruplari |
| `InventModelGroup` | Rezervasyon, maliyetleme ve stok modeli davranisi |
| `InventJournalName` | Journal tipleri icin adlandirma ve varsayilanlar |
| `InventJournalTable` | Stok journal basliklari |
| `InventJournalTrans` | Stok journal satirlari |
| `InventTransferTable` | Transfer siparisi basligi |
| `InventTransferLine` | Transfer siparisi satirlari |
| `InventTransferParmTable` | Transfer posting sureci baslik parametreleri |
| `InventTransferParmLine` | Transfer posting sureci satir parametreleri |
| `InventQualityOrderTable` | Kalite emri basligi |
| `InventSettlement` | Stok kapama/settlement baglantilari |
| `InventCostTrans` | Stok maliyetleme detay hareketleri |

### Diger Onemli Tablo Aileleri

| Pattern / Aile | Aciklama |
|---|---|
| `InventInventory*Staging` | DMF ve entity aktarimlari icin staging tablolari |
| `InventAging*` | Stok yaslandirma raporlari ve hesaplama depolari |
| `InventCost*` | Valuation, cost list, closing ve cost transaction destek ailesi |
| `InventClosing*` | Inventory close ve close log sureci |
| `InventCDSInventoryOnHand*` | CDS/on-hand veri servisi istek, cevap ve staging nesneleri |
| `InventCountingReasonCode*` | Sayim neden kodlari, gruplari ve politikalar |
| `InventFiscalLIFO*_RU` | Rusya maliyetleme / fiscal LIFO lokalizasyonu |
| `InventBailee*_RU` | Rusya emanet/bailee lokalizasyonu |
| `*History` | Gecmis/snapshot kopyalari |
| `*Tmp` | Gecici rapor, UI veya isleme tampon tablolari |
| `*Staging` | DMF / OData / entity aktarimlari icin staging tablolari |

### Invent Cekirdek Operasyon Akisi

**Tipik iliski zinciri:**

`InventTable` -> `InventDim` -> `InventSum`

`InventTable` -> `InventTrans` -> `InventDim`

`InventTrans` -> `InventBatch` (batch boyutu varsa `InventDim` uzerinden)

`InventJournalTable` -> `InventJournalTrans`

`InventTransferTable` -> `InventTransferLine` -> `InventTrans`

`InventTrans` -> `InventSettlement` / `InventCostTrans`

**Hangi senaryoda hangi tabloya bakilir?**

| Senaryo | Ilk bakilacak tablo | Sonra kontrol edilecekler |
|---|---|---|
| On-hand neden bu kadar? | `InventSum` | `InventDim`, `InventTrans` |
| Hangi hareket bu bakiyeyi olusturdu? | `InventTrans` | `InventDim`, ilgili source document |
| Batch bilgisi / raf omru ne? | `InventBatch` | `InventDim`, `InventTrans` |
| Journal ne post etti? | `InventJournalTable` | `InventJournalTrans`, `InventTrans` |
| Transferte ne oldu? | `InventTransferTable` | `InventTransferLine`, `InventTrans` |
| Cost close sonucu neye yansidi? | `InventSettlement` | `InventCostTrans`, `InventTrans` |
| Urunun stok davranisi neden boyle? | `InventTable` | `InventParameters`, model/item group alanlari |

**Hizli notlar:**
- On-hand analizi icin ilk tablo genelde `InventSum`, kok neden analizi icin asil tablo `InventTrans` olur.
- Batch baglantisi dogrudan her tabloda yoktur; cogu zaman `InventDim` uzerinden gidilir.
- Maliyet farklari ve close etkileri canli hareketten cok `InventSettlement` ve `InventCostTrans` tarafinda okunur.
- Journal ve transfer senaryolarinda belge tablosu ile hareket tablosunu birlikte okumak gerekir.

## Purch ile Baslayan Standart Tablo Aciklamalari

Bu bolum, standart system metadata altinda `AxTable/Purch*.xml` olarak bulunan tablo ailesinin hizli ozetidir.

**Detayli referanslar:**
- Alan bazli tam referans (368 tablo, 11000+ satir): [docs/MdDocuments/Purch_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Purch_Tablolari_Alan_Referansi_20260403.md)
- Cekirdek operasyon akis rehberi: [docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md](docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md)

- Kaynak surum: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Kapsam: 368 benzersiz standart tablo
- Not: `Purch*` wildcard'i nedeniyle `PurchaseOrderResponse*` gibi `Purchase...` adli tablolar da bu aileye dahildir.

### Kategori Ozeti

| Kategori | Adet | Aciklama |
|---|---:|---|
| Core | 23 | Cekirdek satin alma is verisi ve posting parametreleri |
| Other | 113 | Yardimci is akislari, policy, response, copilot ve baglanti tablolari |
| History | 28 | Gecmis/snapshot kopyalari |
| Tmp | 54 | Gecici isleme ve raporlama tamponlari |
| Localization | 28 | Ulke/bolge veya public sector genisletmeleri |
| Staging | 122 | DMF / entity staging aktarim tablolari |

### Cekirdek Tablo Ailesi

| Tablo | Aciklama |
|---|---|
| `PurchTable` | Satin alma siparisi basligi; vendor, para birimi, teslimat, tarih, durum ve genel ticari ayarlar |
| `PurchLine` | Satin alma siparisi satiri; urun/hizmet, miktar, fiyat, stok, vergi, teslimat ve muhasebe detaylari |
| `PurchParameters` | Satin alma modulu sistem parametreleri, numara serileri ve davranis ayarlari |
| `PurchParmTable` | Posting sureci icin baslik seviyesi satin alma parametreleri |
| `PurchParmLine` | Posting sureci icin satir seviyesi satin alma parametreleri |
| `PurchParmSubTable` | Posting alt kirilimlari icin baslik tampon parametreleri |
| `PurchParmSubLine` | Posting alt kirilimlari icin satir tampon parametreleri |
| `PurchParmUpdate` | Satin alma posting / update davranisi secimleri |
| `PurchDeliverySchedule` | Satin alma satirlarinin parcali teslimat planlari |
| `PurchAgreementHeader` | Satin alma anlasmasi basligi |
| `PurchAgreementHeaderDefault` | Satin alma anlasmasi varsayilan degerleri |
| `PurchAgreementActivity` | Satin alma anlasmasina bagli aktivite / kullanim kayitlari |
| `PurchAgreementCertification` | Satin alma anlasmasina bagli sertifikasyon kayitlari |
| `PurchAgreementSubcontractor` | Satin alma anlasmasi ile alt yuklenici iliskisi |
| `PurchLineOrigin` | Satin alma satirinin olusum kaynagi ve referans zinciri |
| `PurchOrderRFQLineReference` | PO satiri ile RFQ satiri arasindaki referans |
| `PurchConfirmationRequestJour` | Satin alma onay talebi journal kayitlari |
| `PurchEncumbranceSummary` | Satin alma tarafi encumbrance / butce baglama ozeti |
| `PurchJournalAutoSummary` | Satin alma journal ozetleme davranisi icin yardimci veri |
| `PurchPool` | Satin alma siparis havuz tanimi |
| `PurchPrepayTable` | Satin alma on odeme kayitlari |
| `PurchPriceTolerance` | Satin alma fiyat toleransi kurallari |
| `PurchPurchaseOrderHeader` | Satin alma siparisi basligini entegrasyon / entity odakli tasiyan standart tablo |

### Diger Onemli Tablo Aileleri

| Pattern / Aile | Aciklama |
|---|---|
| `PurchaseOrderResponse*` | Vendor collaboration tarafinda satin alma siparisi yanit / kabul / ret sureci |
| `PurchCopilot*` | Satin alma Copilot senaryolari icin e-posta esleme, gorev, bilgi tabani ve inceleme tamponlari |
| `PurchReqAuthorization*` | Satin alma talebi yetkilendirme kapsam ve organizasyon kirilimlari |
| `PurchReApprovalPolicyRule*` | Yeniden onay kurallari ve rule field tanimlari |
| `PurchBook*_RU` | Rusya satin alma defteri ve KDV lokalizasyonu |
| `PurchImportDeclaration_BR` | Brezilya ithalat beyan ve dis ticaret lokalizasyonu |
| `PurchCommitment*_PSN` | Public Sector satin alma taahhut/butce yapilari |
| `*History` | Belge veya satirlarin gecmis/snapshot kopyalari |
| `*Tmp` | Gecici rapor, UI veya isleme tampon tablolari |
| `*Staging` | DMF / OData / entity aktarimlari icin staging tablolari |

### Purch Cekirdek Operasyon Akisi

**Tipik iliski zinciri:**

`PurchTable` -> `PurchLine` -> `InventDimId` / `InventTransId`

`PurchTable` -> `PurchParmTable` -> `PurchParmLine`

`PurchLine` -> `PurchDeliverySchedule`

`PurchLine` -> `PurchLineOrigin`

`PurchTable` veya `PurchLine` -> `PurchAgreementHeader`

**Hangi senaryoda hangi tabloya bakilir?**

| Senaryo | Ilk bakilacak tablo | Sonra kontrol edilecekler |
|---|---|---|
| Siparis neden bu durumda? | `PurchTable` | `PurchLine`, `PurchParmUpdate` |
| Satir neden farkli miktarda receipt oldu? | `PurchLine` | `PurchParmLine`, `PurchDeliverySchedule` |
| Postingte ne oldu? | `PurchParmTable` | `PurchParmLine`, `PurchParmUpdate` |
| Satirin kaynagi ne? | `PurchLineOrigin` | `PurchLine`, ilgili source document |
| Anlasma baglantisi var mi? | `PurchAgreementHeader` | `PurchLine`, `PurchTable` |
| Vendor confirmation gecmisi ne? | `PurchConfirmationRequestJour` | `PurchaseOrderResponse*` ailesi |

**Hizli notlar:**
- `PurchTable` basliktir; operasyonun asil ayrintisi cogu zaman `PurchLine` tarafindadir.
- Product receipt / invoice farklarinda dogrudan canli satira bakmak yetmez; `PurchParm*` tamponlari kritik olur.
- Teslimat farklari varsa `PurchDeliverySchedule` mutlaka kontrol edilmelidir.
- Kaynak dokuman zinciri icin `PurchLineOrigin`, RFQ baglantisi icin `PurchOrderRFQLineReference` yardimci olur.

## Sales ile Baslayan Standart Tablo Aciklamalari

Bu bolum, standart system metadata altinda `AxTable/Sales*.xml` olarak bulunan tablo ailesinin hizli ozetidir.

**Detayli referanslar:**
- Alan bazli tam referans (267 tablo, 11100+ satir): [docs/MdDocuments/Sales_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Sales_Tablolari_Alan_Referansi_20260403.md)

- Kaynak surum: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Kapsam: 267 benzersiz standart tablo
- Not: `Sales*` wildcard'i nedeniyle `SalesInvoice*`, `SalesOrder*` ve `SalesQuotation*` gibi alt aileler de bu kapsama dahildir.

### Kategori Ozeti

| Kategori | Adet | Aciklama |
|---|---:|---|
| Core | 15 | Cekirdek satis siparisi, teklif ve posting parametreleri |
| Other | 49 | Belge, fatura, teyit, fiyatlama ve destek tablolari |
| History | 13 | Gecmis, arsiv ve snapshot kopyalari |
| Tmp | 33 | Gecici isleme ve raporlama tamponlari |
| Localization | 31 | Ulke/bolge veya public sector genisletmeleri |
| Staging | 126 | DMF / entity staging aktarim tablolari |

### Cekirdek Tablo Ailesi

| Tablo | Aciklama |
|---|---|
| `SalesTable` | Satis siparisi basligi; musteri, para birimi, teslimat ve ticari ayarlar |
| `SalesLine` | Satis siparisi satiri; urun, miktar, fiyat, teslimat ve stok baglantilari |
| `SalesParameters` | Satis modulu sistem parametreleri ve davranis ayarlari |
| `SalesParmTable` | Posting sureci icin baslik seviyesi satis parametreleri |
| `SalesParmLine` | Posting sureci icin satir seviyesi satis parametreleri |
| `SalesParmSubTable` | Posting alt kirilimlari icin baslik tampon parametreleri |
| `SalesParmSubLine` | Posting alt kirilimlari icin satir tampon parametreleri |
| `SalesParmUpdate` | Satis posting / update davranisi secimleri |
| `SalesQuotationTable` | Satis teklif basligi |
| `SalesQuotationLine` | Satis teklif satiri |
| `SalesQuotationParmTable` | Teklif posting/confirm baslik parametreleri |
| `SalesQuotationParmLine` | Teklif posting/confirm satir parametreleri |
| `SalesAgreementHeader` | Satis anlasmasi basligi |
| `SalesDeliverySchedule` | Satis satirlarinin parcali teslimat planlari |
| `SalesPool` | Satis siparisi havuz tanimi |

### Diger Onemli Tablo Aileleri

| Pattern / Aile | Aciklama |
|---|---|
| `SalesInvoice*` | Satis faturasi, fatura detaylari ve destek/staging ailesi |
| `SalesOrderHeaderV*` / `SalesOrderLineV*` | Entity odakli siparis baslik/satir aileleri |
| `SalesConfirm*` | Teyit / confirmation sureci tablolari |
| `SalesPackingSlip*` | Irsaliye / packing slip sureci tablolari |
| `SalesTableHistory` / `SalesLineHistory` | Gecmis ve snapshot kopyalari |
| `*Tmp` | Gecici rapor, UI veya isleme tampon tablolari |
| `*Staging` | DMF / OData / entity aktarimlari icin staging tablolari |

## Vend ile Baslayan Standart Tablo Aciklamalari

Bu bolum, standart system metadata altinda `AxTable/Vend*.xml` olarak bulunan tablo ailesinin hizli ozetidir.

**Detayli referanslar:**
- Alan bazli tam referans: [docs/MdDocuments/Vend_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Vend_Tablolari_Alan_Referansi_20260403.md)

- Kaynak surum: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Kapsam: 411 benzersiz standart tablo
- Not: `Vend*` wildcard'i nedeniyle `VendInvoice*`, `VendPaym*`, `VendRFQ*` ve benzeri alt aileler de bu kapsama dahildir.

### Kategori Ozeti

| Kategori | Adet | Aciklama |
|---|---:|---|
| Core | 16 | Cekirdek vendor, AP, product receipt ve fatura tablolari |
| Other | 159 | Vendor invoice, odeme, RFQ ve destek tablolari |
| History | 6 | Gecmis, arsiv ve snapshot kopyalari |
| Tmp | 90 | Gecici isleme ve raporlama tamponlari |
| Localization | 57 | Ulke/bolge veya public sector genisletmeleri |
| Staging | 83 | DMF / entity staging aktarim tablolari |

### Cekirdek Tablo Ailesi

| Tablo | Aciklama |
|---|---|
| `VendTable` | Tedarikci ana verisi; hesap, odeme, teslimat ve genel ticari ayarlar |
| `VendParameters` | Accounts Payable / vendor modulu sistem parametreleri |
| `VendGroup` | Tedarikci grup tanimlari |
| `VendBankAccount` | Tedarikci banka hesaplari |
| `VendPaymModeTable` | Odeme sekli / paym mode tanimlari |
| `VendInvoiceJour` | Vendor fatura basligi |
| `VendInvoiceInfoTable` | Pending vendor invoice basligi |
| `VendInvoiceInfoLine` | Pending vendor invoice satirlari |
| `VendInvoiceTrans` | Vendor fatura satirlari |
| `VendInvoiceMatchingLine` | Fatura esleme satir kayitlari |
| `VendPackingSlipJour` | Product receipt / packing slip basligi |
| `VendPackingSlipTrans` | Product receipt / packing slip satirlari |
| `VendTrans` | Vendor finansal/AP hareketleri |
| `VendTransOpen` | Acik vendor hareketleri |
| `VendSettlement` | Vendor settlement baglantilari |
| `VendRFQJour` | Vendor request for quotation basligi |

### Diger Onemli Tablo Aileleri

| Pattern / Aile | Aciklama |
|---|---|
| `VendInvoice*` | Vendor invoice, matching, save status ve destek ailesi |
| `VendPaym*` | Vendor odeme ve odeme yurutu aileleri |
| `VendRFQ*` | Vendor teklif isteme ve cevap sureci |
| `VendProductReceipt*` | Product receipt ve ilgili destek/staging ailesi |
| `VendTrans*` | Vendor hareket, acik bakiye ve settlement ailesi |
| `*Tmp` | Gecici rapor, UI veya isleme tampon tablolari |
| `*Staging` | DMF / OData / entity aktarimlari icin staging tablolari |

---

## Standart Metodlar (Gerçek Örneklerle)

### validateWrite() - Koşullu Validasyon

```x++
public boolean validateWrite()
{
    boolean ret;

    ret = super();

    if(this.SalesStatus != SalesStatus::Invoiced && this.SalesStatus != SalesStatus::Delivered)
    {
        ret = false;
        error(strFmt("%1 statusü %2 kaydı için yanlış", this.SalesStatus, this.DocumentId));
    }

    return ret;
}
```

### Display Method - Tablo Üzerinde


---

## KRITIK KURALLAR

### AssetClassification - Ne Zaman Gerekli?

**Gerçek proje analizi:**

- `<AssetClassification>Customer Content</AssetClassification>` - müşteri verisi içeren alanlarda
- `<AssetClassification>System Metadata</AssetClassification>` - sistem metadata alanlarında (TempDB'de)

**Sonuç:** AssetClassification zorunlu DEĞİLDİR. Belirtildiğinde:
- `Customer Content`: ProdId, ItemId, weight alanları gibi gerçek müşteri verisi
- `System Metadata`: LedgerJournalId, MainAccountNum gibi sistem referans alanları

### Label Formatı

Gerçek projeden görülen tüm formatlar:


### IgnoreEDTRelation

```xml
<IgnoreEDTRelation>Yes</IgnoreEDTRelation>
```

- EDT'de tanımlanmış relation'ları yok sayar
- Gerçek projede çok yaygın - hemen her EDT kullanımında `IgnoreEDTRelation>Yes` var
- Özellikle `RefRecId`, `NoYesId`, custom EDT'lerde zorunlu gibi kullanılıyor
- Bazı alanlarda (`ItemId` + `AssetClassification>Customer Content`) olmadan da kullanılmış

### AllowEdit vs AllowEditOnCreate

| Kombinasyon | Anlam |
|-------------|-------|
| `AllowEdit>No` + `AllowEditOnCreate>No` | Hiç düzenlenemez (sistem tarafından doldurulur) |
| `AllowEdit>No` (AllowEditOnCreate belirtilmemiş) | Oluşturulurken düzenlenebilir, sonra salt okunur |
| (hiçbiri belirtilmemiş) | Her zaman düzenlenebilir |

---

# AxTableExtension - Tablo Uzantıları

## Extension Kuralları

### İsimlendirme

| Nesne | Format | Örnek |
|-------|--------|-------|


### Extension'da Yapılabilecekler

1. **Yeni alan ekleme** - `<Fields>` içinde
2. **Yeni FieldGroup ekleme** - `<FieldGroups>` içinde (yeni özel grup)
3. **Mevcut FieldGroup'a alan ekleme** - `<FieldGroupExtensions>` içinde
4. **Yeni ilişki ekleme** - `<Relations>` içinde
5. **Yeni index ekleme** - `<Indexes>` içinde
6. **Tablo özelliği değiştirme** - `<PropertyModifications>` içinde (nadiren)

### Extension'da Yapılamayacaklar

- Mevcut alan özellikleri değiştirilemez (`<FieldModifications>` vardır ama nadir kullanılır)
- Mevcut metod override edilemez (CoC için ayrı Extension Class gerekir)
- Mevcut index değiştirilemez

### Extension ile Normal Tablo Farkları

| Özellik | Normal Tablo | Extension |
|---------|-------------|-----------|
| `<SourceCode>` | Var | Yok |
| `<Label>` | Var (tablo etiketi) | Yok |
| `<SaveDataPerCompany>` | Belirtilebilir | `<PropertyModifications>` ile değiştirilebilir |
| `<TableType>` | Belirtilebilir | Değiştirilemez |
| `<StateMachines>` | Var | Yok |
| `<DeleteActions>` | Var | Yok |

---

## Gerçek Projede Gözlemlenen Tutarsızlıklar

Bu bölüm, projedeki gerçek kod tutarsızlıklarını belgeler. Yeni kod yazarken bu tutarsızlıklardan kaçınılmalıdır.

### 4. Prefix Tutarsızlığı

- Her ikisi de aynı projede kullanılıyor

---

## Hızlı Referans: Hangi Tabloda Ne Var?

| Tablo | SaveDataPerCompany | CacheLookup | TitleField | CreatedBy/Modified | Tip |
|-------|-------------------|-------------|------------|---------------------|-----|

---

# AxForm - Form Tanimlari

## Namespace: Microsoft.Dynamics.AX.Metadata.V6

AxForm dosyalarinda kullanilan namespace:
```xml
<AxForm xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="Microsoft.Dynamics.AX.Metadata.V6">
```

Her iki namespace de zorunludur:
- `xmlns:i="http://www.w3.org/2001/XMLSchema-instance"` - `i:type` ve `i:nil` attribute'lari icin
- `xmlns="Microsoft.Dynamics.AX.Metadata.V6"` - metadata formatini belirtmek icin

---

## XML Yapisi - Genel Sablon


---

## KRITIK: SourceCode Blogu Icindeki xmlns="" Kurali

`<SourceCode>` icindeki her alt eleman `xmlns=""` ile namespace'i sifirlamalidir:
```xml
<SourceCode>
    <Methods xmlns="">          <!-- xmlns="" ZORUNLU -->
        ...
    </Methods>
    <DataSources xmlns="">       <!-- xmlns="" ZORUNLU -->
        ...
    </DataSources>
    <DataControls xmlns="">      <!-- xmlns="" ZORUNLU -->
        ...
    </DataControls>
    <Members xmlns="" />        <!-- xmlns="" ZORUNLU -->
</SourceCode>
```

`<DataSources>` blogu (design-level, SourceCode disinda) ise `xmlns=""` ile baslar:
```xml
<DataSources>
    <AxFormDataSource xmlns="">   <!-- AxFormDataSource'da xmlns="" var -->
        ...
    </AxFormDataSource>
</DataSources>
```

---

## KRITIK: FormControlExtension i:nil="true" Kurali

Her `<AxFormControl>` icinde `<FormControlExtension i:nil="true" />` satiri ZORUNLUDUR. Tek istisna: Eger kontrolun extension'i varsa (nadiren), o zaman bos eleman yazilmaz. Pratikte tum kontrollerde `i:nil="true"` kullanilir.

```xml
<AxFormControl xmlns="" i:type="AxFormStringControl">
    <Name>KontrolAdi</Name>
    <Type>String</Type>
    <FormControlExtension
        i:nil="true" />       <!-- HER ZAMAN YAZILIR -->
    <DataField>AlanAdi</DataField>
    <DataSource>TabloAdi</DataSource>
</AxFormControl>
```

---

## KRITIK: Design icindeki Controls Blogu

`<Design>` icindeki `<Controls>` ve tum alt kontroller `xmlns=""` attribute ile baslar:


Kural: `<Controls>` blogunun kendisinde `xmlns=""` YOKTUR (ust seviyedeki `<Controls xmlns="">` disinda). Ama icindeki her `<AxFormControl>` elementinde `xmlns=""` vardir.

---

## Form Control Tipleri - TAM REFERANS

### Zorunlu Ozellikler (Her Kontrol Icin)

Her `<AxFormControl>` blogu su ogeleri ZORUNLU icerir:
1. `xmlns=""` - element attribute'u
2. `i:type="AxForm...Control"` - element attribute'u
3. `<Name>` - benzersiz kontrol adi
4. `<Type>` - i:type ile eslesen tip metni
5. `<FormControlExtension i:nil="true" />` - extension placeholder

### Kontrol Tipi Esleme Tablosu

| i:type | `<Type>` degeri | X++ Tipi | Kullanim |
|---|---|---|---|
| `AxFormActionPaneControl` | `ActionPane` | - | Ust buton cubugu |
| `AxFormActionPaneTabControl` | `ActionPaneTab` | - | ActionPane icindeki tab |
| `AxFormButtonGroupControl` | `ButtonGroup` | - | Buton grubu |
| `AxFormMenuFunctionButtonControl` | `MenuFunctionButton` | - | MenuItem'a bagli buton |
| `AxFormCommandButtonControl` | `CommandButton` | - | OK, Cancel, New, Delete vb. |
| `AxFormDropDialogButtonControl` | `DropDialogButton` | - | Acilir dialog butonu |
| `AxFormGroupControl` | `Group` | - | Container grup |
| `AxFormTabControl` | `Tab` | - | Tab kapsayici |
| `AxFormTabPageControl` | `TabPage` | - | Tab sayfasi |
| `AxFormGridControl` | `Grid` | - | Veri grid'i |
| `AxFormStringControl` | `String` | `str` | Metin alani |
| `AxFormRealControl` | `Real` | `real` | Ondalikli sayi |
| `AxFormIntegerControl` | `Integer` | `int` | Tamsayi |
| `AxFormInt64Control` | `Int64` | `int64` | 64-bit tamsayi (RecId icin) |
| `AxFormDateTimeControl` | `DateTime` | `utcdatetime` | Tarih/saat |
| `AxFormDateControl` | `Date` | `date` | Tarih |
| `AxFormCheckBoxControl` | `CheckBox` | `NoYes` | Onay kutusu |
| `AxFormComboBoxControl` | `ComboBox` | `enum` | Dropdown |
| `AxFormStaticTextControl` | `StaticText` | - | Salt okunur etiket |

### AxFormCommandButtonControl - Komut Butonu

```xml
<AxFormControl xmlns="" i:type="AxFormCommandButtonControl">
    <Name>BtnAdi</Name>
    <AutoDeclaration>Yes</AutoDeclaration>
    <ElementPosition>596523234</ElementPosition>  <!-- System-generated konumlandirma -->
    <FilterExpression>%1</FilterExpression>
    <HeightMode>Auto</HeightMode>
    <Type>CommandButton</Type>
    <VerticalSpacing>-1</VerticalSpacing>
    <WidthMode>Auto</WidthMode>
    <FormControlExtension i:nil="true" />
    <ButtonDisplay>TextWithImageLeft</ButtonDisplay>
    <!-- Command degerleri: New | DeleteRecord | OK | Cancel | Save | Revert -->
    <Command>New</Command>
    <MultiSelect>Yes</MultiSelect>
    <NormalImage>Add</NormalImage>      <!-- Add | Delete | Edit vb. -->
    <Primary>Yes</Primary>
    <SaveRecord>No</SaveRecord>         <!-- Cancel butonlari icin -->
    <ShowShortCut>No</ShowShortCut>
    <Text>@SYS319116</Text>
</AxFormControl>
```

### AxFormStaticTextControl - Statik Metin

```xml
<AxFormControl xmlns="" i:type="AxFormStaticTextControl">
    <Name>TitleText</Name>
    <Skip>Yes</Skip>
    <Type>StaticText</Type>
    <WidthMode>SizeToAvailable</WidthMode>
    <FormControlExtension i:nil="true" />
    <Style>MainInstruction</Style>      <!-- Baslik stili -->
    <Text>Baslik Metni</Text>
</AxFormControl>
```

### AxFormGridControl - Grid Ozellikleri

```xml
<AxFormControl xmlns="" i:type="AxFormGridControl">
    <Name>GridAdi</Name>
    <AutoDeclaration>Yes</AutoDeclaration>
    <Type>Grid</Type>
    <Visible>No</Visible>                        <!-- Gizli grid -->
    <FormControlExtension i:nil="true" />
    <Controls>...</Controls>
    <DataGroup>AutoReport</DataGroup>            <!-- Tablo field group adi -->
    <DataSource>TabloAdi</DataSource>            <!-- Bagli datasource -->
</AxFormControl>
```

### AxFormComboBoxControl - ComboBox

```xml
<AxFormControl xmlns="" i:type="AxFormComboBoxControl">
    <Name>ComboAdi</Name>
    <Type>ComboBox</Type>
    <FormControlExtension i:nil="true" />
    <DataField>EnumAlan</DataField>
    <DataSource>TabloAdi</DataSource>
    <Items />     <!-- ZORUNLU - Bos olsa bile yazilmali -->
</AxFormControl>
```

### AxFormStringControl - Cok Satirli Metin

```xml
<AxFormControl xmlns="" i:type="AxFormStringControl">
    <Name>MultilineText</Name>
    <AutoDeclaration>Yes</AutoDeclaration>
    <HeightMode>SizeToAvailable</HeightMode>
    <Type>String</Type>
    <WidthMode>SizeToAvailable</WidthMode>
    <FormControlExtension i:nil="true" />
    <DisplayHeight>1000</DisplayHeight>
    <DisplayHeightMode>Fixed</DisplayHeightMode>
    <DisplayLength>1000</DisplayLength>
    <DisplayLengthMode>Fixed</DisplayLengthMode>
    <MultiLine>Yes</MultiLine>
    <!-- DataField YOK - standalone -->
</AxFormControl>
```

---

## DataSource Tanimlari

### AxFormDataSource - Tam Ozellikler

```xml
<AxFormDataSource xmlns="">
    <Name>DataSourceAdi</Name>               <!-- ZORUNLU - Form icindeki referans adi -->
    <Table>TabloAdi</Table>                  <!-- ZORUNLU - Fiziksel tablo adi -->
    <Fields>
        <AxFormDataSourceField>
            <DataField>Alan1</DataField>
        </AxFormDataSourceField>
        <!-- Tum ilgili alanlar listelenir -->
        <!-- Sistem alanlari da dahil: DataAreaId, Partition, RecId, TableId -->
        <!-- CreatedBy, CreatedDateTime, ModifiedBy, ModifiedDateTime -->
    </Fields>
    <ReferencedDataSources />               <!-- Genellikle bos -->

    <!-- OPSIYONEL OZELLIKLER -->
    <JoinSource>ParentDSAdi</JoinSource>    <!-- Parent datasource ile join -->
    <AllowCreate>No</AllowCreate>           <!-- Yeni kayit eklemeyi engelle -->
    <AllowDelete>No</AllowDelete>           <!-- Silmeyi engelle -->
    <AllowEdit>No</AllowEdit>               <!-- Duzenlemeyi engelle -->
    <InsertAtEnd>No</InsertAtEnd>           <!-- Sona ekleme -->
    <InsertIfEmpty>No</InsertIfEmpty>       <!-- ZORUNLU - Bos tabloya otomatik satir eklensin mi -->

    <DataSourceLinks />                     <!-- Farkli datasource baglamalari -->
    <DerivedDataSources />                  <!-- Turetilmis datasource'lar -->
</AxFormDataSource>
```

### InsertIfEmpty Kurali
- `No` (onerilir): Kayit yoksa otomatik bos satir eklenmez
- `Yes`: Tablo bos ise otomatik bir yeni kayit olusturulur
- Parametreler tablolari icin `Yes` daha uygundur (her zaman bir kayit olmali)

### DataSource ve DataSource Adi Farki
- `<Name>`: Formda kullanilan takma ad (ornek: `MarkupAutoTable`)
- `<Table>`: Fiziksel tablo adi (ornek: `MarkupAutoTable`)
- Genellikle ayni olur ama farklilasabilir

### Join Iliskisi - JoinSource

---

## Form Methods

### classDeclaration - Form Class Tanimi

```x++
[Form]                              // ZORUNLU attribute
public class FormAdi extends FormRun
{
    // Form-level member degiskenler
    Common      callerRecord;
    TableId     callerTableId;
    RecId       filteredRecId;
}
```

**Kurallar:**
- `[Form]` attribute mutlaka yazilmali
- Her form `FormRun`'dan turetilmeli
- Member degiskenler burada tanimlanir

### init - Form Baslatma

```x++
public void init()
{
    super();            // MUTLAKA cagir, genellikle ilk satirda

    Args args = element.args();

    // Cagiran formdan kayit alma
    if (args && args.record())
    {
        callerRecord  = args.record();
        callerTableId = args.dataset();
    }

    // Dinamik kontrol erisimi
    FormControl ctrl = this.design().controlname("KontrolAdi");

    // DataSource range ekleme
    QueryBuildRange range = this.dataSource(formDataSourceStr(FormAdi, TabloAdi))
        .queryBuildDataSource()
        .addRange(fieldNum(TabloAdi, Alan));
    range.value("filtre");
    range.status(RangeStatus::Locked);
}
```

### close - Form Kapatma

```x++
public void close()
{
    // kapatmadan onceki islemler
    super();
}
```

### element keyword
`element` formu kendine referans verir. Kullanim ornekleri:
- `element.args()` - cagiran formun parametreleri
- `element.args().caller()` - cagiran formu FormRun olarak al
- `element.args().record()` - cagiran formun secili kaydi
- `element.args().dataset()` - cagiran tablonun tableNum'i
- `element.design().controlname("Ad")` - kontrol ada gore bul
- `element.RefreshCaller()` - ozel metot ornegi

---

## DataSource Methods

### SourceCode/DataSources - DataSource Metod Tanimlari

```xml
<DataSources xmlns="">
    <DataSource>
        <Name>DataSourceAdi</Name>     <!-- Tasarim DataSource adi ile ayni olmali -->
        <Methods>
            <Method>
                <Name>metodAdi</Name>
                <Source><![CDATA[
    public void metodAdi()
    {
        super();
        // islem
    }

]]></Source>
            </Method>
        </Methods>
        <Fields />      <!-- Genellikle bos -->
    </DataSource>
</DataSources>
```

**Onemli:** `<Fields />` etiketinin bos olsa da yazilmasi gerekir.

### Yaygin DataSource Metodlari

**init - Range ve filtre ekleme:**
```x++
public void init()
{
    super();
    QueryBuildRange range = this.queryBuildDataSource()
        .addRange(fieldNum(TabloAdi, AlanAdi));
    range.value("DegerFiltresi");
    range.status(RangeStatus::Locked);
}
```

**executeQuery - Ozel sorgu calistirma:**
```x++
public void executeQuery()
{
    // Ozel filtre mantigi
    super();
}
```

**active - Secim degistiginde:**
```x++
public int active()
{
    int ret = super();
    // secilen kayda gore UI guncelleme
    return ret;
}
```

**validateWrite - Kayit dogrulama:**
```x++
public boolean validateWrite()
{
    boolean ret = super();
    if (kosulHata)
    {
        ret = false;
        error("Hata mesaji");
    }
    return ret;
}
```

### DataSource API


---

## Control Methods

### DataControls - Control Metod Tanimlari

```xml
<DataControls xmlns="">
    <Control>
        <Name>KontrolAdi</Name>          <!-- Design'daki kontrol adi ile ayni -->
        <Type>MenuFunctionButton</Type>  <!-- Kontrol tipi -->
        <Methods>
            <Method>
                <Name>clicked</Name>
                <Source><![CDATA[
    public void clicked()
    {
        super();
        // islem
    }

]]></Source>
            </Method>
        </Methods>
    </Control>
</DataControls>
```

### clicked - Buton Tiklama

```x++
public void clicked()
{
    super();                                    // MenuItem action'i calistirma
    TabloAdi_ds.research(true);                 // Yenileme
    element.KomutMetodu();                      // Form metodunu cagir
}
```

### modified - Deger Degisimi

```x++
public void modified()
{
    super();
    // Baska bir kontrolu guncelle
    BagliKontrol.value(HesaplananDeger);
}
```

### lookup - Ozel Arama

```x++
public void lookup()
{
    SysTableLookup sysTableLookup = SysTableLookup::newParameters(
        tableNum(LookupTablosu), this);

    sysTableLookup.addLookupfield(fieldNum(LookupTablosu, KodAlani));
    sysTableLookup.addLookupfield(fieldNum(LookupTablosu, AciklamaAlani));

    sysTableLookup.performFormLookup();
}
```

---

## HeightMode / WidthMode

Kontrol boyutlari icin kullanilan ozellikler:

| Deger | Aciklama | Kullanim |
|---|---|---|
| `SizeToAvailable` | Mevcut alani doldur | Ana container'lar, grid |
| `SizeToContent` | Icerige gore boyutlan | Buton gruplari, kucuk kontroller |
| `Auto` | Sistem tanimli boyut | Varsayilan, butonlar |

**Ornek - Full-screen grid pattern:**
```xml
<AxFormControl xmlns="" i:type="AxFormGroupControl">
    <Name>MainGroup</Name>
    <HeightMode>SizeToAvailable</HeightMode>  <!-- Yuksekligi doldur -->
    <Type>Group</Type>
    <WidthMode>SizeToAvailable</WidthMode>    <!-- Genisligi doldur -->
    <FormControlExtension i:nil="true" />
    <Controls>
        <AxFormControl xmlns="" i:type="AxFormGridControl">
            <!-- Grid otomatik olarak group'u doldurur -->
```

---

## MenuItemButton vs MenuFunctionButton

| Ozellik | `AxFormMenuFunctionButtonControl` (MenuFunctionButton) | `AxFormCommandButtonControl` (CommandButton) |
|---|---|---|
| Amac | MenuItem'a bagli islem calistirma | Form komutlari (New, Delete, OK, Cancel) |
| `<MenuItemName>` | Zorunlu | Yok |
| `<MenuItemType>` | Action / Display / Output / bos | Yok |
| `<Command>` | Yok | Zorunlu (New/DeleteRecord/OK/Cancel/Save/Revert) |
| `<Big>Yes` | Destekler | Destekler |
| `<NeedsRecord>` | Destekler | - |
| `<MultiSelect>` | Destekler | Destekler |

**Ne zaman hangi tip?**
- Yeni form acan veya class calistiran buton: `AxFormMenuFunctionButtonControl`
- Standart form eylemi (kayit ekle, sil, kaydet, iptal): `AxFormCommandButtonControl`
- DropDialog popup acma: `AxFormDropDialogButtonControl`

---

## AutoDeclaration

`<AutoDeclaration>Yes</AutoDeclaration>` - kontrolu X++ kodu icinden degisken olarak erisebilir yapar.

**Ne zaman `Yes` yapilir:**
- Koddan kontrole erismek gerektiginde (`.value()`, `.visible()`, `.label()` gibi metodlar cagirilacaksa)
- DataSource kontrolune X++'dan erisim gerektiginde (ActionPane'ler)
- Cok kez kullanilacak kontroller

**Ornekler:**
```xml
<!-- AutoDeclaration=Yes: Koddan erisim gerekiyor -->
<Name>CorrectedOnsQty</Name>
<AutoDeclaration>Yes</AutoDeclaration>
<!-- X++: real value = CorrectedOnsQty.realValue(); -->

<!-- AutoDeclaration olmayan: Sadece UI, koddan erisim yok -->
<Name>FormGridControl_Description</Name>
<!-- AutoDeclaration yok -->
```

---

## Design-Level Ozel Ozellikler


---

## Kanonik Property Sirasi ve Edit Discipline

Bu alt bolum, AxForm dosyalarinda gozlemlenen **kanonik property sirasini** belgeler. Edit ederken bu siraya uyulmazsa VS designer dosyayi tekrar acip kapadiginda otomatik olarak yeniden duzenler ve buyuk diff churn olusturur. Deserializer tolerans gosterir ama disiplinli kullanim repo'yu temiz tutar.

Bu sira VS designer'in default cikti formati ile esittir.

### AxForm Root Child Sirasi

| Sira | Element | Zorunlu | Aciklama |
|---|---|---|---|
| 1 | `<Name>` | Evet | Form adi (dosya adi ile esit) |
| 2 | `<SourceCode>` | Evet | Methods/DataSources/DataControls/Members alt-bloklari |
| 3 | `<DataSourceQuery>` | Hayir | Forma bagli standart Query adi (opsiyonel) |
| 4 | `<DataSources>` | Evet | Veri kaynaklari (bos olabilir ama tag mevcut) |
| 5 | `<Design>` | Evet | UI tasarimi (Caption + Controls) |
| 6 | `<Parts>` | Hayir | Genellikle bos `<Parts />` |

### AxFormControl Property Sirasi (Kanonik)

Her `<AxFormControl xmlns="" i:type="AxForm...">` icindeki ic-property sirasi:

| Sira | Property | Tip | Aciklama |
|---|---|---|---|
| 1 | `<Name>` | Tum kontroller | Benzersiz kontrol adi (ZORUNLU) |
| 2 | `<AutoDeclaration>` | Tum | "Yes" ise X++ koddan erisilebilir |
| 3 | `<AllowEdit>` | Data control | "No" ise readonly |
| 4 | `<ElementPosition>` | Tum | VS otomatik konumlandirma (integer) |
| 5 | `<FilterExpression>` | Tum | Genellikle `%1` |
| 6 | `<HeightMode>` | Tum | Auto / SizeToAvailable / SizeToContent / Manual |
| 7 | `<NeededPermission>` | Tum | Read / Update / Delete / vb. |
| 8 | `<Type>` | Tum | `i:type` ile esit metin (ZORUNLU) |
| 9 | `<VerticalSpacing>` | Tum | Genellikle `-1` (default) |
| 10 | `<WidthMode>` | Tum | Auto / SizeToAvailable |
| 11 | `<FormControlExtension i:nil="true" />` | Tum | ZORUNLU - bos placeholder |
| 12 | `<DataField>` | Data control | Tablodaki alan adi |
| 13 | `<DataMethod>` | Data control | Display method referansi (`Class::method`) |
| 14 | `<DataSource>` | Data control | Form datasource adi |
| 15 | `<Label>` | Tum | Label key (`@LabelFile:LabelKey`) |
| 16 | Tip-spesifik | - | NoOfDecimals, Items, MenuItemName, vb. |
| 17 | `<Controls>` | Container | Group/Grid/TabPage icindeki alt kontroller |

**Onemli:** `<FormControlExtension i:nil="true" />` her zaman tip-spesifik property'lerden ONCE gelir. DataField ve DataSource'tan SONRA degil.

### AxFormDataSource Property Sirasi

| Sira | Property | Aciklama |
|---|---|---|
| 1 | `<Name>` | Form icinde datasource takma adi (ZORUNLU) |
| 2 | `<Table>` | Fiziksel tablo adi (ZORUNLU) |
| 3 | `<Fields>` | Gosterilecek alanlar listesi |
| 4 | `<ReferencedDataSources />` | Genellikle bos |
| 5 | `<JoinSource>` | Parent datasource adi (opsiyonel) |
| 6 | `<LinkType>` | OuterJoin / InnerJoin / Passive / Delayed (opsiyonel) |
| 7 | `<AllowCreate>` | Yes / No |
| 8 | `<AllowDelete>` | Yes / No |
| 9 | `<AllowEdit>` | Yes / No |
| 10 | `<InsertAtEnd>` | Yes / No |
| 11 | `<InsertIfEmpty>` | Yes / No (genellikle "No") |
| 12 | `<DataSourceLinks />` | Genellikle bos |
| 13 | `<DerivedDataSources />` | Genellikle bos |

### AxFormControl Tip-Spesifik Zorunlu Eklentiler

Belirli kontrol tiplerinde **tip-spesifik zorunlu** bir veya daha fazla property vardir. Atlanirsa derleme veya render bozulur.

| i:type | Zorunlu ek | Aciklama |
|---|---|---|
| AxFormComboBoxControl | `<Items />` | Bos olsa bile var olmali |
| AxFormGridControl | `<Controls>` | Icindeki kolonlari tasir |
| AxFormGroupControl | `<Controls>` | Icindeki alanlari tasir |
| AxFormTabPageControl | `<Controls>`, `<Caption>` | Icerik ve tab basligi |
| AxFormMenuFunctionButtonControl | `<MenuItemName>`, `<MenuItemType>` | Menu item referansi |
| AxFormCommandButtonControl | `<Command>` | New/DeleteRecord/OK/Cancel/Save/Revert |
| AxFormStringControl (display) | `<DataMethod>` | Method referansi (DataField yerine) |
| AxFormStringControl (multi-line) | `<MultiLine>Yes</MultiLine>` | `<DisplayHeight>` ile beraber |
| AxFormRealControl | `<NoOfDecimals>` | Opsiyonel ama yaygin, -1 default |

### xmlns="" Zorunluluk Matrisi

Form XML'inde `xmlns=""` attribute'unun nerede oldugu cogu deserialize hatasinin kaynagidir. Atlanmamasi gereken yerler:

| Konum | xmlns="" var mi? | Aciklama |
|---|---|---|
| `<SourceCode>` icindeki `<Methods>` | EVET | `<Methods xmlns="">` |
| `<SourceCode>` icindeki `<DataSources>` | EVET | `<DataSources xmlns="">` |
| `<SourceCode>` icindeki `<DataControls>` | EVET | `<DataControls xmlns="">` |
| `<SourceCode>` icindeki `<Members>` | EVET | `<Members xmlns="" />` |
| Root `<DataSources>` (Design'in disinda) | HAYIR | Root tag temiz |
| Root `<DataSources>` icindeki her `<AxFormDataSource>` | EVET | `<AxFormDataSource xmlns="">` |
| `<Design>` icindeki `<Caption>`, `<Controls>` | EVET | Her birinde `xmlns=""` |
| `<Design>` icindeki `<Pattern>`, `<DataSource>` vb. property'ler | EVET | Her property'de `xmlns=""` |
| Root `<Design>` tag'in kendisi | HAYIR | Sade |
| `<Controls>` icindeki her `<AxFormControl>` | EVET | `xmlns=""` + `i:type` birlikte |
| Nested `<Controls>` (Group/Grid/TabPage icinde) | HAYIR | Ic Controls tag'in kendisinde yok |
| Nested icindeki her `<AxFormControl>` | EVET | Yine her control'de `xmlns=""` var |

**Tek satir kural:** `<AxForm...>` ve `<Design>` disindaki **her data tasiyici alt eleman** (Methods, DataSources, DataControls, Members, Caption, AxFormControl, AxFormDataSource, Pattern, DataSource property'si) `xmlns=""` ile baslar. Konteyner tag'ler (Controls, Fields, vb.) sade.

### Self-Closing vs Acma-Kapatma Tag Disiplini

Bos liste-tipi elementler **mutlaka self-closing** yazilir. Acma-kapatma form'u (`<X></X>`) deserialize'da hataya yol acmaz ama VS designer reorder eder ve diff churn olusturur.

| Element | Bos hali | Dolu hali |
|---|---|---|
| `<Items />` (ComboBox) | `<Items />` | `<Items><AxFormComboBoxItem>...</AxFormComboBoxItem></Items>` |
| `<ReferencedDataSources />` | `<ReferencedDataSources />` | (nadiren dolu) |
| `<DataSourceLinks />` | `<DataSourceLinks />` | `<DataSourceLinks><AxFormDataSourceLink>...</AxFormDataSourceLink></DataSourceLinks>` |
| `<DerivedDataSources />` | `<DerivedDataSources />` | (nadiren dolu) |
| `<Parts />` | `<Parts />` | (nadiren dolu) |
| `<Members />` (SourceCode) | `<Members xmlns="" />` | (nadiren dolu) |
| `<Controls>` Group/Grid icinde | `<Controls />` | `<Controls><AxFormControl xmlns="">...</AxFormControl></Controls>` |

### Indentation: TAB Karakteri Zorunlu

AxForm ve AxFormExtension dosyalari **TAB karakteri** ile indent edilir (1 TAB = 1 seviye). CDATA blogu icindeki X++ kodu icin 4 boslukluk space kullanilir.

**Yanlis:** Space (4 boslukluk) indentation veya space + tab karisimi → VS designer reorder eder, diff patlatir.

**Dogru:**
```
<?xml version="1.0" encoding="utf-8"?>
<AxForm xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="Microsoft.Dynamics.AX.Metadata.V6">
\t<Name>FormAdi</Name>
\t<SourceCode>
\t\t<Methods xmlns="">
\t\t\t<Method>
\t\t\t\t<Name>methodAdi</Name>
\t\t\t\t<Source><![CDATA[
    public void methodAdi()
    {
        // CDATA icinde 4 boslukluk space
    }

]]></Source>
\t\t\t</Method>
\t\t</Methods>
\t</SourceCode>
\t...
</AxForm>
```

### Edit Discipline: Property Sirasi Neden Korunmali?

D365 metadata XML deserializer'i property sirasi konusunda tolerans gosterir; yanlis sirayla yazilmis bir XML hata vermez. Ancak VS designer (Visual Studio 17) dosyayi acip kapadiginda otomatik olarak yeniden duzenler:
- Property'ler kanonik siraya gore yeniden yazilir
- Indentation TAB'a normalize edilir
- Bos elementler self-closing'e cevrilir

Sonuc: AI veya el ile yazilan yanlis sirali bir edit, VS'de bir defa acildiktan sonra **dev bir diff** uretir. Bu diff churn:
- Code review'u zorlastirir (hedef degisiklik baska reorder degisikliklerle karisir)
- Merge conflict ihtimalini artirir
- Repository history'sini kirletir

Bu nedenle her edit kanonik sira ve format ile yapilmali.

---

# AxFormExtension - Form Uzantilari

## Namespace

```xml
<AxFormExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
                 xmlns="Microsoft.Dynamics.AX.Metadata.V6">
```

Her iki namespace de `AxForm` ile aynidir.

## XML Yapisi - Genel Sablon


---

## PositionType Secenekleri

| PositionType | Aciklama |
|---|---|
| (yok) | En sona ekle (parent'in son cocugu) |
| `AfterItem` | Belirli bir kardes kontrolden sonra - `<PreviousSibling>` ile birlikte kullanilir |
| `Begin` | Parent'in ilk cocugu olarak ekle |

**Kullanim pattern'i:**
```xml
<!-- En sona ekleme -->
<Parent>GridAdi</Parent>
<!-- PositionType YOK -->

<!-- Belirli bir elemandan sonra ekleme -->
<Parent>GridAdi</Parent>
<PositionType>AfterItem</PositionType>
<PreviousSibling>MevcutKontrolAdi</PreviousSibling>

<!-- Basa ekleme -->
<Parent>TabPageAdi</Parent>
<PositionType>Begin</PositionType>
```

---

## DataMethod Referansi

Display method'lari bir alana gostermek icin `<DataMethod>` kullanilir.

**Format:** `ExtensionSinifAdi.metodAdi`


**Kurallar:**
- `<DataMethod>` varsa `<DataField>` OLMAMALI
- `<DataSource>` yine de gereklidir
- `AutoDeclaration>Yes` onerilen (kod erisimi icin)
- Readonly kontrol (kullanici duzenleyemez, sadece gosterir)

---

## AxFormExtension - KRITIK KURALLAR

### 2. AxFormExtensionControl Name'i Benzersiz Olmali
`<Name>` icindeki deger tum projede benzersiz olmali. Tipik format:
- `FormExtensionControl` + rastgele suffix (ornek: `FormExtensionControlggplnyst1`)
- Veya anlamli isim (ornek: `Copyhxnkna4q1`, `Copyadnpcu1x1`)

### 3. FormControl Icindeki xmlns=""
`<AxFormExtensionControl>` icindeki `<FormControl>` etiketi `xmlns=""` ile baslamalidir:
```xml
<FormControl xmlns="" i:type="AxFormGroupControl">
```

### 4. Alt Kontroller Yapisi
`<FormControl>` icindeki alt `<AxFormControl>` elemanlarinda yine `xmlns=""` kullanilir - tam olarak normal AxForm'daki gibi.

### 5. ControlModifications
Mevcut bir kontrolun ozelliklerini degistirmek icin:
```xml
<ControlModifications>
    <AxExtensionModification xmlns="">
        <Name>MevcutKontrolAdi</Name>
        <PropertyModifications>
            <!-- Degistirilecek ozellikler -->
        </PropertyModifications>
    </AxExtensionModification>
</ControlModifications>
```

### 6. Zorunlu Kapanma Etiketleri
Bos bile olsa su etiketler MUTLAKA bulunmali:
```xml
<ControlModifications />
<Controls />              <!-- veya <Controls>...</Controls> -->
<DataSourceModifications />
<DataSourceReferences />
<DataSources />
<Parts />
<PropertyModifications />
```

---

## AxFormExtension vs AxForm - DataSources Blogu Farki

| Kriter | AxForm | AxFormExtension |
|---|---|---|
| `<SourceCode>` blogu | VAR - form metodlari | YOK |
| `<DataSources>` (design-level) | VAR - tablo baglamalari | `<DataSources />` (bos, genellikle) |
| DataSource metod tanimlari | `<SourceCode><DataSources>` icinde | Yoktur (CoC class kullanilir) |
| Control metod tanimlari | `<SourceCode><DataControls>` icinde | Yoktur (CoC class kullanilir) |
| Yeni DataSource ekleme | `<DataSources>` de tanimlanir | `<DataSources>` de tanimlanabilir |

**Extension'da metodlar nasil tanimlanir?**
FormExtension'da dogrudan metod tanimlama yoktur. Bunun yerine extension class (CoC) kullanilir:

```x++
[ExtensionOf(formStr(PurchTable))]
final class PurchTable_MyModel_Extension
{
    // Form metodunu genisletme
    public void init()
    {
        next init();
        // ek islem
    }
}
```

---

## Edit Senaryolari: ControlModifications vs Controls vs DataSources Karar Matrisi

AxFormExtension dosyasinda **uc temel edit kategorisi** vardir. Hangi kategoride hangi blok kullanilir, hangi pattern uygulanir:

| Senaryo | Kullanilacak blok | XML pattern |
|---|---|---|
| Mevcut control property degistirme | `<ControlModifications>` | `AxExtensionModification` + `PropertyModifications` + `AxPropertyModification` |
| Yeni control ekleme | `<Controls>` | `AxFormExtensionControl` + `FormControl` + `Parent` + `PositionType` |
| Yeni datasource ekleme (join) | `<DataSources>` | `AxFormDataSource` + `JoinSource` + `LinkType` |

**Onemli ayrim:** "Mevcut control" derken standart Microsoft form'unda zaten var olan bir kontrol. "Yeni control" eklemek istiyorsan `<Controls>` bloguna `AxFormExtensionControl` koyarsin; ama mevcut bir kontrolun property'sini degistirmek istiyorsan `<ControlModifications>` blogunu kullanirsin. Karistirmak yaygin bir hata: AI yeni `AxFormControl` ekleyip `Name` alanina mevcut control adini yazarak property degistirmeye calisir — bu, original kontrolu degistirmez, ayni isimde **yeni** kontrol olusturmaya calisir ve calismaz.

### AxExtensionModification Tam XML Sablonu

```xml
<ControlModifications>
\t<AxExtensionModification xmlns="">
\t\t<Name>ControlAdi</Name>
\t\t<PropertyModifications>
\t\t\t<AxPropertyModification>
\t\t\t\t<Name>PropertyAdi</Name>
\t\t\t\t<Value>YeniDeger</Value>
\t\t\t</AxPropertyModification>
\t\t</PropertyModifications>
\t</AxExtensionModification>
</ControlModifications>
```

Gercek ornek (mevcut bir grid kontrolunu gizleme):
```xml
<ControlModifications>
\t<AxExtensionModification xmlns="">
\t\t<Name>groupInterCompany_Grid</Name>
\t\t<PropertyModifications>
\t\t\t<AxPropertyModification>
\t\t\t\t<Name>Visible</Name>
\t\t\t\t<Value>No</Value>
\t\t\t</AxPropertyModification>
\t\t</PropertyModifications>
\t</AxExtensionModification>
</ControlModifications>
```

**Kritik kurallar:**
1. `<AxExtensionModification>`'da `xmlns=""` ZORUNLU. `<AxPropertyModification>`'da xmlns YOK.
2. `<Name>` (Modification'in disindaki) — mevcut formdaki control adini YAZAR. Standart formda yoksa modification etkisiz.
3. Bir `AxExtensionModification` bloguna birden fazla `AxPropertyModification` eklenebilir (ayni control'un birden fazla property'sini degistirmek icin).
4. Birden fazla control'u degistirmek icin birden fazla `AxExtensionModification` ekle (paralel).

### PropertyName icin Yaygin Degerler

| PropertyName | Value tipi | Ne icin |
|---|---|---|
| `Visible` | Yes / No | Kontrolu gizleme |
| `AllowEdit` | Yes / No | Readonly yapma |
| `NoOfDecimals` | int (`-1` = otomatik) | Real control ondalik basamak |
| `MinNoOfDecimals` | int | Real control min basamak |
| `Mandatory` | Yes / No | Zorunlu alan isareti |
| `Label` | Label key (`@LabelFile:LabelKey`) | Etiket degistirme |
| `Skip` | Yes / No | Tab order'da atlama |
| `AutoDeclaration` | Yes / No | X++ koddan erisim |
| `ExtendedDataType` | EDT adi | Type'i degistirme |
| `Width` / `Height` | Pixel int | Boyut |

### Genellikle Bos Birakilan Bloklar

Pratikte cogu AxFormExtension dosyasinda asagidaki bloklar BOS (`<X />`) olarak yer alir:

```xml
<DataSourceModifications />
<DataSourceReferences />
<Parts />
<PropertyModifications />
```

**Aciklamalar:**
- `<DataSourceModifications>`: Mevcut datasource'un property'sini degistirmek (AllowCreate/Delete/Edit) icin kullanilir. Microsoft pattern destekler ancak nadiren ihtiyac olur. Eger bir datasource'un property'sini degistirmen gerekirse asagidaki sablonu kullan:
  ```xml
  <DataSourceModifications>
  \t<AxFormDataSourceModification xmlns="">
  \t\t<DataSourceName>MevcutDsAdi</DataSourceName>
  \t\t<AllowCreate>No</AllowCreate>
  \t</AxFormDataSourceModification>
  </DataSourceModifications>
  ```
- `<DataSourceReferences>`, `<Parts>`, `<PropertyModifications>`: Standart kullanim yok; her zaman bos.

**Anti-pattern:** AxFormExtension'a `<SourceCode>` veya `<Methods>` blogu eklemeye calismak. AxFormExtension dosyasi method tutamaz. Form-level method gerekirse `[FormControlEventHandler]` / `[FormDataSourceEventHandler]` ile mevcut derlenen bir class'a static handler eklenir.

### AxFormExtensionControl Name Suffix Konvansiyonu

Bir form extension'a yeni control eklerken VS otomatik olarak benzersiz bir `Name` uretir:

| Pattern | Aciklama | Ornek |
|---|---|---|
| `FormExtensionControl[8-10char][digit]` | Yeni control olusturuldugunda | `FormExtensionControlwk3sueh31` |
| `Copy[8-10char][digit]` | Bir control'den kopyala-yapistir ile olusturuldugunda | `Copymv4zj5ni1` |

Manuel olarak yazilirken bu pattern'e uyulmasi gerekmiyor ama:
1. **Benzersizlik** zorunlu — ayni form extension icinde iki `AxFormExtensionControl` ayni `Name` tasimamali.
2. **Iç FormControl'un `Name`** (yani `<FormControl><Name>...</Name></FormControl>`) form runtime'da kullanilir; AxFormExtensionControl'un disindaki `Name` sadece metadata ID'sidir.

### Edit Pattern'lerinin Sik Karistirildigi Yerler

| Hata | Belirti | Dogru pattern |
|---|---|---|
| Mevcut control property degistirmek icin yeni AxFormControl eklemek | Property degismez, ayni isimde iki control olusturulur ve runtime'da carpisma | `<ControlModifications>` blogunu kullan |
| AxFormExtension'a method eklemek | XML reddedilir veya silinen blok olur | Static event handler ile mevcut class'a ekle |
| `<ControlModifications>` icinde `<AxFormControl>` yazmak | Yanlis tip — deserialize hatasi | `<AxExtensionModification>` kullan |
| `<Controls>` icinde `<AxFormControl>` direkt yazmak (sarmalayici olmadan) | Parent/PositionType yok, kontrol bagimsiz kalir | `<AxFormExtensionControl>` ile sarmala |
| PositionType=AfterItem ama PreviousSibling vermemek | Control yerlesemez, render'da bosluk olusur | PreviousSibling MUTLAKA gerekli |

---

# AxQuery, AxQuerySimpleExtension, AxMenuItem ve AxReport - Referans Dokümantasyonu


---

## ICINDEKILER

1. [AxQuery - Sorgu Tanımları](#1-axquery---sorgu-tanımları)
2. [AxQuerySimpleExtension - Sorgu Uzantıları](#2-axquerysimpleextension---sorgu-uzantıları)
3. [AxMenuItemAction - Aksiyon Menü Öğeleri](#3-axmenuitemaction---aksiyon-menü-öğeleri)
4. [AxMenuItemDisplay - Görüntüleme Menü Öğeleri](#4-axmenuitemdisplay---görüntüleme-menü-öğeleri)
5. [AxMenuItemOutput - Çıktı Menü Öğeleri](#5-axmenuitemoutput---çıktı-menü-öğeleri)
6. [AxReport - SSRS Rapor Tanımları](#6-axreport---ssrs-rapor-tanımları)

---

# 1. AxQuery - Sorgu Tanımları

## 1.1 Temel XML Yapısı ve Namespace

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxQuery xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns=""
	i:type="AxQuerySimple">
```

**Kritik Notlar:**
- Root element'te hem `xmlns:i="http://www.w3.org/2001/XMLSchema-instance"` hem de `xmlns=""` (boş namespace) bulunur
- `i:type="AxQuerySimple"` zorunludur — tek desteklenen tip budur
- İçerideki tüm child elementler varsayılan namespace devralır (ek namespace gerekmez)

## 1.2 classDeclaration ve [Query] Attribute

Her AxQuery dosyasında `classDeclaration` metodu zorunludur:


**Neden `[Query]` attribute gereklidir?**
- D365 FO metadata compiler bu attribute olmadan sınıfı bir Query nesnesi olarak tanımaz
- `extends QueryRun` — sorgu çalıştırma framework'ü
- Class adı dosya adı ve `<Name>` değeri ile birebir eşleşmek zorundadır
- Sınıf gövdesi her zaman boştur — tüm mantık metadata tarafından tanımlanır

## 1.3 AxQuerySimpleRootDataSource — Root Veri Kaynağı

Her sorguda tam olarak bir adet `AxQuerySimpleRootDataSource` bulunur ve bu ana tablodur.

```xml
<DataSources>
    <AxQuerySimpleRootDataSource>
        <Name>VendPackingSlipTrans</Name>       <!-- Kod içinde erişim adı -->
        <DynamicFields>Yes</DynamicFields>       <!-- Tüm alanlar dahil edilsin mi? -->
        <Table>VendPackingSlipTrans</Table>      <!-- Gerçek tablo adı -->
        <DataSources>
            <!-- Gömülü (JOIN) data source'lar buraya gelir -->
        </DataSources>
        <DerivedDataSources />
        <Fields />                               <!-- DynamicFields=No ise belirli alanlar -->
        <Ranges />
        <GroupBy />
        <Having />
        <OrderBy />
    </AxQuerySimpleRootDataSource>
</DataSources>
```

**Önemli:** `<Name>` ve `<Table>` farklı olabilir. Örneğin:
- `<Name>InventTransPurch</Name>` + `<Table>InventTrans</Table>` → aynı tabloyu farklı isimle iki kez JOIN edebilmek için

## 1.4 DynamicFields: Yes vs No Farkı

| DynamicFields | Davranış | Ne Zaman Kullanılır |
|---|---|---|
| `Yes` | Tablonun tüm alanları otomatik dahil edilir | Tüm alanlar gerektiğinde veya SSRS gibi bağlamlar için |
| `No` (veya `<DynamicFields>` belirtilmediğinde) | Yalnızca `<Fields>` bölümünde listelenen alanlar dahil edilir | Performans optimizasyonu; yalnızca belirli alanlara ihtiyaç varsa |

## 1.5 AxQuerySimpleEmbeddedDataSource — JOIN Tanımları

Alt tablolar (JOIN) `AxQuerySimpleEmbeddedDataSource` ile tanımlanır ve üst veri kaynağının `<DataSources>` bloğuna iç içe girer:

```xml
<AxQuerySimpleEmbeddedDataSource>
    <Name>EcoResProductTranslation</Name>
    <DynamicFields>Yes</DynamicFields>
    <Table>EcoResProductTranslation</Table>
    <DataSources />               <!-- Bu data source'un kendi alt JOIN'leri -->
    <DerivedDataSources />
    <Fields />                    <!-- DynamicFields=No ise belirli alanlar -->
    <Ranges>
        <AxQuerySimpleDataSourceRange>
            <Name>LanguageId</Name>
            <Field>LanguageId</Field>
            <Status>Locked</Status>
            <Value>tr</Value>
        </AxQuerySimpleDataSourceRange>
    </Ranges>
    <JoinMode>OuterJoin</JoinMode>        <!-- Belirtilmezse InnerJoin varsayılan -->
    <UseRelations>Yes</UseRelations>      <!-- Ya UseRelations ya da Relations -->
    <Relations>
        <AxQuerySimpleDataSourceRelation>
            <Name>QueryDataSourceRelation1</Name>
            <Field>Product</Field>               <!-- Bu tablodaki alan -->
            <JoinDataSource>InventTable</JoinDataSource>  <!-- Bağlanan kaynak adı -->
            <RelatedField>RecId</RelatedField>   <!-- Bağlanan tablodaki alan -->
        </AxQuerySimpleDataSourceRelation>
    </Relations>
</AxQuerySimpleEmbeddedDataSource>
```

## 1.6 JoinMode Seçenekleri

Gerçek projedeki tüm kullanımlar:

|---|---|---|
| `InnerJoin` | Her iki tarafta da eşleşme zorunlu (varsayılan, yazılmayabilir) | Çoğu standart ilişki |
| `OuterJoin` | Sol taraf her zaman döner; sağ tarafta eşleşme yoksa NULL | `PurchTable` (fatura olmadan da satır varsa), `ProdBOM` |
| `ExistsJoin` | Eşleşme varsa sol kayıt döner; sağ taraf alanları kullanılamaz | `InventDimSales` — sadece varlık kontrolü |

**Kritik:** `JoinMode` belirtilmezse **InnerJoin** uygulanır. Her zaman açıkça belirtmek best practice'tir.

## 1.7 UseRelations vs Relations Farkı

| Yaklaşım | Açıklama | Ne Zaman |
|---|---|---|
| `<UseRelations>Yes</UseRelations>` | D365 metadata'daki tanımlı table relation'ları otomatik kullanılır; `<Relations />` boş bırakılır | Standart D365 tabloları arasında zaten tanımlı ilişki varsa |

**Gerçek Örnekler:**
- `SalesTable + SalesLine`: `<UseRelations>Yes</UseRelations>` — standart D365 ilişkisi var
- `InventTable + EcoResProductTranslation` (LanguageId=tr filtresiyle): `<Relations>` elle tanımlanmış — çünkü dil filtresi nedeniyle özel join
- `InventDimSales + InventDimPurch`: 3 ayrı `AxQuerySimpleDataSourceRelation` ile cross-join — UseRelations mümkün değil

## 1.8 AxQuerySimpleDataSourceRelation — İlişki Tanımı

```xml
<AxQuerySimpleDataSourceRelation>
    <Name>QueryDataSourceRelation1</Name>        <!-- Benzersiz ad, convention: QueryDataSourceRelation1,2,3... -->
    <Field>inventDimID</Field>                   <!-- Bu data source'taki alan -->
    <JoinDataSource>InventTransPurch</JoinDataSource>  <!-- Bağlanan data source'un <Name> değeri -->
    <RelatedField>inventDimID</RelatedField>     <!-- Bağlanan data source'taki alan -->
</AxQuerySimpleDataSourceRelation>
```

**Çok alanlı JOIN — birden fazla relation:**
```xml
<Relations>
    <AxQuerySimpleDataSourceRelation>
        <Name>QueryDataSourceRelation1</Name>
        <Field>ProdID</Field>
        <JoinDataSource>ProdBOM</JoinDataSource>
        <RelatedField>ProdID</RelatedField>
    </AxQuerySimpleDataSourceRelation>
    <AxQuerySimpleDataSourceRelation>
        <Name>QueryDataSourceRelation2</Name>
        <Field>OprNUm</Field>
        <JoinDataSource>ProdBOM</JoinDataSource>
        <RelatedField>OprNUm</RelatedField>
    </AxQuerySimpleDataSourceRelation>
</Relations>
```

## 1.9 Range (Filtre) Tanımları

```xml
<Ranges>
    <AxQuerySimpleDataSourceRange>
        <Name>LanguageId</Name>          <!-- Benzersiz ad -->
        <Field>LanguageId</Field>         <!-- Filtrelenecek alan -->
        <Status>Locked</Status>           <!-- Kullanıcı bu filtreyi değiştiremez -->
        <Value>tr</Value>                 <!-- Filtre değeri -->
    </AxQuerySimpleDataSourceRange>
</Ranges>
```

### Range Status Değerleri

| Status | Anlamı |
|---|---|
| `Locked` | Filtre sabitlenmiş, kullanıcı değiştiremez, UI'da görünmez |
| `Hidden` | Filtre aktif ama UI'da görünmez; kullanıcı erişemez ama runtime'da değiştirilebilir |
| (belirtilmez) | Varsayılan — filtre açık, kullanıcı değiştirebilir |

## 1.10 Fields — Belirli Alan Seçimi

`DynamicFields>Yes` yerine belirli alanlar seçildiğinde:

```xml
<Fields>
    <AxQuerySimpleDataSourceField>
        <Name>InventTransId</Name>       <!-- Alanın kod erişim adı -->
        <Field>InventTransId</Field>      <!-- Tablodaki gerçek alan adı -->
    </AxQuerySimpleDataSourceField>
    <AxQuerySimpleDataSourceField>
        <Name>ItemId</Name>
        <Field>ItemId</Field>
    </AxQuerySimpleDataSourceField>
</Fields>
```

## 1.11 GroupBy, Having, OrderBy

Root data source'ta boş olarak yer alır — kullanılmıyorsa boş element yeterli:

```xml
<GroupBy />
<Having />
<OrderBy />
```


# 2. AxQuerySimpleExtension - Sorgu Uzantıları

## 2.1 Temel XML Yapısı


**Namespace:** Sadece `xmlns:i="http://www.w3.org/2001/XMLSchema-instance"` — `xmlns=""` YOK


## 2.2 Mevcut Data Source'a Alan Ekleme

`<Fields>` bölümüne `AxQueryExtensionQueryDataSourceField` ile:


## 2.3 Mevcut Data Source'a Yeni Data Source Ekleme

`<DataSources>` bölümüne `AxQueryExtensionEmbeddedDataSource` ile:


## 2.5 Extension İçindeki DerivedTable Farkı

`AxQuerySimpleExtension` içinde `<Fields>` bloğunda data source ekleme yapılırken, alanlar `AxQuerySimpleDataSourceField` (AxQuery içindeki ile aynı yapı) kullanır ancak `<DerivedTable>` eklenebilir. Bu, alanın hangi gerçek tablodan geldiğini açıklar (özellikle alias kullanılan durumlarda).

---

# 3. AxMenuItemAction - Aksiyon Menü Öğeleri

## 3.1 Temel XML Yapısı ve Namespace

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxMenuItemAction xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="Microsoft.Dynamics.AX.Metadata.V1">
```

**Kritik:** Namespace `Microsoft.Dynamics.AX.Metadata.V1` — tüm MenuItem tipleri bu namespace'i kullanır (V6 veya V2 değil).

## 3.2 Element Referansı

| Element | Zorunlu | Açıklama |
|---|---|---|
| `<Name>` | Evet | Nesne adı — dosya adıyla eşleşmeli |
| `<Label>` | Evet | UI'da görünen metin — label key veya doğrudan metin |
| `<Object>` | Evet | Çalıştırılacak class'ın adı |
| `<ObjectType>` | Evet | `Class` (neredeyse her zaman) |
| `<SubscriberAccessLevel>` | Hayır | Güvenlik erişim seviyesi |
| `<NeedsRecord>` | Hayır | `Yes` ise aktif kayıt seçili olmadan buton pasif |
| `<MultiSelect>` | Hayır | `Yes` ise birden fazla kayıt seçilerek çalıştırılabilir |
| `<ConfigurationKey>` | Hayır | Hangi konfigürasyon anahtarı etkinleştirildiğinde görünür |
| `<MaintainUserLicense>` | Hayır | `Enterprise` — değiştirmek için gereken lisans |
| `<ViewUserLicense>` | Hayır | `Universal` — görmek için gereken lisans |
| `<HelpText>` | Hayır | Tooltip metni |

## 3.3 ObjectType Seçenekleri


```xml
<ObjectType>Class</ObjectType>
```

## 3.4 SubscriberAccessLevel Yapısı

```xml
<SubscriberAccessLevel>
    <Read xmlns="">Allow</Read>
</SubscriberAccessLevel>
```

**Önemli:** `<Read xmlns="">` — `xmlns=""` child element üzerinde bulunur, parent üzerinde değil. Bu V1 namespace'in gerektirdiği bir yapıdır.

### Grant Seçenekleri

| Grant Tipi | Anlamı |
|---|---|
| `<Update xmlns="">Allow</Update>` | Güncelleme erişimi |
| `<Create xmlns="">Allow</Create>` | Oluşturma erişimi |
| `<Delete xmlns="">Allow</Delete>` | Silme erişimi |
| `<Invoke xmlns="">Allow</Invoke>` | Class/method çağırma erişimi — Action/Output için |

Birden fazla grant bir arada kullanılabilir.


## 3.6 ConfigurationKey Kullanımı

```xml
<ConfigurationKey>Prod</ConfigurationKey>    <!-- Üretim modülü etkin olduğunda -->
<ConfigurationKey>Currency</ConfigurationKey> <!-- Para birimi özelliği etkin olduğunda -->
```

Configuration key, menu item'ın hangi D365 modülü/feature aktifken görüneceğini belirler.

## 3.7 NeedsRecord ve MultiSelect

```xml
<!-- Sadece bir kayıt seçiliyken aktif -->
<NeedsRecord>Yes</NeedsRecord>

<!-- Birden fazla kayıt seçilerek çalıştırılabilir -->
<MultiSelect>Yes</MultiSelect>
<NeedsRecord>Yes</NeedsRecord>
```

`NeedsRecord>Yes` + `MultiSelect>Yes` kombinasyonu form'daki grid'de birden fazla satır seçilip topluca işlem yapılabilmesini sağlar.

# 4. AxMenuItemDisplay - Görüntüleme Menü Öğeleri

## 4.1 Temel Yapı


**Kritik Fark:** `AxMenuItemAction`'dan farklı olarak `<ObjectType>` elementi YAZILMAZ. Display item her zaman bir form açar.

## 4.2 AxMenuItemDisplay'e Özgü Ek Elementler

| Element | Açıklama | Örnek |
|---|---|---|
| `<OpenMode>` | Form'un hangi modda açılacağı | `Edit` — düzenleme modunda |
| `<NeedsRecord>` | Aktif kayıt gereksinimi | `Yes` |
| `<HelpText>` | Tooltip | `SQL Execute form` |
| `<DisabledResource>` | Devre dışı ikon kaynak ID'si | `0` |
| `<NormalResource>` | Normal ikon kaynak ID'si | `0` |

# 5. AxMenuItemOutput - Çıktı Menü Öğeleri

## 5.1 Temel Yapı


**Action ile farkı:** Yapı neredeyse aynıdır ancak `<ObjectType>Class</ObjectType>` zorunludur ve `<Object>` bir **Controller** sınıfını işaret eder.

## 5.2 Controller Class ile Bağlantı

Output MenuItem → Controller Class → Report adı zinciri:


# 6. AxReport - SSRS Rapor Tanımları

## 6.1 Namespace ve Temel Yapı

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxReport xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="Microsoft.Dynamics.AX.Metadata.V2">
```

**Kritik:** Namespace `Microsoft.Dynamics.AX.Metadata.V2` — V1 (MenuItem) ve V6 (Form) değil.

## 6.3 DataSets Yapısı


**DataSourceType Değerleri:**

| Değer | Açıklama |
|---|---|
| `ReportDataProvider` | DP class kullanır — en yaygın pattern |
| `Query` | Doğrudan AX Query kullanır |
| `AXDataProvider` | Tablo/View'dan doğrudan okur |

## 6.4 Query Formatı — DP Class ve TmpTable Bağlantısı


**Format:** `SELECT * FROM [DPClassName].[TmpTableName]`
- `DPClassName` → DP class'ının adı (classStr referansı gibi)
- `TmpTableName` → DP class'ının işlediği TempDB tablo adı

## 6.5 Alias Formatı — DataSet Fields


**Alias Formatı Analizi:**

| Format | Açıklama |
|---|---|
| `TmpTable.1.FieldName` | Standart alan alias'ı — `.1.` sabit bir indeks/versiyon numarasıdır |
| `TmpTable.1.EnumField:LABEL(...)` | Enum alanının etiket (görünen) değerini verir |
| `TmpTable.1.EnumField:NAME(...)` | Enum alanının kod adını verir |

**Kritik:** `.1.` sayısı değişmez — her zaman `1` kullanılır. Bu D365 SSRS veri bağlama protokolünün bir parçasıdır.

## 6.7 Standart AX Parametreleri (Her Raporda Bulunan)

```xml
<Parameters>
    <AxReportDataSetParameter>
        <Name>AX_PartitionKey</Name>
        <Alias>AX_PartitionKey</Alias>
        <DataType>System.String</DataType>
        <Parameter>AX_PartitionKey</Parameter>
    </AxReportDataSetParameter>
    <AxReportDataSetParameter>
        <Name>AX_CompanyName</Name>
        <Alias>AX_CompanyName</Alias>
        <DataType>System.String</DataType>
        <Parameter>AX_CompanyName</Parameter>
    </AxReportDataSetParameter>
    <AxReportDataSetParameter>
        <Name>AX_UserContext</Name>
        <Alias>AX_UserContext</Alias>
        <DataType>System.String</DataType>
        <Parameter>AX_UserContext</Parameter>
    </AxReportDataSetParameter>
    <AxReportDataSetParameter>
        <Name>AX_RenderingCulture</Name>
        <Alias>AX_RenderingCulture</Alias>
        <DataType>System.String</DataType>
        <Parameter>AX_RenderingCulture</Parameter>
    </AxReportDataSetParameter>
    <AxReportDataSetParameter>
        <Name>AX_ReportContext</Name>
        <Alias>AX_ReportContext</Alias>
        <DataType>System.String</DataType>
        <Parameter>AX_ReportContext</Parameter>
    </AxReportDataSetParameter>
    <AxReportDataSetParameter>
        <Name>AX_RdpPreProcessedId</Name>
        <Alias>AX_RdpPreProcessedId</Alias>
        <DataType>System.String</DataType>
        <Parameter>AX_RdpPreProcessedId</Parameter>
    </AxReportDataSetParameter>
</Parameters>
```

Bu 6 parametre D365 SSRS altyapısının çalışması için zorunludur ve her ReportDataProvider raporunda bulunur.

## 6.8 DefaultParameterGroup — Sistem Parametreleri Tanımı

```xml
<DefaultParameterGroup>
    <Name xmlns="">Parameters</Name>
    <ReportParameterBases xmlns="">
        <AxReportParameterBase xmlns=""
            i:type="AxReportParameter">
            <Name>AX_PartitionKey</Name>
            <AllowBlank>true</AllowBlank>
            <Nullable>true</Nullable>
            <UserVisibility>Hidden</UserVisibility>
            <DefaultValue />
            <Values />
        </AxReportParameterBase>
        <AxReportParameterBase xmlns=""
            i:type="AxReportParameter">
            <Name>AX_CompanyName</Name>
            <UserVisibility>Hidden</UserVisibility>
            <DefaultValue />
            <Values />
        </AxReportParameterBase>
        <!-- AX_UserContext, AX_RenderingCulture, AX_ReportContext, AX_RdpPreProcessedId -->
    </ReportParameterBases>
</DefaultParameterGroup>
```

`<UserVisibility>Hidden</UserVisibility>` — bu parametreler kullanıcıya gösterilmez, sistem tarafından otomatik doldurulur.

## 6.9 Designs — AxReportPrecisionDesign

```xml
<Designs>
    <AxReportDesign xmlns=""
        i:type="AxReportPrecisionDesign">
        <Name>Report</Name>
        <DataNavigation>DocumentMap</DataNavigation>   <!-- Opsiyonel -->
        <Text>
            &lt;?xml version="1.0" encoding="utf-8"?&gt;
            &lt;Report xmlns="http://schemas.microsoft.com/sqlserver/reporting/2016/01/reportdefinition"
                     xmlns:rd="http://schemas.microsoft.com/SQLServer/reporting/reportdesigner"&gt;
            &lt;AutoRefresh&gt;0&lt;/AutoRefresh&gt;
            &lt;DataSources&gt;
                &lt;DataSource Name="AutoGen__ReportDataProvider"&gt;
                    &lt;Transaction&gt;true&lt;/Transaction&gt;
                    &lt;ConnectionProperties&gt;
                        &lt;DataProvider&gt;AXREPORTDATAPROVIDER&lt;/DataProvider&gt;
                    &lt;/ConnectionProperties&gt;
                &lt;/DataSource&gt;
            &lt;/DataSources&gt;
            ...
            &lt;/Report&gt;
        </Text>
    </AxReportDesign>
</Designs>
```

**Text Alanı Kuralları:**
- Tüm içerik HTML-encoded XML'dir: `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`
- SSRS RDL formatındaki tam rapor tanımını içerir
- `xmlns="http://schemas.microsoft.com/sqlserver/reporting/2016/01/reportdefinition"` — SQL Server 2016 RDL şeması
- Visual Studio SSRS Report Designer ile oluşturulur ve D365 metadata formatına çevrilir

**Design Adı:** Her zaman `<Name>Report</Name>` — Controller class'ta `ssrsReportStr(RaporAdı, Report)` ile referans verilir.

**DataNavigation:** `DocumentMap` — rapor içinde belge haritası navigasyonu sağlar.

## 6.10 RDL İçindeki DataSource Yapısı

HTML-encoded text içindeki DataSource:
```
&lt;DataSource Name="AutoGen__ReportDataProvider"&gt;
    &lt;DataProvider&gt;AXREPORTDATAPROVIDER&lt;/DataProvider&gt;
&lt;/DataSource&gt;
```

Bu sabit bir D365 SSRS veri sağlayıcı tanımıdır.

## 6.11 DataSets Boş Olduğunda (Sadece Design)


```xml
<DataSets />
<DefaultParameterGroup>
    <Name xmlns="">Parameters</Name>
    <ReportParameterBases xmlns="" />
</DefaultParameterGroup>
<Designs>
    <AxReportDesign xmlns="" i:type="AxReportPrecisionDesign">
        <Name>Report</Name>
        <Text>...RDL içeriği...</Text>
    </AxReportDesign>
</Designs>
```

Bu durum, rapor tasarımının ayrı tutulduğu ve DataSet'in başka bir rapor dosyasında tanımlandığı durumlarda görülür (aynı isimdeki iki rapor: `CurrecyPayment` ve `CurrencyPaymentReport`).

## 6.13 Caption ile Alan Başlıkları


Caption opsiyoneldir. Verildiğinde rapor designer'da sütun başlığı olarak otomatik kullanılabilir.

---

## EKLER

### EK A: MenuItem Tiplerine Göre Karşılaştırma Tablosu

| Özellik | AxMenuItemDisplay | AxMenuItemAction | AxMenuItemOutput |
|---|---|---|---|
| Namespace | V1 | V1 | V1 |
| `<Object>` içeriği | Form adı | Class adı | Controller class adı |
| `<ObjectType>` | **YAZILMAZ** | `Class` | `Class` |
| Amacı | Form açmak | İş mantığı çalıştırmak | SSRS raporu çalıştırmak |
| `<NeedsRecord>` | Kullanılabilir | Kullanılabilir | Nadiren |
| `<MultiSelect>` | Nadiren | Kullanılabilir | - |
| `<OpenMode>` | `Edit` / `View` | - | - |

### EK B: AxQuery Dosya İsimlendirme

- Dosya adı ve `<Name>` değeri birebir eşleşmeli

### EK C: AxReport DataSet Alanı Olmadan Field Ekleme


`DisableAutoCreateInDataRegion>true` — rapor designer'da tablo bölgesine otomatik sütun eklenmesini önler. Alan tanımlıdır ama varsayılan olarak RDL'e yerleştirilmez; yalnızca manuel yerleştirme için kullanılır.

# AxSecurityPrivilege

## Genel Yapı

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxSecurityPrivilege xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>PrivilegeAdi</Name>
    <!-- Label opsiyonel -->
    <DataEntityPermissions>
        <AxSecurityDataEntityPermission>
            <Grant>
                <Correct>Allow</Correct>
                <Create>Allow</Create>
                <Delete>Allow</Delete>
                <Read>Allow</Read>
                <Update>Allow</Update>
            </Grant>
            <Name>EntityAdi</Name>
            <Fields />
            <Methods />
        </AxSecurityDataEntityPermission>
    </DataEntityPermissions>
    <DirectAccessPermissions />
    <EntryPoints>
        <AxSecurityEntryPointReference>
            <Name>EntryPointRefAdi</Name>
            <Grant>
                <Read>Allow</Read>
            </Grant>
            <ObjectName>MenuItemAdi</ObjectName>
            <ObjectType>MenuItemAction</ObjectType>
            <Forms />
        </AxSecurityEntryPointReference>
    </EntryPoints>
    <FormControlOverrides />
</AxSecurityPrivilege>
```

### Kural: Her privilege ya EntryPoints ya DataEntityPermissions içerir, ikisi birden nadir
1. **Form/Action tabanlı**: EntryPoints dolu, DataEntityPermissions boş → form veya class erişim yetkisi
2. **Data entity tabanlı**: DataEntityPermissions dolu, EntryPoints boş → OData/Data Management erişim yetkisi

---

### 7-24. Diğer DataEntity Privilege Örnekleri (Özet)

Aşağıdaki tüm privilege'lar aynı kalıbı izler. **Maintain** = tam erişim (Correct+Create+Delete+Read+Update), **View** = sadece Read:

| Privilege Adı | Entity Adı | Erişim Tipi |
|---|---|---|

---

## EntryPoint ObjectType Değerleri

| ObjectType | Kullanım | ObjectName |
|---|---|---|
| `MenuItemAction` | Bir class çalıştıran menü öğesi | Class-tabanlı menuitem adı |
| `MenuItemDisplay` | Form açan menü öğesi | Form-tabanlı menuitem adı |
| `MenuItemOutput` | Rapor çalıştıran menü öğesi | Controller-tabanlı menuitem adı |

---

## Grant Seçenekleri

### EntryPoint Grant Değerleri

| Grant Alanı | Allow | Deny | Unset |
|---|---|---|---|
| `Read` | Okuma izni verilir | Okuma engellenir | Üst seviyeden miras alınır |
| `Update` | Güncelleme izni | Engellenir | Miras |
| `Create` | Oluşturma izni | Engellenir | Miras |
| `Delete` | Silme izni | Engellenir | Miras |
| `Correct` | Düzeltme izni (finansal tablolar için) | Engellenir | Miras |
| `Invoke` | Çalıştırma izni | Engellenir | Miras |

- `MenuItemAction` için sadece `Read: Allow` → class'ı çalıştırma izni yeterli
- `MenuItemDisplay` için Maintain pattern → Correct+Create+Delete+Read+Update hepsi Allow
- `MenuItemDisplay` için View pattern → yalnızca Read: Allow

**Unset**: `<Grant>` bloğu hiç yazılmaz veya ilgili element eksik bırakılır. Bu durumda erişim hakkı `Duty` veya `Role` seviyesinden miras alınır.

### DataEntityPermission Grant Değerleri

Aynı grant isimleri kullanılır. Entity için `Correct` genellikle özel finansal düzeltme (correction) operasyonlarını temsil eder.

---

## DataEntityPermissions Yapısı

```xml
<DataEntityPermissions>
    <AxSecurityDataEntityPermission>
        <Grant>
            <Correct>Allow</Correct>   <!-- Finansal düzeltme -->
            <Create>Allow</Create>     <!-- Insert -->
            <Delete>Allow</Delete>     <!-- Delete -->
            <Read>Allow</Read>         <!-- Select -->
            <Update>Allow</Update>     <!-- Update -->
        </Grant>
        <Name>EntityAdi</Name>         <!-- AxDataEntityView'ün adı -->
        <Fields />                     <!-- Alan bazlı kısıtlama (opsiyonel) -->
        <Methods />                    <!-- Method bazlı kısıtlama (opsiyonel) -->
    </AxSecurityDataEntityPermission>
</DataEntityPermissions>
```


---

## DirectAccessPermissions


---

## FormControlOverrides


---

# AxSecurityDuty

## Yapı Analizi

- `<Name>`: Duty adı — suffix `Duty` ile biter
- `<Privileges>`: Bir veya daha fazla `<AxSecurityPrivilegeReference>` içerir
- Her `<AxSecurityPrivilegeReference>` sadece `<Name>` içerir (privilege adı referansı)
- Duty'de doğrudan grant tanımlanmaz; grant'lar privilege seviyesinde belirlenir

## Birden Fazla Privilege Referansı


---

# AxSecurityRole

## SubRoles


---

# AxSecurityDutyExtension


## Genel Yapı (Referans için)

Mevcut bir standart Duty'ye yeni privilege eklemek için kullanılır:

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxSecurityDutyExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>StandartDutyAdi.ModelAdi</Name>
    <Privileges>
        <AxSecurityPrivilegeReference>
            <Name>YeniPrivilegeAdi</Name>
        </AxSecurityPrivilegeReference>
    </Privileges>
</AxSecurityDutyExtension>
```

- `<Name>` dosya adındaki `.xml` olmadan aynı değer

---

# AxSecurityRoleExtension


## Genel Yapı (Referans için)

Mevcut bir standart Role'e yeni duty veya privilege eklemek için kullanılır:

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxSecurityRoleExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>StandartRolAdi.ModelAdi</Name>
    <Duties>
        <AxSecurityDutyReference>
            <Name>YeniDutyAdi</Name>
        </AxSecurityDutyReference>
    </Duties>
    <Privileges>
        <AxSecurityPrivilegeReference>
            <Name>YeniPrivilegeAdi</Name>
        </AxSecurityPrivilegeReference>
    </Privileges>
</AxSecurityRoleExtension>
```

---

# AxSecurityPolicy


## Genel Yapı (Referans için)

D365 F&O'da Security Policy, `Extensible Data Security (XDS)` mekanizmasıdır.
Role-based security'yi tamamlar; menü/form erişimi vermek yerine kullanıcının hangi kayıtları görebileceğini veya hangi kayıtlar üzerinde işlem yapabileceğini kısıtlar.

### XDS Temel Mantığı

- `PrimaryTable`: Security query'nin ana/root tablosu
- `Query`: Erişilebilen kayıt kümesini tanımlayan sorgu
- `ConstrainedTables`: Filtre uygulanacak bağlı tablo veya view'lar
- `Context`: Policy'nin hangi durumda devreye gireceği (`RoleName`, `RoleProperty`, uygulama context'i)

Policy query, ilgili constrained tablo işlemlerinde SQL `WHERE` veya `ON` koşuluna eklenir.
Birden fazla policy aynı anda geçerliyse sonuç birleşim (`union`) değil kesişim (`intersection`) olur.

## Genel Yapı (Gerçek D365 FO Referansına Yakın)


### ConstrainedTables Örneği

Bir policy yalnızca `PrimaryTable`'ı değil, bağlı tablo ve view'ları da kısıtlayabilir:

```xml
<ConstrainedTables>
    <AxSecurityPolicyConstrainedEntity xmlns=""
        i:type="AxSecurityPolicyConstrainedTable">
        <Constrained>Yes</Constrained>
        <Name>PurchLine</Name>
        <ConstrainedTables />
        <TableRelation>VendTable</TableRelation>
    </AxSecurityPolicyConstrainedEntity>
    <AxSecurityPolicyConstrainedEntity xmlns=""
        i:type="AxSecurityPolicyConstrainedExpression">
        <Constrained>Yes</Constrained>
        <Name>ContactPerson</Name>
        <ConstrainedTables />
        <Value>(VendTable.Party = ContactPerson.ContactForParty)</Value>
    </AxSecurityPolicyConstrainedEntity>
</ConstrainedTables>
```

### Sık Kullanılan Alanlar

| Alan | Açıklama |
|---|---|
| `ConstrainedTable` | Policy'nin primary table üzerinde aktif olup olmadığını belirtir (`Yes/No`) |
| `Enabled` | Policy'nin runtime'da devrede olup olmadığını belirtir; bazı standart örneklerde yer alır |
| `PrimaryTable` | Policy query'nin ana/root tablosu |
| `Query` | Kayıt filtresini tanımlayan query |
| `ContextType` | En sık `RoleName` veya `RoleProperty`; policy'nin ne zaman uygulanacağını belirler |
| `RoleName` | Policy'nin belirli bir role atanmış kullanıcılar için çalışmasını sağlar |
| `ContextString` | Application context veya `RoleProperty` senaryolarında kullanılan anahtar değer |
| `UseNotExistJoin` | Policy query'nin `exists join` yerine `not exists join` mantığıyla uygulanmasını sağlar |
| `ConstrainedTables` | Primary table dışındaki tablo/view kısıtlarını tanımlar |

### Operation Değerleri

| Operation | Açıklama |
|---|---|
| `AllOperations` | Select, insert, update, delete dahil tüm işlemler |
| `Insert` | Sadece insert |
| `Update` | Sadece update |
| `Delete` | Sadece delete |
| `InsertUpdateDelete` | Yazma işlemleri |
| `(boş/omit)` | Standart örneklerde çoğu zaman select-only policy bu şekilde serialize edilir |

### ContextType Senaryoları

| ContextType | Açıklama |
|---|---|
| `RoleName` | Policy sadece belirtilen role sahip kullanıcılar için uygulanır |
| `RoleProperty` | Rol üzerindeki `ContextString` ile policy `ContextString` eşleşirse uygulanır |
| `ContextString` | Microsoft Learn'de application context için kullanılır; uygulama kodu `XDS::SetContext(...)` ile context set ederek policy'yi tetikleyebilir. Standart metadata örneklerinde bu senaryoda çoğu zaman yalnızca `ContextString` alanı serialize edilir |

### Dikkat Edilecek Noktalar

- XDS, role-based security'nin yerine geçmez; onu kayıt filtresi ile tamamlar.
- Yanlış tasarlanmış policy query'leri ciddi performans etkisi yaratabilir.
- Aynı anda birden fazla policy varsa tüm policy'ler birlikte uygulanır.
- `XDSDataAccessPolicyBypassRole` atanmış kullanıcılar XDS filtrelerini bypass eder.
- Microsoft dokümantasyonuna göre financial dimensions için XDS kullanılmamalıdır.

---

# AxEventSubscription


## Genel Yapı (Referans için)

Event subscription'lar, tablo veya class olaylarına statik event handler metodlarını bağlar. Genellikle X++ class dosyalarındaki `[DataEventHandler]` ve `[FormControlEventHandler]` attribute'ları ile inline tanımlanır; ayrı XML dosyası nadir kullanılır.

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxEventSubscription xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>EventSubscriptionAdi</Name>
    <EventHandlerClass>HandlerSinifAdi</EventHandlerClass>
    <EventHandlerMethod>HandlerMetodAdi</EventHandlerMethod>
    <EventPublisherClass>YayinlayanSinif</EventPublisherClass>
    <EventPublisherMethod>YayinlayanMetod</EventPublisherMethod>
    <EventType>PostEvent</EventType>
</AxEventSubscription>
```

### EventHandlerType Değerleri

| EventHandlerType | Açıklama |
|---|---|
| `DataEventType::Inserting` | Insert öncesi |
| `DataEventType::Inserted` | Insert sonrası |
| `DataEventType::Updating` | Update öncesi |
| `DataEventType::Updated` | Update sonrası |
| `DataEventType::Deleting` | Delete öncesi |
| `DataEventType::Deleted` | Delete sonrası |
| `DataEventType::ValidatedWrite` | validateWrite sonrası |
| `DataEventType::ValidatedDelete` | validateDelete sonrası |

# AxLabelFile

## Dosya Yapısı

Label sistemi üç dosya türünden oluşur:


## Label Dosyası Formatı (Plain Text .label.txt)

Label dosyaları XML değil, özel bir plain text formatındadır.

### Temel Format Kuralları

```
LabelId=Label metni
 ;Açıklama satırı (opsiyonel, bir boşluk + noktalı virgül ile başlar)
```

**Kurallar**:
1. Her label `LabelId=MetinDeger` formatında bir satırdır
2. Açıklama satırı (comment): tek boşluk + `;` ile başlar, hemen bir sonraki satırda
3. Açıklama satırı opsiyoneldir — bazı label'lar açıklamasız bırakılmıştır
4. Dosya encoding: UTF-8 (Türkçe karakterler için zorunlu)
5. Label ID büyük/küçük harf duyarlıdır
6. Boş satır dosyada görünmez (satırlar bitişik)
7. Son satır boş bir satırla biter

---

## Label ID Formatları


### Format 3: Doğrudan Anlamlı İsim (Prefix yok)


---

## tr-TR ve en-US Ayrımı

### Birlikte İnceleme

| Label ID | en-US | tr-TR |
|---|---|---|
| `ProcessSuccessful` | `The operation has been completed!` | `İşlem tamamlandı!` |
| `ProcessCancelled` | `The operation could not be completed!` | `İşlem gerçekleştirilemedi!` |


### en-US'ta olup tr-TR'de olmayan label'lar


---

## Gerçek Label Örnekleri — Seçmeler (en-US)


---

## Kod İçinde Kullanım


---

# Genel D365 FO Gelistirme Desenleri

Bu bolum, `docs/MdDocuments` altindaki proje dokumanlarindan genellenmistir.

## 1. Warehouse Mobile App (WMS) Gelistirme

### Standart calisma modeli

Warehouse mobile app tarafinda tipik akiş su sirayla ilerler:

1. Mobil cihazdan gelen istek XML/container formatinda AOS tarafina gelir.
2. Aktif menu item setup'i ve calisacak mode/process cozulur.
3. Ilgili `WhsWorkExecuteDisplay*` sinifi veya `ProcessGuide` controller secilir.
4. Onceki ekrandan gelen input parse edilir.
5. Validation ve business logic calisir.
6. Sonraki ekranin container'i olusturulur.
7. Sonuc tekrar cihaza gonderilir.

### Temel bilesenler

| Bilesen | Rol |
|---|---|
| `WhsWorkExecuteDisplay` | Legacy mobil akisin temel base/helper sinifi |
| `WhsWorkExecuteDisplay...` turevleri | Belirli mode veya workflow ekranlarini yonetir |
| `WHSRFPassthrough` | Step'ler arasinda key/value session-state tasir |
| `WHSWorkExecuteMode` | Hangi execution class'inin secilecegini belirler |
| `...Controls` class'lari | Control name, pass key ve teknik sabitleri tutar |
| `WhsControl` / `WHSField` | Gercekten yeni field/control tipi eklenecekse kullanilir |
| `ProcessGuideController` / `ProcessGuideStep` / `ProcessGuidePageBuilder` | Modern, parcali warehouse mobile framework |

### Legacy `displayForm()` vs `ProcessGuide`

| Desen | Ne zaman uygun | Artisi | Dikkat |
|---|---|---|---|
| Legacy `displayForm()` + `WHSRFPassthrough` | Mevcut standart akisa kucuk bir adim veya validation enjekte edilecekse | Mevcut flow'a hizli CoC girisi saglar | State temizligi zor, regression riski daha yuksek |
| `ProcessGuide` | Yepyeni, cok adimli ve uzun omurlu mobil surec gelistirilecekse | Step/page/action sorumluluklari ayrisir | Ilk kurulum maliyeti daha yuksektir |

Pratik kurallar:

- Legacy bir `WhsWorkExecuteDisplay*` sinifina girmeden once declaration seviyesinde `SysObsolete` notu var mi kontrol et.
- Menu item tarafinda `Use process guide = Yes` ise yeni gelistirmeyi legacy `displayForm()` tarafina zorlamamaya calis.
- Mevcut bir standard flow'a sadece ek step veya ek kontrol eklenecekse legacy CoC halen gecerli bir secenektir.

### `displayForm()` CoC deseni

Legacy akista ana giris noktasi genellikle `displayForm(container _con, str _buttonClicked)` metodudur.

```x++
[ExtensionOf(classStr(WHSWorkExecuteDisplayUserDirected))]
final class MyWorkExecuteDisplayUserDirected_Extension
{
    #WHSWorkExecuteControlElements
    #WHSWorkExecuteDisplayCases
    #WHSRF

    container displayForm(container _con, str _buttonClicked)
    {
        container conForNext = _con;
        boolean handled = false;
        container customRet = conNull();
        WHSRFPassthrough localPass = WHSRFPassthrough::create(conPeek(_con, 2));

        this.mergeControlDataIntoPass(_con, localPass);

        if (localPass.exists(#WorkId))
        {
            WHSWorkLine workLine = this.getWorkLineFromPass(localPass);

            if (workLine.RecId && this.isCustomStepNeeded(workLine))
            {
                [handled, customRet, conForNext] =
                    this.processCustomStep(_con, _buttonClicked, localPass, workLine);
            }
        }

        container ret = next displayForm(conForNext, _buttonClicked);

        return handled ? customRet : this.checkInitialEntry(ret);
    }
}
```

Kritik notlar:

- `next displayForm(...)` cagrisi korunmalidir; custom logic standart akisi tamamen bypass etmemelidir.
- `WHSRFPassthrough` cogu legacy akista `conPeek(_con, 2)` ile gelir.
- Kontrol tuple'lari pratikte 3. elemandan itibaren parse edilir.
- Text input degeri bazi legacy flow'larda tuple icinde sabit pozisyondan okunur; bu framework kontratini keyfi degistirme.

### `WHSRFPassthrough` ve state management

`WHSRFPassthrough`, RF mobil step'leri arasinda state tasiyan bir key/value wrapper'idir.

```x++
public static str MyBoxSelectionStepKey() { return 'MyBoxSelectionStep'; }
public static str MyDataEntryStepKey()    { return 'MyDataEntryStep'; }
public static str MyQtyEntryStepKey()     { return 'MyQtyEntryStep'; }

public static int MyStepBoxSelection()    { return 100; }
public static int MyStepDataEntry()       { return 101; }
public static int MyStepQtyEntry()        { return 102; }

private void clearCustomStepKeys(WHSRFPassthrough _pass)
{
    this.removeKeyIfExists(_pass, MyWorkExecuteDisplay::MyBoxSelectionStepKey());
    this.removeKeyIfExists(_pass, MyWorkExecuteDisplay::MyDataEntryStepKey());
    this.removeKeyIfExists(_pass, MyWorkExecuteDisplay::MyQtyEntryStepKey());
}
```

Onemli gozlemler:

- Legacy akislarda hem method icinde olusturulan `localPass` hem de class-level `pass` birlikte kullanilabilir.
- Auto-confirm veya final completion gibi flag'ler iki pass'e birden yazilmak zorunda olabilir.
- Her state gecisinde onceki step key'leri temizlenmezse ekran sonsuz donguye veya yanlis state'e dusebilir.
- String sabitleri extension class icinde tekrar kullanmak icin `public static str` metod deseni pratik ve guvenlidir.

### `buildControl()` ile UI kurma

WMS legacy UI, `buildControl()` ile container tuple'lari olusturularak kurulir.

```x++
container ret = conNull();

ret += [this.buildControl(#RFLabel, 'MyHeader', 'Enter value',
    1, '', #WHSRFUndefinedDataType, '', 0)];

ret += [this.buildControl(#RFText, 'MyInput', 'Serial number',
    1, '', extendedTypeNum(InventSerialId), '', 0)];

ret += [this.buildControl(#RFButton, #RFOK, '@SYS5473',
    1, '', #WHSRFUndefinedDataType, '', 1)];

ret = this.updateModeStepPass(ret, mode,
    MyWorkExecuteDisplay::MyStepDataEntry(), _pass);
```

Kurallar:

- Legacy display extension siniflarinda `#WHSWorkExecuteControlElements`, `#WHSWorkExecuteDisplayCases` ve `#WHSRF` macro'lari genellikle zorunludur.
- Her form donusunde `updateModeStepPass()` cagrisi yapilmali; aksi halde mode, step ve pass bilgisi eksik kalir.
- Mumkunse standart EDT'ler (`InventSerialId`, `InventBatchId`, `Qty`, `ItemId`) ve standart control isimleri kullan; bu, field name/priority ve icon-title eslesmelerini kolaylastirir.

### Custom work type framework

WMS custom work type deseni genellikle su bilesenlerden olusur:

1. Setup'ta benzersiz bir custom work type code tanimlanir.
2. Work template'te sira genellikle `Pick -> Custom -> Put` veya benzeri seklindedir.
3. UI/state yonetimi `WhsWorkExecuteDisplay*` tarafinda yapilir.
4. Final completion dogrulamasi `WhsIWorkTypeCustomProcessor` ile yapilir.
5. Framework'un bekledigi placeholder method `WHSWorkCustomData` extension'inda bulunur.

```x++
[WhsWorkTypeCustomProcessorFactory('MyCustomWorkTypeCode')]
public class MyCustomWorkTypeProcessor implements WhsIWorkTypeCustomProcessor
{
    public void process(WhsWorkTypeCustomProcessParameters _parameters)
    {
        if (!MyWorkExecuteDisplay::isCustomStepComplete(_parameters.workLine))
        {
            throw error('Custom work step is not complete.');
        }
    }
}

[ExtensionOf(classStr(WHSWorkCustomData))]
public final class MyWHSWorkCustomData_Extension
{
    public void MyCustomDataMethod()
    {
    }
}
```

Kurallar:

- Setup'taki code, processor factory attribute'u ve framework'un cagiracagi custom data method'i birbiriyle tutarli olmali.
- `WhsIWorkTypeCustomProcessor` final dogrulama icindir; UI kurma ve persistence bu sinifa yigilmamalidir.
- Yeni factory attribute, control veya field tipi eklediysen `SysExtension` cache etkisini mutlaka kontrol et.

### Menu item setup, field names ve step titles

Kod yazmadan once su setup eksenlerini cikar:

- Menu item tipi: inquiry, indirect, work creation veya existing work execution
- `Directed by`: user directed, system directed, cluster, cycle count vb.
- `Use process guide`, `Cluster profile`, `Generate license plate`, `Show work line list`, `Display inventory status`, `Validated user directed field`
- Field names and priorities setup'i
- Step title / instruction override setup'i

Pratik sonuc:

- Bazi istekler aslinda X++ degisiklik degil setup duzeltmesidir.
- Yeni bir field icin bazen yalnizca `buildControl()` yeterlidir; bazen `WhsControl` + `WHSField` tanimi gerekir.
- UX iyilestirmelerinin bir kismi step title/instruction tarafinda kodsuz cozulur.

### Location directive strategy pattern

Yeni bir location directive strategy icin asgari olarak su uc bilesen gerekir:

1. `WHSLocDirStrategy` enum extension
2. `WhsLocationDirectiveStrategy`'den tureyen X++ class
3. Gerekliyse agregat `AxQuery` + `AxView`

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxEnumExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
	<Name>WHSLocDirStrategy.MyModel</Name>
	<EnumValues>
		<AxEnumValue>
			<Name>MyStrategy</Name>
			<Label>My strategy</Label>
		</AxEnumValue>
	</EnumValues>
	<PropertyModifications />
	<ValueModifications />
</AxEnumExtension>
```

```x++
[WhsLocationDirectiveStrategyFactory(WhsLocDirStrategy::MyStrategy)]
class MyWhsLocationDirectiveStrategy extends WhsLocationDirectiveStrategy
{
    public boolean validate(
        WHSLocDirTable  _locDirTable,
        WHSLocDirLine   _locDirLine,
        WHSLocDirAction _locDirAction)
    {
        if (_locDirTable.WorkType != WHSWorkType::Put
            && _locDirTable.WorkType != WHSWorkType::Pick)
        {
            return checkFailed(strFmt('@WAX4602', _locDirAction.LocDirStrategy, _locDirTable.WorkType));
        }

        return true;
    }

    public boolean modifyPutLocDirActionQuery(WhsLocationDirectiveActionQuery _actionQuery, Query _query)
    {
        return this.modifyQuery(_actionQuery, _query);
    }

    public boolean modifyPickLocDirActionQuery(WhsLocationDirectiveActionQuery _actionQuery, Query _query)
    {
        return this.modifyQuery(_actionQuery, _query);
    }
}
```

Query seviyesinde sik kullanilan kurallar:

- Root datasource genellikle `WMSLocation` olur.
- Bos lokasyon bulma deseninde `NoExistsJoin` ile dolu lokasyonlar elenir.
- `InventUseDimOfInventSumToggle` aciksa `InventSum` dogrudan baglanir; degilse `InventDim -> InventSum` zinciri kullanilir.
- `ClosedQty = No` ve `PhysicalInvent > 0` filtreleri ile stoklu lokasyonlar dislanir.
- Acik veya islenmekte olan incoming `WHSWorkLine` kayitlari ayrica dislanmalidir.
- `allowSplit` senaryosunda `WHSTmpWorkLine` da hesaba katilmalidir.
- Kolon, zone veya item bazli toplamlara gore filtreleme yapilacaksa agregat `AxQuery` + `AxView` okumayi belirgin sekilde kolaylastirir.

### WMS kisa checklist

1. Menu item setup'ini ve `Directed by` secimini dogrula.
2. Legacy mi `ProcessGuide` mi oldugunu belirle.
3. Display class, controls class ve service/controller katmanini ayir.
4. Pass key'lerini ve integer step ID'lerini tasarla.
5. UI donusunde `updateModeStepPass()` cagir.
6. Yeni factory/attribute eklediysen cache etkisini kontrol et.
7. Farkli cihaz ve user setting kombinasyonlariyla test et.

## 2. Form Etkilesim ve Dialog Desenleri

Bu bolum, `AxForm` ve `AxFormExtension` metadata yapisina ek olarak runtime seviyesindeki genel X++ desenlerini toplar.

### Checkbox tabanli visibility + mandatory deseni

```x++
[ExtensionOf(formStr(MyForm))]
final class MyForm_Extension
{
    [FormDataFieldEventHandler(formDataFieldStr(MyForm, MyTable, MyToggle), FormDataFieldEventType::Modified)]
    public static void MyToggle_OnModified(FormDataObject _sender, FormDataFieldEventArgs _e)
    {
        FormDataSource ds = _sender.datasource();
        MyTable record = ds.cursor();
        boolean enabled = (record.MyToggle == NoYes::Yes);
        FormDataObject targetField = ds.object(fieldNum(MyTable, MyDependentField));

        if (targetField)
        {
            targetField.visible(enabled);
            targetField.mandatory(enabled);
        }

        if (!enabled)
        {
            record.MyDependentField = '';
        }
    }

    [FormDataSourceEventHandler(formDataSourceStr(MyForm, MyTable), FormDataSourceEventType::Activated)]
    public static void MyTable_OnActivated(FormDataSource _sender, FormDataSourceEventArgs _e)
    {
        MyTable record = _sender.cursor();
        boolean enabled = (record.MyToggle == NoYes::Yes);
        FormDataObject targetField = _sender.object(fieldNum(MyTable, MyDependentField));

        if (targetField)
        {
            targetField.visible(enabled);
            targetField.mandatory(enabled);
        }
    }
}
```

Kural seti:

- `Modified` eventi kullanici etkileşimini, `Activated` eventi ise mevcut kayitlari senkronize etmeyi saglar.
- Field/control null check'leri her zaman korunmali.
- Bagimli alan artik gecerli degilse, toggle kapanirken alan temizlenmelidir.

### Standart dialog acma deseni

```x++
public static Common openDialog(FormRun _callerFormRun)
{
    Args args = new Args();
    FormRun dialogFormRun;

    args.name(formStr(MyDialogForm));
    args.caller(_callerFormRun);

    dialogFormRun = classfactory.formRunClass(args);
    dialogFormRun.init();
    dialogFormRun.run();
    dialogFormRun.wait();

    if (dialogFormRun.closedOk())
    {
        return dialogFormRun.args().record();
    }

    return null;
}
```

Kullanim notlari:

- Modal dialog akisi icin `wait()` kullanilir.
- Dondurulen kayit `args().record()` uzerinden alinabilir.
- Cancel durumunu acikca ele almak gerekir; aksi halde yarim state kalabilir.

### `validateWrite()` ile kosullu zorunluluk deseni

```x++
[ExtensionOf(tableStr(MyHeaderTable))]
final class MyHeaderTable_Extension
{
    public boolean validateWrite()
    {
        boolean ret = next validateWrite();

        if (ret && this.MyToggle == NoYes::Yes && !this.MyDependentField)
        {
            ret = checkFailed('Dependent field must be specified when the toggle is enabled.');
        }

        return ret;
    }
}
```

Kurallar:

- `next validateWrite()` sonucu korunmali.
- Ilave kosul yalnizca `ret == true` ise calistirilmali.
- UI tarafinda mandatory yapsan bile tablo seviyesinde ikinci savunma kati bulunmalidir.

### Header'dan line'a varsayilan deger veya dimension tasima

```x++
[ExtensionOf(tableStr(MyLineTable))]
final class MyLineTable_Extension
{
    public void insert()
    {
        MyHeaderTable header = this.headerTable();

        if (header.MyToggle == NoYes::Yes && header.DefaultDimension)
        {
            this.DefaultDimension =
                DimensionDefaultingService::serviceMergeDefaultDimensions(
                    header.DefaultDimension,
                    this.DefaultDimension);
        }

        next insert();
    }
}
```

Notlar:

- Buffer uzerindeki varsayilanlar `next insert()` oncesinde set edilmelidir.
- `serviceMergeDefaultDimensions()` icin arguman sirasi precedence'i etkileyebilir; hedef davranisina gore dogrulanmalidir.
- Bu desen sadece dimension icin degil, header kaynakli diger varsayilan alanlar icin de uygulanabilir.

## 3. Entity Kapsam ve Tasarim Checklist'i

Bu bolum metadata XML'inden cok entity tasarim karari verirken izlenecek genel kontrolleri toplar.

### Hedef entity secimi

| Senaryo | Once bakilacak hedef |
|---|---|
| Standart tabloya yeni alan ekleme | Mevcut standart entity ailesi veya mevcut entity extension |
| Custom business tablo | Custom `AxDataEntityView` |
| Parametre veya teknik tablo | Gercek entegrasyon ihtiyacina gore karar ver |
| Mevcut entity extension zaten varsa | Yeni entity acmadan once ayni extension'i genislet |

Karar kurallari:

- Repo icinde entity gorunmuyor diye hemen "entity yok" sonucuna varma; standart paketlerde hazir entity ailesi olabilir.
- Standart tabloya alan eklendiyse, form tarafi kadar entity tarafi da ayni task kapsaminda dusunulmelidir.
- Parametre, log veya yardimci tablolar icin entity zorunlulugu entegrasyon ihtiyacina baglidir.

### Mapped, unmapped ve referans alan ayrimi

| Alan tipi | Ne zaman kullanilir | Not |
|---|---|---|
| Mapped | Alan dogrudan kaynak tablodan geliyorsa | En dusuk bakim maliyeti |
| Unmapped | Hesaplanan veya runtime doldurulan alanlarda | Doldurma noktasi dokumante edilmelidir |
| Reference / join tabanli | Alan baska datasource'tan geliyorsa | Kaynak datasource ve write-back ihtiyaci net olmalidir |

Pratik kural:

- Alanin kaynagi tablo kolonu ise `MappedField` ile basla.
- Hesaplama veya runtime assembly gerekiyorsa `UnmappedField` kullan ve doldurma zincirini belgeye yaz.
- `RefRecId` veya join tabanli alanlarda veri kaynagi ve guncelleme davranisi acikca not edilmelidir.

### Kapsam kontrolu

Entity gelistirmelerinde su ciftler birlikte dusunulmelidir:

- Header + line ciftleri (`SalesTable` + `SalesLine`, `PurchTable` + `PurchLine`)
- Product + released product ciftleri (`EcoResProduct` + `InventTable` / released product entity)
- Müşteri + vendor gibi ayni referans alani paylasan aileler

Tek bir tarafi genisletip digerini atlamak, entegrasyonda yari veri gorunmesine neden olabilir.

### Staging ve entegrasyon kanali

Staging etkisi entity tasariminin en basinda dusunulmelidir.

- Entity DMF ile kullanilacaksa staging etkisi bastan kontrol edilmelidir.
- Alanlarin OData mi, DMF mi, yalnizca UI mi kullanacagi netlestirilmelidir.
- Staging ve alan kapsam kararlari sonradan degil, tasarim asamasinda dokumante edilmelidir.

### Entity tamamlanma checklist'i

1. Kaynak tablo ve alan listesi cikarildi.
2. Hedef entity veya entity extension secimi gerekcelendirildi.
3. Standart paketlerde mevcut hedef olup olmadigi kontrol edildi.
4. Alanlar mapped / unmapped / reference olarak siniflandirildi.
5. Gerekliyse header-line veya product-release ciftleri birlikte degerlendirildi.
6. Staging ve entegrasyon kanali etkisi not edildi.
7. Guvenlik ihtiyaci icin ilgili privilege tasarimi unutulmadi.

## 4. Change Log ve Schedule Batch Desenleri

Bu desen, toplu degisiklik veya planli guncelleme gerektiren modullerde tekrar kullanilabilir.

### Merkezi change log tablosu

Tipik bir degisiklik log tablosu su alanlari icerir:

| Alan | Amac |
|---|---|
| `UserId` | Islemi yapan kullanici |
| `ActionType` | Inline edit, bulk set, overwrite, schedule vb. |
| `ActionDescription` | Insan okunabilir degisiklik ozet metni |
| `Scope` | Etkilenen is alani veya hedef kayit grubu |
| `BusinessKey` | Ilgili is anahtari veya kod |
| `OldValue` / `NewValue` | Degisen degerler |
| `TransDateTime` | Degisiklik zamani |
| `BatchJobId` | Batch ile iliskiliyse baglanti |

```x++
class MyChangeLogHelper
{
    public static void logChange(
        str      _actionType,
        str      _description,
        str      _scope,
        str      _businessKey = '',
        str      _oldValue = '',
        str      _newValue = '',
        RefRecId _batchJobId = 0)
    {
        MyChangeLog log;

        ttsBegin;
        log.UserId            = curUserId();
        log.ActionType        = _actionType;
        log.ActionDescription = _description;
        log.Scope             = _scope;
        log.BusinessKey       = _businessKey;
        log.OldValue          = _oldValue;
        log.NewValue          = _newValue;
        log.TransDateTime     = DateTimeUtil::utcNow();
        log.BatchJobId        = _batchJobId;
        log.insert();
        ttsCommit;
    }
}
```

Kurallar:

- Loglama tek helper sinifta merkezilesirse form, batch ve helper class'lari ayni audit standardini kullanir.
- Gercek veri degisikligi ve log insert'i ayni transaction scope icinde dusunulmelidir.
- Tek basina `info()` mesaji audit izi yerine gecmez.

### Pending change + schedule batch deseni

Planli bir degisikligi sonradan calistirmak icin tipik olarak ayri bir pending tablo kullanilir.

| Alan | Amac |
|---|---|
| `ChangeId` | Benzersiz planli is ID'si |
| `Title` | Kullaniciya gorunen baslik |
| `TargetDescription` | Hedef kayit grubu veya filtre aciklamasi |
| `ScheduledBy` | Planlayan kullanici |
| `ScheduledDateTime` | Planlanan calisma zamani |
| `Status` | `Scheduled`, `Queued`, `Executed`, `Failed`, `Cancelled` |
| `PackedQuery` / `TargetQuery` | Secilen kayit grubunun paketlenmis sorgusu |
| `ExecutedDateTime` | Gercek calisma zamani |
| `BatchJobId` | Ilgili batch baglantisi |

Calisma modeli:

1. Kullanici UI'da filtrelenmis kayit grubunu secer.
2. Islem parametreleri pending tabloya `Scheduled` durumda yazilir.
3. `SysOperationServiceBase` + `DataContractAttribute` + `SysOperationServiceController` ile batch queue'ya is eklenir.
4. Batch calisinca pending kayit okunur, packed query unpack edilir ve hedef kayitlar guncellenir.
5. Son durumda status `Executed`, `Failed` veya `Cancelled` olarak guncellenir.

Pratik notlar:

- Packed query saklamak, planlama anindaki secili kayit kümesini batch calisma anina tasir.
- Pending list ekraninda genellikle sadece `Scheduled` ve `Queued` durumlari gosterilir.
- Iptal davranisi gerekiyorsa status degisikligi tek yerden yonetilmelidir.

### Ne zaman bu desen uygundur?

- Toplu guncelleme aninda degil, planli saatte calisacaksa
- Kullanici seciminin sonradan tekrar oynatilmasi gerekiyorsa
- Basarili/basarisiz batch calismalari audit olarak izlenmek isteniyorsa
- UI operasyonu ile arka plan batch'i arasinda acik bir is kaydi tutulmasi isteniyorsa

---

# AxMenu ve AxMenuExtension

## AxMenu


---

## AxMenuExtension

### Genel XML Yapısı

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxMenuExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
                 xmlns="Microsoft.Dynamics.AX.Metadata.V1">
    <Name>StandartMenuAdi.ModelAdi</Name>
    <Customizations />
    <Elements>
        <AxMenuExtensionElement xmlns="">
            <!-- Parent: mevcut menü içindeki hedef öğe adı -->
            <Parent>HedefMenuOgesi</Parent>
            <!-- PositionType ve PreviousSibling: konumlama (opsiyonel) -->
            <MenuElement xmlns="" i:type="AxMenuElementSubMenu">
                <Name>YeniSubMenu</Name>
                <Label>SubMenu Etiketi</Label>
                <Elements>
                    <AxMenuElement xmlns="" i:type="AxMenuElementMenuItem">
                        <Name>MenuItemReferansAdi</Name>
                        <MenuItemName>MenuItemAdi</MenuItemName>
                        <MenuItemType>Action</MenuItemType> <!-- Display, Action, Output -->
                    </AxMenuElement>
                </Elements>
            </MenuElement>
        </AxMenuExtensionElement>
    </Elements>
    <MenuElementModifications />
    <PropertyModifications />
</AxMenuExtension>
```

---

## MenuExtension Element Tipleri

| i:type | Açıklama |
|---|---|
| `AxMenuElementSubMenu` | Yeni alt menü (grup) oluşturur, `<Elements>` içerebilir |
| `AxMenuElementMenuItem` | Tek bir menü öğesi referansı |
| `AxMenuElementMenuItemOutput` | Output menü öğesi |

---

## MenuElement Özellikleri

| Element | Zorunlu | Açıklama |
|---|---|---|
| `<Name>` | Evet | Extensionda benzersiz referans adı |
| `<MenuItemName>` | MenuItem için | Referans edilen AxMenuItemDisplay/Action/Output adı |
| `<MenuItemType>` | Opsiyonel | `Display` (varsayılan), `Action`, `Output` |
| `<Elements>` | SubMenu için | Alt öğeleri içerir |

---

## AxMenuExtensionElement Konumlama

| Element | Açıklama |
|---|---|
| `<Parent>` | Hedef menüdeki mevcut öğenin adı (örn: `Setup`, `PeriodicTasks`) |
| `<PositionType>` | `AfterItem` → `<PreviousSibling>`'dan sonra ekle |
| `<PreviousSibling>` | `AfterItem` kullanıldığında referans öğe adı |

**Parent olmadan**: `<Parent>` eksik olduğunda SystemAdministration örneğindeki gibi direkt menüye eklenir.

---

# AxMap ve AxMapExtension


## Genel Yapı (Referans için)

Map'ler birden fazla tabloyu ortak bir arayüz üzerinden birleştiren sanal yapılardır. Birden fazla tablo aynı alan adlarını map üzerinden paylaşır.

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxMap xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>MapAdi</Name>
    <Fields>
        <AxMapField xmlns="" i:type="AxMapFieldString">
            <Name>AlanAdi</Name>
        </AxMapField>
    </Fields>
    <Mappings>
        <AxMapMapping>
            <MappingTable>TabloAdi</MappingTable>
            <Fields>
                <AxMapMappingField>
                    <MapField>AlanAdi</MapField>
                    <MapFieldTo>TabloAlani</MapFieldTo>
                </AxMapMappingField>
            </Fields>
        </AxMapMapping>
    </Mappings>
</AxMap>
```

---

# AxService ve AxServiceGroup


## Genel Yapı (Referans için)

Service'ler D365 FO'da dış sistemlerle entegrasyon için SOAP/REST endpoint'leri sağlar.

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxService xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>ServiceAdi</Name>
    <ServiceOperations>
        <AxServiceOperation>
            <Name>OperasyonAdi</Name>
            <ReturnType>void</ReturnType>
        </AxServiceOperation>
    </ServiceOperations>
</AxService>
```

---

# AxConfigurationKey


## Genel Yapı (Referans için)

Configuration key'ler belirli özellikleri veya alanları etkinleştirip devre dışı bırakmak için kullanılır.


---

# X++ Best Practices - D365 F&O En İyi Uygulamalar

> Kaynak: D365_Xpp_Best_Practices.txt — Microsoft Learn, Stoneridge Software, Synoptek, Dynamics Edge, zakharov.com, Confiz, Rand Group

================================================================================
       MICROSOFT DYNAMICS 365 F&O - X++ EN İYİ UYGULAMALAR (BEST PRACTICES)
================================================================================

Hazırlanma Tarihi: 13 Mart 2026
Kaynaklar: Microsoft Learn, Stoneridge Software, Synoptek, Dynamics Edge,
           zakharov.com, Confiz, Rand Group ve diğer topluluk kaynakları

================================================================================
1. GENEL KODLAMA STANDARTLARI
================================================================================

1.1 Kod Yapısı ve Okunabilirlik
    - Her satırda yalnızca bir ifade (statement) bulunmalıdır.
    - Bir satır 140 karakteri geçmemelidir; uzun ifadeler birden fazla satıra
      bölünmelidir.
    - Varlıkları (entity) ayırmak için tek bir boş satır kullanılmalıdır.
    - Tek bir ifade olsa bile her kod bloğunun etrafına süslü parantez ({})
      konulmalıdır.
    - if, switch, for, while anahtar kelimeleri ile açılış parantezi arasına
      bir boşluk eklenmelidir.
    - Fonksiyon adı ile açılış parantezi arasında boşluk bırakılmamalıdır.
    - NOT (!) operatöründen sonra bir boşluk eklenmelidir.

1.2 Metot Tasarımı
    - Metotlar küçük ve anlaşılır olmalıdır. Her metot tek bir iyi
      tanımlanmış görevi yerine getirmelidir.
    - Metot adı, yaptığı işi kolayca anlatabilmelidir.
    - Kodda yalnızca bir başarılı dönüş noktası (return) bulunmalıdır
      (genellikle son ifade). Switch case'leri ve başlangıç koşulu
      kontrolleri istisnadır.
    - Değer ile aktarılan (by value) parametrelere değer ataması yapılmamalı
      veya manipüle edilmemelidir. Bu parametreler sabit (constant) olarak
      ele alınmalıdır.
    - Tüm metotlara erişim belirleyici (access modifier) eklenmelidir:
      public, protected veya private.

1.3 Kod Temizliği
    - Kullanılmayan değişkenler, metotlar ve sınıflar silinmelidir.
    - Yorum satırına alınmış (commented-out) çalışmayan kodlar temizlenmelidir.
    - Kodu tekrar kullanın (reuse): Aynı kod satırlarını birçok yerde
      tekrar etmek yerine bunları bir metoda taşıyın.

1.4 Hata Yönetimi
    - Kullanıcının hiçbir zaman runtime hatası yaşamaması sağlanmalıdır.
    - Hatalar programatik olarak yönetilmeli veya kullanıcıya Infolog
      üzerinden bilgi verilmelidir.
    - infolog.add() doğrudan kullanılmamalıdır. Bunun yerine error(),
      warning(), info() ve checkFailed() yönlendirme metotları kullanılmalıdır.
    - Deadlock'lardan kaçınmak için uygulama buna göre tasarlanmalıdır.

================================================================================
2. ADLANDIRMA KURALLARI (NAMING CONVENTIONS)
================================================================================

2.1 Genel Adlandırma Kuralları
    - Nesne isimleri hiyerarşik olarak oluşturulmalıdır:
      {İş Alanı Adı} + {İş Alanı Açıklaması} + {Eylem/İçerik Türü}
      Örnek: CustInvoicePrintout, PriceDiscAdmCopy
    - Uygulama nesneleri (tablo, sınıf, form, rapor vb.) karma büyük-küçük
      harf (PascalCase) ile adlandırılmalıdır.
      Örnek: AddressFormatHeading, SalesAmount
    - Metotlar, değişkenler ve sistem fonksiyonları camelCase ile
      adlandırılmalıdır (ilk harf küçük).
      Örnek: custParameters, salesAmount

2.2 Extension (Uzantı) Adlandırma Kuralları
    - Extension sınıfları için önek (prefix) yerine sonek (suffix) kullanılır.
    - Sınıf uzantıları: {TemelNesneAdı}_PRJ_Extension
      Örnek: InventJournalTrans_PRJ_Extension
    - Event handler sınıfları: SXAxxx_EventHandler
    - Form, tablo ve diğer nesneler için: {NesneAdı}_PRJ
    - Projeniz içinde tutarlı bir adlandırma kuralı belirleyin ve buna
      sadık kalın.

2.3 Tablo Buffer Değişkenleri
    - Tablo buffer değişkenleri, mümkün olduğunca tablo adıyla aynı
      adlandırılmalıdır, ancak ilk harf küçük olmalıdır.
      Örnek: CustTable tablosu için -> custTable

================================================================================
3. CHAIN OF COMMAND (CoC) EN İYİ UYGULAMALARI
================================================================================

3.1 Temel Kurallar
    - Overlayering (Microsoft'un temel kodunu doğrudan değiştirmek) asla
      yapılmamalıdır. Her zaman extension kullanılmalıdır.
    - Chain of Command, metot mantığını genişletmek için tercih edilen
      birincil mekanizmadır.
    - Extension sınıf adı "_Extension" ile bitmelidir.
    - Sınıf tanımında "final" anahtar kelimesi kullanılmalıdır.
    - [ExtensionOf(classStr(...))] niteliği (attribute) eklenmelidir.

3.2 next Çağrısı
    - Wrapper metotlarda her zaman next çağrısı yapılmalıdır.
      next çağrısı, zincirdeki bir sonraki metodu ve sonunda orijinal
      uygulamayı çağırır.
    - next çağrısı, metot gövdesindeki birinci seviye ifadelerde yer
      almalıdır.
    - next çağrısı if bloğu içinde koşullu olarak yapılamaz.
    - next çağrısı while, do-while veya for döngü ifadelerinde yapılamaz.
    - next ifadesinden önce return ifadesi yer alamaz.
    - Yalnızca [Replaceable] olarak işaretlenmiş metotlarda next çağrısı
      isteğe bağlıdır; bu durumda bile koşullu olarak atlanmalıdır.

3.3 CoC vs Event Handler Tercihi
    - Genel kural: Çekirdek mantığı değiştirmek veya genişletmek için CoC
      kullanılmalıdır.
    - Event handler'lar; belirli bir metot override'ına bağlı olmayan
      framework olaylarına (delegate, form olayları, veri olayları) tepki
      vermek için kullanılmalıdır.
    - CoC mümkün olmadığında (private metotlar, bazı kernel metotları)
      event handler kullanılmalıdır.

3.4 Performans ve Bakım
    - Extension kodunu yalın ve verimli tutun. CoC, performansa duyarlı
      süreçlere kod ekler.
    - Tek bir metot üzerinde aşırı sayıda extension zinciri oluşturmaktan
      kaçının; bu performansı ve debug sürecini olumsuz etkiler.
    - CoC extension'ları odaklı ve amaca yönelik olmalıdır.

================================================================================
4. VERİTABANI VE SORGU OPTİMİZASYONU
================================================================================

4.1 Select İfadeleri
    - SELECT * yerine yalnızca gerekli alanları belirten bir alan listesi
      (field list) kullanın. Bu sorgu süresini optimize eder.
    - firstOnly: Yalnızca ilk kaydı kullanacaksanız veya yalnızca bir kayıt
      bulunabilecekse, firstOnly niteleyicisini kullanarak performansı
      artırın.
    - WHERE yan tümcelerini ve filtreleri mümkün olduğunca erken uygulayın;
      işlenen veri miktarını azaltın.

4.2 Set Tabanlı İşlemler
    - Satır satır (row-by-row) işleme gereksiz veritabanı gidiş-dönüşleri
      yaratır ve ölçeklenmez.
    - Mümkün olduğunca SUM(), COUNT(), UPDATE_RECORDSET, DELETE_FROM ve
      INSERT_RECORDSET gibi set tabanlı işlemleri kullanın; bunlar while
      select'lerden çok daha verimlidir.
    - UPDATE_RECORDSET ve DELETE_FROM ifadelerinin etkili olması için
      gerektiğinde skipDataMethods(), skipDeleteActions() ve
      skipDatabaseLog() uygulanmalıdır.

4.3 Join Kullanımı
    - İç içe while döngüleri yerine join kullanarak yürütme süresini azaltın
      ve okunabilirliği artırın.
    - Birleştirilen tüm tablolardan değerler gerekli değilse 'exists join'
      kullanın.

4.4 İndeksleme
    - Sık filtrelenen alanlara indeks ekleyin.
    - Ancak dikkatli olun: aşırı indeksleme yazma performansını ve bakım
      yükünü olumsuz etkileyebilir.

4.5 Geçici Tablolar
    - Geçici tablolar dikkatli kullanılmalı, kapsam (scope) ve yaşam
      döngüsüne (lifecycle) dikkat edilmelidir.
    - Gereksiz geçici tablo oluşturma işlemlerini azaltarak yükü minimize edin.

================================================================================
5. YORUM (COMMENT) STANDARTLARI
================================================================================

5.1 Kod Yorumları
    - Kodunuzun ne yapması gerektiğini ve parametrelerin ne için
      kullanıldığını anlatan yorumlar ekleyin.
    - Tüm değişiklikler, benzersiz bilet numarası / görev açıklaması dahil
      olmak üzere kodun ne yaptığını açıklayan yorumlara sahip olmalıdır.
    - Önerilen yorum formatı:
      // BEGIN <Özellik/Hata Numarası> <Açıklama> <Tarih> <Baş Harfler>
      <kod buraya>
      // END <Özellik/Hata Numarası> <Açıklama> <Tarih> <Baş Harfler>

5.2 XML Başlık Yorumları
    - Her public metottan önce XML başlık yorumları oluşturulmalıdır.
    - Metot başlığının üstüne "///" yazarak otomatik oluşturulabilir:
      /// <summary>
      /// Metodun açıklaması
      /// </summary>
      /// <param name="paramAdı">Parametre açıklaması</param>
      /// <remarks>
      /// <Özellik/Hata Numarası> <Açıklama> <Tarih> <Baş Harfler>
      /// </remarks>

================================================================================
6. LABEL VE SABİT DEĞER STANDARTLARI
================================================================================

    - Kodda sabit kodlanmış (hard-coded) değerler / etiketler kullanılmamalıdır.
    - Veritabanından gelen değerler veritabanından alınmalıdır.
    - Özel modelinizde yeni bir label dosyası oluşturun.
    - Birden fazla dilde oluşturulması gerekip gerekmediğini belirleyin.
    - Tüm kullanıcıya gösterilen metinler label sistemi üzerinden
      yönetilmelidir.

================================================================================
7. TABLO TASARIM EN İYİ UYGULAMALARI
================================================================================

    - Tablo adı şu yapıyı takip etmelidir:
      Önek (Modül Adı) + Mantıksal Açıklama + Veri Türü Son Eki
      Örnekler: CustTrans, SalesLine, ProjJour, InventParameters
    - Yeni tablolar oluştururken tablo özelliklerinin (Table Group vb.)
      tablonun amacına uygun olduğundan emin olun.
    - İki tablo arasındaki her ilişki için delete action tanımlanmalıdır.
    - Kod yazmak yerine tablo delete action'larını kullanarak silme
      işlemlerinin kısıtlı mı yoksa kademeli (cascade) mi olduğunu belirtin.
    - Alan grupları (field groups), veritabanı ve form/rapor düzeyinde
      aynı gruplama yapısına sahip olmalıdır (önbellekleme performansını
      artırır).
    - Tablo metot özellikleri (delete, validateDelete vb.) varsa; X++ kodu
      yazmak yerine özellik değerlerini ayarlayarak uygulamayı tercih edin.

================================================================================
8. FORM GELİŞTİRME EN İYİ UYGULAMALARI
================================================================================

    - Form kontrol adlarını doğrudan referans almaktan kaçının.
      Kötü örnek: salestable_SalesId.enabled(false)
    - Form stil denetleyicisini (Form Style Checker) kullanarak formların
      önceden tasarlanmış şablonlara uygunluğunu doğrulayın.
    - Global seviyede tanımlanan değişken sayısını minimum tutun; bu
      performansı artırır.
    - Inline değişken bildirimi D365 F&O'da geçerlidir; mümkünse kullanın.

================================================================================
9. ERİŞİM DÜZENLEYİCİLER VE GÜVENLİK
================================================================================

    - Tüm metotlarda erişim düzenleyiciler kullanılmalıdır: public,
      protected, private.
    - Pre veya post event handler'lar için metot erişim düzenleyicilerini
      'public' olarak değiştirmeyin; bunun yerine Chain of Command kullanın.
    - Mümkün olan en kısıtlayıcı erişim düzeyini kullanın.

================================================================================
10. KAYNAK KONTROL VE PROJE YÖNETİMİ
================================================================================

    - Bir kaynak kontrol sistemi (TFS / Azure DevOps / Git) kullanımı çok
      önemlidir; kod güvenliği ve değişiklik geçmişi sağlar.
    - Düzenli olarak kodu çekin (sync) ve çakışmaları önleyin.
    - Mevcut bir nesneyi değiştirmeden önce kaynak kod yedeği alın.
    - Geliştirmeye başlamadan önce veritabanı yedeği almayı alışkanlık
      haline getirin.
    - Kod incelemesi (code review) her zaman yapılmalıdır; farklı bakış
      açıları sağlar ve hataları erken yakalar.

================================================================================
11. MODEL VE PAKET YÖNETİMİ
================================================================================

    - D365 F&O'da herhangi bir özelleştirme için model oluşturmak
      zorunludur.
    - İlgili extension'ları tek bir modelde gruplandırarak dağıtım ve
      bakımı basitleştirin.
    - Mevcut özelleştirmeleri kullanmak yerine özelleştirmeye özgü yeni
      extension oluşturun.
    - Paketler bağımsız derleme birimleridir (assembly/DLL); başka
      paketlere referans verebilirler.

================================================================================
12. PERFORMANS GENEL İPUÇLARI
================================================================================

    - Batch işlerini dikkatli tasarlayın ve zamanlayın; kötü tasarlanmış
      batch işleri önemli performans darboğazları yaratabilir.
    - Planning Optimization gibi mimari iyileştirmelerden yararlanın.
    - Yerleşik tanılama (diagnostics) ve telemetri araçlarını kullanarak
      performans eğilimlerini ve darboğazları izleyin.
    - Sık erişilen formları, uzun süren işlemleri ve kaynak yoğun
      operasyonları düzenli olarak gözden geçirin.

================================================================================
13. BEST PRACTICE DOĞRULAMA ARAÇLARI
================================================================================

    - Visual Studio'da "Best Practice checks" seçeneğini etkinleştirin:
      Tools > Options > Development
    - Derleme sırasında BP (Best Practice) ihlalleri mesaj günlüğünde
      görünür.
    - Form Stil Denetleyicisi (Form Style Checker) kullanarak formların
      şablonlara uygunluğunu doğrulayın.
    - Code Best Practice Framework (CBPF) ile kendi özel kurallarınızı
      yazabilirsiniz.
    - Günlük derlemelerde (daily builds) BP kontrollerini çalıştırarak
      kabul edilemez uygulamaları erken yakalayın.

================================================================================
14. ÖZET KONTROL LİSTESİ
================================================================================

    [x] Her satırda tek ifade, 140 karakter sınırı
    [x] Tüm kod bloklarında süslü parantez kullanımı
    [x] Metotlar küçük, odaklı ve tek bir görevi yerine getiren
    [x] Tutarlı PascalCase/camelCase adlandırma kuralları
    [x] Extension sınıflarında _Extension soneki
    [x] CoC'de her zaman next çağrısı (Replaceable hariç)
    [x] CoC event handler'lara tercih edilir
    [x] SELECT * yerine alan listesi kullanımı
    [x] firstOnly, exists join gibi optimizasyon niteleyicileri
    [x] Set tabanlı işlemler (INSERT_RECORDSET, UPDATE_RECORDSET vb.)
    [x] Hard-coded değerler yerine label sistemi
    [x] XML başlık yorumları ve anlamlı kod yorumları
    [x] Kaynak kontrol ve kod incelemesi
    [x] BP doğrulama araçlarının etkinleştirilmesi
    [x] Kullanılmayan kod ve değişkenlerin temizlenmesi
    [x] Uygun erişim düzenleyicilerin kullanılması
    [x] Tablo ilişkilerinde delete action tanımlanması
    [x] Geliştirme öncesi kaynak kod ve DB yedeği

================================================================================
                              --- SON ---
================================================================================


---

# X++ Kod Örnekleri - D365 F&O Temel İşlevler

> Kaynak: ExamplesX++.txt — community.dynamics.com, nuxulu.com, paragchapre.com, Microsoft Dynamics Community

================================================================================
   DYNAMICS 365 FINANCE & OPERATIONS - TEMEL İŞLEVLER X++ KOD ÖRNEKLERİ
================================================================================
   Hazırlanma Tarihi: 13 Mart 2026
   Kaynak: İnternet araştırması (topluluk blogları, Microsoft Dynamics Community)
   Not: Kodlar D365 F&O ortamına göre düzenlenmiştir. Demo verileri (USMF)
        kullanılmıştır. Kendi ortamınıza göre değerleri değiştirin.
================================================================================


================================================================================
1. SATIN ALMA SİPARİŞİ OLUŞTURMA (Create Purchase Order)
================================================================================

// Satın alma siparişi oluşturma - PurchTable ve PurchLine kullanarak
// Kaynak: community.dynamics.com, nuxulu.com, paragchapre.com

class CreatePurchaseOrder
{
    public static void main(Args _args)
    {
        PurchTable      purchTable;
        PurchLine       purchLine;
        InventDim       inventDim;
        NumberSeq       numberSeq;
        VendTable       vendTable = VendTable::find("US-101"); // Satıcı hesabı

        try
        {
            ttsBegin;

            // Satın alma siparişi başlığı
            numberSeq = NumberSeq::newGetNum(PurchParameters::numRefPurchId());
            numberSeq.used();

            purchTable.PurchId = numberSeq.num();
            purchTable.initValue();
            purchTable.initFromVendTable(vendTable);
            purchTable.PurchaseType = PurchaseType::Purch;
            purchTable.DocumentStatus = DocumentStatus::PurchaseOrder;

            if (!purchTable.validateWrite())
            {
                throw Exception::Error;
            }
            purchTable.insert();

            // Satın alma siparişi satırı
            purchLine.clear();
            purchLine.initFromPurchTable(purchTable);
            purchLine.ItemId = "D0001";     // Ürün numarası
            purchLine.PurchQty = 10;        // Miktar
            purchLine.PurchPrice = 100;     // Birim fiyat

            inventDim.clear();
            inventDim.InventSiteId = "1";       // Site
            inventDim.InventLocationId = "11";   // Ambar
            purchLine.InventDimId = InventDim::findOrCreate(inventDim).inventDimId;

            purchLine.createLine(true, true, true, true, true, true);

            ttsCommit;

            info(strFmt("Satın alma siparişi oluşturuldu: %1", purchTable.PurchId));
        }
        catch (Exception::Error)
        {
            error("Satın alma siparişi oluşturulurken hata oluştu.");
        }
    }
}


================================================================================
2. SATIN ALMA SİPARİŞİ ONAYLAMA (Confirm Purchase Order)
================================================================================

// PO Confirmation - PurchFormLetter sınıfı kullanılarak
// Kaynak: nuxulu.com, d365ffo.com

public void confirmPurchaseOrder(PurchId _purchId)
{
    PurchTable          purchTable = PurchTable::find(_purchId);
    PurchFormLetter     purchFormLetter;

    purchFormLetter = PurchFormLetter::construct(DocumentStatus::PurchaseOrder);
    purchFormLetter.update(purchTable, strFmt("PO-Conf-%1", _purchId));

    info(strFmt("Satın alma siparişi onaylandı: %1", _purchId));
}


================================================================================
3. SATIN ALMA SİPARİŞİ ÜRÜN GİRİŞİ (Purchase Order Product Receipt)
================================================================================

// Product Receipt kaydetme
// Kaynak: nuxulu.com, d365ffo.com

public void postProductReceipt(PurchId _purchId, PackingSlipId _packingSlipId)
{
    PurchTable          purchTable = PurchTable::find(_purchId);
    PurchFormLetter     purchFormLetter;

    purchFormLetter = PurchFormLetter::construct(DocumentStatus::PackingSlip);
    purchFormLetter.update(purchTable, _packingSlipId);

    info(strFmt("Ürün girişi kaydedildi: PO=%1, PackingSlip=%2", _purchId, _packingSlipId));
}


================================================================================
4. SATIN ALMA FATURASI KAYDETME (Post Purchase Invoice / Vendor Invoice)
================================================================================

// Satın alma faturası oluşturma ve kaydetme
// Kaynak: axtechsolutions.blogspot.com, d365ffo.com

public void postPurchaseInvoice(PurchId _purchId, InvoiceId _invoiceId)
{
    PurchTable          purchTable = PurchTable::find(_purchId);
    PurchFormLetter     purchFormLetter;

    ttsBegin;

    purchFormLetter = PurchFormLetter::construct(DocumentStatus::Invoice);
    purchFormLetter.update(purchTable,
                           _invoiceId,
                           systemDateGet());

    ttsCommit;

    info(strFmt("Satın alma faturası kaydedildi: PO=%1, Fatura=%2", _purchId, _invoiceId));
}


================================================================================
5. SATIŞ SİPARİŞİ OLUŞTURMA (Create Sales Order)
================================================================================

// Satış siparişi oluşturma
// Kaynak: community.dynamics.com, paragchapre.com, blog.peterdx.com

class CreateSalesOrder
{
    public static void main(Args _args)
    {
        SalesTable      salesTable;
        SalesLine       salesLine;
        InventDim       inventDim;
        NumberSeq       numberSeq;
        CustTable       custTable = CustTable::find("US-007"); // Müşteri hesabı

        try
        {
            ttsBegin;

            // Satış siparişi başlığı
            numberSeq = NumberSeq::newGetNum(SalesParameters::numRefSalesId());
            numberSeq.used();

            salesTable.SalesId = numberSeq.num();
            salesTable.initValue();
            salesTable.CustAccount = custTable.AccountNum;
            salesTable.initFromCustTable();

            if (!salesTable.validateWrite())
            {
                throw Exception::Error;
            }
            salesTable.insert();

            // Satış siparişi satırı
            salesLine.clear();
            salesLine.initFromSalesTable(salesTable);
            salesLine.ItemId = "D0001";     // Ürün numarası
            salesLine.SalesQty = 5;         // Miktar
            salesLine.SalesPrice = 150;     // Birim fiyat

            inventDim.clear();
            inventDim.InventSiteId = "1";
            inventDim.InventLocationId = "11";
            salesLine.InventDimId = InventDim::findOrCreate(inventDim).inventDimId;

            salesLine.createLine(true, true, true, true, true);

            ttsCommit;

            info(strFmt("Satış siparişi oluşturuldu: %1", salesTable.SalesId));
        }
        catch (Exception::Error)
        {
            error("Satış siparişi oluşturulurken hata oluştu.");
        }
    }
}


================================================================================
6. SATIŞ SİPARİŞİ ONAYLAMA (Sales Order Confirmation)
================================================================================

// Satış siparişi onaylama
// Kaynak: community.dynamics.com, shyamkannadasan.blogspot.com

public void confirmSalesOrder(SalesId _salesId)
{
    SalesTable          salesTable = SalesTable::find(_salesId);
    SalesFormLetter     salesFormLetter;

    salesFormLetter = SalesFormLetter::construct(DocumentStatus::Confirmation);
    salesFormLetter.update(salesTable);

    info(strFmt("Satış siparişi onaylandı: %1", _salesId));
}


================================================================================
7. SATIŞ SİPARİŞİ SEVKİYAT FİŞİ (Sales Packing Slip)
================================================================================

// Satış siparişi sevkiyat fişi (Packing Slip) kaydetme
// Kaynak: rahulmsdax.blogspot.com, sangeethwiki.blogspot.com

public void postSalesPackingSlip(SalesId _salesId)
{
    SalesTable                      salesTable = SalesTable::find(_salesId);
    SalesFormLetter_PackingSlip     salesFormLetter_PackingSlip;
    CustPackingSlipJour             custPackingSlipJour;
    TransDate                       packingSlipDate = systemDateGet();

    ttsBegin;

    if (salesTable && salesTable.SalesStatus == SalesStatus::Backorder)
    {
        salesFormLetter_PackingSlip = SalesFormLetter::construct(DocumentStatus::PackingSlip);
        salesFormLetter_PackingSlip.update(salesTable,
                                           packingSlipDate,
                                           SalesUpdate::All,
                                           AccountOrder::None,
                                           false,  // Proforma
                                           false);  // Yazdırma

        if (salesFormLetter_PackingSlip.parmJournalRecord().TableId == tableNum(CustPackingSlipJour))
        {
            custPackingSlipJour = salesFormLetter_PackingSlip.parmJournalRecord();
            info(strFmt("Sevkiyat fişi kaydedildi: %1", custPackingSlipJour.PackingSlipId));
        }
    }

    ttsCommit;
}


================================================================================
8. SATIŞ SİPARİŞİ FATURASI KAYDETME (Post Sales Invoice)
================================================================================

// Satış faturası kaydetme
// Kaynak: rahulmsdax.blogspot.com, chaituax.wordpress.com

public void postSalesInvoice(SalesId _salesId)
{
    SalesTable          salesTable = SalesTable::find(_salesId);
    SalesFormLetter     salesFormLetter;

    ttsBegin;

    salesFormLetter = SalesFormLetter::construct(DocumentStatus::Invoice);
    salesFormLetter.update(salesTable,
                           systemDateGet(),
                           SalesUpdate::All,
                           AccountOrder::None,
                           NoYes::No,    // Proforma
                           NoYes::No);   // Yazdırma

    ttsCommit;

    info(strFmt("Satış faturası kaydedildi: %1", _salesId));
}


================================================================================
9. ÜRETİM EMRİ OLUŞTURMA (Create Production Order)
================================================================================

// Üretim emri oluşturma
// Kaynak: community.dynamics.com (DAX Beginners), bmdax.blogspot.com

class CreateProductionOrder
{
    public static void main(Args _args)
    {
        ProdQty         qty = 100;
        ItemId          item = "D0005";
        ProdTable       prodTable;
        InventTable     inventTable;
        InventDim       inventDim;

        inventTable = InventTable::find(item);

        // Temel değerleri başlat
        prodTable.initValue();
        prodTable.initFromInventTable(inventTable);
        prodTable.ItemId = inventTable.ItemId;
        prodTable.DlvDate = today();
        prodTable.QtySched = qty;
        prodTable.RemainInventPhysical = qty;

        // InventDim başlat
        inventDim.initValue();

        // Aktif BOM ve Rota ayarla
        prodTable.BOMId = BOMVersion::findActive(
            prodTable.ItemId,
            prodTable.BOMDate,
            prodTable.QtySched,
            inventDim).BOMId;

        prodTable.RouteId = RouteVersion::findActive(
            prodTable.ItemId,
            prodTable.BOMDate,
            prodTable.QtySched,
            inventDim).RouteId;

        // BOM ve Route versiyonlarını başlat
        prodTable.initBOMVersion();
        prodTable.initRouteVersion();

        // ProdTableType sınıfı ile üretim emri oluştur
        prodTable.type().insert();

        info(strFmt("Üretim emri oluşturuldu: %1", prodTable.ProdId));
    }
}


================================================================================
10. ENVANTER HAREKET GÜNLÜĞÜ OLUŞTURMA (Inventory Movement Journal)
================================================================================

// Stok hareket günlüğü oluşturma ve kaydetme
// Kaynak: learnax.blogspot.com, d365opstechtalks.com, community.dynamics.com

class CreateMovementJournal
{
    public static void main(Args _args)
    {
        InventJournalTable      inventJournalTable;
        InventJournalTrans      inventJournalTrans;
        InventJournalNameId     inventJournalName;
        InventDim               inventDim;
        JournalCheckPost        journalCheckPost;

        ttsBegin;

        // Günlük Başlığı Oluşturma
        inventJournalTable.clear();
        inventJournalName = InventJournalName::standardJournalName(InventJournalType::Movement);
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(inventJournalName));
        inventJournalTable.insert();

        // Günlük Satırı Oluşturma
        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.TransDate = systemDateGet();
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.initFromInventTable(InventTable::find("D0001"));
        inventJournalTrans.Qty = 10;     // Pozitif = giriş, Negatif = çıkış

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        inventJournalTrans.InventDimId = InventDim::findOrCreate(inventDim).inventDimId;
        inventJournalTrans.insert();

        ttsCommit;

        // Günlüğü Kaydet (Post)
        journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        journalCheckPost.parmThrowCheckFailed(false);
        journalCheckPost.parmTransferErrors(NoYes::No);
        journalCheckPost.run();

        info(strFmt("Hareket günlüğü oluşturuldu ve kaydedildi: %1",
            inventJournalTable.JournalId));
    }
}


================================================================================
11. ENVANTER TRANSFER GÜNLÜĞÜ OLUŞTURMA (Inventory Transfer Journal)
================================================================================

// Stok transfer günlüğü - ambarlar arası transfer
// Kaynak: community.dynamics.com, linkedin.com (Usama Mehmood)

class CreateTransferJournal
{
    public static void main(Args _args)
    {
        InventJournalTable          inventJournalTable;
        InventJournalTrans          inventJournalTrans;
        InventJournalCheckPost      inventJournalCheckPost;
        InventDim                   fromInventDim, toInventDim;

        ttsBegin;

        // Günlük Başlığı
        inventJournalTable.clear();
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(
                InventParameters::find().TransferJournalNameId));
        inventJournalTable.Description = "Stok Transfer Günlüğü";
        inventJournalTable.insert();

        // Günlük Satırı
        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.Qty = 5;

        // Kaynak (From) Boyut
        fromInventDim.clear();
        fromInventDim.InventSiteId = "1";
        fromInventDim.InventLocationId = "11";  // Kaynak ambar
        inventJournalTrans.InventDimId =
            InventDim::findOrCreate(fromInventDim).inventDimId;

        // Hedef (To) Boyut
        toInventDim.clear();
        toInventDim.InventSiteId = "1";
        toInventDim.InventLocationId = "12";    // Hedef ambar
        inventJournalTrans.ToInventDimId =
            InventDim::findOrCreate(toInventDim).inventDimId;

        inventJournalTrans.insert();

        ttsCommit;

        // Günlüğü Kaydet (Post)
        inventJournalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        inventJournalCheckPost.parmThrowCheckFailed(false);
        inventJournalCheckPost.parmTransferErrors(NoYes::No);
        inventJournalCheckPost.run();

        info(strFmt("Transfer günlüğü oluşturuldu: %1", inventJournalTable.JournalId));
    }
}


================================================================================
12. TRANSFER EMRİ OLUŞTURMA (Create Transfer Order)
================================================================================

// Ambarlar arası transfer emri oluşturma (Transit stok takibi ile)
// Kaynak: d365ffo.com, dynamics2012to365.blogspot.com

class CreateTransferOrder
{
    public static void main(Args _args)
    {
        InventTransferTable     inventTransferTable;
        InventTransferLine      inventTransferLine;
        NumberSeq               numberSeq;
        InventDim               inventDim;

        ttsBegin;

        // Transfer Emri Başlığı
        numberSeq = NumberSeq::newGetNum(InventParameters::numRefTransferId());

        inventTransferTable.clear();
        inventTransferTable.initValue();
        inventTransferTable.TransferId = numberSeq.num();
        numberSeq.used();

        inventTransferTable.InventLocationIdFrom = "11";   // Kaynak ambar
        inventTransferTable.modifiedField(
            fieldNum(InventTransferTable, InventLocationIdFrom));

        inventTransferTable.InventLocationIdTo = "12";     // Hedef ambar
        inventTransferTable.modifiedField(
            fieldNum(InventTransferTable, InventLocationIdTo));

        inventTransferTable.TransferStatus = InventTransferStatus::Created;
        inventTransferTable.insert();

        // Transfer Emri Satırı
        inventTransferLine.clear();
        inventTransferLine.initFromInventTransferTable(inventTransferTable, NoYes::Yes);
        inventTransferLine.ItemId = "D0001";
        inventTransferLine.initFromInventTable(InventTable::find("D0001"));
        inventTransferLine.LineNum =
            InventTransferLine::lastLineNum(inventTransferTable.TransferId) + 1;
        inventTransferLine.QtyTransfer = 5;
        inventTransferLine.QtyRemainShip = 5;
        inventTransferLine.QtyRemainReceive = 5;
        inventTransferLine.QtyShipNow = 0;
        inventTransferLine.QtyReceiveNow = 0;

        inventDim.clear();
        inventDim.InventSiteId =
            InventLocation::find(inventTransferTable.InventLocationIdFrom).InventSiteId;
        inventDim.InventLocationId = inventTransferTable.InventLocationIdFrom;
        inventTransferLine.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;

        inventTransferLine.insert();

        ttsCommit;

        info(strFmt("Transfer emri oluşturuldu: %1", inventTransferTable.TransferId));
    }
}


================================================================================
13. GENEL MUHASEBE GÜNLÜĞÜ OLUŞTURMA (Create General Journal)
================================================================================

// Genel günlük (General Journal) oluşturma ve kaydetme
// Kaynak: community.dynamics.com, denistrunin.com

class CreateGeneralJournal
{
    public static void main(Args _args)
    {
        LedgerJournalTable          ledgerJournalTable;
        LedgerJournalTrans          ledgerJournalTrans;
        LedgerJournalCheckPost      journalCheckPost;
        LedgerJournalName           ledgerJournalName;
        NumberSeq                   numberSeq;
        Voucher                     voucher;

        // Günlük adını bul
        select firstonly ledgerJournalName
            where ledgerJournalName.JournalName == "GenJrn";  // Günlük adı

        ttsBegin;

        // Günlük Başlığı
        ledgerJournalTable.JournalName = ledgerJournalName.JournalName;
        ledgerJournalTable.initFromLedgerJournalName();
        ledgerJournalTable.JournalNum =
            JournalTableData::newTable(ledgerJournalTable).nextJournalId();
        ledgerJournalTable.Name = "X++ ile oluşturulan günlük";
        ledgerJournalTable.insert();

        // Fiş numarası al
        numberSeq = NumberSeq::newGetVoucherFromCode(
            NumberSequenceTable::find(
                ledgerJournalName.NumberSequenceTable).NumberSequence);
        voucher = numberSeq.voucher();

        // Günlük Satırı - Borç
        ledgerJournalTrans.clear();
        ledgerJournalTrans.JournalNum = ledgerJournalTable.JournalNum;
        ledgerJournalTrans.TransDate = today();
        ledgerJournalTrans.AccountType = LedgerJournalACType::Ledger;
        ledgerJournalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "110110",   // Ana hesap numarası
                LedgerJournalACType::Ledger);
        ledgerJournalTrans.AmountCurDebit = 1000;
        ledgerJournalTrans.CurrencyCode = "USD";
        ledgerJournalTrans.Txt = "Test borç hareketi";
        ledgerJournalTrans.Voucher = voucher;
        ledgerJournalTrans.Approved = NoYes::Yes;
        ledgerJournalTrans.insert();

        // Günlük Satırı - Alacak
        ledgerJournalTrans.clear();
        ledgerJournalTrans.JournalNum = ledgerJournalTable.JournalNum;
        ledgerJournalTrans.TransDate = today();
        ledgerJournalTrans.AccountType = LedgerJournalACType::Ledger;
        ledgerJournalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "170150",   // Ana hesap numarası
                LedgerJournalACType::Ledger);
        ledgerJournalTrans.AmountCurCredit = 1000;
        ledgerJournalTrans.CurrencyCode = "USD";
        ledgerJournalTrans.Txt = "Test alacak hareketi";
        ledgerJournalTrans.Voucher = voucher;
        ledgerJournalTrans.Approved = NoYes::Yes;
        ledgerJournalTrans.insert();

        ttsCommit;

        // Günlüğü Kaydet (Post)
        journalCheckPost = LedgerJournalCheckPost::newLedgerJournalTable(
            ledgerJournalTable, NoYes::Yes);
        journalCheckPost.runOperation();

        info(strFmt("Genel günlük oluşturuldu ve kaydedildi: %1",
            ledgerJournalTable.JournalNum));
    }
}


================================================================================
14. SATICI FATURA GÜNLÜĞÜ OLUŞTURMA (Vendor Invoice Journal)
================================================================================

// Satıcı fatura günlüğü oluşturma
// Kaynak: d365opstechtalks.com, community.dynamics.com

class CreateVendorInvoiceJournal
{
    public static void main(Args _args)
    {
        LedgerJournalTable      journalTable;
        LedgerJournalTrans      journalTrans;
        JournalTableData        journalTableData;
        NumberSeq               numberSeq;
        Voucher                 voucher;
        LedgerJournalName       ledgerJournalName;

        ttsBegin;

        // Günlük adını bul (satıcı fatura günlüğü)
        select firstonly ledgerJournalName
            where ledgerJournalName.JournalName == "APInv";  // AP Invoice Journal

        // Başlık oluştur
        journalTable.initValue();
        journalTable.JournalName = ledgerJournalName.JournalName;
        journalTable.initFromLedgerJournalName();
        journalTable.JournalNum =
            JournalTableData::newTable(journalTable).nextJournalId();
        journalTable.Name = "Satıcı Fatura Günlüğü";
        journalTable.insert();

        // Satır oluştur
        journalTrans.clear();
        journalTrans.initValue();
        journalTrans.JournalNum = journalTable.JournalNum;
        journalTrans.TransDate = today();

        // Satıcı hesabı
        journalTrans.AccountType = LedgerJournalACType::Vend;
        journalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "US-101",   // Satıcı hesap numarası
                LedgerJournalACType::Vend);

        journalTrans.AmountCurCredit = 5000;
        journalTrans.CurrencyCode = "USD";

        // Mahsup hesap (Offset)
        journalTrans.OffsetAccountType = LedgerJournalACType::Ledger;
        journalTrans.OffsetLedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "600120",   // Gider hesabı
                LedgerJournalACType::Ledger);

        journalTrans.Txt = "Satıcı faturası";
        journalTrans.Invoice = "INV-001";
        journalTrans.Approved = NoYes::Yes;
        journalTrans.insert();

        ttsCommit;

        info(strFmt("Satıcı fatura günlüğü oluşturuldu: %1", journalTable.JournalNum));
    }
}


================================================================================
15. SATICI ÖDEME GÜNLÜĞÜ OLUŞTURMA (Vendor Payment Journal)
================================================================================

// Satıcı ödeme günlüğü oluşturma
// Kaynak: linkedin.com (Usama Mehmood), axvigneshvaran.wordpress.com

class CreateVendorPaymentJournal
{
    public static void main(Args _args)
    {
        LedgerJournalTable      journalTable;
        LedgerJournalTrans      journalTrans;

        ttsBegin;

        // Başlık
        journalTable.initValue();
        journalTable.JournalName = "VendPay";  // Satıcı ödeme günlük adı
        journalTable.initFromLedgerJournalName();
        journalTable.JournalNum =
            JournalTableData::newTable(journalTable).nextJournalId();
        journalTable.Name = "Satıcı Ödeme Günlüğü";
        journalTable.insert();

        // Satır
        journalTrans.clear();
        journalTrans.initValue();
        journalTrans.JournalNum = journalTable.JournalNum;
        journalTrans.TransDate = today();

        // Satıcı hesabı
        journalTrans.AccountType = LedgerJournalACType::Vend;
        journalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "US-101",
                LedgerJournalACType::Vend);

        journalTrans.AmountCurDebit = 5000;
        journalTrans.CurrencyCode = "USD";

        // Mahsup hesap - Banka
        journalTrans.OffsetAccountType = LedgerJournalACType::Bank;
        journalTrans.OffsetLedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "USMF OPER",    // Banka hesabı
                LedgerJournalACType::Bank);

        journalTrans.Txt = "Satıcı ödemesi";
        journalTrans.Approved = NoYes::Yes;
        journalTrans.insert();

        ttsCommit;

        info(strFmt("Satıcı ödeme günlüğü oluşturuldu: %1", journalTable.JournalNum));
    }
}


================================================================================
16. MÜŞTERİ ÖDEME GÜNLÜĞÜ OLUŞTURMA (Customer Payment Journal)
================================================================================

// Müşteri ödeme günlüğü oluşturma
// Kaynak: dynamicsaxforall.blogspot.com, axvigneshvaran.wordpress.com

class CreateCustomerPaymentJournal
{
    public static void main(Args _args)
    {
        LedgerJournalTable              journalTable;
        LedgerJournalTrans              journalTrans;
        LedgerJournalEngine_CustPayment ledgerJournalEngine;

        ledgerJournalEngine = new LedgerJournalEngine_CustPayment();

        ttsBegin;

        // Başlık
        journalTable.initValue();
        journalTable.JournalNum =
            JournalTableData::newTable(journalTable).nextJournalId();
        journalTable.JournalName = "CustPay";  // Müşteri ödeme günlük adı
        journalTable.initFromLedgerJournalName();
        journalTable.Name = "Müşteri Ödeme Günlüğü";
        journalTable.insert();

        // Satır
        journalTrans.clear();
        journalTrans.initValue();
        ledgerJournalEngine.newJournalActive(journalTable);
        ledgerJournalEngine.initValue(journalTrans);

        journalTrans.JournalNum = journalTable.JournalNum;
        journalTrans.TransDate = today();
        journalTrans.AccountType = LedgerJournalACType::Cust;
        journalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "US-007",   // Müşteri hesap numarası
                LedgerJournalACType::Cust);

        journalTrans.AmountCurCredit = 3000;
        journalTrans.CurrencyCode = "USD";

        // Mahsup hesap - Banka
        journalTrans.OffsetAccountType = LedgerJournalACType::Bank;
        journalTrans.OffsetLedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "USMF OPER",
                LedgerJournalACType::Bank);

        journalTrans.Txt = "Müşteri ödemesi";
        journalTrans.Approved = NoYes::Yes;
        journalTrans.insert();

        ttsCommit;

        info(strFmt("Müşteri ödeme günlüğü oluşturuldu: %1", journalTable.JournalNum));
    }
}


================================================================================
17. SERBEST METİN FATURASI OLUŞTURMA (Create Free Text Invoice)
================================================================================

// Serbest metin faturası oluşturma
// Kaynak: axvigneshvaran.wordpress.com, dynamicsaxforall.blogspot.com

class CreateFreeTextInvoice
{
    public static void main(Args _args)
    {
        CustInvoiceTable    custInvoiceTable;
        CustInvoiceLine     custInvoiceLine;
        CustTable           custTable = CustTable::find("US-007");

        ttsBegin;

        // Fatura Başlığı
        custInvoiceTable.initFromCustTable(custTable);
        custInvoiceTable.InvoiceDate =
            DateTimeUtil::getSystemDate(
                DateTimeUtil::getUserPreferredTimeZone());
        custInvoiceTable.insert();

        // Fatura Satırı
        custInvoiceLine.initValue();
        custInvoiceLine.initFromCustInvoiceTable(custInvoiceTable);
        custInvoiceLine.Description = "Danışmanlık hizmeti";
        custInvoiceLine.AmountCur = 2500;
        custInvoiceLine.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "401200",
                LedgerJournalACType::Ledger);
        custInvoiceLine.insert();

        ttsCommit;

        info(strFmt("Serbest metin faturası oluşturuldu: %1",
            custInvoiceTable.InvoiceId));
    }
}


================================================================================
18. ENVANTER SAYIM GÜNLÜĞÜ OLUŞTURMA (Inventory Counting Journal)
================================================================================

// Stok sayım günlüğü oluşturma
// Kaynak: msdynamicshelper.blogspot.com, allaboutdynamic.com

class CreateCountingJournal
{
    public static void main(Args _args)
    {
        InventJournalTable      inventJournalTable;
        InventJournalTrans      inventJournalTrans;
        InventJournalNameId     inventJournalName;
        InventDim               inventDim;
        JournalCheckPost        journalCheckPost;

        ttsBegin;

        // Sayım Günlüğü Başlığı
        inventJournalTable.clear();
        inventJournalName =
            InventJournalName::standardJournalName(InventJournalType::Count);
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(inventJournalName));
        inventJournalTable.Description = "Stok Sayım Günlüğü";
        inventJournalTable.insert();

        // Sayım Günlüğü Satırı
        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.TransDate = systemDateGet();
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.initFromInventTable(InventTable::find("D0001"));
        inventJournalTrans.Counted = 50;     // Sayılan miktar
        inventJournalTrans.Qty = 0;          // Sistem farkı hesaplayacak

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        inventJournalTrans.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;
        inventJournalTrans.insert();

        ttsCommit;

        // Kaydet
        journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        journalCheckPost.parmThrowCheckFailed(false);
        journalCheckPost.parmTransferErrors(NoYes::No);
        journalCheckPost.run();

        info(strFmt("Sayım günlüğü oluşturuldu: %1", inventJournalTable.JournalId));
    }
}


================================================================================
19. BOM (ÜRÜN AĞACI) GÜNLÜĞÜ OLUŞTURMA (BOM Journal)
================================================================================

// BOM günlüğü oluşturma
// Kaynak: sangeethwiki.blogspot.com

class CreateBOMJournal
{
    public static void main(Args _args)
    {
        InventJournalTable      journalTable;
        InventJournalTrans      journalTrans;
        InventJournalTableData  journalTableData;
        InventJournalTransData  journalTransData;
        InventDim               inventDim;
        JournalCheckPost        journalCheckPost;

        journalTableData = JournalTableData::newTable(journalTable);
        journalTransData = journalTableData.journalStatic()
            .newJournalTransData(journalTrans, journalTableData);

        ttsBegin;

        // BOM Günlük Başlığı
        journalTable.clear();
        journalTable.JournalId = journalTableData.nextJournalId();
        journalTable.JournalNameId = InventParameters::find().BOMJournalNameId;
        journalTable.JournalType = InventJournalType::BOM;
        journalTableData.initFromJournalName(
            journalTableData.journalStatic().findJournalName(
                journalTable.JournalNameId));
        journalTable.insert();

        // BOM Günlük Satırı
        journalTrans.clear();
        journalTransData.initFromJournalTable();
        journalTrans.TransDate = systemDateGet();
        journalTrans.ItemId = "D0001";
        journalTrans.BOMLine = NoYes::Yes;
        journalTrans.Qty = 10;

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        journalTrans.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;

        journalTransData.create();

        ttsCommit;

        // Kaydet
        journalCheckPost = InventJournalCheckPost::newPostJournal(journalTable);
        journalCheckPost.run();

        info(strFmt("BOM günlüğü oluşturuldu: %1", journalTable.JournalId));
    }
}


================================================================================
20. ENVANTER GÜNLÜĞÜ KAYDETME - GENEL (Post Any Inventory Journal)
================================================================================

// Tüm envanter günlüklerini kaydetme (Movement, Transfer, Counting, BOM)
// Kaynak: allaboutdynamic.com

public void postInventoryJournal(InventJournalId _journalId)
{
    JournalCheckPost    journalCheckPost;
    InventJournalTable  inventJournalTable;

    inventJournalTable = InventJournalTable::find(_journalId);

    journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
    journalCheckPost.parmThrowCheckFailed(false);
    journalCheckPost.parmTransferErrors(NoYes::No);
    journalCheckPost.run();

    info(strFmt("Envanter günlüğü kaydedildi: %1", _journalId));
}


================================================================================
21. MUHASEBE GÜNLÜĞÜ KAYDETME (Post Ledger Journal)
================================================================================

// Genel muhasebe günlüğünü kaydetme
// Kaynak: cloudfronts.com

public void postLedgerJournal(LedgerJournalTable _ledgerJournalTable)
{
    LedgerJournalCheckPost jourPost;

    jourPost = LedgerJournalCheckPost::newLedgerJournalTable(
        _ledgerJournalTable, NoYes::Yes);
    jourPost.runOperation();

    info(strFmt("Muhasebe günlüğü kaydedildi: %1",
        _ledgerJournalTable.JournalNum));
}


================================================================================
22. SATIN ALMA TALEBİ OLUŞTURMA (Create Purchase Requisition)
================================================================================

// Satın alma talebi oluşturma (temel yapı)

class CreatePurchRequisition
{
    public static void main(Args _args)
    {
        PurchReqTable       purchReqTable;
        PurchReqLine        purchReqLine;
        InventDim           inventDim;

        ttsBegin;

        // Başlık
        purchReqTable.initValue();
        purchReqTable.PurchReqName = "Test Satın Alma Talebi";
        purchReqTable.RequisitionStatus = PurchReqRequisitionStatus::Draft;
        purchReqTable.insert();

        // Satır
        purchReqLine.clear();
        purchReqLine.initValue();
        purchReqLine.PurchReqTable = purchReqTable.RecId;
        purchReqLine.ItemId = "D0001";
        purchReqLine.PurchQty = 20;
        purchReqLine.CurrencyCode = "USD";

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        purchReqLine.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;

        purchReqLine.insert();

        ttsCommit;

        info(strFmt("Satın alma talebi oluşturuldu: %1", purchReqTable.PurchReqId));
    }
}


================================================================================
23. SATIŞ TEKLİFİ OLUŞTURMA (Create Sales Quotation)
================================================================================

// Satış teklifi oluşturma (temel yapı)

class CreateSalesQuotation
{
    public static void main(Args _args)
    {
        SalesQuotationTable     quotationTable;
        SalesQuotationLine      quotationLine;
        InventDim               inventDim;
        NumberSeq               numberSeq;

        ttsBegin;

        numberSeq = NumberSeq::newGetNum(
            SalesParameters::numRefQuotationId());
        numberSeq.used();

        // Teklif Başlığı
        quotationTable.QuotationId = numberSeq.num();
        quotationTable.initValue();
        quotationTable.CustAccount = "US-007";
        quotationTable.initFromCustTable();
        quotationTable.insert();

        // Teklif Satırı
        quotationLine.clear();
        quotationLine.initFromSalesQuotationTable(quotationTable);
        quotationLine.ItemId = "D0001";
        quotationLine.SalesQty = 10;
        quotationLine.SalesPrice = 200;

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        quotationLine.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;

        quotationLine.createLine(true, true, true, true, true);

        ttsCommit;

        info(strFmt("Satış teklifi oluşturuldu: %1", quotationTable.QuotationId));
    }
}


================================================================================
24. ENVANTER DÜZELTME GÜNLÜĞÜ (Inventory Adjustment Journal)
================================================================================

// Stok düzeltme günlüğü - Movement Journal ile benzer yapı
// Kaynak: allaboutdynamic.com

class CreateAdjustmentJournal
{
    public static void main(Args _args)
    {
        InventJournalTable      inventJournalTable;
        InventJournalTrans      inventJournalTrans;
        InventJournalNameId     inventJournalName;
        InventDim               inventDim;

        ttsBegin;

        inventJournalTable.clear();
        inventJournalName =
            InventJournalName::standardJournalName(
                InventJournalType::LossProfit);  // Düzeltme günlüğü tipi
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(inventJournalName));
        inventJournalTable.Description = "Stok Düzeltme Günlüğü";
        inventJournalTable.insert();

        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.TransDate = systemDateGet();
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.initFromInventTable(InventTable::find("D0001"));
        inventJournalTrans.Qty = 5;  // Pozitif: artış, Negatif: azalış

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        inventJournalTrans.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;
        inventJournalTrans.insert();

        ttsCommit;

        // Kaydet
        InventJournalCheckPost journalCheckPost;
        journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        journalCheckPost.run();

        info(strFmt("Düzeltme günlüğü oluşturuldu: %1",
            inventJournalTable.JournalId));
    }
}


================================================================================
25. BATCH JOB OLUŞTURMA (SysOperation Framework)
================================================================================

// Toplu iş oluşturma - SysOperation Framework
// Kaynak: medium.com, dynamics365musings.com

// --- Controller Sınıfı ---
class MyBatchController extends SysOperationServiceController
{
    public void new()
    {
        super();
        this.parmClassName(classStr(MyBatchService));
        this.parmMethodName(methodStr(MyBatchService, processRecords));
    }

    public static MyBatchController construct()
    {
        return new MyBatchController();
    }

    public ClassDescription defaultCaption()
    {
        return "Toplu İşlem Örneği";
    }

    public static void main(Args _args)
    {
        MyBatchController controller = MyBatchController::construct();
        controller.parmArgs(_args);
        controller.startOperation();
    }
}

// --- Service Sınıfı ---
class MyBatchService extends SysOperationServiceBase
{
    public void processRecords()
    {
        // İş mantığınızı buraya yazın
        info("Toplu iş başarıyla çalıştı.");
    }
}


================================================================================
                          KAYNAKLAR VE REFERANSLAR
================================================================================

Bu dokümandaki kod örnekleri aşağıdaki kaynaklardan derlenmiştir:

- Microsoft Dynamics 365 Community (community.dynamics.com)
- Parag Chapre Blog (paragchapre.com)
- Peter DX Blog (blog.peterdx.com)
- Nuxulu.com (nuxulu.com)
- D365FFO Blog (d365ffo.com)
- Denis Trunin Blog (denistrunin.com)
- CloudFronts Blog (cloudfronts.com)
- D365 Ops Tech Talks (d365opstechtalks.com)
- All About Dynamics (allaboutdynamic.com)
- Learn AX Blog (learnax.blogspot.com)
- AX Tech Solutions (axtechsolutions.blogspot.com)
- Dynamics AX For All (dynamicsaxforall.blogspot.com)
- Rahul MSDAX Blog (rahulmsdax.blogspot.com)
- Sangeeth Wiki (sangeethwiki.blogspot.com)
- Shyam Kannadasan Blog (shyamkannadasan.blogspot.com)
- LinkedIn (Usama Mehmood, Samuel Udoye)
- Medium (Muhammad Ramzan)

# Standart Tablo Alan Referans Dokumanlari

Bu bolum, standart D365 F&O system metadata'sindan uretilmis kapsamli tablo-alan referans dokumanlarini listeler.
Her dokuman ilgili tablo ailesinin **tum alanlarini** tip, EDT/Enum, label ve aciklama bilgileriyle icerir.

- Kaynak surum: `10.0.2345.153 PackagesLocalDirectory`
- Uretim tarihi: 2026-04-03

## Dokuman Listesi

| Dokuman | Kapsam | Boyut | Yol |
|---|---|---|---|
| **Invent Tablolari Alan Referansi** | 569 tablo (Core 30 + Other 230 + History 13 + Tmp 101 + Localization 49 + Staging 146) | ~1.1 MB, 14800 satir | [Invent_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Invent_Tablolari_Alan_Referansi_20260403.md) |
| **Purch Tablolari Alan Referansi** | 368 tablo (Core 23 + Other 113 + History 28 + Tmp 54 + Localization 28 + Staging 122) | ~933 KB, 11000 satir | [Purch_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Purch_Tablolari_Alan_Referansi_20260403.md) |
| **Sales Tablolari Alan Referansi** | 267 tablo (Core 15 + Other 49 + History 13 + Tmp 33 + Localization 31 + Staging 126) | ~1 MB, 11100 satir | [Sales_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Sales_Tablolari_Alan_Referansi_20260403.md) |
| **Purch-Invent Cekirdek Operasyon Rehberi** | Purch ve Invent cekirdek operasyon tablolari, iliski zincirleri ve senaryo bazli rehber | ~4 KB, 115 satir | [Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md](docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md) |

## Ne Zaman Hangi Dokumana Bakilir?

| Senaryo | Bakilacak Dokuman |
|---|---|
| Bir tablonun hangi alanlari var, tipleri ne? | Ilgili alan referans dokumani (Invent/Purch/Sales) |
| AxForm/View/Entity tasarlarken alan adi dogrulama | Ilgili alan referans dokumani → Core kategorisi |
| Tablo iliskisi ve operasyon akisi anlamak | Purch-Invent Cekirdek Operasyon Rehberi |
| Extension'a yeni alan eklerken mevcut alanlari kontrol | Ilgili alan referans dokumani |
| CoC metod parametreleri dogrulama | Ilgili alan referans dokumani → ilgili tablo bolumu |
| DataField esleme sorunlari (bkz: `01-metadata-xml-rules.md` Bolum 6b) | Ilgili alan referans dokumani |

## Dokuman Yapisi

Her alan referans dokumani su yapidadir:
- **Kategori Ozeti:** Core, Other, History, Tmp, Localization, Staging dagilimi
- **Her tablo icin:**
  - Kategori, kaynak metadata dosyasi, tablo amaci, alan sayisi, title field'lar
  - Tum alanlar: Alan adi, XML tipi, EDT/Enum, Label, aciklama

## Purch-Invent Karar Ozeti

| Soru | Purch tarafi | Invent tarafi |
|---|---|---|
| Baslik kaydi nerede? | `PurchTable` | `InventJournalTable` / `InventTransferTable` / senaryoya gore belge basligi |
| Satir detayi nerede? | `PurchLine` | `InventJournalTrans` / `InventTransferLine` / `InventTrans` |
| Anlik operasyon gercegi nerede? | `PurchLine` + `PurchParm*` | `InventTrans` + `InventSum` |
| Boyut / batch kirilimi nerede? | `PurchLine` icindeki `InventDimId` baglantisinda | `InventDim` ve `InventBatch` |
| Posting izi nerede? | `PurchParmTable` / `PurchParmLine` | `InventJournal*`, `InventSettlement`, `InventCostTrans` |

================================================================================
                              ÖNEMLİ NOTLAR
================================================================================

1. Tüm kodlar D365 Finance & Operations ortamı için hazırlanmıştır.
2. Demo verileri USMF şirketi baz alınmıştır.
3. Kendi ortamınızda kullanırken hesap numaraları, ürün kodları,
   ambar ve site bilgilerini değiştirmeniz gerekmektedir.
4. ttsBegin/ttsCommit blokları veritabanı işlem bütünlüğünü sağlar.
5. Üretim ortamında kullanmadan önce test ortamında deneyiniz.
6. NumberSeq sınıfları numara serisi ayarlarına bağlıdır.
7. Bazı kodlar AX 2012 kaynaklı olup D365 FO'ya uyarlanmıştır.

================================================================================