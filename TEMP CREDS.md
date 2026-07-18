
NCSI admin server
muscatapps
Ajm!7nI%yjf1T7!8

My client wants to host a nodejs app and sql server on a windows server. I have never dealt with windows server i have always dealt with linux, nginx etc.. I just purchased a Windows 2022 server VPS from godaddy to train on it. Can you give me exact guide on what to do until the app works like linux and listening on a port while connected to the database

Below is the Windows equivalent of the Linux pattern you are used to:

**Linux:** Nginx on `80/443` → reverse proxy to Node on `3000` → PM2/systemd keeps Node alive  
**Windows:** IIS on `80/443` → ARR + URL Rewrite reverse proxy to Node on `127.0.0.1:3000` → PM2 Windows service keeps Node alive

Microsoft’s IIS URL Rewrite + Application Request Routing setup is the standard IIS reverse-proxy approach, and PM2’s ecosystem file is the cleanest way to define a production Node process. ([Microsoft Learn](https://learn.microsoft.com/en-us/iis/extensions/url-rewrite-module/reverse-proxy-with-url-rewrite-v2-and-application-request-routing "Reverse Proxy with URL Rewrite v2 and Application Request Routing | Microsoft Learn"))

USE [ncsi];  
GO  
  
IF NOT EXISTS (  
SELECT 1  
FROM sys.database_principals  
WHERE name = N'ncsi'  
)  
BEGIN  
CREATE USER [ncsi] FOR LOGIN [ncsi] WITH DEFAULT_SCHEMA = [dbo];  
END  
GO  
  
ALTER ROLE [db_owner] ADD MEMBER [ncsi];  
GO

---

## 1. Recommended final architecture

Use this setup:

```text
https://example.com
        │
        ▼
IIS website on port 80/443
        │
        ├── serves Vue dist files directly
        │
        └── proxies /api/* to http://127.0.0.1:3000
                                │
                                ▼
                         Node.js API via PM2
                                │
                                ▼
                         SQL Server via Prisma
```

Do **not** expose port `3000` publicly. Keep public access on `80` and `443`; let IIS talk to Node locally.

---

## 2. Prepare your folders

Use simple Windows paths. Example:

```powershell
mkdir C:\apps\myapp\api
mkdir C:\apps\myapp\frontend
mkdir C:\sites\myapp
```

Suggested layout:

```text
C:\apps\myapp\api        # Node/Express/backend app
C:\apps\myapp\frontend   # Vue source code
C:\sites\myapp           # Vue production dist served by IIS
```

Copy your backend into:

```text
C:\apps\myapp\api
```

Copy or clone your Vue frontend into:

```text
C:\apps\myapp\frontend
```

---

## 3. Check Prisma + SQL Server

In your Prisma schema, the provider should be:

```prisma
datasource db {
  provider = "sqlserver"
}
```

A typical SQL Server Prisma URL looks like this:

```env
DATABASE_URL="sqlserver://localhost:1433;database=MyDatabase;user=myuser;password={mypassword};encrypt=true;trustServerCertificate=true;schema=dbo"
```

Prisma documents SQL Server URLs in this `sqlserver://HOST[:PORT];database=...` format, and also notes that special characters in usernames/passwords should be wrapped in `{}`. It also supports `trustServerCertificate=true`, which is often useful when connecting to a local/self-signed SQL Server during server setup. ([Prisma](https://www.prisma.io/docs/orm/core-concepts/supported-databases/sql-server?utm_source=chatgpt.com "Microsoft SQL Server - Prisma ORM"))

From the backend folder:

```powershell
cd C:\apps\myapp\api
npm ci
npx prisma generate
```

If you use Prisma migrations:

```powershell
npx prisma migrate deploy
```

Then test manually again:

```powershell
node server.js
```

In another PowerShell window:

```powershell
Invoke-WebRequest http://127.0.0.1:3000
```

Once this works, stop the manual Node process with `Ctrl + C`.

---

## 4. Make sure Node listens locally

In your Node app, prefer this:

```js
const port = process.env.PORT || 3000;

app.listen(port, "127.0.0.1", () => {
  console.log(`API running on http://127.0.0.1:${port}`);
});
```

This keeps the API private to the server. IIS will still be able to reach it, but the public internet will not connect directly to `3000`.

---

## 5. Install PM2

Open **PowerShell as Administrator**.

```powershell
npm install pm2@latest -g
pm2 -v
```

PM2 is installed globally with npm, and PM2’s basic production command is `pm2 start app.js`. ([PM2](https://pm2.keymetrics.io/docs/usage/quick-start/ "PM2 - Quick Start"))

---

## 6. Create a PM2 ecosystem file

Create this file:

```text
C:\apps\myapp\api\ecosystem.config.js
```

Example:

```js
module.exports = {
  apps: [
    {
      name: "my-api",
      cwd: "C:/apps/myapp/api",
      script: "server.js",
      exec_mode: "fork",
      instances: 1,
      autorestart: true,
      max_memory_restart: "500M",
      time: true,
      env_production: {
        NODE_ENV: "production",
        PORT: 3000,
        DATABASE_URL:
          "sqlserver://localhost:1433;database=MyDatabase;user=myuser;password={mypassword};encrypt=true;trustServerCertificate=true;schema=dbo"
      }
    }
  ]
};
```

PM2 recommends ecosystem config files for organized app definitions, supports `cwd`, `script`, app `name`, and environment-specific variables such as `env_production`, which you can start with `--env production`. ([PM2](https://pm2.keymetrics.io/docs/usage/application-declaration/ "PM2 - Configuration File"))

Start it:

```powershell
cd C:\apps\myapp\api
pm2 start ecosystem.config.js --env production
pm2 status
pm2 logs my-api
```

Save the process list:

```powershell
pm2 save
```

Useful PM2 commands:

```powershell
pm2 status
pm2 logs my-api --lines 200
pm2 restart my-api --update-env
pm2 stop my-api
pm2 delete my-api
```

---

## 7. Make PM2 survive reboot on Windows

This is the part that is different from Linux. PM2’s normal `pm2 startup` flow is designed around Linux/macOS init systems such as systemd, launchd, upstart, and similar systems, not native Windows services. ([PM2](https://pm2.keymetrics.io/docs/usage/startup/?utm_source=chatgpt.com "Persistent applications: Startup Script Generator"))

For Windows Server, use a real Windows service approach. A commonly used option is **pm2-installer**, which sets up PM2 as a Windows service, creates `PM2_HOME`, configures npm global paths under `C:\ProgramData`, and runs PM2 in the background so it persists across reboots and logged-out sessions. ([GitHub](https://github.com/jessety/pm2-installer "GitHub - jessety/pm2-installer: Install PM2 offline as a service on Windows or Linux. Mostly designed for Windows. · GitHub"))

Recommended approach:

1. Download or clone `pm2-installer` from its GitHub project.
    
2. Open PowerShell as Administrator inside the `pm2-installer` folder.
    
3. Run:
    

```powershell
npm run configure
npm run configure-policy
npm run setup
```

Then re-run your PM2 app setup from an elevated PowerShell:

```powershell
cd C:\apps\myapp\api
pm2 start ecosystem.config.js --env production
pm2 save
pm2 status
```

Then reboot the server and check:

```powershell
pm2 status
Invoke-WebRequest http://127.0.0.1:3000
```

Important: with the service approach, always run PM2 commands from **PowerShell as Administrator**, because the PM2 service may be running under a service account such as `Local Service`. ([GitHub](https://github.com/jessety/pm2-installer "GitHub - jessety/pm2-installer: Install PM2 offline as a service on Windows or Linux. Mostly designed for Windows. · GitHub"))

---

## 8. Build the Vue SPA

In your Vue folder:

```powershell
cd C:\apps\myapp\frontend
npm ci
npm run build
```

Copy the build output to IIS’s site folder. Depending on your Vue setup, the output folder is usually `dist`.

```powershell
robocopy C:\apps\myapp\frontend\dist C:\sites\myapp /MIR
```

Your Vue production files should now be in:

```text
C:\sites\myapp\index.html
C:\sites\myapp\assets\...
```

If your frontend calls the API, use one of these patterns:

```env
VITE_API_URL=/api
```

or, for Vue CLI:

```env
VUE_APP_API_URL=/api
```

Using `/api` is easiest because browser requests go to the same domain:

```text
https://example.com/api/...
```

Then IIS proxies those requests to Node.

---

## 9. Install IIS

Open PowerShell as Administrator:

```powershell
Install-WindowsFeature Web-Server, Web-Static-Content, Web-Default-Doc, Web-Http-Errors, Web-Http-Logging, Web-WebSockets -IncludeManagementTools
```

Windows Server roles/features can be installed through Server Manager or PowerShell, and Microsoft documents `Add Roles and Features` for Windows Server 2022. ([Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/server-manager/add-remove-roles-features?utm_source=chatgpt.com "Add or Remove Roles and Features in Windows Server"))

Then install:

1. **IIS URL Rewrite**
    
2. **Application Request Routing 3.0**
    

Microsoft documents URL Rewrite as the IIS module for rewrite rules, and ARR as the IIS proxy-based routing module that forwards HTTP requests to application servers. ([iis.net](https://www.iis.net/downloads/microsoft/url-rewrite?utm_source=chatgpt.com "URL Rewrite : The Official Microsoft IIS Site"))

---

## 10. Enable IIS reverse proxy

Open **IIS Manager**.

1. Click the **server name** in the left panel.
    
2. Open **Application Request Routing Cache**.
    
3. Click **Server Proxy Settings**.
    
4. Check **Enable proxy**.
    
5. Click **Apply**.
    

Microsoft’s IIS reverse proxy walkthrough uses URL Rewrite plus ARR to forward requests to backend applications. ([Microsoft Learn](https://learn.microsoft.com/en-us/iis/extensions/url-rewrite-module/reverse-proxy-with-url-rewrite-v2-and-application-request-routing "Reverse Proxy with URL Rewrite v2 and Application Request Routing | Microsoft Learn"))

---

## 11. Create the IIS website

In IIS Manager:

1. Right-click **Sites**.
    
2. Click **Add Website**.
    
3. Site name: `myapp`
    
4. Physical path:
    

```text
C:\sites\myapp
```

5. Binding:
    
    - Type: `http`
        
    - IP address: `All Unassigned`
        
    - Port: `80`
        
    - Host name: `example.com`
        

Then add another binding:

```text
www.example.com
```

You can stop the **Default Web Site** if it conflicts with port `80`.

---

## 12. Add `web.config` for Vue + API proxy

Create this file:

```text
C:\sites\myapp\web.config
```

Use this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>

        <!-- Proxy API requests to Node.js on port 3000 -->
        <rule name="API Reverse Proxy" stopProcessing="true">
          <match url="^api/(.*)" />
          <action type="Rewrite" url="http://127.0.0.1:3000/{R:1}" />
        </rule>

        <!-- Vue SPA fallback for history mode -->
        <rule name="Vue History Fallback" stopProcessing="true">
          <match url=".*" />
          <conditions logicalGrouping="MatchAll">
            <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
            <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
            <add input="{REQUEST_URI}" pattern="^/api/" negate="true" />
          </conditions>
          <action type="Rewrite" url="/index.html" />
        </rule>

      </rules>
    </rewrite>

    <staticContent>
      <remove fileExtension=".json" />
      <mimeMap fileExtension=".json" mimeType="application/json" />
      <mimeMap fileExtension=".webmanifest" mimeType="application/manifest+json" />
    </staticContent>
  </system.webServer>
</configuration>
```

Vue Router’s HTML5 history mode needs a server fallback to `index.html` when the requested URL is not a real static file; otherwise direct visits to routes like `/dashboard` can return `404`. ([Vue Router](https://router.vuejs.org/guide/essentials/history-mode.html "Different History modes | Vue Router"))

Now test locally on the server:

```powershell
Invoke-WebRequest http://localhost
Invoke-WebRequest http://localhost/api
```

If your API has a health route, better:

```powershell
Invoke-WebRequest http://localhost/api/health
```

---

## 13. Open the Windows firewall

Open only `80` and `443` publicly:

```powershell
New-NetFirewallRule -DisplayName "HTTP Inbound" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow

New-NetFirewallRule -DisplayName "HTTPS Inbound" -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow
```

`New-NetFirewallRule` is Microsoft’s PowerShell cmdlet for creating firewall rules. ([Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2025-ps&utm_source=chatgpt.com "New-NetFirewallRule (NetSecurity)"))

Do **not** open these publicly unless you have a specific reason:

```text
3000  # Node app
1433  # SQL Server
```

If your server is with a cloud provider, also open `80` and `443` in the provider firewall/security group.

---

## 14. Point your domain to the Windows server

At your DNS provider, add:

```text
@      A       YOUR_SERVER_PUBLIC_IPV4
www    CNAME   example.com
```

Or use:

```text
www    A       YOUR_SERVER_PUBLIC_IPV4
```

An `A` record maps a domain name to an IPv4 address, and a `CNAME` can point a subdomain like `www` to another hostname. ([Cloudflare](https://www.cloudflare.com/learning/dns/dns-records/dns-a-record/?utm_source=chatgpt.com "DNS A record"))

After DNS is updated, test from your own machine:

```powershell
nslookup example.com
```

Then:

```powershell
Test-NetConnection example.com -Port 80
```

`Test-NetConnection` is Microsoft’s PowerShell network diagnostic cmdlet and supports TCP port testing. ([Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection?view=windowsserver2025-ps&utm_source=chatgpt.com "Test-NetConnection (NetTCPIP)"))

---

## 15. Enable HTTPS with Let’s Encrypt

For Windows + IIS, use **win-acme**.

Run `wacs.exe` as Administrator and choose the IIS site/bindings for:

```text
example.com
www.example.com
```

win-acme is an ACME client for Windows, supports simple certificate creation/installation for local IIS, and automatically creates a scheduled task for renewals. ([GitHub](https://github.com/win-acme/win-acme "GitHub - win-acme/win-acme: Automate SSL/TLS certificates on Windows with ease · GitHub"))

After the certificate is installed, IIS should have HTTPS bindings for:

```text
https://example.com
https://www.example.com
```

Then test:

```powershell
Test-NetConnection example.com -Port 443
```

Open in browser:

```text
https://example.com
```

---

## 16. Optional: redirect HTTP to HTTPS

In IIS Manager, you can either let win-acme configure this during setup, or add a URL Rewrite rule.

Add this rule above the API and Vue fallback rules:

```xml
<rule name="Redirect HTTP to HTTPS" stopProcessing="true">
  <match url="(.*)" />
  <conditions>
    <add input="{HTTPS}" pattern="off" ignoreCase="true" />
  </conditions>
  <action type="Redirect" url="https://{HTTP_HOST}/{R:1}" redirectType="Permanent" />
</rule>
```

Final rule order should be:

```text
1. Redirect HTTP to HTTPS
2. API Reverse Proxy
3. Vue History Fallback
```

---

## 17. If you prefer `api.example.com`

Instead of:

```text
https://example.com/api
```

You can use:

```text
https://api.example.com
```

DNS:

```text
api    A    YOUR_SERVER_PUBLIC_IPV4
```

Create another IIS site or binding for `api.example.com`, then use this `web.config` in that site:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <rewrite>
      <rules>
        <rule name="API Reverse Proxy" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="http://127.0.0.1:3000/{R:1}" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

If you use a separate API subdomain, configure CORS in Node to allow your frontend domain:

```js
app.use(cors({
  origin: "https://example.com",
  credentials: true
}));
```

For a first Windows deployment, I would use `/api` on the same domain because it avoids most CORS issues.

---

## 18. Deployment workflow after setup

When you update the backend:

```powershell
cd C:\apps\myapp\api
git pull
npm ci
npx prisma generate
npx prisma migrate deploy
pm2 restart my-api --update-env
pm2 logs my-api --lines 100
```

When you update the Vue frontend:

```powershell
cd C:\apps\myapp\frontend
git pull
npm ci
npm run build
robocopy C:\apps\myapp\frontend\dist C:\sites\myapp /MIR
```

No PM2 restart is needed for the Vue SPA if IIS is serving static files.

---

## 19. Troubleshooting checklist

Check Node:

```powershell
pm2 status
pm2 logs my-api --lines 200
Invoke-WebRequest http://127.0.0.1:3000
```

Check IIS:

```powershell
Invoke-WebRequest http://localhost
Invoke-WebRequest http://localhost/api/health
```

Check public ports:

```powershell
Test-NetConnection example.com -Port 80
Test-NetConnection example.com -Port 443
```

Check IIS logs:

```text
C:\inetpub\logs\LogFiles
```

Check PM2 logs:

```powershell
pm2 logs my-api
```

Check common mistakes:

```text
DNS points to wrong IP
IIS binding missing the domain host name
ARR proxy not enabled
URL Rewrite or ARR not installed
Vue build not copied to C:\sites\myapp
Node still listening on a different port
PM2 process not saved with pm2 save
Windows firewall or cloud firewall blocking 80/443
SQL Server TCP/IP not enabled
DATABASE_URL has unescaped special characters
```

The key target state is:

```text
PM2 status: my-api online
http://127.0.0.1:3000 works on the server
http://localhost/api/... works through IIS
DNS A record points to server IP
https://example.com loads Vue
https://example.com/api/... reaches Node
```

Once those are true, your Windows deployment is live.