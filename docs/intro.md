---
id: Getting Started with Auth Connect in @ionic-react
slug: /
sidebar_position: 1
---

<CH.Scrollycoding>

<CH.Code>

```bash Terminal
ionic start getting-started-ac-react-tabs --type=react

npm run build
ionic cap add android
ionic cap add ios

npm install @ionic-enterprise/auth

npm install @ionic-enterprise/identity-vault
```

{/_ prettier-ignore _/}

```typescript capacitor.config.ts
import { CapacitorConfig } from "@capacitor/cli";

const config: CapacitorConfig = {
  appId: "io.ionic.starter",
  appName: "getting-started-ac-react",
  webDir: "build",
  bundledWebRuntime: false,
};

export default config;
```

{/_ prettier-ignore _/}

```json package.json
{
  "name": "getting-started-ac-react",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    /* Omitted for brevity */
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
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

{/_ prettier-ignore _/}

```tsx index.tsx
import React from "react";
import { createRoot } from "react-dom/client";
import App from "./App";
import * as serviceWorkerRegistration from "./serviceWorkerRegistration";
import reportWebVitals from "./reportWebVitals";

const container = document.getElementById("root");
const root = createRoot(container!);
root.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

// If you want your app to work offline and load faster, you can change
// unregister() to register() below. Note this comes with some pitfalls.
// Learn more about service workers: https://cra.link/PWA
serviceWorkerRegistration.unregister();

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
```

{/_ prettier-ignore _/}

```tsx App.tsx
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
  </IonApp>
);

export default App;
```

</CH.Code>

This tutorial walks through the basic setup and use of Ionic's Auth Connect in an `@ionic/react` application.

In this tutorial, you will learn how to:

- Install and configure Auth Connect
- Perform Login and Logout operations
- Check if the user is authenticated
- Obtain tokens from Auth Connect
- Integrate Identity Vault with Auth Connect

---

## Generate the Application

Start by generating an Ionic React application using the `tabs` template.

```bash Terminal focus=1

```

---

### Set a unique app identifier

```typescript capacitor.config.ts  focus=4
import { CapacitorConfig } from "@capacitor/cli";

const config: CapacitorConfig = {
  appId: "io.ionic.gettingstartedacreact",
  appName: "getting-started-ac-react",
  webDir: "build",
  bundledWebRuntime: false,
};

export default config;
```

Ionic recommends changing the app identifier _before_ adding any deployment platforms. If you want to change the app identifier after adding iOS or Android, you will have to change native artifacts.

---

### Remove React Strict Mode

```typescript index.tsx focus=9
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

In development mode, `useEffect`s are triggered twice. This will impact integrating with Auth Connect, so we will remove React's Strict Mode for this tutorial.

---

### Add the native platforms

```bash Terminal focus=3:5

```

Build the application then install and create the iOS and Android platforms.

---

### Modify the build script

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

Modify the `build` npm script to sync Capacitor with each build.

---

## Install Auth Connect

```bash Terminal focus=7

```

In order to install Auth Connect, you will need to use `ionic enterprise register` to register your product key. This will create a `.npmrc` file containing the product key.

If you have already performed that step for your production application, you can just copy the `.npmrc` file from your production project. Since this application is for learning purposes only, you don't need to obtain another key.

</CH.Scrollycoding>
