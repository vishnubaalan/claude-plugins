# Component Patterns & Recipes Reference

The complete code for Breezeware's house pattern. Use these recipes as they are: the
only things to adapt are the facts listed in SKILL.md §0 (Tailwind version, base URL
variable, paging names, role constants). If the app already has a piece, import it. If
it doesn't, create it from here at the path in the recipe's heading comment.

---

## main.jsx

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import { Provider } from "react-redux";
import store from "./store/store";
import "./index.css";
import App from "./App.jsx";
import { applyColorTheme, getStoredColorTheme } from "./lib/colorTheme";
import { applyAppearance, getStoredAppearance } from "./lib/appearance";

// The theme colour and light or dark are applied before the first paint
applyColorTheme(getStoredColorTheme());
applyAppearance(getStoredAppearance());

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <Provider store={store}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </Provider>
  </StrictMode>
);
```

## App.jsx: session, interceptors, routers, toasts

```jsx
import { useEffect, useLayoutEffect } from "react";
import { useDispatch, useSelector } from "react-redux";
import { toast } from "sonner";
import { Loader2 } from "lucide-react";
import AppRouter from "./routes/AppRouter";
import AuthRouter from "./routes/AuthRouter";
import { getSessionAsync, refreshSessionAsync } from "./service/CognitiSlice";
import { notificationSliceActions } from "./service/NotificationSlice";
import { registerAuthInterceptors } from "./service/AdminService";
import { Toaster } from "@/components/ui/sonner";
import { hasAdminAccess } from "@/lib/adminAccess";

function App() {
  const dispatch = useDispatch();
  const loading = useSelector((state) => state.cognito.loading);
  const isAuthenticated = useSelector((state) => state.cognito.isAuthenticated);
  const token = useSelector((state) => state.cognito.token);
  // Checked on every token, so a refresh that drops the role closes the app too
  const allowed = isAuthenticated && hasAdminAccess(token);
  const notification = useSelector((state) => state.notification);
  const pageLoading = useSelector((state) => state.notification.loading);

  // Restore the Cognito session on load, refreshing it if it has expired
  useEffect(() => {
    dispatch(getSessionAsync())
      .unwrap()
      .catch(() => dispatch(refreshSessionAsync()));
  }, [dispatch]);

  // A layout effect, so the interceptors are in place before any screen's first request
  useLayoutEffect(() => {
    if (!token) return;
    return registerAuthInterceptors(token, () =>
      dispatch(refreshSessionAsync())
        .unwrap()
        .then((res) => res?.headers?.Authorization)
    );
  }, [token, dispatch]);

  // Show notifications raised through NotificationSlice as toasts
  useEffect(() => {
    if (!notification.open) return;
    const show = toast[notification.severity] || toast;
    show(notification.message, notification.id ? { id: notification.id } : undefined);
    dispatch(notificationSliceActions.closeNotification());
  }, [notification, dispatch]);

  return (
    <>
      {loading ? (
        <div className="flex min-h-screen items-center justify-center">
          <Loader2 className="size-8 animate-spin text-primary" />
        </div>
      ) : allowed ? (
        <AppRouter />
      ) : (
        <AuthRouter />
      )}
      {pageLoading && (
        <div
          className="fixed inset-0 z-[100] flex cursor-wait items-center justify-center bg-black/20 backdrop-blur-[1px] dark:bg-black/40"
          role="status"
          aria-live="polite"
          aria-label="Please wait"
        >
          <div className="flex size-14 items-center justify-center rounded-full border border-border bg-background">
            <Loader2 className="size-7 animate-spin text-primary" />
          </div>
        </div>
      )}
      <Toaster position="top-center" />
    </>
  );
}

export default App;
```

## registerAuthInterceptors (app-wide service)

```jsx
// service/AdminService.jsx (top of file)
import axios from "axios";

/**
 * Attaches the bearer token to every axios request and retries once with a refreshed token when
 * the backend answers 401. Call it from App.jsx in a layout effect keyed on the token, and return
 * the cleanup it gives back so the interceptors are ejected when the token changes.
 *
 * @param {string} token current access token
 * @param {() => Promise<string>} refreshToken resolves to a new access token
 * @returns {() => void} ejects both interceptors
 */
export const registerAuthInterceptors = (token, refreshToken) => {
  const requestInterceptor = axios.interceptors.request.use(
    (config) => {
      config.headers.Authorization = `Bearer ${token}`;
      return config;
    },
    (error) => Promise.reject(error)
  );

  const responseInterceptor = axios.interceptors.response.use(
    (response) => response,
    async (error) => {
      const originalRequest = error.config;
      // If unauthorized and not retried already
      if (error.response?.status === 401 && refreshToken && !originalRequest._retry) {
        originalRequest._retry = true;
        try {
          const newToken = await refreshToken();
          axios.defaults.headers.common.Authorization = `Bearer ${newToken}`;
          originalRequest.headers.Authorization = `Bearer ${newToken}`;
          return axios(originalRequest);
        } catch (refreshErr) {
          return Promise.reject(refreshErr);
        }
      }
      return Promise.reject(error);
    }
  );

  return () => {
    axios.interceptors.request.eject(requestInterceptor);
    axios.interceptors.response.eject(responseInterceptor);
  };
};

/**
 * The message to show for a failed request: the backend's first detail, then its message, then
 * the fallback. DynamoException puts the readable text in `details[0]`.
 */
export const apiError = (err, fallback) =>
  err?.response?.data?.details?.[0] || err?.response?.data?.message || fallback;
```

## Cognito slice and config

```js
// src/cognito-config.js
export const cognitoConfig = {
  userPoolId: import.meta.env.VITE_COGNITO_USER_POOL_ID,
  clientId: import.meta.env.VITE_COGNITO_CLIENT_ID,
};
```

```jsx
// service/CognitiSlice.jsx (the session part; forgot-password, confirm and resend-OTP thunks follow the same shape)
import { createSlice, createAsyncThunk } from "@reduxjs/toolkit";
import { CognitoUser, CognitoUserPool, AuthenticationDetails } from "amazon-cognito-identity-js";
import { cognitoConfig } from "../cognito-config";
import { createAuthStorage } from "../lib/authStorage";

export const UserPool = new CognitoUserPool({
  UserPoolId: cognitoConfig.userPoolId,
  ClientId: cognitoConfig.clientId,
  // the session is kept in the cookie the staff apps share (lib/authStorage), so signing in to one signs in to all
  Storage: createAuthStorage(cognitoConfig.clientId, cognitoConfig.userPoolId),
});

/** A session Cognito refuses to renew for good ends in every app, not only this one. */
const dropIfDead = (err) => {
  if (UserPool.storage.isDead?.(err)) UserPool.storage.endSession();
};

