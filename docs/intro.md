---
id: Getting Started with Auth Connect in @ionic-react
slug: /
sidebar_position: 1
---

<CH.Scrollycoding>

Learn how to get started with Auth Connect by creating a sample project that uses `@ionic/react`.

This tutorial starts with installing and configuring Auth Connect and moves onto achieving common use-cases: performing login/logout, checking the user's authentication status, obtaining tokens from Auth Connect, and storing tokens using <a href="https://ionic.io/docs/identity-vault" target="_blank">Identity Vault</a>.

<CH.Code>

```typescript AuthProvider.tsx
import { AuthConnect, ProviderOptions, Auth0Provider, TokenType } from "@ionic-enterprise/auth";
import { isPlatform } from '@ionic/react';
import { PropsWithChildren, createContext, useState, useEffect, useContext } from 'react';
import { SessionVaultContext } from './SessionVaultProvider';

const isNative = isPlatform('hybrid');

const options: ProviderOptions = {
  audience: 'https://io.ionic.demo.ac',
  clientId: 'yLasZNUGkZ19DGEjTmAITBfGXzqbvd00',
  discoveryUrl: 'https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration',
  logoutUrl: isNative ? 'msauth://login' : 'http://localhost:8100/login',
  redirectUri: isNative ? 'msauth://login' : 'http://localhost:8100/login',
  scope: 'openid offline_access email picture profile',
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? 'capacitor' : 'web',
    logLevel: 'DEBUG',
    ios: { webView: 'private' },
    web: { uiMode: 'popup', authFlow: 'implicit' },
  });
};

const provider = new Auth0Provider();

export const AuthContext = createContext<{
  isAuthenticated: boolean;
  getAccessToken: () => Promise<string | undefined>;
  getUserName: () => Promise<string | undefined>;
  login: () => Promise<void>;
  logout: () => Promise<void>;
}>({
  isAuthenticated: false,
  getAccessToken: () => { throw new Error('Method not implemented.'); },
  getUserName: () => { throw new Error('Method not implemented.'); },
  login: () => { throw new Error('Method not implemented.'); },
  logout: () => { throw new Error('Method not implemented.'); },
});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);
  const { clearSession, getSession, setSession } = useContext(SessionVaultContext);
  const [isAuthenticated, setIsAuthenticated] = useState<boolean>(false);

  const saveAuthResult = async (authResult: AuthResult | null): Promise<void> => {
    if (authResult) {
      await setSession(authResult);
      setIsAuthenticated(true);
    } else {
      await clearSession();
      setIsAuthenticated(false);
    }
  };

  const refreshAuth = async (authResult: AuthResult): Promise<AuthResult | null> => {
    let newAuthResult: AuthResult | null = null;

    if (await AuthConnect.isRefreshTokenAvailable(authResult)) {
      try {
        newAuthResult = await AuthConnect.refreshSession(provider, authResult);
      } catch (err) {
        console.log('Error refreshing session.', err);
      }
    }

    return newAuthResult;
  };

  const getAuthResult = async (): Promise<AuthResult | null> => {
    let authResult = await getSession();

    if (authResult && (await AuthConnect.isAccessTokenExpired(authResult))) {
      const newAuthResult = await refreshAuth(authResult);
      await saveAuthResult(newAuthResult);
    }
    setIsAuthenticated(!!authResult);
    return authResult;
  };

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  const getAccessToken = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    return res?.accessToken;
  };

  const getUserName = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    if (res) {
      const data = await AuthConnect.decodeToken<{ name: string }>(TokenType.id, res);
      return data?.name;
    }
  };

  const login = async (): Promise<void> => {
    const authResult = await AuthConnect.login(provider, options);
    await saveAuthResult(authResult);
  };

  const logout = async (): Promise<void> => {
    const authResult = await getAuthResult();
    if (authResult) {
      await AuthConnect.logout(provider, authResult);
      await saveAuthResult(null);
    }
  };

  return (
    <AuthContext.Provider value={{ isAuthenticated, getAccessToken, getUserName, login, logout }}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```

