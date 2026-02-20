# MoversSuite Sales Analytics

SQL-based sales performance reporting built on top of **MoversSuite** (by EWS Group), the industry-standard platform used by Allied Van Lines agents and other moving companies. This repository documents a reporting system developed to move beyond surface-level commission reports and surface the revenue transparency needed to manage a high-performing household goods sales team.

---

## Background

MoversSuite includes a default commissionable revenue report per salesperson — a reasonable starting point, but one that masks critical revenue behavior. Starting in 2020, we used a straightforward monthly revenue query to track sales team performance against commissionable revenue goals. Over time, it became clear that commissionable revenue alone was an incomplete picture.

This repository documents the **evolution of that reporting system**: from a simple salesperson revenue template to a multi-CTE analytical query that compares estimates, bookable revenue, and commissionable revenue side by side — making it possible to identify compliance issues and protect company margin.

---

## The Problem With Default Reports

MoversSuite's built-in commission reports tell you *how much* commissionable revenue each salesperson generated. What they don't tell you:

- How does that number compare to what the salesperson originally estimated?
- Is the salesperson inflating commissionable line items at the expense of non-commissionable ones to increase their personal payout?
- Are salespeople pulling money out of their own commission to pay drivers to handle customer issues — offloading a core sales responsibility while reducing company revenue?

These behaviors are difficult to detect without a report that holds all three numbers — **estimate, bookable revenue, and commissionable revenue** — in the same view, attributed to the same salesperson.

---

## The Foundation: `vRevenueEntry`

Both queries in this repo rely on a custom view called `vRevenueEntryBMS`. This is a **modified copy** of MoversSuite's system-provided `vRevenueEntry` view, modified with BMS because we are 'Bailey's Moving & Storage'

### ⚠️ Important Note for Other Operators

The original view in MoversSuite carries this notice:

```sql
/**
 * This view is for ProServ, used in custom reports.
 * Don't rename the view without confirmation from ProServ.
 */
```

**Do not modify the original view.** If you want to customize field output, add filters, or alter the schema for your own reporting purposes — create a copy under a new name (e.g. `vRevenueEntryBMS`, `vRevenueEntryCustom`, etc.). This protects any built-in MoversSuite reports that depend on the original.

### What the View Does

`vRevenueEntryBMS` joins the core MoversSuite billing and order tables into a flat, reportable structure. It surfaces every revenue line item alongside its parent order, salesperson, commodity, commission detail, third-party payables, and address information — making it the foundation for nearly any revenue-focused custom report.

<details>
<summary>View definition (click to expand)</summary>