/** The user's attributes as { name: value }. */
const getUserAttributes = (user) =>
  new Promise((resolve, reject) =>
    user.getUserAttributes((err, attributes) =>
      err ? reject(err) : resolve(Object.fromEntries(attributes.map((a) => [a.Name, a.Value])))
    )
  );

export const userLoginAsync = createAsyncThunk("cognito/userLogin", async ({ email, password }) => {
  // Storage passed too: without it CognitoUser falls back to bare localStorage and never shares the session
  const user = new CognitoUser({ Username: email, Pool: UserPool, Storage: UserPool.storage });
  const authDetails = new AuthenticationDetails({ Username: email, Password: password });
  return new Promise((resolve, reject) => {
    user.authenticateUser(authDetails, {
      onSuccess: (data) => resolve({ token: data.getAccessToken().getJwtToken(), user: data }),
      onFailure: (err) => reject({ message: err.message }),
      newPasswordRequired: () => reject({ message: "New Password Required" }),
    });
  });
});

/** The stored session on load, refreshed first when its access token has expired. */
export const getSessionAsync = createAsyncThunk("auth/getSession", async (_, { rejectWithValue }) => {
  const user = UserPool.getCurrentUser();
  if (!user) return rejectWithValue({ message: "No current user found" });

  return new Promise((resolve, reject) => {
    user.getSession(async (err, session) => {
      if (err) {
        dropIfDead(err);
        return reject(rejectWithValue(err));
      }
      try {
        let current = session;
        if (session.getAccessToken().getExpiration() <= Math.floor(Date.now() / 1000)) {
          current = await new Promise((res, rej) =>
            user.refreshSession(session.getRefreshToken(), (e, s) => (e ? rej(e) : res(s)))
          );
        }
        const attributes = await getUserAttributes(user);
        resolve({ isAuthenticated: true, user: attributes, headers: { Authorization: current.getAccessToken().getJwtToken() } });
      } catch (e) {
        // the stored token was refused (revoked when the session ended in another app, or invalid): sign in again
        dropIfDead(e);
        reject(rejectWithValue({ message: "Session expired" }));
      }
    });
  });
});

/** A new access token from the refresh token; the axios interceptor calls it on a 401. */
export const refreshSessionAsync = createAsyncThunk("auth/refreshSession", async (_, { rejectWithValue }) => {
  const user = UserPool.getCurrentUser();
  if (!user) return rejectWithValue({ message: "No current user" });

  return new Promise((resolve, reject) => {
    user.getSession((err, session) => {
      if (err) {
        dropIfDead(err);
        return reject(rejectWithValue(err));
      }
      user.refreshSession(session.getRefreshToken(), async (e, refreshed) => {
        if (e) {
          dropIfDead(e);
          return reject(rejectWithValue(e));
        }
        try {
          const attributes = await getUserAttributes(user);
          resolve({ isAuthenticated: true, user: attributes, headers: { Authorization: refreshed.getAccessToken().getJwtToken() } });
        } catch (attrErr) {
          dropIfDead(attrErr);
          reject(rejectWithValue(attrErr));
        }
      });
    });
  });
});