```typescript Tab1.tsx
import { useContext, useEffect, useState } from 'react';
import { IonButton, IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/react';
import { AuthContext } from '../providers/AuthProvider';
import './Tab1.css';

const Tab1: React.FC = () => {
  const { isAuthenticated, getUserName, login, logout } = useContext(AuthContext);
  const [userName, setUserName] = useState<string>();

  useEffect(() => {
    getUserName().then(setUserName);
  }, [isAuthenticated, getUserName]);

  const handleLogin = async () => {
    try {
      await login();
    } catch (err) {
      console.log('Error logging in:', err);
    }
  };

  const handleLogout = async () => {
    await logout();
  };

  return (
    <IonPage>
      <IonHeader>
        <IonToolbar>
          <IonTitle>Tab 1</IonTitle>
        </IonToolbar>
      </IonHeader>
      <IonContent fullscreen>
        <IonHeader collapse="condense">
          <IonToolbar>
            <IonTitle size="large">Tab 1</IonTitle>
          </IonToolbar>
        </IonHeader>
        {userName && <h1>{userName}</h1>}
        {!isAuthenticated ? (
          <IonButton onClick={handleLogin}>Login</IonButton>
        ) : (
          <IonButton onClick={handleLogout}>Logout</IonButton>
        )}
      </IonContent>
    </IonPage>
  );
};

export default Tab1;
```

```typescript SessionVaultProvider.tsx
import { AuthResult } from '@ionic-enterprise/auth';
import { createContext, PropsWithChildren } from 'react';
import {
  BrowserVault,
  DeviceSecurityType,
  IdentityVaultConfig,
  Vault,
  VaultType,
} from '@ionic-enterprise/identity-vault';
import { isPlatform } from '@ionic/react';

const createVault = (config: IdentityVaultConfig): Vault | BrowserVault => {
  return isPlatform('hybrid') ? new Vault(config) : new BrowserVault(config);
};

const key = 'auth-result';
const vault = createVault({
  key: 'io.ionic.gettingstartedacreact',
  type: VaultType.SecureStorage,
  deviceSecurityType: DeviceSecurityType.None,
  lockAfterBackgrounded: 5000,
  shouldClearVaultAfterTooManyFailedAttempts: true,
  customPasscodeInvalidUnlockAttempts: 2,
  unlockVaultOnLoad: false,
});

export const SessionVaultContext = createContext<{
  clearSession: () => Promise<void>;
  getSession: () => Promise<AuthResult | null>;
  setSession: (value?: AuthResult) => Promise<void>;
}>({
  clearSession: () => {
    throw new Error('Method not implemented.');
  },
  getSession: () => {
    throw new Error('Method not implemented.');
  },
  setSession: () => {
    throw new Error('Method not implemented.');
  },
});

export const SessionVaultProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const clearSession = (): Promise<void> => {
    return vault.clear();
  };

  const getSession = (): Promise<AuthResult | null> => {
    return vault.getValue<AuthResult>(key);
  };

  const setSession = (value?: AuthResult): Promise<void> => {
    return vault.setValue(key, value);
  };

  return (
    <SessionVaultContext.Provider value={{ clearSession, getSession, setSession }}>
      {children}
    </SessionVaultContext.Provider>
  );
};
```

</CH.Code>

---

## Generate the Application

Start by creating an Ionic React Tabbed Layout starter.

<CH.Section lineNumbers={false}>

```bash Terminal
ionic start getting-started-ac-react-tabs --type=react
```

</CH.Section>

---

### Change the app identifier

Ionic recommends changing the app identifier _before_ adding any deployment platforms. If you want to change the app identifier after adding iOS or Android, you will have to update native artifacts.

<CH.Section rows={2}>

```typescript capacitor.config.ts focus=4
import { CapacitorConfig } from "@capacitor/cli";

const config: CapacitorConfig = {
  appId: "io.ionic.gettingstartedacreact",
  appName: "getting-started-ac-react",
  webDir: "build",
  bundledWebRuntime: false,
};

export default config;
```

</CH.Section>

---

### Remove React Strict Mode

Strict mode triggers `useEffect` twice in development, remove it for this tutorial.

<CH.Section rows={2}>

```typescript index.tsx focus=9[12:19]
import React from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
import * as serviceWorkerRegistration from "./serviceWorkerRegistration";
import reportWebVitals from "./reportWebVitals";

const container = document.getElementById("root");
const root = createRoot(container!);
root.render(<App />);

// If you want your app to work offline and load faster, you can change
// unregister() to register() below. Note this comes with some pitfalls.
// Learn more about service workers: https://cra.link/PWA
serviceWorkerRegistration.unregister();

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
```

</CH.Section>

---

### Add the native platforms

Build the application then install and create the iOS and Android platforms.

<CH.Section lineNumbers={false}>

```bash Terminal
npm run build
ionic cap add android
ionic cap add ios
```

</CH.Section>

---

### Modify the build script

Modify the `build` npm script to sync Capacitor with each build.

<CH.Section rows={2}>

