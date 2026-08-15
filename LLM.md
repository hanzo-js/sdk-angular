# hanzoai-angular

**This repo holds no decisions. `src/` is generated and `.spec-lock` says from
what.**

One of three TypeScript projections of hanzoai/cloud's `openapi.yaml`:

| package | repo | generator | for |
|---|---|---|---|
| `hanzoai` | hanzoai/js-sdk | typescript-axios | **the default** — Node, bundled browser apps |
| `hanzoai-fetch` | hanzo-js/sdk-fetch | typescript-fetch | hosts where a dependency tree is the cost |
| `hanzoai-angular` | hanzo-js/sdk-angular | typescript-angular | Angular DI, `HttpClient`, `Observable` |

All three are regenerated from ONE document at ONE digest by a cloud release,
so they never describe different releases.

## The rule that put this repo here

A variant earns a package when a **runtime or a framework cannot use the default
projection**. Angular is the strongest case there is: it owns the transport and
the return type. Interceptors, `HttpContext`, SSR transfer-cache, cancellable
Observables and `HttpTestingController` all follow from calls going through
`HttpClient`, and an axios client is a second HTTP stack none of them can see.
Wrapping it yields promises inside a framework whose surface composes
Observables.

192 services, one `@Injectable({providedIn: 'root'})` each — so a service nobody
injects is unreachable and the bundler drops it. Both entry shapes ship:
`provideApi()` for a standalone application, `ApiModule.forRoot()` for an
NgModule one.

## Editing

Nothing under `src/`. To change what the client says, change the route in
hanzoai/cloud; to change how it is projected, change the **`angular` row** of
`hanzoai/openapi`'s `sdks.yaml` and regenerate. `.generated` records every file
this driver wrote, and `--check` is what makes "nobody edited it" a fact.

```bash
OPENAPI=../../hanzo/openapi ./scripts/generate.sh --check   # clean, or the diff
```

Repo-owned (the generator never writes these): `package.json`, `tsconfig.json`,
`ng-package.json`, `.spec-lock`, `hanzo.yml`, `scripts/`, `README.md`,
`LICENSE`.

## Three things measured here, worth not re-learning

**`serviceSuffix: Api` is forced, not preferred.** The API has a product called
`base`, so under the generator's `Service` default its class is `BaseService` —
the name of the generated transport's own parent in `api.base.service.ts` —
and `base.service.ts` compiles to `export class BaseService extends BaseService`
(TS2440, TS2395, TS2506, five TS2339s). Generator 7.14.0 has
`--model-name-mappings` and `--name-mappings` but **no per-tag api mapping**, so
this global suffix is the only lever that reaches an API class name. It also
removes a second collision for free — the schema `o11y.Service` Pascal-cases to
`O11yService`, which was the o11y tag's own class, TS2308 in `src/index.ts` —
and it puts all three TypeScript clients on one class name per product
(`KeysApi` everywhere).

**`ng-package.json` is the repo's, not the generator's.** The row takes the
whole output into `src/`, so the emitted manifest would land at
`src/ng-package.json`; ng-packagr resolves `dest` relative to the manifest, so
the build would write into the tree it compiles. It is on `hanzoai/openapi`'s
`drop` list and the repo's own sits at the root, naming `src/index.ts`.

**`ngVersion` is deliberately unset.** Measured: generated at the generator's
default 19.0.0 and again at 22.0.0, exactly one of 2671 files differs and it is
`package.json`, which is dropped. So the value would reach nothing this repo
keeps; the peer range is a fact about the published package and lives in the
`package.json` the repo owns.

**`o11y.GettableAgentCheckIn`** collides the same way it does in every other
language — `integration_config`/`integrationConfig` and `removed_at`/`removedAt`
are both live wire, both camel-case to one field. The row renames the identifier
and never the wire name.

## Building and publishing

`npm run build` is `ng-packagr`, which is also the gate: it runs the Angular
compiler over all 192 services and 2460 models in **partial** compilation mode
(full compilation would pin the output to one Angular, and ngcc — which used to
bridge that — was removed in Angular 16), then bundles to FESM2022.

**The publishable package is `dist/`**, which ng-packagr writes with `scripts`
and `devDependencies` stripped. `npm publish` runs from there, not from the repo
root.