const cognitoSlice = createSlice({
  name: "cognito",
  initialState: { token: null, isAuthenticated: false, user: null, error: null, loading: true, loginError: null, newPasswordRequired: false },
  reducers: {
    userLogout: (state) => {
      // signs out of every app: clears the shared session and revokes its refresh token
      UserPool.storage.endSession?.();
      state.token = null;
      state.isAuthenticated = false;
      state.user = null;
    },
  },
  extraReducers: (builder) => {
    const signedIn = (state, action) => {
      state.token = action.payload.headers.Authorization;
      state.isAuthenticated = action.payload.isAuthenticated;
      state.user = action.payload.user;
      state.loading = false;
    };
    const signedOut = (state, action) => {
      state.loading = false;
      state.isAuthenticated = false;
      state.error = action.payload || action.error.message;
    };
    builder
      .addCase(userLoginAsync.fulfilled, (state, action) => {
        state.token = action.payload.token;
        state.isAuthenticated = true;
        state.user = action.payload;
        state.loginError = null;
        state.loading = false;
      })
      .addCase(userLoginAsync.rejected, (state, action) => {
        if (action.error.message === "New Password Required") state.newPasswordRequired = true;
        else state.loginError = action.error.message || "An error occurred.";
        state.loading = false;
      })
      .addCase(getSessionAsync.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(getSessionAsync.fulfilled, signedIn)
      .addCase(getSessionAsync.rejected, signedOut)
      .addCase(refreshSessionAsync.fulfilled, signedIn)
      .addCase(refreshSessionAsync.rejected, signedOut);
  },
});

export const cognitoSliceActions = cognitoSlice.actions;
export default cognitoSlice.reducer;
```

## Shared session storage (`lib/authStorage.js`)

The Cognito session lives in one cookie that the staff apps share, so signing in to any of them
signs in to all of them, and signing out of one signs out of all of them. **Use this file exactly
as it is:** every staff app has a byte-identical copy, and the copies must stay identical. If you
change it, change every app's copy in the same commit. `CognitiSlice` passes it to the user pool
(`Storage: createAuthStorage(clientId, userPoolId)`). Set `VITE_AUTH_COOKIE_DOMAIN` to the parent
domain (`.example.com`) when the apps are deployed on subdomains; leave it unset on localhost.

An app that is **not** part of the shared sign-in (a customer portal with its own user pool) does
not use this file: leave out the `Storage` option on `CognitoUserPool` and `CognitoUser`. The
optional calls in the slice (`isDead?.`, `endSession?.`) keep it working either way.

```js
// lib/authStorage.js
import { CookieStorage } from "amazon-cognito-identity-js";

/**
 * Where the Cognito session is kept: one session for the five staff apps (Flywheel, BreezeThru, Leap,
 * Admin and Atlas), so signing in to any of them signs in to all of them (23 Oct 2026).
 *
 * The session lives only in a cookie the five apps share; nothing is kept in an app's own localStorage.
 * One account is signed in per browser:
 *
 * - Signing in to any app makes that account the session everywhere. If another account was signed in,
 *   its tokens are removed and its refresh token is revoked, so its session ends in every app.
 * - Signing out of any app (endSession) clears the session and revokes its refresh token: every app
 *   falls back to its sign-in screen, and none can renew the session.
 * - A session Cognito refuses to renew (expired or revoked refresh token) is cleared the same way
 *   (endSession, through isDead), so every app signs out together rather than each failing on its own.
 * - Every open app watches the session: when it is shown, when it gets focus, and every 2 seconds. When
 *   another app signs in as someone else, or signs out, it reloads, so nothing the previous account
 *   loaded is left on screen, and opens as the new account or on its sign-in screen.
 *
 * Until 16 Oct 2026 each app kept its own session in localStorage; from then to 23 Oct the four newer
 * apps kept their own session and shared only "the last sign-in" in this cookie, and Flywheel kept its
 * own. A session left in localStorage by those versions moves into the cookie once, if the cookie has
 * none, and is then removed.
 *
 * Cookies are shared by every port of a host (localhost, or one machine's address on the LAN), so no
 * domain is set there. Deployed on subdomains, VITE_AUTH_COOKIE_DOMAIN is their parent (".example.com").
 *
 * Each of the five apps keeps a copy of this file (src/lib/authStorage.js); change them together.
 */
const secure = window.location.protocol === "https:" || window.location.hostname === "localhost";

// how often an open app checks whether the session was changed by another app
const WATCH_MS = 2000;

/**
 * @param {string} clientId the Cognito app client the five apps share
 * @param {string} userPoolId the user pool, for its region (revoking a refresh token calls Cognito directly)
 */
export function createAuthStorage(clientId, userPoolId) {
  const prefix = `CognitoIdentityServiceProvider.${clientId}.`;
  const lastUserKey = `${prefix}LastAuthUser`;
  const region = String(userPoolId || "").split("_")[0] || "us-east-1";
  const shared = new CookieStorage({
    domain: import.meta.env.VITE_AUTH_COOKIE_DOMAIN || undefined,
    path: "/",
    // the refresh token's own lifetime decides how long a session lasts; this only bounds the cookie
    expires: 30,
    secure,
    sameSite: "strict",
  });

  const sharedUser = () => shared.getItem(lastUserKey) || null;
  /** The session's cookies, by name. */
  const sharedKeys = () =>
    document.cookie
      .split(";")
      .map((c) => decodeURIComponent(c.split("=")[0].trim()))
      .filter((name) => name.startsWith(prefix));
  /** The account a key belongs to: "<prefix><user>.idToken" → user; LastAuthUser → null. */
  const userOf = (key) => (key === lastUserKey ? null : key.slice(prefix.length, key.lastIndexOf(".")));

  /**
   * Revokes a refresh token with Cognito, so no app can renew its session. Fire and forget: the page may
   * be about to reload (keepalive lets the request finish), and a client without token revocation
   * enabled refuses it, which changes nothing here.
   */
  const revoke = (refreshToken) => {
    if (!refreshToken) return;
    try {
      fetch(`https://cognito-idp.${region}.amazonaws.com/`, {
        method: "POST",
        headers: {
          "Content-Type": "application/x-amz-json-1.1",
          "X-Amz-Target": "AWSCognitoIdentityProviderService.RevokeToken",
        },
        body: JSON.stringify({ Token: refreshToken, ClientId: clientId }),
        keepalive: true,
      }).catch(() => {});
    } catch {
      // offline or blocked: the token lapses on its own
    }
  };

  /** Ends one account's session in this browser: revokes its refresh token and removes its cookies. */
  const dropUser = (user) => {
    revoke(shared.getItem(`${prefix}${user}.refreshToken`));
    sharedKeys()
      .filter((k) => userOf(k) === user)
      .forEach((k) => shared.removeItem(k));
  };

  // Once: a session the previous versions left in this app's localStorage moves into the cookie when
  // the cookie has none (so nobody is signed out by the change), then leaves localStorage
  try {
    const localKeys = Object.keys(localStorage).filter((k) => k.startsWith(prefix));
    const localUser = localStorage.getItem(lastUserKey);
    if (localUser && !sharedUser()) {
      localKeys
        .filter((k) => userOf(k) === localUser)
        .forEach((k) => shared.setItem(k, localStorage.getItem(k)));
      shared.setItem(lastUserKey, localUser);
    }
    localKeys.forEach((k) => localStorage.removeItem(k));
  } catch {
    // storage blocked: nothing to move
  }

  // The account this page was opened with, or signed in as here; another app changing it reloads the page
  let pageUser = sharedUser();

  const storage = {
    getItem: (key) => shared.getItem(key) ?? null,
    setItem: (key, value) => {
      if (key === lastUserKey) {
        const previous = sharedUser();
        shared.setItem(key, value);
        // a different account signing in ends the previous one's session in every app; any other
        // account's cookies left behind go too, so only the signed-in account's session remains
        if (previous && previous !== value) dropUser(previous);
        sharedKeys()
          .filter((k) => k !== lastUserKey && userOf(k) !== value)
          .forEach((k) => shared.removeItem(k));
        pageUser = value;
      } else {
        shared.setItem(key, value);
      }
      return value;
    },
    removeItem: (key) => {
      shared.removeItem(key);
      // signing out here: this page is on its sign-in screen, as the session now is
      if (key === lastUserKey) pageUser = null;
    },
    clear: () => {
      sharedKeys().forEach((k) => shared.removeItem(k));
      pageUser = null;
    },
    /** Signs out of every app: revokes the session's refresh token and clears the session. */
    endSession: () => {
      const user = sharedUser();
      if (user) revoke(shared.getItem(`${prefix}${user}.refreshToken`));
      storage.clear();
    },
    /**
     * Whether Cognito refused to renew the session for good (refresh token expired or revoked, user
     * disabled or removed), rather than a network error that a later try may get past.
     */
    isDead: (err) => ["NotAuthorizedException", "UserNotFoundException"].includes(err?.code || err?.name),
  };

  // Follow the session when another app changes it: reload as the new account, or onto the sign-in screen
  const watch = () => {
    if (sharedUser() !== pageUser) window.location.reload();
  };
  setInterval(watch, WATCH_MS);
  document.addEventListener("visibilitychange", () => document.visibilityState === "visible" && watch());
  window.addEventListener("focus", watch);

  return storage;
}
```

## NotificationSlice

```jsx
// service/NotificationSlice.jsx
import { createSlice } from "@reduxjs/toolkit";

/**
 * Redux slice for managing notification state, including open status, severity, message, and loading state.
 * An `id` replaces the toast already showing with that id, so repeated saves show one toast.
 */
const notificationSlice = createSlice({
  name: "notification",
  initialState: { open: false, severity: "", message: "", id: null, loading: false },
  reducers: {
    setNotification(state, action) {
      const notification = action.payload;
      state.open = notification.open;
      state.severity = notification.severity;
      state.message = notification.message;
      state.id = notification.id ?? null;
    },
    closeNotification(state) {
      state.open = false;
      state.severity = "";
      state.message = "";
      state.id = null;
    },
    handleLoading(state, action) {
      state.loading = action.payload.loading;
    },
  },
});

export const notificationSliceActions = notificationSlice.actions;
export default notificationSlice.reducer;
```

## store/store.js

```js
import { configureStore } from "@reduxjs/toolkit";
import authenticationReducer from "../service/AuthenticationSlice";
import notificationReducer from "../service/NotificationSlice";
import cognitoReducer from "../service/CognitiSlice";
import requestReducer from "../features/requests/service/RequestSlice";

/**
 * Configures and sets up the Redux store for the application.
 *
 * @returns {Store} - The Redux store with pre-configured reducers and middleware.
 */
const Store = configureStore({
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      // The Cognito login payload carries a CognitoUserSession object
      serializableCheck: false,
    }),
  reducer: {
    authentication: authenticationReducer,
    notification: notificationReducer,
    cognito: cognitoReducer,
    // Requests: approving leave (features/requests)
    request: requestReducer,
  },
});

