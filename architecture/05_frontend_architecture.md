# 05 — Frontend Architecture

> **Back to Index**: [00_index.md](00_index.md)

---

## 5.1 Architecture Type

ResearchAI uses a **Vanilla JavaScript SPA** — no React, Vue, or Angular. The entire frontend is served from a single `templates/index.html` shell. All screen rendering is done via JavaScript functions that return HTML strings and inject them into a central `#screen-container` div.

This approach was chosen for:
- **Zero build step** — no webpack, no npm build, no compilation
- **Fast loading** — no framework bundle overhead
- **Full control** — every pixel is manually controlled CSS

---

## 5.2 SPA Architecture Overview

```
index.html (shell)
├── #app-shell                   (rendered by appShell.js)
│   ├── #sidebar                 (navigation panel)
│   │   ├── .nav-item links      (onclick → nav('screen-id'))
│   │   └── .nav-section labels
│   ├── #topbar                  (title, back button, user menu)
│   └── #screen-container        (swapped on navigation)
│
└── <script> tags (all JS files loaded in order)
    ├── utils/sessionState.js    (loaded first — state singleton)
    ├── api.js                   (HTTP wrapper)
    ├── auth.js                  (auth state)
    ├── appRouter.js             (screen registry + nav())
    ├── appShell.js              (shell HTML + panel switcher)
    ├── services/*.js            (data service layer)
    └── screens/*.js             (50 screen render functions)
```

---

## 5.3 Navigation & Routing

The router is defined in `appRouter.js`. Every screen is registered in the `SCREENS` object:

```javascript
const SCREENS = {
  "dashboard":  { title: "Dashboard", panel: "user", render: screenDashboard },
  "editor":     { title: "Paper Editor", panel: "user", render: screenEditor },
  "diagrams":   { title: "Diagram Studio", panel: "user", render: screenDiagrams },
  // ...50 screens total
};
```

Navigation is triggered by `nav('screen-id')`:
1. Looks up screen in registry
2. Checks role-based access guard (admin/super_admin panels)
3. Pushes current screen to `_navHistory` stack
4. Calls `s.render()` → gets HTML string (or Promise)
5. Injects into `#screen-container`
6. Calls `afterScreen<Name>()` hook if defined
7. Highlights active nav item

### History Stack
A simple array `_navHistory` powers the back button. `goBack()` pops the last screen and navigates to it. No browser URL history manipulation — URLs always remain `/`.

---

## 5.4 State Management

ResearchAI uses a **module-level singleton** pattern for state:

```javascript
// utils/sessionState.js
const SessionState = {
  paper: {
    getId: () => localStorage.getItem('current_paper_id'),
    setId: (id) => localStorage.setItem('current_paper_id', id),
    // ...
  },
  project: {
    getId: () => localStorage.getItem('current_project_id'),
    // ...
  },
  user: { /* cached user object */ }
};
```

State is stored in `localStorage` for persistence across page refreshes. All screens read from `SessionState` to know the active paper/project context.

---

## 5.5 API Communication

All API calls go through the central `apiFetch()` wrapper in `api.js`:

```javascript
async function apiFetch(path, options = {}) {
  const csrfToken = getCookie('csrf_access_token');
  const response = await fetch('/api' + path, {
    credentials: 'include',  // sends HttpOnly cookies
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-TOKEN': csrfToken,  // double-submit CSRF protection
      ...options.headers
    },
    ...options
  });
  
  if (response.status === 401) {
    // Auto-redirect to login on auth failure
    nav('login');
    return;
  }
  
  return response.json();
}
```

Key behaviors:
- Always sends `credentials: 'include'` (JWT cookies)
- Automatically reads CSRF token from cookie and adds header
- 401 responses → auto logout and redirect to login
- Error responses are thrown as exceptions for catch blocks

---

## 5.6 Panel System

Three panels exist: `user`, `admin`, `super`. Each has its own sidebar nav section.

```javascript
// Panel switching
function switchPanel(panel) {
  ['user','admin','super'].forEach(p => {
    document.getElementById(`nav-${p}`).classList.toggle('hidden', p !== panel);
  });
}
```