```sql
select
    OrderId = o.PriKey,
    OrderNumber = o.Orderno,
    OrderBranch = b.BranchID,
    ShipperLastName = o.LastName,
    ShipperFirstName = o.FirstName,
    RevenueClerk = rtrim(rev.FirstName) + ' ' + rtrim(rev.LastName),
    SalesPerson = rtrim(sls.FirstName) + ' ' + rtrim(sls.LastName),
    TypeOfMove = mt.MoveName,
    Commodity = ct.Commodity,
    AuthorityType = at.Description,
    TariffRate = o.TariffContract,
    BilledDate = o.ReleaseDate,
    OriginCity = oao.City,
    OriginState = oao.State,
    OriginZip = oao.PostalCode,
    DestinationCity = oad.City,
    DestinationState = oad.State,
    DestinationZip = oad.PostalCode,
    CustomerNumber = o.CustomerNumber,
    PaymentType = pt.PayName,
    PaymentMethod = pm.Method,
    PurchaseOrder = o.PurchaseOrderNo,
    TotalEstimateAmount = o.EstAmt,
    TotalActualAmount = o.ActualCost,
    EstimatedDiscountedLinehaul = o.LineHaul,
    ActualDiscountedLinehaul = o.ActLineHaul,
    Weight = o.Weight,
    Miles = o.Miles,
    Discount = o.Discount,
    HaulingDocumentsReceivedDate = o.PaperworkRecd,
    NationalAccount = a.AcctName,
    RevenueGroup = bmj.RevGroupPriKey,
    RevenueGroupAmount = bmj.Amount,
    RevenueGroupInv = bmj.InvoiceFlag,
    RevenueItem = bmn.Description,
    RevenueItemServiceCode = ic.ItemCode,
    RevenueItemAmount = bmn.Amount,
    RevenueItemPct = bmn.Percentage,
    RevenueItemQty1 = bmn.Quantity,
    RevenueItemQty2 = bmn.Quantity2,
    RevenueItemRate = bmn.Rate,
    RevenueItemAgent = revagt.AgentID,
    RevenueItemDivision = revdiv.Number,
    RevenueItemInv = bmn.InvoiceFlag,
    RevenueItemDocumentNumber = bmn.DocumentNumber,
    RevenueItemDocumentDate = bmn.DocDate,
    RevenueItemJournalDate = bmn.JournalDate,
    RevenueItemPostedDate = bmn.PostedDate,
    RevenueItemPostedBy = pb.FIRSTNAME + ' ' + pb.LASTNAME,
    CommissionItem = cd.Description,
    CommissionAmount = cd.Amount,
    CommissionPct = cd.CommissionPerc,
    CommissionDocumentNumber = cd.DocumentNumber,
    CommissionDocumentDate = cd.DocDate,
    CommissionJournalDate = cd.JournalDate,
    CommissionPostedDate = cd.PostedDate,
    CommissionPostedBy = cpb.FIRSTNAME + ' ' + cpb.LASTNAME,
    ThirdPartyItem = tp.Description,
    ThirPartyVendorNumber = tp.VendorID,
    ThirdPartyAmount = tp.PayableAmount,
    ThirdPartyDocumentNumber = tp.DocumentNumber,
    ThirdPartyDocumentDate = tp.DocDate,
    ThirdPartyJournalDate = tp.JournalDate,
    ThirdPartyPostedDate = tp.PostedDate,
    ThirdPartyPostedBy = tpb.FIRSTNAME + ' ' + tpb.LASTNAME
from Orders o
join BillingMajorItem bmj on o.PriKey = bmj.OrdPriKey
left join Branch b on o.BranchPriKey = b.BranchPriKey
left join Sysuser rev on o.RevenueClerk = rev.SysuserID
left join Sysuser sls on o.SalesPerson = sls.SysuserID
left join MoveType mt on o.MoveType = mt.PriKey
left join CommType ct on o.Commodity = ct.PriKey
left join AuthorityTypes at on o.AuthPriKey = at.AuthPriKey
left join OrderAddress oao on o.PriKey = oao.OrderFID
    and oao.AddressTypeFID = (select at.AddressTypeID from AddressType at where at.TypeName = 'Origin')
left join OrderAddress oad on o.PriKey = oad.OrderFID
    and oad.AddressTypeFID = (select at.AddressTypeID from AddressType at where at.TypeName = 'Destination')
left join PayType pt on o.PayTypeFID = pt.PayTypeID
left join PayMethod pm on o.PayMethodFID = pm.PayMethodID
left join Accounts a on o.AcctPriKey = a.AccountPriKey
left join BillingMinorItem bmn on bmj.BMajPriKey = bmn.BMajPriKey
left join ItemCode ic on bmn.ICPriKey = ic.ICPriKey
left join Agent revagt on b.AgentPriKey = revagt.AgentPriKey
left join Division revdiv on bmn.DivisionFID = revdiv.DivisionID
left join Sysuser pb on bmn.PostedBy = pb.SysUserID
left join CommissionedDetail cd on bmn.BMinPriKey = cd.BMinPriKey
left join Sysuser cpb on cd.PostedBy = cpb.SysUserID
left join ThirdParty tp on bmn.BMinPriKey = tp.BMinPriKey
left join Sysuser tpb on tp.PostedBy = tpb.SysUserID
```

</details>

---

## Query 1: Monthly Revenue by Salesperson (Starter Template)

This was the original report used beginning in 2020 to track commissionable revenue per salesperson against monthly goals. It's a practical, adaptable template — swap in your branch code, your salesperson names, your commodity, and your date range.

**What it answers:** How much commissionable revenue did each salesperson generate, broken down by month?

