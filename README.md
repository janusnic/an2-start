# an2-start

A super-simple Angular 2 application in TypeScript unit 07

Getting and Saving Data with HTTP

Our stakeholders appreciate our progress. Now they want to get the hero data from a server, let users add, edit, and delete heroes, and save these changes back to the server.

In this chapter we teach our application to make the corresponding http calls to a remote server's web api.

Run the live example.

Where We Left Off

In the previous chapter, we learned to navigate between the dashboard and the fixed heroes list, editing a selected hero along the way. That's our starting point for this chapter.

Keep the app transpiling and running

Open a terminal/console window and enter the following command to start the TypeScript compiler, start the server, and watch for changes:


npm start
The application runs and updates automatically as we continue to build the Tour of Heroes.

Prepare for Http

Http is not a core Angular module. It's Angular's optional approach to web access and it exists as a separate add-on module called @angular/http, shipped in a separate script file as part of the Angular npm package.

Fortunately we're ready to import from @angular/http because systemjs.config configured SystemJS to load that library when we need it.

Register (provide) http services

Our app will depend upon the Angular http service which itself depends upon other supporting services. The HTTP_PROVIDERS array from @angular/http library holds providers for the complete set of http services.

We should be able to access these services from anywhere in the application. So we register them in the bootstrap method of main.ts where we launch the application and its root AppComponent.

app/main.ts (v1)

import { bootstrap }      from '@angular/platform-browser-dynamic';
import { HTTP_PROVIDERS } from '@angular/http';

import { AppComponent }   from './app.component';

bootstrap(AppComponent, [ HTTP_PROVIDERS ]);
Notice that we supply the HTTP_PROVIDERS in an array as the second parameter to the bootstrap method. This has the same effect the providers array in @Component metadata.

Simulating the web api

We generally recommend registering application-wide services in the root AppComponent providers. Here we're registering in main for a special reason.

Our application is in the early stages of development and far from ready for production. We don't even have a web server that can handle requests for heroes. Until we do, we'll have to fake it.

We're going to trick the http client into fetching and saving data from a demo/development service, the in-memory web api.

The application itself doesn't need to know and shouldn't know about this. So we'll slip the in-memory web api into the configuration above the AppComponent.

Here is a version of main that performs this trick

app/main.ts (final)

// Imports for loading & configuring the in-memory web api
import { provide }    from '@angular/core';
import { XHRBackend } from '@angular/http';

import { InMemoryBackendService, SEED_DATA } from 'angular2-in-memory-web-api';
import { InMemoryDataService }               from './in-memory-data.service';

// The usual bootstrapping imports
import { bootstrap }      from '@angular/platform-browser-dynamic';
import { HTTP_PROVIDERS } from '@angular/http';

import { AppComponent }   from './app.component';

bootstrap(AppComponent, [
    HTTP_PROVIDERS,
    provide(XHRBackend, { useClass: InMemoryBackendService }), // in-mem server
    provide(SEED_DATA,  { useClass: InMemoryDataService })     // in-mem server data
]);
We're replacing the default XHRBackend, the service that talks to the remote server, with the in-memory web api service after priming it with the following in-memory-data.service.ts file:

app/in-memory-data.service.ts

export class InMemoryDataService {
  createDb() {
    let heroes = [
      {id: 11, name: 'Mr. Nice'},
      {id: 12, name: 'Narco'},
      {id: 13, name: 'Bombasto'},
      {id: 14, name: 'Celeritas'},
      {id: 15, name: 'Magneta'},
      {id: 16, name: 'RubberMan'},
      {id: 17, name: 'Dynama'},
      {id: 18, name: 'Dr IQ'},
      {id: 19, name: 'Magma'},
      {id: 20, name: 'Tornado'}
    ];
    return {heroes};
  }
}
This file replaces the mock-heroes.ts which is now safe to delete.

This chaper is an introduction to the Angular http client. Please don't be distracted by the details of this backend substitution. Just follow along with the example.

