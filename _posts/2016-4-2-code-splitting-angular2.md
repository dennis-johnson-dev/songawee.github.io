---
layout: post
comments: true
title:  "Code Splitting in Angular 2"
date:   2016-4-2
published: true
---

[Code splitting](https://webpack.github.io/docs/code-splitting.html) generally refers to splitting up your JavaScript application into multiple files and only load what you need when you need it. This is an optimzation that decreases initial download size which improves the initial startup time of your app. The following techniques illustrate how we can achieve this with Webpack 2 beta and Angular 2. The code for this example can be found at <https://github.com/songawee/angular2_code_splitting>.

As JavaScript applications gain more and more features, so grows the filesize of the application bundle. The larger the bundle, the longer it takes for the user to get to the app. Not only do users need to download all of the information, they need to parse it as well. Another approach is bundling a very small initial bundle that contains only the most relevant code for that particular view and then dynamically load the other modules at a later time. This separation of modules can bring down application load times and increase user productivity.

Webpack 2 provides us with `System.import('../path/to/file.js')` syntax to achieve loading the additional chunks via a Promise based api. Webpack processes `System.import` by compiling `file.js` and its dependencies into a separate file and then dynamically loading that code when you resolve that import promise. `System.import` is actually the [loader](https://whatwg.github.io/loader/) part of the ES modules specificiation and we are able to use the syntax with Webpack as a polyfill.

```js
  const importedChunk = System.import('./foo').then((foo) => foo.default);
```

We have a way to split and load the pieces of our application, but we still need to know where to split. It ultimately depends on your app, but it's generally a good idea to split up your app based on views or pages and load the additional chunks dynamically on navigation. With single page apps, you can think of a main page and profile page and loading the code for the profile page when you navigate from the main page.

Let's say we have the following example Angular 2 application.

#### AppModule.ts

```js
// AppModule.ts
import { routes } from './Routing';

import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';

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
export default class App {}

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

Notice that we're also importing a `.scss` file. This is made possible by using the [sass-loader](https://github.com/jtangelder/sass-loader). The Sass imported into this module will be bundled together with the JS and loaded on demand as well.

Let's add the link in the App component for users to click to add the dynamic component.

```js
@Component({
  selector: 'yo',
  template: `
    <div>
      <h1>Code Splitting in Angular 2</h1>
      <div>
        <a [routerLink]="['LoadDynamicSub']">Load Sub Component</a>
      </div>
      <router-outlet></router-outlet>
    </div>
  `,
  directives: [ ROUTER_DIRECTIVES ]
})
```

By defining an [AsyncRoute](https://angular.io/docs/js/latest/api/router/AsyncRoute-class.html) in our RouteConfig decorator, we just need to provide a function that returns a promise as the `loader` config for that route. Now, we have a place to use the `System.import()` syntax to return a promise that contains our component.

```js
// AppModule.ts
new AsyncRoute({
  path: '/sub',
  loader: () => System.import('./DynamicSubComp').then((comp: any) => {
    return comp.default;
  }),
  name: 'LoadDynamicSub'
})
```

With ES2015 modules, we use the `default` export off of our component and are now loading the component asynchronously! If you have other exports, you can reference them similarly i.e.

```js
loader: () => System.import('./DynamicSubComp').then((comp: any) => {
  return comp.otherExportFoo;
}),
```

The result is:

![Dynamic Comp](/img/dynamic.png)

Notice how the `1.bundle.js` file is downloaded separately.

#### System.import with TypeScript and Webpack

Before we start using `System.import`, we need to make sure we have the right type definition for this function. Since Webpack 2 shares similar syntax to SystemJS, we can import the SystemJS type definitions until Webpack's `System.import` definition is defined.

```bash
typings install systemjs
```

Typings is a type definition manager that let's us easily add definitions for use with our own typescript files that utilize these external modules.