```json package.json focus=10
{
  "name": "getting-started-ac-react",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    /* Omitted for brevity */
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build && cap sync",
    "test": "react-scripts test --transformIgnorePatterns 'node_modules/(?!(@ionic/react|@ionic/react-router|@ionic/core|@stencil/core|ionicons)/)'",
    "eject": "react-scripts eject"
  },
  "eslintConfig": {
    "extends": ["react-app", "react-app/jest"]
  },
  "browserslist": {
    /* Omitted for brevity */
  },
  "devDependencies": {
    /* Omitted for brevity */
  },
  "description": "An Ionic project"
}
```

</CH.Section>

---

## Install Auth Connect

In order to install Auth Connect, you will need to use `ionic enterprise register` to register your product key. This will create a `.npmrc` file containing the product key.

If you have already performed that step for your production application, you can just copy the `.npmrc` file from your production project. Since this application is for learning purposes only, you don't need to obtain another key.

<CH.Section lineNumbers={false}>

```bash Terminal
npm install @ionic-enterprise/auth
```

</CH.Section>

---

## Configure Auth Connect

Auth Connect functionality will be exposed through a React Context.

Create `AuthProvider.tsx` within a folder named `src/providers`.

```tsx AuthProvider.tsx
import { ProviderOptions } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  clientId: "",
  discoveryUrl: "",
  scope: "openid offline_access",
  audience: "",
  redirectUri: isNative ? "" : "",
  logoutUrl: isNative ? "" : "",
};

export const AuthContext = createContext<{}>({});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  return <AuthContext.Provider value={{}}>{children}</AuthContext.Provider>;
};
```

Don't forget to wrap `IonReactRouter` with the new provider so the Context can be shared with the Tabs.

<CH.Section rows={3}>

```tsx src/App.tsx focus=41,42,73,74
import { Redirect, Route } from "react-router-dom";
import {
  IonApp,
  IonIcon,
  IonLabel,
  IonRouterOutlet,
  IonTabBar,
  IonTabButton,
  IonTabs,
  setupIonicReact,
} from "@ionic/react";
import { IonReactRouter } from "@ionic/react-router";
import { ellipse, square, triangle } from "ionicons/icons";
import Tab1 from "./pages/Tab1";
import Tab2 from "./pages/Tab2";
import Tab3 from "./pages/Tab3";

/* Core CSS required for Ionic components to work properly */
import "@ionic/react/css/core.css";

/* Basic CSS for apps built with Ionic */
import "@ionic/react/css/normalize.css";
import "@ionic/react/css/structure.css";
import "@ionic/react/css/typography.css";

/* Optional CSS utils that can be commented out */
import "@ionic/react/css/padding.css";
import "@ionic/react/css/float-elements.css";
import "@ionic/react/css/text-alignment.css";
import "@ionic/react/css/text-transformation.css";
import "@ionic/react/css/flex-utils.css";
import "@ionic/react/css/display.css";

/* Theme variables */
import "./theme/variables.css";

setupIonicReact();

const App: React.FC = () => (
  <IonApp>
    <AuthProvider>
      <IonReactRouter>
        <IonTabs>
          <IonRouterOutlet>
            <Route exact path="/tab1">
              <Tab1 />
            </Route>
            <Route exact path="/tab2">
              <Tab2 />
            </Route>
            <Route path="/tab3">
              <Tab3 />
            </Route>
            <Route exact path="/">
              <Redirect to="/tab1" />
            </Route>
          </IonRouterOutlet>
          <IonTabBar slot="bottom">
            <IonTabButton tab="tab1" href="/tab1">
              <IonIcon aria-hidden="true" icon={triangle} />
              <IonLabel>Tab 1</IonLabel>
            </IonTabButton>
            <IonTabButton tab="tab2" href="/tab2">
              <IonIcon aria-hidden="true" icon={ellipse} />
              <IonLabel>Tab 2</IonLabel>
            </IonTabButton>
            <IonTabButton tab="tab3" href="/tab3">
              <IonIcon aria-hidden="true" icon={square} />
              <IonLabel>Tab 3</IonLabel>
            </IonTabButton>
          </IonTabBar>
        </IonTabs>
      </IonReactRouter>
    </AuthProvider>
  </IonApp>
);

export default App;
```

</CH.Section>

---

### Auth Connect options

The `options` object gets passed to the `login()` function to help establish the authentication session.

Obtaining this information likely takes coordination with whomever administers the backend services.

You can use your own configuration for this step; however, we suggest starting with our configuration, get the application working, and then try your own configuration after that.

```tsx AuthProvider.tsx focus=7:15
import { ProviderOptions } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

export const AuthContext = createContext<{}>({});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  return <AuthContext.Provider value={{}}>{children}</AuthContext.Provider>;
};
```


