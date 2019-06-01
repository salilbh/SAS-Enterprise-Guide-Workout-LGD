# SAS-Enterprise-Guide-Workout-LGD
Code used for development of workout LGD for BFSI (Raw Code)


/*---------------Workout LGD Calculation based on the account transaction data --------------------*/


/* Bad Debt Write off Data*/

Data BADDEBT_WRITEOFF (Rename= CR_AMT=CREDIT);
Set ECLMJ.BADDEBT_WRITEOFF (Rename= DR_AMT=WO_DR_AMT);
FORACID = CR_FORACID;
BD_Writeoff_AMT = CR_AMT;
Run;

%Sort_Data (Data_Name=BADDEBT_WRITEOFF, by=FORACID TRAN_DATE CREDIT, Output=BADDEBT_WRITEOFF);

Data BADDEBT_WRITEOFF (Keep=FORACID TRAN_DATE CREDIT BD_Writeoff_AMT);
set BADDEBT_WRITEOFF;
if BD_Writeoff_AMT = . then delete;
Run;


%Sort_Data_NDK (Data_Name=BADDEBT_WRITEOFF, by=FORACID TRAN_DATE CREDIT BD_Writeoff_AMT, Output=BADDEBT_WRITEOFF);


/* MOI Write off Data*/

Data MOI_Writeoff (Rename= CR_AMT=CREDIT);
Set ECLMJ.MOI_Writeoff (Rename= (DR_AMT=WO_DR_AMT TRAN_DATE=TRAN_DATET));
FOrmat TRAN_DATE Date9.;
FORACID = CR_FORACID;
TRAN_DATE = Datepart(TRAN_DATET);
MOI_Writeoff_Amt = CR_AMT;
if substr(Tran_Particular,1, 7) = "INT.FOR" then Delete;
if findw(Tran_Particular,'DD')>0 and findw(Tran_Particular,'MOI')=0 then delete;
Run;

/*if findw(Tran_Particular,'INT')>0 and findw(Tran_Particular,'FOR')>0 and findw(Tran_Particular,'MOI')=0 then Delete;
if findw(Tran_Particular,'DD')>0 and findw(Tran_Particular,'MOI')=0 then delete;*/

%Sort_Data (Data_Name=MOI_Writeoff, by=FORACID TRAN_DATE CREDIT, Output=MOI_Writeoff);

Data MOI_Writeoff (Keep=FORACID TRAN_DATE CREDIT MOI_Writeoff_AMT);
set MOI_Writeoff;
if MOI_Writeoff_AMT = . then delete;
Run;

%Sort_Data_NDK (Data_Name=MOI_Writeoff, by=FORACID TRAN_DATE CREDIT MOI_Writeoff_AMT, Output=MOI_Writeoff);




/* Transaction Statements*/

Data Closed_NPA (drop= NPA_Class_CloseDate);
Set ECLMJ.Closed_NPA;
Run;

%Sort_Data_NDK (Data_Name=Closed_NPA, by=FORACID, Output=Closed_NPA);


%Sort_Data_NDK (Data_Name=ECLMJ.NPA_BALANCE_ROI, by=FORACID, Output=NPA_BALANCE_ROI);
DATA NPA_BALANCE_ROI;
SET NPA_BALANCE_ROI;
RUN;


Data LGD_ACSTATEMENTS;
set ECLMJ.LGD_ACSTATEMENTS;
if FORACID = . then Delete;
Run;

%Sort_Data (Data_Name=LGD_ACSTATEMENTS, by=FORACID, Output=LGD_ACSTATEMENTS);


Data LGD_ACSTATEMENTS;
Merge LGD_ACSTATEMENTS Closed_NPA;
by FORACID;
Run;

Data LGD_ACSTATEMENTS;
set LGD_ACSTATEMENTS;
if Accountid = "" then delete;
Run;

Data LGD_ACSTATEMENTS;
Merge LGD_ACSTATEMENTS NPA_BALANCE_ROI;
by FORACID;
Run;

Data LGD_ACSTATEMENTS;
set LGD_ACSTATEMENTS;
if Accountid = "" then delete;
Run;


/*Merging Transaction data with Bad Debts write off and MOI Write off*/

%Sort_Data (Data_Name=LGD_ACSTATEMENTS, by=FORACID TRAN_DATE CREDIT, Output=LGD_ACSTATEMENTS);

Data LGD_ACSTATEMENTS_Final;
Merge LGD_ACSTATEMENTS BADDEBT_WRITEOFF MOI_Writeoff;
by FORACID TRAN_DATE CREDIT;
Run;