Learn more later about the in-memory web api in the Http chapter. Remember, the in-memory web api is only useful in the early stages of development and for demonstrations such as this Tour of Heroes. Skip it when you have a real web api server.

Heroes and Http

Look at our current HeroService implementation

app/hero.service.ts (getHeroes - old)

getHeroes() {
  return Promise.resolve(HEROES);
}
We returned a promise resolved with mock heroes. It may have seemed like overkill at the time, but we were anticipating the day when we fetched heroes with an http client and we knew that would have to be an asynchronous operation.

That day has arrived! Let's convert getHeroes() to use Angular's Http client:

app/hero.service.ts (getHeroes using Http)

  getHeroes(): Promise<Hero[]> {
    return this.http.get(this.heroesUrl)
               .toPromise()
               .then(response => response.json().data)
               .catch(this.handleError);
  }
Http Promise

We're still returning a promise but we're creating it differently.

The Angular http.get returns an RxJS Observable. Observables are a powerful way to manage asynchronous data flows. We'll learn about Observables later.

For now we get back on familiar ground by immediately converting that Observable to a Promise using the toPromise operator.


.toPromise()
Unfortunately, the Angular Observable doesn't have a toPromise operator ... not out of the box. The Angular Observable is a bare-bones implementation.

There are scores of operators like toPromise that extend Observable with useful capabilities. If we want those capabilities, we have to add the operators ourselves. That's as easy as importing them from the RxJS library like this:


import 'rxjs/add/operator/toPromise';
Extracting the data in the then callback

In the promise's then callback we call the json method of the http Response to extract the data within the response.


.then(response => response.json().data)
That object returned by json has a single data property. The data property holds the array of heroes that the caller really wants. So we grab that array and return it as the resolved promise value.

Pay close attention to the shape of the data returned by the server. This particular in-memory web api example happens to return an object with a data property. Your api might return something else.

Adjust the code to match your web api.

The caller is unaware of these machinations. It receives a promise of heroes just as it did before. It has no idea that we fetched the heroes from the server. It knows nothing of the twists and turns required to turn the http response into heroes. Such is the beauty and purpose of delegating data access to a service like this HeroService.

Error Handling

At the end of getHeroes we catch server failures and pass them to an error handler:


.catch(this.handleError);
This is a critical step! We must anticipate http failures as they happen frequently for reasons beyond our control.

app/hero.service.ts (Error handler)

private handleError(error: any) {
  console.error('An error occurred', error);
  return Promise.reject(error.message || error);
}
In this demo service we log the error to the console; we should do better in real life.

We've also decided to return a user friendly form of the error to to the caller in a rejected promise so that the caller can display a proper error message to the user.

Promises are Promises

Although we made significant internal changes to getHeroes(), the public signature did not change. We still return a promise. We won't have to update any of the components that call getHeroes().

Add, Edit, Delete

Our stakeholders are incredibly pleased with the added flexibility from the api integration, but it doesn't stop there. Next we want to add the capability to add, edit and delete heroes.

We'll complete HeroService by creating post, put and delete http calls to meet our new requirements.

Post

We are using post to add new heroes. Post requests require a little bit more setup than Get requests, but the format is as follows:

app/hero.service.ts (post hero)

// Add new Hero
private post(hero: Hero): Promise<Hero> {
  let headers = new Headers({
    'Content-Type': 'application/json'});

  return this.http
             .post(this.heroesUrl, JSON.stringify(hero), {headers: headers})
             .toPromise()
             .then(res => res.json().data)
             .catch(this.handleError);
}
Now we create a header and set the content type to application/json. We'll call JSON.stringify before we post to convert the hero object to a string.

Put

put is used to edit a specific hero, but the structure is very similar to a post request. The only difference is that we have to change the url slightly by appending the id of the hero we want to edit.

app/hero.service.ts (put hero)

// Update existing Hero
private put(hero: Hero) {
  let headers = new Headers();
  headers.append('Content-Type', 'application/json');

  let url = `${this.heroesUrl}/${hero.id}`;

  return this.http
             .put(url, JSON.stringify(hero), {headers: headers})
             .toPromise()
             .then(() => hero)
             .catch(this.handleError);
}
Delete

