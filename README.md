USE [Front Runner DACH]
GO
/****** Object:  StoredProcedure [dbo].[xx_spPickingSlipEmailNotification_DACH]    Script Date: 2018-01-14 03:02:43 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


/*
14.01.2018		:	xx_spPickingSlipEmailNotification_DACH_14012018
EXEC [dbo].[xx_spPickingSlipEmailNotification_DACH]

*/


ALTER Procedure  [dbo].[xx_spPickingSlipEmailNotification_DACH]
AS

/**********************************************************************
Date	:	11.08.2016
Author	:	Werner Husi
Outline	:	Email Picking Slip for every Sales Order created
			at the time the Sales Order gets created.
			Kits must show actual components to nth level
			
xx_PickingSlipEmailNotification_11082016_V11
		:	Optionl Extras in future: 1) Signature Option
			2) Date Display Option, 3) Total Number of Items
			to count, 4) Frontrunner Logo	
		:	Added Multiple email addresses
			Added Address details
			Added dynamic dbName
			Created stored Procedure xx_spPickingSlips in USA, EU, AUS	

xx_PickingSlipEmailNotification_V12_17082016
		:	Added Logo

xx_PickingSlipEmailNotification_V13_17082016
		:	Added picker name and picker signature

xx_PickingSlipEmailNotification_V14_18082016
		:	Added picker date
		:	Added customer note

xx_PickingSlipEmailNotification_V15_23082016		
		:	Changed subject date from GETDATE() to OrderDate (WH)
		
xx_PickingSlipEmailNotification_V14_31082016		
		:	Added email address werner@blueglobe27.com (WH)	

xx_PickingSlipEmailNotification_V14_06092016
		:	Removed the company address on the notification email (BM)	

NOTE	:	Only commissioned in USA as other countries do not have 
			shipping address as yet
			
xx_PickingSlipEmailNotification_V17_USA_06092016 (WH)
		:	Added customer address 	
		:	Changed delivery address to INVNUM delivery address	

EXEC xx_spPickingSlipEmailNotification_V17_USA

EXEC xx_spPickingSlipEmailNotification_V19_USA_24112016
		:	Added Checker Name x 2 as per email Mark
		:	Changed shipping address as per MArk verbal at site

EXEC xx_spPickingSlipEmailNotification_V20All
		:	Using different function which is sequenced by Sales Order 
			Items and Components within Bil
			Added Description to items/Kits
		
EXEC xx_spPickingSlipEmailNotification_V21All		
		:	changed item description to be in function
		:	changed item type to be in function
		
EXEC xx_spPickingSlipEmailNotification_V21All_30112016
		:	Commissioned to live in USA today
		:	Commissioned to live in EU today		

EXEC xx_spPickingSlipEmailNotification_V22All_01122016
		:	Added Kit Quantity
EXEC xx_spPickingSlipEmailNotification_V23All_09122016
		:	Split Delivery Address from Customer Details as per request 
			Rudi email 8th December 2016
EXEC xx_spPickingSlipEmailNotification_V24All_13042017
		:	Dropped last column 
		:	Widened and renamed 'Kit(s) Picked' and 'Item(s) Picked
		:	Modified Email update flag by adding 4 as a status
		:	Amended subject line to accomodate new procedure
EXEC xx_spPickingSlipEmailNotification_EU_04052017
		:	Added Left(2)='12' as per email 
EXEC xx_spPickingSlipEmailNotification_DACH_02062017
		:	Added Bin Location
		:	Amended to email on store '13' only
EXEC xx_spPickingSlipEmailNotification_DACH_22062017
		:	Changed Bin Location from Kit to component Bin Location

14.01.2018	:	xx_spPickingSlipEmailNotification_DACH_14012018
				Add document starting with 8 & 9 as per email Keith Fri 12.01.2018
				xx_spPickingSlipEmailNotification_DACH_15012018
just because i can
***********************************************************************/


DECLARE @SONumber	Varchar(MAX) =(select  TOP(1) ordernum 
				-- select distinct ordernum
				from InvNum i
				INNER JOIN _btblInvoiceLines il on il.iInvoiceID=i.AutoIndex
				where 
					--	until 14.01.2018
					--	((DocType		=	4 and DocState=1) or (DocType =	4 and DocState=4 and LEFT(OrderNum,2) in('13')))
					--	as from 15.01.2018 onwards
					((DocType		=	4 and DocState=1) or (DocType =	4 and DocState=4 and LEFT(OrderNum,2) in('13','80','90')))
					and OrderNum not in (Select OrderNum from xx_PickingSlips)
					and CONVERT(Date,GETDATE())>'2016-08-10'
					and (select count(iStockCodeID) from _btblInvoiceLines il
								Inner join stkitem s on s.stocklink=il.istockcodeid
								where ServiceItem=0)>0
								and AccountID > 0
								and CONVERT(Date,InvDate)>'2017-04-10')

