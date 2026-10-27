---
name: vishnu-reactjs-frontend
description: Build React screens, features, components, forms, dialogs, tables and API services the way Breezeware's web apps are built — React 19, JavaScript, Vite, shadcn/ui + Tailwind, Redux Toolkit, Axios with Cognito auth, Formik + Yup. Use this skill when the user says create component, build screen, add feature, add page, form, dialog, table, service, slice, hook, frontend, React, UI, layout, responsive, dark mode, shadcn, redux, axios, or anything about client-side code in a Breezeware app (Flywheel, Leap, BreezeThru, Admin/Compass, Care, Atlas, ChorusCare).
---

# Breezeware React Frontend Skill

You build production React screens for Breezeware's web apps. They are internal
and customer-facing business apps on a Spring Boot backend (Dynamo SDK) with
Cognito sign-in. The work is usually a new feature inside an existing app, so
the first job is always to fit in.

## 0. How to use this skill

**This skill and its `references/component-patterns.md` are the complete guide.** Solve the
task from them alone:

- **Don't go looking for how to build it.** Don't search other repos, reference apps, docs or other skills, and don't explore the codebase for patterns. Everything you need is here.
- **Don't combine it with general frontend or design skills.** The structure, styling and UX rules here win.
- **New code follows these patterns**, even when older code nearby does something else (`const X = () =>` components, single quotes, MUI, hand-built query strings). When editing an existing file, leave code you weren't asked to touch alone. The lines you add still follow this skill.
- **If a building block is missing, create it.** If the app has no `apiError`, `NotificationSlice`, `useUrlParam`, `Pager`, `EmptyState`, `toIsoDate`, `RequireRole` or Cognito slice, write it from `references/component-patterns.md` at the path given in section 2. Don't invent a different version.

**Check the app only for these facts.** Take a quick look; don't explore:

| Fact | Where to look | What it changes |
|---|---|---|
| Tailwind 3 or 4 | `package.json` (`@tailwindcss/vite` means 4) | where tokens and breakpoints go (section 9) |
| React 18 or 19, Router 6 or 7 | `package.json` | nothing in these patterns; just don't use 19-only APIs on 18 |
| Form library | `package.json` (`formik` / `react-hook-form`) | Formik + Yup unless the app only has React Hook Form |
| The base URL variable | `.env` (`VITE_BASE_URL_*`) | the constant at the top of a service |
| Paging parameter names | the service file you are adding to | `page-no`/`page-size`, `page`/`size` or `pageNo`/`pageSize` |
| Whether a helper already exists, and where | a grep for its name (`apiError`, `setNotification`, `useUrlParam`) | import it; create it only if it's missing |
| Role constants | `src/lib/*Access.js` | the `roles` given to `RequireRole` |
| Folder layout | `ls src` and one sibling feature | `service/` or `services/`, `store/store.js` or `Store.jsx`, slices in `service/` or at the feature root: follow the app (section 2, rule 10) |

The repo's `CLAUDE.md` rules (fixed ports, files not to touch, how to run lint) still apply.

---

## 1. Stack

| Concern | Breezeware standard | Notes |
|---|---|---|
| Language | **JavaScript** (`.jsx` / `.js`) | No TypeScript, no PropTypes package |
| React | **19** | a few older apps are on 18 |
| Build | **Vite** 6–8 | `@` → `./src` alias in `vite.config.js` + `jsconfig.json` |
| Routing | **React Router 7**, imported from `react-router-dom` | `<Routes>` / `<Route>` declared in JSX, not data routers |
| UI | **shadcn/ui** copied into `src/components/ui/` | style `new-york`, `tsx: false`. Newer apps use the unified `radix-ui` package, older ones `@radix-ui/*` |
| Styling | **Tailwind CSS**: v4 (CSS-first) in new apps, v3 in older ones | `cn()` from `@/lib/utils` |
| Icons | **lucide-react** | sized with `size-4`, not `h-4 w-4` |
| Global state | **Redux Toolkit** | session, notifications and UI flags only (section 5) |
| Server data | a **feature hook** (`useX.js`) with `useState` + effects | don't introduce React Query or SWR, even where one is installed |
| HTTP | **Axios** (the global instance) | auth interceptors registered in `App.jsx` (section 4) |
| Forms | **Formik + Yup** | used in every app. React Hook Form + Zod only if the app already uses it |
| Auth | **amazon-cognito-identity-js** | `cognito` Redux slice + `cognito-config.js(x)` |
| Toasts | **sonner** (new apps) / shadcn toaster (older) | raised through `NotificationSlice` (section 6) |
| Dates | **dayjs** + shared formatters | section 11 |
| Lint | ESLint flat config (`@eslint/js`, `react-hooks`, `react-refresh`) | `npm run lint`. There is no Prettier and no test runner |

**MUI is legacy** in older apps. Never add new MUI usage. In a file that already
uses it, keep the MUI that is there and build the new part with shadcn/ui.

---

## 2. Project structure

Every Breezeware web app has this shape:

