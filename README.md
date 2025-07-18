# HB2 FIIE Funding Pipeline

## 📋 Overview

This document outlines the step-by-step process for calculating the full individual and initial evaluation (FIIE) reimbursement allotment for local educational agencies (LEAs) in Texas. This reimbursement mechanism established by [Texas House Bill (HB) 2, 89th Legislature, Regular Session (2025)](https://www.legis.state.tx.us/BillLookup/Actions.aspx?LegSess=89R&Bill=HB2) provides LEAs with $1,000 per completed initial evaluation of a child for special education eligibility.

> [!IMPORTANT]
> The method is organized in a clear flow from **legislative authority** → **business rule** → **calculation requirement** → **SAS code** → **results**.

## 🏫 Full Individual Initial Evaluation (FIIE) Program

A FIIE requires each LEA to obtain parental consent, as required by [34 CFR 300.300(a)](https://www.ecfr.gov/current/title-34/subtitle-B/chapter-III/part-300#:~:text=Parental%20consent%20for%20initial%20evaluation.), and conduct an individualized assessment of a child suspected of having a disability and then document the findings in a written report before initiating any special education or related services. Federal law mandates that this evaluation be completed prior to the initial provision of services ([34 C.F.R. §300.301(a)](https://www.ecfr.gov/current/title-34/subtitle-B/chapter-III/part-300#:~:text=General.%20Each%20public%20agency%20must%20conduct%20a%20full%20and%20individual%20initial%20evaluation%2C%20in%20accordance%20with%20%C2%A7%C2%A7%20300.304%20through%20300.306%2C%20before%20the%20initial%20provision%20of%20special%20education%20and%20related%20services%20to%20a%20child%20with%20a%20disability%20under%20this%20part.)) and grants states flexibility to set completion timeframes ([34 C.F.R. §300.301(c)(1)(ii)](https://www.ecfr.gov/current/title-34/subtitle-B/chapter-III/part-300#:~:text=If%20the%20State%20establishes%20a%20timeframe%20within%20which%20the%20evaluation%20must%20be%20conducted%2C%20within%20that%20timeframe](https://www.ecfr.gov/current/title-34/subtitle-B/chapter-III/part-300#:~:text=If%20the%20State%20establishes%20a%20timeframe%20within%20which%20the%20evaluation%20must%20be%20conducted%2C%20within%20that%20timeframe)); [20 U.S.C. § 1414(a)(1)(A)](https://sites.ed.gov/idea/statute-chapter-33/subchapter-ii/1414/a#:~:text=A%20State%20educational,under%20this%20subchapter.)). Texas exercises this flexibility through its alternative timeline in [19 TAC §89.1011](https://tea.texas.gov/about-tea/laws-and-rules/commissioner-rules-tac/coe-tac-currently-in-effect/ch089aa.pdf#page=5), as authorized by [TEC §29.004(a-1)](https://statutes.capitol.texas.gov/Docs/ED/htm/ED.29.htm#:~:text=Sec.%2029.004.-,FULL%20INDIVIDUAL%20AND%20INITIAL%20EVALUATION,-.%20%20(a)%20%20A%20written).

## 📜 Legislative Foundation

[HB 2 (TEC §48.159)](https://capitol.texas.gov/tlodocs/89R/billtext/pdf/HB00002F.pdf) contains an FIIE reimbursement provision that supports LEAs in meeting their initial evaluation obligations under the Individuals with Disabilities Education Act (IDEA).

> **SECTION 4.59. Subchapter D, Chapter 48, Education Code, is
amended by adding Section 48.159 to read as follows:**
> 
> **Sec. 48.159. SPECIAL EDUCATION FULL INDIVIDUAL AND INITIAL
EVALUATION.** For each child for whom a **school district conducts a
full individual and initial evaluation** under Section 29.004 or 20
U.S.C. Section 1414(a)(1), the district is entitled to an **allotment
of $1,000 or a greater** amount provided by appropriation.

This law was signed by Governor Greg Abbott on June 20, 2025 ([Journal p. 7610](https://journals.house.texas.gov/hjrnl/89r/pdf/89RDAY81FINAL.PDF#page=30))

## 🎯 Method

The method outlines a complete funding pipeline that calculates reimbursements at both the individual LEA level and statewide totals, accounting for both independent districts and fiscal agent arrangements.

-----

### 🧮 Data Source

**Primary Data Source**: Oracle Database (Program View: **`v_cf_stu_program_view`**)
- **School Year:** 2023-2024
- **Federal Data Collection Period:** July 1 - June 30
- **Deadline:** July 25, 2024 (Last Thursday in July)

**Secondary Data Source**: Fiscal Agent and SSA Membership Data
- Excel file containing fiscal agent assignments and SSA membership flags
- Maps districts to their fiscal agents for consolidated reimbursement

#### 📅 Data Collection Timeline

| School Year | Data Collection Period | Processing Date <br>(Last Thursday in July) | Expected Distribution |
|-------------|------------------------|--------------------|----------------------|
| 2023-24     | 7/1/2023 - 6/30/2024      | July 2024       | TBD                  |
| 2025-26     | 7/1/2024 - 6/30/2025      | July 2026       | TBD                  |
| 2026-27     | 7/1/2025 - 6/30/2026      | July 2027       | TBD                  |

#### 📋 Timeline Notes

- **Data Collection**: Initial evaluations recorded throughout the school year
- **Processing**: Annual calculation after school year completion
- **Distribution Timing**: FIIE reimbursements timing to be determined based on TEA processing schedules

-----

### Step 1: Data Extraction

#### 📜 Business Rule
**BR-001: Child Find Data Source Requirements**
> Initial evaluation data must be extracted from the Oracle database program view in Oracle, which serves as the authoritative source for initial evaluation records in Texas.

#### 📘 Calculation Requirement

Extract all student records from the Child Find system for the specified school year that contain completed initial evaluation dates.

#### 💻 SAS Code

```sas
proc sql;
    create table base as
    select
      a.SCHOOL_YEAR                 label="School Year",
      a.DISTRICT_ID as DISTRICT     label="County-District Number (CDN)",
      a.DISTRICT_NAME               label="District Name",
      b.SSA_FLAG                    label="SSA Member Flag",
      b.FISCAL_AGENT                label="Fiscal Agent County-District Number (CDN)",
      b.FISCAL_AGENT_NAME           label="Fiscal Agent Name",
      b.FISCAL_AGENT_FLAG           label="Fiscal Agent Flag",
      a.TX_UNIQUE_STUDENT_ID        label="Texas Unique Student ID",
      a.STUDENT_ID                  label="Student ID",
      a.INITIAL_EVALUATION_DATE     label="Initial Evaluation Date",
      today() as DATE_PROCESSED
        format=yymmdd10.            label="Date Processed"
    from
        sch_cc_1.v_cf_stu_program_view as a
        left join fiscal_agent_24 as b
        on a.DISTRICT_ID = b.DISTRICT
    where
        a.SCHOOL_YEAR = 2024
        and not missing(a.INITIAL_EVALUATION_DATE)
    order by DISTRICT;
quit;
```

#### ✅ Output

```
NOTE: Table WORK.BASE created, with 188,270 rows and 11 columns.
```

-----

### Step 2: Student Record Deduplication

#### 📜 Business Rule
**BR-002: Student Record Deduplication Rules**
> When multiple initial evaluation records exist for the same student within a district, only the most recent initial evaluation (highest **`INITIAL_EVALUATION_DATE`**) qualifies for reimbursement. This prevents duplicate reimbursement claims for the same student.

#### 📘 Calculation Requirement

Sort student records by **`DISTRICT`**, **`STUDENT_ID`**, and **`INITIAL_EVALUATION_DATE`** (descending) to identify the most recent evaluation. Retain only the first record for each unique district-student combination.

#### 💻 SAS Code

```sas
/* Sort so that the newest datetime is first in each BY-group */
proc sort data=base out=base_sorted;
    by DISTRICT STUDENT_ID descending INITIAL_EVALUATION_DATE;
run;

/* Split into base_nodup (most recent) and base_dup (duplicates) */
data base_nodup base_dup;
    set base_sorted;
    by DISTRICT STUDENT_ID;
    
    format INITIAL_EVALUATION_DATE datetime20.;
    
    if first.STUDENT_ID then output base_nodup;
    else output base_dup;
run;
```

#### ✅ Output

```
NOTE: The data set WORK.BASE_NODUP has 184,894 observations and 11 variables.
NOTE: The data set WORK.BASE_DUP has 3,376 observations and 11 variables.
```
-----

### Step 3: Funding Responsibility Classification

#### 📜 Business Rule
**BR-003: Funding Responsibility Assignment**
> Districts are classified into two funding responsibility types based on their fiscal agent status:
> - **Independent**: Districts that manage their own special education funding (FISCAL_AGENT_FLAG = 0)
> - **Fiscal Agent**: Districts that serve as fiscal agents for SSA arrangements (FISCAL_AGENT_FLAG = 1)

#### 📘 Calculation Requirement

Separate the deduplicated records into two datasets based on the fiscal agent flag to enable appropriate aggregation and reimbursement routing.

#### 💻 SAS Code

```sas
data base_independent base_fiscal_agent;
    set base_nodup;
    
    if FISCAL_AGENT_FLAG = 0 then
        output base_independent;
    else if FISCAL_AGENT_FLAG = 1 then
        output base_fiscal_agent;
run;
```

#### ✅ Output

```
NOTE: Data sets WORK.BASE_INDEPENDENT and WORK.BASE_FISCAL_AGENT created.
```

-----

### Step 4: Fiscal Agent Reimbursement Aggregation

#### 📜 Business Rule
**BR-004: Fiscal Agent Reimbursement Consolidation**
> For districts participating in SSAs, all member district evaluations are consolidated under the fiscal agent for reimbursement purposes. The fiscal agent receives the combined reimbursement for all evaluations conducted by SSA members.

#### 📘 Calculation Requirement

Aggregate initial evaluation counts by fiscal agent and calculate total reimbursement at $1,000 per evaluation.

#### Formula:

![Reimbursement per Fiscal Agent](https://latex.codecogs.com/svg.latex?R_{fa}%3D1000%5Csum_{d=1}^{n_{fa}}e_{d})

**Where**  
- $R_{fa}$ = Total reimbursement for fiscal agent $fa$ (in dollars)  
- $n_{fa}$ = Number of districts served by fiscal agent $fa$  
- $e_{d}$ = Number of initial evaluations conducted in district $d$ under fiscal agent $fa$

#### 💻 SAS Code

```sas
%let reimbursement_rate = 1000;

proc sql;
    create table fiscal_agent_summary as
    select
        SCHOOL_YEAR                                     label="School Year",
        FISCAL_AGENT as RESPONSIBLE_DISTRICT            label="Responsible County-District Number (CDN)",
        FISCAL_AGENT_NAME as RESPONSIBLE_DISTRICT_NAME  label="Responsible District Name",
        
        count(STUDENT_ID)
            as TOTAL_FIIE_CNT                           label="Total Number of Initial Evaluations",
            
        calculated TOTAL_FIIE_CNT * &reimbursement_rate
            as TOTAL_FIIE_REIMBURSEMENT
            format=dollar12.                            label="Total FIIE Reimbursement Amount",
            
        "FISCAL AGENT"
        as FUNDING_RESPONSIBILITY
        length=15                                       label="Funding Responsibility Type"
        
    from base_fiscal_agent
    group by
        SCHOOL_YEAR,
        FISCAL_AGENT,
        FISCAL_AGENT_NAME;
quit;
```

#### ✅ Output

```
NOTE: Table WORK.FISCAL_AGENT_SUMMARY created, with 109 rows and 6 columns.
```

-----

### Step 5: Independent District Reimbursement Calculation

#### 📜 Business Rule
**BR-005: Independent District Direct Reimbursement**
> Districts not participating in SSAs receive reimbursement directly for evaluations conducted within their district. Each district receives $1,000 per completed initial evaluation.

#### 📘 Calculation Requirement

Calculate reimbursement amounts for independent districts based on their initial evaluation counts.

#### Formula:

![Independent District Reimbursement](https://latex.codecogs.com/svg.latex?R_{d}%3D1000%5Ctimes%20E_{d})

**Where**  
- $R_{d}$ = Direct reimbursement amount for independent district $d$ (in USD)  
- $E_{d}$ = Number of completed initial evaluations in district $d$

#### 💻 SAS Code

```sas
%let reimbursement_rate = 1000;

proc sql;
    create table independent_summary as
    select
      SCHOOL_YEAR                                 label="School Year",
      DISTRICT as RESPONSIBLE_DISTRICT            label="Responsible County-District Number (CDN)",
      DISTRICT_NAME as RESPONSIBLE_DISTRICT_NAME  label="Responsible District Name",
      
      count(STUDENT_ID)
          as TOTAL_FIIE_CNT                       label="Total Number of Initial Evaluations",
          
      calculated TOTAL_FIIE_CNT * &reimbursement_rate
          as TOTAL_FIIE_REIMBURSEMENT
          format=dollar12.                        label="Total FIIE Reimbursement Amount",
          
      "INDEPENDENT"
          as FUNDING_RESPONSIBILITY
          length=11
          label="Funding Responsibility Type"
          
    from base_independent
    group by
        SCHOOL_YEAR,
        DISTRICT,
        DISTRICT_NAME;
quit;
```

#### ✅ Output

```
NOTE: Table WORK.INDEPENDENT_SUMMARY created, with 618 rows and 6 columns.
```

-----

### Step 6: Combined District Summary

#### 📜 Business Rule
**BR-006: Unified Reimbursement Summary**
> All reimbursement calculations must be combined into a single summary dataset that maintains the distinction between fiscal agent and independent funding responsibilities for accurate disbursement.

#### 📘 Calculation Requirement

Combine the fiscal agent and independent district summaries into a unified dataset for reporting and distribution.

#### 💻 SAS Code

```sas
data combined_summary;
    length
        FUNDING_RESPONSIBILITY     $15
        RESPONSIBLE_DISTRICT_NAME  $75;
    set
        fiscal_agent_summary
        independent_summary;
run;
```

#### ✅ Output

```
NOTE: The data set WORK.COMBINED_SUMMARY has 727 observations and 6 variables.
```

**Summary Statistics:**
- Total Districts/Fiscal Agents: 727
- Fiscal Agents: 109
- Independent Districts: 618

-----

### Step 7: Statewide Summary Calculation

#### 📜 Business Rule
**BR-007: Statewide Reporting Requirements**
> Statewide totals must be calculated for monitoring, budgeting, and legislative reporting purposes. Totals must be provided overall and by funding responsibility type.

#### 📘 Calculation Requirement

Calculate statewide metrics including total reimbursement amounts, record counts, and initial evaluation counts, with breakdowns by funding responsibility type.

#### Formula:

![Total FIIE Reimbursement](https://latex.codecogs.com/svg.latex?R_%7B%5Cmathrm%7Btotal%7D%7D%3D%5Csum_%7Bi%3D1%7D%5E%7BN%7Dr_%7Bi%7D)

![Independent District Reimbursement](https://latex.codecogs.com/svg.latex?R_%7B%5Cmathrm%7Bind%7D%7D%3D%5Csum_%7B%5Csubstack%7Bi%3D1%5C%5Cf_%7Bi%7D%3D%5Cmathrm%7BINDEPENDENT%7D%7D%7D%5E%7BN%7Dr_%7Bi%7D)

![Fiscal Agent Reimbursement](https://latex.codecogs.com/svg.latex?R_%7B%5Cmathrm%7Bfa%7D%7D%3D%5Csum_%7B%5Csubstack%7Bi%3D1%5C%5Cf_%7Bi%7D%3D%5Cmathrm%7BFISCAL%5C%20AGENT%7D%7D%7D%5E%7BN%7Dr_%7Bi%7D)

**Where**  
- $R_{\mathrm{total}}$ = Total FIIE reimbursement amount (all records)  
- $R_{\mathrm{ind}}$   = Total FIIE reimbursement for independent districts (`FUNDING_RESPONSIBILITY = INDEPENDENT`)  
- $R_{\mathrm{fa}}$    = Total FIIE reimbursement for fiscal agent districts (`FUNDING_RESPONSIBILITY = FISCAL AGENT`)  
- $r_{i}$             = value of `TOTAL_FIIE_REIMBURSEMENT` for record $i$  
- $f_{i}$             = value of `FUNDING_RESPONSIBILITY` for record $i$  
- $N$                 = total number of records in `combined_summary`  

#### 💻 SAS Code

```sas
proc sql;
    create table statewide_summary as
    
    /* Total FIIE Reimbursement (all records) */
    select
        "Total FIIE Reimbursement Amount" as METRIC length=50 label="Metric",
        sum(TOTAL_FIIE_REIMBURSEMENT)     as VALUE  format=dollar20. label="Value"
    from combined_summary
    
    union all
    
    /* Total FIIE Reimbursement - Independent */
    select
        "Total FIIE Reimbursement - Independent" as METRIC length=50 label="Metric",
        sum(TOTAL_FIIE_REIMBURSEMENT)            as VALUE  format=dollar20. label="Value"
    from combined_summary
    where FUNDING_RESPONSIBILITY="INDEPENDENT"
    
    union all
    
    /* Total FIIE Reimbursement - Fiscal Agent */
    select
        "Total FIIE Reimbursement - Fiscal Agent" as METRIC length=50 label="Metric",
        sum(TOTAL_FIIE_REIMBURSEMENT)             as VALUE  format=dollar20. label="Value"
    from combined_summary
    where FUNDING_RESPONSIBILITY="FISCAL AGENT"
    
    /* Additional metrics for counts... */
    order by METRIC;
quit;
```

#### ✅ Output

```
NOTE: Table WORK.STATEWIDE_SUMMARY created, with 9 rows and 2 columns.
```

**Statewide Results:**
- Total FIIE Reimbursement Amount: $184,894,000
- Total Initial Evaluations: 184,894
- Average per District: ~254 evaluations

-----

## 📊 Summary

### Business Rules Summary

The FIIE reimbursement calculation methodology is governed by seven key business rules:

1. **BR-001**: Child Find data source requirements for initial evaluation records
2. **BR-002**: Student record deduplication to prevent duplicate claims
3. **BR-003**: Funding responsibility assignment (Independent vs Fiscal Agent)
4. **BR-004**: Fiscal agent reimbursement consolidation for SSA members
5. **BR-005**: Independent district direct reimbursement calculation
6. **BR-006**: Unified reimbursement summary requirements
7. **BR-007**: Statewide reporting and aggregation requirements

### Dual-Track Reimbursement Flow

This methodology accommodates two distinct reimbursement pathways based on district arrangements:

#### Track 1: Independent Districts
- Districts not participating in SSAs
- Direct reimbursement for evaluations conducted
- 618 districts in 2024-25
- Individual disbursements to each district

#### Track 2: Fiscal Agent Arrangements
- Districts participating in SSAs
- Consolidated reimbursement through fiscal agent
- 109 fiscal agents in 2024-25
- Bulk disbursements to fiscal agents for distribution

### Key Program Metrics (2024-25)

**Initial Evaluation Activity:**
- Total Initial Evaluations Completed: 184,894
- Duplicate Records Removed: 3,376
- Net Evaluations for Reimbursement: 184,894

**Financial Impact:**
- Total Statewide Reimbursement: $184,894,000
- Reimbursement Rate: $1,000 per evaluation
- Average per District/Agent: $254,370

**Distribution Breakdown:**
- Independent Districts: 618 entities
- Fiscal Agents: 109 entities
- Total Funding Entities: 727

-----

## ❓ Frequently Asked Questions (FAQ)

### Q: What qualifies as an initial evaluation for reimbursement purposes?

**A:** An initial evaluation is the first comprehensive individual assessment conducted to determine if a children qualifies for special education services under IDEA. The initial evaluation must be completed and recorded with a valid initial evaluation date in the Texas Student Data System (TSDS).

### Q: How are initial evaluations handled when a student moves between districts?

**A:** The district that completed the eligibility determination receives the reimbursement. If multiple districts have initial evaluation records for the same student, only the most recent initial evaluation and eligibility determination qualifies for reimbursement based on the deduplication rules.

### Q: When do districts receive their FIIE reimbursements?

**A:** Reimbursement timing depends on TEA processing schedules and state appropriation cycles. Districts should monitor TEA communications for specific distribution dates.

### Q: How does the fiscal agent model work for SSAs?

**A:** In an SSA, member districts conduct initial evaluations but the fiscal agent receives the consolidated reimbursement. The fiscal agent is then responsible for distributing funds to member districts according to their SSA agreement.

### Q: Can a district receive multiple reimbursements for the same student?

**A:** No. The deduplication process ensures each student is counted only once per district, even if multiple initial evaluation records exist. Only the most recent initial evaluation date is used for reimbursement.

### Q: What if an evaluation spans multiple school years?

**A:** The reimbursement is tied to the school year in which the eligibility determination occurs even if the initial evaluation (**`INITIAL_EVALUATION_DATE`**) was completed during a prior school year. This scenario is referred to as “holdover students.”

-----

For questions, email [zane.wubbena@tea.texas.gov](mailto:zane.wubbena@tea.texas.gov)
