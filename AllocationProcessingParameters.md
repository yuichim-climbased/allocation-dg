## 1. EXEC_* ごとの start / execute / 生成ロジック マトリクス
### 1.1. 前処理（削除・調整系）
| EXEC_* | 処理カテゴリ | start（データ取得） | execute（ビジネスロジック） | 生成ロジック・主な処理 |
|----|----|----|----|----|
| EXEC_PRE1 | 販管費削除 | getDeleteSgaAndReportSummary() | deleteSgaAndReportSummary(List<SGA__c>) | ReportSummary__c を SGA__c に紐づけて削除 → SGA__c を削除 |
| EXEC_PRE2 | 原価削除 | getDeleteCostAndReportSummary() | deleteCostAndReportSummary(List<Cost__c>) | ReportSummary__c を Cost__c に紐づけて削除 → Cost__c を削除 |
| EXEC_PRE3 | 配賦明細削除 | getDeleteAllocationDetail() | deleteAllocationDetail(List<AllocationDetail__c>) | AllocationDetail__c を削除（単純 delete） |
| EXEC_PRE4 | 帳票集計の集計コード・部門調整 | getAdjustDivisionForProject() | adjustDivisionForProject(List<ReportSummary__c>) | 当月以降案件・後原価対象の ReportSummary__c を取得し、案件/マスタに合わせて集計コード・部門を更新 |

### 1.2. 経費系
| EXEC_* | 処理カテゴリ | start（データ取得） | execute（ビジネスロジック） | 生成ロジック・主な処理 |
|----|----|----|----|----|
| EXEC_EXPENSES_JOB | 経費（JOB原価） | getExpensesJob() | allocationExpensesJob(List<ExpensesDetail__c>, BC) |logic.toCostForExpensesJob(expensesDetail) で Cost__c 生成 → insertCostAndProjectCost() で Cost__c insert ＋ insertProjectCost() |
| EXEC_EXPENSES_TOTALIZATIONCODE | 経費（集計コード原価） | getExpensesTotalizationCode() | allocationExpensesTotalizationCode(List<ExpensesDetail__c>, BC) | logic.toCostForExpensesTotalizationCode() で Cost__c 生成 → insertCostAndProjectCost() |
| EXEC_EXPENSES_COMMON | 経費（共通費 → 原価） | getExpensesCommon() | allocationExpensesCommon(List<ExpensesDetail__c>, BC) | logic.allocationTotalizationCodeEqualityFromExpensesDetail(expensesDetail, divMap) で複数 Cost__c 生成 → insertCostAndProjectCost() |
| EXEC_EXPENSES_SGA_DEPT | 経費（販管費：所属部門配賦） | getExpensesSgaDept() | allocationExpensesSgaDept(List<ExpensesDetail__c>, BC) | logic.toSgaDto(expensesDetail) → logic.allocationDepartments(sgaDto, divMap) で AllocationDetail__c 生成 → insert |
| EXEC_EXPENSES_SGA_COMPANY | 経費（販管費：全社配賦） | getExpensesSgaCompany() | allocationExpensesSgaCompany(List<ExpensesDetail__c>, BC) | logic.toSgaDto(expensesDetail) → logic.allocationEmployees(sgaDto, divMap, userSecMap) で AllocationDetail__c 生成 → insert |

### 1.3. 労務費系
| EXEC_* | 処理カテゴリ | start（データ取得） | execute（ビジネスロジック） | 生成ロジック・主な処理 |
|----|----|----|----|----|
| EXEC_LABORCOST_LABOR | 労務費配賦 | getLaborCost(type='労務費') | allocationLabor(List<LaborCost__c>, BC) | logic.toSgaDto(laborCost) → logic.allocationLabor(sgaDto, divMap, userSecMap) で AllocationDetail__c 生成 → insert |
| EXEC_LABORCOST_SGA | 労務費由来販管費（全社配賦） | getLaborCost(type='販管費') | allocationSga(List<LaborCost__c>, BC) | logic.toSgaDto(laborCost) → logic.allocationEmployees(sgaDto, divMap, userSecMap) で AllocationDetail__c 生成 → insert |

