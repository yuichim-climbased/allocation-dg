## 1. システムの概要
### 1.1. 目的

本バッチクラス BatchAllocation は、Salesforce 上の各種コストデータを元に「配賦処理」を行い、以下を実現する月次バッチです。
-	経費・労務費・材料費・入金（振込手数料）・請求書払い・仕入・部門販管費などを配賦ロジックに基づいて各案件・部門へ按分
-	中間テーブル（AllocationDetail__c）に配賦明細を作成
-	原価（Cost__c）・販管費（SGA__c）・帳票集計（ReportSummary__c）を連鎖的に生成
-	月次毎に不要となる原価/販管費/配賦明細/帳票集計の削除・再生成
-	処理結果のログ（AllocationLog__c）とエラーメール通知

※既存バッチ BatchCostAllocation を全面改修しつつ、外部インターフェース（OUTPUT）は踏襲。

### 1.2. 処理単位・前提
-	処理単位
    -	会社コード（companyCode）
    -	配賦処理日（allocationDate）
    -	プロセス番号（processCount）
-	起動タイミング
    -	月次締め処理画面などから Database.executeBatch(new BatchAllocation(...)) により起動される想定
-	処理フロー
    -	processCount を 0 から EXEC_POST2 まで順次インクリメントしながら、バッチを「多段チェーン」させる構造
    -	各段階で
        -	事前削除処理（PRE）
        -	各種コスト/経費/販管費/仕入/入金の配賦処理（EXPENSES/LABOR/MATERIAL/CREDIT/PAYINVOICE/PROCUREMENT/DIVISIONSGA）
        -	配賦明細から販管費作成（EXEC_AD_TO_SGA）
        -	原価・販管費から帳票集計作成（EXEC_POST1 / EXEC_POST2）
    -	最終プロセス完了時に AllocationLog__c へ成功ログを記録し、MonthlyClosingLogic 側の処理状態を完了に更新

## 2. モジュール構成と各機能の説明
### 2.1. クラス全体

```
public with sharing class BatchAllocation
  implements Database.Batchable<sObject>, Database.Stateful
```

-	役割
-	Batch Apex による月次配賦処理の本体クラス
-	Database.Batchable<sObject> 実装により start, execute, finish を提供
-	Database.Stateful によりバッチ実行中のメンバ変数（errors, マスタキャッシュなど）を跨ぎ保持

### 2.2. 定数（プロセス定義）

```
public static final Integer EXEC_PRE1 = 0;         // 販管費削除
public static final Integer EXEC_PRE2 = ...;       // 原価削除
public static final Integer EXEC_PRE3 = ...;       // 配賦明細削除
public static final Integer EXEC_PRE4 = ...;       // 帳票集計調整
public static final Integer EXEC_EXPENSES_JOB = ...;           // 経費(JOB)
...
public static final Integer EXEC_POST2 = ...;      // 帳票集計（販管費）
```

- 	各プロセスに連番を割り振り、processCount に応じて処理を切り替える。
-	EXECUTEBATCH_START, EXECUTEBATCH_NUMMAX で開始・終了プロセスを定義。

### 2.3. 状態変数・DTO/マスタキャッシュ
-	allocationDate, processYear, processMonth
    -	処理対象年月の基準
-	allocationDetailDelDate
    -	配賦明細削除対象の年月（保持月数ラベルから算出）
-	processCount
    -	現在のプロセス番号
-	executeCount
    -	execute() 呼び出し回数（ロギング/チューニング用）
-	companyCode
    -	対象会社コード
-	batchExecuteNum
    -	1トランザクションあたりの処理件数（プロセス毎に可変）
-	errors : List<Exception>
    -	各 execute() 内で発生した例外を溜め、finish() でまとめて扱う
-	divMap : Map<Id, DivisionDto>
    -	配賦対象部門の DTO キャッシュ
-	divCodeMap : Map<String, Division__c>
    -	部門コード → 部門オブジェクト
-	userSecMap : Map<String, List<UserSection__c>>
    -	従業員コード → 所属部門リスト
-	targetUserStatus, exclusionEmpStatus
    -	労務費配賦時のユーザ抽出条件
-	logic : AllocationLogic
    -	配賦ロジックをカプセル化したサービスクラス
    -	経費・材料費・労務費・入金などを Cost/SGA/AllocationDetail へ変換
-	monthlylogic : MonthlyClosingLogic
    -	月次締め処理の同時実行制御・状態更新用ロジック

### 2.4. コンストラクタ

```
public BatchAllocation(Date allocationDate, Integer processCount, String companyCode)
```

