# HB2 FIIE Funding Pipeline

## 📋 Overview

This document outlines the step-by-step process for calculating the Full Individual Initial Evaluation (FIIE) reimbursement funding allocation for Texas school districts. This funding mechanism provides school districts with $1,000 per completed initial special education evaluation to help offset the costs of comprehensive assessments required to determine student eligibility for special education services.

> [!IMPORTANT]
> Each step in this methodology is organized to create a clear flow from **legislative authority** → **business rule** → **calculation requirement** → **SAS code** → **results**. This structure ensures regulatory compliance, provides audit trails, and makes it easy to understand both the legal foundation and the technical implementation of each calculation step.

## 🏫 Full Individual Initial Evaluation (FIIE) Program

The FIIE reimbursement program provides financial support to Texas school districts for conducting comprehensive initial evaluations to determine student eligibility for special education services. These evaluations are critical for identifying students with disabilities and ensuring they receive appropriate educational support. The program recognizes both independent school districts that conduct their own evaluations and shared services arrangements (SSAs) where a fiscal agent coordinates evaluations for multiple member districts.

## 📜 Legislative Foundation

The FIIE reimbursement program supports Texas school districts in meeting their obligations under the Individuals with Disabilities Education Act (IDEA) and Texas Education Code requirements for timely and comprehensive special education evaluations.

> **Key Program Parameters**
>
> - **Reimbursement Rate**: $1,000 per completed initial evaluation
> - **Eligible Evaluations**: Initial evaluations completed during the school year as recorded in the Child Find system
> - **Funding Structure**: Districts receive reimbursement either independently or through their fiscal agent in SSA arrangements

## 🎯 Method

This methodology produces a complete funding pipeline that calculates reimbursements at both the individual district level and statewide totals, accounting for both independent districts and fiscal agent arrangements.

-----

### 🧮 Data Source

**Primary Data Source**: Oracle Child Find Student Program View (v_cf_stu_program_view)
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

### Step 1: Child Find Data Extraction

#### 📜 Business Rule
**BR-001: Child Find Data Source Requirements**
> Initial evaluation data must be extracted from the Oracle Child Find Student Program View, which serves as the authoritative source for special education evaluation records in Texas.

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
> When multiple evaluation records exist for the same student within a district, only the most recent evaluation (highest **`INITIAL_EVALUATION_DATE`**) qualifies for reimbursement. This prevents duplicate reimbursement claims for the same student.

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

Aggregate evaluation counts by fiscal agent and calculate total reimbursement at $1,000 per evaluation.

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

Calculate reimbursement amounts for independent districts based on their evaluation counts.

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

Calculate statewide metrics including total reimbursement amounts, record counts, and evaluation counts, with breakdowns by funding responsibility type.

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

1. **BR-001**: Child Find data source requirements for evaluation records
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

**Evaluation Activity:**
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

### Q: What qualifies as an "initial evaluation" for reimbursement purposes?

**A:** An initial evaluation is the first comprehensive assessment conducted to determine if a student qualifies for special education services under IDEA. The evaluation must be completed and recorded in the Child Find system with a valid evaluation date.

### Q: How are evaluations handled when a student moves between districts?

**A:** The district that completed the evaluation receives the reimbursement. If multiple districts have evaluation records for the same student, only the most recent evaluation qualifies for reimbursement based on the deduplication rules.

### Q: When do districts receive their FIIE reimbursements?

**A:** Reimbursement timing depends on TEA processing schedules and state appropriation cycles. Districts should monitor TEA communications for specific distribution dates.

### Q: How does the fiscal agent model work for SSAs?

**A:** In an SSA, member districts conduct evaluations but the fiscal agent receives the consolidated reimbursement. The fiscal agent is then responsible for distributing funds to member districts according to their SSA agreement.

### Q: Can a district receive multiple reimbursements for the same student?

**A:** No. The deduplication process ensures each student is counted only once per district, even if multiple evaluation records exist. Only the most recent evaluation date is used for reimbursement.

### Q: What if an evaluation spans multiple school years?

**A:** The reimbursement is tied to the school year in which the evaluation is completed, as indicated by the INITIAL_EVALUATION_DATE in the Child Find system.

-----

For questions, email [zane.wubbena@tea.texas.gov](mailto:zane.wubbena@tea.texas.gov)
