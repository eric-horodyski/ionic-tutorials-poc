---
id: Getting Started with Auth Connect in @ionic-react
slug: /
sidebar_position: 1
---

<CH.Scrollycoding>

Learn how to get started with Auth Connect by creating a sample project that uses `@ionic/react`.

This tutorial starts with installing and configuring Auth Connect and moves onto achieving common use-cases: performing login/logout, checking the user's authentication status, obtaining tokens from Auth Connect, and storing tokens using <a href="https://ionic.io/docs/identity-vault" target="_blank">Identity Vault</a>.

<CH.Code>

{/_ prettier-ignore _/}

```typescript App.tsx
// Completed output here.
```

{/_ prettier-ignore _/}

```typescript AuthProvider.tsx
// Completed output here.
```

{/_ prettier-ignore _/}

```typescript Tab1.tsx
// Completed output here.
```

{/_ prettier-ignore _/}

```typescript SessionVaultProvider.tsx
// Completed output here.
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

### Scaffold the React Provider

Create a folder `src/providers` and scaffold a file named `AuthProvider.tsx`.

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

---

### Wrap `IonReactRouter`

Open `src/App.tsx` and wrap `IonReactRouter` with the provider.

```tsx App.tsx focus=41,42,73,74
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

</CH.Scrollycoding>