export default Store;
```

---

## Access helpers and RequireRole

```js
// lib/currentUser.js
import { useSelector } from "react-redux";

/** Up to two initials for an avatar, from a name or an email. */
export const initials = (name) =>
  String(name || "?")
    .split(/[\s@.]+/)
    .filter(Boolean)
    .slice(0, 2)
    .map((w) => w[0].toUpperCase())
    .join("");

/** The payload of a JWT, or {} when it cannot be read. */
export const jwtPayload = (token) => {
  try {
    const part = String(token || "").replace(/^Bearer\s+/i, "").split(".")[1];
    const json = atob(part.replace(/-/g, "+").replace(/_/g, "/"));
    return JSON.parse(decodeURIComponent(escape(json)));
  } catch {
    return {};
  }
};

/**
 * The signed-in user: `{ id, email, name, groups }`. `state.cognito.user` holds a CognitoUserSession
 * right after login and the plain attributes after a reload, so both shapes are read; the access
 * token supplies the sub and groups in either case.
 */
export function useCurrentUser() {
  const token = useSelector((state) => state.cognito.token);
  const user = useSelector((state) => state.cognito.user);
  const attrs = user?.user?.idToken?.payload || user || {};
  const claims = jwtPayload(token);
  const email = attrs.email || "";
  const name = [attrs.given_name, attrs.family_name].filter(Boolean).join(" ") || attrs.name || email.split("@")[0] || "You";
  return { id: claims.sub || attrs.sub || null, email, name, groups: claims["cognito:groups"] || [] };
}
```

```js
// lib/<app>Access.js (here lib/adminAccess.js; the role names are the backend's)
import { useSelector } from "react-redux";
import { jwtPayload } from "./currentUser";

/**
 * Who may use the app, and which of its pages: the same rules the backend enforces. The SDK turns
 * each Cognito group in `cognito:groups` into `ROLE_` + the group upper-cased, and the APIs check
 * those with hasAnyRole(...). This only decides what is drawn; the backend refuses on its own (403).
 */
export const ADMIN_ROLES = ["R_HR_MANAGER", "R_ACCOUNT_MANAGER"];
export const HR_ROLES = ["R_HR_MANAGER"];

export const rolesOf = (token) => {
  const groups = jwtPayload(token)["cognito:groups"];
  return Array.isArray(groups) ? groups.map((g) => String(g).toUpperCase()) : [];
};

export const hasAnyRole = (token, roles) => rolesOf(token).some((role) => roles.includes(role));

/** Whether the access token may use the app at all (App.jsx draws AppRouter only then). */
export const hasAdminAccess = (token) => hasAnyRole(token, ADMIN_ROLES);

export function useHasAnyRole(roles) {
  const token = useSelector((state) => state.cognito.token);
  const isAuthenticated = useSelector((state) => state.cognito.isAuthenticated);
  return isAuthenticated && hasAnyRole(token, roles);
}
```

```jsx
// routes/RequireRole.jsx
import { Navigate } from "react-router-dom";
import { useHasAnyRole } from "@/lib/adminAccess";

/**
 * Draws a route only for a user holding one of `roles`; anyone else is sent home before the page
 * mounts or asks its API for anything.
 *
 * @param {{ roles: string[], children: JSX.Element }} props
 */
export default function RequireRole({ roles, children }) {
  return useHasAnyRole(roles) ? children : <Navigate to="/people" replace />;
}
```

## Redirect that keeps the query string

```jsx
/** Sends the old /perks address to /creds, keeping the chosen tab. */
function PerksRedirect() {
  const { search } = useLocation();
  return <Navigate to={`/creds${search}`} replace />;
}
```

## AppShell: top bar, side sheet on phones, bottom bar on tablets

```jsx
// components/layout/AppShell.jsx (the structure; the real one adds the Springboard, Personalize and the user menu)
import { NavLink, Outlet, useNavigate } from "react-router-dom";
import { useDispatch } from "react-redux";
import { LogOut, Menu, Users, CalendarDays } from "lucide-react";
import { Button } from "@/components/ui/button";
import { Sheet, SheetContent, SheetTitle, SheetTrigger } from "@/components/ui/sheet";
import { cognitoSliceActions } from "@/service/CognitiSlice";
import { cn } from "@/lib/utils";

/** The app's sections, in the order they are shown. */
const SECTIONS = [
  { to: "/people", label: "People", icon: Users },
  { to: "/team-weekly-plan", label: "Weekly plan", icon: CalendarDays },
];

function SectionLink({ to, label, icon: Icon, onClick }) {
  return (
    <NavLink
      to={to}
      onClick={onClick}
      className={({ isActive }) =>
        cn(
          "flex items-center gap-2 rounded-md px-3 py-2 text-sm font-medium text-muted-foreground hover:bg-muted hover:text-foreground",
          isActive && "bg-primary/10 text-primary",
        )
      }
    >
      <Icon className="size-4" aria-hidden="true" />
      {label}
    </NavLink>
  );
}

/**
 * The frame around every signed-in screen: a top bar with the sections and Log out. From 1000px
 * the sections are in the bar; between 768 and 999px they are a bar along the bottom (pages leave
 * room through --bottom-nav-h); below 768px they are in a side sheet.
 */