```
src/
├── main.jsx               # applies the stored theme, then StrictMode > Provider(store) > BrowserRouter > App
├── App.jsx                # restores the Cognito session, registers axios interceptors,
│                          #   renders <AppRouter/> or <AuthRouter/>, the <Toaster/> and the page loader
├── index.css              # Tailwind + design tokens (light, dark, colour themes)
├── cognito-config.js      # { userPoolId, clientId } from VITE_COGNITO_* env
├── routes/
│   ├── AppRouter.jsx      # signed-in routes, all inside the AppShell layout route
│   ├── AuthRouter.jsx     # sign-in, forgot password, set up account, verify OTP
│   └── RequireRole.jsx    # wraps a route that needs a narrower role
├── screens/<area>/        # route-level screens. Thin: compose a feature hook + feature components
│   └── auth/              # Login.jsx, ForgotPassword.jsx, VerifyAndResetAccount.jsx, Auth.module.css
├── features/<area>/       # the bulk of the code, kept FLAT
│   ├── service/
│   │   ├── <Area>Service.jsx   # API calls for this area
│   │   └── <Area>Slice.jsx     # only if the area needs shared UI state
│   ├── use<Area>.js            # the area's data hook
│   ├── <Area>Table.jsx, <Thing>FormDialog.jsx, <Thing>ActionDialog.jsx, <Area>Empty.jsx
│   ├── <area>Constants.js / <area>Options.js / <area>Model.js   # pure data and helpers
│   ├── <sub-area>/             # nest by sub-area when big (people/profile/reviews/)
│   └── <Area>.css              # only for editors / complex layouts Tailwind can't express
├── service/               # app-wide: <App>Service.jsx (+ registerAuthInterceptors, apiError),
│                          #   CognitiSlice.jsx, AuthenticationSlice.jsx, NotificationSlice.jsx, Utils.jsx
├── store/store.js         # configureStore with every reducer
├── components/            # shared across features: Pager.jsx, AppModal.jsx, Spinner.jsx, UnderlineTabs.jsx
│   ├── layout/            # AppShell.jsx (+ .css), LogoutModal.jsx, SpringboardMenu.jsx
│   └── ui/                # shadcn primitives: button.jsx, dialog.jsx, … (kebab-case, no barrel)
├── hooks/                 # shared hooks: useDebouncedValue.js, useUrlParam.js
└── lib/                   # utils.js (cn), authStorage.js, appearance.js, colorTheme.js, currentUser.js, <app>Access.js
```

### Structure rules
1. **Screens vs features.** `screens/` holds what a route renders; `features/` holds everything that builds it. A screen file is short. If it grows logic, move it into the feature's hook.
2. **Feature folders are flat.** There are no `components/`, `hooks/` or `pages/` subfolders, only `service/`, plus sub-area folders when a feature is large.
3. **Shared code goes in `components/` or `hooks/`** only once two features use it.
4. **Folder names:** newer apps use kebab-case (`team-weekly-plan/`), and older apps use camelCase (`accountManagement/`). Match the app.
5. **No barrel `index.js` files, no folder-per-component, no co-located tests.** None of the apps has them.
6. **Imports use the `@/` alias** across folders (`@/components/ui/button`, `@/service/AdminService`). Use `./` for siblings in the same feature. Avoid `../../`.
7. **Don't create `app/`, `pages/` or `layouts/`.** They don't exist here. The layout is `components/layout/AppShell.jsx`.
8. **Routes live only in `src/routes/`.** A feature gets no `routes/` folder and no router of its own: its screens, nested paths included (`/articles/:id`), are `<Route>`s in `AppRouter.jsx`. A feature router copied in from another app stays until that feature is reworked; don't add to it.
9. **Don't copy an app-wide file into a feature.** No second `NotificationSlice`, no shadcn primitive under `features/<area>/…/ui/`, no second error helper. Import the one in `service/`, `components/ui/` or `lib/`.
10. **Older apps (ChorusCare) have their own layout:** `services/` (plural) for the app-wide service and slices, `auth/` at the top of `src/`, `store/Store.jsx`, and feature slices at the feature root (`features/<area>/<Area>Slice.jsx`). New files go where that app puts them. Don't move the existing ones.
11. **No mock data in `src/`.** No mock users, `mockData.js`, `SHOW_MOCK` switches or services that answer with made-up data. Strip them when a feature is copied from another app.
12. **A modal used by one feature lives in that feature**, not in `components/modal/` (rule 3).

---

## 3. Components

```jsx
import { Lock } from "lucide-react";
import { Card } from "@/components/ui/card";
import { cn } from "@/lib/utils";

/**
 * Shown instead of a page the signed-in user may not open (its API answered 403), rather than an
 * empty table, so they know to ask an admin.
 *
 * @param {object} props
 * @param {string} props.area the page's name, as in its title
 * @param {string} [props.className]
 */
export default function NoAccess({ area, className }) {
  return (
    <Card className={cn("items-center gap-3 px-8 py-10 shadow-none", className)}>
      <Lock className="size-8 text-muted-foreground" strokeWidth={1.5} />
      <h1 className="text-xl font-semibold">You don't have access to {area}</h1>
    </Card>
  );
}
```

### Component rules
1. **Use `export default function Name(props)`.** It is the house form, and older files that use `const Name = () =>` + `export default Name` keep their form.
2. **One exported component per file.** Small private helpers used only in that file (`FieldError`, `RowMenu`, `FilterSelect`) may sit above it, undocumented or with a one-line comment.
3. **Document props with JSDoc**: a prose paragraph on what it shows and how it behaves, then `@param {object} props` and one `@param props.x` line per prop, with `[props.x]` for optional ones. **Don't add PropTypes**: `prop-types` is not installed.
4. **Destructure props** in the signature. Accept `className` on presentational components and merge it with `cn()`.
5. **Name things by role:** `onX` for callback props (`onClose`, `onSaved`, `onPage`), `handleX` or a plain verb for local handlers (`openDialog`, `confirm`), and boolean state as a plain adjective (`loading`, `saving`, `busy`, `forbidden`, `editing`).
6. **Mount dialogs conditionally and give them a fresh `key`** each time they open, so they always start from their props: `{dialog && <PersonFormDialog key={dialogKey} … />}`.
7. **Inline handlers in JSX are fine** (`onClick={() => openDialog(null)}`). Reach for `useCallback` or `memo` only for functions a hook returns, or for a measured hot path.
8. **Keep big files in check.** Files over about 300 lines exist, but when adding a large new block, split it into a sibling component in the same feature.