主な処理
1.	batchExecuteNum をラベル BatchCostAllocateExeNum から取得（失敗時は AppConst.DEFAULT_BATCHCOSTALLNUM）
2.	日付・年月・削除対象年月を算出
3.	processCount, companyCode を保持
4.	AllocationLogic, MonthlyClosingLogic のインスタンス生成
5.	プロセス毎に必要なマスタや DTO を事前取得
    -	DivisionDao.getDivisionDtoMap(...)
    -	DivisionDao.getDivisionCodeMapByCompanyCode(...)
    -	UserSectionDao.getUserSectionMap(...)
    -	etc.

→ 各 getXXX() / allocationXXX() メソッドで何度も SOQL を発行しないように、プロセス単位で必要マスタを先読み・キャッシュする設計。

### 2.5. Batchable インタフェース
#### 2.5.1. start(Database.BatchableContext)
-	processCount に応じて取得クエリを切り替え、Database.QueryLocator を返却

主な分岐：
-	PRE 系
    -	EXEC_PRE1 : getDeleteSgaAndReportSummary()
    -	EXEC_PRE2 : getDeleteCostAndReportSummary()
    -	EXEC_PRE3 : getDeleteAllocationDetail()
    -	EXEC_PRE4 : getAdjustDivisionForProject()
-	経費系
    -	EXEC_EXPENSES_JOB : getExpensesJob()
    -	EXEC_EXPENSES_TOTALIZATIONCODE : getExpensesTotalizationCode()
    -	EXEC_EXPENSES_COMMON : getExpensesCommon()
    -	EXEC_EXPENSES_SGA_DEPT : getExpensesSgaDept()
    -	EXEC_EXPENSES_SGA_COMPANY : getExpensesSgaCompany()
-	労務費/販管費
    -	EXEC_LABORCOST_LABOR : getLaborCost('労務費')
    -	EXEC_LABORCOST_SGA : getLaborCost('販管費')
-	材料費
    -	EXEC_MATERIALCOST_JOB : getMaterialCostJob()
    -	EXEC_MATERIALCOST_TOTALIZAIONCODE : getMaterialCostTotalizationCode()
    -	EXEC_MATERIALCOST_DEPT : getMaterialCostDept()
-	入金/請求書/仕入
    -	EXEC_CREDIT : getCredit()
    -	EXEC_PAYINVOICE_DEPT : getPayInvoiceDept()
    -	EXEC_PAYINVOICE_COMPANY : getPayInvoiceCompany()
    -	EXEC_PROCUREMENTSGA : getProcurement()
-	部門販管費/配賦明細/帳票集計
    -	EXEC_DIVISIONSGA : getDivisionSga()
    -	EXEC_AD_TO_SGA : getAllocationDetailToSga()
    -	EXEC_POST1 : getCost()
    -	EXEC_POST2 : getSga()

#### 2.5.2. execute(Database.BatchableContext, List<sObject> scope)
-	scope を対象オブジェクト型にキャストし、processCount ごとに専用処理へ委譲
-	例外は catch して errors に蓄積（バッチ終了時に一括扱い）

代表例：
-	PRE 系削除
    -	deleteSgaAndReportSummary(List<SGA__c>)
    -	deleteCostAndReportSummary(List<Cost__c>)
    -	deleteAllocationDetail(List<AllocationDetail__c>)
    -	adjustDivisionForProject(List<ReportSummary__c>)
-	経費配賦
    -	allocationExpensesJob(List<ExpensesDetail__c>, BC)
    -	allocationExpensesTotalizationCode(...)
    -	allocationExpensesCommon(...)
    -	allocationExpensesSgaDept(...)
    -	allocationExpensesSgaCompany(...)
-	労務費/販管費配賦
    -	allocationLabor(List<LaborCost__c>, BC)
    -	allocationSga(List<LaborCost__c>, BC)
-	材料費配賦
    -	allocationMaterialCostJob(List<MaterialCost__c>, BC)
    -	allocationMaterialCostTotalizationCode(...)
    -	allocationMaterialCostDept(...)
-	入金/請求書/仕入配賦
    -	allocationCredit(List<Credit__c>, BC)
    -	allocationPayInvoiceDept(List<PayInvoice__c>, BC)
    -	allocationPayInvoiceCompany(List<PayInvoice__c>, BC)
    -	allocationProcurement(List<Procurement__c>, BC)
-	部門販管費/配賦明細から販管費
    -	allocationDivisionSga(List<DivisionSGA__c>, BC)
    -	insertAllocationDetailToSga(List<AllocationDetail__c>, BC)
-	帳票集計作成
    -	insertReportSummaryFromCost(List<Cost__c>)
    -	insertReportSummaryFromSga(List<SGA__c>)