export default function AppShell() {
  const dispatch = useDispatch();
  const navigate = useNavigate();

  const logout = () => {
    dispatch(cognitoSliceActions.userLogout());
    navigate("/", { replace: true });
  };

  return (
    <div className="min-h-screen bg-page">
      <header className="sticky top-0 z-40 border-b bg-background">
        <div className="mx-4 flex h-[72px] items-center gap-3 md:mx-6">
          <Sheet>
            <SheetTrigger asChild>
              <Button variant="ghost" size="icon" className="md:hidden" aria-label="Open sections">
                <Menu className="size-5" />
              </Button>
            </SheetTrigger>
            <SheetContent side="left" className="w-[260px] p-0">
              <SheetTitle className="border-b p-5 text-lg font-semibold">Sections</SheetTitle>
              <nav className="flex flex-col gap-2 p-4" aria-label="Sections">
                {SECTIONS.map((s) => (
                  <SectionLink key={s.to} {...s} />
                ))}
              </nav>
            </SheetContent>
          </Sheet>
          <span className="text-lg font-semibold">App name</span>
          <nav className="ml-6 hidden items-center gap-1 min-[1000px]:flex" aria-label="Sections">
            {SECTIONS.map((s) => (
              <SectionLink key={s.to} {...s} />
            ))}
          </nav>
          <Button variant="ghost" className="ml-auto" onClick={logout}>
            <LogOut className="size-4" />
            Log out
          </Button>
        </div>
      </header>

      {/* the page; its padding is what full-bleed pages cancel with -mx-4 -mt-4 (sm:-mx-6 sm:-mt-5) */}
      <main className="px-4 pt-4 pb-[calc(1.25rem+var(--bottom-nav-h))] sm:px-6 sm:pt-5">
        <Outlet />
      </main>

      {/* 768–999px: the sections along the bottom, as in a tablet app */}
      <nav
        className="fixed inset-x-0 bottom-0 z-40 hidden border-t bg-background pb-[env(safe-area-inset-bottom)] md:flex min-[1000px]:hidden"
        aria-label="Sections"
      >
        {SECTIONS.map(({ to, label, icon: Icon }) => (
          <NavLink
            key={to}
            to={to}
            className={({ isActive }) =>
              cn("flex h-16 flex-1 flex-col items-center justify-center gap-1 text-xs text-muted-foreground", isActive && "text-primary")
            }
          >
            <Icon className="size-5" aria-hidden="true" />
            {label}
          </NavLink>
        ))}
      </nav>
    </div>
  );
}
```

```css
/* index.css: the room pages leave for the bottom bar, only where it shows */
:root {
  --bottom-nav-h: 0px;
}
@media (min-width: 768px) and (max-width: 999.98px) {
  :root {
    --bottom-nav-h: calc(64px + env(safe-area-inset-bottom, 0px));
  }
}
```

## AuthRouter

```jsx
export default function AuthRouter() {
  return (
    <Routes>
      <Route path="/" element={<AuthHome />}>
        <Route index element={<Login />} />
        <Route path="forgot-password" element={<ForgotPassword />} />
        <Route path="setup-account/:email" element={<VerifyAndResetAccount />} />
        <Route path="verify-otp/:email" element={<VerifyAndResetAccount />} />
        <Route path="account-setup-success" element={<SuccessScreen />} />
        <Route path="*" element={<Login />} />
      </Route>
    </Routes>
  );
}
```

---

## Screen: the hook, the panel, the table and the dialogs

```jsx
// screens/people/People.jsx
import { useState } from "react";
import { Lock, Plus, Search, X } from "lucide-react";
import Pager from "@/components/Pager";
import { Button } from "@/components/ui/button";
import { Card } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import PeopleEmpty from "@/features/people/PeopleEmpty";
import PeopleTable from "@/features/people/PeopleTable";
import PersonActionDialog from "@/features/people/PersonActionDialog";
import PersonFormDialog from "@/features/people/PersonFormDialog";
import usePeople from "@/features/people/usePeople";
import { PAGE_SIZE, ROLES } from "@/features/people/peopleOptions";

/** A filter select; "all" stands for no filter, since a shadcn Select item can't be empty. */
function FilterSelect({ label, value, onChange, options, allLabel }) {
  return (
    <Select value={value || "all"} onValueChange={(v) => onChange(v === "all" ? "" : v)}>
      <SelectTrigger aria-label={label} className="w-full bg-background sm:w-[170px]">
        <SelectValue />
      </SelectTrigger>
      <SelectContent>
        <SelectItem value="all">{allLabel}</SelectItem>
        {options.map((o) => (
          <SelectItem key={o.value} value={o.value}>
            {o.label}
          </SelectItem>
        ))}
      </SelectContent>
    </Select>
  );
}

/**
 * People: everyone in the workspace. Search and the role filter, the page of people, and numbered
 * pages. Add person and each row's Edit open the person dialog; Suspend, Activate and Resend
 * invite ask first. Every change reloads the page and the counts.
 */
export default function People() {
  const p = usePeople();
  const [dialog, setDialog] = useState(null); // { person } — person null to add
  const [dialogKey, setDialogKey] = useState(0);
  const [action, setAction] = useState(null); // { type, person }

  const openDialog = (person) => {
    setDialogKey((k) => k + 1);
    setDialog({ person });
  };

  if (p.forbidden) {
    return (
      <div className="mx-auto mt-16 max-w-md text-center">
        <Card className="items-center gap-3 px-8 py-10 shadow-none">
          <Lock className="size-8 text-muted-foreground" strokeWidth={1.5} />
          <h1 className="text-xl font-semibold">You don't have access to People</h1>
          <p className="text-sm text-muted-foreground">Ask an HR manager if you need a change to someone's account.</p>
        </Card>
      </div>
    );
  }

  const { content = [], totalElements = 0, totalPages = 0 } = p.people;
  const from = totalElements ? p.page * PAGE_SIZE + 1 : 0;
  const to = Math.min((p.page + 1) * PAGE_SIZE, totalElements);

  return (
    <div className="flex flex-col gap-3">
      <div className="flex items-center justify-between gap-4">
        <h1 className="text-[22px] font-bold tracking-[-0.02em] text-foreground">People</h1>
        <Button onClick={() => openDialog(null)}>
          <Plus className="size-4" />
          Add person
        </Button>
      </div>

      <section className="overflow-hidden rounded-xl bg-background" aria-label="People">
        <div className="flex flex-col gap-3 border-b p-4 sm:flex-row sm:items-center">
          <div className="relative w-full sm:max-w-[300px]">
            <Search className="pointer-events-none absolute top-1/2 left-3 size-4 -translate-y-1/2 text-muted-foreground" />
            <Input
              value={p.search}
              onChange={(e) => p.setSearch(e.target.value)}
              placeholder="Search by name or email"
              aria-label="Search people"
              className="pr-9 pl-9"
            />
            {p.search && (
              <button
                type="button"
                onClick={() => p.setSearch("")}
                aria-label="Clear search"
                className="absolute top-1/2 right-2.5 -translate-y-1/2 rounded p-0.5 text-muted-foreground hover:text-foreground"
              >
                <X className="size-4" />
              </button>
            )}
          </div>
          <div className="sm:ml-auto">
            <FilterSelect label="Role" value={p.role} onChange={p.setRole} options={ROLES} allLabel="All roles" />
          </div>
        </div>

        <div className="overflow-x-auto p-4 pb-0">
          <PeopleTable
            people={content}
            loading={p.loading}
            onEdit={(person) => openDialog(person)}
            onAction={(type, person) => setAction({ type, person })}
            empty={<PeopleEmpty />}
          />
        </div>

        <div className="flex flex-wrap items-center justify-between gap-3 border-t px-4 py-3 text-sm text-muted-foreground">
          <span aria-live="polite">
            {totalElements ? `Showing ${from} - ${to} of ${totalElements} entries` : "No entries"}
          </span>
          <Pager page={p.page} totalPages={totalPages} disabled={p.loading} onPage={p.setPage} />
        </div>
      </section>

      {dialog && (
        <PersonFormDialog
          key={dialogKey}
          person={dialog.person}
          onClose={() => setDialog(null)}
          onSaved={() => {
            setDialog(null);
            p.refresh();
          }}
        />
      )}

      {action && (
        <PersonActionDialog
          action={action}
          onClose={() => setAction(null)}
          onDone={() => {
            setAction(null);
            p.refresh();
          }}
        />
      )}
    </div>
  );
}
```

## Table: skeleton on first load, dimmed on reload, empty slot, row menu

```jsx
// features/people/PeopleTable.jsx (shape)
export default function PeopleTable({ people, loading, onEdit, onAction, empty }) {
  return (
    <Table>
      <TableHeader>
        <TableRow className="hover:bg-transparent">
          <TableHead className="pl-4">Person</TableHead>
          <TableHead className="hidden md:table-cell">Email</TableHead>
          <TableHead>Status</TableHead>
          <TableHead className="w-20 pr-4 text-center">Actions</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {loading && !people.length ? (
          // first load: skeleton rows shaped like the real ones
          [...Array(6)].map((_, i) => (
            <TableRow key={i}>
              <TableCell className="pl-4"><Skeleton className="h-4 w-36" /></TableCell>
              <TableCell className="hidden md:table-cell"><Skeleton className="h-4 w-44" /></TableCell>
              <TableCell><Skeleton className="h-5 w-16" /></TableCell>
              <TableCell />
            </TableRow>
          ))
        ) : people.length ? (
          people.map((p) => (
            // a reload keeps the old rows, dimmed, rather than flashing skeletons
            <TableRow key={p.userId} className={cn(loading && "opacity-60")}>
              <TableCell className="pl-4">{fullName(p)}</TableCell>
              <TableCell className="hidden md:table-cell">{p.email}</TableCell>
              <TableCell>
                <Badge variant="outline" className={cn("font-medium", STATUS_STYLE[p.status])}>{statusLabel(p.status)}</Badge>
              </TableCell>
              <TableCell className="pr-4 text-center">
                <RowMenu person={p} onEdit={onEdit} onAction={onAction} />
              </TableCell>
            </TableRow>
          ))
        ) : (
          <TableRow className="hover:bg-transparent">
            <TableCell colSpan={4}>{empty}</TableCell>
          </TableRow>
        )}
      </TableBody>
    </Table>
  );
}

