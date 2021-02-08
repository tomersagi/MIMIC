# MIMIC IV to OMOP CDM Conversion #


The project implements an ETL conversion of MIMIC IV PhysioNet dataset to OMOP CDM format.

* Version 1.0


### Concepts / Phylosophy ###

The ETL is based on the five steps.
* Create a snapshot of the source data. The snapshot data is stored in staging source tables with prefix "src_".
* Clean source data: filter out rows to be not used, format values, apply some business rules. Create intermediate tables with prefix "lk_" and postfix "clean".
* Map distinct source codes to concepts in vocabulary tables. Create intermediate tables with prefix "lk_" and postfix "concept".
    * Custom mapping is implemented in custom concepts generated in vocabulary tables beforehand.
* Join cleaned data and mapped codes. Create intermediate tables with prefix "lk_" and postfix "mapped".
* Distribute mapped data by target cdm tables according to target_domain_id values.

Intermediate and staging CDM tables have additional working fields like unit_id. Field unit_id is composed during the ETL steps. From right to left: source table name, initial target table name abbreviation, final target table name or abbreviation. For example: unit_id = 'drug.cond.diagnoses_icd' means that the rows in this unit_id belong to Drug_exposure table, inially were prepared for Condition_occurrence table, and its original is source table diagnoses_icd.

Vocabularies are kept in a separate dataset, and are copied as a part of the snapshot data too.


### How to run the conversion ###

* The ETL process encapsulates the following workflows: ddl, vocabulary_refresh, staging, etl, ut, unload.
* The unload workflow results in creating a final OMOP CDM dataset, which can be analysed with OHDSI tools as Atlas or DQD.

* How to run ETL end-to-end:
    * update config files accordingly
    * perform vocabulary_refresh steps if needed (see vocabulary_refresh/README.md)
    * set the project root (location of this file) as the current directory

    `
    cd vocabulary_refresh
    python vocabulary_refresh.py -s10
    python vocabulary_refresh.py -s20
    python vocabulary_refresh.py -s30
    cd ../
    python scripts/run_workflow.py -e conf/dev.etlconf -c conf/workflow_ddl.conf
    python scripts/run_workflow.py -e conf/dev.etlconf -c conf/workflow_staging.conf
    python scripts/run_workflow.py -e conf/dev.etlconf -c conf/workflow_etl.conf
    python scripts/run_workflow.py -e conf/dev.etlconf -c conf/workflow_ut.conf
    python scripts/run_workflow.py -e conf/dev.etlconf -c conf/workflow_metrics.conf
    `
* How to look at UT and Metrics reports
    * see metrics dataset name in the corresponding .etlconf file

    `
    -- UT report
    SELECT report_starttime, table_id, test_type, field_name
    FROM metrics_dataset.report_unit_test
    WHERE NOT test_passed 
    ;
    -- Metrics - row count
    SELECT * FROM metrics_dataset.me_total ORDER BY table_name;
    -- Metrics - person and visit summary
    SELECT
        category, name, count AS row_count
    FROM metrics_dataset.me_persons_visits ORDER BY category, name;
    -- Metrics - Mapping rates
    SELECT
        table_name, concept_field, 
        count   AS rows_mapped, 
        percent AS percent_mapped,
        total   AS rows_total
    FROM metrics_dataset.me_mapping_rate 
    ORDER BY table_name, concept_field
    ;
    -- Metrics - Top 100 Mapped and Unmapped
    SELECT 
        table_name, concept_field, category, source_value, concept_id, concept_name,
        count       AS row_count,
        percent     AS rows_percent
    FROM metrics_dataset.me_tops_together 
    ORDER BY table_name, concept_field, category, count DESC;
    `

* More option to run ETL parts:
    * Run a workflow:
        * with local variables: `python scripts/run_workflow.py -c conf/workflow_etl.conf`
            * copy "variables" section from file.etlconf
        * with global variables: `python scripts/run_workflow.py -e conf/dev.etlconf -c conf/workflow_etl.conf`
    * Run explicitly named scripts (space delimited):
        `python scripts/run_workflow.py -e conf/dev.etlconf -c conf/workflow_etl.conf etl/etl/cdm_drug_era.sql`
    * Run in background:
        `nohup python scripts/run_workflow.py -e conf/full.etlconf -c conf/workflow_etl.conf > ../out_full_etl.out &`
    * Continue after an error:
        `nohup python scripts/run_workflow.py -e conf/full.etlconf -c conf/workflow_etl.conf etl/etl/cdm_observation.sql etl/etl/cdm_observation_period.sql etl/etl/cdm_fact_relationship.sql etl/etl/cdm_condition_era.sql etl/etl/cdm_drug_era.sql etl/etl/cdm_dose_era.sql etl/etl/cdm_cdm_source.sql >> ../out_full_etl.out &`


### Change Log (latest first) ###


**2021-02-08**

* Set version v.1.0

