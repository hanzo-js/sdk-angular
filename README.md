# hanzoai-angular

The [Hanzo API](https://api.hanzo.ai) as Angular services. Same document, same
2479 operations across 192 products as
[`hanzoai`](https://www.npmjs.com/package/hanzoai) — generated from the OpenAPI
description each subsystem's router emits, so it cannot name an address the
server does not serve — delivered as 192 `@Injectable({providedIn: 'root'})`
classes over `HttpClient`, returning `Observable`.

## Which one you want

**Use `hanzoai`.** It is the default TypeScript client and it is the right one
for Node, for a bundled browser app, and for any Angular code where a promise is
what you actually wanted.

**Use `hanzoai-angular` when the framework is the reason.** Angular owns the
transport and the return type, and an axios client cannot be made to speak
either:

- **Interceptors.** Every call goes through `HttpClient`, so
  `provideHttpClient(withInterceptors([...]))` applies to the API the same way it
  applies to the rest of your app — one place for auth refresh, tracing, retry
  and error mapping. An axios client is a second HTTP stack your interceptors
  never see.
- **Observables.** Methods return `Observable`, so a call composes with
  `switchMap`, `takeUntilDestroyed`, `toSignal`, `combineLatest` and everything
  else your components already use. Wrapping a promise gives you an Observable
  that cannot be cancelled.
- **The framework's own machinery.** `HttpContext` per call, SSR transfer-cache
  (`transferCache`, on by default here), `HttpTestingController` in tests,
  `reportProgress` for uploads, `observe: 'response' | 'events'` when you need
  headers or progress rather than a body.

If you are not consuming these through Angular's DI, `hanzoai` is the better
package.

## Install

```bash
npm i hanzoai-angular
```

Peers: `@angular/core` and `@angular/common` (>= 19), `rxjs` ^7.4. Built and
published in Angular Package Format (partial-Ivy FESM2022 + types), which is
what your application's own build finishes compiling.

## Wiring

A standalone application — `provideApi` returns the environment providers:

```ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient } from '@angular/common/http';
import { provideApi } from 'hanzoai-angular';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    provideApi({
      basePath: 'https://api.hanzo.ai',
      credentials: { bearer: () => localStorage.getItem('hanzo_token') ?? undefined },
    }),
  ],
};
```

An NgModule application — the same configuration through `ApiModule.forRoot`:

```ts
import { HttpClientModule } from '@angular/common/http';
import { ApiModule, Configuration } from 'hanzoai-angular';

@NgModule({
  imports: [
    HttpClientModule,
    ApiModule.forRoot(() => new Configuration({
      basePath: 'https://api.hanzo.ai',
      credentials: { bearer: () => localStorage.getItem('hanzo_token') ?? undefined },
    })),
  ],
})
export class AppModule {}
```

## Calling

Every service is `providedIn: 'root'`, so you inject the one you need and the
bundler drops the other 191:

```ts
import { Component, inject } from '@angular/core';
import { toSignal } from '@angular/core/rxjs-interop';
import { ModelsApi } from 'hanzoai-angular';

@Component({
  selector: 'app-models',
  template: `<p>{{ models()?.length }} models</p>`,
})
export class ModelsComponent {
  private readonly api = inject(ModelsApi);
  readonly models = toSignal(this.api.getModels());
}
```

Operations that take parameters take one object:

```ts
this.keys.postKeys({ keyTypeIn: { type: 'sk' } }).subscribe(minted => ...);
```

`observe` and `reportProgress` are the third and fourth arguments, as everywhere
in Angular:

```ts
this.files.postFilesUpload(params, 'events', true).subscribe(e => ...);
```

## Auth

One scheme: a bearer token — an IAM access token or a Cloud API key. The server
derives your org from the token's `owner` claim, so no route takes an org
argument. 2498 operations across 191 of the 192 services read
`credentials.bearer` and send `Authorization: Bearer <token>`.

`credentials.bearer` takes a function, so a token that rotates is read at call
time and nothing has to be rebuilt. Four operations opt out and take no
credential: `GET /v1/models`, `GET /v1/models/providers`, `GET /v1/commands`,
`GET /v1/openapi.json`.

If your app already refreshes tokens in an `HttpInterceptor`, leave
`credentials` unset and let the interceptor set the header — this client is
`HttpClient`, so your interceptor is already in the path.

## Class names

A service is named for its product with an `Api` suffix — `KeysApi`,
`ModelsApi`, `O11yApi` — the same name the same product has in `hanzoai` and
`hanzoai-fetch`, so moving between the three clients renames nothing. The
generator's `Service` default cannot be used here: the API has a product called
`base`, whose class would then be `BaseService`, which is the name of the
generated transport's own parent class.

## Where the code comes from

`src/` is generated, and nothing in it is edited by hand. `.spec-lock` names the
hanzoai/cloud commit and the sha256 of the `openapi.yaml` this tree is a
projection of; the driver is
[`hanzoai/openapi`](https://github.com/hanzoai/openapi)'s `generate.py`, and
every knob it uses is the `angular` row of that repo's `sdks.yaml`.

```bash
OPENAPI=../../hanzo/openapi ./scripts/generate.sh            # regenerate src/
OPENAPI=../../hanzo/openapi ./scripts/generate.sh --check     # fail if src/ drifted
```

A cloud release regenerates every client in the fleet from one document at one
digest, so `hanzoai`, `hanzoai-fetch` and `hanzoai-angular` always describe the
same release.

## Build

```bash
npm ci
npm run build     # ng-packagr → dist/ (FESM2022 + types + manifest)
```

The publishable package is `dist/`, which ng-packagr writes — that is where
`npm publish` runs.

## License

Apache-2.0