```sql
SELECT
    FORMAT([RevenueItemDocumentDate], 'MMMM') AS 'Month',
    MONTH([RevenueItemDocumentDate]) AS 'MonthNum',
    CASE
        WHEN CommissionItem LIKE '%Salesperson A (Sales)%' THEN 'Salesperson A'
        WHEN CommissionItem LIKE '%Salesperson B (Sales)%' THEN 'Salesperson B'
        WHEN CommissionItem LIKE '%Salesperson C (Sales)%' THEN 'Salesperson C'
        WHEN CommissionItem LIKE '%Salesperson D (Sales)%' THEN 'Salesperson D'
        WHEN CommissionItem LIKE '%Salesperson E (Sales)%' THEN 'Salesperson E'
        ELSE 'Other'
    END AS 'Commission Paid To',
    SUM([RevenueItemAmount]) AS 'Revenue Total'
FROM [MoversSuite2].[dbo].[vRevenueEntryBMS] v
INNER JOIN Orders o ON v.OrderId = o.PriKey
WHERE OrderBranch = '1160'                          -- 🔧 Replace with your branch code
  AND v.Commodity = 'Household Goods'               -- 🔧 Replace with your commodity
  AND [RevenueItemDocumentDate] >= '2020-01-01'     -- 🔧 Update date range
  AND [RevenueItemDocumentDate] < '2021-01-01'
  AND (
      CommissionItem LIKE '%Salesperson A (Sales)%' -- 🔧 Replace with your salesperson names
      OR CommissionItem LIKE '%Salesperson B (Sales)%'
      OR CommissionItem LIKE '%Salesperson C (Sales)%'
      OR CommissionItem LIKE '%Salesperson D (Sales)%'
      OR CommissionItem LIKE '%Salesperson E (Sales)%'
  )
GROUP BY
    FORMAT([RevenueItemDocumentDate], 'MMMM'),
    MONTH([RevenueItemDocumentDate]),
    CASE
        WHEN CommissionItem LIKE '%Salesperson A (Sales)%' THEN 'Salesperson A'
        WHEN CommissionItem LIKE '%Salesperson B (Sales)%' THEN 'Salesperson B'
        WHEN CommissionItem LIKE '%Salesperson C (Sales)%' THEN 'Salesperson C'
        WHEN CommissionItem LIKE '%Salesperson D (Sales)%' THEN 'Salesperson D'
        WHEN CommissionItem LIKE '%Salesperson E (Sales)%' THEN 'Salesperson E'
        ELSE 'Other'
    END;
```

**To adapt this query:**
- Replace branch code `'1160'` with your own branch ID from the `Branch` table
- Replace salesperson name strings with the names as they appear in your `CommissionItem` field
- Update the date range to any year or period you want to analyze
- Change `'Household Goods'` to any commodity type in your system

---

## Query 2: Estimate vs. Bookable vs. Commissionable Revenue by Salesperson

This is the core analytical query. It was built to answer a harder question: not just *how much* commissionable revenue did each salesperson generate, but *how does that compare to what they estimated and what the company actually retained?*

**What it answers:**
- What was each salesperson's total estimate value for the year?
- What percentage of that estimate became bookable (retained) revenue?
- What percentage became commissionable revenue specifically?
- Are there unusual gaps between those percentages that signal pricing manipulation or commission abuse?

### Why Multiple CTEs?

MoversSuite stores `TotalEstimateAmount` (from the `Orders` table) on **every revenue line item row** when the view joins down to `BillingMinorItem`. A single order with 10 line items could appear 10 times in the view — each row carrying the same estimate value. Summing it directly would produce estimate totals that are 10x (or more, based on number of rows) too high.

The solution: isolate the estimate calculation in its own CTE using `MAX()` per order — since the value is identical on every row, `MAX` returns the correct single value — while summing revenue amounts separately. Attempting to do this in a single pass produces inflated estimates and unreliable percentages.