### 1.4. 材料費系
| EXEC_* | 処理カテゴリ | start（データ取得） | execute（ビジネスロジック） | 生成ロジック・主な処理 |
|----|----|----|----|----|
| EXEC_MATERIALCOST_JOB | 材料費（JOB原価） | getMaterialCostJob() | allocationMaterialCostJob(List<MaterialCost__c>, BC) | ProjectDao.findByJobNoSetCompanyCode() で案件取得 → logic.toCostForMaterialCostJob(materialCost, project) で Cost__c 生成 → insertCostAndProjectCost() |
| EXEC_MATERIALCOST_TOTALIZAIONCODE | 材料費（集計コード原価） | getMaterialCostTotalizationCode() | allocationMaterialCostTotalizationCode(List<MaterialCost__c>, BC) | TotalizationCodeDao.getTotalizationCodeByCodeSet() でマスタ取得 → logic.toCostForMaterialCostTotalizationCode(materialCost, totalizationCodeId) → insertCostAndProjectCost() |
| EXEC_MATERIALCOST_DEPT | 材料費（部門原価） | getMaterialCostDept() | allocationMaterialCostDept(List<MaterialCost__c>, BC) | logic.allocationTotalizationCodeEqualityFromMaterialCost(materialCost, divCodeMap, divMap) で Cost__c 群生成 → insertCostAndProjectCost() |

### 1.5. 振込手数料・請求書払い・仕入・部門販管費
| EXEC_* | 処理カテゴリ | start（データ取得） | execute（ビジネスロジック） | 生成ロジック・主な処理 |
|----|----|----|----|----|
| EXEC_CREDIT | 振込手数料（販管費） | getCredit() | allocationCredit(List<Credit__c>, BC) | logic.toSgaDto(credit) → logic.allocationEmployees(sgaDto, divMap, userSecMap) で AllocationDetail__c → insert |
| EXEC_PAYINVOICE_DEPT | 請求書払い（所属部門配賦） | getPayInvoiceDept() | allocationPayInvoiceDept(List<PayInvoice__c>, BC) | logic.toSgaDto(payInvoice) → logic.allocationDepartments(sgaDto, divMap) → AllocationDetail__c insert |
| EXEC_PAYINVOICE_COMPANY | 請求書払い（全社配賦） | getPayInvoiceCompany() | allocationPayInvoiceCompany(List<PayInvoice__c>, BC) | logic.toSgaDto(payInvoice) → logic.allocationEmployees(sgaDto, divMap, userSecMap) → AllocationDetail__c insert |
| EXEC_PROCUREMENTSGA | 仕入（販管費） | getProcurement() | allocationProcurement(List<Procurement__c>, BC) | logic.toSgaDto(procurement) → logic.allocationEmployees(sgaDto, divMap, userSecMap) → AllocationDetail__c insert |
| EXEC_DIVISIONSGA | 部門販管費（部門配賦） | getDivisionSga() | allocationDivisionSga(List<DivisionSGA__c>, BC) | divCodeMap で部門マスタ参照 → logic.toSgaDto(divisionSga, divisionId) → logic.allocationDepartments(sgaDto, divMap) → AllocationDetail__c insert |

### 1.6. 後処理（配賦明細→販管費、帳票集計作成）
| EXEC_* | 処理カテゴリ | start（データ取得） | execute（ビジネスロジック） | 生成ロジック・主な処理 |
|----|----|----|----|----|
| EXEC_AD_TO_SGA | 配賦明細から販管費作成 | getAllocationDetailToSga() | insertAllocationDetailToSga(List<AllocationDetail__c>, BC) | toSga(AllocationDetail__c) で SGA__c 変換 → 発生日＋科目＋部門(GL)単位で集約 → 既存 SGA__c を SGADao.getSGAByAllocationDate() で取得し加算 → upsert |
| EXEC_POST1 | 帳票集計（原価）作成 | getCost() | insertReportSummaryFromCost(List<Cost__c>) | Cost__c ごとに ReportSummary__c 生成 → insert |
| EXEC_POST2 | 帳票集計（販管費）作成 | getSga() | insertReportSummaryFromSga(List<SGA__c>) | SGA__c ごとに ReportSummary__c 生成 → insert |

## 2. データ取得条件（SOQL WHERE 条件）
### 2.1. 前処理系
#### EXEC_PRE1 – getDeleteSgaAndReportSummary（SGA__c）
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 配賦明細データ | AllocationDetail__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニーコード |
| 2 | 販管費 | SGA__c | 発生日 | CALENDAR_YEAR(AccrualDate__c) | = | 画面入力の"年"値 |
| 3 | 販管費 | SGA__c | 発生日 | CALENDAR_MONTH(AccrualDate__c) | = | 画面入力の"月"値 |
| 4 | 販管費 | SGA__c | 締め管理 | Closing__c | = | null |

```
FROM SGA__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(AccrualDate__c)     = :processYear
  AND CALENDAR_MONTH(AccrualDate__c)    = :processMonth
  AND Closing__c                        = null
```

#### EXEC_PRE2 – getDeleteCostAndReportSummary（Cost__c）
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 配賦明細データ | AllocationDetail__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニーコード |
| 2 | 原価 | Cost__c | 発生日 | CALENDAR_YEAR(AccrualDate__c) | = | 画面入力の"年"値 |
| 3 | 原価 | Cost__c | 発生日 | CALENDAR_MONTH(AccrualDate__c) | = | 画面入力の"月"値 |
| 4 | 原価 | Cost__c | 計上済フラグ | CostFlg__c | = | false |

