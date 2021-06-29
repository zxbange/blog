---
layout: post
title:  "Oracle/MySQL函数对比"
subtitle: "对比常用函数区别和替换"
date:   2021-06-25 16:31:29 +0900
categories: MySQL, Oracle
author:  "张鑫"
tags:
  - DBFunctions
---
# 数据库函数

### Oracle MySQL函数链接
* Oracle函数 10g：[]()https://docs.oracle.com/cd/B19306_01/server.102/b14200/functions001.htm
* Oracle函数 11g：[]()https://docs.oracle.com/cd/E11882_01/server.112/e41084/functions001.htm#SQLRF51173
* Oracle函数 12c：[]()https://docs.oracle.com/database/121/SQLRF/functions.htm#SQLRF006
* Oracle函数 19c：[]()https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Functions.html#GUID-D079EFD3-C683-441F-977E-2C9503089982
* MySQL 5.7函数： []()https://dev.mysql.com/doc/refman/5.7/en/built-in-function-reference.html
* 转换辅助参考工具：[]()http://www.sqlines.com/online

### Oracle/MySQL常用函数对比

| |Oracle| |MySQL|
| --- | --- | ------ | --- |
|1|ABS(num)|Get the absolute value|ABS(num)|
|2|ACOS(num)|Get the arc cosine|ACOS(num)|
|3|ADD_MONTHS(date, num)|Add num months to date|TIMESTAMPADD(MONTH, num, date)|
|4|ASCII(str)|Get ASCII code of left-most char|ASCII(str)|
|5|ASCIISTR(string)|Get ASCII code version of string||
|6|ASIN(num)|Get the arcsine|ASIN(num)|
|7|ATAN(num)|Get the arc tangent|ATAN(num)|
|8|ATAN2(x, y)|Get the arc tangent of x and y|ATAN2(x, y)|
|9|BIN_TO_NUM(bit1, bit2, …)|Convert bit vector to number||
|10|BITAND(exp1, exp2)|Perform bitwise AND|(exp1 & exp2)|
|11|CEIL(num)|Get the smallest following integer|CEIL(num)|
|12|CHR(num)|Get character from ASCII code|CHAR(num USING ASCII)|
|13|COALESCE(exp1, exp2, …)|Return first non-NULL expression|COALESCE(exp1, exp2, …)|
|14|CONCAT(char1, char2)|String concatenation|CONCAT(char1, char2)|
|15|CONVERT(string, charset)|Convert string to charset|CONVERT(string USING charset)|
|16|COS(num)|Get the cosine|COS(num)|
|17|COSH(num)|Get hyperbolic cosine|(EXP(num) + EXP(-num)) / 2|
|18|CURRENT_DATE|Get the current date|NOW()|
|19|CURRENT_TIMESTAMP|Get the current date and time|NOW()|
|20|DECODE(exp, when, then, …)|Evaluate conditions|CASE expression|
|21|EXP(n)|Raise e to the nth power|EXP(n)|
|22|EXTRACT(YEAR FROM date)|Extract year from date|YEAR(date)|
|23|EXTRACT(MONTH FROM date)|Extract month from date|MONTH(date)|
|24|EXTRACT(DAY FROM date)|Extract day from date|DAY(date)|
|25|EXTRACT(HOUR FROM time)|Extract hour from time|HOUR(time)|
|26|EXTRACT(MINUTE FROM time)|Extract minute from time|MINUTE(time)|
|27|EXTRACT(SECOND FROM time)|Extract second from time|SECOND(time)|
|28|FLOOR(num)|Get the largest preceding integer|FLOOR(num)|
|29|GREATEST(exp, exp2, …)|Get the maximum value in a set|GREATEST(exp, exp2, …)|
|30|INITCAP(string)|Capitalize words|User-defined function|
|31|INSTR(str, substr)|Get position of substring|INSTR(str, substr)|
||INSTR(str, substr, pos)||LOCATE(str, substr, pos)|
||INSTR(str, substr, pos, num)||User-defined function|
|32|LEAST(exp, exp2, …)|Get the minimum value in a set|LEAST(exp, exp2, …)|
|33|LENGTH(string)|Get length of string in chars|CHAR_LENGTH(string)|
|34|LENGTHB(string)|Get length of string in bytes|LENGTH(string) |
|35|LN(num)|Get natural logarithm of num|LN(num)|
|36|LOCALTIMESTAMP|Get the current date and time|LOCALTIMESTAMP|
||LOCALTIMESTAMP([prec])||LOCALTIMESTAMP()|
|37|LOG(num1, num2)|Get logarithm, base num1, of num2|LOG(num1, num2)|
|38|LOWER(string)|Lowercase string|LOWER(string)|
|39|LPAD(string, len)|Pad the left-side of string|LPAD(string, len, ' ')|
||LPAD(string, len, pad)||LPAD(string, len, pad)|
|40|LTRIM(string)|Remove leading spaces|LTRIM(string)|
||LTRIM(string, set)|Remove leading chars|TRIM(LEADING set FROM string)|
|41|MONTHS_BETWEEN(date1, date2)|Get number of months between|User-defined function|
|||date1 and date2||
|42|MOD(dividend, divisor)|Get remainder|MOD(dividend, divisor)|
|43|NEXT_DAY|Get the next date by day name|NEXT_DAY user-defined function|
|44|NULLIF(exp1, exp2)|Return NULL if exp1 = exp2|NULLIF(exp1, exp2)|
|45|NVL(exp, replacement)|Replace NULL with the specified value|IFNULL(exp, replacement)|
|46|NVL2(exp1, exp2, exp3)|Return exp2 if exp1 is not NULL,otherwise exp3|CASE expression|
|47|POWER(value, n)|Raise value to the nth power|POWER(value, n)|
|48|REMAINDER(n1, n2)|Get remainder|(n1 - n2*ROUND(n1/n2))|
|49|ROUND(num, integer)|Get rounded value|ROUND(num, integer)|
|50|RPAD(string, len)|Pad the right-side of string|RPAD(string, len, ' ')|
||RPAD(string, len, pad)||RPAD(string, len, pad)|
|51|RTRIM(string)|Remove trailing spaces|RTRIM(string)|
||RTRIM(string, set)|Remove trailing chars|TRIM(TRAILING set FROM string)|
|52|SIGN(exp)|Get sign of exp|SIGN(exp)|
|53|SIN(num)|Get sine|SIN(num)|
|54|SINH(num)|Get hyperbolic sine|(EXP(num) - EXP(-num)) / 2|
|55|SOUNDEX(string)|Get 4-character sound code|SOUNDEX(string)|
|56|SQRT(num)|Get square root|SQRT(num)|
|57|SUBSTR(string, pos, len)|Get a substring of string|SUBSTR(string, pos, len)|
|58|SYS_GUID()|Get GUID, 32 characters without dashes|REPLACE(UUID(), '-', '')|
|59|SYSDATE|Get current date and time|SYSDATE()|
|60|SYSTIMESTAMP|Get current timestamp|CURRENT_TIMESTAMP|
|61|TAN(num)|Get tangent|TAN(num)|
|62|TANH(num)|Get hyperbolic tangent|(EXP(2*num) - 1)/(EXP(2*num) + 1)|
|63|TO_CHAR(datetime, format)|Convert datetime to string|DATE_FORMAT(datetime, format) |
||TO_CHAR(number, format)|Convert number to string|FORMAT(number, decimal_digits) |
|64|TO_DATE(string, format)|Convert string to datetime|STR_TO_DATE(string, format) |
|65|TO_LOB(exp)|Convert to LOB||
|66|TO_NCHAR(exp)|Convert to NCHAR||
|67|TO_NCLOB(exp)|Convert to NCLOB||
|68|TO_NUMBER(exp)|Convert to NUMBER||
|69|TO_SINGLE_BYTE(exp)|Convert to single-byte character||
|70|TO_TIMESTAMP(exp)|Convert to TIMESTAMP||
|71|TRANSLATE(string, from, to)|Replace characters|User-defined function|
|72|TRIM([type trim FROM] string)|Remove characters|TRIM([type trim FROM] string)|
|73|TRUNC(num)|Truncate num|TRUNCATE(num, 0)|
||TRUNC(num, num2)||TRUNCATE(num, num2)|
|74|TRUNC(datetime)|Truncate datetime|DATE(datetime), DATE_FORMAT|
|75|UNISTR(string)|Convert Unicode code points to chars|CHAR(string USING UCS2) |
|76|UPPER(string)|Uppercase string|UPPER(string)|
|77|USER|Get the current user|USER()|
|78|USERENV('parameter')|Get the current session information||
|79|VSIZE(exp)|Get the size of exp in bytes||
|80|XMLAGG(exp)|Get a aggregated XML document||
|81|XMLCAST(exp AS datatype)|Convert exp to datatype||
|82|XMLCDATA(exp)|Generate a CDATA section||
|83|XMLCOMMENT(exp)|Generate an XML comment||
|84|XMLCONCAT(exp, exp2, …)|Concatenate XML expressions||
|85|XMLDIFF(doc, doc2)|Compare two XML documents||
|86|XMLELEMENT(NAME element)|Get an XQuery element node||
|87|XMLFOREST(exp, exp2, …)|Get a forest of XML expressions||
|88|XMLISVALID(exp)|Check XML exp||
|89|XMLPARSE(DOCUMENT exp)|Parse XML document||
|90|XMLPATCH(doc, doc2)|Patch XML document||
|91|XMLPI(NAME identifier)|Get XML processing instruction||
|92|XMLROOT(exp, VERSION exp2)|Create a new XML value||
|93|XMLSEQUENCE(exp)|Get a varray of the top-level nodes||
|94|XMLSERIALIZE(CONTENT exp AS datatype)|Get a serialized XML value||
|95|XMLTRANSFORM(instance, exp)|Transform XML document||