```sql
WITH
order_salesman AS (
    SELECT
        o.PriKey,
        MIN(
            RTRIM(LTRIM(REPLACE(
                SUBSTRING(v.CommissionItem, 1, CHARINDEX('(', v.CommissionItem) - 1),
                'Commission - ', ''
            )))
        ) AS Salesman
    FROM [MoversSuite2].[dbo].[vRevenueEntryBMS] v
    JOIN Orders o ON v.OrderId = o.PriKey
    WHERE YEAR(v.RevenueItemDocumentDate) = 2025
      AND v.Commodity = 'Household Goods'
      AND CHARINDEX('(Sales)', v.CommissionItem) > 0
    GROUP BY o.PriKey
),
order_estimate AS (
    SELECT
        o.PriKey,
        MAX(v.TotalEstimateAmount) AS OrderEstimate
    FROM [MoversSuite2].[dbo].[vRevenueEntryBMS] v
    JOIN Orders o ON v.OrderId = o.PriKey
    WHERE YEAR(v.RevenueItemDocumentDate) = 2025
      AND v.Commodity = 'Household Goods'
    GROUP BY o.PriKey
),
order_bookable AS (
    SELECT
        o.PriKey,
        SUM(v.RevenueItemAmount) AS OrderBookableRevenue
    FROM [MoversSuite2].[dbo].[vRevenueEntryBMS] v
    JOIN Orders o ON v.OrderId = o.PriKey
    WHERE YEAR(v.RevenueItemDocumentDate) = 2025
      AND v.Commodity = 'Household Goods'
    GROUP BY o.PriKey
),
order_commissionable AS (
    SELECT
        o.PriKey,
        SUM(v.RevenueItemAmount) AS OrderCommissionableRevenue
    FROM [MoversSuite2].[dbo].[vRevenueEntryBMS] v
    JOIN Orders o ON v.OrderId = o.PriKey
    WHERE YEAR(v.RevenueItemDocumentDate) = 2025
      AND v.Commodity = 'Household Goods'
      AND CHARINDEX('(Sales)', v.CommissionItem) > 0
    GROUP BY o.PriKey
),
estimate_by_salesman AS (
    SELECT
        os.Salesman,
        SUM(oe.OrderEstimate) AS TotalEstimate
    FROM order_salesman os
    JOIN order_estimate oe ON oe.PriKey = os.PriKey
    GROUP BY os.Salesman
),
bookable_by_salesman AS (
    SELECT
        os.Salesman,
        SUM(ob.OrderBookableRevenue) AS TotalBookableRevenue
    FROM order_salesman os
    JOIN order_bookable ob ON ob.PriKey = os.PriKey
    GROUP BY os.Salesman
),
commissionable_by_salesman AS (
    SELECT
        os.Salesman,
        SUM(oc.OrderCommissionableRevenue) AS TotalCommissionableRevenue
    FROM order_salesman os
    JOIN order_commissionable oc ON oc.PriKey = os.PriKey
    GROUP BY os.Salesman
)
SELECT
    e.Salesman,
    ROUND(e.TotalEstimate, 2) AS TotalEstimate,
    b.TotalBookableRevenue,
    c.TotalCommissionableRevenue,

    -- Percent columns based on row totals (not averages)
    CAST(
        CASE WHEN NULLIF(e.TotalEstimate, 0) IS NULL THEN NULL
             ELSE b.TotalBookableRevenue / NULLIF(e.TotalEstimate, 0)
        END
        AS decimal(18,6)
    ) AS BookablePctOfEstimate,

    CAST(
        CASE WHEN NULLIF(e.TotalEstimate, 0) IS NULL THEN NULL
             ELSE c.TotalCommissionableRevenue / NULLIF(e.TotalEstimate, 0)
        END
        AS decimal(18,6)
    ) AS CommissionablePctOfEstimate

FROM estimate_by_salesman e
LEFT JOIN bookable_by_salesman b ON b.Salesman = e.Salesman
LEFT JOIN commissionable_by_salesman c ON c.Salesman = e.Salesman
ORDER BY e.TotalEstimate DESC;
```

### Key Technical Notes

**Isolating sales commissions:** The `CHARINDEX('(Sales)', v.CommissionItem) > 0` filter is essential. The `CommissionItem` field in MoversSuite contains commission entries for multiple roles — including drivers. Without this filter, driver commission entries would be attributed to the wrong person or inflate revenue figures. The `(Sales)` tag in the commission description is what identifies a line item as belonging to a salesperson.

**Extracting salesperson names:** The `order_salesman` CTE uses string functions to parse the salesperson's name out of the `CommissionItem` description field, which stores values like `"Commission - Jane Smith (Sales)"`. `CHARINDEX` finds the opening parenthesis, `SUBSTRING` extracts everything before it, and `REPLACE` strips the `"Commission - "` prefix. `LTRIM`/`RTRIM` clean up any whitespace.

**`NULLIF` for safe division:** All percentage calculations use `NULLIF(e.TotalEstimate, 0)` to avoid division-by-zero errors on orders with a zero estimate.

---

### Sample Output & How to Read It

> Branch: `1160` | Commodity: `Household Goods` | Year: `2025`

