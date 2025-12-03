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
1 AND ((2 AND 3) OR (4 AND 5) OR ((6 AND 7) OR (8 AND 9) OR 10) AND 11 AND 12
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

### 2.2. 経費系（ExpensesDetail__c）
#### EXEC_EXPENSES_JOB – getExpensesJob
1 AND 2 AND 3 AND 4 AND 5
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.拡張項目2 | ExpApplyId__r.ExtraItem2__c | = | 画面で入力したカンパニー |
| 2 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"年"値 |
| 3 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"月"値 |
| 4 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.ステータス | ExpApplyId__r.Status__c | = | 承認済み or 精算済み or 確定済み |
| 5 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.拡張項目3 | ExpApplyId__r.ExtraItem3__c | != | null |

```
FROM AtkEmpExp__c
WHERE ExpApplyId__r.ExtraItem2__c                     = :companyCode
  AND CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c)  = :processYear
  AND CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) = :processMonth
  AND ExpApplyId__r.Status__c      IN :targetStatusList
  AND ExpApplyId__r.ExtraItem3__c                       != null
```

#### EXEC_EXPENSES_TOTALIZATIONCODE – getExpensesTotalizationCode
1 AND 2 AND 3 AND 4 AND 5
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.拡張項目2 | ExpApplyId__r.ExtraItem2__c | = | 画面で入力したカンパニー |
| 2 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"年"値 |
| 3 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"月"値 |
| 4 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.ステータス | ExpApplyId__r.Status__c | = | 承認済み or 精算済み or 確定済み |
| 5 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.拡張項目3 | ExpApplyId__r.ExtraItem1__c | != | null |

```
FROM AtkEmpExp__c
WHERE ExpApplyId__r.ExtraItem2__c                     = :companyCode
  AND CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c)  = :processYear
  AND CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) = :processMonth
  AND ExpApplyId__r.Status__c      IN :targetStatusList
  AND ExpApplyId__r.ExtraItem1__c                       != null
```

#### EXEC_EXPENSES_COMMON – getExpensesCommon
1 AND 2 AND 3 AND 4 AND 5
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.拡張項目2 | ExpApplyId__r.ExtraItem2__c | = | 画面で入力したカンパニー |
| 2 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"年"値 |
| 3 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"月"値 |
| 4 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.ステータス | ExpApplyId__r.Status__c | = | 承認済み or 精算済み or 確定済み |
| 5 | 勤怠交通費 | AtkEmpExp__c | 費目.計上方法 | ExpItemId__r.AccountingMethod__c | = | 共通費 |

```
FROM AtkEmpExp__c
WHERE ExpApplyId__r.ExtraItem2__c                     = :companyCode
  AND CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c)  = :processYear
  AND CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) = :processMonth
  AND ExpApplyId__r.Status__c      IN :targetStatusList
  AND ExpItemId__r.AccountingMethod__c                  = '共通費'
```

#### EXEC_EXPENSES_SGA_DEPT – getExpensesSgaDept
1 AND 2 AND 3 AND 4 AND 5
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.拡張項目2 | ExpApplyId__r.ExtraItem2__c | = | 画面で入力したカンパニー |
| 2 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"年"値 |
| 3 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.#計上月末日 | CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) | = | 画面入力の"月"値 |
| 4 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.ステータス | ExpApplyId__r.Status__c | = | 承認済み or 精算済み or 確定済み |
| 5 | 勤怠交通費 | AtkEmpExp__c | 費目.計上方法 | ExpItemId__r.AccountingMethod__c | = | 販管費 |

```
FROM AtkEmpExp__c
WHERE ExpApplyId__r.ExtraItem2__c                     = :companyCode
  AND CALENDAR_YEAR(ExpApplyId__r.PostMonthEndDate__c)  = :processYear
  AND CALENDAR_MONTH(ExpApplyId__r.PostMonthEndDate__c) = :processMonth
  AND ExpApplyId__r.Status__c      IN :targetStatusList
  AND ExpItemId__r.AccountingMethod__c                  = '販管費'
```

#### EXEC_EXPENSES_SGA_COMPANY – getExpensesSgaCompany
"EXEC_EXPENSES_SGA_DEPT – getExpensesSgaDept"と同じ。

### 2.3. 労務費系（LaborCost__c）
#### EXEC_LABORCOST_LABOR / EXEC_LABORCOST_SGA – getLaborCost(type)
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 労務費 | LaborCost__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 労務費 | LaborCost__c | 発生日 | CALENDAR_YEAR(AccrualDate__c) | = | 画面入力の"年"値 |
| 3 | 労務費 | LaborCost__c | 発生日 | CALENDAR_MONTH(AccrualDate__c) | = | 画面入力の"月"値 |
| 4 | 労務費 | LaborCost__c | 区分 | Type__c | = | 労務費 or 販管費 |

```
FROM LaborCost__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(AccrualDate__c)     = :processYear
  AND CALENDAR_MONTH(AccrualDate__c)    = :processMonth
  AND Type__c                           = :type   -- '労務費' or '販管費'
```

### 2.4. 材料費系（MaterialCost__c）
#### EXEC_MATERIALCOST_JOB – getMaterialCostJob
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 材料費 | MaterialCost__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 材料費 | MaterialCost__c | 発生日 | CALENDAR_YEAR(AccrualDate__c) | = | 画面入力の"年"値 |
| 3 | 材料費 | MaterialCost__c | 発生日 | CALENDAR_MONTH(AccrualDate__c) | = | 画面入力の"年"値 |
| 4 | 材料費 | MaterialCost__c | JOBNo. | JOBNo__c | != | null |

```
FROM MaterialCost__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(AccrualDate__c)     = :processYear
  AND CALENDAR_MONTH(AccrualDate__c)    = :processMonth
  AND JOBNo__c                          != null
```

#### EXEC_MATERIALCOST_TOTALIZAIONCODE – getMaterialCostTotalizationCode
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 材料費 | MaterialCost__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 材料費 | MaterialCost__c | 発生日 | CALENDAR_YEAR(AccrualDate__c) | = | 画面入力の"年"値 |
| 3 | 材料費 | MaterialCost__c | 発生日 | CALENDAR_MONTH(AccrualDate__c) | = | 画面入力の"年"値 |
| 4 | 材料費 | MaterialCost__c | 集計コード | TotalizationCode__c | != | null |

```
FROM MaterialCost__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(AccrualDate__c)     = :processYear
  AND CALENDAR_MONTH(AccrualDate__c)    = :processMonth
  AND TotalizationCode__c               != null
```

#### EXEC_MATERIALCOST_DEPT – getMaterialCostDept
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 材料費 | MaterialCost__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 材料費 | MaterialCost__c | 発生日 | CALENDAR_YEAR(AccrualDate__c) | = | 画面入力の"年"値 |
| 3 | 材料費 | MaterialCost__c | 発生日 | CALENDAR_MONTH(AccrualDate__c) | = | 画面入力の"年"値 |
| 4 | 材料費 | MaterialCost__c | 部門コード | DeptCode__c | != | null |

```
FROM MaterialCost__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(AccrualDate__c)     = :processYear
  AND CALENDAR_MONTH(AccrualDate__c)    = :processMonth
  AND DeptCode__c                       != null
```

### 2.5. 振込手数料・請求書払い・仕入・部門販管費
#### EXEC_CREDIT – getCredit（Credit__c）
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 入金 | Credit__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 入金 | Credit__c | 入金日 | CALENDAR_YEAR(CreditDate__c) | = | 画面入力の"年"値 |
| 3 | 入金 | Credit__c | 入金日 | CALENDAR_MONTH(AccrualDate__c) | = | 画面入力の"年"値 |
| 4 | 入金 | Credit__c | 差額相殺区分 | DiffOffsetType__c | != | null |

```
FROM Credit__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(CreditDate__c)      = :processYear
  AND CALENDAR_MONTH(CreditDate__c)     = :processMonth
  AND DiffOffsetType__c                 != null
```

#### EXEC_PAYINVOICE_DEPT – getPayInvoiceDept（PayInvoice__c）
1 AND 2 AND 3 AND 4 AND 5
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.拡張項目2 | ExpApplyId__r.ExtraItem2__c | = | 画面で入力したカンパニー |
| 2 | 勤怠交通費 | AtkEmpExp__c | 日付 | CALENDAR_YEAR(Date__c) | = | 画面入力の"年"値 |
| 3 | 勤怠交通費 | AtkEmpExp__c | 日付 | CALENDAR_MONTH(Date__c) | = | 画面入力の"月"値 |
| 4 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.ステータス | ExpApplyId__r.Status__c | = | 承認済み or 精算済み or 確定済み |
| 5 | 勤怠交通費 | AtkEmpExp__c | 交通費申請.精算区分 | ExpItemId__r.ExpenseType__c | = | 請求書払い |

```
FROM AtkEmpExp__c
WHERE ExpApplyId__r.ExtraItem2__c                     = :companyCode
  AND CALENDAR_YEAR(Date__c)  = :processYear
  AND CALENDAR_MONTH(Date__c) = :processMonth
  AND ExpApplyId__r.Status__c      IN :targetStatusList
  AND ExpItemId__r.ExpenseType__c                     = '請求書払い'
```

#### EXEC_PAYINVOICE_COMPANY – getPayInvoiceCompany（PayInvoice__c）
"EXEC_PAYINVOICE_DEPT – getPayInvoiceDept（PayInvoice__c）"と同じ

#### EXEC_PROCUREMENTSGA – getProcurement（Procurement__c）
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 仕入 | appsfs__InboundDelivery__c | #カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 仕入 | appsfs__InboundDelivery__c | 納品予定日 | CALENDAR_YEAR(appsfs__fs_DeliveryDate__c) | = | 画面入力の"年"値 |
| 3 | 仕入 | appsfs__InboundDelivery__c | 納品予定日 | CALENDAR_MONTH(appsfs__fs_DeliveryDate__c) | = | 画面入力の"月"値 |
| 4 | 仕入 | appsfs__InboundDelivery__c | レコードタイプ | RecordTypeId | = | RID_Proc_Sga |

```
FROM appsfs__InboundDelivery__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(appsfs__fs_DeliveryDate__c)    = :processYear
  AND CALENDAR_MONTH(appsfs__fs_DeliveryDate__c)   = :processMonth
  AND RecordTypeId                      = :sgaRecordTypeId
```

#### EXEC_DIVISIONSGA – getDivisionSga（DivisionSGA__c）
1 AND 2 AND 3
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 部門販管費 | DivisionSGA__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 部門販管費 | DivisionSGA__c | 計上日 | CALENDAR_YEAR(YMD__c) | = | 画面入力の"年"値 |
| 3 | 部門販管費 | DivisionSGA__c | 計上日 | CALENDAR_MONTH(YMD__c) | = | 画面入力の"月"値 |

```
FROM DivisionSGA__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(YMD__c)             = :processYear
  AND CALENDAR_MONTH(YMD__c)            = :processMonth
```

### 2.6. 配賦明細→販管費・集計作成
#### EXEC_AD_TO_SGA – getAllocationDetailToSga（AllocationDetail__c）
1 AND 2 AND 3
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 配賦明細データ | AllocationDetail__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
| 2 | 配賦明細データ | AllocationDetail__c | 計上日 | CALENDAR_YEAR(YMD__c) | = | 画面入力の"年"値 |
| 3 | 配賦明細データ | AllocationDetail__c | 計上日 | CALENDAR_MONTH(YMD__c) | = | 画面入力の"月"値 |

```
FROM AllocationDetail__c
WHERE CompanyCode__c                    = :companyCode
  AND CALENDAR_YEAR(YMD__c)             = :processYear
  AND CALENDAR_MONTH(YMD__c)            = :processMonth
```

#### EXEC_POST1 – getCost（Cost__c）
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 原価 | Cost__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
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

#### EXEC_POST2 – getSga（SGA__c）
1 AND 2 AND 3 AND 4
| # | オブジェクト | API参照名 | 項目名 | API参照名 | 演算子 | 値 |
|----|----|----|----|----|----|----|
| 1 | 販管費 | SGA__c | カンパニーコード | CompanyCode__c | = | 画面で入力したカンパニー |
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

## 3. データ生成・更新
### 3.1. 配賦明細｜生成（AllocationDetail__c）
- DML

```
// 各種 allocationXXX(...) 内で生成された List<AllocationDetail__c> を insert
insert insertAllocationDetailList;
```

- 生成時に設定される項目（共通）
> いずれも AllocationLogic.createAllocationDetail(...) 内でセット
1.  配賦明細

| # | 項目名 | 設定値（ソース） | 備考 |
|----|----|----|----|
| 1 | CompanyCode__c | companyCode（バッチ全体で共通の会社コード） | 全 EXEC_* 共通 |
| 2 | YMD__c | sgaDto.accrualDate | 発生日 |
| 3 | AccountsCode__c | sgaDto.accountsCode | 勘定科目コード |
| 4 | DivisionGL__c | glDivision.Id | 部門（GL）ID |
| 5 | DivisionCodeGL__c | glDivision.Name | 部門（GL）コード |
| 6 | DivisionNameGL__c | glDivision.DivisionName__c | 部門（GL）名 |
| 7 | DivisionGM__c | glDivision.ParentSection__c | 部門（GM）ID |
| 8 | DivisionCodeGM__c | glDivision.ParentSection__r.Name | 部門（GM）コード |
| 9 | DivisionNameGM__c | glDivision.ParentSection__r.DivisionName__c | 部門（GM）名 |
| 10 | DivisionDept__c | glDivision.ParentSection__r.ParentSection__c | 部門（部）ID |
| 11 | DivisionCodeDept__c | glDivision.ParentSection__r.ParentSection__r.Name | 部門（部）コード |
| 12 | DivisionNameDept__c | glDivision.ParentSection__r.ParentSection__r.DivisionName__c | 部門（部）名 |
| 13 | DivisionHQ__c | glDivision.ParentSection__r.ParentSection__r.ParentSection__c | 部門（本部）ID |
| 14 | DivisionCodeHQ__c | glDivision.ParentSection__r.ParentSection__r.ParentSection__r.Name | 部門（本部）コード |
| 15 | DivisionNameHQ__c | glDivision.ParentSection__r.ParentSection__r.ParentSection__r.DivisionName__c | 部門（本部）名 |
| 16 | Amount__c | amount | GL・ユーザ・部門への配賦後の金額 |
| 17 | AllocationType__c | allocationTypeMap.get(processCount) | EXEC_* ごとにマスタから種別を引く |
| 18 | SourceObjId__c | sgaDto.sourceId | 元オブジェクトID（経費・労務費・請求書・仕入など） |
| 19 | SourceAmount__c | sgaDto.amount | 元オブジェクトの金額 |

- 元オブジェクト参照項目（EXEC_* により分岐）
> createAllocationDetail 内の processCount 分岐

| # | 項目名 | 設定値（ソース） | 適用 EXEC_* |
|----|----|----|----|
| 1 | ExpensesDetail__c | allocationDetail.SourceObjId__c | EXEC_EXPENSES_SGA_DEPT, EXEC_EXPENSES_SGA_COMPANY |
| 2 | LaborCost__c | allocationDetail.SourceObjId__c | EXEC_LABORCOST_LABOR, EXEC_LABORCOST_SGA |
| 3 | ExpensesDetail__c | allocationDetail.SourceObjId__c | EXEC_PAYINVOICE_DEPT, EXEC_PAYINVOICE_COMPANY |
| 4 | Credit__c | allocationDetail.SourceObjId__c | EXEC_CREDIT |
| 5 | Procurement__c | allocationDetail.SourceObjId__c | EXEC_PROCUREMENTSGA |
| 6 | DivisionSGA__c | allocationDetail.SourceObjId__c | EXEC_DIVISIONSGA |

- 配賦ターゲット（ユーザ or 部門）の項目
> createAllocationDetail の末尾で、userSection の有無に応じて分岐
- ユーザ配賦（全社配賦・ユーザ配賦系 EXEC の場合）

| # | 項目名 | 設定値（ソース） | 備考 |
|----|----|----|----|
| 1 | TargetUserSection__c | userSection.Id | ユーザ所属部門 |
| 2 | TargetUserEmpCode__c | userSection.UserId__r.EmpCode__c | 社員コード |
| 3 | TargetUserSectionCode__c | userSection.Section__r.Name | 所属部門コード |
| 4 | TargetUserSectionName__c | userSection.Section__r.DivisionName__c | 所属部門名 |
| 5 | TargetUserSectionPercent__c | userSection.Percent__c | 部門按分率 |
| 6 | TargetUserSectionAmount__c | userAmount | ユーザ別金額 |

- 部門配賦（deptDivisionId が設定されている場合）

| # | 項目名 | 設定値（ソース） | 備考 |
|----|----|----|----|
| 1 | TargetDivision__c | sgaDto.deptDivisionId | 配賦先「部」の Division__c |
| 2 | TargetDivisionCode__c | companyDivisionMap.get(TargetDivision__c).Name | コード |
| 3 | TargetDivisionName__c | companyDivisionMap.get(TargetDivision__c).DivisionName__c | 名称 |
| 4 | TargetDivisionAmount__c | sgaDto.amount | 部門単位での配賦元金額 |

### 3.2. 原価｜生成（Cost__c）
- DML

```
// 経費・材料費から生成された costList の insert
insert costList;
```

- 生成パターン
    1.  経費(JOB)      → toCostForExpensesJob
    2.  経費(集計コード) → toCostForExpensesTotalizationCode
    3.  材料費(JOB)    → toCostForMaterialCostJob
    4.  材料費(集計コード) → toCostForMaterialCostTotalizationCode
    5.  経費／材料費の集計コード均等配賦 → toCostForTotalizationCodeEquality

- 生成時に設定される項目（共通）

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | CompanyCode__c | companyCode |
| 2 | AccrualDate__c | 経費 or 材料費の計上日 |
| 3 | SalesCostDate__c | 案件情報＋ allocationDate による計上期判定 |
| 4 | Amount__c | 経費 or 材料費の金額（または配賦後金額） |
| 5 | AccountsCode__c | 経費科目 or 材料費科目の勘定科目コード |
| 6 | TotalizationCode__c | 案件 or 経費／材料費に紐づく集計コード |
| 7 | Project__c | JOB 原価の場合は案件 ID、集計コード原価の場合は null |
| 8 | AtkEmpExp__c | 元の明細ID |
| 9 | MaterialCost__c | 元の明細ID |

- パターン別に設定される項目
  1. 経費(JOB) → Cost（toCostForExpensesJob）

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | AtkEmpExp__c | AtkEmpExp__c.Id |
| 2 | AccrualDate__c | AtkEmpExp__r.ExpApplyId__r.PostMonthEndDate__c |
| 3 | Amount__c | AtkEmpExp__r.WithoutTax__c |
| 4 | AccountsCode__c | AtkEmpExp__r.ExpItemId__r.Code__c |
| 5 | SalesCostDate__c | Project.RecordType や TargetMonth__c / TransferFromTargetMonth__c と allocationDate から判定した日付 |

  2. 経費(集計コード) → Cost（toCostForExpensesTotalizationCode）

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | AtkEmpExp__c | AtkEmpExp__c.Id |
| 2 | AccrualDate__c | AtkEmpExp__r.ExpApplyId__r.PostMonthEndDate__c |
| 3 | SalesCostDate__c | allocationDate |
| 4 | Amount__c | AtkEmpExp__r.WithoutTax__c |
| 5 | AccountsCode__c | AtkEmpExp__r.ExpItemId__r.Code__c |
| 6 | Project__c | null |

  3. 材料費(JOB) → Cost（toCostForMaterialCostJob）

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | AccrualDate__c | materialCost.AccrualDate__c |
| 2 | Amount__c | materialCost.Amount__c |
| 3 | AccountsCode__c | materialCost.AccountsCode__c |
| 4 | SalesCostDate__c | Project.RecordType や TargetMonth__c / TransferFromTargetMonth__c と allocationDate から判定した日付 |
| 5 | MaterialCost__c | materialCost.Id |
| 6 | TotalizationCode__c | project.TotalizationCode__c |
| 7 | Project__c | project.Id |

  4. 材料費(集計コード) → Cost（toCostForMaterialCostTotalizationCode）

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | AccrualDate__c | materialCost.AccrualDate__c |
| 2 | Amount__c | materialCost.Amount__c |
| 3 | AccountsCode__c | materialCost.AccountsCode__c |
| 4 | SalesCostDate__c | allocationDate |
| 5 | MaterialCost__c | materialCost.Id |
| 6 | TotalizationCode__c | totalizationCodeId（バッチ側で取得したマスタの ID） |
| 7 | Project__c | null |

  5. 集計コード均等配賦 → Cost（toCostForTotalizationCodeEquality）

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | AtkEmpExp__c | atcDto.expensesDetailId（あれば） |
| 2 | MaterialCost__c | atcDto.materialCostId（ExpensesDetail が空の場合） |
| 3 | TotalizationCode__c | totalizationCodeId |
| 4 | SalesCostDate__c | allocationDate |
| 5 | AccountsCode__c | atcDto.accountsCode |
| 6 | AccrualDate__c | atcDto.accrualDate |
| 7 | Amount__c | 分割後の amount |

### 3.3. 販管費｜生成・更新（SGA__c）
  1. 生成（AllocationDetail → SGA__c）
  - DML

```
// 配賦明細から生成された sga をマージしたうえで upsert
upsert upsertSgaList;
```

新規作成時は toSga(allocationDetail) の値がそのまま insert される。
 - toSga(AllocationDetail__c allocationDetail) で設定される項目

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | CompanyCode__c | allocationDetail.CompanyCode__c |
| 2 | AccrualDate__c | allocationDetail.YMD__c |
| 3 | AccountsCode__c | allocationDetail.AccountsCode__c |
| 4 | Division_GL__c | allocationDetail.DivisionGL__c |
| 5 | Amount__c | allocationDetail.Amount__c |

※ SummaryKey__c はコード上では設定しておらず、別ロジック（式項目やトリガ）で生成される。
  2. 更新（集約時の Amount 加算）
insertAllocationDetailToSga 内で、同一キー（SummaryKey）を持つ 既存 SGA がある場合に更新される。

| # | 項目名 | 更新値 | 備考 |
|----|----|----|----|
| 1 | Amount__c | 既存 sga.Amount__c + mergeSgaMap[key].Amount__c | 同一 (CompanyCode, AccrualDate, AccountsCode, Division_GL) などで集約 |

それ以外の項目（CompanyCode__c / AccountsCode__c / Division_GL__c / AccrualDate__c など）は 既存レコードの値を維持 したまま upsert される。

### 3.4. 帳票集計｜生成・更新（ReportSummary__c）
  1. 生成（Cost → ReportSummary）
  - DML

```
insert insertReportSummaryList;
```

  - insertReportSummaryFromCost で設定される項目

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | CompanyCode__c | allocationDetail.CompanyCode__c |
| 2 | YMD__c | existCost.SalesCostDate__c |
| 3 | CostAmount__c | existCost.Amount__c |
| 4 | Cost__c | existCost.Id |
| 5 | Project__c | existCost.Project__c |
| 6 | TotalizationCode__c | existCost.TotalizationCode__c |
| 7 | Division_GL__c | existCost.TotalizationCode__r.Division__c |
| 8 | Division_GM__c | existCost.TotalizationCode__r.Division__r.ParentSection__c |
| 9 | Division_EM__c | existCost.TotalizationCode__r.Division__r.ParentSection__r.ParentSection__c |
| 10 | Division_DD__c | existCost.TotalizationCode__r.Division__r.ParentSection__r.ParentSection__r.ParentSection__c |
| 11 | RecordTypeId | RID_ReportSummary_Normal__c |
| 12 | ReportSummaryKbn__c | '原価(経費)' or '原価(材料費)'（Cost の元明細により判定） |

  2. 生成（SGA → ReportSummary）
  - DML

```
insert insertReportSummaryList;
```

  - insertReportSummaryFromSga で設定される項目

| # | 項目名 | 設定値（ソース） |
|----|----|----|
| 1 | CompanyCode__c | allocationDetail.CompanyCode__c |
| 2 | YMD__c | existCost.SalesCostDate__c |
| 3 | SgaAmount__c | existSga.Amount__c |
| 4 | SGA__c | existSga.Id |
| 5 | Division_GL__c | existSga.Division_GL__c |
| 6 | Division_GM__c | existSga.Division_GL__r.ParentSection__c |
| 7 | Division_EM__c | existSga.Division_GL__r.ParentSection__r.ParentSection__c |
| 8 | Division_DD__c | existSga.Division_GL__r.ParentSection__r.ParentSection__r.ParentSection__c |
| 9 | RecordTypeId | RID_ReportSummary_Normal__c |
| 10 | ReportSummaryKbn__c | '販管費' |

  3. 更新（案件マスタにあわせた集計コード・部門調整）
  - DML

```
update updateList;
```

adjustDivisionForProject(List<ReportSummary__c> inObjList) 内で、「案件側の集計コード・部門」と現在の帳票集計の値が違うときに更新される。
  - 更新される項目

| # | 項目名 | 更新値 | 備考 |
|----|----|----|----|
| 1 | Id | obj.Id | 更新対象のレコード ID |
| 2 | TotalizationCode__c | obj.Project__r.TotalizationCode__c | 案件マスタ側の集計コード |
| 3 | Division_GL__c | obj.Project__r.TotalizationCode__r.Division__c | 部門GL |
| 4 | Division_GM__c | obj.Project__r.TotalizationCode__r.Division__r.ParentSection__c | 部門GM |
| 5 | Division_EM__c | obj.Project__r.TotalizationCode__r.Division__r.ParentSection__r.ParentSection__c | 部門EM |
| 6 | Division_DD__c | obj.Project__r.TotalizationCode__r.Division__r.ParentSection__r.ParentSection__r.ParentSection__c | 部門DD |
| 7 | IsSyncProject__c | true（条件付き） | obj.ReportSummaryKbn__c == '評価損' の場合のみ ON |

