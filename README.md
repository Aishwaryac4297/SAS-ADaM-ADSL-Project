# 📊 ADaM ADSL Dataset Creation

This document outlines the SAS-based process for creating the **ADSL (Subject-Level Analysis Dataset)** as per **CDISC ADaM standards**. The ADSL dataset is derived from the cleaned SDTM `DM6` dataset and includes flags, treatment dates, demographic derivations, and other key subject-level information.

---

## 📥 Input Dataset

- **Source**: `DM6` (Demographics dataset from SDTM-DM)
- **Location**: `WORK.DM6`

---

## 🧬 Step-by-Step Derivations

---

### ✅ Step 1: Initialize Base ADSL

```sas
DATA ADSL1;
    SET WORK.DM6;
RUN;
```
### ✅ Step 2: Derive Key Subject-Level Flags
```sas
DATA ADSL2;
    SET ADSL1;

    /* Concatenated demographics summary */
    SEAR = CATX('/', SEX, ETHO, AGE, RACE);

    /* Informed consent date */
    CONSDT = INPUT(RFICDTC, YYMMDD10.);
    CONSTDTC = RFICDTC;

    /* Enrollment Flag */
    IF CONSDT NE . THEN ENRFL = 'Y';
    ELSE ENRFL = 'N';

    /* Safety Population Flag */
    IF RFXSTDTC NE '' THEN SAFFL = 'Y';
    ELSE SAFFL = 'N';

    /* Reason for exclusion from safety population */
    IF SAFFL = 'N' THEN SAFEXC = 'NOT DOSED';
    ELSE SAFEXC = 'DOSED';
RUN;
```
### ✅ Step 3: Derive Treatment Start/End Dates and Times
```sas
DATA ADSL3;
    SET ADSL2;

    /* Treatment Start DateTime */
    DTM = RFSTDTC;
    DTM_CLEAN = TRANSLATE(DTM, ' ', 'T');  /* Replace 'T' with space */
    TRTSTDTM = INPUT(DTM_CLEAN, ANYDTDTM.);
    TRTSTDT  = DATEPART(TRTSTDTM);
    TRTSTM   = TIMEPART(TRTSTDTM);
    TRTSTDTC = DTM;

    /* Treatment End DateTime */
    ETM = RFENDTC;
    ETM_CLEAN = TRANSLATE(ETM, ' ', 'T');
    TRTENDTM = INPUT(ETM_CLEAN, ANYDTDTM.);
    TRTENDT  = DATEPART(TRTENDTM);
    TRTETM   = TIMEPART(TRTENDTM);
    TRTEDTC  = ETM;

    FORMAT TRTSTDT TRTENDT DATE9.
           TRTSTM TRTETM TIME8.
           TRTSTDTM TRTENDTM DATETIME18.;
RUN;
```
### ✅ Step 4: Calculate Study Day of Consent
```sas
DATA ADSL4;
    SET ADSL3;

    /* Derive consent day relative to treatment */
    IF CONSDT < TRTSTDT THEN CONSDY = CONSDT - TRTSTDT;
    ELSE IF CONSDT >= TRTSTDT THEN CONSDY = (CONSDT - TRTSTDT) + 1;
RUN;
```
### ✅ Step 5: Derive Age Groupings
```sas
DATA ADSL4;
    SET ADSL3;

    LENGTH AGEGR1 $30.;

    IF AGE < 25 THEN DO;
        AGEGR1  = "=<25 Years";
        AGEGR1N = 1;
    END;
    ELSE IF AGE >= 25 THEN DO;
        AGEGR1  = ">=26 Years";
        AGEGR1N = 2;
    END;
RUN;
```
### 📌 Key Variables Summary
#### Variable	Description
- SEAR	Combined demographics (SEX/ETHO/AGE/RACE)
- ENRFL	Enrollment flag (Y/N)
- SAFFL	Safety population flag
- SAFEXC	Reason for exclusion from SAF population
- TRTSTDT	Treatment start date
- TRTSTM	Treatment start time
- TRTENDT	Treatment end date
- TRTETM	Treatment end time
- CONSDT	Parsed consent date
- CONSDY	Study day of consent
- AGEGR1	Age group label
- AGEGR1N	Age group numeric value

### ✅ Output Dataset
- Name: ADSL4

- Format: Compliant with CDISC ADaM guidelines

- Purpose: Ready for statistical analysis, TLFs, and FDA submission

### 📎 Notes
- Ensure all source variables from SDTM (e.g., RFSTDTC, RFENDTC, ETHO, etc.) are present and clean.

- Use consistent ISO8601 formats for date/time values.

- Review values of derived flags like SAFFL, ENRFL for completeness.