Data LGD_ACSTATEMENTS_Final;
set LGD_ACSTATEMENTS_Final;
if Accountid = "" then delete;
Run;


/*Calculation of LGD*/

Data LGD_ACSTATEMENTS_Final;
Set LGD_ACSTATEMENTS_Final;
if Accountid ne "" then t = (datdif(LATEST_NPA_DATE ,TRAN_DATE, 'act/act'));
if Accountid ne "" then ty = (yrdif(LATEST_NPA_DATE ,TRAN_DATE, 'ACTUAL'));
Run;


Data LGD_ACSTATEMENTS_Final;
Set LGD_ACSTATEMENTS_Final;
if ty =>0 then Discounted_Credits = (CREDIT/(1+(ROI_ON_NPA_DATE/100))**(ty));
if ty =>0 then Discounted_Debits = (DEBIT/(1+(ROI_ON_NPA_DATE/100))**(ty));
if ty =>0 and (BD_Writeoff_AMT > 0 or MOI_Writeoff_AMT > 0) then Dis_Writeoff = (CREDIT/(1+(ROI_ON_NPA_DATE/100))**(ty));
if TRAN_RMKS = "ACXFRSOL" then Legal_Trf = (CREDIT/(1+(ROI_ON_NPA_DATE/100))**(ty));
if SCAN (TRAN_PARTICULAR,-1,' ') = "43540" then DECREED = (CREDIT/(1+(ROI_ON_NPA_DATE/100))**(ty));
if SCAN (TRAN_PARTICULAR,-1,' ') = "43505" then Suit = (CREDIT/(1+(ROI_ON_NPA_DATE/100))**(ty));
if SCAN (TRAN_PARTICULAR,-1,' ') = "43511" then CLAIMS = (CREDIT/(1+(ROI_ON_NPA_DATE/100))**(ty));
Run;

Data LGD_ACSTATEMENTS_Final;
Set LGD_ACSTATEMENTS_Final;
if Discounted_Credits = . then Discounted_Credits = 0;
if Discounted_Debits = . then Discounted_Debits = 0;
if Dis_Writeoff = . then Dis_Writeoff = 0;
if Legal_Trf = . then Legal_Trf = 0;
if DECREED = . then DECREED = 0;
if Suit = . then Suit = 0;
if CLAIMS = . then CLAIMS = 0;
Run;


/*-----------Intermediate Data Save---------------*/
Data ECLMJ.LGD_ACSTATEMENTS_Final;
set LGD_ACSTATEMENTS_Final;
Run;
/*--END------Intermediate Data Save---------------*/


Proc sql;
Create table LGD_Calculation as
Select FORACID, sum(Discounted_Credits) as Dis_Credits, sum(Discounted_Debits) as Dis_Debits, sum(Dis_Writeoff) as Dis_Writeoff, sum(Legal_Trf) as Legal_Trf, sum(DECREED) as DECREED, sum(Suit) as Suit, sum(CLAIMS) as CLAIMS from LGD_ACSTATEMENTS_Final
Group by FORACID;
Quit;

Data LGD_Calculation (drop=ROI_ON_NPA_DATE);
Merge LGD_Calculation NPA_BALANCE_ROI;
by FORACID;
if BALANCE_NPADATE <0 then delete;
RUN;


Data LGD_Calculation;
Set LGD_Calculation;
Recovery = sum(Dis_Credits, -Dis_Debits, -Dis_Writeoff, -Legal_Trf, -DECREED, -Suit, -CLAIMS);
Run;

Data LGD_Calculation;
Set LGD_Calculation;
if BALANCE_NPADATE = 0 then Delete;
Run;

Data LGD_Calculation;
Set LGD_Calculation;
if FORACID ne . then R = Recovery/BALANCE_NPADATE;
Run;

Data LGD_Calculation;
Set LGD_Calculation;
if FORACID ne . then LGD = 1-R;
Run;

Data LGD_Calculation;
Set LGD_Calculation;
where LGD >= 0 and LGD <= 1;
Run;

Data LGD_Calculation;
Format Statement_Date Date9.;
Set LGD_Calculation;
Statement_Date = '31Sep2017'd;
Run;

/*-----------Intermediate Data Save---------------*/
Data ECLMJ.LGD_Calculation;
set LGD_Calculation;
Run;
/*--END------Intermediate Data Save---------------*/