```
FROM Cost__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(AccrualDate__c)     = :processYear
  AND CALENDAR_MONTH(AccrualDate__c)    = :processMonth
  AND CostFlg__c                        = false
```

#### EXEC_PRE3 – getDeleteAllocationDetail（AllocationDetail__c）
1 AND (2 OR (3 AND 4))
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 配賦明細データ | AllocationDetail__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニーコード |
| 2 | 配賦明細データ | AllocationDetail__c | 計上日 | YMD__c | < | ADDMONTH(配賦処理実行日, -1) |
| 3 | 配賦明細データ | AllocationDetail__c | 計上日 | CALENDAR_YEAR(YMD__c) | = | 画面入力の"年"値 |
| 4 | 配賦明細データ | AllocationDetail__c | 計上日 | CALENDAR_MONTH(YMD__c) | = | 画面入力の"月"値 |

```
FROM AllocationDetail__c
WHERE CompanyCode__c                    = :companyCode
  AND (
        YMD__c < :allocationDetailDelDate
     OR (
          CALENDAR_YEAR(YMD__c)         = :processYear
      AND CALENDAR_MONTH(YMD__c)        = :processMonth
        )
      )
```

#### EXEC_PRE4 – getAdjustDivisionForProject（ReportSummary__c）
1 AND ((2 AND 3) OR (4 AND 5) OR ((6 AND 7) OR (8 AND 9) OR 10) AND 11 AND 
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 帳票集計 | ReportSummary__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニーコード |
| 2 | 帳票集計 | ReportSummary__c | 【階層】案件.レコードタイプ | Project__r.RecordTypeId | = | RID_Project_Normal__c |
| 3 | 帳票集計 | ReportSummary__c | 【階層】案件.売上予定月 | Project__r.TargetMonth__c | = | 配賦処理実行日の1日 |
| 4 | 帳票集計 | ReportSummary__c | 【階層】案件.レコードタイプ | Project__r.RecordTypeId | = |  RID_Project_SalesTransfer__c |
| 5 | 帳票集計 | ReportSummary__c | 【階層】案件.売上予定日（振替元） | Project__r.TransferFromTargetMonth__c | >= | 配賦処理実行日の1日 |
| 6 | 帳票集計 | ReportSummary__c | 【階層】案件.レコードタイプ | Project__r.RecordTypeId | = | RID_Project_Normal__c |
| 7 | 帳票集計 | ReportSummary__c | 【階層】案件.売上予定月 | Project__r.TargetMonth__c | = | 配賦処理実行日の1日 |
| 8 | 帳票集計 | ReportSummary__c | 【階層】案件.レコードタイプ | Project__r.RecordTypeId | = |  RID_Project_SalesTransfer__c |
| 9 | 帳票集計 | ReportSummary__c | 【階層】案件.売上予定日（振替元） | Project__r.TransferFromTargetMonth__c | >= | 配賦処理実行日の1日 |
| 10 | 帳票集計 | ReportSummary__c | 【階層】案件.レコードタイプ | Project__r.RecordTypeId | = | RID_Project_Pre__c |
| 11 | 帳票集計 | ReportSummary__c | 仕入.納品予定日 | Procurement__r.appsfs__fs_DeliveryDate__c | >= | 配賦処理実行日の1日 |
| 12 | 帳票集計 | ReportSummary__c | 集計区分 | ReportSummaryKbn__c | = | 原価 |

```
FROM ReportSummary__c
WHERE CompanyCode__c                         = :companyCode
  AND (
        -- 当月以降案件・仕掛の調整対象
        (
          Project__r.RecordTypeId            = :recTypeNormal
      AND Project__r.TargetMonth__c         >= :reportSummarySearchDate
        )
     OR (
          Project__r.RecordTypeId           IN :projectRecTypeIdTransfer
      AND Project__r.TransferFromTargetMonth__c >= :reportSummarySearchDate
        )
     OR (
        -- 後原価の調整対象
        (
          Project__r.RecordTypeId            = :recTypeNormal
      AND Project__r.TargetMonth__c         < :reportSummarySearchDate
        )
      OR (
          Project__r.RecordTypeId           IN :projectRecTypeIdTransfer
      AND Project__r.TransferFromTargetMonth__c < :reportSummarySearchDate
        )
      OR  Project__r.RecordTypeId           = :recTypePre
      )
      AND Procurement__r.DeliveryDate__c    >= :reportSummarySearchDate
      AND ReportSummaryKbn__c              = :repSumKbnCost
     )
     ```

