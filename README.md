# FU-Contrib
FU contrib (General)
USE [MOS_Reporting]
GO
/****** Object:  StoredProcedure [dbo].[PerfRpt_FUDailyContrib_2016_compundrtn]    Script Date: 2/27/2017 9:22:52 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


-- =======================================================================================================================================
-- Author:		Lau Seo Heng
-- Create date: 14-Dec-16
-- Description:	1. This stored procedure is called from MOS Reports Manager
--              2. The purpose of the stored procedure is to retrieve and compute returns for futures
--              3. The CRNAV is bgn of day NAV 
-- =======================================================================================================================================


ALTER procedure [dbo].[PerfRpt_FUDailyContrib_2016_compundrtn]


	-- Add the parameters for the stored procedure here
	@RevalDateStart		datetime,
	@RevalDateEnd		datetime,
	@Port               varchar(100)

as
begin

	
	DECLARE @RunTypeId			int
	DECLARE @portcode			varchar(100)
	DECLARE @portcode		        varchar(100)
	DECLARE @portcode			varchar(100)
	DECLARE @portcode			varchar(100)
	DECLARE @Portfolio			varchar(100)


	Set @RunTypeId = 1
	Set @portcode_1 = 'portcode_A,portcode_B'
	Set @portcode_2	 = 'portcode_C'
        Set @portcode_3	 = 'portcode_D'
        Set @portcode_4	 = 'portcode_E'

	if OBJECT_ID('tempdb..#TempFURtnsSummary') IS NOT NULL 	DROP TABLE  #TempFURtnsSummary;
	if OBJECT_ID('tempdb..#TempFUContribSummary') IS NOT NULL 	DROP TABLE  #TempFUContribSummary;


	create table #TempFURtnsSummary(
	[reval_date] [datetime],
	[asset_class] [varchar](20),
	[pnl] [decimal](19, 6) NULL,
	[fees] [decimal](19, 6) NULL,
	[nav] [decimal](19, 6) NULL,
	[record_type] [int] NULL
	);

	create table #TempFUContribSummary(
	[date] [datetime],
	[eq_netpnl] [decimal](19, 6) NULL,
	[fi_netpnl] [decimal](19, 6) NULL,
	[cmd_netpnl] [decimal](19, 6) NULL,
	[total_netpnl] [decimal](19, 6) NULL,
	[fees] [decimal](19, 6) NULL,
	[base] [decimal](19, 6) NULL,
	[eqrtn] [decimal](19, 10) NULL,
	[firtn] [decimal](19, 10) NULL,
	[cmdrtn] [decimal](19, 10) NULL,
	[totalrtn] [decimal](19, 10) NULL,
	[record_type] [int] NULL
	);

If  @RevalDateStart > '2016-11-30'
BEGIN


	-- Identify futures portfolio
	if @Port = 'portcode_1'

	begin
		Set @Portfolio = @portcode_1
	end
	
	else if @Port = 'portcode_2'

	begin
		Set @Portfolio = @portcode_2
	end

	else if @Port = 'portcode_3'

	begin
		Set @Portfolio = @portcode_3
	end
	;


	-- Retieve non-UT portfolios 
	insert into #TempFURtnsSummary 
		(reval_date, asset_class, nav, record_type)
	 select dateadd("d",1,reval_date) 'reval_date', portfolio_code, sum(computed_nav_pccy) 'nav', 1 'record_type'
	  from portfolio_nav
	  where run_type_id = 1
	  and portfolio_code in ('cnes', 'tfacsh', 'cnesf')
	  and reval_date between dateadd("d",-1,@RevalDateStart) and dateadd("d",-1,@RevalDateEnd) 
	  group by reval_date, portfolio_code
	;
	

	-- Retieve UT portfolios 
	insert into #TempFURtnsSummary 
		(reval_date, asset_class, nav, record_type)
	 select dateadd("d",1,reval_date) 'reval_date'
		  ,case when security_code in ('sgfaaf') then 'TAG_SGFAAF'			
		   else security_code end 'portfolio_code'
		  ,asset_value_base 'nav', 1 'record_type'
	  from security_position
	  where portfolio_code in ('tag')
	  and run_type_id = 1
	  and reval_date between dateadd("d",-1,@RevalDateStart) and dateadd("d",-1,@RevalDateEnd) 
	  and security_code not in ('sgfaag')
	;
	
	

	
	-- Insert Futures PnL
	insert into #TempFURtnsSummary
		(reval_date, asset_class, pnl, fees, record_type)
	select contract_date, asset_class, sum(pnl), sum(fees), 1 'record_type'
	from (
		select	contract_date, left(security_code,1) + 'FU' 'asset_class' 
				,-isnull(total_cost_proceed_pccy, 0) 'pnl'
				,-isnull(clearing , 0)*dbo.GetCrossFxSpot(currency_1,'sgd',contract_date) 'fees'
		from	variation_margin_generation
		where	portfolio_code in (select distinct strvalue from stringtotable(',', @Portfolio))
		and		category = 'fu'
		and		contract_date between @RevalDateStart and @RevalDateEnd
		
		union all

		select	contract_date, left(security_code,1) + 'FU' 'asset_class'
				,-isnull(total_cost_proceed_pccy, 0) 'pnl'
				,-isnull(clearing , 0)*dbo.GetCrossFxSpot(currency_1,'sgd',contract_date) 'fees'
		from	transaction_master
		where	portfolio_code in (select distinct strvalue from stringtotable(',', @Portfolio))
		and		contract_date between @RevalDateStart and @RevalDateEnd
		and		category = 'fu'

	) combined 
	group by contract_date, asset_class
	;


	insert into #TempFUContribSummary 
		(date, total_netpnl, fees, base, record_type)
	select		reval_date , sum(pnl), sum(fees), sum(nav), 1
	from		#TempFURtnsSummary
	group by	Reval_Date
	order by	Reval_Date	
	;


	update		#TempFUContribSummary
	set			eq_netpnl = a1.pnl,
				eqrtn = a1.pnl/base
	from		#TempFURtnsSummary a1
	where		date = a1.reval_date
	and			a1.asset_class = 'efu'
	;


	update		#TempFUContribSummary
	set			fi_netpnl = a1.pnl,
				firtn = a1.pnl/base
	from		#TempFURtnsSummary a1
	where		date = a1.reval_date
	and			a1.asset_class = 'bfu'
	;


	update		#TempFUContribSummary
	set			cmd_netpnl = a1.pnl,
				cmdrtn = a1.pnl/base
	from		#TempFURtnsSummary a1
	where		date = a1.reval_date
	and			a1.asset_class = 'cfu'
	;


	update		#TempFUContribSummary
	set			totalrtn = total_netpnl/base
	;


	insert into #TempFUContribSummary 
		(date, eq_netpnl, fi_netpnl, cmd_netpnl, total_netpnl, fees, base, eqrtn, firtn, cmdrtn, totalrtn, record_type)
	select		@RevalDateEnd, sum(eq_netpnl) 'eq_netpnl', sum(fi_netpnl) 'fi_netpnl', sum(cmd_netpnl) 'cmd_netpnl', 
				sum(total_netpnl) 'total_netpnl', sum(fees) 'fees', sum(base)/(datediff("d", @RevalDateStart, @RevalDateEnd) + 1) 'base'
				--, sum(eqrtn) 'eqrtn'
				--, (power(10.00000000000,(sum(case when eqrtn <> 0 then log10((eqrtn/10000)+1) else 0 end)))-1)*10000 'eqrtn'
				, (power(10.00000000000,(sum(case when eqrtn <> 0 then log10(eqrtn+1) else 0 end)))-1) 'eqrtn'
				--, sum(firtn) 'firtn'
				, (power(10.00000000000,(sum(case when firtn <> 0 then log10(firtn+1) else 0 end)))-1) 'firtn'
				--, sum(cmdrtn) 'cmdrtn'
				, (power(10.00000000000,(sum(case when cmdrtn <> 0 then log10(cmdrtn+1) else 0 end)))-1) 'cmdrtn'
				--, sum(totalrtn) 'totalrtn'
				, (power(10.00000000000,(sum(case when totalrtn <> 0 then log10(totalrtn+1) else 0 end)))-1) 'totalrtn'
				, 2 record_type
	from		#TempFUContribSummary
	;
		

	select	date, isnull(eq_netpnl,0) 'eq_netpnl', isnull(fi_netpnl,0) 'fi_netpnl', isnull(cmd_netpnl,0) 'cmd_netpnl'
			,isnull(total_netpnl,0) 'total_netpnl', isnull(fees,0) 'fees', isnull(base,0) 'base', isnull(eqrtn,0) 'eqrtn'
			,isnull(firtn,0) 'firtn', isnull(cmdrtn,0) 'cmdrtn', isnull(totalrtn,0) 'totalrtn', record_type
	from #TempFUContribSummary
	order by date, record_type


end
end