| Salesman | TotalEstimate | TotalBookableRevenue | TotalCommissionableRevenue | BookablePctOfEstimate | CommissionablePctOfEstimate |
|---|---|---|---|---|---|
| Alex Rivera | $2,084,500.00 | $1,674,210.00 | $876,440.00 | 0.8034 | 0.4205 |
| Jordan Meeks | $2,031,800.00 | $1,355,900.00 | $529,800.00 | 0.6674 | 0.2608 |
| Casey Thornton | $1,876,400.00 | $1,309,220.00 | $655,980.00 | 0.6977 | 0.3495 |
| Morgan Hale | $1,838,750.00 | $1,454,600.00 | $806,110.00 | 0.7912 | 0.4385 |
| Drew Callahan | $1,548,200.00 | $1,165,840.00 | $576,320.00 | 0.7530 | 0.3722 |
| Reese Navarro | $1,518,900.00 | $1,062,480.00 | $446,700.00 | 0.6994 | 0.2940 |
| Taylor Quinn | $1,374,600.00 | $1,097,340.00 | $714,280.00 | 0.7982 | 0.5197 |

---

**Understanding the percentages**

`BookablePctOfEstimate` tells you how much of what a salesperson estimated actually became retained revenue. `CommissionablePctOfEstimate` tells you what share of the estimate landed in commissionable line items specifically. The gap between those two numbers — and how each compares to the team median — is where the insight lives.

In this sample, the median `CommissionablePctOfEstimate` across the team sits around **0.37–0.39**. That becomes the baseline for identifying outliers in either direction.

**Low commissionable % relative to the median (e.g., Jordan Meeks at 0.2608)**
A salesperson who is meaningfully below the median on commissionable revenue — while their bookable revenue looks otherwise normal — may be redirecting commission dollars to resolve service problems rather than handling them directly. If a customer or driver situation becomes difficult, the path of least resistance is to pull money out of the commission line and pay it out, which closes the issue but reduces what the company retains and offloads a core sales responsibility. This report doesn't confirm that's happening, but it creates the visibility to ask the question.

**High commissionable % relative to the median (e.g., Taylor Quinn at 0.5197)**
A salesperson significantly above the median warrants a different kind of review. In some cases, this reflects a legitimate product mix — certain move types or service configurations are simply more commissionable by nature. In other cases, it may indicate a salesperson manipulating line item rates: raising the rates on commissionable items while suppressing non-commissionable ones to inflate their personal payout. The total revenue may look fine on the surface while the underlying pricing structure has been gamed.

**Bookable revenue vs. estimate (beyond commissions)**
The `BookablePctOfEstimate` column reveals something separate: how effectively a salesperson converts their estimate into actual retained revenue for the company. A rep who consistently captures a higher percentage isn't just closing — they may be making smart operational decisions that keep revenue in-house. Examples include recommending storage-in-transit at the origin agency rather than destination, or keeping labor and services in-house rather than subcontracting to third parties when the company has the capacity to perform the work. These decisions don't show up in commission reports at all, but they show up here.

---

## What This Enables

This reporting system gives sales leadership the ability to:

- **Detect commission inflation** — salespeople who raise commissionable line item rates while lowering non-commissionable ones to increase their personal payout at the company's expense
- **Identify responsibility offloading** — salespeople who resolve customer or driver disputes by pulling from their own commission rather than managing the situation directly, reducing company revenue in the process
- **Benchmark estimate accuracy** — understand which salespeople are pricing moves accurately versus padding or underquoting estimates
- **Drive behavioral change over time** — month-over-month and year-over-year comparisons make it possible to coach specific reps and measure improvement

The default MoversSuite commission reports show commissionable revenue in isolation. These queries put that number in context.

---

## Requirements

- Microsoft SQL Server (MoversSuite runs on MSSQL)
- Access to the `MoversSuite2` database
- A copy of `vRevenueEntry` saved as your own custom view (see note above)
- Appropriate read permissions on the Orders, BillingMajorItem, BillingMinorItem, CommissionedDetail, and related tables

---

## Compatibility

Built and tested against MoversSuite by EWS Group. Allied Van Lines agents and other moving companies using MoversSuite should be able to adapt these queries with minimal changes. Branch codes, salesperson names, and commodity values will vary by organization.

---

*Built by the marketing and operations team at an Allied Van Lines agent. Shared for the benefit of other MoversSuite operators looking to go deeper than out-of-the-box reporting.*
