# ACS SSO JavaScript Integration Guide

This guide explains how to protect static HTML pages using the ACS Single Sign-On (SSO). This version is **Cookie-Independent**, meaning it works even if the browser blocks third-party cookies (Safari, Chrome Incognito, etc.).

---

## 🚀 Minimum Viable Integration

Add this snippet to the **very top** of your `<head>` section. It will hide the page content immediately and only reveal it once the user is verified.

### 1. Copy the Protection Script
```html
<!-- ACS SSO Protection -->
<script>
(function() {
    const SSO_PORTAL_URL = "https://api.acs.gr/sso/";
    const API_ENDPOINT = "https://api.acs.gr/sso/api/guardian-profile.php";
    
    const urlParams = new URLSearchParams(window.location.search);
    const tokenFromUrl = urlParams.get('sso_token');
    if (tokenFromUrl) sessionStorage.setItem('sso_token', tokenFromUrl);

    const savedToken = sessionStorage.getItem('sso_token');
    const LOGIN_URL = SSO_PORTAL_URL + "?return_to=" + encodeURIComponent(window.location.origin + window.location.pathname);

    document.documentElement.style.display = 'none'; // Hide page

    async function checkAuthentication() {
        try {
            const fetchUrl = new URL(API_ENDPOINT);
            if (savedToken) fetchUrl.searchParams.set('sso_token', savedToken);

            const response = await fetch(fetchUrl.toString(), { method: 'GET' });
            const data = await response.json();

            if (data && data.success) {
                // Success: Show user name if element exists
                const nameEl = document.getElementById('sso-user-name');
                if (nameEl) nameEl.textContent = data.fullName;
                
                document.documentElement.style.display = 'block'; // Show page
                
                // Clean URL (removes ?sso_token=... from address bar)
                if (tokenFromUrl) {
                    urlParams.delete('sso_token');
                    urlParams.delete('ps_auth');
                    const cleanUrl = window.location.pathname + (urlParams.toString() ? '?' + urlParams.toString() : '');
                    window.history.replaceState({}, document.title, cleanUrl);
                }
            } else {
                window.location.href = LOGIN_URL;
            }
        } catch (err) {
            window.location.href = LOGIN_URL;
        }
    }
    checkAuthentication();
})();
</script>
```

### 2. (Optional) Display User Name & Role
If you want to show the logged-in user's name and role on your page, add elements with the IDs `sso-user-name` and `sso-user-role`:

```html
<p>Logged in as: <strong id="sso-user-name">...</strong> (<span id="sso-user-role">...</span>)</p>
```

You can update the protection script's success block (line 37-39) to also populate the role:
```javascript
            if (data && data.success) {
                // Success: Show user name and role if elements exist
                const nameEl = document.getElementById('sso-user-name');
                if (nameEl) nameEl.textContent = data.fullName;
                
                const roleEl = document.getElementById('sso-user-role');
                if (roleEl) {
                    // Capitalize role for display
                    roleEl.textContent = data.role.charAt(0).toUpperCase() + data.role.slice(1);
                }
```

---

## 🛠️ How it Works
1.  **Gatekeeping**: The script hides the entire `<html>` tag before the page even finishes loading.
2.  **Redirect**: If no session is found, it redirects the user to the ACS Identity Portal.
3.  **Token Bridge**: After login, the portal sends the user back with a `sso_token`. This token is saved in `sessionStorage` to keep the user logged in even if the browser blocks cookies.
4.  **Auto-Cleanup**: The script automatically removes the token from the URL bar once it's saved, keeping your links clean.

## 🌍 Integration Requirements
- **HTTPS**: Both the SSO portal and your page must run on HTTPS.
- **CORS**: Automatically handled. The SSO server is configured to allow any domain to integrate out-of-the-box.