#### 2.5.3. finish(Database.BatchableContext)
1.	エラー有りの場合
	システム管理者向けメール送信（カスタムラベル BatchCostAllocateErrorMailSend）
    -	AllocationLog__c にエラー内容を HTML 改行付きで保存
    -	monthlylogic.setProcessingLogToCompleted() により月次処理状態を完了へ
2.	エラー無しの場合
    -	processCount < EXECUTEBATCH_NUMMAX
→ 次プロセスの BatchAllocation を Database.executeBatch で起動
    -	次プロセスの種類に応じて nextBatchExecutNum（1バッチの処理件数）を増減
        -	PRE/POST 系は 2000 件
        -	全社配賦系は 10 件など
    -	processCount == EXECUTEBATCH_NUMMAX
→ 最終プロセス:
        -	AllocationLog__c に成功ログを書き込み
        -	monthlylogic.setProcessingLogToCompleted() を呼び出し

### 2.6. ドメイン別の主な処理モジュール
#### 2.6.1. PRE 系（事前削除・調整）
-	getDeleteSgaAndReportSummary / deleteSgaAndReportSummary
    -	当月未締めの SGA__c と紐づく ReportSummary__c を削除
-	getDeleteCostAndReportSummary / deleteCostAndReportSummary
    -	CostFlg__c=false の原価と、その Cost__c を参照する帳票集計を削除
-	getDeleteAllocationDetail / deleteAllocationDetail
    -	保持月数を過ぎた配賦明細、あるいは当月分を削除
-	getAdjustDivisionForProject / adjustDivisionForProject
    -	当月以降案件／後原価の帳票集計について、案件・集計コードマスタの最新情報で部門・集計コードを更新

#### 2.6.2. 経費配賦（Expenses）
-	JOB/集計コード/共通費/販管費（部門・全社）を対象に、ExpensesDetail__c を取得し、以下を実施：
    -	原価へ変換 → Cost__c + ProjectCost__c
    -	あるいは SgaDto へ変換し、AllocationLogic 経由で AllocationDetail__c を生成（部門配賦 or 全社配賦）

#### 2.6.3. 労務費/販管費配賦（LaborCost）
-	getLaborCost で LaborCost__c 取得
-	allocationLabor / allocationSga
    -	LaborCost__c → SgaDto → 部門/全社配賦で AllocationDetail__c 生成

#### 2.6.4. 材料費配賦（MaterialCost）
-	JOB/集計コード/部門ごとに対象 MaterialCost__c を抽出
-	JOB:
    -	JOBNo → ProjectDao.findByJobNoSetCompanyCode → Project__c を取得
    -	logic.toCostForMaterialCostJob() により原価化
-	集計コード・部門:
    -	TotalizationCodeDao, DivisionDao からマスタ取得
    -	AllocationLogic にて均等按分・コード変換等のロジックを委譲

#### 2.6.5. 入金・請求書・仕入配賦
-	Credit__c, PayInvoice__c, Procurement__c を取得し、SgaDto へ変換
-	部門/全社配賦ロジックで AllocationDetail__c を生成

#### 2.6.6. 部門販管費配賦（DivisionSGA）
-	DivisionSGA__c を取得し、部門コードから Division__c を引く
-	SgaDto へ変換し、部門配賦で AllocationDetail__c を生成

#### 2.6.7. 配賦明細 → 販管費集約（AllocationDetail→SGA）
-	getAllocationDetailToSga
    -	当月分の配賦明細取得
-	insertAllocationDetailToSga
    -	AllocationDetail__c → SGA__c へ変換 (toSga)
    -	「発生日 + 科目コード + 部門(GL)」単位で金額を集約
    -	既存 SGA__c とマージ（SummaryKey__c キー）して upsert

#### 2.6.8. 原価/販管費 → 帳票集計（ReportSummary）
-	insertReportSummaryFromCost
    -	Cost__c から原価系帳票集計を生成（原価(経費)/原価(材料費) など）
-	insertReportSummaryFromSga
    -	SGA__c から販管費系帳票集計を生成

#### 2.6.9. 共通処理
-	insertCostAndProjectCost
    -	Cost__c insert + insertProjectCost 呼び出し
-	insertProjectCost
    -	Cost__c の Project__c を基に ProjectCost__c を作成
    -	紐づきオブジェクト有無から種別（材料費/請求書払い/経費明細）を判定
-	buildAdminErrorMsg
    -	管理者向けエラーメール本文を整形

