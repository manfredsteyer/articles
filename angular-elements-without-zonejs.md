# Angular Elements without zone.js

Since Version 6, we can very easily create Web Components with Angular. To be more precise, we should speak about Custom Elements -- a standard behind the umbrella term Web Components that allows for creating own HTML-Elements.

However, Angular depends on zone.js for Change Tracking and in most cases we don't want to force the customers of our widgets into using it.

In this short article, I explain why excluding zone.js is a good idea and how to deal with the consequences. The sample I use for this can be found in my [github repo](https://github.com/manfredsteyer/angular-elements-dashboard/tree/noop-zone). Make sure to use the branch ``noop-zone``.


## Why zone.js might be a bad idea for Custom Elements

In general, we want our custom Elements to be as small as possible in terms of bundle size. The upcoming ngIvy view engine will help a lot with this goal as it produces more tree-shakable code and hence allows Angular blowing itself mostly away during the compilation. 

Another approach to shrink bundles is reusing Angular packages across several Angular Elements and the host application. After an enlightening discussion with Angular's [Rob Wormald](https://twitter.com/robwormald), I've created [ngx-build-plus](https://www.npmjs.com/package/ngx-build-plus) -- a simple CLI extension that helps to implement this idea.

However, in both cases we cannot get rid of ``zone.js`` which is used by Angular since it first days for change tracking. This library monkey patches a lot of browser objects to get informed about all events after which Angular needs to check the displayed components for changes. 

While this provides convenience in Angular application having such a dependency for a custom element is not desirable, especially when the hosting application is not Angular based: Not every consumer wants to monkey patch browser objects and in many cases ``zone.js`` is bigger than the custom element itself. 

## Getting rid of zone.js

Getting rid of ``zone.js`` is the easiest part. Just set configure the noop zone (no operation zone) when bootstrapping the Angular application:

```TypeScript
platformBrowserDynamic()
   .bootstrapModule(
       AppModule, { ngZone: 'noop' })
   .catch(err => console.log(err));
```

However, dealing with the consequences of removing ``zone.js`` isn't that easy as the consequence is triggering change detection manually.

## Triggering Change Detection manually

For my demonstrations, I use a simple Angular component that displays three numeric values:

```typescript
@Component({
  [...]
})
export class ExternalDashboardTileComponent {

  @Input() a: number;
  @Input() b: number;
  @Input() c: number;

  more(): void {
    this.a = Math.round(Math.random() * 100);
    this.b = Math.round(Math.random() * 100);
    this.c = Math.round(Math.random() * 100);
  }
}
```

It also provides a ``more`` method that updates those values. For the sake of simplicity, I use random numbers here.

The values are displayed in an table and the method is bound to the click event of a button:

```html
<table class="table table-condensed">
  <tr>
    <td>A</td>
    <td>{{a}}</td>
  </tr>
  <tr>
    <td>B</td>
    <td>{{b}}</td>
  </tr>
  <tr>
    <td>C</td>
    <td>{{c}}</td>
  </tr>
</table>

<button class="btn btn-default btn-sm" (click)="more()">More</button>
```

When using ``zone.js``, Angular is automatically performing change detection after the click event and hence updating the bound values. But without ``zone.js`` Angular is not aware of the click event. This means, we have to trigger change detection by hand. 

This can be accomplished by calling the ``markForCheck`` method of the current ``ChangeDetectorRef``:

```typescript
@Component({
  [...]
})
export class ExternalDashboardTileComponent {

  @Input() a: number;
  @Input() b: number;
  @Input() c: number;

  constructor(private cd: ChangeDetectorRef) {
  }

  more(): void {
    this.a = Math.round(Math.random() * 100);
    this.b = Math.round(Math.random() * 100);
    this.c = Math.round(Math.random() * 100);

    this.cd.markForCheck();
  }

}
```

As this is a very explicit approach, one can easily forget about calling the method at the right moment. Therefore I present an alternative in the next section.

## Push-Pipe

A more declarative way for triggering change detection is using Observables. Every time prove a new value, a pipe can tell Angular to check for changes. While Angular comes with the ``async`` pipe for such cases, it also demands on ``zone.js``. 

What we need is a tuned ``async`` pipe. A prototypical (!) one comes from [Fabian Wiles](https://github.com/Toxicable) who is an active community member. He calls it [push pipe](https://raw.githubusercontent.com/Toxicable/angular/798ce0b5288c7a8b522d1ca710a4f64e427e931c/packages/common/src/pipes/push_pipe.ts).

To use it, we need to introduce an Observable. In my simple, I put it directly into the component. In an more advanced case, it should be provided by a service instead. To be able to directly notify it, I'm using a ``BehaviorSubject`` too:

```typescript
@Component({
   [...]
})
export class ExternalDashboardTileComponent implements OnInit {

  @Input() a: number;
  @Input() b: number;
  @Input() c: number;

  private statsSubject = new BehaviorSubject<Stats>(null);
  public stats$ = this.statsSubject.asObservable();

  [...]
}
```

To get along with just one Observable for all three values, I group them with a class ``Stats``:

```typescript
class Stats {
  constructor(
    readonly a: number,
    readonly b: number,
    readonly c: number
  ) { }
}
```

After Angular created the component, we have to publish the three numeric values for the first time:

```typescript
ngOnInit(): void {
  this.statsSubject.next(new Stats(this.a, this.b, this.c));
}
```

After each modification, we have to do the same:

```typescript
more(): void {
  this.a = Math.round(Math.random() * 100);
  this.b = Math.round(Math.random() * 100);
  this.c = Math.round(Math.random() * 100);

  this.statsSubject.next(new Stats(this.a, this.b, this.c));
}
```

In the template, we can subscribe to the Observable with the new ``push`` pipe. In the next listing I'm using an ngIf for this. The as clause writes the received object into the ``stats`` template variable.

```html
<div class="content" *ngIf="stats$ | push as stats">
  <div style="height:200px;"> 
    <br>
    <table class="table table-condensed">
      <tr>
        <td>A</td>
        <td>{{stats.a}}</td>
      </tr>
      <tr>
        <td>B</td>
        <td>{{stats.b}}</td>
      </tr>
      <tr>
        <td>C</td>
        <td>{{stats.c}}</td>
      </tr>
    </table>
    <button class="btn btn-default btn-sm" (click)="more()">More</button>
    
  </div>
</div>
```

Also, we can switch to OnPush now, as we are just relying on Observables and Immutables:

```typescript
@Component({
  [...],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ExternalDashboardTileComponent implements OnInit {
    [...]
}
```