* Drug_exposure table
    * pharmacy.medication is replacing particular values of prescription.drug
    * source value format is changed to COALESCE(pharmacy.medication.selected, prescription.drug) || prescription.prod_strength
* Labevents mapping is replaced with new reviewed version
    * vocabulary affected: mimiciv_meas_lab_loinc
    * lk_meas_labevents_clean and lk_meas_labevents_mapped are changed accordingly
* Unload for Atlas
    * Technical fields unit_id, load_row_id, load_table_id, trace_id are removed from Atlas devoted tables
* Delivery export script
    * tables are exported to a single directory and single files. If a table is too large, it is exported to multiple files
* Bugfixes and cleanup
* Real environmental names are replaced with placeholders


**2021-02-01**

* Waveform POC-2 is created for 4 MIMIC III Waveform files uploaded to the bucket
    * iterate through the folders tree, capture metadata and load the CSVs
* Bugfixes


**2021-01-25**

* Mapping improvement
* New visit logic for measurement (picking visits by event datetime)
* Bugfixes


**2021-01-13**

* Export and import python scripts:
    * `scripts/delivery/export_from_bq.py`
        * the script is adjusted to export BQ tables from Atlas dataset to GS bucket, and optionally to the local storage to CSV files
    * `scripts/delivery/load_to_bq.py`
        * the script is adjusted to load the exported CSVs from local storage or from the bucket to the given target BQ dataset, created automatically


**2020-12-14**

* TUF-55 Custom mapping derived from MIMIC III
    * A little more custom mapping
* TUF-77 UT, QA and Metrics for tables with basic logic
    * UT for all tables:
        * Unique fields
        * FK to person_id, visit_occurrence_id, some individual FK


**2020-12-03**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * Bugfix: Visit and Visit_detail FK violation - fixed
* TUF-55 Custom mapping derived from MIMIC III
    * Measurement.value_as_concept_id - mapped partially
    * Unit - mapping is loaded
* TUF-77 UT, QA and Metrics for tables with basic logic
    * Make Top100 step faster - started

**2020-12-01**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * Bugfix: Person.race_concept_id and Co - is fixed
    * Bugfix: Measurement.unit_concept_id - is fixed
    * Bugfix: Measurement.value_as_concept_id - copied for mapping
    * Bugfix: Visit_detail.services.visit_detail_concept_id and Co - copied for mapping
    * Bugfix: Visit and Visit_detail FK violation - tables implementation is refactored
* TUF-75 Basic Orchestration
    * Bugfix: runs end-to-end if a given script is not found - fixed
        * bq_run_script.py now stops if any of given scripts are not found, and reports the names


**2020-11-30**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * Minor update to Care_site
    * Updates to Measurement (chartevents)
        * 4 types are added: Heart Rate, Respiratory Rate, O2 Saturation, Heart Rythm
    * Fact_relationship is added
    * Bugfix to visit_source_value (hadm_id -> concat(subject_id, '|', hadm_id))
        * Visit_occurrence, Visit_detail, and all event tables
* TUF-55 Custom mapping derived from MIMIC III
    * Microbiology mapping
        * Microtest is created
        * Organism is updated
    * Admission mapping - 3 vocabularies added
    * Place of service is added
* TUF-75 Basic Orchestration
    * global config (etlconf) is added to bq_run_scirpt.py and run_workflow.py
    * Bug: run_workflow.py runs end-to-end if the given script is not found
* TUF-77 UT, QA and Metrics for tables with basic logic
    * metrics_gen is added (from mimic POC in May 2020)
    * gen_bq_ut_basic.py is created, in progress
    * src_statistics.sql is started (source row counts)
* TUF-83 Complete run on the Full set
    * full.etlconf is added, end-to-end run is started


**2020-11-20**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * Mapping for Visits is updated
    * Condition_era, Drug_era, Dose_era are added
* TUF-55 Custom mapping derived from MIMIC III
    * Microbiology mapping - 4 vocabularies are added, 1 to be created
    * Analysis for other mapping
    * Minor updates


**2020-11-13**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-19 Measurement - Rule 3 (microbiology) - done (without tests)
    * TUF-58 Specimen - done (without tests)
* TUF-75 Basic Orchestration
    * run_job.py is renamed to run_workflow.py, 
    * Variables section is added to configs for bq_run_script.py and run_workflow.py
    * Dataset names are replaced in ut/\*.sql and qa/\*.sql
    * run_workflow.py is updated, the issue with not stopping on an error is fixed
    * Nice output is added to bq_run_script


**2020-11-12**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-19 Measurement - Rule 3 (microbiology) ready to run


**2020-11-10**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-19 Measurement - Rule 3 (microbiology) is in progress
    * TUF-58 Specimen - ready to run
    * Other updates


**2020-10-27**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-20 Death table - done
    * TUF-19 Measurement - Rule 2 (chartevent) is started
