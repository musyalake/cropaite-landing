# Cropaite Technologies — Website Developer Guide
## Complete Hosting, Updating & Maintenance Guide

---

## 1. FOLDER STRUCTURE

Your website is made of 6 files. Keep them all in the same folder.

```
cropaite-site/
├── index.html        ← Main landing page (Home)
├── privacy.html      ← Privacy Policy
├── terms.html        ← Terms & Conditions
├── contact.html      ← Contact page
├── style.css         ← Shared styles (ALL pages use this)
├── version.json      ← APK version info (update this when releasing new APK)
└── cropaite.apk      ← Your APK file (rename to match apk_url in version.json)
```

**Critical rule:** Do NOT move any file out of this folder. All pages link to 
`style.css` and `version.json` using relative paths (`./`). If you move a file, 
the links will break.

---

## 2. HOW TO HOST FOR FREE

### Option A — GitHub Pages (Recommended for developers)
1. Create a GitHub account at github.com
2. Create a new repository called `cropaite-site` (make it Public)
3. Upload all 6 files to the repository
4. Go to: Repository → Settings → Pages → Source → `main` branch → `/root`
5. Your website will be live at: `https://yourusername.github.io/cropaite-site`

**To update:** Edit any file and commit → the site updates in ~1 minute.

### Option B — Netlify (Easiest, drag-and-drop)
1. Go to netlify.com → Sign up free
2. Drag your entire `cropaite-site/` folder onto the Netlify dashboard
3. Your site goes live immediately with a URL like `https://cropaite.netlify.app`
4. Connect a custom domain later (e.g. www.cropaite.com) in Netlify settings

**To update:** Drag the updated folder again, or connect to GitHub for auto-deploy.

### Option C — Firebase Hosting
Since you already use Firebase, this is a natural fit.
```bash
npm install -g firebase-tools
firebase login
firebase init hosting
# Set public directory to: cropaite-site
firebase deploy
```
Your site will be at: `https://your-project.web.app`

### Option D — Custom Domain (e.g. www.cropaite.com)
1. Buy domain at Namecheap (~$10/year) or Google Domains
2. Host on Netlify (free) or Firebase Hosting (free tier)
3. Point your domain DNS to the hosting provider (they give you instructions)

---

## 3. HOW TO RELEASE A NEW APK

When you build a new version of the Flutter app and want to update the website:

### Step 1 — Build the APK
```bash
# In your Flutter project
flutter build apk --release
```
The APK is at: `build/app/outputs/flutter-apk/app-release.apk`

### Step 2 — Rename the APK
Rename it clearly, e.g.: `cropaite_v1.1.0.apk`

### Step 3 — Update version.json
Open `version.json` and update these fields:
```json
{
  "version": "1.1.0",
  "release_date": "May 2026",
  "apk_url": "./cropaite_v1.1.0.apk",
  "release_notes": "Improved disease detection accuracy. Fixed login bug. Added offline scan queue.",
  "force_update": false
}
```

### Step 4 — Upload both files to your hosting
Upload `cropaite_v1.1.0.apk` and the updated `version.json` to your site folder.

### What happens next:
- Anyone who visits the website will see the new version number automatically
- Anyone who visited before and has the old version will see an **update popup** 
  the next time they open the page
- The popup shows your release_notes and a "Download Update" button

### force_update field:
- `false` = shows update popup but user can dismiss it
- `true`  = shows popup but there is NO dismiss button (use for critical updates)

---

## 4. HOW TO UPDATE THE FLUTTER APP TO CHECK FOR UPDATES

Add this to your Flutter app's startup flow so users get prompted inside the app too.

### Add dependencies (pubspec.yaml)
```yaml
dependencies:
  http: ^1.1.0
  package_info_plus: ^5.0.1
  url_launcher: ^6.2.0
```