IF @SONumber is NOT null or @SONumber<>''

BEGIN

	Declare 
	-- Email Variables
	@EmailSubject			VARCHAR(MAX),
	@EmailBody				VARCHAR(MAX),

	-- Common Variables
	@BodyHeader				varchar(MAX),
	@BodyDetails			varchar(MAX),
	@BodyTail				varchar(MAX),
	@DBName					Varchar(MAX),
	@AddressDetails			VarChar(MAX),
	@ToEmail				Varchar(Max),
	@CopyEmail				Varchar(Max),
	@Message1				Varchar(MAx),
	@PickerName				VarChar(MAX)	=	'Picker Name &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Signature ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Date ........................' + '<br/>',
	@CheckerName1			VarChar(MAX)	=	'Checked by Name ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Signature ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Date ........................' + '<br/>',
	@CheckerName2			VarChar(MAX)	=	'Checked by Name ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Signature ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Date ........................' + '<br/>',
	
	@CustDCLink				Int
	
	SET @DBName			=	(SELECT DB_NAME())
	SET @AddressDetails	=	(select PhAddress1 + ', '+ PhAddress2 +', '+PhAddress3 +', Telephone:'+ Telephone1 from Entities)
	
	SET @ToEmail = 'ShippingDACH@frontrunneroutfitters.eu'
	--SET @CopyEmail = CASE 
	--	WHEN DB_NAME()='Frontrunner' THEN 'shipping@frontrunner.co.za'
	--	WHEN DB_NAME()='Frontrunner EU' THEN 'shipping@frontrunneroutfitters.eu'
	--	WHEN DB_NAME()='Front Runner DACH' THEN 'ShippingDACH@frontrunneroutfitters.eu'
	--	WHEN DB_NAME()='Frontrunner AUS' THEN 'werner@blueglobe27.com'
	--	WHEN DB_NAME()='Frontrunner USA' THEN 'Shipping-USA@frontrunneroutfitters.com'
	----	WHEN DB_NAME()='Frontrunner USA' THEN 'boipelo@blueglobe27.com'

		
							
	SET @Message1=	(Select Message1 from InvNum
						WHERE OrderNum=@SONumber and DocType=4)
						
	SET @CustDCLink= (Select top(1) AccountID from InvNum where OrderNum=@SONumber)
	
	DECLARE @CustAccount			Varchar(MAX)	=	(Select ISNULL(Account,'N/A') from Client where DCLink=@CustDCLink)	
	DECLARE @CustName				Varchar(MAX)	=	(Select ISNULL(Name,'N/A') from Client where DCLink=@CustDCLink)
	DECLARE @Phys1					Varchar(MAX)	=	(Select ISNULL(Address1,'N/A') from InvNum where OrderNum=@SONumber and DocType=4)
	DECLARE @Phys2					Varchar(MAX)	=	(Select ISNULL(Address2,'') from InvNum where OrderNum=@SONumber and DocType=4)
	DECLARE @Phys3					Varchar(MAX)	=	(Select ISNULL(Address3,'') from InvNum where OrderNum=@SONumber and DocType=4)
	DECLARE @Phys4					Varchar(MAX)	=	(Select ISNULL(Address4,'') from InvNum where OrderNum=@SONumber and DocType=4)
	DECLARE @Phys5					Varchar(MAX)	=	(Select ISNULL(Address5,'') from InvNum where OrderNum=@SONumber and DocType=4)
	DECLARE @PhysicalPC				Varchar(MAX)	=	(Select ISNULL(Address6,'') from InvNum where OrderNum=@SONumber and DocType=4)
	DECLARE @CustDetails			Varchar(MAX)	=	
				(Select 'Customer: ' 
					+@CustAccount+', '
					+@CustName)
	DECLARE @DeliveryDetails		VARCHAR(MAX)	=	
					(Select 'Deliver to: ' 
					+@Phys1+', '
					+@Phys2+', '
					+@Phys3+', '
					+@Phys4+', '
					+@Phys5+', '
					+@PhysicalPC)
						
	--print @SONumber				
	    
	-- Set  Header Line 		
	SET @EmailSubject =(
		select c.Account + ' - ' + Name + ' (' + OrderNum + '/'+ convert(Varchar(20),OrderDate,105) +') ' 	
		-- select c.account,i.ordernum,GETDATE() As emailed into xx_PickingSlips
		FROM [dbo].[InvNum] i
		INNER JOIN [dbo].[Client] c
				On c.DCLink=i.AccountID	
		where 1=1
		--and DocType	=	4
		--and DocState	=	1
		and OrderNum	=	@SONumber and DocType=4
	--	and @SONumber not IN(Select OrderNum from xx_PickingSlips where emailed is null)
		)
		
	-- set global table tail
	Set @BodyTail = '</table></body></html>'


	-- set global table head
	 SET @BodyHeader = '<html><head>' + '<style>'
		+ 'td {border: solid black 1px;padding-left:1px;padding-right:1px;padding-top:1px;padding-bottom:1px;font-size:10pt;} '
		+ '</style>' + '</head>'
		+ '<body><table cellpadding=0 cellspacing=0 border=0>'
		+ '<td align=center bgcolor=#CCCCCC><b>ItemInSO</b></td>'
		+ '<td align=center bgcolor=#CCCCCC><b>Level1</b></td>'
		+ '<td align=center bgcolor=#CCCCCC><b>Level2</b></td>' 
		+ '<td align=center bgcolor=#CCCCCC><b>Level3</b></td>' 
		+ '<td align=center bgcolor=#CCCCCC><b>Component</b></td> ' 
		+ '<td align=center bgcolor=#CCCCCC><b>Description</b></td> ' 
		+ '<td align=center bgcolor=#CCCCCC><b>Bin</b></td> ' 
		+ '<td align=center bgcolor=#CCCCCC><b>Kit(s) Picked</b></td> ' 
		+ '<td align=center bgcolor=#CCCCCC><b>Item(s) Picked</b></td> ' 
	--	+ '<td align=center bgcolor=#CCCCCC><b>Picked</b></td> ' 		 	
		 
