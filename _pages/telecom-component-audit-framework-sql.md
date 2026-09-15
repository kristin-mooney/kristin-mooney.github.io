-- ============================================================
-- File:    sp_component_audit_details_and_summary_tables.sql
-- Author:  Kristin Mooney | Analytics Portfolio
--          github.com/kristin-mooney
-- Version: 1.1 (portfolio) | Last Updated: 2026-09-14
-- ============================================================

-- NOTE
--   This file contains a portfolio/demonstration version of the
--   stored procedure. Proprietary system names, internal field
--   references, and confidential company information have been
--   abstracted or removed to protect confidential information.
-- ============================================================

ALTER PROCEDURE [aud].[sp_component_audit_details_and_summary_tables]
    @run_id BIGINT = 0
AS

BEGIN TRY

    DECLARE @sp_Name  VARCHAR(100) = 'aud.sp_component_audit_details_and_summary_tables'
    DECLARE @logcount INT          = 1

    INSERT INTO audit.ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'begin', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- ============================================================
    -- STEP 1 | Combine System1->System2 and System1->System3 audits
    -- ============================================================
    --
    -- Joins the two independent audit tables on component + order_id.
    -- A record is marked FAIL if either downstream audit fails.
    -- Records within a 5-day timing window are excluded to avoid
    -- flagging mid-flight orders that have not yet fully propagated.
    -- ============================================================

    TRUNCATE TABLE tmp.component_audit_summary_S1

    INSERT INTO tmp.component_audit_summary_S1
    SELECT DISTINCT
        CAST(GETDATE() AS DATE)                                                 report_date
        , CASE WHEN system2.audit_results = 'FAIL'
                 OR system3.audit_results = 'FAIL'
               THEN 'FAIL' ELSE 'PASS'
          END                                                                AS audit_results
        , CASE WHEN system2.audit_results = 'FAIL'
               THEN system2.scenario ELSE 'CORRECT'
          END                                                                AS system2_failure_code
        , CASE WHEN system3.audit_results = 'FAIL'
               THEN system3.scenario ELSE 'CORRECT'
          END                                                                AS system3_failure_code
        , A.component
        , A.order_id
        , A.component_status
        , A.customer
        , A.ban
        , A.account
        , A.product
        , CASE WHEN A.component_status = 'ACTIVE' THEN A.bsd ELSE A.bed END AS order_dt
        , system2.ticket_status                                              AS system2_ticket_status
        , system2.ticket_type_needed                                         AS system2_ticket_type_needed
        , system3.ticket_status                                              AS system3_ticket_status
        , system3.ticket_type_needed                                         AS system3_ticket_type_needed
    FROM ref.system1_components A WITH (NOLOCK)
    LEFT JOIN (
        SELECT DISTINCT component, order_id, audit_results,
                        scenario, ticket_status, ticket_type_needed
        FROM aud.system1_to_system2 WITH (NOLOCK)
    ) system2 ON a.component = system2.component
              AND a.order_id  = system2.order_id
    LEFT JOIN (
        SELECT DISTINCT component, order_id, audit_results,
                        scenario, ticket_status, ticket_type_needed
        FROM aud.system1_to_system3 WITH (NOLOCK)
    ) system3 ON a.component = system3.component
              AND a.order_id  = system3.order_id
    WHERE
        a.product_cd IN ('product1', 'product2')

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'write table tmp.component_audit_summary_S1', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    DELETE s
    FROM tmp.component_audit_summary_S1 s
    WHERE order_dt >= GETDATE() - 5

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'remove recent order dates', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- ============================================================
    -- STEP 2 | Join risk/root-cause reference; stage audit fields
    -- ============================================================
    --
    -- Joins to a manually maintained reference table that maps
    -- failure code combinations to: issue label, risk tier,
    -- root cause description, recommended solution, and point of
    -- failure. Ticket status is then assigned based on the
    -- solution type returned from the reference table.
    -- ============================================================

    TRUNCATE TABLE tmp.component_audit_results

    INSERT INTO tmp.component_audit_results
    SELECT DISTINCT
        CAST(GETDATE() AS DATE)            report_date,
        A.audit_results,
        A.system2_failure_code          AS system2_scenario,
        A.system3_failure_code          AS system3_scenario,
        A.component,
        A.order_id,
        A.component_status,
        A.customer,
        A.ban,
        A.account,
        A.product,
        A.order_dt,
        S.issue,
        S.risk,
        S.root_cause,
        S.solution,
        S.point_of_failure,
        ''                              AS ticket_status,
        0                               AS system2_rejects,
        0                               AS system3_rejects,
        0                               AS usage_billing_correctly,
        0                               AS mrcs_billing_correctly,
        0                               AS nrcs_billing_correctly,
        0                               AS usage_billing_incorrectly,
        0                               AS mrcs_billing_incorrectly,
        0                               AS nrcs_billing_incorrectly,
        NULL                            AS wrong_account_billed,
        NULL                            AS wrong_ban_billed,
        NULL                            AS wrong_customer_billed
    FROM tmp.component_audit_summary_S1 a WITH (NOLOCK)
    LEFT JOIN ref.component_issue_risk_rootcause_solutions s
        ON a.system2_failure_code = s.system2_failure_code
       AND a.system3_failure_code = s.system3_failure_code
    WHERE
        a.system3_failure_code <> 'DATA INTEGRITY'

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'write table tmp.component_audit_results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    UPDATE t
    SET t.ticket_status = s.system2_ticket_status
    FROM tmp.component_audit_results t
    INNER JOIN tmp.component_audit_summary_S1 s ON t.component = s.component
    WHERE solution = 'Help Desk Ticket'

    UPDATE t
    SET t.ticket_status = s.system3_ticket_status
    FROM tmp.component_audit_results t
    INNER JOIN tmp.component_audit_summary_S1 s ON t.component = s.component
    WHERE solution = 'System3 Ticket'

    UPDATE t
    SET t.ticket_status = s.system2_ticket_status
    FROM tmp.component_audit_results t
    INNER JOIN tmp.component_audit_summary_S1 s ON t.component = s.component
    WHERE solution = 'System2 Ticket'

    UPDATE t
    SET t.ticket_status = s.ticket_status
    FROM tmp.component_audit_results t
    INNER JOIN (
        SELECT component,
               CASE WHEN system2_ticket_status = 'PENDING'
                      OR system3_ticket_status = 'PENDING'
                    THEN 'PENDING' ELSE 'NEED TO SUBMIT TICKET'
               END ticket_status
        FROM tmp.component_audit_summary_S1
    ) s ON t.component = s.component
    WHERE solution = 'System2 and System3 Tickets'

    UPDATE t
    SET t.ticket_status = 'NA'
    FROM tmp.component_audit_results t
    WHERE audit_results = 'PASS'

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'update ticket status to tmp.component_audit_results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- ============================================================
    -- STEP 3 | Populate correct and incorrect billing (USAGE/MRC/NRC)
    -- ============================================================
    --
    -- Correctly billed: PASS + ACTIVE + invoiced after order date
    --   on the matching account.
    --
    -- Incorrectly billed -- two patterns detected:
    --   Pattern A: ACTIVE component billed to the WRONG account
    --   Pattern B: DISCONNECTED component still generating charges
    --
    -- Covers all three charge types: USAGE, MRC (monthly recurring),
    -- and NRC (non-recurring / one-time charges).
    -- ============================================================

    -- USAGE billed correctly
    UPDATE a
    SET usage_billing_correctly = rev
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'USAGE' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component AND a.account = b.account
    WHERE audit_results        = 'PASS'
      AND a.component_status   = 'ACTIVE'    -- BUG FIX 1: was a.component = 'ACTIVE'
      AND a.order_dt           < b.invoice_dt

    -- MRC billed correctly
    UPDATE a
    SET mrcs_billing_correctly = rev
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'MRC' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component AND a.account = b.account
    WHERE audit_results        = 'PASS'
      AND a.component_status   = 'ACTIVE'
      AND a.order_dt           < b.invoice_dt

    -- NRC billed correctly
    UPDATE a
    SET nrcs_billing_correctly = rev
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'NRC' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component
       AND a.account   = b.account          -- BUG FIX 3: was b.acct_no (proprietary field)
    WHERE audit_results        = 'PASS'
      AND a.component_status   = 'ACTIVE'   -- BUG FIX 2: was a.component-status (hyphen)
      AND a.order_dt           < b.invoice_dt

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'update correct billing to tmp.component_audit_results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- USAGE incorrectly billed -- Pattern A: ACTIVE, wrong account
    UPDATE a
    SET usage_billing_incorrectly = rev, wrong_account_billed = b.account
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'USAGE' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component
    WHERE component_status = 'ACTIVE' AND a.account <> b.account AND a.order_dt < b.invoice_dt

    -- USAGE incorrectly billed -- Pattern B: DISCONNECT still billing
    UPDATE a
    SET usage_billing_incorrectly = rev, wrong_account_billed = b.account
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'USAGE' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component
    WHERE component_status = 'DISCONNECT' AND a.account = b.account AND a.order_dt < b.invoice_dt

    -- MRC incorrectly billed -- Pattern A: ACTIVE, wrong account
    UPDATE a
    SET mrcs_billing_incorrectly = rev, wrong_account_billed = b.account
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'MRC' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component
    WHERE component_status = 'ACTIVE' AND a.account <> b.account AND a.order_dt < b.invoice_dt

    -- MRC incorrectly billed -- Pattern B: DISCONNECT still billing
    UPDATE a
    SET mrcs_billing_incorrectly = rev, wrong_account_billed = b.account
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'MRC' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component
    WHERE component_status = 'DISCONNECT'
      AND a.account = b.account             -- BUG FIX 4: was b.acct_no (proprietary field)
      AND a.order_dt < b.invoice_dt

    -- NRC incorrectly billed -- Pattern A: ACTIVE, wrong account
    UPDATE a
    SET nrcs_billing_incorrectly = rev
      , wrong_account_billed     = b.account  -- BUG FIX 5: was b.Aaccount (casing typo)
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'NRC' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component
    WHERE component_status = 'ACTIVE' AND a.account <> b.account AND a.order_dt < b.invoice_dt

    -- NRC incorrectly billed -- Pattern B: DISCONNECT still billing
    UPDATE a
    SET nrcs_billing_incorrectly = rev, wrong_account_billed = b.account
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, account, invoice_dt, SUM(rev) rev
        FROM rpt.component_invoice_detail WITH (NOLOCK)
        WHERE charge_type = 'NRC' AND rev > 0
        GROUP BY component, account, invoice_dt
    ) b ON a.component = b.component
    WHERE component_status = 'DISCONNECT' AND a.account = b.account AND a.order_dt < b.invoice_dt

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'update incorrect billing to tmp.component_audit_results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- ============================================================
    -- STEP 4 | Match failed components to System 2 & System 3 rejects
    -- ============================================================
    --
    -- Joins failed components from audit results to downstream
    -- reject logs. System 2 rejects are staged in a temp table
    -- first, then aggregated into the final reject table with
    -- estimated revenue. System 3 rejects are inserted directly.
    -- Indexes are disabled before bulk inserts and rebuilt after
    -- to maximize insert performance.
    -- ============================================================

    TRUNCATE TABLE rej.component_system3_rejects

    ALTER INDEX [IX_COMPONENT] ON rej.component_system3_rejects DISABLE
    ALTER INDEX [IX_CUSTOMER]  ON rej.component_system3_rejects DISABLE
    ALTER INDEX [IX_ACCOUNT]   ON rej.component_system3_rejects DISABLE
    ALTER INDEX [IX_ISSUE]     ON rej.component_system3_rejects DISABLE

    INSERT INTO rej.component_system3_rejects
    SELECT DISTINCT
        CAST(GETDATE() AS DATE)                    report_date,
        source,
        external_id                             AS component,
        customer, ban, account, issue, ticket_status,
        miu_error_code1                         AS ec,
        SUM(sum_e_est_post_discount_amount)     AS rev,
        MIN(CAST(trans_dt AS DATE))                first_trans_dt,
        MAX(CAST(trans_dt AS DATE))                last_trans_dt,
        @run_id                                 -- BUG FIX 6: was @ssis_run_id (undeclared)
    FROM rej.system3_reject_summary r WITH (NOLOCK)
    INNER JOIN (
        SELECT DISTINCT component, customer, ban, account, issue, ticket_status
        FROM tmp.component_audit_results        -- BUG FIX 8: was tmp.components_audit_results
        WHERE audit_results = 'FAIL'
    ) h ON r.external_id = h.component
    WHERE r.source = 'System3'
    GROUP BY source, external_id, customer, ban, account, issue, ticket_status, miu_error_code1

    ALTER INDEX [IX_COMPONENT] ON rej.component_system3_rejects REBUILD
    ALTER INDEX [IX_CUSTOMER]  ON rej.component_system3_rejects REBUILD
    ALTER INDEX [IX_ACCOUNT]   ON rej.component_system3_rejects REBUILD
    ALTER INDEX [IX_ISSUE]     ON rej.component_system3_rejects REBUILD

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'write table rej.component_system3_rejects', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    TRUNCATE TABLE tmp.component_system2_reject

    INSERT INTO tmp.component_system2_reject
    SELECT DISTINCT
        CAST(GETDATE() AS DATE)        report_date,
        carrier, component, customer, ban,
        h.account, h.issue, h.ticket_status,
        error_code                  AS ec,
        SUM(calldur) / 60           AS mou,
        MIN(CAST(discdate AS DATE))    fcd,
        MAX(CAST(discdate AS DATE))    lcd
    FROM rej.system2_reject r WITH (NOLOCK)
    INNER JOIN (
        SELECT DISTINCT component, customer, ban, account, issue, ticket_status
        FROM tmp.component_audit_results
        WHERE audit_results = 'FAIL'
    ) h ON r.component = h.component
    GROUP BY carrier, component, customer, ban, h.account, h.issue, h.ticket_status, error_code

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'write table tmp.component_system2_reject', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    TRUNCATE TABLE rej.component_system2_rejects

    ALTER INDEX [IX_COMPONENT] ON rej.component_system2_rejects DISABLE
    ALTER INDEX [IX_CUSTOMER]  ON rej.component_system2_rejects DISABLE
    ALTER INDEX [IX_ACCOUNT]   ON rej.component_system2_rejects DISABLE
    ALTER INDEX [IX_ISSUE]     ON rej.component_system2_rejects DISABLE

    INSERT INTO rej.component_system2_rejects
    SELECT DISTINCT
        CAST(GETDATE() AS DATE)        report_date,
        carrier, component, customer, ban, account, issue, ticket_status, ec,
        SUM(mou) * 0.005            AS est_rev,  -- BUG FIX 7: removed stray leading comma
        MIN(fcd)                       fcd,
        MAX(lcd)                       lcd,
        @run_id                                  -- BUG FIX 9: was @ssis_run_id (undeclared)
    FROM tmp.component_system2_reject
    GROUP BY carrier, component, customer, ban, account, issue, ticket_status, ec

    ALTER INDEX [IX_COMPONENT] ON rej.component_system2_rejects REBUILD
    ALTER INDEX [IX_CUSTOMER]  ON rej.component_system2_rejects REBUILD
    ALTER INDEX [IX_ACCOUNT]   ON rej.component_system2_rejects REBUILD
    ALTER INDEX [IX_ISSUE]     ON rej.component_system2_rejects REBUILD

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'write table rej.component_system2_rejects', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    UPDATE a
    SET system2_rejects = rev
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, SUM(est_rev) rev
        FROM rej.component_system2_rejects WITH (NOLOCK)
        GROUP BY component
    ) b ON a.component = b.component
    WHERE audit_results = 'FAIL'

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'update with system 2 rejects results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    UPDATE a
    SET system3_rejects = rev
    FROM tmp.component_audit_results a
    INNER JOIN (
        SELECT DISTINCT component, SUM(rev) rev
        FROM rej.component_system3_rejects WITH (NOLOCK)
        GROUP BY component
    ) b ON a.component = b.component
    WHERE audit_results = 'FAIL'

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'update with system3_rejects results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    UPDATE r
    SET wrong_ban_billed = b.ban, wrong_customer_billed = b.bill_company
    FROM tmp.component_audit_results r
    INNER JOIN (
        SELECT account, ban, bill_company FROM ref.billing_customers WITH (NOLOCK)
    ) b ON r.wrong_account_billed = b.account

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'update wrong account details', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- ============================================================
    -- STEP 5 | Materialize detail table + risk tiers + summary
    -- ============================================================
    --
    -- Writes all staged results into the final detail table, then
    -- applies risk-tier assessment via a waterfall UPDATE pattern
    -- (first match wins -- order of UPDATEs is intentional):
    --
    --   HIGH RISK   -- component is actively billing incorrectly
    --   MEDIUM RISK -- not billing incorrectly, but generating rejects
    --   LOW RISK    -- failed audit with no active financial impact
    --   NO RISK     -- all passing components
    -- ============================================================

    TRUNCATE TABLE aud.component_audit_results

    ALTER INDEX [IX_CUSTOMER]       ON aud.component_audit_results DISABLE
    ALTER INDEX [IX_BAN]            ON aud.component_audit_results DISABLE
    ALTER INDEX [IX_ACCOUNT]        ON aud.component_audit_results DISABLE
    ALTER INDEX IX_COMPONENT        ON aud.component_audit_results DISABLE
    ALTER INDEX [IX_ISSUE]          ON aud.component_audit_results DISABLE
    ALTER INDEX [IX_ORDER_DT]       ON aud.component_audit_results DISABLE
    ALTER INDEX IX_COMPONENT_STATUS ON aud.component_audit_results DISABLE

    INSERT INTO aud.component_audit_results
    SELECT DISTINCT
        CAST(GETDATE() AS DATE)  report_date,
        audit_results,
        system2_scenario,
        system3_scenario,
        component,               -- BUG FIX 10: was 'compnent' (typo)
        order_id,
        component_status,
        customer, ban, account, product, order_dt,
        issue, risk, root_cause, solution, point_of_failure,
        ''                    AS assessment,  -- BUG FIX 11: was 'assesement' (typo)
        ticket_status,
        system2_rejects, system3_rejects,
        usage_billing_correctly, mrcs_billing_correctly, nrcs_billing_correctly,
        usage_billing_incorrectly, mrcs_billing_incorrectly, nrcs_billing_incorrectly,
        wrong_account_billed, wrong_ban_billed, wrong_customer_billed,
        @run_id
    FROM tmp.component_audit_results

    ALTER INDEX [IX_CUSTOMER]       ON aud.component_audit_results REBUILD
    ALTER INDEX [IX_BAN]            ON aud.component_audit_results REBUILD
    ALTER INDEX [IX_ACCOUNT]        ON aud.component_audit_results REBUILD
    ALTER INDEX IX_COMPONENT        ON aud.component_audit_results REBUILD
    ALTER INDEX [IX_ISSUE]          ON aud.component_audit_results REBUILD
    ALTER INDEX [IX_ORDER_DT]       ON aud.component_audit_results REBUILD
    ALTER INDEX IX_COMPONENT_STATUS ON aud.component_audit_results REBUILD

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'write table aud.component_audit_results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- Risk tier waterfall (order is intentional -- first match wins)

    UPDATE a SET assessment = 'HIGH RISK'   -- actively billing incorrectly
    FROM aud.component_audit_results a
    WHERE audit_results = 'FAIL'
      AND (usage_billing_incorrectly > 0 OR mrcs_billing_incorrectly > 0 OR nrcs_billing_incorrectly > 0)

    UPDATE a SET assessment = 'MEDIUM RISK' -- generating rejects, not billing incorrectly
    FROM aud.component_audit_results a
    WHERE audit_results = 'FAIL' AND assessment = ''
      AND (system2_rejects > 0 OR system3_rejects > 0)

    UPDATE a SET assessment = 'LOW RISK'    -- failed audit, no active financial impact
    FROM aud.component_audit_results a
    WHERE audit_results = 'FAIL' AND assessment = ''

    UPDATE a SET assessment = 'NO RISK'     -- all passing components
    FROM aud.component_audit_results a
    WHERE audit_results = 'PASS' AND assessment = ''

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'add risk assessments to aud.component_audit_results', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- Power BI summary table (aggregated grain)
    TRUNCATE TABLE aud.component_audit_summary

    ALTER INDEX [IX_AUDIT_RESULTS]    ON aud.component_audit_summary DISABLE
    ALTER INDEX [IX_COMPONENT_STATUS] ON aud.component_audit_summary DISABLE
    ALTER INDEX [IX_CUSTOMER]         ON aud.component_audit_summary DISABLE
    ALTER INDEX [IX_ACCOUNT]          ON aud.component_audit_summary DISABLE
    ALTER INDEX [IX_ORDER_DT]         ON aud.component_audit_summary DISABLE
    ALTER INDEX [IX_ISSUE]            ON aud.component_audit_summary DISABLE
    ALTER INDEX [IX_POINT_OF_FAILURE] ON aud.component_audit_summary DISABLE

    INSERT INTO aud.component_audit_summary
    SELECT DISTINCT
        CAST(GETDATE() AS DATE)  report_date,
        audit_results, component_status, customer, ban, account, product, order_dt,
        issue, risk, root_cause, solution, point_of_failure, assessment,
        SUM(system2_rejects + system3_rejects)                           AS rejects,
        SUM(usage_billing_incorrectly + mrcs_billing_incorrectly
            + nrcs_billing_incorrectly)                                  AS incorrect_billing,
        SUM(usage_billing_correctly + mrcs_billing_correctly
            + nrcs_billing_correctly)                                    AS correct_billing,
        COUNT(*)                                                         AS cnt,
        MIN(component)                                                   AS example,
        @run_id                 -- BUG FIX 12: was @ssis_run_id (undeclared variable)
    FROM aud.component_audit_results
    GROUP BY
        audit_results, component_status, customer, ban, account, product, order_dt,
        issue, risk, root_cause, solution, point_of_failure, assessment

    ALTER INDEX [IX_AUDIT_RESULTS]    ON aud.component_audit_summary REBUILD
    ALTER INDEX [IX_COMPONENT_STATUS] ON aud.component_audit_summary REBUILD
    ALTER INDEX [IX_CUSTOMER]         ON aud.component_audit_summary REBUILD
    ALTER INDEX [IX_ACCOUNT]          ON aud.component_audit_summary REBUILD
    ALTER INDEX [IX_ORDER_DT]         ON aud.component_audit_summary REBUILD
    ALTER INDEX [IX_ISSUE]            ON aud.component_audit_summary REBUILD
    ALTER INDEX [IX_POINT_OF_FAILURE] ON aud.component_audit_summary REBUILD

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'write table aud.component_audit_summary', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    -- ============================================================
    -- STEP 6 | Truncate temp tables
    -- ============================================================

    TRUNCATE TABLE tmp.component_audit_summary_S1
    TRUNCATE TABLE tmp.component_audit_results
    TRUNCATE TABLE tmp.component_system2_reject
    TRUNCATE TABLE tmp.component_system3_reject

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'truncate tables', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, @logcount, 'end', GETDATE(), @run_id)
    SET @logcount = @logcount + 1

