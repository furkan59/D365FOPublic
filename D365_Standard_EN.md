# D365 Finance & Operations - Comprehensive Metadata & X++ Reference Guide

> Each section contains real XML examples and actual X++ code.
> Date: 2026-03-13

---

## Table of Contents

1. [AxClass - X++ Class Definitions](#axclass)
2. [AxTable - Table Definitions](#axtable)
3. [AxTableExtension - Table Extensions](#axtableextension)
4. [AxForm - Form Definitions](#axform)
5. [AxFormExtension - Form Extensions](#axformextension)
6. [AxEnum - Enumeration Definitions](#axenum)
7. [AxEnumExtension - Enum Extensions](#axenumextension)
8. [AxEdt - Extended Data Type](#axedt)
9. [AxEdtExtension - EDT Extensions](#axedtextension)
10. [AxView - SQL View Definitions](#axview)
11. [AxDataEntityView - Data Entity](#axdataentityview)
12. [AxDataEntityViewExtension - Entity Extensions](#axdataentityviewextension)
13. [AxQuery - Query Definitions](#axquery)
14. [AxQuerySimpleExtension - Query Extensions](#axquerysimpleextension)
15. [AxMenuItemAction / Display / Output](#axmenuitem)
16. [AxReport - SSRS Report Definitions](#axreport)
17. [AxSecurityPrivilege / Duty / Role](#security)
18. [AxMenuExtension - Menu Extensions](#axmenuextension)
19. [AxLabelFile - Label Files](#axlabelfile)
20. [Naming Rules and Conventions](#naming)
21. [Metadata Folder Structure](#folder)
22. [General D365 FO Development Patterns](#general-d365-fo-development-patterns)
23. [Standard Table Field Reference Documents](#standard-table-field-reference-documents)

---

# AxClass - X++ Class Definitions


---

## XML Structure - General Template

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxClass xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
	<Name>ClassName</Name>
	<SourceCode>
		<Declaration><![CDATA[
public class ClassName
{
    // member variables
}
]]></Declaration>
		<Methods>
			<Method>
				<Name>methodName</Name>
				<Source><![CDATA[
    public void methodName()
    {
        // body
    }

]]></Source>
			</Method>
		</Methods>
	</SourceCode>
</AxClass>
```

### Indentation Rules (CRITICAL)

- `<AxClass>`, `<Name>`, `<SourceCode>` -> 0 tab (root)
- `<Declaration>`, `<Methods>` -> 1 tab
- `<Method>` -> 2 tab
- `<Name>` and `<Source>` (inside Method) -> 3 tab
- X++ code inside CDATA: class declaration 0 indented, method body 4 spaces (or 1 tab) indented
- Each `<Source>` CDATA block ends before closing with a blank line: `\n]]>`
- `<Declaration>` CDATA block also leaves a blank line after the closing `}`

### Blank Line Rule at End of CDATA

**Declaration:**
```
<Declaration><![CDATA[
class ClassName
{
}
]]></Declaration>
```
No blank line, closes directly with `]]>`.

**Method Source:**
```
<Source><![CDATA[
    public void run()
    {
    }

]]></Source>
```
Method CDATA always has a blank line (`\n`) at the end, then `]]>`.

---

## Class Types and Examples

---

### 1. Normal Class (Business Logic Class)


**Learned Rules:**
- `class` keyword can be used without `public` (access modifier optional)
- parm methods can implement fluent interface with `return this;` (for setter chaining)
- `RunStatic` static factory/entry method pattern - external systems call this method
- TAB alignments can be used between member variables (visual alignment)
- Blank line after the last `}` inside Declaration CDATA: `}\n]]>`

---

### 2. Batch Class (RunBaseBatch) - Simple


**Learned Rules:**
- `class` keyword (no public modifier) - standard for RunBaseBatch
- `#define.CurrentVersion(1)` and `#define.CurrentList(version)` defined inside Declaration CDATA
- `pack()` container returns as `[#CurrentVersion, parmCompany]`
- `unpack()` simple case: directly `[version, parmCompany] = _packedClass;`
- `description()` static + `ClassDescription` return type + label reference
- `canRunInNewSession()` -> `protected boolean`
- `canGoBatchJournal()` -> `public boolean`
- `[SysEntryPointAttribute]` attribute at the start of `main()` method
- `prompt()` shows the dialog, `runOperation()` runs through the batch framework

---

### 3. Batch Class (RunBaseBatch) - Advanced with QueryRun


**Learned Rules:**
- `internal final class` modifier combination is possible
- Batch with QueryRun: `pack()` -> `[queryRun.pack()]`, `unpack()` -> `queryRun = new QueryRun(packedQuery)`
- Protected `new()` override: `protected void new()` + `super()` call
- `caption()` -> `public ClassDescription`, `description()` -> `public static ClassDescription` (both can be used)
- `runsImpersonated()` -> if it returns `true`, runs as service account
- `canGoBatch()` -> `public boolean` (for triggering from Batch Manager)
- Inside `run()`, `#OCCRetryCount` macro + `super()` + try/catch (Deadlock + UpdateConflict)
- `retry` keyword is used inside the Deadlock catch
- `handleUpdateConflict()` helper method for OCC (Optimistic Concurrency Control)
- `showQueryValues()` and `showQuerySelectButton()` -> show query in dialog

---

### 4. Batch Class - internal final + BatchRetryable + #LOCALMACRO


**Learned Rules:**
- `implements BatchRetryable` -> `isRetryable()` method is required
- `[Hookable(false)]` attribute + `public final boolean isRetryable()` -> CoC extension is prevented
- `#LOCALMACRO.CurrentList ... #ENDMACRO` -> for pack/unpack with multiple parm variables
- `pack()` -> in `[#CurrentVersion, #CurrentList]` format
- `unpack()` -> version check with `switch (version)`, `default: return false`
- `Main` (capital M) is also valid - not case-insensitive but capital M is used in the observed example
- Table variable in Declaration: `smmParametersTable smmParametersTable;` (type name == variable name is valid)

---

### 5. Controller Class (SrsReportRunController)


**Learned Rules:**
- `extends SrsReportRunController` can be placed on a separate line (if long)
- `implements BatchRetryable` as a continuation of extends
- `{}` empty body in Declaration CDATA -> just `{` at end of line + blank line + `}`
- `showDialog()` -> `protected boolean` -> `return false` to not show the dialog
- `preRunModifyContract()` -> `protected void` -> to fill the contract before the controller runs
- `this.parmArgs().caller() is FormRun` -> caller type check
- `formDataSourceStr(FormName, DataSourceName)` -> form datasource string intrinsic
- `this.parmReportContract().parmRdpContract()` -> access to contract object
- `ssrsReportStr(ReportName, DesignName)` -> report + design name (two parameters)
- `startOperation()` -> in main() use `startOperation()` not `runOperation()` (for SrsReportRunController)
- `controller.parmReportName(...)` no line break, continues on the next line

---

### 6. Contract Class (DataContract)


**Learned Rules:**
- `[DataContract]` class attribute -> first line of Declaration CDATA
- Member variable type can use EDT: `LedgerJournalId journalNum`
- Multiple attributes on parm method: `[DataMember,\n     SysOperationDisplayOrder('1'),\n     SysOperationControlVisibility(false)]`
- `SysOperationControlVisibility(false)` -> hides this field in the dialog
- `SysOperationDisplayOrder('1')` -> order number in dialog (as string)
- parm method return type is the same as the member variable type

---

### 7. Contract Class - DataContractAttribute (DTO Pattern)


**Learned Rules:**
- Both `[DataContractAttribute]` (Attribute suffix version) and `[DataContract]` are valid
- `[DataMemberAttribute('SerializationKey')]` -> key name given for JSON/OData serialization
- `private str` -> member variables can be marked with `private` keyword
- DTO (Data Transfer Object) pattern: no public before class keyword
- All parm methods follow the same pattern: `public str parmXxx(str _xxx = xxx)`
- We don't have to use EDT for primitive `str` types

---

### 8. DP Class (SrsReportDataProviderPreProcessTempDB)


**Learned Rules:**
- Attributes come before the class keyword (each on a separate line)
- `[SRSReportQueryAttribute(queryStr(...))]` -> query binding
- `[SRSReportParameterAttribute(classStr(...))]` -> contract class binding
- `extends` can be placed on a separate line (for long names)
- `private` member variables: `private LedgerJournalTable tmpLedgerJournalTable;`
- Dataset method: marked with `[SRSReportDataSetAttribute(tableStr(...))]` attribute
- Dataset method as `select tmpTable; return tmpTable;`
- `processReport()` -> contract is retrieved with `this.parmDataContract() as ContractClass`
- `delete_from paymentTmp;` -> clear before each report run
- `paymentTmp.clear()` -> clear buffer, then fill fields, then `paymentTmp.insert()`
- Comment lines (`// ----`) can be used to separate method groups
- `select firstOnly` (with capital O) is also valid (e.g., `firstOnly` vs `firstonly1`)

---

### 9. Extension Class - Table CoC (Chain of Command)


**Learned Rules:**
- `[ExtensionOf(tableStr(SalesTable))]` -> `tableStr()` for table extension
- `final class ClassName_Model_Extension` -> no `public`, `final` is required
- Empty extension class body: `{}`  or `{\n}` (also possible without blank line)
- DataEventHandler can be defined inside the same extension class
- DataEventHandler must be `public static void`
- `sender as SalesTable` -> type conversion
- `ttsBegin`/`ttsCommit` (capital B and C) also valid (lowercase too: `ttsbegin`/`ttscommit`)
- `doUpdate()` -> update without triggering events
- modifiedField: `next modifiedField(_fieldId)` is called FIRST (standard first, custom after)
- `insert(boolean _skipMarkup)` -> SalesTable's insert signature is parametric
- `update()` -> empty parameters
- display method can also be defined without `public` modifier

---

### 10. Extension Class - Advanced modifiedField + switch Pattern


**Declaration:**
```x++
[ExtensionOf(tableStr(PurchLine))]
final class PurchLine_MyModel_Extension
{


}
```

**Learned Rules:**
- Blank lines can appear inside Declaration (missing/incorrect formatting still compiles)
- `modifiedField(FieldId _fieldId, boolean _userInput)` -> PurchLine's two-parameter signature
- `next modifiedField(_fieldId, _userInput)` -> both parameters are passed
- `switch (_fieldId)` with `case fieldNum(PurchLine, FieldName):` pattern
- `update(boolean dropInvent, boolean updateOrderLineOfDeliverySchedule, boolean updatePurchTableDropShipStatus)` -> multi-parameter override
- `next update(dropInvent, updateOrderLineOfDeliverySchedule, updatePurchTableDropShipStatus)` -> all parameters passed
- Buffer changes are made BEFORE `next update()`, other table operations AFTER
- Same rule for `insert()` override
- `[SysClientCacheDataMethodAttribute(true)]` -> cache attribute for display method
- `this.orig().FieldName` -> access to the record's original value (for change detection)
- CoC override of `registeredInPurchUnit()` display method: `real result = next registeredInPurchUnit();`
- `insert(boolean dropInvent, boolean findMarkup, ...)` - `next insert()` -> parameterless next call is valid (parameters are optional)

---

### 11. Extension Class - Class CoC (SalesInvoiceDP)


**Learned Rules:**
- `[ExtensionOf(classStr(SalesInvoiceDP))]` -> `classStr()` for class extension
- `protected void` methods can also be overridden (not only `public`)
- Long parameter list can be wrapped to next line (4 spaces indented)
- `next` call same way with long parameter list: `next methodName(p1, p2, p3, p4, p5, p6)`
- `salesInvoiceTmp` -> the DP class's protected member field, accessible from CoC without `this.`
- `_custInvoiceTrans.salesLine()` -> access to related table via relation method

---

### 12. Extension Class - Only Static Main (SalesInvoiceController)


**Full XML:**
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

**Learned Rules:**
- `next main(_args)` -> `next` is also used in static methods
- Minimal extension for just an entry point: one method, only `next` call
- `public static void main(Args _args)` -> static method override

---

### 13. Extension Class - Form CoC


**Learned Rules:**
- `[ExtensionOf(formStr(EcoResProductCreate))]` -> `formStr()` for form extension
- `this.updateLayout()` -> recalculate form layout
- `conIns(container, position, element)` -> add to container
- `conLen(container) + 1` -> add at the end of container
- `FormControlEventHandler` can also be defined inside the form extension class (not required to be in a separate EventHandler class)
- `ce.cancelSuperCall()` -> with lowercase c (`CancelSuperCall()` with capital C also seen elsewhere)

---

### 14. Form DataSource Event Handler (Form Extension Class)


**Learned Rules:**
- `[FormDataSourceEventHandler(formDataSourceStr(FormName, DataSourceName), FormDataSourceEventType::Initialized)]`
- `FormDataSourceEventType::Initialized` -> at the initial setup of the datasource
- `FormDataSourceEventType::Activated` -> when a row is selected / becomes active
- `sender.formRun()` -> access to FormRun object
- `sender.cursor()` -> access to active record
- `formRun.design().controlName(formControlStr(FormName, ControlName))` -> finding control by name
- `ctrl as FormRealControl` -> type cast to access special control methods
- `realCtrl.noOfDecimalsValue(4)` -> change decimal places at runtime
- `FormDropDialogButtonControl.enabled(boolean)` -> enable/disable control
- Blank lines and comments can appear inside Declaration

---

### 15. Data Event Handler (Inserted) - Minimal


**Learned Rules:**
- `DataEventType::Inserted` (past tense - AFTER insert)
- Single line if: `if(condition)\n    statement;` (no empty braces)
- `sender` parameter comes as `Common` type, can be cast or used directly inside the method
- If the extension class only contains the DataEventHandler method, no CoC method is required

---

### 16. Normal Class - CLR Interop + API Integration


**Learned Rules:**
- `using NamespaceName;` -> external DLL namespace import, at the top of Declaration CDATA
- `CLRObject` type -> for holding .NET objects
- `new CLRObject("System.Collections.Generic.List\`1[System.String]")` -> creating a generic list
- Multiple variables in one line: `str url1, url2, url3;`
- TAB alignment: `NoYes\t\t\t\t\toverrideRecords;` for visual alignment
- `protected void new(...)` -> to protect the constructor (object can only be created with construct())
- `public static ClassName construct(Common _common = null)` -> factory method

**CLR usage example:**
```x++
CLRObject exchangeRates = new CLRObject("...");
CLRObject enumerator = exchangeRates.GetEnumerator();
while (enumerator.MoveNext())
{
    exchangeRatesList += [enumerator.get_Current()];
}
// CLR property accesses: obj.get_PropertyName(), obj.set_PropertyName(value)
// CLR method call: obj.MethodName(parameter)
```

---

### 17. Normal Class - final + Helper Pattern


**Learned Rules:**
- `public final class` -> inheritance can be prevented (extends is not allowed)
- For `Common` typed parameters, table detection can be done with `_parmSourceLine.TableId`
- `tableNum(SalesLine)` -> table ID comparison in switch case
- `_parmSourceLine as SalesLine` -> cast then initFrom method
- `tableId2Name(tableId)` -> get name from table ID
- `MarkupTrans::lastLineNum(tableId, recId)` -> find last line number
- Helper class: no inheritance, no member variables, only methods

---

### 18. Normal Class - HTTP/Flow Integration


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
        error(strFmt("Error: %1", ex.ToString()));
    }
}
```

**Learned Rules:**
- `System.Net.HttpWebRequest`, `System.Net.HttpWebResponse` etc. -> .NET namespaces can be used directly
- CLR property set: `request.set_Method("POST")`, `request.set_ContentType("...")`
- CLR property get: `System.Text.Encoding::get_UTF8()`
- `as System.Net.HttpWebRequest` -> CLR downcasting
- `Exception::CLRError` catch -> exception retrieved with `CLRInterop::getLastException()`
- `ex.ToString()` -> CLR string conversion
- JSON body: `strFmt('{"key":[%1]}', value)` -> single quotes can be used

---

### 19. Normal Class - Vendor Invoice Creation (VendInvoiceInfo Pattern)


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
        // sub records...
    }
    else
    {
        numberSeq.abort();  // to release the number sequence
    }
}
```

**Learned Rules:**
- `NumberSeq::newGetNum(PurchParameters::numRefParmId())` -> ParmId number sequence
- `numberSeq.num()` -> get number
- `numberSeq.abort()` -> release on failure
- `clear()` -> clear buffer
- `initValue()` -> fill with default values
- `initFromPurchTable()` -> copy from source
- `validateWrite()` -> validation before insert
- `ParmId = Num` and `TableRefId = ParmId` -> standard identity chain

---

### 20. Extension Class - SalesFormletterParmData CoC (insertParmLine)


**Learned Rules:**
- It is shown that `protected void` methods can also be overridden
- `_parmLine.(fieldNum(SalesParmLine, DeliverNow))` -> dynamic field access (with fieldNum)
- `_parmLine.data(salesParmLine)` -> copy data into Common buffer
- `salesParmLine.setLineAmount()` -> refresh calculated field
- `next insertParmLine(_parmLine)` -> called LAST (custom logic first, standard after)

---

## Critical Rules

### XML Structure Rules

1. Root element: `<AxClass xmlns:i="http://www.w3.org/2001/XMLSchema-instance">`
2. `<Name>` content is the same as the file name without `.xml`
3. TAB indentation: one extra TAB for each nested XML element
4. Each `<Source>` CDATA block leaves a blank line after the last line (`\n]]>`)
5. In `<Declaration>` CDATA, a blank line exists after the closing `}`
6. X++ method body is written with 4 spaces (or 1 TAB) indentation

### Class Declaration Rules

| Case | Correct Form |
|---|---|
| Normal class | `class ClassName` or `public class ClassName` |
| Final class | `public final class ClassName` |
| Internal final | `internal final class ClassName` |
| Extends | `public class ClassName extends ParentClass` |
| Implements | `... implements InterfaceName` (next to extends) |
| Extension | `[ExtensionOf(tableStr/classStr/formStr(...))]` + `final class Name_Model_Extension` |
| DataContract | `[DataContract]` or `[DataContractAttribute]` + class |

### CoC Rules

- `next` call must be made **exactly once**
- `next methodName()` -> continues the CoC chain
- `next methodName(param1, param2)` -> parameters passed
- In methods like `modifiedField`, `next` is usually called FIRST (standard first)
- In methods like `insert`, `update`, custom buffer changes are made BEFORE `next`
- Other table operations (MarkupTrans etc.) are made AFTER `next` (deadlock risk)

### Attribute Ordering

Attributes on the class come before the class keyword:
```x++
[SRSReportQueryAttribute(queryStr(...))]
[SRSReportParameterAttribute(classStr(...))]
public class DPClass extends SrsReportDataProviderPreProcessTempDB
```

Multiple attributes on a method with commas and new lines:
```x++
[DataMember,
 SysOperationDisplayOrder('1'),
 SysOperationControlVisibility(false)]
public ReturnType parmXxx(...)
```

---

## Commonly Used X++ Patterns

### parm Method Pattern
```x++
// Simple getter/setter
public DataType parmFieldName(DataType _fieldName = fieldName)
{
    fieldName = _fieldName;
    return fieldName;
}

// Fluent (chaining) - for service class
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

### Collecting Unique Records with Set
```x++
Set purchIdSet = new Set(Types::String);
// fill with while select
purchIdSet.add(purchLine.PurchId);
// usage
SetEnumerator enumerator = purchIdSet.getEnumerator();
while (enumerator.moveNext())
{
    PurchId current = enumerator.current();
}
int count = purchIdSet.elements();
```

### Query Range Value as String
```x++
QueryBuildRange range = qbds.addRange(fieldNum(PurchTable, PurchId));
range.value("PO-001,PO-002,PO-003");  // multiple values with comma
```

### OCC Retry Count Pattern (Batch)
```x++
public void run()
{
    #OCCRetryCount
    super();

    try
    {
        // operations
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

### Insert with validateWrite Check
```x++
if (record.validateWrite())
{
    record.insert();
}
else
{
    numberSeq.abort();  // release the number sequence if any
}
```

### Display Method (Inside Extension Class)
```x++
// Inside table extension class:
display ReturnType methodName()
{
    return this.relatedTable().field;
}

// With cache:
[SysClientCacheDataMethodAttribute(true)]
public display ReturnType methodName()
{
    return this.someCalculation();
}
```

### Lookup Override (FormControlEventHandler)
```x++
[FormControlEventHandler(formControlStr(FormName, ControlName), FormControlEventType::Lookup)]
public static void ControlName_OnLookup(FormControl sender, FormControlEventArgs e)
{
    SysTableLookup sysTableLookup = SysTableLookup::newParameters(tableNum(LookupTable), sender);

    sysTableLookup.addLookupfield(fieldNum(LookupTable, Field1));
    sysTableLookup.addLookupfield(fieldNum(LookupTable, Field2));

    sysTableLookup.performFormLookup();

    FormControlCancelableSuperEventArgs cancelArgs = e as FormControlCancelableSuperEventArgs;
    cancelArgs.CancelSuperCall();  // or cancelArgs.cancelSuperCall()
}
```

### DataEventHandler - Inserting/Updating
```x++
[DataEventHandler(tableStr(TableName), DataEventType::Inserted)]
public static void TableName_onInserted(Common sender, DataEventArgs e)
{
    TableName record = sender as TableName;

    ttsBegin;
    record.selectForUpdate(true);  // or doUpdate
    record.FieldName = value;
    record.doUpdate();
    ttsCommit;
}

[DataEventHandler(tableStr(TableName), DataEventType::Updating)]
public static void ClassName_onUpdating(Common _sender, DataEventArgs _e)
{
    TableName st = _sender;  // direct cast, also valid without 'as'
}
```

### FormDataSourceEventHandler
```x++
[FormDataSourceEventHandler(formDataSourceStr(FormName, DataSourceName), FormDataSourceEventType::Activated)]
public static void DataSource_OnActivated(FormDataSource sender, FormDataSourceEventArgs e)
{
    FormRun formRun = sender.formRun();
    TableName record = sender.cursor() as TableName;

    FormControl ctrl = formRun.design().controlName(formControlStr(FormName, ControlName));
    FormRealControl realCtrl = ctrl as FormRealControl;

    if (realCtrl)
    {
        realCtrl.noOfDecimalsValue(4);
    }
}
```

---

## InventTrans Marking Pattern (Programmatic Marking)

To perform programmatic marking between sales and purchase transactions, `TmpInventTransMark` and `InventTransMarkCollection` are used.

### Basic Marking Method

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

### Important Notes
- `_receiptId`: Receipt side (PurchLine) InventTransId
- `_issueId`: Issue side (SalesLine) InventTransId
- `TmpInventTransMark::markingCollection()` collects suitable marking candidates from the receipt side
- `TmpInventTransMark::updateTmpMark()` performs the marking operation
- `qtyToMark` is the quantity of the issue side (negative value)
- D365 standard marking API is used; `InventTrans.MarkingRefInventTransOrigin` is not updated directly

---

## Attribute Reference

| Attribute | Usage | Example |
|---|---|---|
| `[ExtensionOf(tableStr(T))]` | Table extension class | `tableStr(SalesTable)` |
| `[ExtensionOf(classStr(C))]` | Class extension class | `classStr(SalesInvoiceDP)` |
| `[ExtensionOf(formStr(F))]` | Form extension class | `formStr(EcoResProductCreate)` |
| `[DataEventHandler(tableStr(T), DataEventType::X)]` | Table event handler | `DataEventType::Inserted` |
| `[FormControlEventHandler(formControlStr(F,C), FormControlEventType::X)]` | Form control event | `FormControlEventType::Lookup` |
| `[FormDataSourceEventHandler(formDataSourceStr(F,DS), FormDataSourceEventType::X)]` | Form datasource event | `FormDataSourceEventType::Activated` |
| `[SysEntryPointAttribute]` | Entry point | On `main()` method |
| `[DataContract]` | DataContract class | On class |
| `[DataContractAttribute]` | DataContract (Attribute suffix) | On class (alternative) |
| `[DataMember]` | DataContract member | On parm method |
| `[DataMemberAttribute('Key')]` | DataContract member + key name | On parm method |
| `[SRSReportQueryAttribute(queryStr(Q))]` | DP query binding | On DP class |
| `[SRSReportParameterAttribute(classStr(C))]` | DP contract binding | On DP class |
| `[SRSReportDataSetAttribute(tableStr(T))]` | DP dataset method | On get* method |
| `[SysOperationDisplayOrder('N')]` | Dialog ordering | On parm method |
| `[SysOperationControlVisibility(false)]` | Dialog hiding | On parm method |
| `[Hookable(false)]` | Disable CoC | On methods like `isRetryable()` |
| `[SysClientCacheDataMethodAttribute(true)]` | Display method cache | On display method |

---

## Naming Conventions (From Real Files)

| Type | Pattern | Example |
|---|---|---|

---

## Important Observations

1. **`class` vs `public class`**: In normal classes, `public` is optional. In extension classes, `public` is never used.

2. **`final` modifier**: Extension classes and some helper classes use `final`.

3. **`internal` modifier**: Internal classes are not exposed outside the same model. The `internal final class` combination is often seen.

4. **Empty Declaration body**: `{\n}` (two lines) or `{ }` (one line) are all valid.

5. **Method signature compatibility**: In CoC, the signature of the overridden method must exactly match the source. In `PurchLine.update(boolean, boolean, boolean)` override, the parameters are written exactly the same.

6. **Parameter in `next` call**: A parameterless call like `next update()` is also valid in X++ - parameters are passed optionally.

7. **Comment lines**: `//` for single-line, `/* */` for block comment. DocComment (`/// <summary>`) is also used.

8. **`#OCCRetryCount` macro**: Pre-built macro used in RunBaseBatch.run() override, together with `#RetryNum`.

9. **`ttsBegin`/`ttsCommit` case insensitivity**: No case difference - mixed usage observed in the project.

10. **MultiSelectionContext**: Multiple selection from form: `_args.multiSelectionContext()`, `.getFirst()`, `.getNext()`.

---

# AxTable - Table Definitions


---

## XML Structure - General Template


---

## Real Table Examples

### Example 15: Parameter Table (AxTableFieldTime + initValue override)

**Lessons Learned:**
- `<AxTableFieldTime>` type - used with `TimeOfDay` EDT - `time` type in X++
- `initValue()` override: automatic ID assignment when record is created
- Project-specific macro usage (like `#MyMacro`)
- `ClusteredIndex` and `PrimaryIndex` can be written as empty string: `<PrimaryIndex></PrimaryIndex>`
- `index hint IndexName` - index hint for query optimization
- `like` operator for wildcard pattern: `where oldTable.ParmID like newParm`
- `strDel`, `strLen`, `num2Str`, `str2Num` string manipulation functions

---

## Field Types Full Reference

### i:type Values and Usage Rules

| i:type | X++ Type | Additional Required/Optional Properties | Real Example |
|--------|----------|--------------------------------|--------------|
| `AxTableFieldString` | `str` | `<StringSize>` optional (default 10) | `CargoCompany`, `AttributeName` |
| `AxTableFieldReal` | `real` | - | `SalesExchangeRate`, `AmountAED`, `LineNumber` |
| `AxTableFieldEnum` | `enum` | `<EnumType>` required | `OverrideRecords`, `SalesStatus`, `AttributeType` |
| `AxTableFieldTime` | `time` | Used with `TimeOfDay` EDT | `BeginTime`, `EndTime` |

### CRITICAL: EDT - Field Type Compatibility Rule

Compatibilities verified from the real project:

| EDT | Base Type | Correct Field Type | Project Example |
|-----|---------|-----------------|--------------|
| `NoYesId` | enum | `AxTableFieldEnum` | Common in all tables |
| `TimeOfDay` | time | `AxTableFieldTime` | Parameter tables |

**ATTENTION - LineNumber EDT is real type:**
```xml
<!-- CORRECT: LineNumber EDT is real-based -->
<AxTableField xmlns="" i:type="AxTableFieldReal">
    <Name>LineNumber</Name>
    <ExtendedDataType>LineNumber</ExtendedDataType>
    <IgnoreEDTRelation>Yes</IgnoreEDTRelation>
</AxTableField>

<!-- WRONG: Cannot be used with Int -->
<AxTableField xmlns="" i:type="AxTableFieldInt">
    <Name>LineNumber</Name>
    <ExtendedDataType>LineNumber</ExtendedDataType>
</AxTableField>
```

---

## Index Types and Rules

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

- Guarantees uniqueness (UNIQUE index)
- Bound to table with `<ReplacementKey>CargoCompanyIDx</ReplacementKey>`
- Can contain multiple fields (composite unique key)

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

- Only for performance and search
- No uniqueness guarantee

### Normal Index (neither AlternateKey nor AllowDuplicates)

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

- Default: non-unique (AllowDuplicates=No default)
- `AlternateKey` not specified = No

### IsSystemGenerated Index (Staging tables)

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

### Using System Fields Inside Index

```xml
<!-- CreatedBy system field can be used inside index -->
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

## Relation Types and Rules

### Standard Field Relation

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

### ForeignKey Relation (In Extensions)


- Marked with `AxTableRelationForeignKey` i:type
- `<Index>RecId</Index>` specifies which index of the related table is used

### RelatedFixed Constraint (Fixed value condition)


- `AxTableRelationConstraintRelatedFixed`: filter by fixed value of the field in the related table
- `<ValueStr>` X++ enum value written as string

### Validate=No Relation


- `<Validate>No</Validate>` - referential integrity is not checked

### OnDelete Values

| Value | Behavior | Usage |
|-------|----------|----------|
| `Cascade` | Related records are automatically deleted | Parent-child relationship |
| `Restricted` | Delete is prevented while related record exists | Reference tables |
| (not specified) | No restriction | View relations |

---

## Field Groups - Required and Optional

### The 5 standard groups that must exist in every table:

```xml
<FieldGroups>
    <AxTableFieldGroup>
        <Name>AutoReport</Name>
        <Fields>
            <!-- Fields to be displayed in report -->
        </Fields>
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoLookup</Name>
        <Fields />  <!-- Usually empty -->
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoIdentification</Name>
        <AutoPopulate>Yes</AutoPopulate>
        <Fields>
            <!-- Unique identity fields (optional but recommended) -->
        </Fields>
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoSummary</Name>
        <Fields />  <!-- Usually empty -->
    </AxTableFieldGroup>
    <AxTableFieldGroup>
        <Name>AutoBrowse</Name>
        <Fields />  <!-- Usually empty -->
    </AxTableFieldGroup>
</FieldGroups>
```

### Special groups:

| Group Name | Usage |
|----------|----------|
| `Grid` | Field group for form grid control |
| `Calculated` | Group containing display methods |

### Group Notes (From Real Project):

- `AutoReport`: always contains main fields (can be left empty except TempDB)
- `AutoIdentification` + `<AutoPopulate>Yes</AutoPopulate>`: always required, fields optional
- `<IsSystemGenerated>Yes</IsSystemGenerated>`: in staging tables, system groups are marked with this attribute

---

## Table Properties

### SaveDataPerCompany

| Value | Meaning | When to use |
|-------|-------|---------------------|

**Rule:** Tables with `SaveDataPerCompany=No` are usually:
- Company-independent data like marketplace, product attributes
- Integration parameters (API URL, user info)
- Global lookup/setup tables

### TableType

| Value | Meaning | Features |
|-------|-------|-----------|
| (not specified) | Normal persistent table | Physical table in database |
| `TempDB` | Temporary database table | SSRS report, session-based data |
| (Staging) | DMF staging | Together with `<TableGroup>Staging</TableGroup>` |

### AllowRowVersionChangeTracking

- `<AllowRowVersionChangeTracking>Yes</AllowRowVersionChangeTracking>`: row version change tracking
- Used in performance queries
- Found in almost every table in the real project

### CacheLookup

- `<CacheLookup>EntireTable</CacheLookup>`: entire table is cached
- Used in small and frequently read setup/lookup tables

### CreatedBy / ModifiedBy / CreatedDateTime / ModifiedDateTime

```xml
<CreatedBy>Yes</CreatedBy>
<CreatedDateTime>Yes</CreatedDateTime>
<ModifiedBy>Yes</ModifiedBy>
<ModifiedDateTime>Yes</ModifiedDateTime>
```

- For audit trail - who/when created/modified
- Usually skipped in TempDB and simple lookup tables

### TitleField1 / TitleField2

```xml
<TitleField1>CargoCompany</TitleField1>
<TitleField2>CargoCompanyDescription</TitleField2>
```

- Fields displayed in form title
- In lookup tables, the first field is usually `code`, the second field is `description`

### ReplacementKey

```xml
<ReplacementKey>CargoCompanyIDx</ReplacementKey>
```

- Natural key of the table - name of index with AlternateKey=Yes
- Used in form navigation and FK relationships
- Recommended to specify in every table

### ConfigurationKey

```xml
<ConfigurationKey>LogisticsBasic</ConfigurationKey>
```

- Seen in staging tables
- Module activation key

## Standard Table Descriptions Starting with Invent

This section is a quick summary of the table family found as `AxTable/Invent*.xml` under standard system metadata.

**Detailed references:**
- Field-based full reference (569 tables, 14800+ lines): [docs/MdDocuments/Invent_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Invent_Tablolari_Alan_Referansi_20260403.md)
- Core operations flow guide: [docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md](docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md)

- Source version: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Scope: 569 unique standard tables
- Note: Due to the `Invent*` wildcard, sub-families like `InventInventory*`, `InventCost*`, and `InventCDS*` are also included in this scope.

### Category Summary

| Category | Count | Description |
|---|---:|---|
| Core | 30 | Core inventory master data, on-hand, journal, and transfer transactions |
| Other | 230 | Costing, quality, reporting, data service, and support tables |
| History | 13 | Historical/snapshot copies |
| Tmp | 101 | Temporary processing and reporting buffers |
| Localization | 49 | Country/region or public sector extensions |
| Staging | 146 | DMF / entity staging transfer tables |

### Core Table Family

| Table | Description |
|---|---|
| `InventTable` | Company-based inventory, planning, cost, warehouse, and supply settings of the product |
| `InventDim` | Site, warehouse, batch, serial, and variant-based inventory dimension combinations |
| `InventSum` | On-hand quantity summary per dimension |
| `InventTrans` | Physical and financial transaction records of all inventory movements |
| `InventBatch` | Batch/lot master data and shelf life/traceability information |
| `InventParameters` | Inventory management module system parameters |
| `InventLocation` | Warehouse definitions |
| `InventSite` | Site/facility definitions |
| `InventItemGroup` | Inventory and accounting behavior groups based on item group |
| `InventModelGroup` | Reservation, costing, and inventory model behavior |
| `InventJournalName` | Naming and defaults for journal types |
| `InventJournalTable` | Inventory journal headers |
| `InventJournalTrans` | Inventory journal lines |
| `InventTransferTable` | Transfer order header |
| `InventTransferLine` | Transfer order lines |
| `InventTransferParmTable` | Transfer posting process header parameters |
| `InventTransferParmLine` | Transfer posting process line parameters |
| `InventQualityOrderTable` | Quality order header |
| `InventSettlement` | Inventory close/settlement links |
| `InventCostTrans` | Inventory costing detail transactions |

### Other Important Table Families

| Pattern / Family | Description |
|---|---|
| `InventInventory*Staging` | Staging tables for DMF and entity transfers |
| `InventAging*` | Inventory aging reports and calculation stores |
| `InventCost*` | Valuation, cost list, closing, and cost transaction support family |
| `InventClosing*` | Inventory close and close log process |
| `InventCDSInventoryOnHand*` | CDS/on-hand data service request, response, and staging objects |
| `InventCountingReasonCode*` | Counting reason codes, groups, and policies |
| `InventFiscalLIFO*_RU` | Russia costing / fiscal LIFO localization |
| `InventBailee*_RU` | Russia consignment/bailee localization |
| `*History` | Historical/snapshot copies |
| `*Tmp` | Temporary report, UI, or processing buffer tables |
| `*Staging` | Staging tables for DMF / OData / entity transfers |

### Invent Core Operations Flow

**Typical relationship chain:**

`InventTable` -> `InventDim` -> `InventSum`

`InventTable` -> `InventTrans` -> `InventDim`

`InventTrans` -> `InventBatch` (if batch dimension exists, via `InventDim`)

`InventJournalTable` -> `InventJournalTrans`

`InventTransferTable` -> `InventTransferLine` -> `InventTrans`

`InventTrans` -> `InventSettlement` / `InventCostTrans`

**Which table to look at for which scenario?**

| Scenario | First table to check | Then check |
|---|---|---|
| Why is on-hand this much? | `InventSum` | `InventDim`, `InventTrans` |
| Which transaction created this balance? | `InventTrans` | `InventDim`, related source document |
| Batch info / shelf life? | `InventBatch` | `InventDim`, `InventTrans` |
| What did the journal post? | `InventJournalTable` | `InventJournalTrans`, `InventTrans` |
| What happened in transfer? | `InventTransferTable` | `InventTransferLine`, `InventTrans` |
| Cost close result reflected where? | `InventSettlement` | `InventCostTrans`, `InventTrans` |
| Why is the inventory behavior of the item like this? | `InventTable` | `InventParameters`, model/item group fields |

**Quick notes:**
- For on-hand analysis, the first table is usually `InventSum`; for root cause analysis, the main table is `InventTrans`.
- Batch connection is not directly in every table; usually goes through `InventDim`.
- Cost differences and close impacts are read more from `InventSettlement` and `InventCostTrans` than live transactions.
- In journal and transfer scenarios, the document table and the transaction table must be read together.

## Standard Table Descriptions Starting with Purch

This section is a quick summary of the table family found as `AxTable/Purch*.xml` under standard system metadata.

**Detailed references:**
- Field-based full reference (368 tables, 11000+ lines): [docs/MdDocuments/Purch_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Purch_Tablolari_Alan_Referansi_20260403.md)
- Core operations flow guide: [docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md](docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md)

- Source version: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Scope: 368 unique standard tables
- Note: Due to the `Purch*` wildcard, tables named `Purchase...` like `PurchaseOrderResponse*` are also included in this family.

### Category Summary

| Category | Count | Description |
|---|---:|---|
| Core | 23 | Core purchase business data and posting parameters |
| Other | 113 | Helper workflows, policy, response, copilot, and connection tables |
| History | 28 | Historical/snapshot copies |
| Tmp | 54 | Temporary processing and reporting buffers |
| Localization | 28 | Country/region or public sector extensions |
| Staging | 122 | DMF / entity staging transfer tables |

### Core Table Family

| Table | Description |
|---|---|
| `PurchTable` | Purchase order header; vendor, currency, delivery, date, status, and general commercial settings |
| `PurchLine` | Purchase order line; product/service, quantity, price, inventory, tax, delivery, and accounting details |
| `PurchParameters` | Purchase module system parameters, number sequences, and behavior settings |
| `PurchParmTable` | Header-level purchase parameters for posting process |
| `PurchParmLine` | Line-level purchase parameters for posting process |
| `PurchParmSubTable` | Header buffer parameters for posting sub-breakdowns |
| `PurchParmSubLine` | Line buffer parameters for posting sub-breakdowns |
| `PurchParmUpdate` | Purchase posting / update behavior selections |
| `PurchDeliverySchedule` | Partial delivery plans for purchase lines |
| `PurchAgreementHeader` | Purchase agreement header |
| `PurchAgreementHeaderDefault` | Purchase agreement default values |
| `PurchAgreementActivity` | Activity / usage records linked to purchase agreement |
| `PurchAgreementCertification` | Certification records linked to purchase agreement |
| `PurchAgreementSubcontractor` | Purchase agreement to subcontractor relationship |
| `PurchLineOrigin` | Origin source and reference chain of the purchase line |
| `PurchOrderRFQLineReference` | Reference between PO line and RFQ line |
| `PurchConfirmationRequestJour` | Purchase confirmation request journal records |
| `PurchEncumbranceSummary` | Purchase side encumbrance / budget commitment summary |
| `PurchJournalAutoSummary` | Helper data for purchase journal summarization behavior |
| `PurchPool` | Purchase order pool definition |
| `PurchPrepayTable` | Purchase prepayment records |
| `PurchPriceTolerance` | Purchase price tolerance rules |
| `PurchPurchaseOrderHeader` | Standard table carrying purchase order header in integration / entity-focused way |

### Other Important Table Families

| Pattern / Family | Description |
|---|---|
| `PurchaseOrderResponse*` | Vendor collaboration side purchase order response / accept / reject process |
| `PurchCopilot*` | Email mapping, task, knowledge base, and review buffers for purchase Copilot scenarios |
| `PurchReqAuthorization*` | Purchase request authorization scope and organization breakdowns |
| `PurchReApprovalPolicyRule*` | Re-approval rules and rule field definitions |
| `PurchBook*_RU` | Russia purchase book and VAT localization |
| `PurchImportDeclaration_BR` | Brazil import declaration and foreign trade localization |
| `PurchCommitment*_PSN` | Public Sector purchase commitment/budget structures |
| `*History` | Historical/snapshot copies of documents or lines |
| `*Tmp` | Temporary report, UI, or processing buffer tables |
| `*Staging` | Staging tables for DMF / OData / entity transfers |

### Purch Core Operations Flow

**Typical relationship chain:**

`PurchTable` -> `PurchLine` -> `InventDimId` / `InventTransId`

`PurchTable` -> `PurchParmTable` -> `PurchParmLine`

`PurchLine` -> `PurchDeliverySchedule`

`PurchLine` -> `PurchLineOrigin`

`PurchTable` or `PurchLine` -> `PurchAgreementHeader`

**Which table to look at for which scenario?**

| Scenario | First table to check | Then check |
|---|---|---|
| Why is the order in this state? | `PurchTable` | `PurchLine`, `PurchParmUpdate` |
| Why did the line receive a different quantity? | `PurchLine` | `PurchParmLine`, `PurchDeliverySchedule` |
| What happened during posting? | `PurchParmTable` | `PurchParmLine`, `PurchParmUpdate` |
| What is the line's source? | `PurchLineOrigin` | `PurchLine`, related source document |
| Is there an agreement connection? | `PurchAgreementHeader` | `PurchLine`, `PurchTable` |
| What is the vendor confirmation history? | `PurchConfirmationRequestJour` | `PurchaseOrderResponse*` family |

**Quick notes:**
- `PurchTable` is the header; the main detail of the operation is usually on the `PurchLine` side.
- For product receipt / invoice differences, looking at the live line directly is not enough; `PurchParm*` buffers are critical.
- If there are delivery differences, `PurchDeliverySchedule` must be checked.
- For source document chain, `PurchLineOrigin` helps, and for RFQ connection, `PurchOrderRFQLineReference` helps.

## Standard Table Descriptions Starting with Sales

This section is a quick summary of the table family found as `AxTable/Sales*.xml` under standard system metadata.

**Detailed references:**
- Field-based full reference (267 tables, 11100+ lines): [docs/MdDocuments/Sales_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Sales_Tablolari_Alan_Referansi_20260403.md)

- Source version: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Scope: 267 unique standard tables
- Note: Due to the `Sales*` wildcard, sub-families like `SalesInvoice*`, `SalesOrder*`, and `SalesQuotation*` are also included in this scope.

### Category Summary

| Category | Count | Description |
|---|---:|---|
| Core | 15 | Core sales order, quotation, and posting parameters |
| Other | 49 | Document, invoice, confirmation, pricing, and support tables |
| History | 13 | Historical, archive, and snapshot copies |
| Tmp | 33 | Temporary processing and reporting buffers |
| Localization | 31 | Country/region or public sector extensions |
| Staging | 126 | DMF / entity staging transfer tables |

### Core Table Family

| Table | Description |
|---|---|
| `SalesTable` | Sales order header; customer, currency, delivery, and commercial settings |
| `SalesLine` | Sales order line; product, quantity, price, delivery, and inventory connections |
| `SalesParameters` | Sales module system parameters and behavior settings |
| `SalesParmTable` | Header-level sales parameters for posting process |
| `SalesParmLine` | Line-level sales parameters for posting process |
| `SalesParmSubTable` | Header buffer parameters for posting sub-breakdowns |
| `SalesParmSubLine` | Line buffer parameters for posting sub-breakdowns |
| `SalesParmUpdate` | Sales posting / update behavior selections |
| `SalesQuotationTable` | Sales quotation header |
| `SalesQuotationLine` | Sales quotation line |
| `SalesQuotationParmTable` | Quotation posting/confirm header parameters |
| `SalesQuotationParmLine` | Quotation posting/confirm line parameters |
| `SalesAgreementHeader` | Sales agreement header |
| `SalesDeliverySchedule` | Partial delivery plans for sales lines |
| `SalesPool` | Sales order pool definition |

### Other Important Table Families

| Pattern / Family | Description |
|---|---|
| `SalesInvoice*` | Sales invoice, invoice details, and support/staging family |
| `SalesOrderHeaderV*` / `SalesOrderLineV*` | Entity-focused order header/line families |
| `SalesConfirm*` | Confirmation process tables |
| `SalesPackingSlip*` | Packing slip process tables |
| `SalesTableHistory` / `SalesLineHistory` | Historical and snapshot copies |
| `*Tmp` | Temporary report, UI, or processing buffer tables |
| `*Staging` | Staging tables for DMF / OData / entity transfers |

## Standard Table Descriptions Starting with Vend

This section is a quick summary of the table family found as `AxTable/Vend*.xml` under standard system metadata.

**Detailed references:**
- Field-based full reference: [docs/MdDocuments/Vend_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Vend_Tablolari_Alan_Referansi_20260403.md)

- Source version: `C:\Users\Monster\AppData\Local\Microsoft\Dynamics365\10.0.2345.153\PackagesLocalDirectory`
- Scope: 411 unique standard tables
- Note: Due to the `Vend*` wildcard, sub-families like `VendInvoice*`, `VendPaym*`, `VendRFQ*`, and similar are also included in this scope.

### Category Summary

| Category | Count | Description |
|---|---:|---|
| Core | 16 | Core vendor, AP, product receipt, and invoice tables |
| Other | 159 | Vendor invoice, payment, RFQ, and support tables |
| History | 6 | Historical, archive, and snapshot copies |
| Tmp | 90 | Temporary processing and reporting buffers |
| Localization | 57 | Country/region or public sector extensions |
| Staging | 83 | DMF / entity staging transfer tables |

### Core Table Family

| Table | Description |
|---|---|
| `VendTable` | Vendor master data; account, payment, delivery, and general commercial settings |
| `VendParameters` | Accounts Payable / vendor module system parameters |
| `VendGroup` | Vendor group definitions |
| `VendBankAccount` | Vendor bank accounts |
| `VendPaymModeTable` | Payment method / paym mode definitions |
| `VendInvoiceJour` | Vendor invoice header |
| `VendInvoiceInfoTable` | Pending vendor invoice header |
| `VendInvoiceInfoLine` | Pending vendor invoice lines |
| `VendInvoiceTrans` | Vendor invoice lines |
| `VendInvoiceMatchingLine` | Invoice matching line records |
| `VendPackingSlipJour` | Product receipt / packing slip header |
| `VendPackingSlipTrans` | Product receipt / packing slip lines |
| `VendTrans` | Vendor financial/AP transactions |
| `VendTransOpen` | Open vendor transactions |
| `VendSettlement` | Vendor settlement links |
| `VendRFQJour` | Vendor request for quotation header |

### Other Important Table Families

| Pattern / Family | Description |
|---|---|
| `VendInvoice*` | Vendor invoice, matching, save status, and support family |
| `VendPaym*` | Vendor payment and payment journal families |
| `VendRFQ*` | Vendor request for quotation and response process |
| `VendProductReceipt*` | Product receipt and related support/staging family |
| `VendTrans*` | Vendor transaction, open balance, and settlement family |
| `*Tmp` | Temporary report, UI, or processing buffer tables |
| `*Staging` | Staging tables for DMF / OData / entity transfers |

---

## Standard Methods (With Real Examples)

### validateWrite() - Conditional Validation

```x++
public boolean validateWrite()
{
    boolean ret;

    ret = super();

    if(this.SalesStatus != SalesStatus::Invoiced && this.SalesStatus != SalesStatus::Delivered)
    {
        ret = false;
        error(strFmt("%1 status is wrong for %2 record", this.SalesStatus, this.DocumentId));
    }

    return ret;
}
```

### Display Method - On Table


---

## CRITICAL RULES

### AssetClassification - When Required?

**Real project analysis:**

- `<AssetClassification>Customer Content</AssetClassification>` - for fields containing customer data
- `<AssetClassification>System Metadata</AssetClassification>` - for system metadata fields (in TempDB)

**Conclusion:** AssetClassification is NOT required. When specified:
- `Customer Content`: real customer data like ProdId, ItemId, weight fields
- `System Metadata`: system reference fields like LedgerJournalId, MainAccountNum

### Label Format

All formats observed from the real project:


### IgnoreEDTRelation

```xml
<IgnoreEDTRelation>Yes</IgnoreEDTRelation>
```

- Ignores relations defined in the EDT
- Very common in the real project - `IgnoreEDTRelation>Yes` is in almost every EDT usage
- Used as required in `RefRecId`, `NoYesId`, custom EDTs in particular
- Has also been used without it in some fields (`ItemId` + `AssetClassification>Customer Content`)

### AllowEdit vs AllowEditOnCreate

| Combination | Meaning |
|-------------|-------|
| `AllowEdit>No` + `AllowEditOnCreate>No` | Never editable (filled by the system) |
| `AllowEdit>No` (AllowEditOnCreate not specified) | Editable on creation, then read-only |
| (neither specified) | Always editable |

---

# AxTableExtension - Table Extensions

## Extension Rules

### Naming

| Object | Format | Example |
|-------|--------|-------|


### What Can Be Done in Extension

1. **Add new field** - inside `<Fields>`
2. **Add new FieldGroup** - inside `<FieldGroups>` (new custom group)
3. **Add field to existing FieldGroup** - inside `<FieldGroupExtensions>`
4. **Add new relation** - inside `<Relations>`
5. **Add new index** - inside `<Indexes>`
6. **Change table property** - inside `<PropertyModifications>` (rarely)

### What Cannot Be Done in Extension

- Properties of existing fields cannot be changed (`<FieldModifications>` exists but rarely used)
- Existing methods cannot be overridden (separate Extension Class required for CoC)
- Existing indexes cannot be changed

### Differences Between Extension and Normal Table

| Property | Normal Table | Extension |
|---------|-------------|-----------|
| `<SourceCode>` | Exists | None |
| `<Label>` | Exists (table label) | None |
| `<SaveDataPerCompany>` | Can be specified | Can be changed with `<PropertyModifications>` |
| `<TableType>` | Can be specified | Cannot be changed |
| `<StateMachines>` | Exists | None |
| `<DeleteActions>` | Exists | None |

---

## Inconsistencies Observed in Real Project

This section documents real code inconsistencies in the project. These inconsistencies should be avoided when writing new code.

### 4. Prefix Inconsistency

- Both are used in the same project

---

## Quick Reference: Which Table Has What?

| Table | SaveDataPerCompany | CacheLookup | TitleField | CreatedBy/Modified | Type |
|-------|-------------------|-------------|------------|---------------------|-----|

---

# AxForm - Form Definitions

## Namespace: Microsoft.Dynamics.AX.Metadata.V6

Namespace used in AxForm files:
```xml
<AxForm xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="Microsoft.Dynamics.AX.Metadata.V6">
```

Both namespaces are required:
- `xmlns:i="http://www.w3.org/2001/XMLSchema-instance"` - for `i:type` and `i:nil` attributes
- `xmlns="Microsoft.Dynamics.AX.Metadata.V6"` - to specify the metadata format

---

## XML Structure - General Template


---

## CRITICAL: xmlns="" Rule Inside SourceCode Block

Each sub-element inside `<SourceCode>` must reset the namespace with `xmlns=""`:
```xml
<SourceCode>
    <Methods xmlns="">          <!-- xmlns="" REQUIRED -->
        ...
    </Methods>
    <DataSources xmlns="">       <!-- xmlns="" REQUIRED -->
        ...
    </DataSources>
    <DataControls xmlns="">      <!-- xmlns="" REQUIRED -->
        ...
    </DataControls>
    <Members xmlns="" />        <!-- xmlns="" REQUIRED -->
</SourceCode>
```

The `<DataSources>` block (design-level, outside SourceCode) starts with `xmlns=""`:
```xml
<DataSources>
    <AxFormDataSource xmlns="">   <!-- AxFormDataSource has xmlns="" -->
        ...
    </AxFormDataSource>
</DataSources>
```

---

## CRITICAL: FormControlExtension i:nil="true" Rule

The `<FormControlExtension i:nil="true" />` line is REQUIRED inside every `<AxFormControl>`. The only exception: if the control has an extension (rarely), then the empty element is not written. In practice, `i:nil="true"` is used in all controls.

```xml
<AxFormControl xmlns="" i:type="AxFormStringControl">
    <Name>ControlName</Name>
    <Type>String</Type>
    <FormControlExtension
        i:nil="true" />       <!-- ALWAYS WRITTEN -->
    <DataField>FieldName</DataField>
    <DataSource>TableName</DataSource>
</AxFormControl>
```

---

## CRITICAL: Controls Block Inside Design

`<Controls>` inside `<Design>` and all sub-controls start with `xmlns=""` attribute:


Rule: The `<Controls>` block itself does NOT have `xmlns=""` (except for top-level `<Controls xmlns="">`). But every `<AxFormControl>` element inside it has `xmlns=""`.

---

## Form Control Types - FULL REFERENCE

### Required Properties (For Every Control)

Each `<AxFormControl>` block REQUIRED contains these items:
1. `xmlns=""` - element attribute
2. `i:type="AxForm...Control"` - element attribute
3. `<Name>` - unique control name
4. `<Type>` - type text matching i:type
5. `<FormControlExtension i:nil="true" />` - extension placeholder

### Control Type Matching Table

| i:type | `<Type>` value | X++ Type | Usage |
|---|---|---|---|
| `AxFormActionPaneControl` | `ActionPane` | - | Top button bar |
| `AxFormActionPaneTabControl` | `ActionPaneTab` | - | Tab inside ActionPane |
| `AxFormButtonGroupControl` | `ButtonGroup` | - | Button group |
| `AxFormMenuFunctionButtonControl` | `MenuFunctionButton` | - | Button linked to MenuItem |
| `AxFormCommandButtonControl` | `CommandButton` | - | OK, Cancel, New, Delete, etc. |
| `AxFormDropDialogButtonControl` | `DropDialogButton` | - | Drop dialog button |
| `AxFormGroupControl` | `Group` | - | Container group |
| `AxFormTabControl` | `Tab` | - | Tab container |
| `AxFormTabPageControl` | `TabPage` | - | Tab page |
| `AxFormGridControl` | `Grid` | - | Data grid |
| `AxFormStringControl` | `String` | `str` | Text field |
| `AxFormRealControl` | `Real` | `real` | Decimal number |
| `AxFormIntegerControl` | `Integer` | `int` | Integer |
| `AxFormInt64Control` | `Int64` | `int64` | 64-bit integer (for RecId) |
| `AxFormDateTimeControl` | `DateTime` | `utcdatetime` | Date/time |
| `AxFormDateControl` | `Date` | `date` | Date |
| `AxFormCheckBoxControl` | `CheckBox` | `NoYes` | Checkbox |
| `AxFormComboBoxControl` | `ComboBox` | `enum` | Dropdown |
| `AxFormStaticTextControl` | `StaticText` | - | Read-only label |

### AxFormCommandButtonControl - Command Button

```xml
<AxFormControl xmlns="" i:type="AxFormCommandButtonControl">
    <Name>BtnName</Name>
    <AutoDeclaration>Yes</AutoDeclaration>
    <ElementPosition>596523234</ElementPosition>  <!-- System-generated positioning -->
    <FilterExpression>%1</FilterExpression>
    <HeightMode>Auto</HeightMode>
    <Type>CommandButton</Type>
    <VerticalSpacing>-1</VerticalSpacing>
    <WidthMode>Auto</WidthMode>
    <FormControlExtension i:nil="true" />
    <ButtonDisplay>TextWithImageLeft</ButtonDisplay>
    <!-- Command values: New | DeleteRecord | OK | Cancel | Save | Revert -->
    <Command>New</Command>
    <MultiSelect>Yes</MultiSelect>
    <NormalImage>Add</NormalImage>      <!-- Add | Delete | Edit, etc. -->
    <Primary>Yes</Primary>
    <SaveRecord>No</SaveRecord>         <!-- For Cancel buttons -->
    <ShowShortCut>No</ShowShortCut>
    <Text>@SYS319116</Text>
</AxFormControl>
```

### AxFormStaticTextControl - Static Text

```xml
<AxFormControl xmlns="" i:type="AxFormStaticTextControl">
    <Name>TitleText</Name>
    <Skip>Yes</Skip>
    <Type>StaticText</Type>
    <WidthMode>SizeToAvailable</WidthMode>
    <FormControlExtension i:nil="true" />
    <Style>MainInstruction</Style>      <!-- Title style -->
    <Text>Title Text</Text>
</AxFormControl>
```

### AxFormGridControl - Grid Properties

```xml
<AxFormControl xmlns="" i:type="AxFormGridControl">
    <Name>GridName</Name>
    <AutoDeclaration>Yes</AutoDeclaration>
    <Type>Grid</Type>
    <Visible>No</Visible>                        <!-- Hidden grid -->
    <FormControlExtension i:nil="true" />
    <Controls>...</Controls>
    <DataGroup>AutoReport</DataGroup>            <!-- Table field group name -->
    <DataSource>TableName</DataSource>            <!-- Linked datasource -->
</AxFormControl>
```

### AxFormComboBoxControl - ComboBox

```xml
<AxFormControl xmlns="" i:type="AxFormComboBoxControl">
    <Name>ComboName</Name>
    <Type>ComboBox</Type>
    <FormControlExtension i:nil="true" />
    <DataField>EnumField</DataField>
    <DataSource>TableName</DataSource>
    <Items />     <!-- REQUIRED - Must be written even if empty -->
</AxFormControl>
```

### AxFormStringControl - Multi-line Text

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
    <!-- No DataField - standalone -->
</AxFormControl>
```

---

## DataSource Definitions

### AxFormDataSource - Full Properties

```xml
<AxFormDataSource xmlns="">
    <Name>DataSourceName</Name>               <!-- REQUIRED - Reference name in form -->
    <Table>TableName</Table>                  <!-- REQUIRED - Physical table name -->
    <Fields>
        <AxFormDataSourceField>
            <DataField>Field1</DataField>
        </AxFormDataSourceField>
        <!-- All relevant fields are listed -->
        <!-- System fields are also included: DataAreaId, Partition, RecId, TableId -->
        <!-- CreatedBy, CreatedDateTime, ModifiedBy, ModifiedDateTime -->
    </Fields>
    <ReferencedDataSources />               <!-- Usually empty -->

    <!-- OPTIONAL PROPERTIES -->
    <JoinSource>ParentDSName</JoinSource>    <!-- Join with parent datasource -->
    <AllowCreate>No</AllowCreate>           <!-- Prevent adding new record -->
    <AllowDelete>No</AllowDelete>           <!-- Prevent delete -->
    <AllowEdit>No</AllowEdit>               <!-- Prevent edit -->
    <InsertAtEnd>No</InsertAtEnd>           <!-- Append at end -->
    <InsertIfEmpty>No</InsertIfEmpty>       <!-- REQUIRED - Should auto-add row to empty table -->

    <DataSourceLinks />                     <!-- Different datasource bindings -->
    <DerivedDataSources />                  <!-- Derived datasources -->
</AxFormDataSource>
```

### InsertIfEmpty Rule
- `No` (recommended): No automatic empty row is added if no record exists
- `Yes`: If the table is empty, a new record is automatically created
- For parameter tables, `Yes` is more suitable (there should always be one record)

### Difference Between DataSource and DataSource Name
- `<Name>`: Alias used in the form (example: `MarkupAutoTable`)
- `<Table>`: Physical table name (example: `MarkupAutoTable`)
- Usually the same but can differ

### Join Relationship - JoinSource

---

## Form Methods

### classDeclaration - Form Class Definition

```x++
[Form]                              // REQUIRED attribute
public class FormName extends FormRun
{
    // Form-level member variables
    Common      callerRecord;
    TableId     callerTableId;
    RecId       filteredRecId;
}
```

**Rules:**
- `[Form]` attribute must be written
- Every form must be derived from `FormRun`
- Member variables are defined here

### init - Form Initialization

```x++
public void init()
{
    super();            // MUST call, usually as the first line

    Args args = element.args();

    // Get record from caller form
    if (args && args.record())
    {
        callerRecord  = args.record();
        callerTableId = args.dataset();
    }

    // Dynamic control access
    FormControl ctrl = this.design().controlname("ControlName");

    // Adding DataSource range
    QueryBuildRange range = this.dataSource(formDataSourceStr(FormName, TableName))
        .queryBuildDataSource()
        .addRange(fieldNum(TableName, Field));
    range.value("filter");
    range.status(RangeStatus::Locked);
}
```

### close - Form Closing

```x++
public void close()
{
    // operations before closing
    super();
}
```

### element keyword
`element` is a self-reference to the form. Usage examples:
- `element.args()` - parameters of the caller form
- `element.args().caller()` - get caller form as FormRun
- `element.args().record()` - selected record of the caller form
- `element.args().dataset()` - tableNum of the caller table
- `element.design().controlname("Name")` - find control by name
- `element.RefreshCaller()` - example custom method

---

## DataSource Methods

### SourceCode/DataSources - DataSource Method Definitions

```xml
<DataSources xmlns="">
    <DataSource>
        <Name>DataSourceName</Name>     <!-- Must be same as design DataSource name -->
        <Methods>
            <Method>
                <Name>methodName</Name>
                <Source><![CDATA[
    public void methodName()
    {
        super();
        // operation
    }

]]></Source>
            </Method>
        </Methods>
        <Fields />      <!-- Usually empty -->
    </DataSource>
</DataSources>
```

**Important:** The `<Fields />` tag must be written even if empty.

### Common DataSource Methods

**init - Adding Range and filter:**
```x++
public void init()
{
    super();
    QueryBuildRange range = this.queryBuildDataSource()
        .addRange(fieldNum(TableName, FieldName));
    range.value("ValueFilter");
    range.status(RangeStatus::Locked);
}
```

**executeQuery - Custom query execution:**
```x++
public void executeQuery()
{
    // Custom filter logic
    super();
}
```

**active - When selection changes:**
```x++
public int active()
{
    int ret = super();
    // UI update based on selected record
    return ret;
}
```

**validateWrite - Record validation:**
```x++
public boolean validateWrite()
{
    boolean ret = super();
    if (errorCondition)
    {
        ret = false;
        error("Error message");
    }
    return ret;
}
```

### DataSource API


---

## Control Methods

### DataControls - Control Method Definitions

```xml
<DataControls xmlns="">
    <Control>
        <Name>ControlName</Name>          <!-- Same as design control name -->
        <Type>MenuFunctionButton</Type>  <!-- Control type -->
        <Methods>
            <Method>
                <Name>clicked</Name>
                <Source><![CDATA[
    public void clicked()
    {
        super();
        // operation
    }

]]></Source>
            </Method>
        </Methods>
    </Control>
</DataControls>
```

### clicked - Button Click

```x++
public void clicked()
{
    super();                                    // Run MenuItem action
    TableName_ds.research(true);                 // Refresh
    element.CommandMethod();                      // Call form method
}
```

### modified - Value Change

```x++
public void modified()
{
    super();
    // Update another control
    LinkedControl.value(CalculatedValue);
}
```

### lookup - Custom Search

```x++
public void lookup()
{
    SysTableLookup sysTableLookup = SysTableLookup::newParameters(
        tableNum(LookupTable), this);

    sysTableLookup.addLookupfield(fieldNum(LookupTable, CodeField));
    sysTableLookup.addLookupfield(fieldNum(LookupTable, DescriptionField));

    sysTableLookup.performFormLookup();
}
```

---

## HeightMode / WidthMode

Properties used for control sizes:

| Value | Description | Usage |
|---|---|---|
| `SizeToAvailable` | Fill available space | Main containers, grid |
| `SizeToContent` | Resize to content | Button groups, small controls |
| `Auto` | System-defined size | Default, buttons |

**Example - Full-screen grid pattern:**
```xml
<AxFormControl xmlns="" i:type="AxFormGroupControl">
    <Name>MainGroup</Name>
    <HeightMode>SizeToAvailable</HeightMode>  <!-- Fill height -->
    <Type>Group</Type>
    <WidthMode>SizeToAvailable</WidthMode>    <!-- Fill width -->
    <FormControlExtension i:nil="true" />
    <Controls>
        <AxFormControl xmlns="" i:type="AxFormGridControl">
            <!-- Grid automatically fills the group -->
```

---

## MenuItemButton vs MenuFunctionButton

| Property | `AxFormMenuFunctionButtonControl` (MenuFunctionButton) | `AxFormCommandButtonControl` (CommandButton) |
|---|---|---|
| Purpose | Run operation linked to MenuItem | Form commands (New, Delete, OK, Cancel) |
| `<MenuItemName>` | Required | None |
| `<MenuItemType>` | Action / Display / Output / empty | None |
| `<Command>` | None | Required (New/DeleteRecord/OK/Cancel/Save/Revert) |
| `<Big>Yes` | Supports | Supports |
| `<NeedsRecord>` | Supports | - |
| `<MultiSelect>` | Supports | Supports |

**When to use which type?**
- Button opening a new form or running a class: `AxFormMenuFunctionButtonControl`
- Standard form action (add record, delete, save, cancel): `AxFormCommandButtonControl`
- Opening DropDialog popup: `AxFormDropDialogButtonControl`

---

## AutoDeclaration

`<AutoDeclaration>Yes</AutoDeclaration>` - makes the control accessible as a variable inside X++ code.

**When to set to `Yes`:**
- When access to the control from code is needed (if methods like `.value()`, `.visible()`, `.label()` will be called)
- When access from X++ to DataSource control is needed (ActionPanes)
- Controls used multiple times

**Examples:**
```xml
<!-- AutoDeclaration=Yes: Access from code needed -->
<Name>CorrectedOnsQty</Name>
<AutoDeclaration>Yes</AutoDeclaration>
<!-- X++: real value = CorrectedOnsQty.realValue(); -->

<!-- Without AutoDeclaration: UI only, no code access -->
<Name>FormGridControl_Description</Name>
<!-- No AutoDeclaration -->
```

---

## Design-Level Special Properties


---

## Canonical Property Order and Edit Discipline

This subsection documents the **canonical property order** observed in AxForm files. If this order is not followed during edits, the VS designer will automatically reorder when the file is reopened and create large diff churn. The deserializer is tolerant, but disciplined usage keeps the repo clean.

This order matches the VS designer's default output format.

### AxForm Root Child Order

| Order | Element | Required | Description |
|---|---|---|---|
| 1 | `<Name>` | Yes | Form name (same as file name) |
| 2 | `<SourceCode>` | Yes | Methods/DataSources/DataControls/Members sub-blocks |
| 3 | `<DataSourceQuery>` | No | Standard Query name bound to form (optional) |
| 4 | `<DataSources>` | Yes | Data sources (can be empty but tag must exist) |
| 5 | `<Design>` | Yes | UI design (Caption + Controls) |
| 6 | `<Parts>` | No | Usually empty `<Parts />` |

### AxFormControl Property Order (Canonical)

Inner property order inside each `<AxFormControl xmlns="" i:type="AxForm...">`:

| Order | Property | Type | Description |
|---|---|---|---|
| 1 | `<Name>` | All controls | Unique control name (REQUIRED) |
| 2 | `<AutoDeclaration>` | All | If "Yes", accessible from X++ code |
| 3 | `<AllowEdit>` | Data control | If "No", readonly |
| 4 | `<ElementPosition>` | All | VS automatic positioning (integer) |
| 5 | `<FilterExpression>` | All | Usually `%1` |
| 6 | `<HeightMode>` | All | Auto / SizeToAvailable / SizeToContent / Manual |
| 7 | `<NeededPermission>` | All | Read / Update / Delete / etc. |
| 8 | `<Type>` | All | Text matching `i:type` (REQUIRED) |
| 9 | `<VerticalSpacing>` | All | Usually `-1` (default) |
| 10 | `<WidthMode>` | All | Auto / SizeToAvailable |
| 11 | `<FormControlExtension i:nil="true" />` | All | REQUIRED - empty placeholder |
| 12 | `<DataField>` | Data control | Field name in table |
| 13 | `<DataMethod>` | Data control | Display method reference (`Class::method`) |
| 14 | `<DataSource>` | Data control | Form datasource name |
| 15 | `<Label>` | All | Label key (`@LabelFile:LabelKey`) |
| 16 | Type-specific | - | NoOfDecimals, Items, MenuItemName, etc. |
| 17 | `<Controls>` | Container | Child controls inside Group/Grid/TabPage |

**Important:** `<FormControlExtension i:nil="true" />` always comes BEFORE type-specific properties. Not AFTER DataField and DataSource.

### AxFormDataSource Property Order

| Order | Property | Description |
|---|---|---|
| 1 | `<Name>` | Datasource alias in form (REQUIRED) |
| 2 | `<Table>` | Physical table name (REQUIRED) |
| 3 | `<Fields>` | List of fields to display |
| 4 | `<ReferencedDataSources />` | Usually empty |
| 5 | `<JoinSource>` | Parent datasource name (optional) |
| 6 | `<LinkType>` | OuterJoin / InnerJoin / Passive / Delayed (optional) |
| 7 | `<AllowCreate>` | Yes / No |
| 8 | `<AllowDelete>` | Yes / No |
| 9 | `<AllowEdit>` | Yes / No |
| 10 | `<InsertAtEnd>` | Yes / No |
| 11 | `<InsertIfEmpty>` | Yes / No (usually "No") |
| 12 | `<DataSourceLinks />` | Usually empty |
| 13 | `<DerivedDataSources />` | Usually empty |

### AxFormControl Type-Specific Required Additions

Certain control types have one or more **type-specific required** properties. If skipped, compilation or rendering breaks.

| i:type | Required addition | Description |
|---|---|---|
| AxFormComboBoxControl | `<Items />` | Must exist even if empty |
| AxFormGridControl | `<Controls>` | Carries the columns inside |
| AxFormGroupControl | `<Controls>` | Carries the fields inside |
| AxFormTabPageControl | `<Controls>`, `<Caption>` | Content and tab title |
| AxFormMenuFunctionButtonControl | `<MenuItemName>`, `<MenuItemType>` | Menu item reference |
| AxFormCommandButtonControl | `<Command>` | New/DeleteRecord/OK/Cancel/Save/Revert |
| AxFormStringControl (display) | `<DataMethod>` | Method reference (instead of DataField) |
| AxFormStringControl (multi-line) | `<MultiLine>Yes</MultiLine>` | Together with `<DisplayHeight>` |
| AxFormRealControl | `<NoOfDecimals>` | Optional but common, -1 default |

### xmlns="" Requirement Matrix

Where the `xmlns=""` attribute belongs in form XML is the source of most deserialization errors. Places where it must not be skipped:

| Location | xmlns="" present? | Description |
|---|---|---|
| `<Methods>` inside `<SourceCode>` | YES | `<Methods xmlns="">` |
| `<DataSources>` inside `<SourceCode>` | YES | `<DataSources xmlns="">` |
| `<DataControls>` inside `<SourceCode>` | YES | `<DataControls xmlns="">` |
| `<Members>` inside `<SourceCode>` | YES | `<Members xmlns="" />` |
| Root `<DataSources>` (outside Design) | NO | Root tag is plain |
| Each `<AxFormDataSource>` inside root `<DataSources>` | YES | `<AxFormDataSource xmlns="">` |
| `<Caption>`, `<Controls>` inside `<Design>` | YES | `xmlns=""` in each |
| `<Pattern>`, `<DataSource>` etc. properties inside `<Design>` | YES | `xmlns=""` in each property |
| The root `<Design>` tag itself | NO | Plain |
| Each `<AxFormControl>` inside `<Controls>` | YES | `xmlns=""` + `i:type` together |
| Nested `<Controls>` (inside Group/Grid/TabPage) | NO | Inner Controls tag itself does not have it |
| Each `<AxFormControl>` inside nested | YES | Every control has `xmlns=""` |

**One-line rule:** Outside of `<AxForm...>` and `<Design>`, **every data-carrying sub-element** (Methods, DataSources, DataControls, Members, Caption, AxFormControl, AxFormDataSource, Pattern, DataSource property) starts with `xmlns=""`. Container tags (Controls, Fields, etc.) are plain.

### Self-Closing vs Open-Close Tag Discipline

Empty list-type elements **must be written as self-closing**. The open-close form (`<X></X>`) does not cause deserialization errors but the VS designer reorders and creates diff churn.

| Element | Empty form | Filled form |
|---|---|---|
| `<Items />` (ComboBox) | `<Items />` | `<Items><AxFormComboBoxItem>...</AxFormComboBoxItem></Items>` |
| `<ReferencedDataSources />` | `<ReferencedDataSources />` | (rarely filled) |
| `<DataSourceLinks />` | `<DataSourceLinks />` | `<DataSourceLinks><AxFormDataSourceLink>...</AxFormDataSourceLink></DataSourceLinks>` |
| `<DerivedDataSources />` | `<DerivedDataSources />` | (rarely filled) |
| `<Parts />` | `<Parts />` | (rarely filled) |
| `<Members />` (SourceCode) | `<Members xmlns="" />` | (rarely filled) |
| `<Controls>` inside Group/Grid | `<Controls />` | `<Controls><AxFormControl xmlns="">...</AxFormControl></Controls>` |

### Indentation: TAB Character Required

AxForm and AxFormExtension files are indented with **TAB character** (1 TAB = 1 level). For X++ code inside CDATA blocks, 4 spaces are used.

**Wrong:** Space (4-space) indentation or space + tab mixture → VS designer reorders, blows up the diff.

**Correct:**
```
<?xml version="1.0" encoding="utf-8"?>
<AxForm xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="Microsoft.Dynamics.AX.Metadata.V6">
	<Name>FormName</Name>
	<SourceCode>
		<Methods xmlns="">
			<Method>
				<Name>methodName</Name>
				<Source><![CDATA[
    public void methodName()
    {
        // 4-space space inside CDATA
    }

]]></Source>
			</Method>
		</Methods>
	</SourceCode>
	...
</AxForm>
```

### Edit Discipline: Why Should Property Order Be Preserved?

The D365 metadata XML deserializer is tolerant about property order; an XML written in the wrong order does not cause errors. However, the VS designer (Visual Studio 17) automatically reorders when the file is opened and closed:
- Properties are rewritten according to canonical order
- Indentation is normalized to TAB
- Empty elements are converted to self-closing

Result: An edit written incorrectly by AI or by hand will produce a **huge diff** once opened in VS. This diff churn:
- Makes code review difficult (the intended change gets mixed with other reorder changes)
- Increases the chance of merge conflicts
- Pollutes the repository history

For this reason, every edit must be done with canonical order and format.

---

# AxFormExtension - Form Extensions

## Namespace

```xml
<AxFormExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
                 xmlns="Microsoft.Dynamics.AX.Metadata.V6">
```

Both namespaces are the same as in `AxForm`.

## XML Structure - General Template


---

## PositionType Options

| PositionType | Description |
|---|---|
| (absent) | Append at the end (last child of parent) |
| `AfterItem` | After a specific sibling control - used together with `<PreviousSibling>` |
| `Begin` | Insert as the first child of the parent |

**Usage pattern:**
```xml
<!-- Append at end -->
<Parent>GridName</Parent>
<!-- No PositionType -->

<!-- Insert after a specific element -->
<Parent>GridName</Parent>
<PositionType>AfterItem</PositionType>
<PreviousSibling>ExistingControlName</PreviousSibling>

<!-- Insert at the beginning -->
<Parent>TabPageName</Parent>
<PositionType>Begin</PositionType>
```

---

## DataMethod Reference

`<DataMethod>` is used to display a display method in a field.

**Format:** `ExtensionClassName.methodName`


**Rules:**
- If `<DataMethod>` is present, `<DataField>` MUST NOT be present
- `<DataSource>` is still required
- `AutoDeclaration>Yes` is recommended (for code access)
- Readonly control (user cannot edit, only displays)

---

## AxFormExtension - CRITICAL RULES

### 2. AxFormExtensionControl Name Must Be Unique
The value inside `<Name>` must be unique across the entire project. Typical format:
- `FormExtensionControl` + random suffix (example: `FormExtensionControlggplnyst1`)
- Or meaningful name (example: `Copyhxnkna4q1`, `Copyadnpcu1x1`)

### 3. xmlns="" Inside FormControl
The `<FormControl>` tag inside `<AxFormExtensionControl>` must start with `xmlns=""`:
```xml
<FormControl xmlns="" i:type="AxFormGroupControl">
```

### 4. Sub-Controls Structure
The child `<AxFormControl>` elements inside `<FormControl>` again use `xmlns=""` - exactly like in normal AxForm.

### 5. ControlModifications
To change properties of an existing control:
```xml
<ControlModifications>
    <AxExtensionModification xmlns="">
        <Name>ExistingControlName</Name>
        <PropertyModifications>
            <!-- Properties to change -->
        </PropertyModifications>
    </AxExtensionModification>
</ControlModifications>
```

### 6. Required Closing Tags
The following tags MUST exist even if empty:
```xml
<ControlModifications />
<Controls />              <!-- or <Controls>...</Controls> -->
<DataSourceModifications />
<DataSourceReferences />
<DataSources />
<Parts />
<PropertyModifications />
```

---

## AxFormExtension vs AxForm - DataSources Block Difference

| Criterion | AxForm | AxFormExtension |
|---|---|---|
| `<SourceCode>` block | EXISTS - form methods | NONE |
| `<DataSources>` (design-level) | EXISTS - table bindings | `<DataSources />` (empty, usually) |
| DataSource method definitions | inside `<SourceCode><DataSources>` | None (CoC class is used) |
| Control method definitions | inside `<SourceCode><DataControls>` | None (CoC class is used) |
| Adding a new DataSource | defined in `<DataSources>` | can be defined in `<DataSources>` |

**How are methods defined in Extension?**
Direct method definition is not allowed in FormExtension. Instead, an extension class (CoC) is used:

```x++
[ExtensionOf(formStr(PurchTable))]
final class PurchTable_MyModel_Extension
{
    // Extending form method
    public void init()
    {
        next init();
        // additional operation
    }
}
```

---

## Edit Scenarios: ControlModifications vs Controls vs DataSources Decision Matrix

There are **three basic edit categories** in an AxFormExtension file. Which block to use in which category, which pattern to apply:

| Scenario | Block to use | XML pattern |
|---|---|---|
| Change existing control property | `<ControlModifications>` | `AxExtensionModification` + `PropertyModifications` + `AxPropertyModification` |
| Add new control | `<Controls>` | `AxFormExtensionControl` + `FormControl` + `Parent` + `PositionType` |
| Add new datasource (join) | `<DataSources>` | `AxFormDataSource` + `JoinSource` + `LinkType` |

**Important distinction:** "Existing control" refers to a control that already exists in the standard Microsoft form. If you want to add a "new control", you place an `AxFormExtensionControl` in the `<Controls>` block; but if you want to change the property of an existing control, you use the `<ControlModifications>` block. Mixing them up is a common mistake: AI tries to change a property by adding a new `AxFormControl` and writing the existing control name in the `Name` field — this does not change the original control, instead it tries to create a **new** control with the same name and does not work.

### AxExtensionModification Full XML Template

```xml
<ControlModifications>
	<AxExtensionModification xmlns="">
		<Name>ControlName</Name>
		<PropertyModifications>
			<AxPropertyModification>
				<Name>PropertyName</Name>
				<Value>NewValue</Value>
			</AxPropertyModification>
		</PropertyModifications>
	</AxExtensionModification>
</ControlModifications>
```

Real example (hiding an existing grid control):
```xml
<ControlModifications>
	<AxExtensionModification xmlns="">
		<Name>groupInterCompany_Grid</Name>
		<PropertyModifications>
			<AxPropertyModification>
				<Name>Visible</Name>
				<Value>No</Value>
			</AxPropertyModification>
		</PropertyModifications>
	</AxExtensionModification>
</ControlModifications>
```

**Critical rules:**
1. `xmlns=""` is REQUIRED in `<AxExtensionModification>`. xmlns is NOT in `<AxPropertyModification>`.
2. `<Name>` (the one outside Modification) — WRITES the control name in the existing form. If not in the standard form, the modification has no effect.
3. Multiple `AxPropertyModification` can be added to one `AxExtensionModification` block (to change multiple properties of the same control).
4. To change multiple controls, add multiple `AxExtensionModification` (in parallel).

### Common Values for PropertyName

| PropertyName | Value type | What for |
|---|---|---|
| `Visible` | Yes / No | Hide control |
| `AllowEdit` | Yes / No | Make readonly |
| `NoOfDecimals` | int (`-1` = automatic) | Real control decimal places |
| `MinNoOfDecimals` | int | Real control min places |
| `Mandatory` | Yes / No | Required field marker |
| `Label` | Label key (`@LabelFile:LabelKey`) | Change label |
| `Skip` | Yes / No | Skip in tab order |
| `AutoDeclaration` | Yes / No | Access from X++ code |
| `ExtendedDataType` | EDT name | Change type |
| `Width` / `Height` | Pixel int | Size |

### Blocks Usually Left Empty

In practice, in most AxFormExtension files, the following blocks appear EMPTY (`<X />`):

```xml
<DataSourceModifications />
<DataSourceReferences />
<Parts />
<PropertyModifications />
```

**Explanations:**
- `<DataSourceModifications>`: Used to change the property (AllowCreate/Delete/Edit) of an existing datasource. The Microsoft pattern supports it but rarely needed. If you need to change a datasource property, use the following template:
  ```xml
  <DataSourceModifications>
  	<AxFormDataSourceModification xmlns="">
  		<DataSourceName>ExistingDsName</DataSourceName>
  		<AllowCreate>No</AllowCreate>
  	</AxFormDataSourceModification>
  </DataSourceModifications>
  ```
- `<DataSourceReferences>`, `<Parts>`, `<PropertyModifications>`: No standard usage; always empty.

**Anti-pattern:** Trying to add a `<SourceCode>` or `<Methods>` block to AxFormExtension. The AxFormExtension file cannot hold methods. If a form-level method is needed, a static handler is added to an existing compiled class with `[FormControlEventHandler]` / `[FormDataSourceEventHandler]`.

### AxFormExtensionControl Name Suffix Convention

When adding a new control to a form extension, VS automatically generates a unique `Name`:

| Pattern | Description | Example |
|---|---|---|
| `FormExtensionControl[8-10char][digit]` | When a new control is created | `FormExtensionControlwk3sueh31` |
| `Copy[8-10char][digit]` | When created by copy-paste from a control | `Copymv4zj5ni1` |

When written manually, you do not have to follow this pattern but:
1. **Uniqueness** is required — two `AxFormExtensionControl` in the same form extension must not have the same `Name`.
2. **Inner FormControl's `Name`** (i.e., `<FormControl><Name>...</Name></FormControl>`) is used at form runtime; the `Name` outside `AxFormExtensionControl` is only the metadata ID.

### Where Edit Patterns Are Most Often Confused

| Error | Symptom | Correct pattern |
|---|---|---|
| Adding a new AxFormControl to change existing control property | Property doesn't change, two controls with same name are created and collide at runtime | Use the `<ControlModifications>` block |
| Adding a method to AxFormExtension | XML is rejected or the block is removed | Add to existing class with static event handler |
| Writing `<AxFormControl>` inside `<ControlModifications>` | Wrong type — deserialization error | Use `<AxExtensionModification>` |
| Writing `<AxFormControl>` directly inside `<Controls>` (without wrapper) | No Parent/PositionType, the control remains independent | Wrap with `<AxFormExtensionControl>` |
| PositionType=AfterItem but no PreviousSibling | Control cannot be placed, gap appears in rendering | PreviousSibling is REQUIRED |

---

# AxQuery, AxQuerySimpleExtension, AxMenuItem and AxReport - Reference Documentation


---

## TABLE OF CONTENTS

1. [AxQuery - Query Definitions](#1-axquery---query-definitions)
2. [AxQuerySimpleExtension - Query Extensions](#2-axquerysimpleextension---query-extensions)
3. [AxMenuItemAction - Action Menu Items](#3-axmenuitemaction---action-menu-items)
4. [AxMenuItemDisplay - Display Menu Items](#4-axmenuitemdisplay---display-menu-items)
5. [AxMenuItemOutput - Output Menu Items](#5-axmenuitemoutput---output-menu-items)
6. [AxReport - SSRS Report Definitions](#6-axreport---ssrs-report-definitions)

---

# 1. AxQuery - Query Definitions

## 1.1 Basic XML Structure and Namespace

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxQuery xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns=""
	i:type="AxQuerySimple">
```

**Critical Notes:**
- The root element has both `xmlns:i="http://www.w3.org/2001/XMLSchema-instance"` and `xmlns=""` (empty namespace)
- `i:type="AxQuerySimple"` is required — this is the only supported type
- All child elements inside inherit the default namespace (no additional namespace needed)

## 1.2 classDeclaration and [Query] Attribute

The `classDeclaration` method is required in every AxQuery file:


**Why is the `[Query]` attribute required?**
- Without this attribute, the D365 FO metadata compiler does not recognize the class as a Query object
- `extends QueryRun` — query execution framework
- The class name must match the file name and `<Name>` value exactly
- The class body is always empty — all logic is defined by metadata

## 1.3 AxQuerySimpleRootDataSource — Root Data Source

Every query has exactly one `AxQuerySimpleRootDataSource` and this is the main table.

```xml
<DataSources>
    <AxQuerySimpleRootDataSource>
        <Name>VendPackingSlipTrans</Name>       <!-- Access name in code -->
        <DynamicFields>Yes</DynamicFields>       <!-- Should all fields be included? -->
        <Table>VendPackingSlipTrans</Table>      <!-- Actual table name -->
        <DataSources>
            <!-- Embedded (JOIN) data sources go here -->
        </DataSources>
        <DerivedDataSources />
        <Fields />                               <!-- If DynamicFields=No, specific fields -->
        <Ranges />
        <GroupBy />
        <Having />
        <OrderBy />
    </AxQuerySimpleRootDataSource>
</DataSources>
```

**Important:** `<Name>` and `<Table>` can be different. For example:
- `<Name>InventTransPurch</Name>` + `<Table>InventTrans</Table>` → to be able to JOIN the same table twice with different names

## 1.4 DynamicFields: Yes vs No Difference

| DynamicFields | Behavior | When Used |
|---|---|---|
| `Yes` | All fields of the table are automatically included | When all fields are needed or for contexts like SSRS |
| `No` (or when `<DynamicFields>` is not specified) | Only fields listed in the `<Fields>` section are included | Performance optimization; when only specific fields are needed |

## 1.5 AxQuerySimpleEmbeddedDataSource — JOIN Definitions

Sub-tables (JOIN) are defined with `AxQuerySimpleEmbeddedDataSource` and nested in the `<DataSources>` block of the parent data source:

```xml
<AxQuerySimpleEmbeddedDataSource>
    <Name>EcoResProductTranslation</Name>
    <DynamicFields>Yes</DynamicFields>
    <Table>EcoResProductTranslation</Table>
    <DataSources />               <!-- This data source's own sub-JOINs -->
    <DerivedDataSources />
    <Fields />                    <!-- If DynamicFields=No, specific fields -->
    <Ranges>
        <AxQuerySimpleDataSourceRange>
            <Name>LanguageId</Name>
            <Field>LanguageId</Field>
            <Status>Locked</Status>
            <Value>tr</Value>
        </AxQuerySimpleDataSourceRange>
    </Ranges>
    <JoinMode>OuterJoin</JoinMode>        <!-- If not specified, InnerJoin is the default -->
    <UseRelations>Yes</UseRelations>      <!-- Either UseRelations or Relations -->
    <Relations>
        <AxQuerySimpleDataSourceRelation>
            <Name>QueryDataSourceRelation1</Name>
            <Field>Product</Field>               <!-- Field in this table -->
            <JoinDataSource>InventTable</JoinDataSource>  <!-- Connected source name -->
            <RelatedField>RecId</RelatedField>   <!-- Field in connected table -->
        </AxQuerySimpleDataSourceRelation>
    </Relations>
</AxQuerySimpleEmbeddedDataSource>
```

## 1.6 JoinMode Options

All usages in the real project:

|---|---|---|
| `InnerJoin` | Match required on both sides (default, can be omitted) | Most standard relations |
| `OuterJoin` | Left side always returns; if no match on right, NULL | `PurchTable` (if row exists without invoice), `ProdBOM` |
| `ExistsJoin` | If match exists, the left record is returned; right side fields cannot be used | `InventDimSales` — existence check only |

**Critical:** If `JoinMode` is not specified, **InnerJoin** is applied. Always explicitly specifying it is best practice.

## 1.7 UseRelations vs Relations Difference

| Approach | Description | When |
|---|---|---|
| `<UseRelations>Yes</UseRelations>` | The table relations defined in D365 metadata are used automatically; `<Relations />` is left empty | When the standard D365 tables already have a defined relation |

**Real Examples:**
- `SalesTable + SalesLine`: `<UseRelations>Yes</UseRelations>` — standard D365 relation exists
- `InventTable + EcoResProductTranslation` (with LanguageId=tr filter): `<Relations>` defined manually — because of language filter, special join
- `InventDimSales + InventDimPurch`: 3 separate `AxQuerySimpleDataSourceRelation` cross-join — UseRelations not possible

## 1.8 AxQuerySimpleDataSourceRelation — Relation Definition

```xml
<AxQuerySimpleDataSourceRelation>
    <Name>QueryDataSourceRelation1</Name>        <!-- Unique name, convention: QueryDataSourceRelation1,2,3... -->
    <Field>inventDimID</Field>                   <!-- Field in this data source -->
    <JoinDataSource>InventTransPurch</JoinDataSource>  <!-- <Name> value of joined data source -->
    <RelatedField>inventDimID</RelatedField>     <!-- Field in joined data source -->
</AxQuerySimpleDataSourceRelation>
```

**Multi-field JOIN — multiple relations:**
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

## 1.9 Range (Filter) Definitions

```xml
<Ranges>
    <AxQuerySimpleDataSourceRange>
        <Name>LanguageId</Name>          <!-- Unique name -->
        <Field>LanguageId</Field>         <!-- Field to filter -->
        <Status>Locked</Status>           <!-- User cannot change this filter -->
        <Value>tr</Value>                 <!-- Filter value -->
    </AxQuerySimpleDataSourceRange>
</Ranges>
```

### Range Status Values

| Status | Meaning |
|---|---|
| `Locked` | Filter is fixed, user cannot change, hidden in UI |
| `Hidden` | Filter is active but hidden in UI; user cannot access but can be changed at runtime |
| (not specified) | Default — filter is open, user can change |

## 1.10 Fields — Specific Field Selection

When specific fields are selected instead of `DynamicFields>Yes`:

```xml
<Fields>
    <AxQuerySimpleDataSourceField>
        <Name>InventTransId</Name>       <!-- Code access name of field -->
        <Field>InventTransId</Field>      <!-- Real field name in table -->
    </AxQuerySimpleDataSourceField>
    <AxQuerySimpleDataSourceField>
        <Name>ItemId</Name>
        <Field>ItemId</Field>
    </AxQuerySimpleDataSourceField>
</Fields>
```

## 1.11 GroupBy, Having, OrderBy

Appears empty in the root data source — empty element is enough if not used:

```xml
<GroupBy />
<Having />
<OrderBy />
```


# 2. AxQuerySimpleExtension - Query Extensions

## 2.1 Basic XML Structure


**Namespace:** Only `xmlns:i="http://www.w3.org/2001/XMLSchema-instance"` — NO `xmlns=""`


## 2.2 Adding a Field to Existing Data Source

In the `<Fields>` section with `AxQueryExtensionQueryDataSourceField`:


## 2.3 Adding a New Data Source to Existing Data Source

In the `<DataSources>` section with `AxQueryExtensionEmbeddedDataSource`:


## 2.5 DerivedTable Difference Inside Extension

While adding data source in the `<Fields>` block inside `AxQuerySimpleExtension`, fields use `AxQuerySimpleDataSourceField` (same structure as in AxQuery) but `<DerivedTable>` can be added. This explains which real table the field comes from (especially in cases where alias is used).

---

# 3. AxMenuItemAction - Action Menu Items

## 3.1 Basic XML Structure and Namespace

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxMenuItemAction xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="Microsoft.Dynamics.AX.Metadata.V1">
```

**Critical:** Namespace is `Microsoft.Dynamics.AX.Metadata.V1` — all MenuItem types use this namespace (not V6 or V2).

## 3.2 Element Reference

| Element | Required | Description |
|---|---|---|
| `<Name>` | Yes | Object name — must match file name |
| `<Label>` | Yes | Text displayed in UI — label key or direct text |
| `<Object>` | Yes | Name of the class to execute |
| `<ObjectType>` | Yes | `Class` (almost always) |
| `<SubscriberAccessLevel>` | No | Security access level |
| `<NeedsRecord>` | No | If `Yes`, button is disabled unless an active record is selected |
| `<MultiSelect>` | No | If `Yes`, can be executed by selecting multiple records |
| `<ConfigurationKey>` | No | When which configuration key is enabled, it appears |
| `<MaintainUserLicense>` | No | `Enterprise` — license required to modify |
| `<ViewUserLicense>` | No | `Universal` — license required to view |
| `<HelpText>` | No | Tooltip text |

## 3.3 ObjectType Options


```xml
<ObjectType>Class</ObjectType>
```

## 3.4 SubscriberAccessLevel Structure

```xml
<SubscriberAccessLevel>
    <Read xmlns="">Allow</Read>
</SubscriberAccessLevel>
```

**Important:** `<Read xmlns="">` — `xmlns=""` is on the child element, not on the parent. This is a structure required by the V1 namespace.

### Grant Options

| Grant Type | Meaning |
|---|---|
| `<Update xmlns="">Allow</Update>` | Update access |
| `<Create xmlns="">Allow</Create>` | Create access |
| `<Delete xmlns="">Allow</Delete>` | Delete access |
| `<Invoke xmlns="">Allow</Invoke>` | Class/method invoke access — for Action/Output |

Multiple grants can be used together.


## 3.6 ConfigurationKey Usage

```xml
<ConfigurationKey>Prod</ConfigurationKey>    <!-- When production module is enabled -->
<ConfigurationKey>Currency</ConfigurationKey> <!-- When currency feature is enabled -->
```

Configuration key determines which D365 module/feature must be active for the menu item to be visible.

## 3.7 NeedsRecord and MultiSelect

```xml
<!-- Active only when a single record is selected -->
<NeedsRecord>Yes</NeedsRecord>

<!-- Can be executed by selecting multiple records -->
<MultiSelect>Yes</MultiSelect>
<NeedsRecord>Yes</NeedsRecord>
```

The `NeedsRecord>Yes` + `MultiSelect>Yes` combination allows multiple rows to be selected in the form's grid and processed in bulk.

# 4. AxMenuItemDisplay - Display Menu Items

## 4.1 Basic Structure


**Critical Difference:** Unlike `AxMenuItemAction`, the `<ObjectType>` element is NOT WRITTEN. Display item always opens a form.

## 4.2 AxMenuItemDisplay-Specific Additional Elements

| Element | Description | Example |
|---|---|---|
| `<OpenMode>` | The mode in which the form will be opened | `Edit` — in edit mode |
| `<NeedsRecord>` | Active record requirement | `Yes` |
| `<HelpText>` | Tooltip | `SQL Execute form` |
| `<DisabledResource>` | Disabled icon resource ID | `0` |
| `<NormalResource>` | Normal icon resource ID | `0` |

# 5. AxMenuItemOutput - Output Menu Items

## 5.1 Basic Structure


**Difference from Action:** The structure is almost identical, but `<ObjectType>Class</ObjectType>` is required, and `<Object>` points to a **Controller** class.

## 5.2 Connection with Controller Class

Output MenuItem → Controller Class → Report name chain:


# 6. AxReport - SSRS Report Definitions

## 6.1 Namespace and Basic Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxReport xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
    xmlns="Microsoft.Dynamics.AX.Metadata.V2">
```

**Critical:** Namespace is `Microsoft.Dynamics.AX.Metadata.V2` — not V1 (MenuItem) and not V6 (Form).

## 6.3 DataSets Structure


**DataSourceType Values:**

| Value | Description |
|---|---|
| `ReportDataProvider` | Uses DP class — the most common pattern |
| `Query` | Uses AX Query directly |
| `AXDataProvider` | Reads directly from table/view |

## 6.4 Query Format — DP Class and TmpTable Connection


**Format:** `SELECT * FROM [DPClassName].[TmpTableName]`
- `DPClassName` → name of the DP class (like classStr reference)
- `TmpTableName` → TempDB table name processed by the DP class

## 6.5 Alias Format — DataSet Fields


**Alias Format Analysis:**

| Format | Description |
|---|---|
| `TmpTable.1.FieldName` | Standard field alias — `.1.` is a fixed index/version number |
| `TmpTable.1.EnumField:LABEL(...)` | Returns the label (displayed) value of the enum field |
| `TmpTable.1.EnumField:NAME(...)` | Returns the code name of the enum field |

**Critical:** The `.1.` number does not change — `1` is always used. This is part of the D365 SSRS data binding protocol.

## 6.7 Standard AX Parameters (Present in Every Report)

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

These 6 parameters are required for the D365 SSRS infrastructure to function and are present in every ReportDataProvider report.

## 6.8 DefaultParameterGroup — System Parameters Definition

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

`<UserVisibility>Hidden</UserVisibility>` — these parameters are not shown to the user; they are automatically filled by the system.

## 6.9 Designs — AxReportPrecisionDesign

```xml
<Designs>
    <AxReportDesign xmlns=""
        i:type="AxReportPrecisionDesign">
        <Name>Report</Name>
        <DataNavigation>DocumentMap</DataNavigation>   <!-- Optional -->
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

**Text Field Rules:**
- All content is HTML-encoded XML: `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`
- Contains the full report definition in SSRS RDL format
- `xmlns="http://schemas.microsoft.com/sqlserver/reporting/2016/01/reportdefinition"` — SQL Server 2016 RDL schema
- Created with Visual Studio SSRS Report Designer and converted to D365 metadata format

**Design Name:** Always `<Name>Report</Name>` — referenced in Controller class with `ssrsReportStr(ReportName, Report)`.

**DataNavigation:** `DocumentMap` — provides document map navigation within the report.

## 6.10 DataSource Structure Inside RDL

DataSource inside HTML-encoded text:
```
&lt;DataSource Name="AutoGen__ReportDataProvider"&gt;
    &lt;DataProvider&gt;AXREPORTDATAPROVIDER&lt;/DataProvider&gt;
&lt;/DataSource&gt;
```

This is a fixed D365 SSRS data provider definition.

## 6.11 When DataSets Is Empty (Only Design)


```xml
<DataSets />
<DefaultParameterGroup>
    <Name xmlns="">Parameters</Name>
    <ReportParameterBases xmlns="" />
</DefaultParameterGroup>
<Designs>
    <AxReportDesign xmlns="" i:type="AxReportPrecisionDesign">
        <Name>Report</Name>
        <Text>...RDL content...</Text>
    </AxReportDesign>
</Designs>
```

This situation is seen when the report design is kept separately and the DataSet is defined in another report file (two reports with the same name: `CurrecyPayment` and `CurrencyPaymentReport`).

## 6.13 Field Headers with Caption


Caption is optional. When provided, it can be automatically used as a column header in the report designer.

---

## APPENDICES

### APPENDIX A: Comparison Table by MenuItem Types

| Property | AxMenuItemDisplay | AxMenuItemAction | AxMenuItemOutput |
|---|---|---|---|
| Namespace | V1 | V1 | V1 |
| `<Object>` content | Form name | Class name | Controller class name |
| `<ObjectType>` | **NOT WRITTEN** | `Class` | `Class` |
| Purpose | Open a form | Run business logic | Run SSRS report |
| `<NeedsRecord>` | Can be used | Can be used | Rarely |
| `<MultiSelect>` | Rarely | Can be used | - |
| `<OpenMode>` | `Edit` / `View` | - | - |

### APPENDIX B: AxQuery File Naming

- File name and `<Name>` value must match exactly

### APPENDIX C: Adding a Field Without AxReport DataSet


`DisableAutoCreateInDataRegion>true` — prevents auto-creation of columns in the table region of the report designer. The field is defined but not placed in the RDL by default; only used for manual placement.

# AxSecurityPrivilege

## General Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxSecurityPrivilege xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>PrivilegeName</Name>
    <!-- Label optional -->
    <DataEntityPermissions>
        <AxSecurityDataEntityPermission>
            <Grant>
                <Correct>Allow</Correct>
                <Create>Allow</Create>
                <Delete>Allow</Delete>
                <Read>Allow</Read>
                <Update>Allow</Update>
            </Grant>
            <Name>EntityName</Name>
            <Fields />
            <Methods />
        </AxSecurityDataEntityPermission>
    </DataEntityPermissions>
    <DirectAccessPermissions />
    <EntryPoints>
        <AxSecurityEntryPointReference>
            <Name>EntryPointRefName</Name>
            <Grant>
                <Read>Allow</Read>
            </Grant>
            <ObjectName>MenuItemName</ObjectName>
            <ObjectType>MenuItemAction</ObjectType>
            <Forms />
        </AxSecurityEntryPointReference>
    </EntryPoints>
    <FormControlOverrides />
</AxSecurityPrivilege>
```

### Rule: Each privilege contains either EntryPoints or DataEntityPermissions, both at once is rare
1. **Form/Action based**: EntryPoints filled, DataEntityPermissions empty → form or class access permission
2. **Data entity based**: DataEntityPermissions filled, EntryPoints empty → OData/Data Management access permission

---

### 7-24. Other DataEntity Privilege Examples (Summary)

All privileges below follow the same pattern. **Maintain** = full access (Correct+Create+Delete+Read+Update), **View** = only Read:

| Privilege Name | Entity Name | Access Type |
|---|---|---|

---

## EntryPoint ObjectType Values

| ObjectType | Usage | ObjectName |
|---|---|---|
| `MenuItemAction` | Menu item that runs a class | Class-based menuitem name |
| `MenuItemDisplay` | Menu item that opens a form | Form-based menuitem name |
| `MenuItemOutput` | Menu item that runs a report | Controller-based menuitem name |

---

## Grant Options

### EntryPoint Grant Values

| Grant Field | Allow | Deny | Unset |
|---|---|---|---|
| `Read` | Read permission granted | Read blocked | Inherited from upper level |
| `Update` | Update permission | Blocked | Inherited |
| `Create` | Create permission | Blocked | Inherited |
| `Delete` | Delete permission | Blocked | Inherited |
| `Correct` | Correct permission (for financial tables) | Blocked | Inherited |
| `Invoke` | Invoke permission | Blocked | Inherited |

- For `MenuItemAction`, only `Read: Allow` → permission to run the class is enough
- For `MenuItemDisplay`, Maintain pattern → Correct+Create+Delete+Read+Update all Allow
- For `MenuItemDisplay`, View pattern → only Read: Allow

**Unset**: The `<Grant>` block is not written at all or the related element is omitted. In this case, access right is inherited from `Duty` or `Role` level.

### DataEntityPermission Grant Values

Same grant names are used. For entity, `Correct` usually represents special financial correction operations.

---

## DataEntityPermissions Structure

```xml
<DataEntityPermissions>
    <AxSecurityDataEntityPermission>
        <Grant>
            <Correct>Allow</Correct>   <!-- Financial correction -->
            <Create>Allow</Create>     <!-- Insert -->
            <Delete>Allow</Delete>     <!-- Delete -->
            <Read>Allow</Read>         <!-- Select -->
            <Update>Allow</Update>     <!-- Update -->
        </Grant>
        <Name>EntityName</Name>         <!-- AxDataEntityView name -->
        <Fields />                     <!-- Field-based restriction (optional) -->
        <Methods />                    <!-- Method-based restriction (optional) -->
    </AxSecurityDataEntityPermission>
</DataEntityPermissions>
```


---

## DirectAccessPermissions


---

## FormControlOverrides


---

# AxSecurityDuty

## Structure Analysis

- `<Name>`: Duty name — ends with suffix `Duty`
- `<Privileges>`: Contains one or more `<AxSecurityPrivilegeReference>`
- Each `<AxSecurityPrivilegeReference>` contains only `<Name>` (privilege name reference)
- Grant is not defined directly in Duty; grants are set at privilege level

## Multiple Privilege References


---

# AxSecurityRole

## SubRoles


---

# AxSecurityDutyExtension


## General Structure (For Reference)

Used to add a new privilege to an existing standard Duty:

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxSecurityDutyExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>StandardDutyName.ModelName</Name>
    <Privileges>
        <AxSecurityPrivilegeReference>
            <Name>NewPrivilegeName</Name>
        </AxSecurityPrivilegeReference>
    </Privileges>
</AxSecurityDutyExtension>
```

- `<Name>` is the same value as the file name without `.xml`

---

# AxSecurityRoleExtension


## General Structure (For Reference)

Used to add a new duty or privilege to an existing standard Role:

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxSecurityRoleExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>StandardRoleName.ModelName</Name>
    <Duties>
        <AxSecurityDutyReference>
            <Name>NewDutyName</Name>
        </AxSecurityDutyReference>
    </Duties>
    <Privileges>
        <AxSecurityPrivilegeReference>
            <Name>NewPrivilegeName</Name>
        </AxSecurityPrivilegeReference>
    </Privileges>
</AxSecurityRoleExtension>
```

---

# AxSecurityPolicy


## General Structure (For Reference)

In D365 F&O, Security Policy is the `Extensible Data Security (XDS)` mechanism.
It complements role-based security; instead of giving menu/form access, it restricts which records the user can see or which records they can operate on.

### Basic XDS Logic

- `PrimaryTable`: Main/root table of the security query
- `Query`: Query that defines the set of accessible records
- `ConstrainedTables`: Bound tables or views to which the filter will be applied
- `Context`: When the policy will take effect (`RoleName`, `RoleProperty`, application context)

The policy query is added to the SQL `WHERE` or `ON` condition in operations of the related constrained table.
If multiple policies are active at the same time, the result is intersection (`intersection`), not union (`union`).

## General Structure (Close to Real D365 FO Reference)


### ConstrainedTables Example

A policy can restrict not only the `PrimaryTable` but also bound tables and views:

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

### Commonly Used Fields

| Field | Description |
|---|---|
| `ConstrainedTable` | Indicates whether the policy is active on the primary table (`Yes/No`) |
| `Enabled` | Indicates whether the policy is enabled at runtime; appears in some standard examples |
| `PrimaryTable` | Main/root table of the policy query |
| `Query` | Query defining the record filter |
| `ContextType` | Most commonly `RoleName` or `RoleProperty`; determines when the policy is applied |
| `RoleName` | Makes the policy work for users assigned to a specific role |
| `ContextString` | Key value used in application context or `RoleProperty` scenarios |
| `UseNotExistJoin` | Causes the policy query to be applied with `not exists join` logic instead of `exists join` |
| `ConstrainedTables` | Defines table/view restrictions other than the primary table |

### Operation Values

| Operation | Description |
|---|---|
| `AllOperations` | All operations including select, insert, update, delete |
| `Insert` | Only insert |
| `Update` | Only update |
| `Delete` | Only delete |
| `InsertUpdateDelete` | Write operations |
| `(empty/omit)` | In standard examples, select-only policy is usually serialized this way |

### ContextType Scenarios

| ContextType | Description |
|---|---|
| `RoleName` | Policy is applied only for users having the specified role |
| `RoleProperty` | If the `ContextString` on the role matches the policy `ContextString`, it is applied |
| `ContextString` | In Microsoft Learn, used for application context; application code can trigger the policy by setting context with `XDS::SetContext(...)`. In standard metadata examples, in this scenario, often only the `ContextString` field is serialized |

### Points to Watch Out For

- XDS does not replace role-based security; it complements it with record filter.
- Poorly designed policy queries can have serious performance impact.
- If multiple policies are active at the same time, all policies are applied together.
- Users assigned the `XDSDataAccessPolicyBypassRole` bypass XDS filters.
- According to Microsoft documentation, XDS should not be used for financial dimensions.

---

# AxEventSubscription


## General Structure (For Reference)

Event subscriptions bind static event handler methods to table or class events. Usually defined inline with `[DataEventHandler]` and `[FormControlEventHandler]` attributes in X++ class files; separate XML file is rarely used.

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxEventSubscription xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>EventSubscriptionName</Name>
    <EventHandlerClass>HandlerClassName</EventHandlerClass>
    <EventHandlerMethod>HandlerMethodName</EventHandlerMethod>
    <EventPublisherClass>PublisherClass</EventPublisherClass>
    <EventPublisherMethod>PublisherMethod</EventPublisherMethod>
    <EventType>PostEvent</EventType>
</AxEventSubscription>
```

### EventHandlerType Values

| EventHandlerType | Description |
|---|---|
| `DataEventType::Inserting` | Before insert |
| `DataEventType::Inserted` | After insert |
| `DataEventType::Updating` | Before update |
| `DataEventType::Updated` | After update |
| `DataEventType::Deleting` | Before delete |
| `DataEventType::Deleted` | After delete |
| `DataEventType::ValidatedWrite` | After validateWrite |
| `DataEventType::ValidatedDelete` | After validateDelete |

# AxLabelFile

## File Structure

The label system consists of three file types:


## Label File Format (Plain Text .label.txt)

Label files are not XML but a special plain text format.

### Basic Format Rules

```
LabelId=Label text
 ;Description line (optional, starts with a space + semicolon)
```

**Rules**:
1. Each label is a line in `LabelId=TextValue` format
2. Comment line (description): starts with a single space + `;`, on the very next line
3. Comment line is optional — some labels are left without description
4. File encoding: UTF-8 (required for Turkish characters)
5. Label ID is case-sensitive
6. Empty lines do not appear in the file (lines are adjacent)
7. The last line ends with an empty line

---

## Label ID Formats


### Format 3: Direct Meaningful Name (No prefix)


---

## tr-TR and en-US Distinction

### Joint Inspection

| Label ID | en-US | tr-TR |
|---|---|---|
| `ProcessSuccessful` | `The operation has been completed!` | `İşlem tamamlandı!` |
| `ProcessCancelled` | `The operation could not be completed!` | `İşlem gerçekleştirilemedi!` |


### Labels present in en-US but missing in tr-TR


---

## Real Label Examples — Selections (en-US)


---

## Usage in Code


---

# General D365 FO Development Patterns

This section is generalized from the project documents under `docs/MdDocuments`.

## 1. Warehouse Mobile App (WMS) Development

### Standard work model

On the warehouse mobile app side, the typical flow proceeds in this order:

1. The request from the mobile device arrives at the AOS side in XML/container format.
2. The active menu item setup and the mode/process to run are resolved.
3. The relevant `WhsWorkExecuteDisplay*` class or `ProcessGuide` controller is selected.
4. Input from the previous screen is parsed.
5. Validation and business logic run.
6. The container for the next screen is built.
7. The result is sent back to the device.

### Basic Components

| Component | Role |
|---|---|
| `WhsWorkExecuteDisplay` | Basic base/helper class of legacy mobile flow |
| Derivatives of `WhsWorkExecuteDisplay...` | Manages screens for a specific mode or workflow |
| `WHSRFPassthrough` | Carries key/value session state between steps |
| `WHSWorkExecuteMode` | Determines which execution class to select |
| `...Controls` classes | Holds control name, pass key, and technical constants |
| `WhsControl` / `WHSField` | Used if a really new field/control type will be added |
| `ProcessGuideController` / `ProcessGuideStep` / `ProcessGuidePageBuilder` | Modern, modular warehouse mobile framework |

### Legacy `displayForm()` vs `ProcessGuide`

| Pattern | When suitable | Advantage | Caution |
|---|---|---|---|
| Legacy `displayForm()` + `WHSRFPassthrough` | When a small step or validation is to be injected into the existing standard flow | Provides fast CoC entry into the existing flow | State cleanup is difficult, higher regression risk |
| `ProcessGuide` | When a brand new, multi-step, long-lived mobile process will be developed | Step/page/action responsibilities separate | Initial setup cost is higher |

Practical rules:

- Before entering a legacy `WhsWorkExecuteDisplay*` class, check if there is a `SysObsolete` note at the declaration level.
- If `Use process guide = Yes` on the menu item side, try not to force new development into the legacy `displayForm()` side.
- If only an additional step or additional control will be added to an existing standard flow, legacy CoC is still a valid option.

### `displayForm()` CoC Pattern

In the legacy flow, the main entry point is usually the `displayForm(container _con, str _buttonClicked)` method.

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

Critical notes:

- The `next displayForm(...)` call must be preserved; custom logic must not fully bypass the standard flow.
- `WHSRFPassthrough` in most legacy flows comes from `conPeek(_con, 2)`.
- Control tuples are practically parsed starting from the 3rd element.
- The text input value is read from a fixed position inside the tuple in some legacy flows; do not arbitrarily change this framework contract.

### `WHSRFPassthrough` and State Management

`WHSRFPassthrough` is a key/value wrapper that carries state between RF mobile steps.

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

Important observations:

- In legacy flows, both `localPass` created inside the method and class-level `pass` can be used together.
- Flags like auto-confirm or final completion may need to be written to both passes.
- If previous step keys are not cleared on each state transition, the screen can fall into an infinite loop or wrong state.
- The `public static str` method pattern is practical and safe for reusing string constants inside the extension class.

### Building UI with `buildControl()`

WMS legacy UI is built by creating container tuples with `buildControl()`.

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

Rules:

- In legacy display extension classes, the `#WHSWorkExecuteControlElements`, `#WHSWorkExecuteDisplayCases`, and `#WHSRF` macros are usually required.
- On each form return, `updateModeStepPass()` must be called; otherwise mode, step, and pass information is left incomplete.
- If possible, use standard EDTs (`InventSerialId`, `InventBatchId`, `Qty`, `ItemId`) and standard control names; this makes field name/priority and icon-title matching easier.

### Custom work type framework

The WMS custom work type pattern usually consists of these components:

1. A unique custom work type code is defined in setup.
2. The order in the work template is usually `Pick -> Custom -> Put` or similar.
3. UI/state management is done on the `WhsWorkExecuteDisplay*` side.
4. Final completion validation is done with `WhsIWorkTypeCustomProcessor`.
5. The placeholder method expected by the framework is in the `WHSWorkCustomData` extension.

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

Rules:

- The code in setup, the processor factory attribute, and the custom data method the framework will call must be consistent with each other.
- `WhsIWorkTypeCustomProcessor` is for final validation; UI construction and persistence should not be piled into this class.
- If you have added a new factory attribute, control, or field type, be sure to check the `SysExtension` cache effect.

### Menu item setup, field names, and step titles

Before writing code, extract these setup axes:

- Menu item type: inquiry, indirect, work creation or existing work execution
- `Directed by`: user directed, system directed, cluster, cycle count, etc.
- `Use process guide`, `Cluster profile`, `Generate license plate`, `Show work line list`, `Display inventory status`, `Validated user directed field`
- Field names and priorities setup
- Step title / instruction override setup

Practical result:

- Some requests are actually not X++ changes but setup corrections.
- For a new field, sometimes only `buildControl()` is enough; sometimes `WhsControl` + `WHSField` definition is needed.
- Some UX improvements can be solved without code on the step title/instruction side.

### Location directive strategy pattern

For a new location directive strategy, at least these three components are needed:

1. `WHSLocDirStrategy` enum extension
2. X++ class derived from `WhsLocationDirectiveStrategy`
3. Aggregate `AxQuery` + `AxView` if needed

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

Commonly used rules at the query level:

- The root datasource is usually `WMSLocation`.
- In the empty location finding pattern, full locations are eliminated with `NoExistsJoin`.
- If `InventUseDimOfInventSumToggle` is on, `InventSum` is connected directly; otherwise, the `InventDim -> InventSum` chain is used.
- Locations with stock are excluded with `ClosedQty = No` and `PhysicalInvent > 0` filters.
- Open or in-progress incoming `WHSWorkLine` records must also be excluded.
- In the `allowSplit` scenario, `WHSTmpWorkLine` must also be considered.
- If filtering by column, zone, or item-based totals is going to be done, aggregate `AxQuery` + `AxView` reading makes it noticeably easier.

### WMS Short Checklist

1. Verify menu item setup and `Directed by` selection.
2. Determine whether it's legacy or `ProcessGuide`.
3. Separate display class, controls class, and service/controller layer.
4. Design pass keys and integer step IDs.
5. Call `updateModeStepPass()` on UI return.
6. If you've added a new factory/attribute, check the cache effect.
7. Test with different device and user setting combinations.

## 2. Form Interaction and Dialog Patterns

This section gathers general X++ patterns at the runtime level in addition to the `AxForm` and `AxFormExtension` metadata structure.

### Checkbox-based visibility + mandatory pattern

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

Rule set:

- The `Modified` event handles user interaction; the `Activated` event syncs existing records.
- Field/control null checks must always be preserved.
- If the dependent field is no longer valid, the field should be cleared when the toggle is turned off.

### Standard dialog opening pattern

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

Usage notes:

- `wait()` is used for the modal dialog flow.
- The returned record can be retrieved via `args().record()`.
- Cancel state must be handled explicitly; otherwise the half state may remain.

### Conditional mandatory pattern with `validateWrite()`

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

Rules:

- The result of `next validateWrite()` must be preserved.
- The additional condition must only run if `ret == true`.
- Even if you mark mandatory on the UI side, there should be a second defense layer at the table level.

### Carrying default value or dimension from header to line

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

Notes:

- Defaults on the buffer must be set BEFORE `next insert()`.
- For `serviceMergeDefaultDimensions()`, argument order may affect precedence; must be validated according to target behavior.
- This pattern can be applied not only for dimension but also for other header-sourced default fields.

## 3. Entity Scope and Design Checklist

This section gathers general controls to follow when making entity design decisions rather than the metadata XML.

### Target entity selection

| Scenario | Target to look at first |
|---|---|
| Adding a new field to standard table | Existing standard entity family or existing entity extension |
| Custom business table | Custom `AxDataEntityView` |
| Parameter or technical table | Decide based on the actual integration need |
| If existing entity extension already exists | Extend the same extension before opening a new entity |

Decision rules:

- Do not jump to the "no entity" conclusion just because no entity is visible in the repo; standard packages may have a ready entity family.
- If a field has been added to a standard table, the entity side must be considered within the same task scope as the form side.
- For parameter, log, or helper tables, entity requirement depends on integration need.

### Mapped, unmapped, and reference field distinction

| Field type | When used | Note |
|---|---|---|
| Mapped | If field comes directly from source table | Lowest maintenance cost |
| Unmapped | For calculated or runtime-filled fields | Filling point must be documented |
| Reference / join based | If field comes from another datasource | Source datasource and write-back need must be clear |

Practical rule:

- If the source of the field is a table column, start with `MappedField`.
- If calculation or runtime assembly is needed, use `UnmappedField` and write the filling chain in documentation.
- For `RefRecId` or join-based fields, data source and update behavior must be clearly noted.

### Scope control

In entity developments, these pairs should be considered together:

- Header + line pairs (`SalesTable` + `SalesLine`, `PurchTable` + `PurchLine`)
- Product + released product pairs (`EcoResProduct` + `InventTable` / released product entity)
- Families sharing the same reference field like customer + vendor

Extending only one side and skipping the other can cause half-data to appear in integration.

### Staging and integration channel

The staging effect must be considered at the very beginning of entity design.

- If the entity will be used with DMF, the staging effect must be checked from the start.
- Whether the fields will be used in OData, DMF, or only in UI must be clarified.
- Staging and field scope decisions must be documented in the design phase, not later.

### Entity completion checklist

1. Source table and field list extracted.
2. Target entity or entity extension selection justified.
3. Checked if there is an existing target in standard packages.
4. Fields classified as mapped / unmapped / reference.
5. If needed, header-line or product-release pairs considered together.
6. Staging and integration channel effect noted.
7. The relevant privilege design for security need was not forgotten.

## 4. Change Log and Schedule Batch Patterns

This pattern is reusable in modules requiring bulk changes or scheduled updates.

### Central change log table

A typical change log table contains these fields:

| Field | Purpose |
|---|---|
| `UserId` | User who performed the operation |
| `ActionType` | Inline edit, bulk set, overwrite, schedule, etc. |
| `ActionDescription` | Human-readable change summary text |
| `Scope` | Affected business area or target record group |
| `BusinessKey` | Related business key or code |
| `OldValue` / `NewValue` | Changed values |
| `TransDateTime` | Change time |
| `BatchJobId` | Connection if related to batch |

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

Rules:

- If logging is centralized in a single helper class, form, batch, and helper classes use the same audit standard.
- The actual data change and the log insert should be considered within the same transaction scope.
- A standalone `info()` message does not replace the audit trail.

### Pending change + schedule batch pattern

To run a planned change later, a separate pending table is typically used.

| Field | Purpose |
|---|---|
| `ChangeId` | Unique planned job ID |
| `Title` | Title visible to the user |
| `TargetDescription` | Target record group or filter description |
| `ScheduledBy` | User scheduling |
| `ScheduledDateTime` | Planned run time |
| `Status` | `Scheduled`, `Queued`, `Executed`, `Failed`, `Cancelled` |
| `PackedQuery` / `TargetQuery` | Packed query of the selected record group |
| `ExecutedDateTime` | Actual run time |
| `BatchJobId` | Related batch connection |

Operating model:

1. User selects the filtered record group in UI.
2. Operation parameters are written to the pending table in `Scheduled` state.
3. With `SysOperationServiceBase` + `DataContractAttribute` + `SysOperationServiceController`, the job is added to the batch queue.
4. When the batch runs, the pending record is read, the packed query is unpacked, and target records are updated.
5. In the final state, status is updated as `Executed`, `Failed`, or `Cancelled`.

Practical notes:

- Storing a packed query carries the record set selected at scheduling time to the batch run time.
- In the pending list screen, usually only `Scheduled` and `Queued` states are shown.
- If cancel behavior is needed, status change should be managed from a single place.

### When is this pattern suitable?

- If the bulk update will run at a planned time, not at the moment
- If the user's selection needs to be replayed later
- If successful/failed batch runs are to be tracked as audit
- If a clear job record is desired between the UI operation and the background batch

---

# AxMenu and AxMenuExtension

## AxMenu


---

## AxMenuExtension

### General XML Structure

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxMenuExtension xmlns:i="http://www.w3.org/2001/XMLSchema-instance"
                 xmlns="Microsoft.Dynamics.AX.Metadata.V1">
    <Name>StandardMenuName.ModelName</Name>
    <Customizations />
    <Elements>
        <AxMenuExtensionElement xmlns="">
            <!-- Parent: target item name in the existing menu -->
            <Parent>TargetMenuItem</Parent>
            <!-- PositionType and PreviousSibling: positioning (optional) -->
            <MenuElement xmlns="" i:type="AxMenuElementSubMenu">
                <Name>NewSubMenu</Name>
                <Label>SubMenu Label</Label>
                <Elements>
                    <AxMenuElement xmlns="" i:type="AxMenuElementMenuItem">
                        <Name>MenuItemReferenceName</Name>
                        <MenuItemName>MenuItemName</MenuItemName>
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

## MenuExtension Element Types

| i:type | Description |
|---|---|
| `AxMenuElementSubMenu` | Creates a new submenu (group), can contain `<Elements>` |
| `AxMenuElementMenuItem` | A single menu item reference |
| `AxMenuElementMenuItemOutput` | Output menu item |

---

## MenuElement Properties

| Element | Required | Description |
|---|---|---|
| `<Name>` | Yes | Unique reference name in the extension |
| `<MenuItemName>` | For MenuItem | Referenced AxMenuItemDisplay/Action/Output name |
| `<MenuItemType>` | Optional | `Display` (default), `Action`, `Output` |
| `<Elements>` | For SubMenu | Contains sub items |

---

## AxMenuExtensionElement Positioning

| Element | Description |
|---|---|
| `<Parent>` | Name of the existing item in the target menu (e.g., `Setup`, `PeriodicTasks`) |
| `<PositionType>` | `AfterItem` → insert after `<PreviousSibling>` |
| `<PreviousSibling>` | Reference item name when `AfterItem` is used |

**Without Parent**: When `<Parent>` is missing, it is added directly to the menu like the SystemAdministration example.

---

# AxMap and AxMapExtension


## General Structure (For Reference)

Maps are virtual structures that combine multiple tables through a common interface. Multiple tables share the same field names through the map.

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxMap xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>MapName</Name>
    <Fields>
        <AxMapField xmlns="" i:type="AxMapFieldString">
            <Name>FieldName</Name>
        </AxMapField>
    </Fields>
    <Mappings>
        <AxMapMapping>
            <MappingTable>TableName</MappingTable>
            <Fields>
                <AxMapMappingField>
                    <MapField>FieldName</MapField>
                    <MapFieldTo>TableField</MapFieldTo>
                </AxMapMappingField>
            </Fields>
        </AxMapMapping>
    </Mappings>
</AxMap>
```

---

# AxService and AxServiceGroup


## General Structure (For Reference)

Services in D365 FO provide SOAP/REST endpoints for integration with external systems.

```xml
<?xml version="1.0" encoding="utf-8"?>
<AxService xmlns:i="http://www.w3.org/2001/XMLSchema-instance">
    <Name>ServiceName</Name>
    <ServiceOperations>
        <AxServiceOperation>
            <Name>OperationName</Name>
            <ReturnType>void</ReturnType>
        </AxServiceOperation>
    </ServiceOperations>
</AxService>
```

---

# AxConfigurationKey


## General Structure (For Reference)

Configuration keys are used to enable or disable specific features or fields.


---

# X++ Best Practices - D365 F&O Best Practices

> Source: D365_Xpp_Best_Practices.txt — Microsoft Learn, Stoneridge Software, Synoptek, Dynamics Edge, zakharov.com, Confiz, Rand Group

================================================================================
       MICROSOFT DYNAMICS 365 F&O - X++ BEST PRACTICES
================================================================================

Preparation Date: March 13, 2026
Sources: Microsoft Learn, Stoneridge Software, Synoptek, Dynamics Edge,
           zakharov.com, Confiz, Rand Group, and other community sources

================================================================================
1. GENERAL CODING STANDARDS
================================================================================

1.1 Code Structure and Readability
    - Only one statement should be on each line.
    - A line must not exceed 140 characters; long statements should be
      split into multiple lines.
    - A single blank line should be used to separate entities.
    - Curly braces ({}) should be placed around every code block even if
      it contains a single statement.
    - A space should be added between if, switch, for, while keywords and
      the opening parenthesis.
    - No space should be left between the function name and the opening
      parenthesis.
    - A space should be added after the NOT (!) operator.

1.2 Method Design
    - Methods should be small and understandable. Each method should perform
      a single well-defined task.
    - The method name should easily describe what it does.
    - There should be only one successful return point (return) in the code
      (usually the last statement). Switch cases and initial condition
      checks are exceptions.
    - Value-passed (by value) parameters should not be assigned a value
      or manipulated. These parameters should be treated as constants.
    - An access modifier should be added to all methods:
      public, protected, or private.

1.3 Code Cleanliness
    - Unused variables, methods, and classes should be deleted.
    - Commented-out non-working code should be cleaned up.
    - Reuse code: Move repeated lines into a method instead of
      repeating them in many places.

1.4 Error Handling
    - The user should never experience a runtime error.
    - Errors should be handled programmatically or the user should be
      informed via Infolog.
    - infolog.add() should not be used directly. Instead, the redirect
      methods error(), warning(), info(), and checkFailed() should be used.
    - The application should be designed accordingly to avoid deadlocks.

================================================================================
2. NAMING CONVENTIONS
================================================================================

2.1 General Naming Rules
    - Object names should be created hierarchically:
      {Business Area Name} + {Business Area Description} + {Action/Content Type}
      Example: CustInvoicePrintout, PriceDiscAdmCopy
    - Application objects (table, class, form, report, etc.) should be
      named with mixed case (PascalCase).
      Example: AddressFormatHeading, SalesAmount
    - Methods, variables, and system functions should be named with
      camelCase (first letter lowercase).
      Example: custParameters, salesAmount

2.2 Extension Naming Rules
    - Suffix is used instead of prefix for extension classes.
    - Class extensions: {BaseObjectName}_PRJ_Extension
      Example: InventJournalTrans_PRJ_Extension
    - Event handler classes: SXAxxx_EventHandler
    - For forms, tables, and other objects: {ObjectName}_PRJ
    - Set a consistent naming convention within your project and stick
      to it.

2.3 Table Buffer Variables
    - Table buffer variables should be named the same as the table name
      as much as possible, but the first letter should be lowercase.
      Example: For CustTable table -> custTable

================================================================================
3. CHAIN OF COMMAND (CoC) BEST PRACTICES
================================================================================

3.1 Basic Rules
    - Overlayering (directly modifying Microsoft's base code) should never
      be done. Always use extension.
    - Chain of Command is the preferred primary mechanism for extending
      method logic.
    - Extension class name should end with "_Extension".
    - The "final" keyword should be used in class definition.
    - The [ExtensionOf(classStr(...))] attribute should be added.

3.2 next Call
    - The next call should always be made in wrapper methods.
      The next call invokes the next method in the chain and ultimately
      the original implementation.
    - The next call must be in the first-level statements of the method
      body.
    - The next call cannot be made conditionally inside an if block.
    - The next call cannot be made in while, do-while, or for loop statements.
    - A return statement cannot be before the next statement.
    - The next call is optional only in methods marked [Replaceable]; even
      in this case it should be skipped conditionally.

3.3 CoC vs Event Handler Choice
    - General rule: CoC should be used to modify or extend core logic.
    - Event handlers should be used to react to framework events not bound
      to a specific method override (delegate, form events, data events).
    - Event handler should be used when CoC is not possible (private methods,
      some kernel methods).

3.4 Performance and Maintenance
    - Keep extension code lean and efficient. CoC adds code to
      performance-sensitive processes.
    - Avoid creating an excessive number of extension chains on a single
      method; this negatively affects performance and the debug process.
    - CoC extensions should be focused and purpose-driven.

================================================================================
4. DATABASE AND QUERY OPTIMIZATION
================================================================================

4.1 Select Statements
    - Use a field list specifying only the necessary fields instead of
      SELECT *. This optimizes query time.
    - firstOnly: If you will use only the first record or only one record
      can be found, increase performance by using the firstOnly qualifier.
    - Apply WHERE clauses and filters as early as possible; reduce the
      amount of data processed.

4.2 Set-Based Operations
    - Row-by-row processing creates unnecessary database round-trips
      and does not scale.
    - As much as possible, use set-based operations like SUM(), COUNT(),
      UPDATE_RECORDSET, DELETE_FROM, and INSERT_RECORDSET; these are much
      more efficient than while selects.
    - For UPDATE_RECORDSET and DELETE_FROM statements to be effective,
      skipDataMethods(), skipDeleteActions(), and skipDatabaseLog() should
      be applied when needed.

4.3 Join Usage
    - Reduce execution time and improve readability by using join instead
      of nested while loops.
    - Use 'exists join' if values from all joined tables are not needed.

4.4 Indexing
    - Add indexes to frequently filtered fields.
    - But be careful: excessive indexing can negatively impact write
      performance and maintenance overhead.

4.5 Temporary Tables
    - Temporary tables should be used carefully, with attention to scope
      and lifecycle.
    - Minimize load by reducing unnecessary temporary table creation operations.

================================================================================
5. COMMENT STANDARDS
================================================================================

5.1 Code Comments
    - Add comments explaining what your code should do and what the
      parameters are used for.
    - All changes should have comments explaining what the code does,
      including unique ticket number / task description.
    - Recommended comment format:
      // BEGIN <Feature/Bug Number> <Description> <Date> <Initials>
      <code here>
      // END <Feature/Bug Number> <Description> <Date> <Initials>

5.2 XML Header Comments
    - XML header comments should be created before every public method.
    - Can be auto-created by typing "///" above the method header:
      /// <summary>
      /// Description of the method
      /// </summary>
      /// <param name="paramName">Parameter description</param>
      /// <remarks>
      /// <Feature/Bug Number> <Description> <Date> <Initials>
      /// </remarks>

================================================================================
6. LABEL AND CONSTANT VALUE STANDARDS
================================================================================

    - Hard-coded values / labels should not be used in code.
    - Values coming from the database should be retrieved from the database.
    - Create a new label file in your custom model.
    - Determine if it needs to be created in multiple languages.
    - All texts shown to the user should be managed through the label
      system.

================================================================================
7. TABLE DESIGN BEST PRACTICES
================================================================================

    - Table name should follow this structure:
      Prefix (Module Name) + Logical Description + Data Type Suffix
      Examples: CustTrans, SalesLine, ProjJour, InventParameters
    - When creating new tables, ensure that the table properties (Table
      Group etc.) are appropriate for the table's purpose.
    - A delete action must be defined for each relationship between two tables.
    - Instead of writing code, use table delete actions to indicate whether
      delete operations are restricted or cascading.
    - Field groups should have the same grouping structure at the database
      and form/report level (improves caching performance).
    - If table method properties (delete, validateDelete, etc.) are
      available; prefer adjusting property values to implement instead of
      writing X++ code.

================================================================================
8. FORM DEVELOPMENT BEST PRACTICES
================================================================================

    - Avoid directly referencing form control names.
      Bad example: salestable_SalesId.enabled(false)
    - Use the Form Style Checker to verify that forms comply with
      predefined templates.
    - Keep the number of variables defined at global level minimum; this
      improves performance.
    - Inline variable declaration is valid in D365 F&O; use it when possible.

================================================================================
9. ACCESS MODIFIERS AND SECURITY
================================================================================

    - Access modifiers should be used in all methods: public,
      protected, private.
    - Do not change method access modifiers to 'public' for pre or post
      event handlers; instead use Chain of Command.
    - Use the most restrictive access level possible.

================================================================================
10. SOURCE CONTROL AND PROJECT MANAGEMENT
================================================================================

    - It is very important to use a source control system (TFS / Azure
      DevOps / Git); it provides code security and change history.
    - Regularly pull (sync) the code and prevent conflicts.
    - Take a source code backup before modifying an existing object.
    - Make it a habit to take a database backup before starting development.
    - Code review should always be done; provides different perspectives
      and catches errors early.

================================================================================
11. MODEL AND PACKAGE MANAGEMENT
================================================================================

    - In D365 F&O, creating a model is mandatory for any customization.
    - Simplify deployment and maintenance by grouping related extensions
      in a single model.
    - Create a customization-specific new extension instead of using
      existing customizations.
    - Packages are independent compilation units (assembly/DLL); they
      can reference other packages.

================================================================================
12. GENERAL PERFORMANCE TIPS
================================================================================

    - Design and schedule batch jobs carefully; poorly designed batch
      jobs can create significant performance bottlenecks.
    - Take advantage of architectural improvements like Planning Optimization.
    - Use built-in diagnostics and telemetry tools to monitor performance
      trends and bottlenecks.
    - Regularly review frequently accessed forms, long-running processes,
      and resource-intensive operations.

================================================================================
13. BEST PRACTICE VALIDATION TOOLS
================================================================================

    - Enable "Best Practice checks" option in Visual Studio:
      Tools > Options > Development
    - During compilation, BP (Best Practice) violations appear in the
      message log.
    - Use the Form Style Checker to verify forms comply with templates.
    - You can write your own custom rules with Code Best Practice
      Framework (CBPF).
    - Run BP checks in daily builds to catch unacceptable practices early.

================================================================================
14. SUMMARY CHECKLIST
================================================================================

    [x] One statement per line, 140-character limit
    [x] Curly braces used in all code blocks
    [x] Methods small, focused, and performing a single task
    [x] Consistent PascalCase/camelCase naming conventions
    [x] _Extension suffix on extension classes
    [x] Always call next in CoC (except Replaceable)
    [x] CoC preferred over event handlers
    [x] Field list usage instead of SELECT *
    [x] Optimization qualifiers like firstOnly, exists join
    [x] Set-based operations (INSERT_RECORDSET, UPDATE_RECORDSET, etc.)
    [x] Label system instead of hard-coded values
    [x] XML header comments and meaningful code comments
    [x] Source control and code review
    [x] Enabling BP validation tools
    [x] Cleaning up unused code and variables
    [x] Use of appropriate access modifiers
    [x] Defining delete action in table relations
    [x] Source code and DB backup before development

================================================================================
                              --- END ---
================================================================================


---

# X++ Code Examples - D365 F&O Basic Operations

> Source: ExamplesX++.txt — community.dynamics.com, nuxulu.com, paragchapre.com, Microsoft Dynamics Community

================================================================================
   DYNAMICS 365 FINANCE & OPERATIONS - BASIC OPERATIONS X++ CODE EXAMPLES
================================================================================
   Preparation Date: March 13, 2026
   Source: Internet research (community blogs, Microsoft Dynamics Community)
   Note: The codes have been adapted to the D365 F&O environment. Demo data (USMF)
        has been used. Change values according to your environment.
================================================================================


================================================================================
1. CREATE PURCHASE ORDER
================================================================================

// Create purchase order - using PurchTable and PurchLine
// Source: community.dynamics.com, nuxulu.com, paragchapre.com

class CreatePurchaseOrder
{
    public static void main(Args _args)
    {
        PurchTable      purchTable;
        PurchLine       purchLine;
        InventDim       inventDim;
        NumberSeq       numberSeq;
        VendTable       vendTable = VendTable::find("US-101"); // Vendor account

        try
        {
            ttsBegin;

            // Purchase order header
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

            // Purchase order line
            purchLine.clear();
            purchLine.initFromPurchTable(purchTable);
            purchLine.ItemId = "D0001";     // Item number
            purchLine.PurchQty = 10;        // Quantity
            purchLine.PurchPrice = 100;     // Unit price

            inventDim.clear();
            inventDim.InventSiteId = "1";       // Site
            inventDim.InventLocationId = "11";   // Warehouse
            purchLine.InventDimId = InventDim::findOrCreate(inventDim).inventDimId;

            purchLine.createLine(true, true, true, true, true, true);

            ttsCommit;

            info(strFmt("Purchase order created: %1", purchTable.PurchId));
        }
        catch (Exception::Error)
        {
            error("An error occurred while creating the purchase order.");
        }
    }
}


================================================================================
2. CONFIRM PURCHASE ORDER
================================================================================

// PO Confirmation - using PurchFormLetter class
// Source: nuxulu.com, d365ffo.com

public void confirmPurchaseOrder(PurchId _purchId)
{
    PurchTable          purchTable = PurchTable::find(_purchId);
    PurchFormLetter     purchFormLetter;

    purchFormLetter = PurchFormLetter::construct(DocumentStatus::PurchaseOrder);
    purchFormLetter.update(purchTable, strFmt("PO-Conf-%1", _purchId));

    info(strFmt("Purchase order confirmed: %1", _purchId));
}


================================================================================
3. PURCHASE ORDER PRODUCT RECEIPT
================================================================================

// Posting Product Receipt
// Source: nuxulu.com, d365ffo.com

public void postProductReceipt(PurchId _purchId, PackingSlipId _packingSlipId)
{
    PurchTable          purchTable = PurchTable::find(_purchId);
    PurchFormLetter     purchFormLetter;

    purchFormLetter = PurchFormLetter::construct(DocumentStatus::PackingSlip);
    purchFormLetter.update(purchTable, _packingSlipId);

    info(strFmt("Product receipt posted: PO=%1, PackingSlip=%2", _purchId, _packingSlipId));
}


================================================================================
4. POST PURCHASE INVOICE / VENDOR INVOICE
================================================================================

// Creating and posting purchase invoice
// Source: axtechsolutions.blogspot.com, d365ffo.com

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

    info(strFmt("Purchase invoice posted: PO=%1, Invoice=%2", _purchId, _invoiceId));
}


================================================================================
5. CREATE SALES ORDER
================================================================================

// Creating sales order
// Source: community.dynamics.com, paragchapre.com, blog.peterdx.com

class CreateSalesOrder
{
    public static void main(Args _args)
    {
        SalesTable      salesTable;
        SalesLine       salesLine;
        InventDim       inventDim;
        NumberSeq       numberSeq;
        CustTable       custTable = CustTable::find("US-007"); // Customer account

        try
        {
            ttsBegin;

            // Sales order header
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

            // Sales order line
            salesLine.clear();
            salesLine.initFromSalesTable(salesTable);
            salesLine.ItemId = "D0001";     // Item number
            salesLine.SalesQty = 5;         // Quantity
            salesLine.SalesPrice = 150;     // Unit price

            inventDim.clear();
            inventDim.InventSiteId = "1";
            inventDim.InventLocationId = "11";
            salesLine.InventDimId = InventDim::findOrCreate(inventDim).inventDimId;

            salesLine.createLine(true, true, true, true, true);

            ttsCommit;

            info(strFmt("Sales order created: %1", salesTable.SalesId));
        }
        catch (Exception::Error)
        {
            error("An error occurred while creating the sales order.");
        }
    }
}


================================================================================
6. SALES ORDER CONFIRMATION
================================================================================

// Sales order confirmation
// Source: community.dynamics.com, shyamkannadasan.blogspot.com

public void confirmSalesOrder(SalesId _salesId)
{
    SalesTable          salesTable = SalesTable::find(_salesId);
    SalesFormLetter     salesFormLetter;

    salesFormLetter = SalesFormLetter::construct(DocumentStatus::Confirmation);
    salesFormLetter.update(salesTable);

    info(strFmt("Sales order confirmed: %1", _salesId));
}


================================================================================
7. SALES PACKING SLIP
================================================================================

// Posting sales packing slip
// Source: rahulmsdax.blogspot.com, sangeethwiki.blogspot.com

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
                                           false);  // Print

        if (salesFormLetter_PackingSlip.parmJournalRecord().TableId == tableNum(CustPackingSlipJour))
        {
            custPackingSlipJour = salesFormLetter_PackingSlip.parmJournalRecord();
            info(strFmt("Packing slip posted: %1", custPackingSlipJour.PackingSlipId));
        }
    }

    ttsCommit;
}


================================================================================
8. POST SALES INVOICE
================================================================================

// Posting sales invoice
// Source: rahulmsdax.blogspot.com, chaituax.wordpress.com

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
                           NoYes::No);   // Print

    ttsCommit;

    info(strFmt("Sales invoice posted: %1", _salesId));
}


================================================================================
9. CREATE PRODUCTION ORDER
================================================================================

// Creating production order
// Source: community.dynamics.com (DAX Beginners), bmdax.blogspot.com

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

        // Initialize base values
        prodTable.initValue();
        prodTable.initFromInventTable(inventTable);
        prodTable.ItemId = inventTable.ItemId;
        prodTable.DlvDate = today();
        prodTable.QtySched = qty;
        prodTable.RemainInventPhysical = qty;

        // Initialize InventDim
        inventDim.initValue();

        // Set active BOM and Route
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

        // Initialize BOM and Route versions
        prodTable.initBOMVersion();
        prodTable.initRouteVersion();

        // Create production order with ProdTableType class
        prodTable.type().insert();

        info(strFmt("Production order created: %1", prodTable.ProdId));
    }
}


================================================================================
10. INVENTORY MOVEMENT JOURNAL
================================================================================

// Creating and posting inventory movement journal
// Source: learnax.blogspot.com, d365opstechtalks.com, community.dynamics.com

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

        // Create Journal Header
        inventJournalTable.clear();
        inventJournalName = InventJournalName::standardJournalName(InventJournalType::Movement);
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(inventJournalName));
        inventJournalTable.insert();

        // Create Journal Line
        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.TransDate = systemDateGet();
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.initFromInventTable(InventTable::find("D0001"));
        inventJournalTrans.Qty = 10;     // Positive = receipt, Negative = issue

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        inventJournalTrans.InventDimId = InventDim::findOrCreate(inventDim).inventDimId;
        inventJournalTrans.insert();

        ttsCommit;

        // Post Journal
        journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        journalCheckPost.parmThrowCheckFailed(false);
        journalCheckPost.parmTransferErrors(NoYes::No);
        journalCheckPost.run();

        info(strFmt("Movement journal created and posted: %1",
            inventJournalTable.JournalId));
    }
}


================================================================================
11. INVENTORY TRANSFER JOURNAL
================================================================================

// Inventory transfer journal - transfer between warehouses
// Source: community.dynamics.com, linkedin.com (Usama Mehmood)

class CreateTransferJournal
{
    public static void main(Args _args)
    {
        InventJournalTable          inventJournalTable;
        InventJournalTrans          inventJournalTrans;
        InventJournalCheckPost      inventJournalCheckPost;
        InventDim                   fromInventDim, toInventDim;

        ttsBegin;

        // Journal Header
        inventJournalTable.clear();
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(
                InventParameters::find().TransferJournalNameId));
        inventJournalTable.Description = "Inventory Transfer Journal";
        inventJournalTable.insert();

        // Journal Line
        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.Qty = 5;

        // Source (From) Dimension
        fromInventDim.clear();
        fromInventDim.InventSiteId = "1";
        fromInventDim.InventLocationId = "11";  // Source warehouse
        inventJournalTrans.InventDimId =
            InventDim::findOrCreate(fromInventDim).inventDimId;

        // Target (To) Dimension
        toInventDim.clear();
        toInventDim.InventSiteId = "1";
        toInventDim.InventLocationId = "12";    // Target warehouse
        inventJournalTrans.ToInventDimId =
            InventDim::findOrCreate(toInventDim).inventDimId;

        inventJournalTrans.insert();

        ttsCommit;

        // Post Journal
        inventJournalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        inventJournalCheckPost.parmThrowCheckFailed(false);
        inventJournalCheckPost.parmTransferErrors(NoYes::No);
        inventJournalCheckPost.run();

        info(strFmt("Transfer journal created: %1", inventJournalTable.JournalId));
    }
}


================================================================================
12. CREATE TRANSFER ORDER
================================================================================

// Creating inter-warehouse transfer order (with transit stock tracking)
// Source: d365ffo.com, dynamics2012to365.blogspot.com

class CreateTransferOrder
{
    public static void main(Args _args)
    {
        InventTransferTable     inventTransferTable;
        InventTransferLine      inventTransferLine;
        NumberSeq               numberSeq;
        InventDim               inventDim;

        ttsBegin;

        // Transfer Order Header
        numberSeq = NumberSeq::newGetNum(InventParameters::numRefTransferId());

        inventTransferTable.clear();
        inventTransferTable.initValue();
        inventTransferTable.TransferId = numberSeq.num();
        numberSeq.used();

        inventTransferTable.InventLocationIdFrom = "11";   // Source warehouse
        inventTransferTable.modifiedField(
            fieldNum(InventTransferTable, InventLocationIdFrom));

        inventTransferTable.InventLocationIdTo = "12";     // Target warehouse
        inventTransferTable.modifiedField(
            fieldNum(InventTransferTable, InventLocationIdTo));

        inventTransferTable.TransferStatus = InventTransferStatus::Created;
        inventTransferTable.insert();

        // Transfer Order Line
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

        info(strFmt("Transfer order created: %1", inventTransferTable.TransferId));
    }
}


================================================================================
13. CREATE GENERAL JOURNAL
================================================================================

// Creating and posting general journal
// Source: community.dynamics.com, denistrunin.com

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

        // Find journal name
        select firstonly ledgerJournalName
            where ledgerJournalName.JournalName == "GenJrn";  // Journal name

        ttsBegin;

        // Journal Header
        ledgerJournalTable.JournalName = ledgerJournalName.JournalName;
        ledgerJournalTable.initFromLedgerJournalName();
        ledgerJournalTable.JournalNum =
            JournalTableData::newTable(ledgerJournalTable).nextJournalId();
        ledgerJournalTable.Name = "Journal created with X++";
        ledgerJournalTable.insert();

        // Get voucher number
        numberSeq = NumberSeq::newGetVoucherFromCode(
            NumberSequenceTable::find(
                ledgerJournalName.NumberSequenceTable).NumberSequence);
        voucher = numberSeq.voucher();

        // Journal Line - Debit
        ledgerJournalTrans.clear();
        ledgerJournalTrans.JournalNum = ledgerJournalTable.JournalNum;
        ledgerJournalTrans.TransDate = today();
        ledgerJournalTrans.AccountType = LedgerJournalACType::Ledger;
        ledgerJournalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "110110",   // Main account number
                LedgerJournalACType::Ledger);
        ledgerJournalTrans.AmountCurDebit = 1000;
        ledgerJournalTrans.CurrencyCode = "USD";
        ledgerJournalTrans.Txt = "Test debit transaction";
        ledgerJournalTrans.Voucher = voucher;
        ledgerJournalTrans.Approved = NoYes::Yes;
        ledgerJournalTrans.insert();

        // Journal Line - Credit
        ledgerJournalTrans.clear();
        ledgerJournalTrans.JournalNum = ledgerJournalTable.JournalNum;
        ledgerJournalTrans.TransDate = today();
        ledgerJournalTrans.AccountType = LedgerJournalACType::Ledger;
        ledgerJournalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "170150",   // Main account number
                LedgerJournalACType::Ledger);
        ledgerJournalTrans.AmountCurCredit = 1000;
        ledgerJournalTrans.CurrencyCode = "USD";
        ledgerJournalTrans.Txt = "Test credit transaction";
        ledgerJournalTrans.Voucher = voucher;
        ledgerJournalTrans.Approved = NoYes::Yes;
        ledgerJournalTrans.insert();

        ttsCommit;

        // Post Journal
        journalCheckPost = LedgerJournalCheckPost::newLedgerJournalTable(
            ledgerJournalTable, NoYes::Yes);
        journalCheckPost.runOperation();

        info(strFmt("General journal created and posted: %1",
            ledgerJournalTable.JournalNum));
    }
}


================================================================================
14. VENDOR INVOICE JOURNAL
================================================================================

// Creating vendor invoice journal
// Source: d365opstechtalks.com, community.dynamics.com

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

        // Find journal name (vendor invoice journal)
        select firstonly ledgerJournalName
            where ledgerJournalName.JournalName == "APInv";  // AP Invoice Journal

        // Create Header
        journalTable.initValue();
        journalTable.JournalName = ledgerJournalName.JournalName;
        journalTable.initFromLedgerJournalName();
        journalTable.JournalNum =
            JournalTableData::newTable(journalTable).nextJournalId();
        journalTable.Name = "Vendor Invoice Journal";
        journalTable.insert();

        // Create Line
        journalTrans.clear();
        journalTrans.initValue();
        journalTrans.JournalNum = journalTable.JournalNum;
        journalTrans.TransDate = today();

        // Vendor account
        journalTrans.AccountType = LedgerJournalACType::Vend;
        journalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "US-101",   // Vendor account number
                LedgerJournalACType::Vend);

        journalTrans.AmountCurCredit = 5000;
        journalTrans.CurrencyCode = "USD";

        // Offset account
        journalTrans.OffsetAccountType = LedgerJournalACType::Ledger;
        journalTrans.OffsetLedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "600120",   // Expense account
                LedgerJournalACType::Ledger);

        journalTrans.Txt = "Vendor invoice";
        journalTrans.Invoice = "INV-001";
        journalTrans.Approved = NoYes::Yes;
        journalTrans.insert();

        ttsCommit;

        info(strFmt("Vendor invoice journal created: %1", journalTable.JournalNum));
    }
}


================================================================================
15. VENDOR PAYMENT JOURNAL
================================================================================

// Creating vendor payment journal
// Source: linkedin.com (Usama Mehmood), axvigneshvaran.wordpress.com

class CreateVendorPaymentJournal
{
    public static void main(Args _args)
    {
        LedgerJournalTable      journalTable;
        LedgerJournalTrans      journalTrans;

        ttsBegin;

        // Header
        journalTable.initValue();
        journalTable.JournalName = "VendPay";  // Vendor payment journal name
        journalTable.initFromLedgerJournalName();
        journalTable.JournalNum =
            JournalTableData::newTable(journalTable).nextJournalId();
        journalTable.Name = "Vendor Payment Journal";
        journalTable.insert();

        // Line
        journalTrans.clear();
        journalTrans.initValue();
        journalTrans.JournalNum = journalTable.JournalNum;
        journalTrans.TransDate = today();

        // Vendor account
        journalTrans.AccountType = LedgerJournalACType::Vend;
        journalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "US-101",
                LedgerJournalACType::Vend);

        journalTrans.AmountCurDebit = 5000;
        journalTrans.CurrencyCode = "USD";

        // Offset account - Bank
        journalTrans.OffsetAccountType = LedgerJournalACType::Bank;
        journalTrans.OffsetLedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "USMF OPER",    // Bank account
                LedgerJournalACType::Bank);

        journalTrans.Txt = "Vendor payment";
        journalTrans.Approved = NoYes::Yes;
        journalTrans.insert();

        ttsCommit;

        info(strFmt("Vendor payment journal created: %1", journalTable.JournalNum));
    }
}


================================================================================
16. CUSTOMER PAYMENT JOURNAL
================================================================================

// Creating customer payment journal
// Source: dynamicsaxforall.blogspot.com, axvigneshvaran.wordpress.com

class CreateCustomerPaymentJournal
{
    public static void main(Args _args)
    {
        LedgerJournalTable              journalTable;
        LedgerJournalTrans              journalTrans;
        LedgerJournalEngine_CustPayment ledgerJournalEngine;

        ledgerJournalEngine = new LedgerJournalEngine_CustPayment();

        ttsBegin;

        // Header
        journalTable.initValue();
        journalTable.JournalNum =
            JournalTableData::newTable(journalTable).nextJournalId();
        journalTable.JournalName = "CustPay";  // Customer payment journal name
        journalTable.initFromLedgerJournalName();
        journalTable.Name = "Customer Payment Journal";
        journalTable.insert();

        // Line
        journalTrans.clear();
        journalTrans.initValue();
        ledgerJournalEngine.newJournalActive(journalTable);
        ledgerJournalEngine.initValue(journalTrans);

        journalTrans.JournalNum = journalTable.JournalNum;
        journalTrans.TransDate = today();
        journalTrans.AccountType = LedgerJournalACType::Cust;
        journalTrans.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "US-007",   // Customer account number
                LedgerJournalACType::Cust);

        journalTrans.AmountCurCredit = 3000;
        journalTrans.CurrencyCode = "USD";

        // Offset account - Bank
        journalTrans.OffsetAccountType = LedgerJournalACType::Bank;
        journalTrans.OffsetLedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "USMF OPER",
                LedgerJournalACType::Bank);

        journalTrans.Txt = "Customer payment";
        journalTrans.Approved = NoYes::Yes;
        journalTrans.insert();

        ttsCommit;

        info(strFmt("Customer payment journal created: %1", journalTable.JournalNum));
    }
}


================================================================================
17. CREATE FREE TEXT INVOICE
================================================================================

// Creating free text invoice
// Source: axvigneshvaran.wordpress.com, dynamicsaxforall.blogspot.com

class CreateFreeTextInvoice
{
    public static void main(Args _args)
    {
        CustInvoiceTable    custInvoiceTable;
        CustInvoiceLine     custInvoiceLine;
        CustTable           custTable = CustTable::find("US-007");

        ttsBegin;

        // Invoice Header
        custInvoiceTable.initFromCustTable(custTable);
        custInvoiceTable.InvoiceDate =
            DateTimeUtil::getSystemDate(
                DateTimeUtil::getUserPreferredTimeZone());
        custInvoiceTable.insert();

        // Invoice Line
        custInvoiceLine.initValue();
        custInvoiceLine.initFromCustInvoiceTable(custInvoiceTable);
        custInvoiceLine.Description = "Consulting service";
        custInvoiceLine.AmountCur = 2500;
        custInvoiceLine.LedgerDimension =
            LedgerDynamicAccountHelper::getDynamicAccountFromAccountNumber(
                "401200",
                LedgerJournalACType::Ledger);
        custInvoiceLine.insert();

        ttsCommit;

        info(strFmt("Free text invoice created: %1",
            custInvoiceTable.InvoiceId));
    }
}


================================================================================
18. INVENTORY COUNTING JOURNAL
================================================================================

// Creating inventory counting journal
// Source: msdynamicshelper.blogspot.com, allaboutdynamic.com

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

        // Counting Journal Header
        inventJournalTable.clear();
        inventJournalName =
            InventJournalName::standardJournalName(InventJournalType::Count);
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(inventJournalName));
        inventJournalTable.Description = "Inventory Counting Journal";
        inventJournalTable.insert();

        // Counting Journal Line
        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.TransDate = systemDateGet();
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.initFromInventTable(InventTable::find("D0001"));
        inventJournalTrans.Counted = 50;     // Counted quantity
        inventJournalTrans.Qty = 0;          // System will calculate difference

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        inventJournalTrans.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;
        inventJournalTrans.insert();

        ttsCommit;

        // Post
        journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        journalCheckPost.parmThrowCheckFailed(false);
        journalCheckPost.parmTransferErrors(NoYes::No);
        journalCheckPost.run();

        info(strFmt("Counting journal created: %1", inventJournalTable.JournalId));
    }
}


================================================================================
19. BOM JOURNAL
================================================================================

// Creating BOM journal
// Source: sangeethwiki.blogspot.com

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

        // BOM Journal Header
        journalTable.clear();
        journalTable.JournalId = journalTableData.nextJournalId();
        journalTable.JournalNameId = InventParameters::find().BOMJournalNameId;
        journalTable.JournalType = InventJournalType::BOM;
        journalTableData.initFromJournalName(
            journalTableData.journalStatic().findJournalName(
                journalTable.JournalNameId));
        journalTable.insert();

        // BOM Journal Line
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

        // Post
        journalCheckPost = InventJournalCheckPost::newPostJournal(journalTable);
        journalCheckPost.run();

        info(strFmt("BOM journal created: %1", journalTable.JournalId));
    }
}


================================================================================
20. POST ANY INVENTORY JOURNAL
================================================================================

// Posting all inventory journals (Movement, Transfer, Counting, BOM)
// Source: allaboutdynamic.com

public void postInventoryJournal(InventJournalId _journalId)
{
    JournalCheckPost    journalCheckPost;
    InventJournalTable  inventJournalTable;

    inventJournalTable = InventJournalTable::find(_journalId);

    journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
    journalCheckPost.parmThrowCheckFailed(false);
    journalCheckPost.parmTransferErrors(NoYes::No);
    journalCheckPost.run();

    info(strFmt("Inventory journal posted: %1", _journalId));
}


================================================================================
21. POST LEDGER JOURNAL
================================================================================

// Posting general ledger journal
// Source: cloudfronts.com

public void postLedgerJournal(LedgerJournalTable _ledgerJournalTable)
{
    LedgerJournalCheckPost jourPost;

    jourPost = LedgerJournalCheckPost::newLedgerJournalTable(
        _ledgerJournalTable, NoYes::Yes);
    jourPost.runOperation();

    info(strFmt("Ledger journal posted: %1",
        _ledgerJournalTable.JournalNum));
}


================================================================================
22. CREATE PURCHASE REQUISITION
================================================================================

// Creating purchase requisition (basic structure)

class CreatePurchRequisition
{
    public static void main(Args _args)
    {
        PurchReqTable       purchReqTable;
        PurchReqLine        purchReqLine;
        InventDim           inventDim;

        ttsBegin;

        // Header
        purchReqTable.initValue();
        purchReqTable.PurchReqName = "Test Purchase Requisition";
        purchReqTable.RequisitionStatus = PurchReqRequisitionStatus::Draft;
        purchReqTable.insert();

        // Line
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

        info(strFmt("Purchase requisition created: %1", purchReqTable.PurchReqId));
    }
}


================================================================================
23. CREATE SALES QUOTATION
================================================================================

// Creating sales quotation (basic structure)

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

        // Quotation Header
        quotationTable.QuotationId = numberSeq.num();
        quotationTable.initValue();
        quotationTable.CustAccount = "US-007";
        quotationTable.initFromCustTable();
        quotationTable.insert();

        // Quotation Line
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

        info(strFmt("Sales quotation created: %1", quotationTable.QuotationId));
    }
}


================================================================================
24. INVENTORY ADJUSTMENT JOURNAL
================================================================================

// Inventory adjustment journal - similar structure to Movement Journal
// Source: allaboutdynamic.com

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
                InventJournalType::LossProfit);  // Adjustment journal type
        inventJournalTable.initFromInventJournalName(
            InventJournalName::find(inventJournalName));
        inventJournalTable.Description = "Inventory Adjustment Journal";
        inventJournalTable.insert();

        inventJournalTrans.clear();
        inventJournalTrans.initFromInventJournalTable(inventJournalTable);
        inventJournalTrans.TransDate = systemDateGet();
        inventJournalTrans.ItemId = "D0001";
        inventJournalTrans.initFromInventTable(InventTable::find("D0001"));
        inventJournalTrans.Qty = 5;  // Positive: increase, Negative: decrease

        inventDim.clear();
        inventDim.InventSiteId = "1";
        inventDim.InventLocationId = "11";
        inventJournalTrans.InventDimId =
            InventDim::findOrCreate(inventDim).inventDimId;
        inventJournalTrans.insert();

        ttsCommit;

        // Post
        InventJournalCheckPost journalCheckPost;
        journalCheckPost = InventJournalCheckPost::newPostJournal(inventJournalTable);
        journalCheckPost.run();

        info(strFmt("Adjustment journal created: %1",
            inventJournalTable.JournalId));
    }
}


================================================================================
25. CREATE BATCH JOB (SysOperation Framework)
================================================================================

// Creating batch job - SysOperation Framework
// Source: medium.com, dynamics365musings.com

// --- Controller Class ---
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
        return "Batch Operation Example";
    }

    public static void main(Args _args)
    {
        MyBatchController controller = MyBatchController::construct();
        controller.parmArgs(_args);
        controller.startOperation();
    }
}

// --- Service Class ---
class MyBatchService extends SysOperationServiceBase
{
    public void processRecords()
    {
        // Write your business logic here
        info("Batch job ran successfully.");
    }
}


================================================================================
                          SOURCES AND REFERENCES
================================================================================

The code examples in this document have been compiled from the following sources:

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

# Standard Table Field Reference Documents

This section lists comprehensive table-field reference documents produced from standard D365 F&O system metadata.
Each document contains **all fields** of the related table family with type, EDT/Enum, label, and description information.

- Source version: `10.0.2345.153 PackagesLocalDirectory`
- Production date: 2026-04-03

## Document List

| Document | Scope | Size | Path |
|---|---|---|---|
| **Invent Tables Field Reference** | 569 tables (Core 30 + Other 230 + History 13 + Tmp 101 + Localization 49 + Staging 146) | ~1.1 MB, 14800 lines | [Invent_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Invent_Tablolari_Alan_Referansi_20260403.md) |
| **Purch Tables Field Reference** | 368 tables (Core 23 + Other 113 + History 28 + Tmp 54 + Localization 28 + Staging 122) | ~933 KB, 11000 lines | [Purch_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Purch_Tablolari_Alan_Referansi_20260403.md) |
| **Sales Tables Field Reference** | 267 tables (Core 15 + Other 49 + History 13 + Tmp 33 + Localization 31 + Staging 126) | ~1 MB, 11100 lines | [Sales_Tablolari_Alan_Referansi_20260403.md](docs/MdDocuments/Sales_Tablolari_Alan_Referansi_20260403.md) |
| **Purch-Invent Core Operations Guide** | Purch and Invent core operations tables, relationship chains, and scenario-based guide | ~4 KB, 115 lines | [Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md](docs/MdDocuments/Purch_Invent_Cekirdek_Operasyon_Tablolari_20260403.md) |

## When to Look at Which Document?

| Scenario | Document to Look At |
|---|---|
| What fields does a table have, what are their types? | Related field reference document (Invent/Purch/Sales) |
| Field name validation when designing AxForm/View/Entity | Related field reference document → Core category |
| Understanding table relationship and operation flow | Purch-Invent Core Operations Guide |
| Checking existing fields when adding new field to extension | Related field reference document |
| Validating CoC method parameters | Related field reference document → related table section |
| DataField mapping issues (see: `01-metadata-xml-rules.md` Section 6b) | Related field reference document |

## Document Structure

Each field reference document has the following structure:
- **Category Summary:** Core, Other, History, Tmp, Localization, Staging distribution
- **For each table:**
  - Category, source metadata file, table purpose, field count, title fields
  - All fields: Field name, XML type, EDT/Enum, Label, description

## Purch-Invent Decision Summary

| Question | Purch side | Invent side |
|---|---|---|
| Where is the header record? | `PurchTable` | `InventJournalTable` / `InventTransferTable` / depending on scenario, document header |
| Where is the line detail? | `PurchLine` | `InventJournalTrans` / `InventTransferLine` / `InventTrans` |
| Where is the instant operation reality? | `PurchLine` + `PurchParm*` | `InventTrans` + `InventSum` |
| Where is the dimension / batch breakdown? | In `InventDimId` link inside `PurchLine` | `InventDim` and `InventBatch` |
| Where is the posting trace? | `PurchParmTable` / `PurchParmLine` | `InventJournal*`, `InventSettlement`, `InventCostTrans` |

================================================================================
                              IMPORTANT NOTES
================================================================================

1. All codes are prepared for the D365 Finance & Operations environment.
2. Demo data is based on the USMF company.
3. You need to change account numbers, item codes, warehouse, and site
   information when using in your own environment.
4. ttsBegin/ttsCommit blocks ensure database transaction integrity.
5. Test in a test environment before using in a production environment.
6. NumberSeq classes depend on number sequence settings.
7. Some codes originate from AX 2012 and have been adapted to D365 FO.

================================================================================