--	SET @BodyDetails=CAST((SELECT
--		CASE WHEN f.BomItem IS NULL THEN s.code ELSE f.BomItem END AS 'td',''
--		,ISNULL(f.BomProductionQty,'') AS 'td',''
--		,CASE WHEN f.ItemCode IS NULL THEN '' ELSE f.ItemCode END AS 'td',''
--		,ISNULL(f.itemDescription,s.description_1) AS 'td',''
--		,CASE WHEN f.BomItem IS NULL THEN 'Stock Item' ELSE 'Kit Item' END AS 'td',''
--		,CONVERT(DECIMAL(10,0),ISNULL(f.extendedQty,1) * il.fquantity) AS 'td',''
--		,'' AS 'td',''
--		-- select f.*
--	from InvNum i
--	INNER JOIN _btblinvoicelines il on il.iinvoiceid=i.AutoIndex
--	INNER JOIN Stkitem s on s.StockLink=il.iStockCodeID
--	OUTER APPLY xx_ufnQtyCompInKits_V13 (s.Code) f
----	where i.OrderNum='500002869'
--	where i.OrderNum=@SONumber
--	and s.ServiceItem=0
--	and DocState=1
--	FOR XML PATH('tr'), ELEMENTS ) AS NVARCHAR(MAX))   
		
	SET @BodyDetails=CAST((select
		CASE 
			WHEN f2.Level IS NULL THEN s.code
			WHEN f2.Level=0 THEN f2.BomstockCode ELSE '' END AS 'td',''
		,CASE WHEN f2.Level=1 THEN f2.BomstockCode ELSE '' END AS 'td','' 
		,CASE WHEN f2.Level=2 THEN f2.BomstockCode ELSE '' END AS 'td','' 
		,CASE WHEN f2.Level=3 THEN f2.BomstockCode ELSE '' END AS 'td',''
		,ISNULL(f2.code,s.Code)  AS 'td',''
		,ISNULL(s1.Description_1,s.Description_1)  AS 'td',''
		,ISNULL(bl.cBinLocationName,'')  AS 'td',''
		,CASE WHEN ComponentIndex=1 THEN CONVERT(VARCHAR(5),f2.BomProductionQty*il.fQuantity) ELSE '' ENd AS 'td',''
		,CONVERT(VARCHAR(20),ISNULL(f2.ExtendedQty,1)*il.fQuantity) AS 'td',''
	--	,'' AS 'td',''
		-- select *
		FROM [dbo].[InvNum] i
				inner join _btblInvoiceLines il on i.AutoIndex=il.iInvoiceID
				inner join stkitem s on s.Stocklink=il.iStockCodeID	
				outer apply [Front Runner DACH].[dbo].[xx_ufnQtyCompInKits_21012017V1] (s.code) f2
				LEFT JOIN stkitem s1 on s1.Code=f2.code
				LEFT JOIN _btblBINLocation bl on bl.idBinLocation=ISNULL(s1.iBinLocationID,s.iBinLocationID)
			where 1=1
			and	((DocType = 4 and DocState = 1) or (DocType = 4 and DocState = 4 and LEFT(OrderNum,2) in('13','80','90')))
			and s.ServiceItem	=	0
			and OrderNum		=	@SONumber
			
		order by il.idInvoiceLines, f2.Level,f2.BomstockCode, ComponentIndex
		FOR XML PATH('tr'), ELEMENTS ) AS NVARCHAR(MAX))


	/**************** Blueglobe 27 ******************/
	
	SET @EmailSubject	=	CASE WHEN @EmailSubject ='' THEN 'TEST' ELSE @EmailSubject END