---

### Initialization

Auth Connect needs to be initialized before any functions can be used. 

A flag in `AuthProvider` will prevent components within `IonReactRouter` from rendering until Auth Connect has initialized.

```tsx AuthProvider.tsx focus=17:24,29:38
import { AuthConnect, ProviderOptions } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const AuthContext = createContext<{}>({});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  return (
    <AuthContext.Provider value={{}}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```

---

### The Provider

A provider object specifying details on communicating with the OIDC service is required.

Auth Connect offers several common providers out of the box: `Auth0Provider`, `AzureProvider`, `CognitoProvider`, `OktaProvider`, and `OneLoginProvider`. You can also create your own provider, though doing so is beyond the scope of this tutorial.

```tsx AuthProvider.tsx mark=1[40:53] focus=26
import { AuthConnect, ProviderOptions, Auth0Provider } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const provider = new Auth0Provider();

export const AuthContext = createContext<{}>({});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  return (
    <AuthContext.Provider value={{}}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```

---

### Login and Logout

Login and logout are the two most fundamental operations in the authentication flow. 

Login requires both the `provider` and `options` established above. Login resolves an `AuthResult` if the operation succeeds. The `AuthResult` contains auth tokens as well as some other information. This object needs to be passed to almost all other `AuthConnect` functions; as such, it needs to be saved. The `login()` call rejects with an error if the user cancels the login or if something else prevents the login to complete.


```tsx AuthProvider.tsx focus=29,31,36,42:45,48[36:41]
import { AuthConnect, ProviderOptions, Auth0Provider, AuthResult } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const provider = new Auth0Provider();

export const AuthContext = createContext<{
  login: () => Promise<void>;
}>({
  login: () => { throw new Error("Method not implemented."); }
});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);
  const [authResult, setAuthResult] = useState<AuthResult | null>(null);

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  const login = async (): Promise<void> => {
    const authResult = await AuthConnect.login(provider, options);
    setAuthResult(authResult);
  };

  return (
    <AuthContext.Provider value={{ login }}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```

---

Logout requires the `provider` as well as the `AuthResult` object returned by the `login()` call.

```tsx AuthProvider.tsx focus=30,33,49:54,57[43:49]
import { AuthConnect, ProviderOptions, Auth0Provider, AuthResult } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const provider = new Auth0Provider();

export const AuthContext = createContext<{
  login: () => Promise<void>;
  logout: () => Promise<void>;
}>({
  login: () => { throw new Error("Method not implemented."); },
  logout: () => { throw new Error("Method not implemented."); }
});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);
  const [authResult, setAuthResult] = useState<AuthResult | null>(null);

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  const login = async (): Promise<void> => {
    const authResult = await AuthConnect.login(provider, options);
    setAuthResult(authResult);
  };

  const logout = async (): Promise<void> => {
    if (authResult) {
      await AuthConnect.logout(provider, authResult);
      setAuthResult(null);
    }
  };

  return (
    <AuthContext.Provider value={{ login, logout }}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```

---

To test these new functions, replace `ExploreContainer` with "Login" and "Logout" buttons in `src/pages/Tab1.tsx`.

Using Ionic's Auth0 provider, functionality can be tested with the following credentials:

- Email Address: `test@ionic.io`
- Password: `Ion54321`

```tsx Tab1.tsx focus=3,7,22:23
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/react';
import ExploreContainer from '../components/ExploreContainer';
import { AuthContext } from '../providers/AuthProvider';
import './Tab1.css';

const Tab1: React.FC = () => {
  const { login, logout } = useContext(AuthContext);

  return (
    <IonPage>
      <IonHeader>
        <IonToolbar>
          <IonTitle>Tab 1</IonTitle>
        </IonToolbar>
      </IonHeader>
      <IonContent fullscreen>
        <IonHeader collapse="condense">
          <IonToolbar>
            <IonTitle size="large">Tab 1</IonTitle>
          </IonToolbar>
        </IonHeader>
        <IonButton onClick={login}>Login</IonButton>
        <IonButton onClick={logout}>Logout</IonButton>
      </IonContent>
    </IonPage>
  );
};

export default Tab1;
```

--- 

### Configure the native projects

Login will fail when testing on a native device.

This occurs because the native device doesn't know which application(s) handle navigation to the `msauth://` scheme (using Ionic's Auth0 provider). To register the application to handle the scheme, modify the `build.gradle` and `Info.plist` files as <a href="https://ionic.io/docs/auth-connect/install" target="_blank">noted here</a>.

Replace `$AUTH_URL_SCHEME` with `msauth` when using Ionic's Auth0 provider.