## 3. 使用している技術スタック
### 3.1. プラットフォーム
-	Salesforce Platform (Force.com)
-	Salesforce 標準機能：
    -	Custom Objects (Cost__c, SGA__c, AllocationDetail__c, ReportSummary__c, 他多数)
    -	Record Types (RecordTypeID__c カスタムオブジェクトで ID 管理)
    -	Custom Labels (System.Label.*)

### 3.2. 言語・フレームワーク
-	Apex
    -	Batch Apex (Database.Batchable, Database.Stateful)
    -	SOQL / DML / Savepoint & Rollback
    -	メール送信 (Messaging.SingleEmailMessage)

### 3.3. ロジック・周辺クラス
-	AllocationLogic
    -	各ドメインの配賦ロジックを集約したサービス層
-	MonthlyClosingLogic
    -	月次締め処理の同時実行制御、処理状態管理
-	DAO クラス群
    -	DivisionDao, UserSectionDao, ProjectDao, TotalizationCodeDao, SGADao など
    -	マスタ/トランザクションの検索を担うリポジトリ層

## 4. データフロー / 簡易シーケンス図（テキスト）
### 4.1. 全体プロセスチェーン（会社コード・月次単位）

```
[月次締め画面]
    |
    | BatchAllocation(allocationDate, EXEC_PRE1, companyCode) を executeBatch
    v
[BatchAllocation (EXEC_PRE1)]
  start() : getDeleteSgaAndReportSummary()
  execute(): 販管費 + 帳票集計削除
  finish():
    - エラーなし → EXEC_PRE2 バッチを起動

[BatchAllocation (EXEC_PRE2)]
  start() : getDeleteCostAndReportSummary()
  execute(): 原価 + 帳票集計削除
  finish():
    → EXEC_PRE3 バッチを起動

[BatchAllocation (EXEC_PRE3)]
  start() : getDeleteAllocationDetail()
  execute(): 配賦明細削除
  finish():
    → EXEC_PRE4 バッチを起動

[BatchAllocation (EXEC_PRE4)]
  start() : getAdjustDivisionForProject()
  execute(): 帳票集計の集計コード・部門調整
  finish():
    → EXEC_EXPENSES_JOB バッチを起動

...（経費/労務費/材料費/入金/請求書/仕入/部門販管費の各プロセスが続く）...

[BatchAllocation (EXEC_AD_TO_SGA)]
  start() : getAllocationDetailToSga()
  execute(): 配賦明細 → 販管費(SGA__c) 集約・upsert
  finish():
    → EXEC_POST1 バッチを起動

[BatchAllocation (EXEC_POST1)]
  start() : getCost()
  execute(): Cost__c → ReportSummary__c（原価系）作成
  finish():
    → EXEC_POST2 バッチを起動

[BatchAllocation (EXEC_POST2)]
  start() : getSga()
  execute(): SGA__c → ReportSummary__c（販管費系）作成
  finish():
    - AllocationLog__c(成功) 登録
    - MonthlyClosingLogic.setProcessingLogToCompleted()
    - チェーン完了
```

### 4.2. 代表的なドメイン別フロー例
#### 4.2.1. 経費(JOB) → 原価 → 帳票集計

```
ユーザ
  → 経費申請 (ExpensesNo__c, ExpensesDetail__c) 登録 & 承認
  → 月次締めで配賦バッチ起動 (EXEC_EXPENSES_JOB)

BatchAllocation(EXEC_EXPENSES_JOB).start()
  → getExpensesJob()
       - 当月計上月末日
       - ステータス: 精算済/支払処理済
       - 種別: JOB

BatchAllocation(EXEC_EXPENSES_JOB).execute()
  → allocationExpensesJob(List<ExpensesDetail__c>)
       - 各行を AllocationLogic.toCostForExpensesJob()
            → Cost__c へ変換
       - insertCostAndProjectCost()
            → Cost__c insert
            → ProjectCost__c insert (案件紐づけ)

後続プロセス EXEC_POST1
  → Cost__c (原価) を元に ReportSummary__c (原価(経費)) を作成
```

#### 4.2.2. 配賦明細 → 販管費 → 帳票集計

```
1. 経費/労務費/材料費/入金/請求書/仕入/部門販管費プロセス
   → AllocationLogic により AllocationDetail__c を大量に生成

2. EXEC_AD_TO_SGA
   start(): getAllocationDetailToSga()
   execute(): insertAllocationDetailToSga()
      - AllocationDetail__c → SGA__c へ変換 (toSga)
      - (発生日 + 科目コード + 部門(GL)) で集約
      - 既存SGA__cの SummaryKey__c とマージし upsert

3. EXEC_POST2
   start(): getSga()
   execute(): insertReportSummaryFromSga()
      - SGA__c → ReportSummary__c (販管費) 作成
```