--	SET @EmailBody		=	 '<h3>' + @DBName + ' - PICKING SLIP</h3>' + @AddressDetails + '<br/>' +@BodyHeader + @BodyDetails + @BodyTail + '<br/>'
--	SET @EmailBody		=	 '<h3>' + @DBName + ' - PICKING SLIP</h3>' + '<img alt="Front Runner Outfitters South Africa" src="https://www.frontrunneroutfitters.com/multi/skin/frontend/idstore/frontrunner/images/logo.png?v=2016"'+ '<br/>' +'<br/>'+ @AddressDetails + '<br/>' +@BodyHeader + @BodyDetails + @BodyTail + '<br/>' 
--	SET @EmailBody		=	 '<h3>' + @DBName + ' - PICKING SLIP</h3>' + '<img alt="Front Runner Outfitters South Africa" src="https://www.frontrunneroutfitters.com/multi/skin/frontend/idstore/frontrunner/images/logo.png?v=2016"'+ '<br/>' +'<br/>'+'Customer Note: '+ @Message1 + '<br/>' +@BodyHeader + @BodyDetails + @BodyTail + '<br/>' + '<br/>' + '<br/>' + 'Picker Name ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Signature ........................&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;' + 'Date ........................' + '<br/>'
	SET @EmailBody		=	 '<h3>' + @DBName + ' - PICKING SLIP</h3>' + '<img alt="Front Runner Outfitters South Africa" src="https://www.frontrunneroutfitters.com/multi/skin/frontend/idstore/frontrunner/images/logo.png?v=2016"'+ '<br/><br/>' 
			+	@Custdetails + '<br/>'
			+	@DeliveryDetails + '<br/><br/>'
			+	'Customer Note: '
			+	@Message1 + '<br/>' 
			+	@BodyHeader 
			+	@BodyDetails 
			+	@BodyTail + '<br/>' + '<br/>' + '<br/>' 
			+	@PickerName + '<br/><br/>'
			+	@CheckerName1 + '<br/><br/>'
			+	@CheckerName2 + '<br/>'

	SET @EmailBody = msdb.dbo._ufnSpecChar2html(@EmailBody)
			
--print @ToEmail
--print @EmailSubject
--print @EmailBody
	-- Send mail 
	 EXEC msdb.dbo.sp_send_dbmail 
		@recipients = @ToEmail,
		--'printer2.boksberg@hpeprint.com',
		@Copy_recipients = 'johna@frontrunner.co.za',
	--	@recipients='werner@blueglobe27.com',
	--	@blind_copy_Recipients='werner@blueglobe27.com',
		@subject = @EmailSubject, 
		@profile_name = 'Evolution', 
		@body = @EmailBody,
		@body_format = 'HTML' ;
	
	/********** End ***********************************/


	 --Update table and email flag
	INSERT INTO xx_PickingSlips (Account,OrderNum,Emailed)
	SELECT c.Account,i.OrderNum,GETDATE()
	FROM [dbo].[InvNum] i
	INNER JOIN [dbo].[Client] c
				On c.DCLink=i.AccountID	
	where 1=1
	and DocType		=	4
	and DocState	in	(1,4)
	and OrderNum	=	@SONumber
	and i.OrderNum not IN(Select OrderNum from xx_PickingSlips)

END
	