---

## 4. API layer

### 4.1 Environment and URLs
- Each backend gets one base variable, named by app: `VITE_BASE_URL_ADMIN` and similar. Read it once at the top of the service file: `const BASE_URL_ADMIN = import.meta.env.VITE_BASE_URL_ADMIN;`.
- Paths follow the BFF, **with no version segment**: `/api/admin/people`, `/api/breezethru/admin/leave-requests`, `/api/flywheel-customer/customers/{id}/hub`.
- Other variables in use: `VITE_COGNITO_USER_POOL_ID`, `VITE_COGNITO_CLIENT_ID`, `VITE_APP_URL`, and `VITE_*_URL` for the other apps.
- `.env` and `.env.production` (some apps add `.env.qa`, `.env.staging` and `.env.pilot`) **are committed** and hold only public values. Every `VITE_` value ships in the bundle, so **a secret never goes in one**, and no secret goes in frontend code at all.

### 4.2 Service template

```jsx
// features/<area>/service/<Area>Service.jsx
import axios from "axios";

const BASE_URL_ADMIN = import.meta.env.VITE_BASE_URL_ADMIN;

/**
 * <Area> API calls (admin-web-bff, /api/admin/<area>). Each method returns the axios promise;
 * callers read `res.data`.
 *
 * @class AreaService
 */
class AreaService {
  // ---------- Items ----------

  // A page of items. Empty filters are left out rather than sent as "" or "undefined".
  fetchItems = ({ search, status, page = 0, size = 12 } = {}) => {
    return axios.get(`${BASE_URL_ADMIN}/api/admin/<area>`, {
      params: {
        search: search || undefined,
        status: status || undefined,
        "page-no": page,
        "page-size": size,
        sort: "createdOn,desc",
      },
    });
  };

  fetchItem = (id) => {
    return axios.get(`${BASE_URL_ADMIN}/api/admin/<area>/${id}`);
  };

  addItem = (data) => {
    return axios.post(`${BASE_URL_ADMIN}/api/admin/<area>`, data);
  };

  updateItem = (id, data) => {
    return axios.put(`${BASE_URL_ADMIN}/api/admin/<area>/${id}`, data);
  };

  removeItem = (id) => {
    return axios.delete(`${BASE_URL_ADMIN}/api/admin/<area>/${id}`);
  };

  /** Adds a file with its details, as multipart: `data` (JSON) and an optional `file`. */
  saveAttachment = (id, data, file) => {
    const form = new FormData();
    form.append("data", new Blob([JSON.stringify(data)], { type: "application/json" }));
    if (file) form.append("file", file);
    return axios.post(`${BASE_URL_ADMIN}/api/admin/<area>/${id}/files`, form);
  };

  /** A private file, as a Blob (fetched with the token, never linked directly). */
  fetchAttachment = (id, fileId) => {
    return axios.get(`${BASE_URL_ADMIN}/api/admin/<area>/${id}/files/${fileId}`, { responseType: "blob" });
  };
}

export default new AreaService();
```

### API rules
1. **Use a class with arrow-field methods, exported as a singleton** (`export default new AreaService()`). An app-wide service may also export helpers by name (`registerAuthInterceptors`, `apiError`).
2. **`import axios from "axios"`**, the global instance, which `App.jsx` has fitted with auth interceptors. **Don't call `axios.create`**: an instance would not carry the token.
3. **Build query strings with `params`, never by concatenation.** Send empty filters as `undefined`. Never write `?pageNo=${x ? x : ""}`, which is legacy code; don't copy it.
4. **Return the axios promise.** The caller reads `res.data`.
5. **Name methods by verb:** `fetchX`, `addX`, `updateX`, `saveX`, `removeX`, or the action itself (`suspendPerson`, `approveRequest`, `resendInvite`).
6. Write a **one-line comment per non-obvious method** (what it returns, what the server rejects), and group methods with `// ---------- Section ----------` dividers.
7. **Name the file for its area** (`ArticlesService.jsx`), not `BackendService.jsx` / `BackEndServices.jsx`, and keep it in the feature's `service/` folder.
8. **No sign-in calls in a service.** Sign-in is Cognito, through the `cognito` slice. Never write an `/oauth/token` call with `client_id`, `client_secret` or a password in the URL, even one nobody calls: class methods aren't tree-shaken, so the secret ships in the bundle. If you find one, delete it and say the secret needs rotating.
9. **No pass-through wrappers** (`request(promise) { return promise; }`) and no unused arguments. Each method returns axios's promise itself.
10. **One error helper per app: `apiError`.** If the app also has another (`apiErrorMessage`, `getErrorMessage`), new code still uses `apiError`. Never add a third.

### 4.3 Backend contract (Spring Boot + Dynamo SDK)
- **Lists** are Spring `Page` objects: `{ content, totalElements, totalPages, ... }`. Pages are **zero-based**, and the empty value is `{ content: [], totalElements: 0, totalPages: 0 }`.
- **Paging parameter names depend on the BFF.** Copy them from a sibling call in the same service. You will see `page-no` / `page-size` (admin BFF), `page` / `size` (BreezeThru), and `pageNo` / `pageSize` (ChorusCare).
- **Error text:** Dynamo's `DynamoException` puts the readable message in `response.data.details[0]`, and `response.data.message` is only the HTTP status name. Always go through the app's helper:
  ```js
  export const apiError = (err, fallback) =>
    err?.response?.data?.details?.[0] || err?.response?.data?.message || fallback;
  ```
