/************************************************************************************************************************\ 
Part 1: Web Scraping with MACROS

	MACROS INCLUDED 

	IN ORDER OF APPEARANCE
	NOT IN ORDER OF CALL

	%GET_HTML     (GLOBAL);
	%FIND_URLS    
	%ADD_POLL_URL 
	%ADD_DNAMES
	%VERIFY_POLL
	%REMOVE_POLLS
	%REMOVE_THREE_RACE
	%FIND_RACE_RESULT
	%FIND_POLL_RESULT      <---!!IMPORTANT!!---> YOU MUST CHANGE ROOT TO POINT TO WHERE RCP.MAP IS LOCATED!
	%SCANLOOP_IDENTIFY                           C:\Users\johnlo\Desktop\RCP.map
	%SCANLOOP_RACE_RESULT
	%GET_NAMES
	%INSERT_NAMES
	%SCANLOOP_FIND_NAMES
	%INSERT_PARTIES

NOTE: SCANLOOP is set to denote a certain do process that accesses the database (RACES_URLS) and iterates over it
NOTE: Many of these MACROS can either be simplified or better constructed for more global use

\************************************************************************************************************************/;

/* We will treat RACE_URLS as our consistently updated directory log aka database*/
/* thus it is important to format RACE_URLS beforehand */

/******************************************************************************************\ 
Given a URL returns the html contents of a webpage in a dataset 
\*******************************************************************************************/

%MACRO GET_HTML(url); 
	FILENAME HTML URL &url DEBUG LRECL=10000; 
	QUIT;
	DATA HTML; 
	id=1; 
	INFILE HTML length=len; 
	INPUT record $varying10000. len; 
	RUN; 
	QUIT;

%MEND GET_HTML; 
%GET_HTML("http://www.realclearpolitics.com/epolls/2010/governor/2010_elections_governor_map.html");

/******************************************************************************************\ 
Given a URL returns the html contents of a webpage in a dataset 
\*******************************************************************************************/

%MACRO FIND_URLS(DATASET);

	PROC sql;
	   create table RACE_URLS as
	   select * from &DATASET 
	   where prxmatch('(<option?)', record);
	RUN;
	QUIT;

	DATA RACE_URLS;
	SET RACE_URLS;
	filter = tranwrd(record, "<option value=","");
	filter = tranwrd(filter, "</option>","");
	/*filter = tranwrd(filter, ">","");*/
	/*Remove trailing blanks */
	filter = strip(filter);
	RUN;
	QUIT;

	DATA RACE_URLS (DROP = record id);
	SET RACE_URLS;
	RUN;
	QUIT;

	DATA RACE_URLS;
	SET RACE_URLS;
	patternID = prxparse('/>/');
	position =  prxmatch(patternID, filter);
	length = length(filter);
	race_url = substr(filter, 1, position-1);
	
	/* Remove heading Options that select Senate, Governor, Congress since those are not actual race urls (which are longer than 20) */
	IF length LT 20 THEN DELETE;
	RUN;

	/* Clean up */
	DATA RACE_URLS(DROP = filter patternID position length);
	SET RACE_URLS;
	RUN;
	QUIT;
%MEND FIND_URLS;

%FIND_URLS(HTML);

/******************************************************************************************\ 
Add the found poll url to the database
\*******************************************************************************************/

