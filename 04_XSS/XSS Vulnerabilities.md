In this report, the OWASP Juice Shop webapp is used to showcase the exploitation of two XSS vulnerabilities.

## Tools
- OWASP Juice Shop (running on Heroku) v19.1.1
- BURP Suite Proxy v2025.12.3

## Challenge #1. Reflected XSS

>Description: Perform a _reflected_ XSS attack with `<iframe src="javascript:alert('xss')>`.

## Preliminary steps

1. **User Registration/Login** 
	Navigate to **Account → Login → Register** .
	Create account and log in to enable basket/checkout functionality. 
 
 2. **Add Products to Basket** 
    Go to **/#/search** (main product listing). 
    Click **Add to basket** on 2-3 products (e.g., "Banana Juice", "Apple Juice").
 
 3. **Complete Checkout Process** 
	 - Click **Checkout** from basket 
	 - **Delivery**: Add *fake address* → Continue 
	 - **Payment**: Use `1234567890123456` (fake CC) → Continue 
	 - **Confirm**: Click **Place your order and pay** 
	 - **Result**: Order confirmation + **tracking ID** generated

## Discovery

- After completing checkout, click the **truck icon** (Track Order) in **Account → Order History**. 
- ![[xss_1.png]]
-   Notice the **order number** appears both: 
	- **In URL**: `track-result?id=9d7f-46b5fbb2c8bf75e1` 
	- **In page content**: "Search Results - 9d7f-46b5fbb2c8bf75e1"

- User input (order ID) rendered in **both URL parameter AND HTML page content** might be an indicator of an XSS vulnerability if there is no input sanitization or output encoding.

**The actual exploitation attempt procedure is outlined in the following:**

- Let's try to replace order ID with payload: `<iframe src="javascript:alert('xss')">` and reload the page.
- **Result**: JavaScript executes immediately as part of the DOM → **vulnerability confirmed**.
  ![[xss_2.png]]

## Under the hood analysis

- When sniffing network traffic with Burp Suite or browser DevTools, we observe that the client makes an HTTP `GET` request to `/rest/track-order/<iframe%20src%3D"javascript:alert(%60xss%60)">` and the server responds with a JSON object that contains `"orderId":"<iframe src=\"javascript:alert('xss')\">"` .
  ![[xss_3.png]]
- **Critical Observation**: The server receives the malicious payload and **reflects it unchanged** within the JSON response's `orderId` field.
- To understand how this request is processed internally, we can look more closely at the OWASP Juice Shop source code.![[xss_4.png]]
	1. we can notice at this line `this.orderId = this.route.snapshot.queryParams.id`: When the page loads, it grabs the `id` value directly from the `URL` query string.
	2. `this.trackOrderService.find(this.orderId).subscribe(...)`: It sends the `orderId` to the backend. The application then waits for the server to reply with the order details . Inside the subscription, it maps the data from the server (`results.data[0]`) to the `this.results` object so it can be displayed in the HTML template.
	3. The reason the vulnerability is present is this line `this.results.orderNo = this.sanitizer.bypassSecurityTrustHtml(``<code>${results.data[0].orderId}</code>`)`
	- Angular has built-in Cross-Site Scripting (XSS) protection. By default, it "sanitizes" any values you bind to the DOM, stripping out dangerous script tags.
	- The method `bypassSecurityTrustHtml` tells Angular: "I trust this value completely. Do not check it for dangerous scripts; just render it exactly as is."
	- The value being trusted is `${results.data[0].orderId}`. If the application reflects the user's input back to them in the `orderId` field without validation, an attacker can inject malicious HTML or JavaScript.


## Challenge #2. DOM XSS

>Description: Perform a _DOM_ XSS attack with `<iframe src="javascript:alert('xss')">`.


No preliminary steps are required for this challenge.

## Discovery

1. Navigate to the search bar and enter a legitimate term like **"apple"**.
2. Notice the `searchquery` parameter appears in **both**: 
	-  **URL**: `/#/search?q=apple` 
	- **Page content**: Search results highlight "apple"
	- ![[xss_5.png]]
3. User input rendered in **URL parameter AND page content** suggests potential XSS risk, **especially if no client-side sanitization** is implemented.

**The actual exploitation attempt procedure is outlined in the following:**

- Replace search term with payload: `<iframe src="javascript:alert('xss')">`
- **Result**: JavaScript executes immediately → **DOM XSS confirmed**. ![[xss_6.png]]

## Under the Hood Analysis:

- When sniffing network traffic with Burp Suite or browser DevTools, we observe that:![[xss_7.png]]
	1. **No payload in HTTP request** - Burp shows clean request 
	2. **No payload in server response** - Server unaware of attack
- **Execution happens entirely client-side**.
- Looking more closely at the OWASP Juice Shop source code we can notice:
- ![[xss_8.png]]
	1. In the `filterTable()` function, Angular reads `this.route.snapshot.queryParams.q` from hash fragment.
	2. And then it processes: `this.searchValue = this.sanitizer.bypassSecurityTrustHtml(queryParam)`
	3. And in the **HTML template file** we can see it renders `[innerHTML]="searchValue"` → **payload executes client-side**.
	   ![[xss_9.png]]

## Conclusions:

* **Reflected XSS**:  
	* Attacker Input → HTTP Request → Server Reflects → Malicious HTML Response → Browser Executes.
	* The server receives malicious input, echoes it back **without sanitization** in the HTTP response, creating an executable payload.
* **DOM XSS (Search Bar)**:
	* Attacker Input → URL (#/search?q=\<payload>) → Client-side Processing → DOM Execution.
	* The attack executes **entirely client-side**—server never sees the payload, browser processes hash fragment malicious script directly.
* Corporate proxies and browsers can easily block reflected XSS by checking whether the string sent contains a script and looking to see if the exact same string comes back. They can stop the hack as it is happening.
* The same protection does not happen for DOM-based XSS. The attack happens entirely in the browser, which prevents corporate proxies from stopping it. It is a lot trickier for a browser to stop this type of attack than it is for one to stop reflected XSS.