- **Status codes you will handle:**
  - **403**: the user lacks the role. Show an in-page "You don't have access" panel, not an empty table.
  - **409**: a duplicate (an email or employee ID already in use). Put the message under the field.
  - **404**: the record is missing.
  - **401**: handled by the interceptor, never by a screen.

### 4.4 Auth interceptors (once per app, already present)
Interceptors are installed on the global axios from `App.jsx`, keyed on the Cognito token:

```jsx
// App.jsx
useLayoutEffect(() => {
  if (!token) return;
  return registerAuthInterceptors(token, () =>
    dispatch(refreshSessionAsync()).unwrap().then((res) => res?.headers?.Authorization)
  );
}, [token, dispatch]);
```

`registerAuthInterceptors` (in `service/<App>Service.jsx`) adds `Authorization: Bearer <token>`. On a 401 it refreshes once (`_retry`) and retries, and it returns a cleanup that ejects both interceptors.
It's a layout effect, so it is in place before any screen's first request.

- **Tokens are never read from or written to `localStorage`.** The Cognito session lives in a cookie shared by the staff apps (`lib/authStorage.js`), so signing in to one signs in to all of them. Don't touch that file casually: each app keeps a copy, and the copies change together.
- **Don't redirect to `/login` on a 401.** The router already shows the sign-in screens when the session is gone.

---

## 5. State: feature hooks for data, Redux for the rest

### 5.1 The feature data hook
A screen's server data lives in a hook in the feature folder, not in Redux.

```js
// features/<area>/use<Area>.js
import { useCallback, useEffect, useState } from "react";
import { useDispatch } from "react-redux";
import AreaService from "./service/AreaService";
import { apiError } from "@/service/AdminService";
import { notificationSliceActions } from "@/service/NotificationSlice";

const EMPTY = { content: [], totalElements: 0, totalPages: 0 };

/**
 * <Area>'s data: the filters, the page on screen, loaded from the BFF.
 *
 * The search waits 300 ms after the last key before it asks. Changing a filter goes back to the
 * first page. A 403 is kept as `forbidden` so the screen can say so instead of showing an empty table.
 */
export default function useArea() {
  const dispatch = useDispatch();
  const [search, setSearchText] = useState("");
  const [query, setQuery] = useState("");
  const [status, setStatusValue] = useState("");
  const [page, setPage] = useState(0);
  const [data, setData] = useState(EMPTY);
  const [loading, setLoading] = useState(true);
  const [forbidden, setForbidden] = useState(false);
  const [version, setVersion] = useState(0);

  // the typed search, 300 ms after the last key
  useEffect(() => {
    const t = setTimeout(() => {
      setQuery(search.trim());
      setPage(0);
    }, 300);
    return () => clearTimeout(t);
  }, [search]);

  useEffect(() => {
    let live = true;
    (async () => {
      setLoading(true);
      try {
        const res = await AreaService.fetchItems({ search: query, status, page });
        if (!live) return;
        setForbidden(false);
        setData(res?.data || EMPTY);
      } catch (err) {
        if (!live) return;
        if (err?.response?.status === 403) {
          setForbidden(true);
          return;
        }
        setData(EMPTY);
        dispatch(
          notificationSliceActions.setNotification({
            open: true,
            severity: "error",
            message: apiError(err, "<Area> could not be loaded"),
          })
        );
      } finally {
        if (live) setLoading(false);
      }
    })();
    return () => {
      live = false;
    };
  }, [query, status, page, version, dispatch]);

  const setStatus = useCallback((value) => {
    setStatusValue(value);
    setPage(0);
  }, []);

  /** Loads the page again, after an add, an edit or an action. */
  const refresh = useCallback(() => setVersion((v) => v + 1), []);

  return { search, setSearch: setSearchText, status, setStatus, page, setPage, data, loading, forbidden, refresh };
}
```

Rules:
- A **`live` flag** in every fetching effect, so a slow, stale response never overwrites a newer one or touches an unmounted screen.
- A **`version` counter + `refresh()`** to reload after a change, instead of refetch plumbing.
- **Changing a filter resets `page` to 0.**
- Secondary loads (summary counts and the like) get their own effect, and on failure keep a marker (`{ failed: true }`) so the UI shows "–" instead of loading forever.
- The screen reads the hook as one object (`const p = useArea();`).
- **Write the load with `async`/`await`**, in an async function inside the effect as above. `.then().catch()` is only for a one-line secondary load, and it still checks `live`:
  ```js
  AreaService.fetchSummary()
    .then((res) => live && setSummary(res?.data || null))
    .catch(() => live && setSummary({ failed: true }));
  ```
- **Don't call a component's `getItems()` from an effect and switch off `react-hooks/exhaustive-deps` or `set-state-in-effect` to get it past lint.** That is the legacy pattern: it has no `live` flag, so a slow, stale answer overwrites the newer one. Move the load into the hook.
- **Loading two things at once** (a page and its tab counts): use `Promise.allSettled`, so one failure doesn't hide the other. Each result is applied on its own, and one toast reports the first failure (recipe in references).
- **Optimistic updates roll back.** When the screen changes before the request (a like, a toggle, a reorder), keep the previous value, and on failure put it back and raise an error toast. Never send a request without handling its failure.

### 5.2 Redux slices
Redux holds **app-wide session state** (`cognito`, `authentication`), the **notification and page-loader state**, and **UI flags shared across a feature's components**: open modals, tab counts, and a "load again" counter. It doesn't hold server lists.