delete is used to delete heroes and the format is identical to put except for the function name.

app/hero.service.ts (delete hero)

delete(hero: Hero) {
  let headers = new Headers();
  headers.append('Content-Type', 'application/json');

  let url = `${this.heroesUrl}/${hero.id}`;

  return this.http
             .delete(url, headers)
             .toPromise()
             .catch(this.handleError);
}
We add a catch to handle our errors for all three cases.

Save

We combine the call to the private post and put methods in a single save method. This simplifies the public api and makes the integration with HeroDetailComponent easier. HeroService determines which method to call based on the state of the hero object. If the hero already has an id we know it's an edit. Otherwise we know it's an add.

app/hero.service.ts (save hero)

save(hero: Hero): Promise<Hero>  {
  if (hero.id) {
    return this.put(hero);
  }
  return this.post(hero);
}
After these additions our HeroService looks like this:

app/hero.service.ts

import { Injectable }    from '@angular/core';
import { Headers, Http } from '@angular/http';

import 'rxjs/add/operator/toPromise';

import { Hero } from './hero';

@Injectable()
export class HeroService {

  private heroesUrl = 'app/heroes';  // URL to web api

  constructor(private http: Http) { }

  getHeroes(): Promise<Hero[]> {
    return this.http.get(this.heroesUrl)
               .toPromise()
               .then(response => response.json().data)
               .catch(this.handleError);
  }

  getHero(id: number) {
    return this.getHeroes()
               .then(heroes => heroes.filter(hero => hero.id === id)[0]);
  }

  save(hero: Hero): Promise<Hero>  {
    if (hero.id) {
      return this.put(hero);
    }
    return this.post(hero);
  }

  delete(hero: Hero) {
    let headers = new Headers();
    headers.append('Content-Type', 'application/json');

    let url = `${this.heroesUrl}/${hero.id}`;

    return this.http
               .delete(url, headers)
               .toPromise()
               .catch(this.handleError);
  }

  // Add new Hero
  private post(hero: Hero): Promise<Hero> {
    let headers = new Headers({
      'Content-Type': 'application/json'});

    return this.http
               .post(this.heroesUrl, JSON.stringify(hero), {headers: headers})
               .toPromise()
               .then(res => res.json().data)
               .catch(this.handleError);
  }

  // Update existing Hero
  private put(hero: Hero) {
    let headers = new Headers();
    headers.append('Content-Type', 'application/json');

    let url = `${this.heroesUrl}/${hero.id}`;

    return this.http
               .put(url, JSON.stringify(hero), {headers: headers})
               .toPromise()
               .then(() => hero)
               .catch(this.handleError);
  }

  private handleError(error: any) {
    console.error('An error occurred', error);
    return Promise.reject(error.message || error);
  }
}
Updating Components

Loading heroes using Http required no changes outside of HeroService, but we added a few new features as well. In the following section we will update our components to use our new methods to add, edit and delete heroes.

Add/Edit in the HeroDetailComponent

We already have HeroDetailComponent for viewing details about a specific hero. Add and Edit are natural extensions of the detail view, so we are able to reuse HeroDetailComponent with a few tweaks. The original component was created to render existing data, but to add new data we have to initialize the hero property to an empty Hero object.

app/hero-detail.component.ts (ngOnInit)

ngOnInit() {
  if (this.routeParams.get('id') !== null) {
    let id = +this.routeParams.get('id');
    this.navigated = true;
    this.heroService.getHero(id)
        .then(hero => this.hero = hero);
  } else {
    this.navigated = false;
    this.hero = new Hero();
  }
}
In order to differentiate between add and edit we are adding a check to see if an id is passed in the url. If the id is absent we bind HeroDetailComponent to an empty Hero object. In either case, any edits made through the UI will be bound back to the same hero property.

The next step is to add a save method to HeroDetailComponent and call the corresponding save method in HeroesService.

