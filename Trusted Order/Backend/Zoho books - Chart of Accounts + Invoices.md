How to authenticate zoho account to prepare for our project:

- Get Zoho token

1. Open this in browser (replace values + correct DC):
Format: https://accounts.zoho.<dc>/oauth/v2/auth?response_type=code&client_id=YOUR_CLIENT_ID&scope=ZohoBooks.fullaccess.all&redirect_uri=YOUR_REGISTERED_REDIRECT_URI&access_type=offline&prompt=consent

Example:  https://accounts.zoho.com/oauth/v2/auth?response_type=code&client_id=1000.ICI12JI6DWFYZELUTGYJV2JKH9YXGI&scope=ZohoBooks.fullaccess.all&redirect_uri=https://webhook.site/8f49230e-d80d-444b-b541-a974b18f6f20&access_type=offline&prompt=consent

 - <dc> = com / sa / eu / in / com.au / ca
 - After consent, Zoho redirects to your redirect_uri with:
 ?code=...  â this is the grant code (valid ~2 minutes).

PS: you can use webhook live website: webhook.site

 2. Exchange that code for tokens:

 curl -X POST "https://accounts.zoho.<dc>/oauth/v2/token" \
   -d "client_id=YOUR_CLIENT_ID" \
   -d "client_secret=YOUR_CLIENT_SECRET" \
   -d "grant_type=authorization_code" \
   -d "redirect_uri=YOUR_REGISTERED_REDIRECT_URI" \
   -d "code=THE_CODE_FROM_REDIRECT"

 3. In JSON response, copy:

 - refresh_token  â save as ZOHO_REFRESH_TOKEN
 - access_token (short-lived, donât store as env for your flow)

Important: redirect_uri must exactly match what you configured in Zoho API Console, and access_type=offline is required to receive a refresh token.


# **TASKS:**

Rename the fees to:
- Seller fee instead of fee in seller 
- Our fee to Service fee

Check if i can rename the organization to the same name

- Journal failed only not validation, add it in the scheduled tasks table, t1a record in table  
  remove due date and check if PO number can be not unique get link of invoice in url and save it in the database.

- Other transactions (t2,t3) will propagate error

- Check custom templates and create it like talabat (structure elements not shape)

- Return url of the pdf in the invoice creation. EDIT: The link will be inferred from zoho by sending the order id

- Make sure the following elements exist on the invoice:  
* [x] order id
- [x] invoice id
- [x] date
- [x] tax
- [x] customer name
- [x] customer phone



The first digit follows the common accounting category; the remaining digits organize accounts logically and leave space for expansion.

- `1010 — Bank Account (Cash)`
    
    - `1xxx` means asset.
    - `10xx` places highly liquid cash and bank accounts first.
    - `1010` leaves `1000` available as a category heading and permits future accounts such as `1020` or `1030`.
- `1200 — Payment Gateway`
    
    - `1xxx` means asset because unsettled PSP funds are money owed to the company.
    - `12xx` separates funds in transit from actual bank cash.
    - Future gateway accounts could use `1210`, `1220`, etc.
- `2100 — Sellers Payable`
    
    - `2xxx` means liability.
    - `21xx` represents short-term operational payables.
    - It distinguishes seller obligations from taxes, payroll liabilities, loans, and other liabilities.
- `4100 — Buyer Fees`
    
    - `4xxx` means income or revenue.
    - `41xx` represents primary operating revenue.
    - It leaves other ranges available for commissions, delivery revenue, interest, and other income.



https://www.zoho.com/books/academy/accounting-principles/what-is-chart-of-accounts.html


Zoho’s accounting guidance uses the same ranges: `1000–1999` assets, `2000–2999` liabilities, `3000–3999` equity, `4000–4999` income, and `5000–5999` expenses. It also recommends unique account codes for identification and organization