```jsx
// features/<area>/service/<Area>Slice.jsx
import { createSlice } from "@reduxjs/toolkit";

/**
 * Redux slice for the <Area> screen: the counts on its tabs, the open modals, and a counter that
 * tells the lists to load again after a change.
 */
const areaSlice = createSlice({
  name: "area",
  initialState: {
    pendingCount: null,
    decisionModal: { open: false, itemId: null, action: "" },
    fetchItems: 0,
  },
  reducers: {
    setPendingCount(state, action) {
      state.pendingCount = action.payload.count;
    },
    openDecisionModal(state, action) {
      state.decisionModal = { open: true, itemId: action.payload.itemId, action: action.payload.action };
    },
    closeDecisionModal(state) {
      state.decisionModal = { open: false, itemId: null, action: "" };
    },
    setFetchItems(state) {
      state.fetchItems += 1;
    },
  },
});

export const areaSliceActions = areaSlice.actions;
export default areaSlice.reducer;
```

Rules:
1. **Name the file `<Area>Slice.jsx`** (PascalCase, `.jsx`, the house convention). Export `<area>SliceActions` and the reducer as default.
2. **Reducers are methods** (`setX(state, action) {}`) and take an **object payload** (`action.payload.count`, `action.payload.modal`).
3. **Register the reducer in `store/store.js`** with a one-line comment saying what it is for.
4. **`createAsyncThunk` is for the Cognito session slice** (`userLoginAsync`, `getSessionAsync`, `refreshSessionAsync`, …). Don't add thunks for feature data: use the hook above.
5. The store sets `serializableCheck: false` because the Cognito payload holds a `CognitoUserSession`. Leave it.
6. **Older apps (ChorusCare) keep lists, filters, sort order and date ranges in Redux** and define reducers as arrow functions. Leave that code as it is, but don't extend it: new screens use a hook.

### 5.3 Passing data between components
- **Data flows hook → screen → props.** The screen reads its hook once and gives each child the values it shows and the callbacks it raises (`onSaved`, `onClose`, `onPage`).
- **Don't use React Context for feature data or navigation helpers.** The apps don't use it. If a value passes through more than two levels, let the child call the feature hook itself, or keep a shared UI flag in the feature's slice.
- **Read Redux with a narrow selector** (`useSelector((state) => state.cognito.token)`), not the whole slice.
- **Take the signed-in user from `useCurrentUser()`** (`lib/currentUser.js`). Never hard-code a user or pass one around in props.

---

## 6. Notifications and blocking loaders

Raise every success and error message through the notification slice. `App.jsx` turns it into a toast:

```js
dispatch(notificationSliceActions.setNotification({ open: true, severity: "success", message: `${name} saved` }));
dispatch(notificationSliceActions.setNotification({ open: true, severity: "error", message: apiError(err, "The changes could not be saved") }));
// an id replaces the toast already showing with that id, so repeated saves show one toast
dispatch(notificationSliceActions.setNotification({ open: true, severity: "success", message: "Saved", id: "plan-save" }));
```

- `severity` is `success`, `error`, `info` or `warning`. The `<Toaster position="top-center" />` is mounted once in `App.jsx`.
- **Write messages as sentences about the thing:** "Invite sent to x@y", "People could not be loaded", "The invite could not be sent". Never show raw server text or a stack trace. `apiError` already falls back to your message.
- `notificationSliceActions.handleLoading({ loading: true })` shows a **full-page blocking loader**. Use it only for operations that must not be interrupted, and always clear it in `finally`.
- In the four newest apps a few files call sonner's `toast()` directly. That is acceptable there, but the slice is the default because it works in every app.

---

## 7. Routing and access

```jsx
// App.jsx: the shape every app shares
return (
  <>
    {loading ? <FullPageLoader /> : allowed ? <AppRouter /> : <AuthRouter />}
    {pageLoading && <BlockingOverlay />}
    <Toaster position="top-center" />
  </>
);
```

```jsx
// routes/AppRouter.jsx
export default function AppRouter() {
  return (
    <Routes>
      <Route element={<AppShell />}>
        <Route path="/" element={<Navigate to="/people" replace />} />
        <Route path="/people" element={<People />} />
        <Route path="/people/:userId" element={<PersonProfile />} />
        {/* HR managers only, as its API (/api/breezethru/admin) */}
        <Route path="/requests" element={<RequireRole roles={HR_ROLES}><Requests /></RequireRole>} />
        {/* It was /perks until 23 Oct 2026: old links keep their ?tab= */}
        <Route path="/perks" element={<PerksRedirect />} />
        <Route path="*" element={<Navigate to="/people" replace />} />
      </Route>
    </Routes>
  );
}
```

Rules:
1. **`App.jsx` decides signed in or not** from `state.cognito` (`loading`, `isAuthenticated`, `token`). There is no `ProtectedRoute`, no `/login` route inside `AppRouter`, and no `state.auth`.
2. **Every signed-in screen sits inside the `AppShell` layout route.**
3. **Roles come from the access token's `cognito:groups`**, upper-cased to match the backend's `ROLE_*` names. Use the app's `lib/<app>Access.js` helpers (`hasAnyRole`, `useHasAnyRole`, role lists like `HR_ROLES`). Wrap a narrower page in `<RequireRole roles={…}>`. The UI only decides what is drawn: the backend enforces access on its own.
4. **Unknown paths redirect home** (`<Route path="*" element={<Navigate to="…" replace />} />`). The apps have no 404, 403 or 500 pages, and a 403 is shown in-page (section 4.3).
5. **When a route is renamed, keep the old URL** as a `<Navigate replace>` redirect that carries `location.search` along.
6. **`AuthRouter` nests the sign-in screens under one frame** (`<Route path="/" element={<AuthHome />}>`): login, forgot-password, `verify-otp/:email`, `setup-account/:email`, and the success screens. Users are invited, so there is no self-registration.
7. **Lazy loading** (`lazy()` + `Suspense`) is worth it for heavy editors (BlockSuite, Milkdown, FullCalendar) and for new apps. Existing routers import screens directly, and that is fine.