%MACRO ADD_POLL_URL();
	DATA RACE_URLS;
	SET RACE_URLS;
	filter = tranwrd(race_url, """","");
	patternID = prxparse('/-/');
	position =  prxmatch(patternID, filter);
	length = length(filter);
	id = substr(filter, position+1, length-position-5);
	poll_url = "http://cdn.realclearpolitics.com/epolls/charts/"||put(id,4.)||".xml";
	RUN;

	DATA RACE_URLS (DROP=race_url patternID position length id);
	SET RACE_URLS;
	RUN;
%MEND ADD_POLL_URL;

%ADD_POLL_URL();

/******************************************************************************************\ 
Adds variable for the eventual (D)ataset(Names), poll_url, result_url 
that can be accessed later on sequentially to create datasets for those names
\*******************************************************************************************/
%MACRO ADD_DNAMES();
	DATA RACE_URLS;
	SET RACE_URLS;
	/* Change / into > for regex purposes */
	filter2 = tranwrd(filter, "/",">");
	/* remove what is default to all url names */
	filter2 = tranwrd(filter2, "http:>>www.realclearpolitics.com>epolls>","");
	length = length(filter2);
	/* remove year> and .html */
	filter2 = substr(filter2,8,length-8-4);

	/* remove each '>' sequentially */
	patternID = prxparse('/>/');
	position =  prxmatch(patternID, filter2);
	length = length(filter2);
	filter2 = substr(filter2, position+1);

	position =  prxmatch(patternID, filter2);
	length = length(filter2);
	filter2 = substr(filter2, position+1);

	position =  prxmatch(patternID, filter2);
	length = length(filter2);
	filter2 = substr(filter2, position+1);
	
	/* remove four digit id and '-' */
	length = length(filter2);
	id=substr(filter2, 1 , length-5);
	
	/* since SAS dataset name cannot be too long, remove state name, type of race, and district if available from title*/
	patternID = prxparse('/_/');
	position =  prxmatch(patternID, id);
	length = length(id);
	id = substr(id, position+1);

	patternID = prxparse('/_/');
	position =  prxmatch(patternID, id);
	length = length(id);
	id = substr(id, position+1);

	id = tranwrd(id, "district_","");
	id = tranwrd(id, "senate_","");
	id = tranwrd(id, "governor_","");
	id = tranwrd(id, "special_election_","");
	id = tranwrd(id, "atlarge_","");
	id = tranwrd(id, "th_","");
	id = tranwrd(id, "nd_","");

	id = tranwrd(id, "0","");
	id = tranwrd(id, "1","");
	id = tranwrd(id, "2","");
	id = tranwrd(id, "3","");
	id = tranwrd(id, "4","");
	id = tranwrd(id, "5","");
	id = tranwrd(id, "6","");
	id = tranwrd(id, "7","");
	id = tranwrd(id, "8","");
	id = tranwrd(id, "9","");

	/*Anomalies*/
	id = tranwrd(id, "senat",".");

	id = compress(id);

	IF id = "." THEN DELETE;

	result_dname = put(id,1000.)||"_result";
	result_dname = compress(result_dname);

	poll_dname = put(id,1000.)||"_poll";
	poll_dname = compress(poll_dname);

	error_dname = put(id,1000.)||"_error";
	error_dname = compress(error_dname);



	RUN;
	QUIT;
	
	DATA RACE_URLS (DROP= patternID position length id filter2 RENAME=(filter=result_url));
	SET RACE_URLS;
	RUN;
	QUIT;
	
%MEND ADD_DNAMES;

%ADD_DNAMES();

/******************************************************************************************\ 
For a given poll url, try to access it, if it returns an error then mark it to be removed
\*******************************************************************************************/

/*Gracefully terminating a datastep if input file is unavailable*/
/*http://www2.sas.com/proceedings/sugi31/029-31.pdf*/
%MACRO VERIFY_POLL(URL, NUM);

	FILENAME HTML URL &url DEBUG LRECL=10000; 
	QUIT;
	
	DATA HTML; 
	id=1; 
	INFILE HTML length=len; 
	INPUT record $varying10000. len; 
	RUN; 
	QUIT;

	%put &syserr;
	%if &syserr > 0 %then %do;
	/*DATA RACE_URLS;
	SET RACE_URLS;
	IF POLL_URL = &URL THEN DELETE;
	RUN;
	QUIT;*/
	DATA RACE_URLS;
	SET RACE_URLS;
	/* Index all the races that do not have (does not exist (DNE)) sufficient historical polling data */
	IF POLL_URL = &URL THEN POLL_DNE = 1;
	RUN;
	QUIT;
	%end;

	/* Edge Case, sometimes the URL for xml poll data is accessible yet there is no actual data, tag these races to be removed as well */
	%else %do;
	DATA _NULL_;
	SET HTML;
	/* Setting MACRO within a DATA STEP using SYMPUT() */
	/* http://support.sas.com/documentation/cdl/en/mcrolref/61885/HTML/default/viewer.htm#a000210266.htm */
	length = length(record);
	call symput('edge_length', length);
	RUN;
	QUIT;
	%if &edge_length LT 100 %then %do;
	DATA RACE_URLS;
	SET RACE_URLS;
	/* Index all the races that do not have (does not exist (DNE)) sufficient historical polling data */
	IF POLL_URL = &URL THEN POLL_DNE = 1;
	RUN;
	QUIT;
	%end;
	%end;
%MEND VERIFY_POLL;

/* Edge Test using a url that is accessible yet has no actual polling data */
/*%VERIFY_POLL("http://cdn.realclearpolitics.com/epolls/charts/1163.xml", 1);*/

/******************************************************************************************\ 
Remove all marked observations (for not having past polling results)
\*******************************************************************************************/

/* Call this Macro after Macro SCANLOOP_IDENTIFY since SCANLOOP_IDENTIFY actually goes through RACE_URL to mark which race's poll DNE */
%MACRO REMOVE_POLLS;
	DATA RACE_URLS;
	SET RACE_URLS;
	IF POLL_DNE = 1 THEN DELETE;
	RUN;
	QUIT;
	DATA RACE_URLS (DROP= POLL_DNE);
	SET RACE_URLS;
	RUN;
	QUIT;
%MEND REMOVE_POLLS;

/******************************************************************************************\ 
Remove races with three candidates
\*******************************************************************************************/
%MACRO REMOVE_THREE_RACE;
	DATA RACE_URLS;
	SET RACE_URLS;
	IF count(POLL_DNAME,"_") > 3 THEN DELETE;
	RUN;
	QUIT;
%MEND REMOVE_THREE_RACE;

/******************************************************************************************\ 
Parse the HTML of a certain returned poll page for the race results contained in a table
html element
\*******************************************************************************************/

/* Note: the DATASET must be in HTML format, usually derived from %GET_HTML */
/* where D_OUT_NAME is dataset output name */
%MACRO FIND_RACE_RESULT(DATASET, D_OUT_NAME);

	proc sql;
	   create table &D_OUT_NAME as
	   select * from &DATASET 
	   where prxmatch('(<p>?)', record);
	run;
	quit;
	proc sql;
	   create table &D_OUT_NAME as
	   select * from &D_OUT_NAME 
	   where prxmatch('(<td?)', record);
	run;
	quit;

	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	filter = strip(record);
	filter = compress(filter);
	length = length(filter);
	/* it seems this pattern denotes the actual results, as can be seen on the webpage */
	patternID = prxparse('/--/');
	position =  prxmatch(patternID, filter);
	/* Pattern on website is usually 20 characters within -- contain the actual result */
	/* Also we restrict at 20 to remove any independents that may have run in the races as a three variable analysis causes
	issues with the code later on and does not add that much statistical benefit since our observations are large */
	/* we add +20 since --</td><td>--</td> is roughly 20 characters where -- is our marker */
	race_result = substr(filter, position+20, 20);
	RUN;
	QUIT;

	DATA &D_OUT_NAME (DROP = id record filter length patternID position);
	SET &D_OUT_NAME;
	race_result = tranwrd(race_result, "<","");
	race_result = tranwrd(race_result, ">","");
	race_result = tranwrd(race_result, "--","");
	/* removes all lower case letters */
	race_result = compress(race_result,'ABCD','l');
	/* we use '/' as the marker between the two results so we can seperate them into different columns */
	/* we change '/' to m for regex purposes */
	race_result = tranwrd(race_result, "/","m");
	/* remove white, trailing, and leading spaces*/
	race_result = compress(race_result);
	length_update = length(race_result);
	patternID_update = prxparse('/m/');
	position_update =  prxmatch(patternID_update, race_result);
	candidate1_result = substr(race_result, 1, position_update-1);
	candidate2_result = substr(race_result, position_update+1, length_update-position_update+1);

	/* Convert Char Variable into Numerical Variable! VERY IMPORTANT! Can cause great headaches later on when trying to do math */
	/*http://www.ciser.cornell.edu/faq/sas/char2num.shtml*/
	candidate1_actual_result = candidate1_result +0;
	candidate2_actual_result = candidate2_result + 0;
	RUN;
	QUIT;

	/*"Hey Kids Its Clean Up Time!"-Barney */
	DATA &D_OUT_NAME (DROP = race_result length_update patternID_update position_update candidate1_result candidate2_result);
	SET &D_OUT_NAME;
	RUN;
	QUIT;
	
	/* Rename back to original for code integrity purposes*/
	DATA &D_OUT_NAME (RENAME =(candidate1_actual_result = candidate1_result candidate2_actual_result = candidate2_result));
	SET &D_OUT_NAME;
	RUN;
	QUIT;

	/* Certain webpages produce a strange second row, perhaps the existence of uncaught trailing spaces? Regardless it does not affect the first row, but still delete
	any extra rows that may be in the dataset */
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	num = _N_;
	RUN;
	QUIT;
	
	/* Assign a serial number to each observation */
	/*https://kb.iu.edu/d/adxp*/
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	num = _N_;
	RUN;
	QUIT;

	/* if num =2 then there is a second row present, delete it */
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	IF num = 2 THEN DELETE;
	RUN;
	QUIT;

	DATA &D_OUT_NAME(DROP = num);
	SET &D_OUT_NAME;
	RUN;
	QUIT;

%MEND FIND_RACE_RESULT;

/******************************************************************************************\ 
Parse the XML file returned by the poll url call and format into data set
\*******************************************************************************************/


/* !!!!!!!!!!!!! IMPORTANT!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!*/

/* NOTE: CHANGE THE PATH NAME OF THE MAP FILE TO WHATEVER IS YOUR ROOT DIRECTORY */

%MACRO FIND_POLL_RESULT(URL, Poll_DNAME);
	filename POLL URL &URL;
	filename MAP "C:\Users\johnlo\Desktop\RCP.map";
	libname POLL xml xmlmap=MAP;
	
	/* Test to verify import successful */
	/*
	PROC print data=POLL.CANDIDATES; 
	RUN; 
	QUIT; 
	*/
	DATA POLL_CANDIDATE;
	SET POLL.CANDIDATES;
	RUN;
	QUIT;

	/*Prepare Dates*/
	DATA POLL_DATES;
	SET POLL.DATES;
	RUN;
	QUIT;

	DATA POLL_DATES;
	SET POLL_DATES;
	RUN;
	QUIT;
 
	/* TODO: What about edge case if there are three candidates? */
	/*Prepare Candidate1 Results */
	DATA POLL_CANDIDATE1;
	SET POLL.CANDIDATES; /* Call Candidates Dataset from Poll Library */
	IF id = 2 THEN DELETE;
	RUN;
	QUIT;

	DATA POLL_CANDIDATE1;
	SET POLL_CANDIDATE1;
	IF party = "#D30015" THEN party = "_R";
	IF party = "#3B5998" THEN party = "_D";
	RUN;
	QUIT;

	DATA POLL_CANDIDATE1;
	SET POLL_CANDIDATE1;
	name_party = name || party;
	name_party = compress(name_party);
	RUN;
	QUIT;

	DATA POLL_CANDIDATE1 (DROP = id party name);
	SET POLL_CANDIDATE1;
	call symput('Candidate1', strip(name_party));
	RUN;
	QUIT;

	DATA POLL_CANDIDATE1;
	SET POLL_CANDIDATE1;
	&Candidate1 = poll;
	RUN;
	QUIT;

	DATA POLL_CANDIDATE1(DROP = poll name_party);
	SET POLL_CANDIDATE1;
	RUN;
	QUIT;


	
	/*Prepare Candidate1 Results */
	DATA POLL_CANDIDATE2;
	SET POLL.CANDIDATES; /* Call Candidates Dataset from Poll Library */
	IF id = 1 THEN DELETE;
	RUN;
	QUIT;

	DATA POLL_CANDIDATE2;
	SET POLL_CANDIDATE2;
	IF party = "#D30015" THEN party = "_R";
	IF party = "#3B5998" THEN party = "_D";
	RUN;
	QUIT;

	DATA POLL_CANDIDATE2;
	SET POLL_CANDIDATE2;
	name_party = name || party;
	name_party = compress(name_party);
	RUN;
	QUIT;

	DATA POLL_CANDIDATE2 (DROP = id party name);
	SET POLL_CANDIDATE2;
	call symput('Candidate2', strip(name_party));
	RUN;
	QUIT;

	DATA POLL_CANDIDATE2;
	SET POLL_CANDIDATE2;
	&Candidate2 = poll;
	RUN;
	QUIT;

	DATA POLL_CANDIDATE2(DROP = poll name_party);
	SET POLL_CANDIDATE2;
	RUN;
	QUIT;

	/*Merge into one Dataset...finally!*/
	DATA &Poll_DNAME;
	MERGE POLL_DATES POLL_CANDIDATE1 POLL_CANDIDATE2;
	RUN;
	QUIT;

	DATA &Poll_DNAME;
	SET &Poll_DNAME;
	new_date = input(substr(strip(date),1,10),MMDDYY10.);
	RUN;
	QUIT;

	DATA &Poll_DNAME (DROP =date RENAME=(new_date=date));
	SET &Poll_DNAME;
	RUN;
	QUIT;

%MEND FIND_POLL_RESULT;

/******************************************************************************************\ 
Iterative: Call VERIFY_POLL to remove observation if calling poll url returns an ERROR
\*******************************************************************************************/

/* 1 . Macro to SCAN through RACE_URLS and if poll xml returns 404 error removes race from list of races to be evaluated */
%MACRO SCANLOOP_IDENTIFY(SCANFILE,FIELD1,FIELD2,FIELD3,FIELD4);
	/* First obtain the number of records in RACE_URLS */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables of interest in macro variables */
	CALL SYMPUT('VAR1',&FIELD1);
	CALL SYMPUT('VAR2',&FIELD2);
	STOP;
	RUN;
	/* now perform the tasks that  wish repeated for each observation */
	%VERIFY_POLL("&var2", &I);

	%END;
%MEND SCANLOOP_IDENTIFY;
%SCANLOOP_IDENTIFY(RACE_URLS,result_url,poll_url, result_dname, poll_dname);

/* Remove all races where polls have been identified as DNE, update Dataset RACE_URLS */
%REMOVE_POLLS;
%REMOVE_THREE_RACE;

/******************************************************************************************\ 
Iterative: Call %GET_HTML %FIND_RACE_RESULT %FIND_POLL_RESULT for each observation 
in database
\*******************************************************************************************/

/* 2 . Macro to SCAN through updated RACE_URLS and create race result dataset and create the poll dataset for each race */
%MACRO SCANLOOP_RACE_RESULT(SCANFILE,FIELD1,FIELD2,FIELD3, FIELD4);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* result_url */
	CALL SYMPUT('VAR2',&FIELD2); /* poll_url */
	CALL SYMPUT('VAR3',&FIELD3); /* result_dname */
	CALL SYMPUT('VAR4',&FIELD4); /* poll_dname */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */
	%GET_HTML("&VAR1");
	%FIND_RACE_RESULT(HTML,&VAR3);
	%FIND_POLL_RESULT("&VAR2", &VAR4);

	%END;
%MEND SCANLOOP_RACE_RESULT;
%SCANLOOP_RACE_RESULT(RACE_URLS,result_url,poll_url, result_dname, poll_dname);

/******************************************************************************************\ 
Parse the HTML of a certain returned poll page for the candidate names contained in a table
html element
\*******************************************************************************************/

%MACRO GET_NAMES(DATASET, NUM);
	* http://stackoverflow.com/questions/17851217/list-only-the-column-names-of-a-dataset;
	/* There are some potential drawbacks to using the above solution from stackoverflow 
	since after testing proc contents does not always return the column headings in order,
	which alters our percentage calculations later on and ultimately our error calculations. 
	Therefore since proc tranpose always lists column headings in order we will use that 
	as a method to determine our headings and their positions. We also know due to the way
	the poll results were formatted and scraped that _polls are accurate in positioning */

	%if &NUM = 1 %then %do;

	proc transpose data=&DATASET
	               out=meta&NUM;
	RUN;
	QUIT;

	DATA meta&NUM(KEEP = _NAME_);
	SET meta&NUM;
	RUN;
	QUIT;

	DATA meta&NUM;
	SET meta&NUM;
	ID = &NUM;
	RUN;
	QUIT;
	
	%end;
	%else %do;
	proc transpose data=&DATASET
	               out=meta_merge;
	RUN;
	QUIT;

	DATA meta_merge(KEEP = _NAME_);
	SET meta_merge;
	RUN;
	QUIT;

	DATA meta_merge;
	SET meta_merge;
	ID = &NUM;
	RUN;
	QUIT;

	DATA meta1;
	SET meta1 meta_merge;
	RUN;
	QUIT;

	%end;
	
	* proc print data=meta ;
	* run; 
%MEND GET_NAMES;

/******************************************************************************************\ 
Insert names into the database
\*******************************************************************************************/

%MACRO INSERT_NAMES;
	/* Here we use the prefix option to prevent SAS from making the candidates into the 
	column names and producing some strange tranpose results */
	PROC transpose data=Meta1
               	   out=Meta1
	   	   prefix=col;
	by id;
	var _name_;
	run;
	quit;

	DATA META1 (DROP = ID _NAME_ _LABEL_ COL3 RENAME=(COL1=CANDIDATE1 COL2=CANDIDATE2));
	SET META1;
	RUN;
	QUIT;

	DATA RACE_URLS;
	MERGE RACE_URLS META1;
	RUN;
	QUIT;

	/* Clean out any anomalies */
	DATA RACE_URLS;
	SET RACE_URLS;
	IF CANDIDATE2 = 'date' THEN DELETE;
	RUN;
	QUIT;

%MEND INSERT_NAMES;

/******************************************************************************************\ 
Iterative: %GET_NAMES
\*******************************************************************************************/


/* 3 . Macro to SCAN RACE_URLS and update it with candidate 1 and 2 name information 
from poll dataset using GET_COLUMN_NAMES and replacing each with SCANLOOP_UPDATE_NAMES, 
basically a nested for loop in a sense*/
%MACRO SCANLOOP_FIND_NAMES(SCANFILE,FIELD1,FIELD2,FIELD3, FIELD4);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* result_url */
	CALL SYMPUT('VAR2',&FIELD2); /* poll_url */
	CALL SYMPUT('VAR3',&FIELD3); /* result_dname */
	CALL SYMPUT('VAR4',&FIELD4); /* poll_dname */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */
	%GET_NAMES(&VAR4, &I);
	/*%SCANLOOP_UPDATE_NAMES(META, _NAME_, &VAR3);*/
	%END;
%MEND SCANLOOP_FIND_NAMES;

%SCANLOOP_FIND_NAMES(RACE_URLS,result_url,poll_url, result_dname, poll_dname);
%INSERT_NAMES;

/******************************************************************************************\ 
Insert candidate party affliation into database
\*******************************************************************************************/

/* We Also want to insert parties for graph coloring purposes */
%MACRO INSERT_PARTIES();

	DATA RACE_URLS;
	SET RACE_URLS;
	patternID = prxparse('/_/');
	position = prxmatch(patternID, CANDIDATE1);
	CANDIDATE1_PARTY = substr(CANDIDATE1, position+1);

	patternID = prxparse('/_/');
	position = prxmatch(patternID, CANDIDATE2);
	CANDIDATE2_PARTY = substr(CANDIDATE2, position+1);
	RUN;
	QUIT;
	
	/* Clean up any malformed data */
	/* Assumes we've removed any three party races */
	DATA RACE_URLS(DROP = patternID position);
	SET RACE_URLS;
	IF CANDIDATE1_PARTY = 'R' THEN CANDIDATE2_PARTY ='D';
	IF CANDIDATE1_PARTY = 'D' THEN CANDIDATE2_PARTY ='R';
	IF CANDIDATE2_PARTY = 'R' THEN CANDIDATE1_PARTY ='D';
	IF CANDIDATE2_PARTY = 'D' THEN CANDIDATE1_PARTY ='R';

	IF length(CANDIDATE1_PARTY) > 1 THEN CANDIDATE1_PARTY ='D';
	IF length(CANDIDATE1_PARTY) < 1 THEN CANDIDATE1_PARTY ='D';
	IF length(CANDIDATE2_PARTY) > 1 THEN CANDIDATE2_PARTY ='R';
	IF length(CANDIDATE2_PARTY) < 1 THEN CANDIDATE2_PARTY ='R';

	IF CANDIDATE1_PARTY = 'R' THEN CANDIDATE2_PARTY ='D';
	IF CANDIDATE1_PARTY = 'D' THEN CANDIDATE2_PARTY ='R';
	IF CANDIDATE2_PARTY = 'R' THEN CANDIDATE1_PARTY ='D';
	IF CANDIDATE2_PARTY = 'D' THEN CANDIDATE1_PARTY ='R';

	IF CANDIDATE1_PARTY = 'R' THEN CANDIDATE1_PARTY ='Red';
	IF CANDIDATE1_PARTY = 'D' THEN CANDIDATE1_PARTY ='Blue';
	IF CANDIDATE2_PARTY = 'R' THEN CANDIDATE2_PARTY ='Red';
	IF CANDIDATE2_PARTY = 'D' THEN CANDIDATE2_PARTY ='Blue';

	RUN;
	QUIT;

%MEND INSERT_PARTIES;
%INSERT_PARTIES;


/************************************************************************************************************************\ 
Part 2: Exploratory Analysis

	MACROS INCLUDED 

	IN ORDER OF APPEARANCE
	NOT IN ORDER OF CALL

	%ERROR_RESULT
	%ERROR_FORMAT
	%ERROR_FORMAT_CONCAT
	%GRAPH_DATASET
	%SCANLOOP_ERROR_RESULTS
	%SCANLOOP_GRAPH
	%GRAPH_ERROR_RESULTS_CONCAT 
	%EVAL_DAYS 




\************************************************************************************************************************/;

/* This was for loading from external data source, no longer supported */
/*
%MACRO CREATE_DATASET_POLL(PATH);
	* infile in&i; 
	* for some strange reason it ignores the first '.' after macro path ;
	proc import datafile="C:\Users\johnlo\Desktop\test\&PATH..csv"
	out=&PATH
	dbms=csv
	replace;
	getnames=yes;
	run;
	proc contents data=sashelp.class out=contents noprint;
	run;
%MEND CREATE_DATASET_POLL;
%CREATE_DATASET_POLL(Deeds_McDonnell_poll);

%MACRO CREATE_DATASET_RESULT(PATH);
	* infile in&i; 
	* for some strange reason it ignores the first '.' after macro path ;
	proc import datafile="C:\Users\johnlo\Desktop\result\&PATH..csv"
	out=&PATH
	dbms=csv
	replace;
	getnames=yes;
	run;
	proc contents data=sashelp.class out=contents noprint;
	run;
%MEND CREATE_DATASET_RESULT;
%CREATE_DATASET_RESULT(Deeds_McDonnell_result);
*/



/******************************************************************************************\ 
Calculate the error for each candidate in a race
\*******************************************************************************************/
%MACRO ERROR_RESULT(PATH, PATH_RESULT, CANDIDATE1, CANDIDATE2, ERROR_DNAME);

	DATA &ERROR_DNAME;
	SET &PATH;
	CANDIDATE1_PCT = &CANDIDATE1 / (&CANDIDATE1 + &CANDIDATE2) * 100;
	CANDIDATE2_PCT = &CANDIDATE2 / (&CANDIDATE1 + &CANDIDATE2) * 100;
	/* Remove Rows without Observations */
	IF CANDIDATE1_PCT = . THEN DELETE;
	RUN;
	QUIT;

	/************************************************************************************************/
	/* Append the Last_Date column to every row of observation in order to calculate forecast_length */
	/* Work flow for appending the last observation on to every row of a new variable (Last_Date) */
	/* This work flow could probably be made into a seperate MACRO */
	/* Index the last date with 1, may be irrelevant later on, but still good to index */
	DATA &ERROR_DNAME;
	SET &ERROR_DNAME end = _end;
	end = _end;
	RUN;
	QUIT;

	/* Create temp data set with just the dates in order to keep the correct amount of observations */
	DATA LAST_DATE (KEEP=date end RENAME=(date=last_date));
	SET &ERROR_DNAME;
	RUN;
	QUIT;

	/* Update temp data set that reverses the order of the dates, so the last becomes the first */
	/* Also remove end variable so data can be transposed using BY later on */
	DATA LAST_DATE(DROP = end) ;
	DO k= nobs TO 1 BY -1;
	SET LAST_DATE nobs = nobs POINT=k;
	OUTPUT;
	END;
	STOP;
	RUN;
	QUIT;

	/* Transpose the reversed data set to set up a BY tranposition */
	PROC TRANSPOSE DATA = LAST_DATE
	   OUT = LAST_DATE;
	RUN;
	QUIT;
	
	/* Since the last date was reversed as the first, it falls under the default variable COL1*/
	/* Tranpose the data using BY in order to replicate the last date throughout all the observations */
	PROC TRANSPOSE DATA = LAST_DATE
	   	   OUT = LAST_DATE;
	BY COL1;
	RUN;
	QUIT;

	/* Drop uneccessary dates and NAMES to have just one column of the last date filling up as many rows are there are original date observations*/
	/* Rename the column last_date */
	DATA LAST_DATE (DROP = _NAME_ last_date RENAME=(COL1=last_date));
	SET LAST_DATE;
	RUN;
	QUIT;

	/* Merge back into original data set */
	DATA &ERROR_DNAME;
	MERGE LAST_DATE &ERROR_DNAME;
	/* Remove rows without observations */ /*Area of Contention */
	SET &ERROR_DNAME;
	IF last_date = . THEN DELETE;
	RUN;
	QUIT;

	/************************************************************************************************/
	/* Append the actual result for Candidate1 column to every row of observation in order to calculate forecast_length */
	/* Append the actual result for Candidate2 column to every row of observation in order to calculate forecast_length */
	/* While we're at it, it'd be a good time to use the one-to-many with the results with the main dataset as well */
	DATA ACTUAL_RESULT;
	/* Borrow LAST_DATE for its correct number of observations we have to merge into (one-to-many) */
	MERGE LAST_DATE &PATH_RESULT;
	RUN;
	QUIT;

	/* Clean out unecessary items now that we have correct number of observations to merge into */
	/* For Candidate 1 (C1) */
	DATA ACTUAL_RESULT_C1 (DROP= last_date end CANDIDATE2_RESULT);
	SET ACTUAL_RESULT;
	RUN;
	QUIT;
	
	/* For Candidate 2 (C2) */
	DATA ACTUAL_RESULT_C2 (DROP= last_date end CANDIDATE1_RESULT);
	SET ACTUAL_RESULT;
	RUN;
	QUIT;

	/* Transpose the reversed data set to set up a BY tranposition */
	PROC TRANSPOSE DATA = ACTUAL_RESULT_C1
	   	   OUT = ACTUAL_RESULT_C1;
	RUN;
	QUIT;
	PROC TRANSPOSE DATA = ACTUAL_RESULT_C2
	   	   OUT = ACTUAL_RESULT_C2;
	RUN;
	QUIT;
	
	/* Tranpose the data using BY in order to replicate the last date throughout all the observations */
	PROC TRANSPOSE DATA = ACTUAL_RESULT_C1
	   	   OUT = ACTUAL_RESULT_C1;
	BY COL1;
	RUN;
	QUIT;

	PROC TRANSPOSE DATA = ACTUAL_RESULT_C2
	   	   OUT = ACTUAL_RESULT_C2;
	BY COL1;
	RUN;
	QUIT;
	
	/* Drop uneccessary dates and NAMES to have just one column of the last date filling up as many rows are there are original date observations*/
	/* Rename the column last_date */
	DATA ACTUAL_RESULT_C1 (DROP = _NAME_ CANDIDATE1_RESULT RENAME=(COL1= CANDIDATE1_RESULTS));
	SET ACTUAL_RESULT_C1;
	RUN;
	QUIT;

	DATA ACTUAL_RESULT_C2 (DROP = _NAME_ CANDIDATE2_RESULT RENAME=(COL1= CANDIDATE2_RESULTS));
	SET ACTUAL_RESULT_C2;
	RUN;
	QUIT;

	/* Append the actual result for Candidate1 and Candidate 2 to every row of observation */
	DATA &ERROR_DNAME;
	MERGE &ERROR_DNAME ACTUAL_RESULT_C2 ACTUAL_RESULT_C1;
	RUN;
	QUIT;

	/************************************************************************************************/
	/* Calculate forecast_length column */
	/* Calculate Candidate1_error */
	/* Calculate Candidate1_error */
	/* Clean up the data set: drop end (index no longer necessary) and _ */
	DATA &ERROR_DNAME (DROP= end _);
	SET &ERROR_DNAME;
	forecast_length = last_date - date;
	CANDIDATE1_ERROR = CANDIDATE1_PCT - CANDIDATE1_RESULTS;
	CANDIDATE2_ERROR = CANDIDATE2_PCT - CANDIDATE2_RESULTS;
	/*CLEANUP */
	IF CANDIDATE2_RESULTS = "." THEN DELETE;
	RUN;
	QUIT;
	

%MEND ERROR_RESULT;
/* TEST */
/*%ERROR_RESULT(mccain_vs_glassman_poll, mccain_vs_glassman_result, McCain_R, GLASSMAN_D, ZZZZZZZERROR );*/

/******************************************************************************************\ 
Reformat the dataset so it can be SET with other datasets
\*******************************************************************************************/

%MACRO ERROR_FORMAT(DATASET_ERROR_RESULT, CANDIDATE1, CANDIDATE2);

	/* Cleaning */
	/* We only need the variables CANDIDATE, ERROR, FORECAST LENGTH, PERCENTAGE for each Candidate so drop the rest */

	DATA ERROR_FORMAT_C1 (KEEP= &CANDIDATE1 CANDIDATE1_ERROR  forecast_length CANDIDATE1_PCT);
	SET &DATASET_ERROR_RESULT;
	RUN;
	QUIT;

	DATA ERROR_FORMAT_C2 (KEEP= &CANDIDATE2 CANDIDATE2_ERROR forecast_length CANDIDATE2_PCT);
	SET &DATASET_ERROR_RESULT;
	RUN;
	QUIT;

	/* FORMATTING */
	/* We want each Observation to have the following structure: CANDIDATE_NAME, ERROR, FORECAST_LENGTH, PERCENTAGE */
	/* However CANDIDATE_NAME is currently a column heading (variable) not an observation for each row */
	/* Need to transform this longitudinal data into individual candidate data */
	/* Just keep the number of observations and the candidate name, we will merge it back to ERROR_FORMAT_C later on */
	DATA ERROR_FORMAT_C1_NAME (KEEP= &CANDIDATE1);
	SET ERROR_FORMAT_C1;
	RUN;
	QUIT;

	DATA ERROR_FORMAT_C2_NAME (KEEP= &CANDIDATE2);
	SET ERROR_FORMAT_C2;
	RUN;
	QUIT;

	/* TRANSPOSE 1 */

	PROC TRANSPOSE DATA = ERROR_FORMAT_C1_NAME
	   	   OUT = ERROR_FORMAT_C1_NAME;
	RUN;
	QUIT;

	PROC TRANSPOSE DATA = ERROR_FORMAT_C2_NAME
	   	   OUT = ERROR_FORMAT_C2_NAME;
	RUN;
	QUIT;

	/* TRANSPOSE 2 with BY */
	PROC TRANSPOSE DATA = ERROR_FORMAT_C1_NAME
	   	   OUT = ERROR_FORMAT_C1_NAME (KEEP = _NAME_ );
	BY _NAME_;
	RUN;
	QUIT;

	PROC TRANSPOSE DATA = ERROR_FORMAT_C2_NAME
	   	   OUT = ERROR_FORMAT_C2_NAME (KEEP = _NAME_ );
	BY _NAME_;
	RUN;
	QUIT;

	/* MERGE */

	DATA ERROR_FORMAT_C1(DROP = &CANDIDATE1 RENAME=(_NAME_=Candidate CANDIDATE1_PCT = PCT CANDIDATE1_ERROR = ERROR));
	MERGE ERROR_FORMAT_C1 ERROR_FORMAT_C1_NAME;
	RUN;
	QUIT;

	DATA ERROR_FORMAT_C2(DROP = &CANDIDATE2 RENAME=(_NAME_=Candidate CANDIDATE2_PCT = PCT CANDIDATE2_ERROR = ERROR));
	MERGE ERROR_FORMAT_C2 ERROR_FORMAT_C2_NAME;
	RUN;
	QUIT;
	

%MEND ERROR_FORMAT;
/* Test */
/*%ERROR_FORMAT(ERROR, McCain_R, GLASSMAN_D);*/

/******************************************************************************************\ 
Merge all error datasets into one dataset
\*******************************************************************************************/

/* NOTE: IF YOU WANT TO RE-RUN THIS MACRO SPECIFICALLY, YOU NEED TO DELETE ERROR_RESULTS_CONCAT DATASET FIRST! */
/* Concat all formatted error data for each candidate to main Error Dataset */
%MACRO ERROR_FORMAT_CONCAT(ERROR_C1_DATASET, ERROR_C2_DATASET);
	%IF %SYSFUNC(EXIST(ERROR_RESULTS_CONCAT)) %THEN %DO;
	DATA ERROR_RESULTS_CONCAT;
	SET ERROR_RESULTS_CONCAT &ERROR_C1_DATASET &ERROR_C2_DATASET;
	RUN;
	QUIT;
	%END;
	%ELSE %DO;
	DATA ERROR_RESULTS_CONCAT;
	SET &ERROR_C1_DATASET &ERROR_C2_DATASET;
	RUN;
	QUIT;
	%END;
%MEND ERROR_FORMAT_CONCAT;

/******************************************************************************************\ 
Graph the polling data
\*******************************************************************************************/

%MACRO GRAPH_DATASET(DATASET, CANDIDATE1, CANDIDATE2, CANDIDATE1_PARTY, CANDIDATE2_PARTY);

	PROC sgplot data = &DATASET;
	series X = date Y=&CANDIDATE1 	  /  lineattrs=(color="light strong &CANDIDATE1_PARTY" thickness=3); 
	series X = date Y=CANDIDATE1_RESULTS  /  lineattrs=(color="light strong &CANDIDATE1_PARTY" thickness=1 pattern=dash); 
	series X = date Y=&CANDIDATE2 	  /  lineattrs=(color="light strong &CANDIDATE2_PARTY" thickness=3); 
	series X = date Y=CANDIDATE2_RESULTS  /  lineattrs=(color="light strong &CANDIDATE2_PARTY" thickness=1 pattern=dash); 
	TITLE "Race Polls";
	YAXIS GRID DISPLAY = (NOLABEL) VALUES=(20 to 80 BY 10);

	RUN;
	QUIT;
	
%MEND GRAPH_DATASET;

/******************************************************************************************\ 
Iterative: %ERROR_RESULT %ERROR_FORMAT %ERROR_FORMAT_CONCAT
\*******************************************************************************************/

%MACRO SCANLOOP_ERROR_RESULTS(SCANFILE,FIELD1,FIELD2,FIELD3, FIELD4, FIELD5);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* CANDIDATE1 */
	CALL SYMPUT('VAR2',&FIELD2); /* CANDIDATE2 */
	CALL SYMPUT('VAR3',&FIELD3); /* result_dname */
	CALL SYMPUT('VAR4',&FIELD4); /* poll_dname */
	CALL SYMPUT('VAR5',&FIELD5); /* error_dname */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */

	%ERROR_RESULT(&VAR4, &VAR3, &VAR1, &VAR2, &VAR5);
	%ERROR_FORMAT(&VAR5, &VAR1, &VAR2);
	%ERROR_FORMAT_CONCAT (ERROR_FORMAT_C1, ERROR_FORMAT_C2);

	%END;
%MEND SCANLOOP_ERROR_RESULTS;
%SCANLOOP_ERROR_RESULTS(RACE_URLS,CANDIDATE1,CANDIDATE2, result_dname, poll_dname, error_dname);

/******************************************************************************************\ 
Iterative: %ERROR_RESULT %ERROR_FORMAT %ERROR_FORMAT_CONCAT
\*******************************************************************************************/

%MACRO SCANLOOP_GRAPH(SCANFILE,FIELD1,FIELD2,FIELD3, FIELD4, FIELD5, FIELD6, FIELD7);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* CANDIDATE1 */
	CALL SYMPUT('VAR2',&FIELD2); /* CANDIDATE2 */
	CALL SYMPUT('VAR3',&FIELD3); /* result_dname */
	CALL SYMPUT('VAR4',&FIELD4); /* poll_dname */
	CALL SYMPUT('VAR5',&FIELD5); /* error_dname */
	CALL SYMPUT('VAR6',&FIELD6); /* candidate1_party */
	CALL SYMPUT('VAR7',&FIELD7); /* candidate2_party */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */
	%GRAPH_DATASET(&VAR5, &VAR1, &VAR2, &VAR6, &VAR7);

	%END;
%MEND SCANLOOP_GRAPH;
%SCANLOOP_GRAPH(RACE_URLS,CANDIDATE1,CANDIDATE2, result_dname, poll_dname, error_dname, candidate1_party, candidate2_party);

/******************************************************************************************\ 
Graph Error distribution
\*******************************************************************************************/
/*Histogram of Errors after error_data and concat procedures */
/* RUN AFTER SCANLOOP */
%MACRO GRAPH_ERROR_RESULTS_CONCAT (DATASET, TITLE);
/* Testing with well formatted external file */
	/*
	proc import datafile="C:\Users\johnlo\Desktop\test\&PATH..csv"
	out=&PATH
	dbms=csv
	replace;
	getnames=yes;
	run;
	*/

	/* TODO: Customize the histogram some more */
	PROC UNIVARIATE DATA = &DATASET noprint;
	TITLE &TITLE;
	VAR ERROR;
	HISTOGRAM ERROR / grid WBARLINE=3 endpoints=-20 to 20 by 1;
	INSET STD = 'Standard Deviation' (6.3)/ FONT='Arial' POS = NW HEIGHT=3;
	RUN;
	QUIT;
%MEND GRAPH_ERROR_RESULTS_CONCAT;

%GRAPH_ERROR_RESULTS_CONCAT(ERROR_RESULTS_CONCAT, "Distribution of error of every polling measurement in the data");

/******************************************************************************************\ 
Graph Error Distribution by changing forecast_length
\*******************************************************************************************/

/* Some Exploratory Analysis */
/* How much more or less accurate are the polls if forecast_length is less than a week ( <7 ) ? */
/* How much more or less accurate are the polls if forecast_length is more a month ( >30 ) ? */
%MACRO EVAL_DAYS (DATASET);
	DATA EVAL_DAYS_7;
	SET &DATASET;
	IF forecast_length > 7 THEN DELETE;
	RUN;
	QUIT;

	DATA EVAL_DAYS_30;
	SET &DATASET;
	IF forecast_length < 30 THEN DELETE;
	RUN;
	QUIT;

	DATA EVAL_DAYS_14;
	SET &DATASET;
	IF 14 < forecast_length < 30 THEN DELETE;
	RUN;
	QUIT;
	
	%GRAPH_ERROR_RESULTS_CONCAT(EVAL_DAYS_7, "Distribution of error of polling measurements whose forecast_length is less than a week");
	%GRAPH_ERROR_RESULTS_CONCAT(EVAL_DAYS_30, "Distribution of error of polling measurements whose forecast_length is more than a month" );
	%GRAPH_ERROR_RESULTS_CONCAT(EVAL_DAYS_14, "Distribution of error of polling measurements whose forecast_length is more than two weeks yet less than a month" );
%MEND EVAL_DAYS;

%EVAL_DAYS(ERROR_RESULTS_CONCAT);
/* perhaps make a graph showing the relationship between standard deviation (error) and eval_days? */

/******************************************************************************************\ 
Graph Scatter plot error versus forecast_length
\*******************************************************************************************/

PROC sgplot data = ERROR_RESULTS_CONCAT;

	scatter X = forecast_length Y=error / markerattrs=(size=5 symbol=CircleFilled color="Light Strong Brownish Orange" ) transparency=0.9;
	ellipse X = forecast_length Y=error / lineattrs=(thickness=2 color="Light Strong Blue") transparency = 0.7;
	yaxis grid values=(-20 to 20 by 5);
	xaxis grid display=(nolabel) values=(0 to 400 by 50);
	keylegend / location=inside position=bottomright;
	TITLE "Error versus Forecast_length";

RUN;
QUIT;

/************************************************************************************************************************\ 
Part 3: Predictions and Quantitative Analysis

	MACROS INCLUDED 

	IN ORDER OF APPEARANCE
	NOT IN ORDER OF CALL

	%GET_CURRENT_RACES
	%ADD_CURRENT_DNAMES
	%FIND_RACE_CURRENT
	%SCANLOOP_RACE_RESULT
	%FIND_NAME_CURRENT
	%SCANLOOP_RACE_CURRENT_NAME
	%ADD_NAMES_CURRENT
	%INSERT_CURRENT_PARTIES
	%BOOTSTRAP
	%WIN_LOSS 
	%SCANLOOP_CURRENT_BOOTSTRAP
	%MERGE_WIN_LOSS
	%CALC_PROB_WIN
	%GRAPH_PREDICTIONS
	%SCANLOOP_CURRENT_GRAPH




\************************************************************************************************************************/;


/******************************************************************************************\ 
Retrieve all the race urls from the javascript load script on the Real Clear Politics Page
\*******************************************************************************************/

/* It seems for more recent versions races are no longer hard-coded into the "search by race" 
as option tags. Rather option tags are now stored in the following */
%MACRO GET_CURRENT_RACES (NUM);
	%GET_HTML("http://www.realclearpolitics.com/epolls/2014/widget/search_by_race.js");

	%DO I=1 %TO &NUM;
	%IF &I = 1 %THEN %DO;
	DATA RACE_URLS_CURRENT;
	SET HTML;
	filter = strip(record);
	filter = compress(record);
	filter = tranwrd(filter,">","");
	filter = tranwrd(filter, 'value=\"','<');
	filter = tranwrd(filter, '\"\','>');
	patternID = prxparse('/</');
	position =  prxmatch(patternID, filter);
	filter = substr(filter, position);

	patternID = prxparse('/>/');
	position =  prxmatch(patternID, filter);
	hello = substr(filter, 2, position-2);

	filter = substr(filter,position+1);
	RUN;
	QUIT;

	PROC TRANSPOSE DATA = RACE_URLS_CURRENT OUT = RACE_URLS_CURRENT; 
	BY filter;
	VAR hello;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT_LIST (DROP = filter _NAME_);
	SET RACE_URLS_CURRENT;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT (KEEP = filter RENAME=(filter=record));
	SET RACE_URLS_CURRENT;
	RUN;
	QUIT;
	%END;
	%ELSE %DO;
	DATA RACE_URLS_CURRENT;
	SET RACE_URLS_CURRENT;
	filter = strip(record);
	filter = compress(record);
	filter = tranwrd(filter, 'value=\"','<');
	filter = tranwrd(filter, '\"\','>');
	patternID = prxparse('/</');
	position =  prxmatch(patternID, filter);
	filter = substr(filter, position);

	patternID = prxparse('/>/');
	position =  prxmatch(patternID, filter);
	hello = substr(filter, 2, position-2);

	filter = substr(filter,position+1);
	RUN;
	QUIT;

	PROC TRANSPOSE DATA = RACE_URLS_CURRENT OUT = RACE_URLS_CURRENT; 
	BY filter;
	VAR hello;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT_LIST_TEMP (DROP = filter _NAME_);
	SET RACE_URLS_CURRENT;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT_LIST;
	SET RACE_URLS_CURRENT_LIST_TEMP RACE_URLS_CURRENT_LIST;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT (KEEP = filter RENAME=(filter=record));
	SET RACE_URLS_CURRENT;
	RUN;
	QUIT;
	%END;
	%END;

	/*CLEAN UP ON AISLE 12 */
	/* Since we couldn't dynamically determine the number of current races (can probably be done using some sort of perl
	regex) we had to make a finitely determined do loop. Therefore we could have overshot the number of races returned, so
	we clean out those redundant or returned links that do not match actual race links */
	DATA RACE_URLS_CURRENT (RENAME=(COL1 = RESULT_URL));
	SET RACE_URLS_CURRENT_LIST;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT;
	SET RACE_URLS_CURRENT;
	length = length(RESULT_URL);
	IF length < 83 THEN DELETE;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT (DROP = length);
	SET RACE_URLS_CURRENT;
	RUN;
	QUIT;

%MEND GET_CURRENT_RACES;

%GET_CURRENT_RACES(100);


/******************************************************************************************\ 
Add dataset names current_dname and predict_dname to database be called later
\*******************************************************************************************/

/* Need to add the dataset name for the current results to RACE_URLS_CURRENT */ 
/* This MACRO is almost exactly the same as ADD_DNAMES from Part 1, save for a few variable and dataset changes */

%MACRO ADD_CURRENT_DNAMES();
	DATA RACE_URLS_CURRENT;
	SET RACE_URLS_CURRENT;
	/* Change / into > for regex purposes */
	filter2 = tranwrd(result_url, "/",">");
	/* remove what is default to all url names */
	filter2 = tranwrd(filter2, "http:>>www.realclearpolitics.com>epolls>","");
	length = length(filter2);
	/* remove year> and .html */
	filter2 = substr(filter2,8,length-8-4);

	/* remove each '>' sequentially */
	patternID = prxparse('/>/');
	position =  prxmatch(patternID, filter2);
	length = length(filter2);
	filter2 = substr(filter2, position+1);

	position =  prxmatch(patternID, filter2);
	length = length(filter2);
	filter2 = substr(filter2, position+1);

	position =  prxmatch(patternID, filter2);
	length = length(filter2);
	filter2 = substr(filter2, position+1);
	
	/* remove four digit id and '-' */
	length = length(filter2);
	id=substr(filter2, 1 , length-5);
	
	/* since SAS dataset name cannot be too long, remove state name, type of race, and district if available from title*/
	patternID = prxparse('/_/');
	position =  prxmatch(patternID, id);
	length = length(id);
	id = substr(id, position+1);

	patternID = prxparse('/_/');
	position =  prxmatch(patternID, id);
	length = length(id);
	id = substr(id, position+1);

	id = tranwrd(id, "district_","");
	id = tranwrd(id, "senate_","");
	id = tranwrd(id, "governor_","");
	id = tranwrd(id, "special_election_","");
	id = tranwrd(id, "atlarge_","");
	id = tranwrd(id, "th_","");
	id = tranwrd(id, "nd_","");

	id = tranwrd(id, "0","");
	id = tranwrd(id, "1","");
	id = tranwrd(id, "2","");
	id = tranwrd(id, "3","");
	id = tranwrd(id, "4","");
	id = tranwrd(id, "5","");
	id = tranwrd(id, "6","");
	id = tranwrd(id, "7","");
	id = tranwrd(id, "8","");
	id = tranwrd(id, "9","");

	/*Anomalies*/
	id = tranwrd(id, "senat",".");

	id = compress(id);

	IF id = "." THEN DELETE;

	current_dname = put(id,1000.)||"_current";
	current_dname = compress(current_dname);

	predict_dname = put(id,1000.)||"_predict";
	predict_dname = compress(predict_dname);

	RUN;
	QUIT;
	
	DATA RACE_URLS_CURRENT (DROP= patternID position length id filter2);
	SET RACE_URLS_CURRENT;
	RUN;
	QUIT;
	
%MEND ADD_CURRENT_DNAMES;

%ADD_CURRENT_DNAMES();

/******************************************************************************************\ 
Get polling numbers for each race
\*******************************************************************************************/

/* Now armed with a list of current races we're going to open each url and get the current polling numbers of each race.
We will remove those that return an error or malformed data. We will borrow some if not all 
the code from a previous MACRO FIND_RACE_RESULT that returned the final results of previous races*/

/* This function should be used within a MACRO SCANLOOP format */
/* Note: the DATASET must be in HTML format, usually derived from %GET_HTML */
/* where D_OUT_NAME is dataset output name */

%MACRO FIND_RACE_CURRENT(DATASET, D_OUT_NAME);
	proc sql;
	   create table &D_OUT_NAME as
	   select * from &DATASET 
	   where prxmatch('(<p>?)', record);
	run;
	quit;
	proc sql;
	   create table &D_OUT_NAME as
	   select * from &D_OUT_NAME 
	   where prxmatch('(<td?)', record);
	run;
	quit;

	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	filter = strip(record);
	filter = compress(filter);
	length = length(filter);
	/* it seems this pattern denotes the actual results, as can be seen on the webpage */
	patternID = prxparse('/--/');
	position =  prxmatch(patternID, filter);
	/* Pattern on website is usually 20 characters within -- contain the actual result */
	/* Also we restrict at 20 to remove any independents that may have run in the races as a three variable analysis causes
	issues with the code later on and does not add that much statistical benefit since our observations are large */
	/* we add +20 since --</td><td>--</td> is roughly 20 characters where -- is our marker */
	race_result = substr(filter, position+20, 20);
	RUN;
	QUIT;

	DATA &D_OUT_NAME (DROP = id record filter length patternID position);
	SET &D_OUT_NAME;
	race_result = tranwrd(race_result, "<","");
	race_result = tranwrd(race_result, ">","");
	race_result = tranwrd(race_result, "--","");
	/* removes all lower case letters */
	race_result = compress(race_result,'ABCD','l');
	/* we use '/' as the marker between the two results so we can seperate them into different columns */
	/* we change '/' to m for regex purposes */
	race_result = tranwrd(race_result, "/","m");
	/* remove white, trailing, and leading spaces*/
	race_result = compress(race_result);
	length_update = length(race_result);
	patternID_update = prxparse('/m/');
	position_update =  prxmatch(patternID_update, race_result);
	candidate1_result = substr(race_result, 1, position_update-1);
	candidate2_result = substr(race_result, position_update+1, length_update-position_update+1);

	/* Convert Char Variable into Numerical Variable! VERY IMPORTANT! Can cause great headaches later on when trying to do math */
	/*http://www.ciser.cornell.edu/faq/sas/char2num.shtml*/
	candidate1_actual_result = candidate1_result +0;
	candidate2_actual_result = candidate2_result + 0;
	RUN;
	QUIT;

	/*"Hey Kids Its Clean Up Time!"-Barney */
	DATA &D_OUT_NAME (DROP = race_result length_update patternID_update position_update candidate1_result candidate2_result);
	SET &D_OUT_NAME;
	RUN;
	QUIT;
	
	/* Rename back to original for code integrity purposes*/
	DATA &D_OUT_NAME (RENAME =(candidate1_actual_result = candidate1_result candidate2_actual_result = candidate2_result));
	SET &D_OUT_NAME;
	RUN;
	QUIT;

	/* Certain webpages produce a strange second row, perhaps the existence of uncaught trailing spaces? Regardless it does not affect the first row, but still delete
	any extra rows that may be in the dataset */
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	num = _N_;
	RUN;
	QUIT;
	
	/* Assign a serial number to each observation */
	/*https://kb.iu.edu/d/adxp*/
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	num = _N_;
	RUN;
	QUIT;

	/* if num =2 then there is a second row present, delete it */
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	IF num = 2 THEN DELETE;
	RUN;
	QUIT;

	DATA &D_OUT_NAME(DROP = num);
	SET &D_OUT_NAME;
	RUN;
	QUIT;

%MEND FIND_RACE_CURRENT;


/******************************************************************************************\ 
Iterative: %GET_HTML %FIND_RACE_CURRENT
\*******************************************************************************************/

/*Macro to SCAN through updated RACE__CURRENT_URLS and create race result dataset and create the poll dataset for each race */
%MACRO SCANLOOP_RACE_RESULT(SCANFILE,FIELD1,FIELD2);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* result_url */
	CALL SYMPUT('VAR2',&FIELD2); /*current_dname */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */
	%GET_HTML("&VAR1");
	%FIND_RACE_CURRENT(HTML, &VAR2);

	%END;
%MEND SCANLOOP_RACE_RESULT;
%SCANLOOP_RACE_RESULT(RACE_URLS_CURRENT,result_url,current_dname);


/******************************************************************************************\ 
Get the candidate names
\*******************************************************************************************/

/* However we have a huge issue! The position of the candidates is determined by who is currently ahead in the polls, thus the html is not static!
We need to find a way to determine who is actually the first candidate and who is the second candidate. One way to to this is to scrape for the
names themselves, place them in a dataset and then sequentially merge them back into the main dataset or, functionally a database, RACE_URLS_CURRENT */

%MACRO FIND_NAME_CURRENT(DATASET, D_OUT_NAME, NUM);
	/* D_OUT_NAME is NAME_TEMP in this MACRO since we will simply merge these name to RESULT_URLS_CURRENT */
	proc sql;
	   create table &D_OUT_NAME as
	   select * from &DATASET 
	   where prxmatch('(<p>?)', record);
	run;
	quit;
	proc sql;
	   create table &D_OUT_NAME as
	   select * from &D_OUT_NAME 
	   where prxmatch('(<th?)', record);
	run;
	quit;

	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	filter = strip(record);
	filter = compress(filter);
	length = length(filter);
	/* it seems this pattern denotes the actual results, as can be seen on the webpage */
	patternID = prxparse('/MoE/');
	position =  prxmatch(patternID, filter);
	/* Pattern on website is usually 20 characters within -- contain the actual result */
	/* Also we restrict at 20 to remove any independents that may have run in the races as a three variable analysis causes
	issues with the code later on and does not add that much statistical benefit since our observations are large */
	/* we add +20 since --</td><td>--</td> is roughly 20 characters where -- is our marker */
	race_result = substr(filter, position+8, 60);
	RUN;
	QUIT;

	DATA &D_OUT_NAME (DROP = id record filter length patternID position);
	SET &D_OUT_NAME;
	race_result = tranwrd(race_result, "<th>","");
	race_result = tranwrd(race_result, "<","");
	race_result = tranwrd(race_result, "th>","");

	/* we use '/' as the marker between the two results so we can seperate them into different columns */
	/* we change '/' to '>' for regex purposes */
	race_result = tranwrd(race_result, "/",">");
	/* remove white, trailing, and leading spaces*/
	race_result = compress(race_result);
	length_update = length(race_result);
	patternID_update = prxparse('/>/');
	position_update =  prxmatch(patternID_update, race_result);
	candidate1_result = substr(race_result, 1, position_update-1);
	/* change (D) to _D  and (R) to _R to prevent SAS from thinking we mean array if we call it later on */
	candidate1_result = tranwrd(candidate1_result, "(D)","_D");
	candidate1_result = tranwrd(candidate1_result, "(R)","_R");

	race_result2 = race_result;
	patternID_update = prxparse('/>/');
	position_update =  prxmatch(patternID_update, race_result2);
	race_result2 = substr(race_result2, position_update+1);
	position_update =  prxmatch(patternID_update, race_result2);
	candidate2_result = substr(race_result2, 1, position_update-1);
	/* change (D) to _D  and (R) to _R to prevent SAS from thinking we mean array if we call it later on */
	candidate2_result = tranwrd(candidate2_result, "(D)","_D");
	candidate2_result = tranwrd(candidate2_result, "(R)","_R");
	RUN;
	QUIT;

	/*"Hey Kids Its Clean Up Time!"-Barney */
	DATA &D_OUT_NAME (KEEP=candidate1_result candidate2_result);
	SET &D_OUT_NAME;
	RUN;
	QUIT;

	/* Certain webpages produce a strange second row, perhaps the existence of uncaught trailing spaces? Regardless it does not affect the first row, but still delete
	any extra rows that may be in the dataset */
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	num = _N_;
	RUN;
	QUIT;
	
	/* Assign a serial number to each observation */
	/*https://kb.iu.edu/d/adxp*/
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	num = _N_;
	RUN;
	QUIT;

	/* if num =2 then there is a second row present, delete it */
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	IF num = 2 THEN DELETE;
	RUN;
	QUIT;

	DATA &D_OUT_NAME(DROP = num);
	SET &D_OUT_NAME;
	RUN;
	QUIT;

	/* create a temporary name list that can be merged with race_urls_current */
	%IF &NUM = 1 %THEN %DO;
	DATA NAME_TEMP_LIST;
	SET &D_OUT_NAME;
	RUN;
	QUIT;
	%END;
	%ELSE %DO;
	DATA NAME_TEMP_LIST;
	SET NAME_TEMP_LIST &D_OUT_NAME;
	RUN;
	QUIT;
	%END;

%MEND FIND_NAME_CURRENT;

/******************************************************************************************\ 
Iterative: %GET_HTML %FIND_NAME_CURRENT
\*******************************************************************************************/

/*Macro to SCAN through updated RACE_CURRENT_URLS and create race result dataset and create the poll dataset for each race */
%MACRO SCANLOOP_RACE_CURRENT_NAME(SCANFILE,FIELD1,FIELD2);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* result_url */
	CALL SYMPUT('VAR2',&FIELD2); /*current_dname */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */
	%GET_HTML("&VAR1");
	%FIND_NAME_CURRENT(HTML, NAME_TEMP, &I);

	%END;
%MEND SCANLOOP_RACE_CURRENT_NAME;
%SCANLOOP_RACE_CURRENT_NAME(RACE_URLS_CURRENT,result_url,current_dname);

/* merge the name list with the RACE_URLS_CURRENT */
%MACRO ADD_NAMES_CURRENT();
	DATA RACE_URLS_CURRENT;
	MERGE RACE_URLS_CURRENT NAME_TEMP_LIST;
	RUN;
	QUIT;
%MEND ADD_NAMES_CURRENT;

%ADD_NAMES_CURRENT;

/******************************************************************************************\ 
Get candidate party affliations
\*******************************************************************************************/

/* We also want to add parties for graph styling purposes */

%MACRO INSERT_CURRENT_PARTIES();

	DATA RACE_URLS_CURRENT;
	SET RACE_URLS_CURRENT;
	patternID = prxparse('/_/');
	position = prxmatch(patternID, CANDIDATE1_RESULT);
	CANDIDATE1_PARTY = substr(CANDIDATE1_RESULT, position+1);

	patternID = prxparse('/_/');
	position = prxmatch(patternID, CANDIDATE2_RESULT);
	CANDIDATE2_PARTY = substr(CANDIDATE2_RESULT, position+1);
	RUN;
	QUIT;
	
	/* Clean up any malformed data */
	/* Assumes we've removed any three party races */
	DATA RACE_URLS_CURRENT(DROP = patternID position);
	SET RACE_URLS_CURRENT;
	IF CANDIDATE1_PARTY = 'R' THEN CANDIDATE2_PARTY ='D';
	IF CANDIDATE1_PARTY = 'D' THEN CANDIDATE2_PARTY ='R';
	IF CANDIDATE2_PARTY = 'R' THEN CANDIDATE1_PARTY ='D';
	IF CANDIDATE2_PARTY = 'D' THEN CANDIDATE1_PARTY ='R';

	IF length(CANDIDATE1_PARTY) > 1 THEN CANDIDATE1_PARTY ='D';
	IF length(CANDIDATE1_PARTY) < 1 THEN CANDIDATE1_PARTY ='D';
	IF length(CANDIDATE2_PARTY) > 1 THEN CANDIDATE2_PARTY ='R';
	IF length(CANDIDATE2_PARTY) < 1 THEN CANDIDATE2_PARTY ='R';

	IF CANDIDATE1_PARTY = 'R' THEN CANDIDATE2_PARTY ='D';
	IF CANDIDATE1_PARTY = 'D' THEN CANDIDATE2_PARTY ='R';
	IF CANDIDATE2_PARTY = 'R' THEN CANDIDATE1_PARTY ='D';
	IF CANDIDATE2_PARTY = 'D' THEN CANDIDATE1_PARTY ='R';

	IF CANDIDATE1_PARTY = 'R' THEN CANDIDATE1_PARTY ='Red';
	IF CANDIDATE1_PARTY = 'D' THEN CANDIDATE1_PARTY ='Blue';
	IF CANDIDATE2_PARTY = 'R' THEN CANDIDATE2_PARTY ='Red';
	IF CANDIDATE2_PARTY = 'D' THEN CANDIDATE2_PARTY ='Blue';

	RUN;
	QUIT;

%MEND INSERT_CURRENT_PARTIES;
%INSERT_CURRENT_PARTIES;

/******************************************************************************************\ 
Bootstrap sampling:
1. Pick N=1000 experiments to run
2. For each experiment randomly select a value from the distribution of polling errors.
3. Assume the error for Candidate A is the selected value. Assume the error for Candidate A is the opposite of the selected value.
4. Calculate the winner in this simulation
5. Repeat simulation N=1000 times
6. Determine the percentage of simulations for wins for Candidate A and Candidate B

\*******************************************************************************************/

%MACRO BOOTSTRAP(DATASET_CURRENT_POLL, CANDIDATE1, CANDIDATE2, NUM_EXPERIMENTS, D_OUT_NAME);

	/* Randomly select an Error value from ERROR_RESULTS_CONCAT *./
	/* We assume ERROR_RESULTS_CONCAT represent a reasonable error distribution for the current poll data */
	PROC SURVEYSELECT DATA = ERROR_RESULTS_CONCAT OUT = SAMPLE(KEEP=ERROR) METHOD =SRS
	  SAMPSIZE =&NUM_EXPERIMENTS SEED=1234567 noprint;
	RUN;
	QUIT;
	
	/* Repeat current poll results to n observations in order to merge with n randomly selected error values later */
	DATA &D_OUT_NAME (drop=i);
	  SET &DATASET_CURRENT_POLL;
	  DO i = 1 TO &NUM_EXPERIMENTS;
	    OUTPUT;
	  END;
	RUN;
	QUIT;
	
	/* Merge N randomly selected error values and current poll results */
	DATA &D_OUT_NAME;
	MERGE SAMPLE &D_OUT_NAME;
	RUN;
	QUIT;

	/* Calculate the predicted poll numbers using the randomly selected error value from above for each Candidate */
	DATA &D_OUT_NAME;
	SET &D_OUT_NAME;
	&CANDIDATE1 = CANDIDATE1_result - ERROR;
	&CANDIDATE2 = CANDIDATE2_result + ERROR;
	RUN;
	QUIT;

	/* Calculate Proportion of Wins for each Candidate */

%MEND BOOTSTRAP;


/******************************************************************************************\ 
Assign an index 1 if Candidate has wom, transpose dataset and then sum total wins through
row addition. Create new database win_loss_list that can easily be MERGE'd with current database
\*******************************************************************************************/

%MACRO WIN_LOSS (PREDICT_DNAME, CANDIDATE1, CANDIDATE2, NUM, I);
	DATA &PREDICT_DNAME;
	SET &PREDICT_DNAME;
	IF &CANDIDATE1 > &CANDIDATE2 THEN CANDIDATE1_SIM = 1;
	IF &CANDIDATE1 < &CANDIDATE2 THEN CANDIDATE1_SIM = 0;

	IF &CANDIDATE2 > &CANDIDATE1 THEN CANDIDATE2_SIM = 1;
	IF &CANDIDATE2 < &CANDIDATE1 THEN CANDIDATE2_SIM = 0;
	RUN;
	QUIT;

	DATA WIN_LOSS_TEMP(KEEP=CANDIDATE1_SIM CANDIDATE2_SIM);
	SET &PREDICT_DNAME;
	RUN;

	PROC transpose data=WIN_LOSS_TEMP
               	   out=WIN_LOSS_TEMP;
	RUN;
	QUIT;

	DATA WIN_LOSS_TEMP;
	SET WIN_LOSS_TEMP;
	TOTAL = SUM(OF COL1-COL&NUM);
	RUN;
	QUIT;

	DATA WIN_LOSS_TEMP(KEEP= _NAME_ TOTAL);
	SET WIN_LOSS_TEMP;
	RUN;
	QUIT;

	PROC transpose data=WIN_LOSS_TEMP
               	   out=WIN_LOSS_TEMP;
	RUN;
	QUIT;

	DATA WIN_LOSS_TEMP(DROP= _NAME_);
	SET WIN_LOSS_TEMP;
	RUN;
	QUIT;

	
	%IF &I = 1 %THEN %DO;

	DATA WIN_LOSS_LIST;
	SET WIN_LOSS_TEMP;
	RUN;
	QUIT;

	%END;
	%ELSE %DO;

	DATA WIN_LOSS_LIST;
	SET WIN_LOSS_LIST WIN_LOSS_TEMP;
	RUN;
	QUIT;

	%END;


%MEND WIN_LOSS;
/* %WIN_LOSS(mead_vs_gosar_predict, Mead_R, Gosar_D, 100);

/******************************************************************************************\ 
Iterative: %Bootstrap %Win_Loss

\*******************************************************************************************/

/*Macro to SCAN through updated RACE__CURRENT_URLS and create race result dataset and create the poll dataset for each race */
%MACRO SCANLOOP_CURRENT_BOOTSTRAP(SCANFILE,FIELD1,FIELD2,FIELD3,FIELD4,FIELD5,FIELD6,FIELD7);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* result_url */
	CALL SYMPUT('VAR2',&FIELD2); /* current_dname */
	CALL SYMPUT('VAR3',&FIELD3); /* candidate1_result */
	CALL SYMPUT('VAR4',&FIELD4); /* candidate2_result */
	CALL SYMPUT('VAR5',&FIELD5); /* predict_dname */
	CALL SYMPUT('VAR6',&FIELD6); /* candidate1_party */
	CALL SYMPUT('VAR7',&FIELD7); /* candidate2_party */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */

	/* NOTE: THE TWO NUMBERS IN BOOTSTRAP AND WIN_LOSS MUST MATCH! */
	%BOOTSTRAP(&VAR2, &VAR3, &VAR4, 10000, &VAR5);
	%WIN_LOSS(&VAR5, &VAR3, &VAR4, 10000, &I);
	
	%END;
%MEND SCANLOOP_CURRENT_BOOTSTRAP;
%SCANLOOP_CURRENT_BOOTSTRAP(RACE_URLS_CURRENT,result_url,current_dname,candidate1_result, candidate2_result, predict_dname, candidate1_party, candidate2_party);


/******************************************************************************************\ 
Merge win loss database with candidate database
\*******************************************************************************************/
%MACRO MERGE_WIN_LOSS();
	DATA RACE_URLS_CURRENT;
	MERGE RACE_URLS_CURRENT WIN_LOSS_LIST;
	RUN;
	QUIT;
%MEND MERGE_WIN_LOSS;
%MERGE_WIN_LOSS;

/******************************************************************************************\ 
Calculate the probability of win based on simulations in database
\*******************************************************************************************/
%MACRO CALC_PROB_WIN();
	/* Delete malformed data or else will rpoduce error upon dividing by zero below */
	DATA RACE_URLS_CURRENT;
	SET RACE_URLS_CURRENT;
	IF CANDIDATE1_SIM ='.' THEN DELETE;
	RUN;
	QUIT;

	DATA RACE_URLS_CURRENT;
	SET RACE_URLS_CURRENT;
	CANDIDATE1_PROB_WIN = CANDIDATE1_SIM / (CANDIDATE1_SIM + CANDIDATE2_SIM);
	CANDIDATE2_PROB_WIN = CANDIDATE2_SIM / (CANDIDATE1_SIM + CANDIDATE2_SIM);
	RUN;
	QUIT;
%MEND CALC_PROB_WIN;
%CALC_PROB_WIN;

/******************************************************************************************\ 
Graph probability distributions
\*******************************************************************************************/
%MACRO GRAPH_PREDICTIONS(DATASET, Candidate1, Candidate2, CANDIDATE1_PARTY, CANDIDATE2_PARTY, CANDIDATE1_PROB_WIN, CANDIDATE2_PROB_WIN);

	proc sgplot data=&DATASET;
	histogram &CANDIDATE1 / fillattrs=(color="light strong &CANDIDATE1_PARTY") transparency=0.3 binstart=0 binwidth=1;
	density &CANDIDATE1 / lineattrs=(color="light strong &CANDIDATE1_PARTY" thickness=3);
	histogram &CANDIDATE2 / fillattrs=(color="light strong &CANDIDATE2_PARTY") transparency=0.3 binstart=0 binwidth=1;
	density &CANDIDATE2 / lineattrs=(color="light strong &CANDIDATE2_PARTY" thickness=3);
	keylegend / location=inside position=topright noborder across=2;
	yaxis grid;
	xaxis grid values=(20 to 70 by 5) label="Simulated Results";
	TITLE "Boostrap simulated race outcomes for &Candidate1 and &Candidate2";
	INSET "Out of 10,000 simulations:"
	  "&CANDIDATE1 wins &CANDIDATE1_PROB_WIN of the time." 
	  "&CANDIDATE2 wins &CANDIDATE2_PROB_WIN of the time." / POSITION=TOPLEFT ;
	run;
	quit;

%MEND GRAPH_PREDICTIONS;

/******************************************************************************************\ 
Iterative: %GRAPH_PREDICTIONS
\*******************************************************************************************/
/*Macro to SCAN through updated RACE__CURRENT_URLS and create race result dataset and create the poll dataset for each race */
%MACRO SCANLOOP_CURRENT_GRAPH(SCANFILE,FIELD1,FIELD2,FIELD3,FIELD4,FIELD5,FIELD6,FIELD7,FIELD8,FIELD9);
	/* First obtain the number of */
	/* records in DATALOG */
	DATA _NULL_;
	IF 0 THEN SET &SCANFILE NOBS=X;
	CALL SYMPUT('RECCOUNT',X); 
	STOP; 
	RUN;
	/* loop from one to number of */
	/* records */
	%DO I=1 %TO &RECCOUNT;
	/* Advance to the Ith record */
	DATA _NULL_;
	SET &SCANFILE (FIRSTOBS=&I);
	/* store the variables */
	/* of interest in */
	/* macro variables */
	CALL SYMPUT('VAR1',&FIELD1); /* result_url */
	CALL SYMPUT('VAR2',&FIELD2); /* current_dname */
	CALL SYMPUT('VAR3',&FIELD3); /* candidate1_result */
	CALL SYMPUT('VAR4',&FIELD4); /* candidate2_result */
	CALL SYMPUT('VAR5',&FIELD5); /* predict_dname */
	CALL SYMPUT('VAR6',&FIELD6); /* candidate1_party */
	CALL SYMPUT('VAR7',&FIELD7); /* candidate2_party */
	CALL SYMPUT('VAR8',&FIELD8); /* candidate1_prob_win */
	CALL SYMPUT('VAR9',&FIELD9); /* candidate2_prob_win */
	STOP;
	RUN;
	/*%CREATE_DATASET(&VAR1);*/
	/* now perform the tasks that */
	/* wish repeated for each */
	/* observation */

	/* NOTE: THE TWO NUMBERS IN BOOTSTRAP AND WIN_LOSS MUST MATCH! */
	%GRAPH_PREDICTIONS(&VAR5, &VAR3, &VAR4, &VAR6, &VAR7, &VAR8, &VAR9);
	
	%END;
%MEND SCANLOOP_CURRENT_GRAPH;
%SCANLOOP_CURRENT_GRAPH(RACE_URLS_CURRENT,result_url,current_dname,candidate1_result, candidate2_result, predict_dname, candidate1_party, candidate2_party, candidate1_prob_win, candidate2_prob_win);