/** Active reads as done (green), Invited as waiting (amber), Suspended as muted. */
const STATUS_STYLE = {
  active: "border-transparent bg-[var(--b-done)] text-[var(--t-done)]",
  invited: "border-transparent bg-[var(--b-review)] text-[var(--t-review)]",
  suspended: "border-border bg-transparent text-muted-foreground",
};

function RowMenu({ person, onEdit, onAction }) {
  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline" size="icon" className="size-8" aria-label={`Actions for ${fullName(person)}`}>
          <MoreVertical className="size-4" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end" className="w-48">
        <DropdownMenuItem onSelect={() => onEdit(person)}>
          <Pencil className="size-4" />
          Edit
        </DropdownMenuItem>
        {/* only the actions valid for the row's status */}
        {person.status === "active" && !person.self && (
          <>
            <DropdownMenuSeparator />
            <DropdownMenuItem className="text-destructive focus:text-destructive" onSelect={() => onAction("suspend", person)}>
              <UserMinus className="size-4 text-destructive" />
              Suspend
            </DropdownMenuItem>
          </>
        )}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

## Pager

```jsx
// components/Pager.jsx
import { ChevronLeft, ChevronRight } from "lucide-react";
import { cn } from "@/lib/utils";

/** The pages to offer: the first, the last and the ones around the current page, with gaps as "…". */
function pageList(page, totalPages) {
  const keep = new Set([0, totalPages - 1, page - 1, page, page + 1]);
  const pages = [];
  for (let i = 0; i < totalPages; i++) {
    if (keep.has(i)) pages.push(i);
    else if (pages[pages.length - 1] !== "…") pages.push("…");
  }
  return pages;
}

const pagerButton =
  "inline-grid size-8 place-items-center rounded-md border border-border bg-background text-sm font-medium tabular-nums text-muted-foreground transition-colors hover:text-foreground disabled:pointer-events-none disabled:opacity-40";

/**
 * Previous, the page numbers and Next, as small square buttons; the current page is filled. Shown
 * even for a single page (or none), with the arrows disabled.
 *
 * @param {object} props
 * @param {number} props.page the current page, from 0
 * @param {number} props.totalPages
 * @param {boolean} [props.disabled] while a page is loading
 * @param {(page: number) => void} props.onPage
 */
export default function Pager({ page, totalPages: total, disabled, onPage }) {
  const totalPages = Math.max(total, 1);
  return (
    <nav className="flex items-center gap-1.5" aria-label="Pages">
      <button type="button" className={pagerButton} disabled={page === 0 || disabled} onClick={() => onPage(page - 1)} aria-label="Previous page">
        <ChevronLeft className="size-4" />
      </button>
      {pageList(page, totalPages).map((n, i) =>
        n === "…" ? (
          <span key={`gap-${i}`} className="px-1 text-sm text-muted-foreground">…</span>
        ) : (
          <button
            key={n}
            type="button"
            className={cn(pagerButton, n === page && "border-primary bg-primary text-primary-foreground hover:text-primary-foreground")}
            aria-current={n === page ? "page" : undefined}
            disabled={disabled}
            onClick={() => onPage(n)}
          >
            {n + 1}
          </button>
        )
      )}
      <button type="button" className={pagerButton} disabled={page + 1 >= totalPages || disabled} onClick={() => onPage(page + 1)} aria-label="Next page">
        <ChevronRight className="size-4" />
      </button>
    </nav>
  );
}
```

## Empty state

```jsx
// features/<area>/EmptyState.jsx
import { cn } from "@/lib/utils";

/**
 * What a table shows when it has no rows: an icon in a circle, a quiet line, then a smaller
 * explanation and an optional action.
 *
 * @param {object} props
 * @param {React.ElementType} props.icon - A lucide icon.
 * @param {string} props.title - The heading.
 * @param {React.ReactNode} props.description - The line under it.
 * @param {React.ReactNode} [props.action] - A button.
 * @param {boolean} [props.success] - Draws the icon in green, for "all caught up".
 */
export default function EmptyState({ icon: Icon, title, description, action, success }) {
  return (
    <div className="flex flex-col items-center gap-2 px-6 py-12 text-center">
      <span
        className={cn(
          "grid size-12 place-items-center rounded-full bg-muted text-primary dark:bg-white/10 dark:text-white/80",
          success && "bg-emerald-50 text-emerald-700 dark:bg-emerald-500/15 dark:text-emerald-300",
        )}
      >
        <Icon className="size-6" strokeWidth={1.6} />
      </span>
      <p className="m-0 text-sm font-medium text-muted-foreground">{title}</p>
      {description && <p className="m-0 max-w-[52ch] text-xs leading-relaxed text-muted-foreground/80">{description}</p>}
      {action && <div className="mt-1">{action}</div>}
    </div>
  );
}
```

## Confirm dialog for consequential actions

```jsx
// features/people/PersonActionDialog.jsx
const ACTIONS = {
  suspend: {
    title: (n) => `Suspend ${n}?`,
    body: "They can't sign in until they're activated again. Nothing they own is deleted.",
    confirm: "Suspend",
    done: (n) => `${n} suspended`,
    failed: "The person could not be suspended",
    run: (id) => AdminService.suspendPerson(id),
    danger: true,
  },
  activate: {
    title: (n) => `Activate ${n}?`,
    body: "They can sign in again straight away.",
    confirm: "Activate",
    done: (n) => `${n} activated`,
    failed: "The person could not be activated",
    run: (id) => AdminService.activatePerson(id),
  },
};

/**
 * Asks before Suspend or Activate, then runs it. It cannot be dismissed while the request is running.
 *
 * @param {object} props
 * @param {{type: "suspend"|"activate", person: object}} props.action
 * @param {() => void} props.onClose
 * @param {() => void} props.onDone called after the action succeeds
 */
export default function PersonActionDialog({ action, onClose, onDone }) {
  const dispatch = useDispatch();
  const [busy, setBusy] = useState(false);
  const spec = ACTIONS[action.type];
  const name = fullName(action.person);

  const confirm = async () => {
    setBusy(true);
    try {
      await spec.run(action.person.userId);
      dispatch(notificationSliceActions.setNotification({ open: true, severity: "success", message: spec.done(name) }));
      onDone();
    } catch (err) {
      dispatch(notificationSliceActions.setNotification({ open: true, severity: "error", message: apiError(err, spec.failed) }));
      setBusy(false);
    }
  };

  return (
    <AlertDialog open onOpenChange={(open) => !open && !busy && onClose()}>
      <AlertDialogContent className="sm:max-w-md">
        <AlertDialogHeader className="text-left">
          <AlertDialogTitle>{spec.title(name)}</AlertDialogTitle>
          <AlertDialogDescription>{spec.body}</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={busy}>Cancel</AlertDialogCancel>
          {/* a plain Button: AlertDialogAction closes the dialog before the request has finished */}
          <Button onClick={confirm} disabled={busy} variant={spec.danger ? "destructive" : "default"}>
            {busy && <Loader2 className="size-4 animate-spin" />}
            {spec.confirm}
          </Button>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

---

## URL state: tabs and selected items

```js
// hooks/useUrlParam.js
import { useCallback } from "react";
import { useSearchParams } from "react-router-dom";

/**
 * One query-string parameter as state, so a view can be reloaded or shared as a link.
 * Setting null or "" removes the parameter; other parameters are left as they are.
 *
 * @param {string} name - The parameter's name.
 * @returns {[string|null, (value: *) => void]} The value and its setter.
 */
export default function useUrlParam(name) {
  const [params, setParams] = useSearchParams();
  const value = params.get(name);

  const setValue = useCallback(
    (nextValue) => {
      setParams((current) => {
        const updated = new URLSearchParams(current);
        if (nextValue === null || nextValue === undefined || nextValue === "") updated.delete(name);
        else updated.set(name, String(nextValue));
        return updated;
      });
    },
    [name, setParams],
  );

  return [value, setValue];
}
```

```jsx
// usage: an unknown or missing ?tab= falls back to the first tab
const TABS = ["people", "skills", "requests"];
const [tabParam, setTab] = useUrlParam("tab");
const tab = TABS.includes(tabParam) ? tabParam : TABS[0];

<Tabs value={tab} onValueChange={setTab}> … </Tabs>
```

## Debounced value

```js
// hooks/useDebouncedValue.js
import { useEffect, useState } from "react";

/**
 * The value once it has stopped changing, so a search box sends one request for what the user
 * typed rather than one per keystroke.
 */
export default function useDebouncedValue(value, delay = 350) {
  const [settledValue, setSettledValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setSettledValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return settledValue;
}
```

## Parallel loads and optimistic updates

```js
// features/requests/useRequests.js: the page and its tab counts, loaded together
useEffect(() => {
  let live = true;
  (async () => {
    setLoading(true);
    // allSettled, so a failed count doesn't hide the page (or the other way round)
    const [pageRes, countsRes] = await Promise.allSettled([
      RequestService.fetchApprovalQueue({ status, search: query, page }),
      RequestService.fetchApprovalCounts({ search: query }),
    ]);
    if (!live) return;
    if (pageRes.status === "fulfilled") setData(pageRes.value?.data || EMPTY);
    else setData(EMPTY);
    if (countsRes.status === "fulfilled") setCounts(countsRes.value?.data || null);
    else setCounts({ failed: true });
    const failed = [pageRes, countsRes].find((r) => r.status === "rejected");
    if (failed) {
      dispatch(
        notificationSliceActions.setNotification({
          open: true,
          severity: "error",
          message: apiError(failed.reason, "Requests could not be loaded"),
          id: "requests-load",
        })
      );
    }
    setLoading(false);
  })();
  return () => {
    live = false;
  };
}, [status, query, page, version, dispatch]);
```

```js
// An optimistic like: the count changes at once, and goes back if the server refuses
const toggleLike = async (post) => {
  const before = post;
  const liked = !post.liked;
  const update = (next) => setPosts((list) => list.map((p) => (p.id === post.id ? next : p)));
  update({ ...post, liked, likeCount: post.likeCount + (liked ? 1 : -1) });
  try {
    await (liked ? ArticlesService.likePost(post.id) : ArticlesService.unlikePost(post.id));
  } catch (err) {
    update(before);
    dispatch(
      notificationSliceActions.setNotification({
        open: true,
        severity: "error",
        message: apiError(err, "The like could not be saved"),
      })
    );
  }
};
```

---

## Theme tokens: Tailwind 4 (CSS-first)

```css
/* src/index.css */
@import "tailwindcss";

@custom-variant dark (&:is(.dark *));

@theme inline {
  --radius-lg: var(--radius);
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);
  --color-muted-foreground: var(--muted-foreground);
  --color-border: var(--border);
  /* Breezeware surfaces */
  --color-page: var(--page);
  --color-raise: var(--raise);
  --color-line: var(--line);
  --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
}

:root {
  --radius: 12px;
  --background: #ffffff;
  --foreground: #0f172a;
  --primary: #1b3764; /* Breezeware navy; data-color-theme on <html> overrides it */
  /* status pairs: b = background, t = text */
  --b-todo: #dbeafe;   --t-todo: #1d4ed8;
  --b-prog: #ede9fe;   --t-prog: #5b45d9;
  --b-review: #fef0c3; --t-review: #8a5a00;
  --b-done: #d3f5e5;   --t-done: #1a7f37;
  --b-late: #fde2e8;   --t-late: #b4234a;
}

.dark {
  --background: #121212;
  --foreground: #eef0f3;
  --b-done: #0f2e21;   --t-done: #3fb950;
  /* … every light token gets a dark value */
}
```

To add a colour, give it a light and a dark value under `:root` / `.dark`, map it in
`@theme inline` as `--color-<name>`, and then use it as `bg-<name>` / `text-<name>`.

## Theme tokens: Tailwind 3 (older apps)

```js
// tailwind.config.js
export default {
  darkMode: ["class"],
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: {
    // some apps define extra screens like these; use one only if the app's config has it
    screens: { xs: "480px", sm: "640px", md: "768px", lg: "1024px", xl: "1280px", "2xl": "1400px", "3xl": "1800px" },
    extend: {
      colors: {
        border: "var(--border)",
        background: "var(--background)",
        foreground: "var(--foreground)",
        primary: { DEFAULT: "var(--primary)", foreground: "var(--primary-foreground)" },
      },
    },
  },
};
```

Never paste a v3 config into a v4 app, or `@theme` into a v3 app.

## Appearance and colour theme (applied before the first paint)

```js
// lib/appearance.js
import { useCallback, useEffect, useState } from "react";

/** Light, dark, or whatever the device is set to; the choice belongs to this browser. */
export const APPEARANCES = [
  { id: "light", label: "Light" },
  { id: "dark", label: "Dark" },
  { id: "system", label: "Match device" },
];

const STORAGE_KEY = "themeMode";
const isKnown = (id) => APPEARANCES.some((a) => a.id === id);
const systemPrefersDark = () => Boolean(window.matchMedia?.("(prefers-color-scheme: dark)").matches);

export function getStoredAppearance() {
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    return isKnown(stored) ? stored : "light";
  } catch {
    return "light";
  }
}