### URL state
Tabs, selected items and filters that someone may reload or share go in the query string, through `hooks/useUrlParam.js`:

```js
const [tab, setTab] = useUrlParam("tab");          // ?tab=feedback
const [requestId, setRequestId] = useUrlParam("request");
```

Setting `null` or `""` removes the parameter and leaves the others alone.

---

## 8. Forms (Formik + Yup)

```jsx
import { useState } from "react";
import { useDispatch } from "react-redux";
import { useFormik } from "formik";
import * as Yup from "yup";
import { Loader2 } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import AreaService from "./service/AreaService";
import { apiError } from "@/service/AdminService";
import { notificationSliceActions } from "@/service/NotificationSlice";

const schema = Yup.object({
  name: Yup.string().trim().required("Enter a name.").max(80, "At most 80 characters."),
  email: Yup.string().trim().required("Enter an email.").email("Enter a valid email."),
});

function FieldError({ formik, name }) {
  const error = formik.submitCount > 0 && formik.errors[name];
  return error ? (
    <p id={`${name}-error`} role="alert" className="text-xs font-medium text-destructive">
      {error}
    </p>
  ) : null;
}

function Required({ htmlFor, children }) {
  return (
    <Label htmlFor={htmlFor}>
      {children} <span className="text-destructive" aria-hidden="true">*</span>
    </Label>
  );
}

/**
 * Add item and Edit item. Errors show under each field once Save is pressed; the backend's own
 * error is shown as a toast and the dialog stays open. While saving, the dialog cannot be closed.
 *
 * Mounted with a fresh `key` each time it opens, so it always starts from the item (or blank).
 *
 * @param {object} props
 * @param {object|null} props.item the item to edit, or null to add one
 * @param {() => void} props.onClose
 * @param {(item: object) => void} props.onSaved called with the saved item
 */
export default function ItemFormDialog({ item, onClose, onSaved }) {
  const dispatch = useDispatch();
  const [saving, setSaving] = useState(false);
  const editing = Boolean(item);

  const formik = useFormik({
    initialValues: { name: item?.name || "", email: item?.email || "" },
    validationSchema: schema,
    validateOnChange: true,
    validateOnBlur: false,
    onSubmit: async (v, helpers) => {
      const body = { name: v.name.trim(), email: v.email.trim() };
      setSaving(true);
      try {
        const res = editing ? await AreaService.updateItem(item.id, body) : await AreaService.addItem(body);
        dispatch(notificationSliceActions.setNotification({ open: true, severity: "success", message: `${res.data.name} saved` }));
        onSaved(res.data);
      } catch (err) {
        if (err?.response?.status === 409) {
          helpers.setFieldError("email", apiError(err, "This email is already in use."));
          return;
        }
        dispatch(notificationSliceActions.setNotification({ open: true, severity: "error", message: apiError(err, "The item could not be saved") }));
      } finally {
        setSaving(false);
      }
    },
  });

  const field = (name) => ({
    id: name,
    name,
    value: formik.values[name],
    onChange: formik.handleChange,
    "aria-invalid": (formik.submitCount > 0 && Boolean(formik.errors[name])) || undefined,
    "aria-describedby": formik.submitCount > 0 && formik.errors[name] ? `${name}-error` : undefined,
  });

  return (
    <Dialog open onOpenChange={(open) => !open && !saving && onClose()}>
      <DialogContent
        className="gap-0 p-0 sm:max-w-lg"
        onInteractOutside={(e) => saving && e.preventDefault()}
        onEscapeKeyDown={(e) => saving && e.preventDefault()}
      >
        <DialogHeader className="border-b px-6 py-4 text-left">
          <DialogTitle>{editing ? "Edit item" : "Add item"}</DialogTitle>
          <DialogDescription>{editing ? "Change its name or email." : "It is added straight away."}</DialogDescription>
        </DialogHeader>

        <form onSubmit={formik.handleSubmit} noValidate>
          <fieldset disabled={saving} className="max-h-[65vh] min-w-0 space-y-4 overflow-y-auto px-6 py-5">
            <div className="space-y-1.5">
              <Required htmlFor="name">Name</Required>
              <Input {...field("name")} maxLength={80} autoComplete="off" autoFocus />
              <FieldError formik={formik} name="name" />
            </div>
            <div className="space-y-1.5">
              <Required htmlFor="email">Email</Required>
              <Input {...field("email")} type="email" autoComplete="off" />
              <FieldError formik={formik} name="email" />
            </div>
          </fieldset>

          <DialogFooter className="border-t px-6 py-4">
            <Button type="button" variant="outline" onClick={onClose} disabled={saving}>
              Cancel
            </Button>
            <Button type="submit" disabled={saving}>
              {saving && <Loader2 className="size-4 animate-spin" />}
              {editing ? "Save" : "Add item"}
            </Button>
          </DialogFooter>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

### Form rules
1. **Use Formik (`useFormik`) + Yup.** Put the schema at module scope, with messages as short sentences ("Enter a first name.", "At most 40 characters."). Use React Hook Form + Zod **only** in an app or file that already does.
2. **Validate on change, not on blur, and show errors only after the first submit** (`formik.submitCount > 0`).
3. **Every input gets `<Label htmlFor>`, plus `aria-invalid` and `aria-describedby` pointing at its error.** The error is `<p id="x-error" role="alert" className="text-xs font-medium text-destructive">`. A `field(name)` helper spreads these props.
4. **Mark required fields** with `<span className="text-destructive" aria-hidden="true">*</span>`.
5. **While saving:** wrap the inputs in `<fieldset disabled={saving}>`, disable both buttons, show `Loader2` in the submit button, and block Escape and outside clicks.
6. **Server errors:** map a field-specific code (409) onto the field with `helpers.setFieldError`, and send everything else to a toast. The dialog stays open on failure.
7. **Trim strings before sending.** Send optional fields as `undefined` when empty.
8. **Groups of checkboxes or radios** use `<fieldset>` + `<legend>`.

---

## 9. Styling, theming and responsive layout

### Tailwind version: check first
- **Tailwind 4** (`@tailwindcss/vite` in `package.json`, `@import "tailwindcss"` in `index.css`) is used by the newer apps. There is **no `tailwind.config.js`**. Tokens are mapped in `@theme inline { --color-x: var(--x); }`, and dark mode is `@custom-variant dark (&:is(.dark *));`. Add a new token to `index.css`, never to a config file.
- **Tailwind 3** (`tailwind.config.js` + `postcss.config.js`) is used by the older apps. Tokens and breakpoints live in `theme.extend`.

### Tokens over hex
- **Use semantic tokens:** `bg-background`, `text-foreground`, `text-muted-foreground`, `border-border`, `bg-primary`, `text-destructive`, plus the Breezeware surfaces where the app has them (`bg-page`, `bg-raise`, `bg-soft`, `border-line`, `text-mute`, `text-brand-text`).
- **Status colours are token pairs:** `bg-[var(--b-done)] text-[var(--t-done)]`, `--b-review` / `--t-review`, and so on. Reuse the pair for the status's meaning.
- Hard-coded hex (`text-[#1b3764]`) is common in existing files. **Don't add more.** Use the token, or add one to `index.css` with a light and a dark value.
- **Dark mode** is a `.dark` class on `<html>`, set before the first paint from `lib/appearance.js` (light, dark or match device). The brand colour can be changed through `data-color-theme` (`lib/colorTheme.js`). Give every new colour a `dark:` counterpart or a token that already has one. Sign-in screens are light-only.
- **Prefer Tailwind classes over `style={{}}`.** `style` is acceptable for truly dynamic values (a computed width or a colour from data). A feature `.css` file is acceptable for editors and complex layouts (BlockSuite, Milkdown, calendars), with classes prefixed by area (`.adm-…`).

### Responsive layout
- **Mobile-first classes, and test down to 360px.** Nothing may scroll sideways except a table inside its own `overflow-x-auto`.
- **The shell (`components/layout/AppShell.jsx`) is a top bar, not a sidebar.** Sections sit in the bar from **1000px**, in a fixed **bottom bar** from 768 to 999px, and in a side `Sheet` below 768px. Full-height pages subtract the bar and the bottom nav: `min-h-[calc(100vh-73px-var(--bottom-nav-h))]`.
- **Breakpoints:** Tailwind's `sm`, `md`, `lg` and `xl`, plus arbitrary `min-[1000px]:` for the shell's switch. Use a custom screen (`xs`, `3xl` …) only if the app's Tailwind config defines it.
- **Common page grid:** title on the left, tabs in the centre and the primary action on the right, with the tabs dropping under the title on narrow screens. Toolbars become `flex-col` below `sm`. Selects are `w-full sm:w-[170px]`.
- **Tables:** shadcn `Table` inside `overflow-hidden rounded-lg`. On narrow screens either scroll horizontally or hide secondary columns (`hidden md:table-cell`).

---

## 10. The four states, empty states and confirmations

**Every component that loads data handles loading, error, empty and success.**

| State | House pattern |
|---|---|
| Loading | `Skeleton` rows or cards that match the final layout. Use `Loader2 className="size-4 animate-spin"` only inside buttons and blocking overlays |
| Error | a toast through the notification slice, with the view falling back to its empty value. A 403 shows the "You don't have access" panel |
| Empty | an `<Area>Empty.jsx` / `EmptyState` (icon in a circle, a quiet title, a short explanation, an optional action), passed to the table as `empty={<AreaEmpty />}` |
| Success | the content, with a count line such as `Showing 1 - 12 of 40 entries` (`aria-live="polite"`) and `<Pager>` under tables |

- **Destructive or consequential actions** (suspend, delete, decline, revoke) go through a shadcn `AlertDialog`. Describe the action in a config object (`title`, `body`, `confirm`, `done`, `failed`, `run`, `danger`). The confirm button is a plain `Button`, not `AlertDialogAction`, because `AlertDialogAction` closes the dialog before the request finishes. The dialog can't be dismissed while busy.
- **Pagination** uses `components/Pager.jsx` (zero-based `page`, `totalPages`, `onPage`, `disabled` while loading). Never load a whole list at once.
- **Search** waits 300 ms (`useDebouncedValue` or the timeout in the hook) and has a clear button (`aria-label="Clear search"`).
- **Row actions** go in a `DropdownMenu` behind an icon `Button` with `aria-label={\`Actions for ${name}\`}`, showing only the actions valid for the row's status.

---

## 11. Dates and shared formatting

- **Use the app's shared formatters** in `service/Utils.jsx` (or `lib/`) before writing new ones: `formatDate` (the house format **"14 Sep 2026"**), `formatDateRange` ("20 Oct 2026 – 21 Oct 2026"), `toIsoDate`, `plural`, `nameInitials` and `avatarHue`.
- **Send date-only values as local `yyyy-MM-dd`** with `toIsoDate`. **Never use `toISOString()` for a date**, because it shifts across time zones. `new Date().toISOString().slice(0, 10)` is the same bug: in India it gives yesterday's date until 5:30 AM, so "can't be in the future" checks reject today. Today is `toIsoDate(new Date())`. `toISOString()` is right only for an instant (`createdOn`, `updatedAt`).
- Use **dayjs** for date arithmetic. Don't add date-fns or moment to an app that doesn't have them.

---

## 12. Accessibility

- **Interactive elements are `<button type="button">` or `<a>`/`<Link>`, never `<div onClick>`.** Icon-only buttons need an `aria-label`.
- **Use shadcn `Dialog` / `AlertDialog` / `Sheet`** for focus trapping and Escape. Give each a `DialogTitle` (or `SheetTitle`) and a description.
- Use `aria-live="polite"` for counts and status that change, `role="status"` with `aria-label` for loaders, `aria-current="page"` for the current page or tab, and `aria-expanded` on disclosure buttons.
- **Images:** meaningful `alt`, or `alt=""` for decoration. Mark decorative icons in text `aria-hidden="true"`.
- **Text contrast** at least 4.5:1, in both light and dark.
- **Touch targets** at least 32px in dense tables and 40px or more elsewhere on mobile. Don't use `tabIndex` greater than 0.

---

## 13. Code style

| Aspect | Rule |
|---|---|
| Quotes | **Double** quotes in JS and JSX |
| Semicolons | yes |
| Trailing commas | in multi-line literals and arguments |
| Indent | 2 spaces |
| Import order | React, then libraries (react-redux, react-router-dom, formik/yup, lucide-react), then `@/components/ui/*`, then `@/` app modules, then `./` siblings |
| Icons | `size-4` / `size-5`; spacing comes from the parent's `gap-*`, not `mr-2` on the icon |
| Comments | explain **why** and the behaviour a reader can't see (what the server rejects, why a value is `undefined`, why a layout effect). Prose JSDoc on every exported component, hook and service method |
| Logging | no `console.log` in committed code. `catch {}` with a comment saying why it is safe to ignore |
| Formatting | there is no Prettier. Keep to the surrounding file and run `npm run lint` |
| ESLint | `react-hooks` rules must pass (complete dependency arrays). `react-refresh/only-export-components` is switched off for `components/ui/**` and the `service/` and `store/` files |

### Naming

| Element | Convention | Example |
|---|---|---|
| Component / screen file | PascalCase `.jsx` | `PeopleTable.jsx`, `screens/people/People.jsx` |
| Dialog | `<Thing>FormDialog`, `<Thing>ActionDialog`, `<Thing>Dialog` | `PersonFormDialog.jsx` |
| Empty state | `<Area>Empty` | `PeopleEmpty.jsx` |
| API service | PascalCase `<Area>Service.jsx` | `RequestService.jsx` |
| Redux slice | PascalCase `<Area>Slice.jsx`, actions `<area>SliceActions` | `RequestSlice.jsx` |
| Hook | `use<Thing>.js` | `usePeople.js`, `useUrlParam.js` |
| Pure helpers and constants | camelCase `.js` | `peopleOptions.js`, `requestConstants.js`, `reviewModel.js` |
| shadcn primitive | kebab-case `.jsx` | `components/ui/alert-dialog.jsx` |
| Constants | UPPER_SNAKE_CASE | `PAGE_SIZE`, `HR_ROLES` |
| Env | `VITE_` + purpose | `VITE_BASE_URL_ADMIN`, `VITE_COGNITO_CLIENT_ID` |

---

## 14. Never

- Don't import `prop-types` or add PropTypes.
- Don't create an axios instance, store tokens in `localStorage`, or redirect to `/login` from an interceptor.
- Don't build query strings by concatenation, or send `""`, `" "` or `"undefined"` as a filter.
- Don't put a secret (client secret, API key) anywhere in frontend code or `.env`.
- Don't call third-party services from the frontend without a documented need.
- Don't add new MUI, a new data library (React Query, SWR) or a new form library to an app that doesn't use one.
- Don't put server lists in Redux, or add `createAsyncThunk` for feature data.
- Don't create `app/`, `pages/`, `layouts/`, barrel `index.js` files or folder-per-component layouts.
- Don't use `toISOString()` for a date-only value.
- Don't show raw server errors or stack traces.
- Don't edit an app's auth files (`lib/authStorage.js`, `CognitiSlice`, `screens/auth/` or `features/auth/`) unless the task is about auth.
- Don't write an OAuth or password sign-in call, or put a `client_secret` in a URL.
- Don't ship mock data, mock users or a fake fallback for a failed request.
- Don't use React Context for feature state, or give a feature its own router.
- Don't switch off `react-hooks` lint rules to get a data load past them.
- Don't send an optimistic request without rolling back on failure.
- Don't add a second error helper, notification slice or copy of a shadcn primitive.
- Don't go looking outside this skill for how to build something (section 0).

## 15. What `references/component-patterns.md` holds

Read the matching recipe there **before** writing any of these:

| Writing | Recipe |
|---|---|
| `main.jsx`, `App.jsx`, `store/store.js` | the app wiring |
| the Cognito slice, `cognito-config.js` | the session thunks and reducer |
| `lib/authStorage.js` | the shared session storage (use it unchanged) |
| `registerAuthInterceptors`, `apiError` | the app-wide service |
| `NotificationSlice` | the notification slice |
| `lib/currentUser.js`, `lib/<app>Access.js`, `RequireRole`, route redirects | access |
| `AppShell` | the layout shell |
| `AuthRouter` | the sign-in routes |
| a list screen, a table, an empty state, a confirm dialog, `Pager` | screens and tables |
| `useUrlParam`, `useDebouncedValue` | hooks |
| two loads at once, an optimistic update | parallel loads and optimistic updates |
| `lib/appearance.js`, `lib/colorTheme.js`, tokens in `index.css` or `tailwind.config.js` | theming |
| `formatDate`, `toIsoDate`, `plural` | formatters |
| `vite.config.js`, `jsconfig.json`, `.env` | config |
