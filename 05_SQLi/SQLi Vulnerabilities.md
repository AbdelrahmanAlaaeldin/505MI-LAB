In this report, the OWASP Juice Shop webapp is used to showcase the exploitation of two XSS vulnerabilities.

## Tools
- OWASP Juice Shop (running on Heroku) v19.1.1
- BURP Suite Proxy v2025.12.3

## Challenge #1. Database Schema

>Description: Exfiltrate the entire DB schema definition via SQL Injection.

## Discovery:

- As mentioned in this [Article](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05-Testing_for_SQL_Injection) on owasp.org:
  ![[SQLi_1.png]]
- So, we intercept the traffic in burp suite and look for the request related to our input field.
  ![[SQLi_2.png]]
- We send it to the repeater, and we enter this following payload `/rest/product/search?q=';` to try to provoke an error response.
  ![[SQLi_3.png]]
- This response provoked error exposing the sql query and what database engine it is using:
	1. SQLITE (database engine)
	2. `SELECT * FROM Products WHERE ((name LIKE ‘%payload%’ OR description LIKE ‘%payload%’) AND deletedAt IS NULL) ORDER BY name` (sql query)

- We notice that normally the response for empty search query is this
  ![[SQLi_4.png]]
- So we can see that there are 9 columns, but we can also test with payload `')) order by 10 -- ` for example:
  ![[SQLi_5.png]]
- We get an error that number of columns should be in range 1-9.


The actual exploitation attempt procedure is outlined in the following:

- As mentioned in www.sqlite.org: 
  ![[SQLi_6.png]]
- ![[SQLi_7.png]]
- So in order to exfiltrate the entire database we could call the 'sql' column in the 'sqlite_schema' table.
- Payload for schema exfiltration:
  `anything'))%20union%20all%20select%20sql,2,3,4,5,6,7,8,9%20from%20sqlite_schema%20--%20`
- ![[SQLi_8.png]]
- ![[SQLi_9.png]]
- **Challenge complete**.

## Challenge #2. Login Admin

>Description: Log in with the administrator's user account.


## Discovery:

- As mentioned in this [Article](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/05-Testing_for_SQL_Injection) on owasp.org:
  ![[SQLi_10.png]]
- So we can try to provoke an error response like we did in the previous challenge.
- Go to login page and try to login and capture this request in Burpsuite:
- ![[SQLi_11.png]]
- To provoke the error response, we send the request to the repeater and give payload:
  ![[SQLi_12.png]]
- As per the SQL query in the response error we can create the payload to break the authentication and login as an admin.
- We first have to obtain the email address of admin and then we can craft a payload to allow us to bypass authentication by logging in just by email address and without password.
- Since we were able to exfiltrate the entire database in the first challenge we can show the contents of the table that contains all of the users credentials.
- ![[SQLi_8.png]]
- **Column names of Users:**
	id
	username  
	email  
	password  
	role  
	deluxeToken
	lastLoginIP  
	profileImage  
	totpSecret  
	isActive  
	createdAt  
	updatedAt  
	deletedAt
- **Payload for User Credentials:** `anything'))%20union%20all%20select%201,email,username,password,5,6,7,8,9%20from%20Users%20--%20`
- And we obtain all users email addresses and password hashes.
- ![[SQLi_13.png]]
- Now that we have the admin's email address, we can inject the payload `admin@juice-sh.op' --` in the email address field and write "anything" as password![[SQLi_14.png]]
- **Challenge Completed**![[SQLi_15.png]]