### Create lib/services/update_checker.dart
```dart
import 'dart:convert';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:package_info_plus/package_info_plus.dart';
import 'package:url_launcher/url_launcher.dart';

class UpdateChecker {
  // ← Replace with your actual website URL
  static const String versionUrl = 'https://yoursite.com/version.json';

  static Future<void> check(BuildContext context) async {
    try {
      final res = await http.get(Uri.parse(versionUrl))
          .timeout(const Duration(seconds: 5));

      if (res.statusCode != 200) return;

      final data    = jsonDecode(res.body);
      final remote  = data['version'] as String;
      final notes   = data['release_notes'] as String? ?? '';
      final apkUrl  = data['apk_url'] as String? ?? '';
      final force   = data['force_update'] as bool? ?? false;

      final info    = await PackageInfo.fromPlatform();
      final current = info.version; // e.g. "1.0.0"

      if (_isNewer(remote, current) && context.mounted) {
        _showDialog(context, remote, notes, apkUrl, force);
      }
    } catch (_) {
      // Silent fail — don't crash the app if update check fails
    }
  }

  static bool _isNewer(String remote, String current) {
    final r = remote.split('.').map(int.parse).toList();
    final c = current.split('.').map(int.parse).toList();
    for (int i = 0; i < 3; i++) {
      if (r[i] > c[i]) return true;
      if (r[i] < c[i]) return false;
    }
    return false;
  }

  static void _showDialog(BuildContext context, String version, 
      String notes, String apkUrl, bool force) {
    showDialog(
      context: context,
      barrierDismissible: !force,
      builder: (_) => AlertDialog(
        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
        title: const Text('Update Available 🌿'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text('Version $version is available.', 
                style: const TextStyle(fontWeight: FontWeight.bold)),
            if (notes.isNotEmpty) ...[
              const SizedBox(height: 12),
              Text(notes, style: const TextStyle(fontSize: 13)),
            ],
          ],
        ),
        actions: [
          if (!force)
            TextButton(
              onPressed: () => Navigator.pop(context),
              child: const Text('Later'),
            ),
          ElevatedButton(
            style: ElevatedButton.styleFrom(
              backgroundColor: const Color(0xFF1A6B3C),
            ),
            onPressed: () async {
              final uri = Uri.parse(apkUrl.startsWith('http') 
                  ? apkUrl 
                  : 'https://yoursite.com/$apkUrl');
              if (await canLaunchUrl(uri)) launchUrl(uri);
            },
            child: const Text('Download Update'),
          ),
        ],
      ),
    );
  }
}
```

### Call it in your main app screen (e.g. AppBootstrapScreen or HomeScreen)
```dart
@override
void initState() {
  super.initState();
  // Check for updates after a short delay so UI loads first
  Future.delayed(const Duration(seconds: 2), () {
    if (mounted) UpdateChecker.check(context);
  });
}
```

### Set the version in pubspec.yaml
```yaml
version: 1.0.0+1
# format: version_name+version_code
# Update version_name (1.0.0) to match version.json when releasing
```

---

## 5. HOW TO CUSTOMISE THE WEBSITE

### Change the contact email
Search for `hello@cropaite.com` in all files and replace with your real email.

### Change the phone number
Search for `0741473833` and replace with your real number.

### Update the APK install instructions
In `index.html`, find the text about enabling "Install from unknown sources" 
and update it if needed for your target Android version.

### Change pilot status tags
In `index.html`, find the section with `tag-live`, `tag-building`, `tag-roadmap` 
tags. Update them as you complete features:
```html
<!-- Change this: -->
<span class="tag tag-building">⚙ Task Management — Building</span>
<!-- To this when done: -->
<span class="tag tag-live">✓ Task Management — Live</span>
```

### Add your real Effective Date
In `privacy.html` and `terms.html`, find `[DD/MM/YYYY]` and replace with the 
actual date you want these documents to take effect.

### Add your ODPC Registration Number
In `privacy.html`, find `[Insert upon registration]` and add your ODPC number 
once you have it. Register at: odpc.go.ke

---

## 6. MAKING THE CONTACT FORM WORK PROPERLY

The contact form currently uses `mailto:` (opens the user's email client). 
This works but is not ideal. To collect form submissions in a database:

### Option A — Formspree (Free, no backend needed)
1. Go to formspree.io and create a free account
2. Create a new form → get your endpoint URL (e.g. `https://formspree.io/f/abcdefgh`)
3. In `contact.html`, replace the `submitForm()` function's `window.location.href` line with:
```javascript
const res = await fetch('https://formspree.io/f/abcdefgh', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name, email, phone, farmName, subject, message })
});
if (res.ok) {
  document.getElementById('formSuccess').style.display = 'block';
}
```

### Option B — Your own FastAPI backend
Add a POST endpoint to your existing Cropaite backend:
```python
@app.post("/contact")
async def contact(data: dict):
    # Send email via SendGrid, or store in Firestore
    return {"status": "ok"}
```

---

## 7. CHECKLIST BEFORE GOING LIVE

- [ ] Replace `[DD/MM/YYYY]` with real effective dates in privacy.html and terms.html
- [ ] Replace placeholder phone `+254 700 000 000` with your real number  
- [ ] Replace `hello@cropaite.com` with your real email if different
- [ ] Add your real APK file named `cropaite.apk` (or update version.json `apk_url`)
- [ ] Add your ODPC registration number in privacy.html once registered
- [ ] Test all navigation links work correctly
- [ ] Test the APK download button
- [ ] Test on mobile (open in Chrome on your Android phone)
- [ ] Set up your domain (optional but professional)

---

## 8. QUICK REFERENCE — FILE RESPONSIBILITIES

| File          | What it does                                  | When to edit |
|---------------|-----------------------------------------------|--------------|
| index.html    | Main landing page, download button, features  | When features change or go live |
| privacy.html  | Full privacy policy                           | When policy changes |
| terms.html    | Full terms & conditions                       | When terms change |
| contact.html  | Contact form, FAQ                             | Rarely |
| style.css     | ALL visual styling for every page             | When changing colours or layout |
| version.json  | APK version number and release notes          | Every time you release a new APK |
| cropaite.apk  | The actual app file                           | Every new release |

---

*Cropaite Technologies — Built in Naivasha, Kenya*
