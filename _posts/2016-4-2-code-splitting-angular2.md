---
layout: post
comments: true
title:  "Code Splitting in Angular 2"
date:   2016-4-2
published: true
---

[Code splitting](https://webpack.js.org/guides/code-splitting/) generally refers to splitting up your JavaScript application into multiple files and only load what you need when you need it. This is an optimzation that decreases initial download size which improves the initial startup time of your app. The following techniques illustrate how we can achieve this with Webpack@2.2.0-rc.1 and Angular 2. The code for this example can be found at <https://github.com/songawee/angular2_code_splitting>.

As JavaScript applications gain more and more features, so grows the filesize of the application bundle. The larger the bundle, the longer it takes for the user to get to the app. Not only do users need to download all of the information, they need to parse it as well. Another approach to bundling everything is bundling a very small initial bundle that contains only the most relevant code for that particular view or route and then dynamically load the other modules at a later time. This separation of modules can bring down application load times and increase user productivity.

Webpack 2 provides us with `System.import('../path/to/file.js')` syntax (and soon [`import` syntax](https://github.com/tc39/proposal-dynamic-import)) to achieve loading of the additional chunks via a Promise based api. Webpack processes `System.import` by compiling `file.js` and its dependencies into a separate file and then dynamically loading that code when that promise is resolved.

```js
  const importedChunk = System.import('./foo').then((foo) => foo.default);
```

We have a way to split and load the pieces of our application, but we still need to know where to split. It ultimately depends on your app, but it's generally a good idea to split up your app based on views or pages and load the additional chunks dynamically on navigation. With single page apps, you can think of a main page and profile page and loading the code for the profile page when you navigate from the main page.

Let's say we have the following example Angular 2 application.

#### AppModule.ts

```js
// AppModule.ts
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';

import { routes } from './Routing';

import AppComponent from './AppComponent';
import InlineComp from './InlineComp';

@NgModule({
  declarations: [
    AppComponent,
    InlineComp
  ],
  imports: [ BrowserModule, routes ],
  bootstrap: [ AppComponent ]
})
export default class AppModule {}

```

#### AppComponent.ts

```js
// AppComponent.ts
import { Component } from '@angular/core';

@Component({
  selector: 'yo',
  template: `
    <h1>yo</h1>
    <router-outlet></router-outlet>
  `
})
export default class AppComponent {}
```

#### InlineComp.ts

```js
// InlineSubComp.ts
import { Component } from '@angular/core';

@Component({
  selector: 'inline-comp',
  template: `
    <div>
      <p>This component is the default component and was loaded inline.</p>
    </div>
  `
})
export default class InlineComp {}
```

#### Routing.ts

```js
// Routing.ts
import { ModuleWithProviders } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';

import InlineComp from './InlineComp';

const appRoutes: Routes = [
  {
    path: '',
    component: InlineComp
  }
];

export const routes: ModuleWithProviders = RouterModule.forRoot(appRoutes);
```

This is a really simple application using Angular 2's routing solution that just puts some text on the page. We have configured the routing so that when a user navigates to `/`, we render `InlineComp` within the `router-outlet` in `AppComponent.ts`.

The result is:

![code splitting](/img/inline.png)

Now, let's load the following component on demand to replace the inline component when a user clicks a link.

```js
// DynamicSubComp.ts
import { Component, NgModule } from '@angular/core';
import { RouterModule } from '@angular/router';

import '../styles/DynamicSubComp.scss';

@Component({
  selector: 'dynamic-sub-comp',
  template: `
    <div class="addMargin">
      <div><strong>Sub Sandwiches!</strong></div>
      <div>This sub component was loaded dynamically... along with its corresponding CSS styles</div>
    </div>
  `
})
class DynamicSubComp {}

export const routes = [
  { path: '', component: DynamicSubComp, pathMatch: 'full' }
];

@NgModule({
  declarations: [
    DynamicSubComp
  ],
  imports: [
    RouterModule.forChild(routes)
  ]
})
export default class DynamicModule {
  static routes = routes;
}

```

<br />

```css
/* DynamicSubComp.scss */
.addMargin {
  margin: 10px;
}
```

Using [Angular 2's modules](https://angular.io/docs/ts/latest/guide/ngmodule.html), we can encapsulate sub routes and dependencies in NgModules. In this way, when we resolve this chunk, we are also resolving its code dependencies as well as defining all of the sub routes associated with this module.

Notice also that we're additionally importing a `.scss` file. This is made possible by using the [sass-loader](https://github.com/jtangelder/sass-loader). The Sass imported into this module will be bundled together with the JS and loaded on demand as well.

Let's add the link in the inline component for users to click to add the dynamic component.

```js
@Component({
  selector: 'inline-comp',
  template: `
  <div>
    <p>This component is the default component and was loaded inline.</p>
    <a routerLink="/dynamic" routerLinkActive="active">Load a dynamic component</a>
  </div>`
})
export default class InlineComp {}
```

Note that the dynamic component will eventually be rendered through the <router-outlet> tag in AppComponent.

By defining a route with a `loadChildren` property, we have a handle to load another Angular module asynchronously. Now, we have a place to use the `System.import()` syntax to return a promise that contains our component.

```js
// Routing.ts
const appRoutes: Routes = [
  {
    path: 'dynamic',
    loadChildren: () => {
      return System.import('./DynamicSubComp').then((comp: any) => {
        return comp.default;
      });
    }
  },
  {
    path: '',
    component: InlineComp
  }
];
```

Now, when we visit the `/dynamic` path from clicking the link in InlineComp, Angular will know to resolve this Promise before trying to render anything for that route.

With ES2015 modules, we use the `default` export off of our component and are now loading the component asynchronously! If you have other exports, you can reference them similarly i.e.

```js
loadChildren: () => {
  return System.import('./DynamicSubComp').then((comp: any) => {
    return comp.otherExport;
  });
}
```

The result is:

![Dynamic Comp](/img/dynamic.gif)

Notice how the `0.bundle.js` file is downloaded separately!

#### System.import with TypeScript and Webpack

Before we start using `System.import`, we need to make sure we have the right type definition for this function. Since Webpack 2 shares similar syntax to SystemJS, we can import the SystemJS type definitions until Webpack's `System.import` definition is defined.

Note: the @types package is built from the [https://github.com/DefinitelyTyped/DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) repo definitions.

```bash
npm i -D @types/systemjs
```
