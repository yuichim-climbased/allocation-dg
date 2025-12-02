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