---

### Determining the current auth status

```tsx AuthProvider.tsx focus=29,33,41,43:46,55,62,67[36:52]
import { AuthConnect, ProviderOptions, Auth0Provider, AuthResult } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const provider = new Auth0Provider();

export const AuthContext = createContext<{
  isAuthenticated: boolean;
  login: () => Promise<void>;
  logout: () => Promise<void>;
}>({
  isAuthenticated: false,
  login: () => { throw new Error("Method not implemented."); },
  logout: () => { throw new Error("Method not implemented."); }
});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);
  const [authResult, setAuthResult] = useState<AuthResult | null>(null);
  const [isAuthenticated, setIsAuthenticated] = useState<boolean>(false);

  const getAuthResult = async (): Promise<AuthResult | null> => {
    setIsAuthenticated(!!authResult);
    return authResult;
  };

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  const login = async (): Promise<void> => {
    const authResult = await AuthConnect.login(provider, options);
    setAuthResult(authResult);
    setIsAuthenticated(true);
  };

  const logout = async (): Promise<void> => {
    if (authResult) {
      await AuthConnect.logout(provider, authResult);
      setAuthResult(null);
      setIsAuthenticated(false);
    }
  };

  return (
    <AuthContext.Provider value={{ isAuthenticated, login, logout }}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```


Users are shown both the login and logout buttons but you don't really know if the user is logged in or not. Let's change that.

A simple strategy to use is tracking the status via state, updating the value after calling certain `AuthConnect` methods.

Ignore the extra complexity with the `getAuthResult()` function -- we will expand on that as we go.

---

```tsx Tab1.tsx focus=7[11:26],9:19,34:38
import { IonContent, IonHeader, IonPage, IonTitle, IonToolbar } from '@ionic/react';
import ExploreContainer from '../components/ExploreContainer';
import { AuthContext } from '../providers/AuthProvider';
import './Tab1.css';

const Tab1: React.FC = () => {
  const { isAuthenticated, login, logout } = useContext(AuthContext);

  const handleLogin = async () => {
    try {
      await login();
    } catch (err) {
      console.log("Error logging in:", err);
    }
  };

  const handleLogout = async () => {
    await logout();
  };

  return (
    <IonPage>
      <IonHeader>
        <IonToolbar>
          <IonTitle>Tab 1</IonTitle>
        </IonToolbar>
      </IonHeader>
      <IonContent fullscreen>
        <IonHeader collapse="condense">
          <IonToolbar>
            <IonTitle size="large">Tab 1</IonTitle>
          </IonToolbar>
        </IonHeader>
        {!isAuthenticated ? (
          <IonButton onClick={handleLogin}>Login</IonButton>
        ) : (
          <IonButton onClick={handleLogout}>Logout</IonButton>
        )}
      </IonContent>
    </IonPage>
  );
};

export default Tab1;
```

Use `isAuthenticated` to determine which button to display, depending on the current auth status.

Notice the `try...catch` in the `handleLogin()` method. Production apps should have some kind of handling here, but this tutorial will only log the fact.

At this point, you should see the "Login" button if you are not logged in and the "Logout" button if you are.


---

### Get the tokens

```tsx AuthProvider.tsx focus=30:31,36:37,70:84,88[24:52]
import { AuthConnect, ProviderOptions, Auth0Provider, AuthResult, TokenType } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const provider = new Auth0Provider();

export const AuthContext = createContext<{
  isAuthenticated: boolean;
  getAccessToken: () => Promise<string | undefined>;
  getUserName: () => Promise<string | undefined>;
  login: () => Promise<void>;
  logout: () => Promise<void>;
}>({
  isAuthenticated: false,
  getAccessToken: () => { throw new Error("Method not implemented."); },
  getUserName: () => { throw new Error("Method not implemented."); },
  login: () => { throw new Error("Method not implemented."); },
  logout: () => { throw new Error("Method not implemented."); }
});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);
  const [authResult, setAuthResult] = useState<AuthResult | null>(null);
  const [isAuthenticated, setIsAuthenticated] = useState<boolean>(false);

  const getAuthResult = async (): Promise<AuthResult | null> => {
    setIsAuthenticated(!!authResult);
    return authResult;
  };

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  const login = async (): Promise<void> => {
    const authResult = await AuthConnect.login(provider, options);
    setAuthResult(authResult);
    setIsAuthenticated(true);
  };

  const logout = async (): Promise<void> => {
    if (authResult) {
      await AuthConnect.logout(provider, authResult);
      setAuthResult(null);
      setIsAuthenticated(false);
    }
  };

  const getAccessToken = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    return res?.accessToken;
  };

  const getUserName = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    if (res) {
      const data = await AuthConnect.decodeToken<{ name: string }>(
        TokenType.id,
        res
      );
      return data?.name;
    }
  };

  return (
    <AuthContext.Provider value={{ 
      isAuthenticated, getAccessToken, getUserName, login, logout 
    }}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```


