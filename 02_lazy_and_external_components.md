# Angular Elements, Part II: Lazy and External Web Components


<hr>
<p><b style="font-weight: bold">Table of Contents</b></p>
<p>This blog post is part of an article series.</p>
<p>
<ul>
<li><a href="https://www.softwarearchitekt.at/post/2018/07/13/angular-elements-part-i-a-dynamic-dashboard-in-four-steps-with-web-components.aspx">Angular Elements, Part I: A Dynamic Dashboard In Four Steps With Web Components</a></li>
<li><a href="https://www.softwarearchitekt.at/post/2018/07/29/angular-elements-part-ii-lazy-and-external-web-components.aspx" rel="nofollow">Angular Elements, Part II: Lazy And External Web Components</a></li>
</ul>
</p>
<p><b style="font-weight: bold">See also:</b></p>
<p>
<ul>
<li><a href="https://www.softwarearchitekt.at/post/2018/05/04/microservice-clients-with-web-components-using-angular-elements-dreams-of-the-near-future.aspx">Micro Apps With Web Components Using Angular Elements</a></li>
</ul>
</p>
<hr>

In the [first part of this series](https://www.softwarearchitekt.at/post/2018/07/13/angular-elements-part-i-a-dynamic-dashboard-in-four-steps-with-web-components.aspx), I've shown how to leverage Angular Elements for dynamically adding components to a page. The discussed example dynamically adds tiles to a dashboard.

In this article, I'm going one step further: I'll extend the shown example by loading the Web Components on demand. For this, I demonstrate two approaches: Lazy Loading and loading external components.

<!-- Picture -->

The [solution](https://github.com/manfredsteyer/angular-elements-dashboard) can be found in my [GitHub repo](https://github.com/manfredsteyer/angular-elements-dashboard).

## Lazy Loading vs. Loading External Components

Perhaps you are wondering what the differences between to two outlined approaches -- lazy loading and loading external components -- are.

Lazy Loading demands on compiling the component and its hosting application together. This allows for optimizations like tree shaking but also limits your possibilities as the application needs to know all possible web components in advance. It's more or less like what you know from the first article. In addition, you have to leverage Angular's and the CLI's features for code splitting and lazy loading.

On the other side, you could also put your web component and all libraries it depends on -- like ``@angular/core`` or ``@angular/elements`` -- into one self-contained bundle. After this, you can load this bundle in your host application. This increases bundle sizes but also gives you more flexibility as the host can dynamically load components not known at build time. The upcoming ngIvy compiler will help a lot with shrinking such bundles to a minimum. Also, my simple CLI extension [ngx-build-plus](https://www.npmjs.com/package/ngx-build-plus) allows to share common dependencies between different bundles. I will talk about those options in a later blog post. 

Of course, what I'm calling "loading external components" here, is also lazy loading. But as it is not the kind of lazy loading Angular provides out of the box, I've decided to use this paraphrase.

## Implementing Lazy Loading (without the Router)

Lazy Loading is baked into Angular since its first days. There are low level APIs for it and the router provides a nice abstraction that makes this concept easy to use. However, in the example shown here, using the router is not beneficial because this is not about loading routes but loading some tiles into a dashboard on demand.

That's why I'm leveraging a quite new feature the CLI provides since version 6. Using it, you can point to specific modules which are split off during bundling. After this, you can make use of the mentioned low level APIs to load those modules on demand. 

To get started, reference the module file(s) with your web components in your ``angular.json``:

```json
"lazyModules": [
  "src/app/lazy-dashboard-tile/lazy-dashboard-tile.module"
],
```

The next listing shows how you can leverage the ``NgModuleFactoryLoader`` to load the bundle:

```typescript
@Injectable({
    providedIn: 'root'
})
export class LazyDashboardTileService  {

    constructor(
        private loader: NgModuleFactoryLoader,
        private injector: Injector
    ) {
    }

    private moduleRef: NgModuleRef<any>;

    load(): Promise<void> {
        
        if (this.moduleRef) {
            return Promise.resolve();
        }

        const path = 'src/app/lazy-dashboard-tile/lazy-dashboard-tile.module#LazyDashboardTileModule'
        
        return this
            .loader
            .load(path)
            .then(moduleFactory => {
                this.moduleRef = moduleFactory.create(this.injector).instance;
                console.debug('moduleRef', this.moduleRef);
            })
            .catch(err => {
                console.error('error loading module', err); 
            });
        
    }
}
```

For the sake of simplicity, I'm not taking care of every possible race condition. As with the router, you have to provide a string with both, the filename of the module as well as the name of the module class. After loading it, you have to instantiate the module with ``create``. 

After this, you could search this instance for components, services etc. However, this is not easy due to the lack of respective APIs. The good message is that you don't have to this when going with web components: As they directly register with the browser, all you need is to create html elements with the right names. For instance, the next listing creates a ``lazy-dashboard-tile`` element:

```typescript
const tile = document.createElement('lazy-dashboard-tile');
tile.setAttribute('class', 'col-lg-4 col-md-3 col-sm-2');
tile.setAttribute('a', '100');
tile.setAttribute('b', '50');
tile.setAttribute('c', '25');

const content = document.getElementById('content');
content.appendChild(tile);
```

Also, you have to make sure that the web components are registered when the module is loaded. To achieve that, you could put the necessary code into the module's constructor:

```typescript
@NgModule({
  […],
  declarations: [
    […]
    DashboardTileComponent
  ],
  entryComponents: [
    DashboardTileComponent
  ]
})
export class DashboardModule { 

  constructor(private injector: Injector) {

    const tileCE = createCustomElement(DashboardTileComponent, { injector: this.injector });
    customElements.define('dashboard-tile', tileCE);

  }

}
```

Don't forget to put the component in question not only into the module's ``declarations`` section but also into its ``entryComponents`` array.

## Loading External Components

For providing an external Web Component, you can just scaffold a new Angular application and make sure the Angular Element is registered when it starts up. For this, I'm using the ``AppModule``'s ``ngDoBootstrap`` method:

```typescript
@NgModule({
   […],
   declarations: [
       ExternalDashboardTileComponent
   ],
   bootstrap: [],
   entryComponents: [
       ExternalDashboardTileComponent
   ]
})
export class AppModule { 

    constructor(private injector: Injector) {
    }

    ngDoBootstrap() {
        const externalTileCE = createCustomElement(ExternalDashboardTileComponent, { injector: this.injector });
        customElements.define('external-dashboard-tile', externalTileCE);
    }

}
```

Please also note that this example doesn't define an ``bootstrap`` component. The reason is, I don't want to load an Angular Component on startup but just register a web component. To test this component, just call the web component directly in your ``index.html`` and ``ng serve`` your project:

```html
<external-dashboard-tile a="50" b="60" c="70">
</external-dashboard-tile>
```

For publishing our web components, we need one self-contained bundle that can be loaded into a host application. However, the current version of the CLI always creates several bundles. To solve this, we can use [ngx-build-plus](https://www.npmjs.com/package/ngx-build-plus) -- a simple extension for the CLI:

```
npm i ngx-build-plus --save-dev
```

After installing it, update your application's ``builder`` section within the ``angular.json`` file so that it points to ``ngx-build-plus``:

```json
"builder": "ngx-build-plus:build",
```

After building your application, you should end up with one self-contained main bundle. In addition, you might also get other bundles, e. g. bundles with external scripts or polyfills. But everything you directly need to run your web component -- your code and the libraries it depends on -- ends up in the main bundle. 

In the example shown here, I'm using a build task in my ``package.json`` for coping over this bundle into the host's ``assets`` folder.

To dynamically load the web component into the host, you just need some DOM manipulations to create a respective ``script`` tag as well as a tag for the component itself:

```typescript
// add script tag
const script = document.createElement('script');
script.src = 'assets/external-dashboard-tile.bundle.js';
document.body.appendChild(script);

// add web component
const tile = document.createElement('dashboard-tile');
tile.setAttribute('class', 'col-lg-4 col-md-3 col-sm-2');
tile.setAttribute('a', '100');
tile.setAttribute('b', '50');
tile.setAttribute('c', '25');

const content = document.getElementById('content');
content.appendChild(tile);
```