/** Whether a choice means dark right now. */
export const resolveDark = (mode) => mode === "dark" || (mode === "system" && systemPrefersDark());

/** Puts the choice on the page: the `dark` class every dark: style follows, and data-theme for editors. */
export function applyAppearance(mode) {
  const dark = resolveDark(mode);
  document.documentElement.classList.toggle("dark", dark);
  document.documentElement.dataset.theme = dark ? "dark" : "light";
  return dark;
}

/** The choice and a setter that applies and remembers it; "Match device" follows the device live. */
export function useAppearance() {
  const [mode, setModeState] = useState(getStoredAppearance);

  useEffect(() => {
    applyAppearance(mode);
    if (mode !== "system" || !window.matchMedia) return undefined;
    const media = window.matchMedia("(prefers-color-scheme: dark)");
    const follow = () => applyAppearance("system");
    media.addEventListener("change", follow);
    return () => media.removeEventListener("change", follow);
  }, [mode]);

  const setMode = useCallback((next) => {
    try {
      localStorage.setItem(STORAGE_KEY, next);
    } catch {
      // blocked storage: the page still changes, it just won't be remembered
    }
    setModeState(next);
  }, []);

  return [mode, setMode];
}
```

```js
// lib/colorTheme.js
/** The brand colours. The colours live in index.css, keyed on data-color-theme on <html>. */
export const COLOR_THEMES = [
  { id: "blue", label: "Navy", swatch: "#1B3764" },
  { id: "indigo", label: "Indigo", swatch: "#3730A3" },
  { id: "royal", label: "Royal", swatch: "#1D4ED8" },
  { id: "ocean", label: "Ocean", swatch: "#0369A1" },
  { id: "black", label: "Black", swatch: "#0B0D10" },
];
export const DEFAULT_COLOR_THEME = "blue";
const STORAGE_KEY = "colorTheme";