END TRY

BEGIN CATCH

    INSERT INTO ctl.sp_log (sp_name, step, description, complete_date, ssis_run_id)
    VALUES (@sp_Name, 999,
            'On batch line: ' + CAST(ERROR_LINE() AS VARCHAR(10))
            + ', error: ' + ERROR_MESSAGE(), GETDATE(), @run_id)

    DECLARE @step_prior_to_fail BIGINT
    SET @step_prior_to_fail = (
        SELECT MAX(step) FROM ctl.sp_log x
        WHERE x.sp_name = @sp_Name
          AND complete_date = (
              SELECT MAX(complete_date) FROM ctl.sp_log x
              WHERE x.sp_name = @sp_Name AND x.step != 999
          )
          AND step != 999
    )

    INSERT INTO ctl.sp_error_log (
         sp_name, step_prior_to_fail, ssis_run_id
        ,ErrorNumber, ErrorSeverity, ErrorState
        ,ErrorProcedure, ErrorLine, ErrorMessage, complete_date
    )
    VALUES (
         @sp_Name, @step_prior_to_fail, @run_id
        ,ERROR_NUMBER(), ERROR_SEVERITY(), ERROR_STATE()
        ,ERROR_PROCEDURE(), ERROR_LINE(), ERROR_MESSAGE(), GETDATE()
    )

    DECLARE @ErrorText VARCHAR(512)
    SET @ErrorText = ERROR_MESSAGE() + ' occurred at Line_Number: '
                     + CAST(ERROR_LINE() AS VARCHAR(50))
    RAISERROR (@ErrorText, 16, 1)

END CATCH;
