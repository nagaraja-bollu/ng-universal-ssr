## Configure [Angular universal](https://universal.angular.io/) to angular-cli project for server side rendering

### prerequisites:
- Angular-cli template  v1.7
-  @angular/animations
-  @nguniversal/express-engine
-  @nguniversal/module-map-ngfactory-loader
-  ts-loader
-  @angular/platform-server

### Create Angular application using angular/cli

Run below commands for create a angular template,, skip this step if you have the application ready.
```
$ ng g new project-name
$ cd project-name
```
Now Angular application is ready to use..

### Configure angular universal to your appliation
```
ng generate universal app-name
```
After running the above command, below files will be add to your project
```
 create src/app/app.server.module.ts (318 bytes)
  create src/main.server.ts (220 bytes)
  create src/tsconfig.server.json (307 bytes)
  update package.json (1338 bytes)
  update .angular-cli.json (1888 bytes)
  update src/main.ts (430 bytes)
  update src/app/app.module.ts (361 bytes)
  update .gitignore (556 bytes)
```
#### Install the new packages updated in package.json
```
npm install
```

### HttpInterceptor
Angular server-side render through @angular/platform-server requires absolute URLs in HTTP request. Absolute paths are automatically prepend the URL using an HttpClient interceptor.

#### Create a file in src/app/ universal.interceptor.ts

```
import { Injectable, Inject, Optional } from '@angular/core';
import { HttpInterceptor, HttpHandler, HttpRequest } from '@angular/common/http';

@Injectable()
export class UniversalInterceptor implements HttpInterceptor {

  constructor(@Optional() @Inject('serverUrl') protected serverUrl: string) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {

    const serverReq = !this.serverUrl ? req : req.clone({
      url: `${this.serverUrl}${req.url}`
    });

    return next.handle(serverReq);

  }

}

```

Then provide this in AppServerModule.

```
import { NgModule } from '@angular/core';
import { ServerModule } from '@angular/platform-server';
import { HTTP_INTERCEPTORS } from '@angular/common/http';

import { AppModule } from './app.module';
import { AppComponent } from './app.component';
import { UniversalInterceptor } from './universal.interceptor';

@NgModule({
  imports: [
    AppModule,
    ServerModule
  ],
  providers: [{
    provide: HTTP_INTERCEPTORS,
    useClass: UniversalInterceptor,
    multi: true
  }],
  bootstrap: [AppComponent]
})
export class AppServerModule {}

```

Install the unversal packages
```
npm install @nguniversal/express-engine --save
npm install @nguniversal/module-map-ngfactory-loader --save
npm install ts-loader --save
```

### Create Server.Ts file under root folder for rendering the client modules on npm server

```
// These are important and needed before anything else
import 'zone.js/dist/zone-node';
import 'reflect-metadata';

import { enableProdMode } from '@angular/core';

import * as express from 'express';
import { join } from 'path';

import * as yargs from 'yargs';
import { environment } from './src/environments/environment';
// Faster server renders w/ Prod mode (dev mode never needed)


// Express server
const app = express();
// debugger;
// const argv = yargs.argv;
// console.log(argv.ship);
console.log(environment.production);
if(environment.production){
  enableProdMode();
  console.log(environment.production);
}


const PORT = process.env.PORT || 4200;
const DIST_FOLDER = join(process.cwd(), '');

// * NOTE :: leave this as require() since this file is built Dynamically from webpack
const { AppServerModuleNgFactory, LAZY_MODULE_MAP } = require('./dist/dist-server/main.bundle');

// Express Engine
import { ngExpressEngine } from '@nguniversal/express-engine';
// Import module map for lazy loading
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';

app.engine('html', ngExpressEngine({
  bootstrap: AppServerModuleNgFactory,
  providers: [
    provideModuleMap(LAZY_MODULE_MAP)
  ]
}));

app.set('view engine', 'html');
app.set('views', DIST_FOLDER);

// TODO: implement data requests securely
app.get('/health-check/*', (req, res) => {
  res.send('working!');
});

// Server static files from /browser
app.get('*.*', express.static(DIST_FOLDER));

// All regular routes use the Universal engine
app.get('*', (req, res) => {
  res.render(join(DIST_FOLDER, 'index.html'), { req });
});

// Start up the Node server
app.listen(PORT, () => {
  console.log(`Node server listening on http://localhost:${PORT}`);
});

```

### Create webpack.server.config.js file under the root folder for bundling the server.ts file
```
const path = require('path');
const webpack = require('webpack');

module.exports = {
  entry: { server: './server.ts' },
  resolve: { extensions: ['.js', '.ts'] },
  target: 'node',
  // this makes sure we include node_modules and other 3rd party libraries
  externals: [/(node_modules|main\..*\.js)/],
  output: {
    path: path.join(__dirname, 'dist'),
    filename: '[name].js'
  },
  module: {
    rules: [{ test: /\.ts$/, loader: 'ts-loader' }]
  },
  plugins: [
    // Temporary Fix for issue: https://github.com/angular/angular/issues/11580
    // for 'WARNING Critical dependency: the request of a dependency is an expression'
    new webpack.ContextReplacementPlugin(
      /(.+)?angular(\\|\/)core(.+)?/,
      path.join(__dirname, 'src'), // location of your src
      {} // a map of your routes
    ),
    new webpack.ContextReplacementPlugin(
      /(.+)?express(\\|\/)(.+)?/,
      path.join(__dirname, 'src'),
      {}
    )
  ]
};

```


### update the package.json scripts.

```
 "scripts": {
    "ng": "ng",
    "start": "npm run build && npm run server",
    "build": "ng build --prod  && ng build --prod --app 1 --output-hashing=none && npm run webpack:server",
    "test": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e",
    "server": "cd dist && node server.js",
    "webpack:server": "webpack --config webpack.server.config.js --progress --colors"
  }
```

Run the application and check the view source of the webpage contains the rendered html.
```
npm run start
```
Now all configurations for server side rendering is done and happy coding......

Angular universal is more suitable for static webapges, renders on server side using node server.
Some packages like Jquery, D3.js etc will not rendered on the server side. Currently, Universal team is working on it.

Upcoming features on Angular universal..:
https://github.com/angular/universal#in-progress

Reference blogs:
https://angular.io/guide/universal
https://github.com/cyrilletuzi/nguniversal-expressengine-httpinterceptor