app/hero-detail.component.ts (save)

save() {
  this.heroService
      .save(this.hero)
      .then(hero => {
        this.hero = hero; // saved hero, w/ id if new
        this.goBack(hero);
      })
      .catch(error => this.error = error); // TODO: Display error message
}
The same save method is used for both add and edit since HeroService will know when to call post vs put based on the state of the Hero object.

After we save a hero, we redirect the browser back to the to the previous page using the goBack() method.

app/hero-detail.component.ts (goBack)

goBack(savedHero: Hero = null) {
  this.close.emit(savedHero);
  if (this.navigated) { window.history.back(); }
}
Here we call emit to notify that we just added or modified a hero. HeroesComponent is listening for this notification and will automatically refresh the list of heroes to include our recent updates.

The emit "handshake" between HeroDetailComponent and HeroesComponent is an example of component to component communication. This is a topic for another day, but we have detailed information in our Component Interaction Cookbook

Here is HeroDetailComponent with its new save button.

Hero Details With Save Button
Add/Delete in the HeroesComponent

The user can add a new hero by clicking a button and entering a name.

When the user clicks the Add New Hero button, we display the HeroDetailComponent. We aren't navigating to the component so it won't receive a hero id; As we noted above, that is the component's cue to create and present an empty hero.

Add the following HTML to the heroes.component.html, just below the hero list (the *ngFor).

app/heroes.component.html (add)

<button (click)="addHero()">Add New Hero</button>
<div *ngIf="addingHero">
  <my-hero-detail (close)="close($event)"></my-hero-detail>
</div>
The user can delete an existing hero by clicking a delete button next to the hero's name.

Add the following HTML to the heroes.component.html right after the name in the repeated <li> tag:

app/heroes.component.html (delete)

<button class="delete-button" (click)="delete(hero, $event)">Delete</button>
Now let's fix-up the HeroesComponent to support the add and delete actions in the template. Let's start with add.

We're using the HeroDetailComponent to capture the new hero information. We have to tell Angular about that by importing the HeroDetailComponent and referencing it in the component metadata directives array.

app/heroes.component.ts (HeroDetailComponent)

import { HeroDetailComponent } from './hero-detail.component';

@Component({
  selector: 'my-heroes',
  templateUrl: 'app/heroes.component.html',
  styleUrls:  ['app/heroes.component.css'],
  directives: [HeroDetailComponent]
})
These are the same lines that we removed in the previous Routing chapter. We didn't know at the time that we'd need the HeroDetailComponent again. So we tidied up.

Now we must put these lines back. If we don't, Angular will ignore the <my-hero-detail> tag and pushing the Add New Hero button will have no visible effect.

Next we implement the click handler for the Add New Hero button.

app/heroes.component.ts (add)

addHero() {
  this.addingHero = true;
  this.selectedHero = null;
}

close(savedHero: Hero) {
  this.addingHero = false;
  if (savedHero) { this.getHeroes(); }
}
The HeroDetailComponent does most of the work. All we do is toggle an *ngIf flag that swaps it into the DOM when were add a hero and remove it from the DOM when the user is done.

The delete logic is a bit trickier.

app/heroes.component.ts (delete)

delete(hero: Hero, event: any) {
  event.stopPropagation();
  this.heroService
      .delete(hero)
      .then(res => {
        this.heroes = this.heroes.filter(h => h !== hero);
        if (this.selectedHero === hero) { this.selectedHero = null; }
      })
      .catch(error => this.error = error); // TODO: Display error message
}
Of course we delegate the persistence of hero deletion to the HeroService. But the component is still responsible for updating the display. So the delete method removes the deleted hero from the list.

Let's see it

Here are the fruits of labor in action:

Heroes List Editting w/ HTTP
Review the App Structure

Let’s verify that we have the following structure after all of our good refactoring in this chapter:

angular2-tour-of-heroes
app
app.component.ts
app.component.css
dashboard.component.css
dashboard.component.html
dashboard.component.ts
hero.ts
hero-detail.component.css
hero-detail.component.html
hero-detail.component.ts
hero.service.ts
heroes.component.css
heroes.component.html
heroes.component.ts
main.ts
hero-data.service.ts
node_modules ...
typings ...
index.html
package.json
styles.css
sample.css
systemjs.config.json
tsconfig.json
typings.json
Home Stretch

We are at the end of our journey for now, but we have accomplished a lot.

We added the necessary dependencies to use Http in our application.
We refactored HeroService to load heroes from an api.
We extended HeroService to support post, put and delete calls.
We updated our components to allow adding, editing and deleting of heroes.
We configured an in-memory web api.
Below is a summary of the files we changed.

app.comp...ts 

import { Component } from '@angular/core';
import { RouteConfig, ROUTER_DIRECTIVES, ROUTER_PROVIDERS } from '@angular/router-deprecated';
import { DashboardComponent }  from './dashboard.component';
import { HeroesComponent }     from './heroes.component';
import { HeroDetailComponent } from './hero-detail.component';
import { HeroService }         from './hero.service';
@Component({
  selector: 'my-app',
  template: `
    <h1>{{title}}</h1>
    <nav>
      <a [routerLink]="['Dashboard']">Dashboard</a>
      <a [routerLink]="['Heroes']">Heroes</a>
    </nav>
    <router-outlet></router-outlet>
  `,
  styleUrls: ['app/app.component.css'],
  directives: [ROUTER_DIRECTIVES],
  providers: [
    ROUTER_PROVIDERS,
    HeroService,
  ]
})
@RouteConfig([
  { path: '/dashboard',  name: 'Dashboard',  component: DashboardComponent, useAsDefault: true },
  { path: '/detail/:id', name: 'HeroDetail', component: HeroDetailComponent },
  { path: '/heroes',     name: 'Heroes',     component: HeroesComponent }
])
export class AppComponent {
  title = 'Tour of Heroes';
}

heroes.comp...ts 

import { Component, OnInit } from '@angular/core';
import { Router }            from '@angular/router-deprecated';
import { Hero }                from './hero';
import { HeroService }         from './hero.service';
import { HeroDetailComponent } from './hero-detail.component';
@Component({
  selector: 'my-heroes',
  templateUrl: 'app/heroes.component.html',
  styleUrls:  ['app/heroes.component.css'],
  directives: [HeroDetailComponent]
})
export class HeroesComponent implements OnInit {
  heroes: Hero[];
  selectedHero: Hero;
  addingHero = false;
  error: any;
  constructor(
    private router: Router,
    private heroService: HeroService) { }
  getHeroes() {
    this.heroService
        .getHeroes()
        .then(heroes => this.heroes = heroes)
        .catch(error => this.error = error); // TODO: Display error message
  }
  addHero() {
    this.addingHero = true;
    this.selectedHero = null;
  }
  close(savedHero: Hero) {
    this.addingHero = false;
    if (savedHero) { this.getHeroes(); }
  }
  delete(hero: Hero, event: any) {
    event.stopPropagation();
    this.heroService
        .delete(hero)
        .then(res => {
          this.heroes = this.heroes.filter(h => h !== hero);
          if (this.selectedHero === hero) { this.selectedHero = null; }
        })
        .catch(error => this.error = error); // TODO: Display error message
  }
  ngOnInit() {
    this.getHeroes();
  }
  onSelect(hero: Hero) {
    this.selectedHero = hero;
    this.addingHero = false;
  }
  gotoDetail() {
    this.router.navigate(['HeroDetail', { id: this.selectedHero.id }]);
  }
}

heroes.comp...html 
<h2>My Heroes</h2>
<ul class="heroes">
  <li *ngFor="let hero of heroes" (click)="onSelect(hero)" [class.selected]="hero === selectedHero">
    <span class="hero-element">
      <span class="badge">{{hero.id}}</span> {{hero.name}}
    </span>
    <button class="delete-button" (click)="delete(hero, $event)">Delete</button>
  </li>