* TUF-76 Bugfix according to Achilles Heel Report
    * Person count - done
    * Condition mapping rate - done
    * Observation periods - done
    * For next updates:
        * Drug concepts in Condition
        * Observation value_as fields all null
        * Event dates before year of birth
* TUF-75 Basic Orchestration
    * Fixed: dealing with quotes (") inside a query
    * Known issue: do not use semicolon (;) inside comments


**2020-10-26**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-19 Minor fixes for Measurement lookups
    * TUF-51 Device_exposure - done with Rule 1 lk_drug_mapped
    * TUF-58 Specimen - is started
* TUF-75 Basic Orchestration
    * run_job.sql is created
* ATLAS dataset is populated: `mimiciv_202010_cdm_531`


**2020-10-21**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-19 Measurement - basic is done, Rule 1 (labevents) and Rule 10 (wf poc demo) are implemented
    * TUF-53 Observation_period - basic is done
* TUF-75 Basic Orchestration
    * Config design in progress


**2020-10-21**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-23 Drug_exposure - basic is done, Rule 1 is implemented
    * TUF-55 Custom mapping - vocabulary references in the code are updated, CSV's folder is moved
* TUF-50 Create dataset for ATLAS - the SQL script generator is created, script is generated


**2020-10-18**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-23 Drug_exposure - in progress
    * TUF-20 Death - in progress
    * TUF-55 Custom mapping - CSV's for implemented tables are converted (todo: rename custom vocabularies in the code)


**2020-10-12**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-18 Condition_occurrence - basic logic, UT, QA are created
    * TUF-22 Procedure_occurrence - basic logic is created, UT and QA - to do
    * TUF-21 Observation - from Procedure III is started, from Observation III is next
    * TUF-23 Drug_exposure - started
    * TUF-20 Death - in progress


**2020-10-01**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-19 Measurement - Rule 1 (lab from labevents) is implemented, dry run is done
    * MIMIC III extras/concept custom mapping CSVs are loaded to BQ to raw tables (for reference)


**2020-09-24**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-17 Visit_detail - basic ETL, UT, QA are done (see Known issues in the ETL script)
    * TUF-19 Measurement - start


**2020-09-22**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-15 Visit_occurrence - basic ETL, UT, QA are done, proper to do
    * TUF-16 Orchestration overhead - config design
    * TUF-17 Visit_detail - basic ETL is in progress


**2020-09-18**

* TUF-11 Load Vocabularies
    * vocabulary_refresh.conf - updated
    * sql scripts - project.dataset template is set
    * py scripts - some error handling is added, minor bug is fixed
    * Athena vocabularies are loaded to `vocabulary_2020_09_11`


**2020-09-15**

* TUF-10 Basic logic for CDM tables based on MIMIC-III logic
    * TUF-12 Location - Hardcode the single location
    * TUF-13 Care_site - basic ETL, UT, QA are done, to add proper mapping and ID
    * TUF-14 Person - basic ETL, UT, QA are done, to add proper mapping and ID


**2020-09-11**

Start development
* The project includes:
    * folders
    * DDL script
    * vocabulary_refresh scripts set

### The End ###


### Concepts / best practice / lessons learned ###

* keep vocabularies in a separate dataset standard, and add custom mapping each time vocabs are copied to the cdm dataset? 
    * usual practice: apply custom mapping to vocabs in the vocab dataset
    * plus of this alternative: no clean-up is needed, we just add the custom concepts
* working on a task, 
    * create a branch / complete prev PR
    * declare a comparison dataset, i.e. the result of previously completed task
    * create a new working dataset named mimiciv_cdm_task_id_developer_yyyy_mm_dd
    * create a UAT dataset when a significant piece of work is completed and tested
        * the name is mimiciv_uat_yyyy_mm_dd
        * uat = copy of the best and lates cdm
    * cleanup cdm datasets when uat is created or earlier if they are not needed anymore

### Concepts / Phylosophy ###

The ETL is based on the four steps.
* Clean source data: filter out rows to be not used, format values, apply some business rules. Create intermediate tables with postfix "clean".
* Map distinct source codes to concepts in vocabulary tables. Create intermediate tables with postfix "concept".
    * Custom mapping is implemented in custom concepts generated in vocabulary tables beforehand.
* Join cleaned data and mapped codes. Create intermediate tables with postfix "mapped".
* Distribute mapped data by target cdm tables according to target_domain_id values.

Field unit_id is composed during the ETL steps. From right to left: source table name, initial target table name abbreviation, final target table name or abbreviation. For example: unit_id = 'drug.cond.diagnoses_icd' means that the rows in this unit_id belong to Drug_exposure table, inially were prepared for Condition_occurrence table, and its original is source table diagnoses_icd.




### How do I get set up? ###

* Summary of set up
* Configuration
* Dependencies
* Database configuration
* How to run tests
* Deployment instructions

### Contribution guidelines ###

* Writing tests
* Code review
* Other guidelines