Switching panels is automatic when `nav()` detects the target screen belongs to a different panel. Role guards prevent unauthorized panel access:
- `admin` panel: requires `role === 'admin' || 'super_admin'`
- `super` panel: requires `role === 'super_admin'`

---

## 5.7 Screen Render Pattern

Every screen follows this pattern:

```javascript
// screens/editor.js
function screenEditor() {
  // 1. Return loading state immediately (sync)
  return `<div class="loading-spinner">Loading...</div>`;
}

// After render hook
window.afterScreenEditor = async function() {
  // 2. Fetch data from API
  const paperId = SessionState.paper.getId();
  const paper = await apiFetch(`/papers/${paperId}`);
  
  // 3. Re-render with real data
  document.getElementById('screen-container').innerHTML = buildEditorHTML(paper);
  
  // 4. Attach event listeners
  attachEditorEventListeners();
};
```

The two-phase render (immediate skeleton → async data fill) prevents blank screens during API calls.

---

## 5.8 Design System (CSS)

All styles are in `static/css/main.css`. Uses CSS custom properties (variables):

```css
:root {
  --primary: #6366f1;          /* Indigo - brand color */
  --primary-light: #e0e7ff;
  --surface-1: #ffffff;        /* Main backgrounds */
  --surface-2: #f8fafc;        /* Card backgrounds */
  --border: #e2e8f0;
  --text-1: #0f172a;           /* Primary text */
  --text-2: #475569;           /* Secondary text */
  --text-3: #94a3b8;           /* Muted text */
}
```

**Component classes** (defined once, used everywhere):
- `.card` — white rounded card with shadow
- `.btn`, `.btn-primary`, `.btn-outline`, `.btn-ghost` — button variants
- `.badge` — pill label
- `.ai-input` — styled form input
- `.nav-item` — sidebar navigation link
- `.format-toolbar` — text editing toolbar

**Google Fonts**: Inter (body), loaded via CDN.

---

## 5.9 Screen Inventory

| Screen ID | JS Function | Description |
|-----------|-------------|-------------|
| `landing` | `screenLanding` | Public landing page |
| `login` | `screenLogin` | Login form |
| `register` | `screenRegister` | Registration form |
| `otp` | `screenOTP` | OTP email verification |
| `2fa` | `screen2FA` | TOTP 2FA verification |
| `dashboard` | `screenDashboard` | User home dashboard |
| `projects` | `screenProjects` | My projects list |
| `new-project` | `screenNewProject` | Project setup wizard |
| `upload` | `screenUpload` | Document upload |
| `ai-analysis` | `screenAIAnalysis` | AI analysis results |
| `generation` | `screenGeneration` | Paper generation |
| `editor` | `screenEditor` | Full paper editor |
| `diagrams` | `screenDiagrams` | Diagram Studio |
| `citations` | `screenCitations` | Citation manager |
| `citation-workspace` | `screenCitationWorkspace` | Citation editor |
| `plagiarism` | `screenPlagiarism` | Plagiarism checker |
| `plagiarism-workspace` | `screenPlagiarismWorkspace` | Scan results |
| `paraphraser-workspace` | `screenParaphraserWorkspace` | Paraphraser |
| `standalone-plagiarism` | `screenStandalonePlagiarism` | Standalone scanner |
| `ai-detection` | `screenAIDetection` | AI detection |
| `ai-humanizer` | `screenAIHumanizer` | Humanizer |
| `export` | `screenExport` | Export paper |
| `notifications` | `screenNotifications` | Notifications |
| `profile` | `screenProfile` | User profile |
| `subscription` | `screenSubscription` | Subscription |
| `admin-*` | `screenAdmin*` | Admin panel (6 screens) |
| `super-*` | `screenSuper*` | Super admin (12 screens) |

---

## 5.10 Error States & Loading States

**Loading**: `loadingHTML()` function returns a skeleton spinner injected while async data loads.

**Error**: `errorHTML(message)` function returns a styled error card with message.

**Toast notifications**: `showToast(message, type)` — top-right toast system for success/error/info feedback. Auto-dismisses after 3 seconds.

**Empty states**: Each screen defines its own empty state HTML for when no data exists (no papers, no projects, etc.).