</ul>
<button (click)="addHero()">Add New Hero</button>
<div *ngIf="addingHero">
  <my-hero-detail (close)="close($event)"></my-hero-detail>
</div>
<div *ngIf="selectedHero">
  <h2>
    {{selectedHero.name | uppercase}} is my hero
  </h2>
  <button (click)="gotoDetail()">View Details</button>
</div>


hero-detail.comp...ts 
import { Component, EventEmitter, Input, OnInit, Output } from '@angular/core';
import { RouteParams } from '@angular/router-deprecated';
import { Hero }        from './hero';
import { HeroService } from './hero.service';
@Component({
  selector: 'my-hero-detail',
  templateUrl: 'app/hero-detail.component.html',
  styleUrls: ['app/hero-detail.component.css']
})
export class HeroDetailComponent implements OnInit {
  @Input() hero: Hero;
  @Output() close = new EventEmitter();
  error: any;
  navigated = false; // true if navigated here
  constructor(
    private heroService: HeroService,
    private routeParams: RouteParams) {
  }
  ngOnInit() {
    if (this.routeParams.get('id') !== null) {
      let id = +this.routeParams.get('id');
      this.navigated = true;
      this.heroService.getHero(id)
          .then(hero => this.hero = hero);
    } else {
      this.navigated = false;
      this.hero = new Hero();
    }
  }
  save() {
    this.heroService
        .save(this.hero)
        .then(hero => {
          this.hero = hero; // saved hero, w/ id if new
          this.goBack(hero);
        })
        .catch(error => this.error = error); // TODO: Display error message
  }
  goBack(savedHero: Hero = null) {
    this.close.emit(savedHero);
    if (this.navigated) { window.history.back(); }
  }
}


hero-detail.comp...html 

<div *ngIf="hero">
  <h2>{{hero.name}} details!</h2>
  <div>
    <label>id: </label>{{hero.id}}</div>
  <div>
    <label>name: </label>
    <input [(ngModel)]="hero.name" placeholder="name" />
   </div>
  <button (click)="goBack()">Back</button>
  <button (click)="save()">Save</button>
</div>


hero.service.ts

import { Injectable }    from '@angular/core';
import { Headers, Http } from '@angular/http';
import 'rxjs/add/operator/toPromise';
import { Hero } from './hero';
@Injectable()
export class HeroService {
  private heroesUrl = 'app/heroes';  // URL to web api
  constructor(private http: Http) { }
  getHeroes(): Promise<Hero[]> {
    return this.http.get(this.heroesUrl)
               .toPromise()
               .then(response => response.json().data)
               .catch(this.handleError);
  }
  getHero(id: number) {
    return this.getHeroes()
               .then(heroes => heroes.filter(hero => hero.id === id)[0]);
  }
  save(hero: Hero): Promise<Hero>  {
    if (hero.id) {
      return this.put(hero);
    }
    return this.post(hero);
  }
  delete(hero: Hero) {
    let headers = new Headers();
    headers.append('Content-Type', 'application/json');
    let url = `${this.heroesUrl}/${hero.id}`;
    return this.http
               .delete(url, headers)
               .toPromise()
               .catch(this.handleError);
  }
  // Add new Hero
  private post(hero: Hero): Promise<Hero> {
    let headers = new Headers({
      'Content-Type': 'application/json'});
    return this.http
               .post(this.heroesUrl, JSON.stringify(hero), {headers: headers})
               .toPromise()
               .then(res => res.json().data)
               .catch(this.handleError);
  }
  // Update existing Hero
  private put(hero: Hero) {
    let headers = new Headers();
    headers.append('Content-Type', 'application/json');
    let url = `${this.heroesUrl}/${hero.id}`;
    return this.http
               .put(url, JSON.stringify(hero), {headers: headers})
               .toPromise()
               .then(() => hero)
               .catch(this.handleError);
  }
  private handleError(error: any) {
    console.error('An error occurred', error);
    return Promise.reject(error.message || error);
  }
}
