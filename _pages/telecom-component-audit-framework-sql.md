```sql
USE [Audit]
GO
/****** Object:  StoredProcedure [aud].[sp_component_audit_details_and_summary_tables]    Script Date: 9/14/2026 9:51:50 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

--CREATE 
--ALTER 
--EXEC --execute EXEC [aud].[sp_component_audit_details_and_summary_tables] 

ALTER  PROCEDURE [aud].[sp_component_audit_details_and_summary_tables]
@run_id bigint = 0
AS

/*
HIGH LEVEL PURPOSE: This sp combines the results from the System 1 to System 2 audit and the System 1 to System 3 audits.  This sp
creates a detailed end table with insights into rejects, incorrect billing, and risk assessment of the component.  The summary table is built
for the powerbi report and to maintain history of issues and issue counts.  


STEP 1 - combine audit tables from System 1 to System 2 audit and the System 1 to System 3 audits
STEP 2 - joins to risk assessment table created above and enters in fields for final audit table
STEP 3 - enter in data for rev that billed correctly and incorrectly
STEP 4 - reject matches to System 2 and System 3 
STEP 5 - Summary table for PowerBI 

Data sources:
select top 5* from audit.aud.system1_to_system2 WITH(NOLOCK) 
select top 5* from audit.aud.system1_to_system3 WITH(NOLOCK) 
SELECT TOP 5* FROM audit.rpt.invoice_detail WITH(NOLOCK)

Manual reference table created to create more precise issue descriptions: audit.ref.components_issue_risk_rootcause_solutions

/*end tables created*/
select top 5* from audit.aud.component_audit_results
select top 5* from audit.aud.component_audit_summary


/*temp table list*/
select top 5* from audit.tmp.component_audit_summary_S1
select top 5* from audit.tmp.component_audit_results
select top 5* from audit.tmp.component_rejects
SELECT TOP 5* from audit.tmp.component_audit_results
*/


BEGIN TRY

declare @sp_Name as varchar(100) = 'aud.sp_component_audit_details_and_summary_tables'


declare @logcount int = 1

----add time to log
insert into audit.ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'begin', getdate(),@run_id) 
set @logcount = @logcount + 1


-------------------------------------------------------------------------------------------------------------------
/*STEP 1 - combine audit tables from system 1 to system 2 and system 1 to system 3 audits*/
--eventually want field for point of failure, issue, root cause, solution instead of scenarios in this summarized audit table
TRUNCATE TABLE tmp.component_audit_summary_S1
INSERT INTO tmp.component_audit_summary_S1
SELECT distinct
	cast(getdate() as date) report_date
	, CASE WHEN system2.audit_results = 'FAIL' OR system3.audit_results = 'FAIL' THEN 'FAIL' ELSE 'PASS' END AS audit_results
	, CASE WHEN system2.audit_results = 'FAIL' THEN system2.scenario ELSE 'CORRECT' END system2_failure_code
	, CASE WHEN system3.audit_results = 'FAIL' THEN system3.scenario ELSE 'CORRECT' END system3_failure_code
	, A.component
	, A.order_id
	, A.component_status
	, A.customer
	, A.ban
	, A.account
	, A.product
	, CASE WHEN A.component_status = 'ACTIVE' THEN A.bsd ELSE A.bed END ORDER_DT
	, system2.ticket_status AS system2_ticket_status
	, system2.ticket_type_needed AS system2_ticket_type_needed 
	, system3.ticket_status AS system3_ticket_status
	, system3.ticket_type_needed AS system3_ticket_type_needed
FROM ref.system1_components A with(nolock)
LEFT JOIN (SELECT DISTINCT component, order_id, audit_results, scenario, ticket_status, ticket_type_needed 	FROM aud.system1_to_system2 WITH(NOLOCK)) system2
	ON a.component = system2.component and a.order_id = system2.order_id
LEFT JOIN (SELECT DISTINCT component, order_id, audit_results, scenario, ticket_status, ticket_type_needed FROM aud.system1_to_system3 WITH(NOLOCK)) system3
	ON a.component = system3.component and a.order_id = system3.order_id
WHERE 
	a.product_cd IN ('product1', 'product2')

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'write table tmp.component_audit_summary_S1', getdate(),@run_id) 
set @logcount = @logcount + 1

--remove where possible timing gap
DELETE s
--select *
FROM tmp.component_audit_summary_S1 s
WHERE
	order_dt >= getdate()-5

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'remove recent order dates', getdate(),@run_id) 
set @logcount = @logcount + 1

------------------------------------------------------------------------------------------------------------
/*STEP 2 - joins to risk assessment table created above and enters in fields for final audit table*/

--5 min
TRUNCATE TABLE tmp.component_audit_results
INSERT into tmp.component_audit_results
SELECT DISTINCT 
	cast(getdate() as date) report_date,
	A.audit_results,
	A.system2_failure_code AS system2_scenario,
	A.system3_failure_code AS system3_scenario,
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
	'' AS ticket_status ,
	0 as system2_rejects,
	0 AS system3_rejects,
	0 AS usage_billing_correctly,
	0 AS mrcs_billing_correctly,
	0 AS nrcs_billing_correctly,
	0 AS usage_billing_incorrectly,
	0 AS mrcs_billing_incorrectly,
	0 AS nrcs_billing_incorrectly,
	NULL AS wrong_account_billed,
	NULL AS wrong_ban_billed,
	NULL AS wrong_customer_billed
--select top 5*
FROM tmp.component_audit_summary_S1 a with(nolock)
LEFT JOIN ref.component_issue_risk_rootcause_solutions s
	ON a.system2_failure_code = s.system2_failure_code and a.system3_failure_code = s.system3_failure_code
WHERE 
	a.system3_failure_code <> 'DATA INTEGRITY'

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'write table tmp.component_audit_results', getdate(),@run_id) 
set @logcount = @logcount + 1


UPDATE t 
SET t.ticket_status = s.system2_ticket_status
FROM tmp.component_audit_results t
INNER JOIN tmp.component_audit_summary_S1 s
	ON t.component = s.component
WHERE 
	solution = 'Help Desk Ticket'


UPDATE t 
SET t.ticket_status = s.system3_ticket_status
FROM tmp.component_audit_results t
INNER JOIN tmp.component_audit_summary_S1 s
	ON t.component = s.component
WHERE
	solution = 'System3 Ticket'

UPDATE t 
SET t.ticket_status = s.system2_ticket_status
FROM tmp.component_audit_results t
INNER JOIN tmp.component_audit_summary_S1 s
	ON t.component = s.component
WHERE 
	solution = 'System2 Ticket'

UPDATE t 
SET t.ticket_status = s.ticket_status
--select *
FROM tmp.component_audit_results t
INNER JOIN (select component, case when system2_ticket_status = 'PENDING' OR system3_ticket_status = 'PENDING' THEN 'PENDING' ELSE 'NEED TO SUBMIT TICKET' END ticket_status FROM tmp.component_audit_summary_S1) s
	ON t.component = s.component
WHERE 
	solution = 'System2 and System3 Tickets'--erds issue so in theory kenan is correct but UP is not

update t 
SET t.ticket_status = 'NA' 
--select *
FROM tmp.component_audit_results t
WHERE 
	audit_results = 'PASS'

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'update ticket status to tmp.component_audit_results', getdate(),@run_id) 
set @logcount = @logcount + 1


----------------------------------------------------------------------------------------------------------
/*step 3 - enter in data for rev that billed correctly and incorrectly */


--enter in usage that billed correctly
UPDATE a
SET usage_billing_correctly = rev
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN 
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.invoice_detail with(nolock) 
		WHERE charge_type = 'USAGE' 
		and rev > 0
		GROUP BY component, account, invoice_dt) b
	ON a.component = b.component
	and a.account = b.account
WHERE 
	audit_results = 'PASS'
	and a.component = 'ACTIVE'
	and a.order_dt < b.invoice_dt--before the last invoice


UPDATE a
SET mrcs_billing_correctly = rev
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'MRC' 
		and rev > 0
		GROUP BY  component, account, invoice_dt) b
	ON a.component = b.component 	and a.account = b.account
WHERE 
	audit_results = 'PASS'
	and a.component_status = 'ACTIVE'
	and a.order_dt < b.invoice_dt--before the last invoice

--nrcs that billed correctly
UPDATE a
SET nrcs_billing_correctly = rev
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'NRC' 
		and rev > 0
		GROUP BY  component, account, invoice_dt) b
	ON a.component = b.component and a.account = b.kenan_acct_no
WHERE 
	audit_results = 'PASS'
	and a.component-status = 'ACTIVE'
	and a.order_dt < b.invoice_dt--before the last invoice

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'update correct billing to tmp.component_audit_results', getdate(),@run_id) 
set @logcount = @logcount + 1

-------------------------------------------------------------------------
--enter in usage that billed incorrectly
UPDATE a
SET usage_billing_incorrectly = rev
, wrong_account_billed = b.account
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN 
		(SELECT distinct component, account, invoice_dt
		, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'USAGE' 
		and rev > 0
		GROUP BY  component, account, invoice_dt) b
	ON a.component = b.component
WHERE 
	component_status = 'ACTIVE'
	and A.account <> B.account
	and a.order_dt < b.invoice_dt--before the last invoice


--ALSO CHANCE THAT DISCO BILLS WHEN IT SHOULDN'T
UPDATE a
SET usage_billing_incorrectly = rev
, wrong_account_billed = b.account
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'USAGE' 
		and rev > 0
		GROUP BY  component, account, invoice_dt) b
	ON a.component = b.component
WHERE 
	component_status = 'DISCONNECT'
	and A.account = B.account
	and a.order_dt < b.invoice_dt--before the last invoice


--enter in MRCS that billed incorrectly
UPDATE a
SET mrcs_billing_incorrectly = rev
, wrong_account_billed = b.account
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN 
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'MRC' 
		and rev > 0
		GROUP BY component, account, invoice_dt) b
	ON a.component = b.component
WHERE 
	component_status = 'ACTIVE'
	and a.account <> b.account
	and a.order_dt < b.invoice_dt--before the last invoice


--ALSO CHANCE THAT DISCO BILLS WHEN IT SHOULDN'T
UPDATE a
SET mrcs_billing_incorrectly = rev
, wrong_account_billed = b.account
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'MRC' 
		and rev > 0
		GROUP BY component, account, invoice_dt) b
	ON a.component = b.component
WHERE 
	component_status = 'DISCONNECT'
	AND a.account = b.kenan_acct_no
	and a.order_dt < b.invoice_dt--before the last invoice

--------------
--enter in NRCS that billed incorrectly
UPDATE a
SET nrcs_billing_incorrectly = rev
, wrong_account_billed = b.Aaccount
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'NRC' 
		and rev > 0
		GROUP BY component, account, invoice_dt) b
	ON a.component = b.component
WHERE 
	component_status = 'ACTIVE'
	and a.account <> b.account
	and a.order_dt < b.invoice_dt--before the last invoice


--ALSO CHANCE THAT DISCO BILLS WHEN IT SHOULDN'T
UPDATE a
SET nrcs_billing_incorrectly = rev
, wrong_account_billed = b.account
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, account, invoice_dt, sum(rev) rev 
		FROM rpt.component_invoice_detail with(nolock) 
		WHERE charge_type = 'NRC' and rev > 0 group by component, account, invoice_dt) b
	ON a.component = b.component
WHERE 
	component_status = 'DISCONNECT'
	AND a.account = b.account
	and a.order_dt < b.invoice_dt--before the last invoice

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'update incorrect billing to tmp.component_audit_results', getdate(),@run_id) 
set @logcount = @logcount + 1




---------------------------------------------------------------------------------------------------------------
/* STEP 4 - REJECT AND BILLING MATCHES --BOTH UP AND KENAN*/

/*
CREATE INDEX [IX_COMPONENT] ON rej.component_system3_rejects (COMPONENT)
CREATE INDEX [IX_CUSTOMER] ON rej.component_system3_rejects (CUSTOMER)
CREATE INDEX [IX_ACCOUNT] ON rej.component_system3_rejects (ACCOUNT)
CREATE INDEX [IX_ISSUE] ON rej.component_system3_rejects (ISSUE)
*/

--system3 reject check
TRUNCATE TABLE rej.component_system3_rejects

ALTER INDEX [IX_COMPONENT] ON rej.component_system3_rejects DISABLE
ALTER INDEX [IX_CUSTOMER] ON rej.component_system3_rejects DISABLE
ALTER INDEX [IX_ACCOUNT] ON rej.component_system3_rejects DISABLE
ALTER INDEX [IX_ISSUE] ON rej.component_system3_rejects DISABLE

INSERT INTO rej.component_system3_rejects
SELECT distinct
	cast(getdate() as date) report_date,
	source,
	external_id as component,
	customer,
	ban,
	account,
	issue,
	ticket_status,
	miu_error_code1 as ec,
	sum(sum_e_est_post_discount_amount) as rev,
	min(cast(trans_dt as date)) first_trans_dt,
	max(cast(trans_dt as date)) last_trans_dt,
	@ssis_run_id
FROM rej.system3_reject_summary r with(nolock)
	INNER JOIN 
		(SELECT distinct component, customer, ban, account, issue, ticket_status
		FROM tmp.components_audit_results 
		WHERE audit_results = 'FAIL') h
	ON r.external_id = h.component
WHERE 
	r.source = 'System3' 
GROUP BY
	source,
	external_id ,
	customer,
	ban,
	account,
	issue,
	ticket_status,
	miu_error_code1

	
ALTER INDEX [IX_COMPONENT] ON rej.component_system3_rejects REBUILD
ALTER INDEX [IX_CUSTOMER] ON rej.component_system3_rejects REBUILD
ALTER INDEX [IX_ACCOUNT] ON rej.component_system3_rejects REBUILD
ALTER INDEX [IX_ISSUE] ON rej.component_system3_rejects REBUILD

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'write table rej.component_system3_rejects', getdate(),@run_id) 
set @logcount = @logcount + 1


--system2 rejects
TRUNCATE TABLE tmp.component_system2_reject
INSERT INTO tmp.component_system2_reject
SELECT distinct
	cast(getdate() as date) report_date,
	carrier,
	component,
	customer,
	ban,
	h.account,
	h.issue,
	h.ticket_status,
	error_code as ec,
	sum(calldur)/60 as mou,
	min(cast(discdate as date)) fcd,
	max(cast(discdate as date)) lcd
FROM rej.system2_reject r with(nolock)
INNER JOIN 
		(SELECT distinct component, customer, ban, account, issue, ticket_status
		FROM tmp.component_audit_results 
		WHERE audit_results = 'FAIL') h
	ON r.component = h.component
GROUP BY
	carrier
	, component 
	, customer
	, ban
	, h.account
	, h.issue
	, h.ticket_status
	, error_code


insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'write table tmp.component_system2_reject', getdate(),@run_id) 
set @logcount = @logcount + 1

/*
CREATE INDEX [IX_COMPONENT] ON rej.component_system2_rejects (COMPONENT)
CREATE INDEX [IX_CUSTOMER] ON rej.component_system2_rejects (CUSTOMER)
CREATE INDEX [IX_ACCOUNT] ON rej.component_system2_rejects (ACCOUNT)
CREATE INDEX [IX_ISSUE] ON rej.component_system2_rejects (ISSUE)
*/


TRUNCATE TABLE rej.component_system2_rejects

ALTER INDEX [IX_COMPONENT] ON rej.component_system2_rejects DISABLE
ALTER INDEX [IX_CUSTOMER] ON rej.component_system2_rejects DISABLE
ALTER INDEX [IX_ACCOUNT] ON rej.component_system2_rejects DISABLE
ALTER INDEX [IX_ISSUE] ON rej.component_system2_rejects DISABLE


INSERT INTO rej.component_system2_rejects
SELECT DISTINCT 
	cast(getdate() as date) report_date,
	carrier,
	component,
	customer,
	ban,
	account,
	issue,
	ticket_status,
	ec,
	, SUM(MOU)*.005 AS est_rev
	, MIN(FCD) fcd
	, MAX(LCD) lcd
	, @ssis_run_id
FROM tmp.component_system2_reject
GROUP BY
	carrier,
	component,
	customer,
	ban,
	account,
	issue,
	ticket_status,
	ec

ALTER INDEX [IX_COMPONENT] ON rej.component_system2_rejects REBUILD
ALTER INDEX [IX_CUSTOMER] ON rej.component_system2_rejects REBUILD
ALTER INDEX [IX_ACCOUNT] ON rej.component_system2_rejects REBUILD
ALTER INDEX [IX_ISSUE] ON rej.component_system2_rejects REBUILD



insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'write table rej.component_system2_rejects', getdate(),@run_id) 
set @logcount = @logcount + 1


UPDATE a
SET system2_rejects = rev
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, sum(est_rev) rev 
		FROM rej.component_system2_rejects with(nolock) 
		GROUP BY component) b
	ON a.component = b.component
WHERE 
	audit_results = 'FAIL'

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'update with system 2 rejects results', getdate(),@run_id) 
set @logcount = @logcount + 1

UPDATE a
SET system3_rejects = rev
--select top 5* 
FROM tmp.component_audit_results a
INNER JOIN
		(SELECT distinct component, sum(rev) rev 
		FROM rej.component_system3_rejects with(nolock) 
		GROUP BY component) b
	ON a.component = b.component
WHERE 
	audit_results = 'FAIL'

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'update with system3_rejects results', getdate(),@run_id) 
set @logcount = @logcount + 1



/*update customer information*/
UPDATE r
SET wrong_ban_billed = b.ban
, wrong_customer_billed = b.bill_company
FROM tmp.component_audit_results r
INNER JOIN (SELECT account, ban, bill_company from ref.billing_customers with(nolock)) b 
	ON r.WRONG_ACCOUNT_BILLED = b.account	

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'update wrong account details', getdate(),@run_id) 
set @logcount = @logcount + 1



/*
create indexes

CREATE INDEX [IX_CUSTOMER] ON aud.component_audit_results (CUSTOMER)
CREATE INDEX [IX_BAN] ON aud.component_audit_results (BAN)
CREATE INDEX [IX_ACCOUNT] ON aud.component_audit_results (ACCOUNT)
CREATE INDEX [IX_COMPONENT] ON aud.component_audit_results (COMPONENT)
CREATE INDEX [IX_ISSUE] ON aud.component_audit_results (ISSUE)
CREATE INDEX [IX_ORDER_DT] ON aud.component_audit_results (ORDER_DT)
CREATE INDEX [IX_COMPONENT_STATUS] ON aud.component_audit_results (COMPONENT_STATUS)

*/

TRUNCATE TABLE aud.component_audit_results

ALTER INDEX [IX_CUSTOMER] ON aud.component_audit_results DISABLE
ALTER INDEX [IX_BAN] ON aud.component_audit_results DISABLE
ALTER INDEX [IX_ACCOUNT] ON aud.component_audit_results DISABLE
ALTER INDEX IX_COMPONENT ON aud.component_audit_results DISABLE
ALTER INDEX [IX_ISSUE] ON aud.component_audit_results DISABLE
ALTER INDEX [IX_ORDER_DT] ON aud.component_audit_results DISABLE
ALTER INDEX IX_COMPONENT_STATUS ON aud.component_audit_results DISABLE

insert into aud.component_audit_results
SELECT DISTINCT
	cast(getdate() as date) report_date,
	audit_results,
	system2_scenario,
	system3_scenario,
	compnent,
	order_id,
	component_status,
	customer,
	ban,
	account,
	product,
	order_dt,
	issue,
	risk,
	root_cause,
	solution,
	point_of_failure,
	'' as assesement,
	ticket_status,
	system2_rejects,
	system3_rejects,
	usage_billing_correctly,
	mrcs_billing_correctly,
	nrcs_billing_correctly,
	usage_billing_incorrectly,
	mrcs_billing_incorrectly,
	nrcs_billing_incorrectly,
	wrong_account_billed,
	wrong_ban_billed,
	wrong_customer_billed,
	@ssis_run_id
from tmp.component_audit_results

ALTER INDEX [IX_CUSTOMER] ON aud.component_audit_results REBUILD
ALTER INDEX [IX_BAN] ON aud.component_audit_results REBUILD
ALTER INDEX [IX_ACCOUNT] ON aud.component_audit_results REBUILD
ALTER INDEX IX_COMPONENT ON aud.component_audit_results REBUILD
ALTER INDEX [IX_ISSUE] ON aud.component_audit_results REBUILD
ALTER INDEX [IX_ORDER_DT] ON aud.component_audit_results REBUILD
ALTER INDEX IX_COMPONENT_STATUS ON aud.component_audit_results REBUILD

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'write table aud.component_audit_results', getdate(),@run_id) 
set @logcount = @logcount + 1


--HIGH RISK BC BILLING INCORRECTLY
UPDATE a
SET assessment = 'HIGH RISK'
--SELECT 
FROM aud.component_audit_results a
WHERE audit_results = 'FAIL' 
and (usage_billing_incorrectly >0 or mrcs_billing_incorrectly >0 OR nrcs_billing_incorrectly >0)

--MEDIUM RISK--NOT BILLING INCORRECTLY BUT IN REJECTS
UPDATE a
set assessment = 'MEDIUM RISK'
--SELECT *
FROM aud.component_audit_results a
WHERE audit_results = 'FAIL' 
AND assessment = ''
AND (system2_rejects >0 OR system3_rejects >0)

--LOW RISK
UPDATE a
set assessment = 'LOW RISK'
--SELECT 
FROM aud.component_audit_results a
WHERE audit_results = 'FAIL' 
AND assessment = ''

--NO RISK
UPDATE a
set assessment = 'NO RISK'
--SELECT 
FROM aud.component_audit_results a
WHERE audit_results = 'PASS' 
AND assessment = ''

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'add risk assessments to aud.component_audit_results', getdate(),@run_id) 
set @logcount = @logcount + 1

--------------------------------------------------------------------------------------------------------------------------------------
/*STEP 5 SUMMARY TABLE FOR POWERBI*/
--HIGH LEVEL_SUMMARY
/*
CREATE INDEX [IX_AUDIT_RESULTS] ON aud.component_audit_summary (AUDIT_RESULTS)
CREATE INDEX [IX_COMPONENT_STATUS] ON aud.component_audit_summary (COMPONENT_STATUS)
CREATE INDEX [IX_CUSTOMER] ON aud.component_audit_summary (CUSTOMER)
CREATE INDEX [IX_ACCOUNT] ON aud.component_audit_summary (ACCOUNT)
CREATE INDEX [IX_ORDER_DT] ON aud.component_audit_summary (ORDER_DT)
CREATE INDEX [IX_ISSUE] ON aud.component_audit_summary (ISSUE)
CREATE INDEX [IX_POINT_OF_FAILURE] ON aud.component_audit_summary (POINT_OF_FAILURE)

*/

TRUNCATE TABLE aud.component_audit_summary

ALTER INDEX [IX_AUDIT_RESULTS] ON aud.component_audit_summary DISABLE
ALTER INDEX [IX_COMPONENT_STATUS] ON aud.component_audit_summary DISABLE
ALTER INDEX [IX_CUSTOMER] ON aud.component_audit_summary DISABLE
ALTER INDEX [IX_ACCOUNT] ON aud.component_audit_summary DISABLE
ALTER INDEX [IX_ORDER_DT] ON aud.component_audit_summary DISABLE
ALTER INDEX [IX_ISSUE] ON aud.component_audit_summary DISABLE
ALTER INDEX [IX_POINT_OF_FAILURE] ON aud.component_audit_summary DISABLE

INSERT INTO aud.component_audit_summary
SELECT DISTINCT 
	cast(getdate() as date) report_date,
	audit_Results,
	component_status,
	customer,
	ban,
	account,
	product,
	order_dt,
	issue,
	risk,
	root_cause,
	solution,
	point_of_failure,
	assessment,
	SUM(system2_rejects+system3_rejects) AS rejects,
	SUM(USAGE_BILLING_INCORRECTLY+MRCS_BILLING_INCORRECTLY+NRCS_BILLING_INCORRECTLY) AS incorrect_billing,
	SUM(USAGE_BILLING_CORRECTLY+MRCS_BILLING_CORRECTLY+NRCS_BILLING_CORRECTLY) AS correct_billing,
	COUNT(*) AS cnt,
	MIN(component) AS example,
	@ssis_run_id
FROM  aud.component_audit_results
GROUP BY  
	audit_results
	, component_status
	, customer
	, ban
	, account
	, product
	, order_dt
	, issue
	, risk
	, root_cause
	, solution
	, point_of_failure
	, assessment


ALTER INDEX [IX_AUDIT_RESULTS] ON aud.component_audit_summary REBUILD
ALTER INDEX [IX_COMPONENT_STATUS] ON aud.component_audit_summary REBUILD
ALTER INDEX [IX_CUSTOMER] ON aud.component_audit_summary REBUILD
ALTER INDEX [IX_ACCOUNT] ON aud.component_audit_summary REBUILD
ALTER INDEX [IX_ORDER_DT] ON aud.component_audit_summary REBUILD
ALTER INDEX [IX_ISSUE] ON aud.component_audit_summary REBUILD
ALTER INDEX [IX_POINT_OF_FAILURE] ON aud.component_audit_summary REBUILD

insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'write table aud.component_audit_summary', getdate(),@run_id) 
set @logcount = @logcount + 1


------------------------------------------------------------------------------------------------------------------------------------
/*STEP 6 TRUNCATE TABLES*/

TRUNCATE TABLE tmp.component_audit_summary_S1
TRUNCATE TABLE tmp.component_audit_results
TRUNCATE TABLE tmp.component_system2_reject
TRUNCATE TABLE tmp.component_system3_reject
TRUNCATE TABLE tmp.component_audit_results


insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'truncates tables', getdate(),@run_id) 
set @logcount = @logcount + 1


insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
values (@sp_Name, @logcount, 'end', getdate(),@run_id) 
set @logcount = @logcount + 1  



END TRY

BEGIN CATCH  



	insert into ctl.sp_log (sp_name,step,description,complete_date,ssis_run_id) 
	values (@sp_Name, 999, 'On batch line: '+CAST(ERROR_LINE() AS VARCHAR(10))+', error: '+(select ERROR_MESSAGE())+'', getdate(),@run_id);



	declare @step_prior_to_fail bigint
	set     @step_prior_to_fail= (select max(step) from ctl.sp_log x 
							      where x.sp_name = @sp_Name 
							      and complete_date = (select max(complete_date) from ctl.sp_log x 
												   where x.sp_name = @sp_Name AND X.STEP !=999) 
							      and step!=999)

	--select * from ctl.sp_error_log

	insert into ctl.sp_error_log (
		 sp_name
		,step_prior_to_fail
		,ssis_run_id
		,ErrorNumber
		,ErrorSeverity
		,ErrorState
		,ErrorProcedure
		,ErrorLine
		,ErrorMessage
		,complete_date
	   )
	values (
		 @sp_Name
		,@step_prior_to_fail
	    ,@run_id
		,ERROR_NUMBER()
		,ERROR_SEVERITY()
		,ERROR_STATE()
		,ERROR_PROCEDURE()
		,ERROR_LINE()
		,ERROR_MESSAGE()
		,getdate())



	SELECT  
     ERROR_NUMBER() AS ErrorNumber  
    ,ERROR_SEVERITY() AS ErrorSeverity  
    ,ERROR_STATE() AS ErrorState  
    ,ERROR_PROCEDURE() AS ErrorProcedure  
    ,ERROR_LINE() AS ErrorLine  
    ,ERROR_MESSAGE() AS ErrorMessage

	  Declare @ErrorText varchar(512)
       Set @ErrorText = Error_Message() + ' occurred at Line_Number: ' + CAST(ERROR_LINE() AS VARCHAR(50));
       RAISERROR (@ErrorText, 16, 1);

END CATCH;

