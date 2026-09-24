# App structure and routing

How a CxJS application starts, how routes are organized, and how pages get their own data. See [Routing](https://cxjs.io/docs/concepts/routing.md), [Route](https://cxjs.io/docs/concepts/route.md) and [History](https://cxjs.io/docs/concepts/history.md) for every option.

## Entry point

```tsx
// index.tsx
import { Store } from "cx/data";
import { History, startHotAppLoop } from "cx/ui";
import { enableMsgBoxAlerts, enableTooltips } from "cx/widgets";
import Routes from "./routes";

enableTooltips();
enableMsgBoxAlerts();

const store = new Store();
History.connect(store, "url");

startHotAppLoop(module, document.getElementById("app")!, store, <Routes />);
```

- `History.connect(store, "url")` keeps the browser location in the store's `url`, using `pushState`. Routes bind to it.
- With Vite, pass `{ hot: import.meta.hot }` instead of `module`, and add `if (import.meta.hot) import.meta.hot.accept();` in the same file.
- If the app is not served from the site root, call `Url.setBase("/my-app/")` before connecting.

## The root model

The application-wide data — the current `url`, the page storage, the signed-in user — lives in a root model:

```ts
// model.ts (next to index.tsx)
interface AppModel {
  url: string;
  pages: Record<string, unknown>;
  user: User;
}

export default createModel<AppModel>();
```

## Page data: Sandbox per route

Each page keeps its data in `$page`, private to that page. A `Sandbox` provides it: it stores each page's data in `pages` under a key — by default the URL — and exposes the current page's slot as `$page`. Navigating to another page gives it fresh data; coming back restores the old.

Every app defines a small `SandboxedRoute` component that combines a `Route` with a `Sandbox`:

```tsx
// components/SandboxedRoute.tsx
import { createFunctionalComponent, type ChildNode, type StringProp } from "cx/ui";
import { Route, Sandbox } from "cx/widgets";
import app from "../model";

interface SandboxedRouteProps {
  route: string;
  prefix?: boolean;
  sandboxKey?: StringProp;
  children?: ChildNode | ChildNode[];
}

export const SandboxedRoute = createFunctionalComponent(
  ({ route, prefix, sandboxKey, children }: SandboxedRouteProps) => (
    <Route route={route} url={app.url} prefix={prefix}>
      <Sandbox accessKey={sandboxKey ?? app.url} storage={app.pages}>
        {children}
      </Sandbox>
    </Route>
  ),
);
```

- **Leaf pages** use the default key, the URL: `/orders/1` and `/orders/2` each get their own data.
- **A route with sub-pages** that should share data — tabs of one record, for example — passes a stable `sandboxKey`, so navigating between the sub-pages does not reset the page and re-run its controller.

A page declares `$page`, and the route parameters under `$route`, in its model:

```ts
// routes/orders/detail/model.ts
interface PageModel {
  $route: { id: string };
  $page: {
    order: Order;
    loading: boolean;
  };
}

export default createModel<PageModel>();
```

```ts
// routes/orders/detail/Controller.ts
export default class extends Controller {
  onInit() {
    this.load(this.store.get(m.$route.id));
  }
  // …
}
```

## Routes

```tsx
// routes/index.tsx
export default (
  <FirstVisibleChildLayout>
    <RedirectRoute route="~/" redirect="~/orders" url={app.url} />
    <SandboxedRoute route="~/orders" prefix>
      <Orders />
    </SandboxedRoute>
    <SandboxedRoute route="~/customers">
      <Customers />
    </SandboxedRoute>
    <NotFound />
  </FirstVisibleChildLayout>
);
```

```tsx
// routes/orders/index.tsx — a section with a list and a detail page
export default (
  <FirstVisibleChildLayout>
    <SandboxedRoute route="+">
      <List />
    </SandboxedRoute>
    <SandboxedRoute route="+/:id">
      <Detail />
    </SandboxedRoute>
  </FirstVisibleChildLayout>
);
```

- `~/` is the application root; `+/` is relative to the parent route, which must have `prefix` so that sub-paths match.
- Patterns: `:id` a parameter (available as `$route.id`), `*path` the rest of the path, `(…)` an optional part — e.g. `~/orders(/)`.
- `FirstVisibleChildLayout` renders only the first matching route, so a final child without a route becomes the not-found page.

## Navigation

- **Links**: `<Link href="~/orders" url={app.url} match="prefix">Orders</Link>` — `url` marks the link as active when it matches; `match` is `equal` (default), `prefix` or `subroute`.
- **From code**: `History.pushState({}, null, "~/orders/42")`, or `History.replaceState` to replace the current entry.
- **Unsaved changes**: `History.addNavigateConfirmation((url) => MsgBox.yesNo("Discard changes?").then((a) => a == "yes"))` asks before leaving the page.

Hash-based routing (`#/` links) is rare — use it only when the server cannot serve the application for every URL on refresh. A project that uses it already has it set up; follow it.

## Lazy loading

Load **large** sections on demand; keep small routes in the main bundle.

```tsx
<SandboxedRoute route="~/reports" prefix>
  <ContentResolver onResolve={() => import("./reports").then((x) => x.default)}>
    <div class="loading">Loading…</div>
  </ContentResolver>
</SandboxedRoute>
```

- The children are shown until the chunk arrives.
- `ContentResolver` without `params` resolves once. Do not add dummy params such as `params={1}`.