%Export_Data (Data=LGD_Calculation, Outfile="D:\ECL_Sep17\LGD_Calculation.xlsx");

Proc sql;
Create Table LGD as 
select Statement_Date, Count(FORACID) as Count_Accounts, Avg(R) as Average_Recovery, Avg(LGD) as Average_LGD from LGD_Calculation
Group by Statement_Date;
Quit;

/* Refining accounts considered for lGD and Deriving finalised LGD*/


Data Closed_NPA (drop= LATEST_NPA_DATE);
Set ECLMJ.Closed_NPA;
Run;

%Sort_Data_NDK (Data_Name=Closed_NPA, by=FORACID, Output=Closed_NPA);
%Sort_Data (Data_Name=ECLMJ.LGD_Calculation, by=FORACID, Output=LGD_Calculation_Final);


Data LGD_Calculation_Final;
Merge LGD_Calculation_Final Closed_NPA;
by FORACID;
Run;

Data LGD_Calculation_Final;
Set LGD_Calculation_Final;
if Statement_Date = . then delete;
Run;

Data LGD_Calculation_Final;
Set LGD_Calculation_Final;
AccountID = Put(FORACID,$15.);
SCHM = substr(AccountID,6,3);
Run;

Data LGD_Calculation_Final;
Set LGD_Calculation_Final;
If SCHM="101" then Delete;/*SAVINGS BANK ACCOUNT*/
If SCHM="716" then Delete;/*V NIVESH*/
If SCHM="714" then Delete;/*V PENSION*/
If SCHM="999" then Delete;/*STAFF FESTIVAL ADVANCE*/
If SCHM="951" then Delete;/*SUIT FILED ACCOUNTS*/
If SCHM="952" then Delete;/*DECREED ACCOUNTS*/
If SCHM="030" then Delete;/*V PLATINUM CURRENT ACCOUNT*/
If SCHM="893" then Delete;/*STAFF HOUSING LOAN*/
If SCHM="953" then Delete;/*CLAIMS PREFERRED UNDER PUB ACT*/
If SCHM="110" then Delete;/*NRO SAVINGS BANK ACCOUNT*/
If SCHM="892" then Delete;/*STAFF SECURED LOAN*/
If SCHM="736" then Delete;/*ADW&DR SCHEME -2008 (OTS)*/
If SCHM="103" then Delete;/*INOPERATIVE SAVINGS BANK A/C*/
If SCHM="810" then Delete;/*STAFF OVER DRAFT*/
If SCHM="609" then Delete;/*CLAIMS PAID ON DEFAULTED GURNT*/
If SCHM="114" then Delete;/*SAVINGS BANK ACCOUNT*/
If SCHM="891" then Delete;/*STAFF LOAN AGAINST MOTOR VEHIC*/
If SCHM="123" then Delete;/*VIJAYA SARAL SAVINGS A/C*/
If SCHM="155" then Delete;/*NRE SAVINGS BANKS ACCOUNT*/
If SCHM="118" then Delete;/*INOPERATIVE SAVINGS BANK A/C*/
If SCHM="897" then Delete;/*STAFF MORTGAGE LOAN*/
If SCHM="111" then Delete;/*INOPERATIVE SAVINGS BANK A/C*/
If SCHM="815" then Delete;/*STAFF PRONOTE LOAN*/
If SCHM="816" then Delete;/*STAFF PRONOTE LOAN*/
If SCHM="104" then Delete;/*IN-OPER NRO SAVINGS A/C*/
If SCHM="826" then Delete;/*CALAMITY RELIEF LOAN TO STAFF*/
If SCHM="890" then Delete;/*STAFF SECURED LOAN*/
If SCHM="145" then Delete;/*INOPERATIVE SAVINGS BANK A/C*/
If SCHM="842" then Delete;/*STAFF LOAN AGAINST MOTOR VEHIC*/
if NPA_Class_CloseDate = 1 then Delete;
Run;


Data LGD_Calculation_Final;
Set LGD_Calculation_Final;
if Dis_Writeoff=0 and Legal_Trf=0 and DECREED=0 and Suit=0 and CLAIMS=0 then Delete;
Run;

Proc sql;
Create Table ECLMJ.LGD_Rate as 
select Statement_Date, Avg(LGD) as Average_LGD from LGD_Calculation_Final
Group by Statement_Date;
Quit;
Data ECLMJ.LGD_Rate;
Set ECLMJ.LGD_Rate (Rename= (Statement_date=StatementDate Average_LGD=LGD));
Run;