export function getStoredColorTheme() {
  try {
    const stored = localStorage.getItem(STORAGE_KEY);
    return COLOR_THEMES.some((t) => t.id === stored) ? stored : DEFAULT_COLOR_THEME;
  } catch {
    return DEFAULT_COLOR_THEME;
  }
}

/** Sets the theme on the page. Blue is the stylesheet's default, so it needs no attribute. */
export function applyColorTheme(id) {
  if (id === DEFAULT_COLOR_THEME) document.documentElement.removeAttribute("data-color-theme");
  else document.documentElement.setAttribute("data-color-theme", id);
}
```

```css
/* index.css: each theme overrides --primary, in light and dark */
[data-color-theme="indigo"] { --primary: #3730a3; }
.dark[data-color-theme="indigo"] { --primary: #818cf8; }
```

---

## Shared formatters

```js
// service/Utils.jsx (excerpt)
/** A date as yyyy-MM-dd in local time. `toISOString()` would shift it across a timezone. */
export function toIsoDate(date) {
  if (!date) return "";
  const value = new Date(date);
  const month = `${value.getMonth() + 1}`.padStart(2, "0");
  const day = `${value.getDate()}`.padStart(2, "0");
  return `${value.getFullYear()}-${month}-${day}`;
}

const MONTHS_SHORT = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"];

/** A date as "14 Sep 2026", the one format used everywhere. An ISO date string is read as local midnight. */
export function formatDate(value) {
  if (!value) return "";
  const date =
    typeof value === "string" && /^\d{4}-\d{2}-\d{2}$/.test(value) ? new Date(`${value}T00:00:00`) : new Date(value);
  if (Number.isNaN(date.getTime())) return "";
  return `${date.getDate()} ${MONTHS_SHORT[date.getMonth()]} ${date.getFullYear()}`;
}

/** "20 Oct 2026 – 21 Oct 2026", or one date when they match. */
export function formatDateRange(from, to) {
  if (!from) return "";
  return from === to || !to ? formatDate(from) : `${formatDate(from)} – ${formatDate(to)}`;
}

/** "1 task", "2 tasks". */
export function plural(count, word) {
  return `${count} ${word}${count === 1 ? "" : "s"}`;
}
```

## jsconfig.json and the Vite alias

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

```js
// vite.config.js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite"; // Tailwind 4 apps only
import path from "path";
import { fileURLToPath } from "url";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: { alias: { "@": path.resolve(__dirname, "./src") } },
  server: {
    // ports are fixed: they are in VITE_APP_URL, email links and the backend's CORS allow-list
    port: 3001,
    strictPort: true,
  },
});
```

## Environment files

```bash
# .env: committed; public values only (every VITE_ value ships in the bundle)
VITE_BASE_URL_ADMIN=http://localhost:8080
VITE_COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
VITE_COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx
VITE_APP_URL=http://localhost:3001
```

`.env.production` (and, in apps that have them, `.env.qa`, `.env.staging` and `.env.pilot`) carry
the same keys per environment. Builds pick one with `vite build --mode production`. Never
put a client secret, API key or password in any of them.