We can now log in and out, but what about getting the tokens that the OIDC provider gave us? That information is stored as part of the `AuthResult`. Auth Connect provides methods to allow us to easily look at the contents of the tokens.

**Note:** the format and data stored in the ID token may change based on your provider and configuration. Check the documentation and configuration of your own provider for details.

These methods can be used wherever a specific token is required. For example, if you are accessing a backend API that requires you to include a bearer token, you can use `getAccessToken()` to create an HTTP interceptor that adds the token to HTTP requests.

An interceptor isn't needed for this app, but as a challenge to you, update `Tab1.tsx` to show the user's name when they are logged in. You could also display the access token if you want (though you'd _never_ do that in a real app).

---

### Refreshing the authentication session

```tsx AuthProvider.tsx focus=47:68
import { AuthConnect, ProviderOptions, Auth0Provider } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const provider = new Auth0Provider();

export const AuthContext = createContext<{
  isAuthenticated: boolean;
  getAccessToken: () => Promise<string | undefined>;
  getUserName: () => Promise<string | undefined>;
  login: () => Promise<void>;
  logout: () => Promise<void>;
}>({
  isAuthenticated: false,
  getAccessToken: () => { throw new Error("Method not implemented."); },
  getUserName: () => { throw new Error("Method not implemented."); },
  login: () => { throw new Error("Method not implemented."); },
  logout: () => { throw new Error("Method not implemented."); }
});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);
  const [authResult, setAuthResult] = useState<AuthResult | null>(null);
  const [isAuthenticated, setIsAuthenticated] = useState<boolean>(false);

  const refreshAuth = async (authResult: AuthResult): Promise<AuthResult | null> => {
    let newAuthResult: AuthResult | null = null;

    if (await AuthConnect.isRefreshTokenAvailable(authResult)) {
      try {
        newAuthResult = await AuthConnect.refreshSession(provider, authResult);
      } catch (err) {
        console.log("Error refreshing session.", err);
      }
    }

    return newAuthResult;
  };

  const getAuthResult = async (): Promise<AuthResult | null> => {
    if (authResult && (await AuthConnect.isAccessTokenExpired(authResult))) {
      const newAuthResult = await refreshAuth(authResult);
      setAuthResult(newAuthResult);
    }
    setIsAuthenticated(!!authResult);
    return authResult;
  };

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  const login = async (): Promise<void> => {
    const authResult = await AuthConnect.login(provider, options);
    setAuthResult(authResult);
    setIsAuthenticated(true);
  };

  const logout = async (): Promise<void> => {
    if (authResult) {
      await AuthConnect.logout(provider, authResult);
      setAuthResult(null);
      setIsAuthenticated(false);
    }
  };

  const getAccessToken = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    return res?.accessToken;
  };

  const getUserName = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    if (res) {
      const data = await AuthConnect.decodeToken<{ name: string }>(
        TokenType.id,
        res
      );
      return data?.name;
    }
  };

  return (
    <AuthContext.Provider value={{ 
      isAuthenticated, getAccessToken, getUserName, login, logout 
    }}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```

In a typical OIDC implementation, access tokens are very short lived. It is common to use a longer lived refresh token to obtain a new `AuthResult`.

Add a function that performs a refresh, then modify `getAuthResult()` to refresh the session when needed. Now anything using `getAuthResult()` to get the current auth result will automatically trigger a refresh if needed. 

---

## Store the Auth Result

So far the `AuthResult` has been stored in a local state variable, which has a couple of disadvantages:

- Tokens could show up in a stack trace.
- Tokens do not survive a browser refresh or application restart.

There are several options we could use to store the AuthResult, but one that handles persistence as well as storing the data in a secure location on native devices is <a href="https://ionic.io/docs/identity-vault" target="_blank">Ionic Identity Vault</a>.

For our application, we will install Identity Vault and use it in "Secure Storage" mode to store the tokens. 

<CH.Section lineNumbers={false}>

```bash Terminal
npm install @ionic-enterprise/identity-vault
```

</CH.Section>

---

### Create a Vault factory

```tsx SessionVaultProvider.tsx
import { BrowserVault, IdentityVaultConfig, Vault } from "@ionic-enterprise/identity-vault";
import { isPlatform } from "@ionic/react";

const createVault = (config: IdentityVaultConfig): Vault | BrowserVault => {
  return isPlatform("hybrid") ? new Vault(config) : new BrowserVault(config);
};
```

Create `SessionVaultProvider.tsx` within `src/providers`. In this file setup a factory that builds either the actual Vault - if we are on a device - or a browser-based "Vault" suitable for development in the browser. 

This provides us with a secure Vault on device, or a fallback Vault that allows us to keep using our browser-based development flow.

---

### Set up a React context

```tsx SessionVaultProvider.tsx focus=8:53
import { BrowserVault, IdentityVaultConfig, Vault } from "@ionic-enterprise/identity-vault";
import { isPlatform } from "@ionic/react";

const createVault = (config: IdentityVaultConfig): Vault | BrowserVault => {
  return isPlatform("hybrid") ? new Vault(config) : new BrowserVault(config);
};

const key = 'auth-result';
const vault = createVault({
  key: 'io.ionic.gettingstartedacreact',
  type: VaultType.SecureStorage,
  deviceSecurityType: DeviceSecurityType.None,
  lockAfterBackgrounded: 5000,
  shouldClearVaultAfterTooManyFailedAttempts: true,
  customPasscodeInvalidUnlockAttempts: 2,
  unlockVaultOnLoad: false,
});

export const SessionVaultContext = createContext<{
  clearSession: () => Promise<void>;
  getSession: () => Promise<AuthResult | null>;
  setSession: (value?: AuthResult) => Promise<void>;
}>({
  clearSession: () => {
    throw new Error('Method not implemented.');
  },
  getSession: () => {
    throw new Error('Method not implemented.');
  },
  setSession: () => {
    throw new Error('Method not implemented.');
  },
});

export const SessionVaultProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const clearSession = (): Promise<void> => {
    return vault.clear();
  };

  const getSession = (): Promise<AuthResult | null> => {
    return vault.getValue<AuthResult>(key);
  };

  const setSession = (value?: AuthResult): Promise<void> => {
    return vault.setValue(key, value);
  };

  return (
    <SessionVaultContext.Provider value={{ clearSession, getSession, setSession }}>
      {children}
    </SessionVaultContext.Provider>
  );
};
```

Like `AuthProvider`, a React Context will be used to expose functionality related to managing the authentication result. 

Don't forget to wrap `IonReactRouter` with the new provider so the Context can be shared with the Tabs.

<CH.Section rows={3}>

```tsx src/App.tsx focus=41:42,75:76
import { Redirect, Route } from "react-router-dom";
import {
  IonApp,
  IonIcon,
  IonLabel,
  IonRouterOutlet,
  IonTabBar,
  IonTabButton,
  IonTabs,
  setupIonicReact,
} from "@ionic/react";
import { IonReactRouter } from "@ionic/react-router";
import { ellipse, square, triangle } from "ionicons/icons";
import Tab1 from "./pages/Tab1";
import Tab2 from "./pages/Tab2";
import Tab3 from "./pages/Tab3";

/* Core CSS required for Ionic components to work properly */
import "@ionic/react/css/core.css";

/* Basic CSS for apps built with Ionic */
import "@ionic/react/css/normalize.css";
import "@ionic/react/css/structure.css";
import "@ionic/react/css/typography.css";

/* Optional CSS utils that can be commented out */
import "@ionic/react/css/padding.css";
import "@ionic/react/css/float-elements.css";
import "@ionic/react/css/text-alignment.css";
import "@ionic/react/css/text-transformation.css";
import "@ionic/react/css/flex-utils.css";
import "@ionic/react/css/display.css";

/* Theme variables */
import "./theme/variables.css";

setupIonicReact();

const App: React.FC = () => (
  <IonApp>
    <SessionVaultProvider>
      <AuthProvider>
        <IonReactRouter>
          <IonTabs>
            <IonRouterOutlet>
              <Route exact path="/tab1">
                <Tab1 />
              </Route>
              <Route exact path="/tab2">
                <Tab2 />
              </Route>
              <Route path="/tab3">
                <Tab3 />
              </Route>
              <Route exact path="/">
                <Redirect to="/tab1" />
              </Route>
            </IonRouterOutlet>
            <IonTabBar slot="bottom">
              <IonTabButton tab="tab1" href="/tab1">
                <IonIcon aria-hidden="true" icon={triangle} />
                <IonLabel>Tab 1</IonLabel>
              </IonTabButton>
              <IonTabButton tab="tab2" href="/tab2">
                <IonIcon aria-hidden="true" icon={ellipse} />
                <IonLabel>Tab 2</IonLabel>
              </IonTabButton>
              <IonTabButton tab="tab3" href="/tab3">
                <IonIcon aria-hidden="true" icon={square} />
                <IonLabel>Tab 3</IonLabel>
              </IonTabButton>
            </IonTabBar>
          </IonTabs>
        </IonReactRouter>
      </AuthProvider>
    </SessionVaultProvider>
  </IonApp>
);

export default App;
```

</CH.Section>

---

### Modify the Auth context

```tsx AuthProvider.tsx focus=45,48:56,89,95
import { AuthConnect, ProviderOptions, Auth0Provider, TokenType } from "@ionic-enterprise/auth";
import { isPlatform } from "@ionic/react";
import { PropsWithChildren, createContext } from "react";
import { SessionVaultContext } from './SessionVaultProvider';

const isNative = isPlatform("hybrid");

const options: ProviderOptions = {
  audience: "https://io.ionic.demo.ac",
  clientId: "yLasZNUGkZ19DGEjTmAITBfGXzqbvd00",
  discoveryUrl:
    "https://dev-2uspt-sz.us.auth0.com/.well-known/openid-configuration",
  logoutUrl: isNative ? "msauth://login" : "http://localhost:8100/login",
  redirectUri: isNative ? "msauth://login" : "http://localhost:8100/login",
  scope: "openid offline_access email picture profile",
};

const setupAuthConnect = async (): Promise<void> => {
  return AuthConnect.setup({
    platform: isNative ? "capacitor" : "web",
    logLevel: "DEBUG",
    ios: { webView: "private" },
    web: { uiMode: "popup", authFlow: "implicit" },
  });
};

export const provider = new Auth0Provider();

export const AuthContext = createContext<{
  isAuthenticated: boolean;
  getAccessToken: () => Promise<string | undefined>;
  getUserName: () => Promise<string | undefined>;
  login: () => Promise<void>;
  logout: () => Promise<void>;
}>({
  isAuthenticated: false,
  getAccessToken: () => { throw new Error("Method not implemented."); },
  getUserName: () => { throw new Error("Method not implemented."); },
  login: () => { throw new Error("Method not implemented."); },
  logout: () => { throw new Error("Method not implemented."); }
});

export const AuthProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [isSetup, setIsSetup] = useState<boolean>(false);
  const { clearSession, getSession, setSession } = useContext(SessionVaultContext);
  const [isAuthenticated, setIsAuthenticated] = useState<boolean>(false);

  const saveAuthResult = async (authResult: AuthResult | null): Promise<void> => {
    if (authResult) {
      await setSession(authResult);
      setIsAuthenticated(true);
    } else {
      await clearSession();
      setIsAuthenticated(false);
    }
  };

  const refreshAuth = async (authResult: AuthResult): Promise<AuthResult | null> => {
    let newAuthResult: AuthResult | null = null;

    if (await AuthConnect.isRefreshTokenAvailable(authResult)) {
      try {
        newAuthResult = await AuthConnect.refreshSession(provider, authResult);
      } catch (err) {
        console.log("Error refreshing session.", err);
      }
    }

    return newAuthResult;
  };

  const getAuthResult = async (): Promise<AuthResult | null> => {
    let authResult = await getSession();

    if (authResult && (await AuthConnect.isAccessTokenExpired(authResult))) {
      const newAuthResult = await refreshAuth(authResult);
      await saveAuthResult(newAuthResult);
    }
    setIsAuthenticated(!!authResult);
    return authResult;
  };

  useEffect(() => {
    setupAuthConnect().then(() => setIsSetup(true));
  }, []);

  const login = async (): Promise<void> => {
    const authResult = await AuthConnect.login(provider, options);
    await saveAuthResult(authResult);
  };

  const logout = async (): Promise<void> => {
    if (authResult) {
      await AuthConnect.logout(provider, authResult);
      await saveAuthResult(authResult);
    }
  };

  const getAccessToken = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    return res?.accessToken;
  };

  const getUserName = async (): Promise<string | undefined> => {
    const res = await getAuthResult();
    if (res) {
      const data = await AuthConnect.decodeToken<{ name: string }>(
        TokenType.id,
        res
      );
      return data?.name;
    }
  };

  return (
    <AuthContext.Provider value={{ 
      isAuthenticated, getAccessToken, getUserName, login, logout 
    }}>
      {isSetup && children}
    </AuthContext.Provider>
  );
};
```


The goal is to no longer store the auth result in a session variable. Instead, we will use the session Vault to store the result and retrieve it as needed.

You should now be able to refresh the app and have a persistent session. The tutorial is now complete!

</CH.Scrollycoding>