### 字段对比

||Oracle||MySQL|
| --- | --- | ------ | --- |
|1|BFILE|Pointer to binary file, ⇐ 4G|VARCHAR(255)||
|2|BINARY_FLOAT|32-bit floating-point number|FLOAT||
|3|BINARY_DOUBLE|64-bit floating-point number|DOUBLE||
|4|BLOB|Binary large object, ⇐ 4G|LONGBLOB||
|5|CHAR(n), CHARACTER(n)|Fixed-length string, 1 ⇐ n ⇐ 255|CHAR(n), CHARACTER(n)||
|6|CHAR(n), CHARACTER(n)|Fixed-length string, 256 ⇐ n ⇐ 2000|VARCHAR(n)||
|7|CLOB|Character large object, ⇐ 4G|LONGTEXT||
|8|DATE|Date and time|DATETIME||
|9|DECIMAL(p,s), DEC(p,s)|Fixed-point number|DECIMAL(p,s), DEC(p,s)||
|10|DOUBLE PRECISION|Floating-point number|DOUBLE PRECISION||
|11|FLOAT(p)|Floating-point number|DOUBLE||
|12|INTEGER, INT|38 digits integer|INT|DECIMAL(38)|
|13|INTERVAL YEAR(p) TO MONTH|Date interval|VARCHAR(30)||
|14|INTERVAL DAY(p) TO SECOND(s)|Day and time interval|VARCHAR(30)||
|15|LONG|Character data, ⇐ 2G|LONGTEXT||
|16|LONG RAW|Binary data, ⇐ 2G|LONGBLOB||
|17|NCHAR(n)|Fixed-length UTF-8 string, 1 ⇐ n ⇐ 255|NCHAR(n)||
|18|NCHAR(n)|Fixed-length UTF-8 string, 256 ⇐ n ⇐ 2000|NVARCHAR(n)||
|19|NCHAR VARYING(n)|Varying-length UTF-8 string, 1 ⇐ n ⇐ 4000|NCHAR VARYING(n)||
|20|NCLOB|Variable-length Unicode string, ⇐ 4G|NVARCHAR(max)||
|21|NUMBER(p,0), NUMBER(p)|8-bit integer, 1 <= p < 3|TINYINT|(0 to 255)|
|||16-bit integer, 3 <= p < 5|SMALLINT||
|||32-bit integer, 5 <= p < 9|INT||
|||64-bit integer, 9 <= p < 19|BIGINT||
|||Fixed-point number, 19 <= p <= 38|DECIMAL(p)||
|22|NUMBER(p,s)|Fixed-point number, s > 0|DECIMAL(p,s)||
|23|NUMBER, NUMBER(*)|Floating-point number|DOUBLE||
|24|NUMERIC(p,s)|Fixed-point number|NUMERIC(p,s)||
|25|NVARCHAR2(n)|Variable-length UTF-8 string, 1 ⇐ n ⇐ 4000|NVARCHAR(n)||
|26|RAW(n)|Variable-length binary string, 1 ⇐ n ⇐ 2000|VARBINARY(n)||
|27|REAL|Floating-point number|DOUBLE||
|28|ROWID|Physical row address|CHAR(10)||
|29|SMALLINT|38 digits integer|DECIMAL(38)||
|30|TIMESTAMP(p)|Date and time with fraction|DATETIME(p)||
|31|TIMESTAMP(p) WITH TIME ZONE|Date and time with fraction and time zone|DATETIME(p) ||
|32|UROWID(n)|Logical row addresses, 1 ⇐ n ⇐ 4000|VARCHAR(n)||
|33|VARCHAR(n)|Variable-length string, 1 ⇐ n ⇐ 4000|VARCHAR(n)||
|34|VARCHAR2(n)|Variable-length string, 1 ⇐ n ⇐ 4000|VARCHAR(n)||
|35|XMLTYPE|XML data|LONGTEXT||
