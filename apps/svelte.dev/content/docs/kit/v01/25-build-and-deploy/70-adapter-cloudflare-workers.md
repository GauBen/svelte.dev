---
title: Cloudflare Workers
---

To deploy to [Cloudflare Workers](https://workers.cloudflare.com/), use [`adapter-cloudflare-workers`](https://github.com/sveltejs/kit/tree/master/packages/adapter-cloudflare-workers).

> Unless you have a specific reason to use `adapter-cloudflare-workers`, it's recommended that you use `adapter-cloudflare` instead. Both adapters have equivalent functionality, but Cloudflare Pages offers features like GitHub integration with automatic builds and deploys, preview deployments, instant rollback and so on.

## Usage

Install with `npm i -D @sveltejs/adapter-cloudflare-workers`, then add the adapter to your `svelte.config.js`:

```js
// @errors: 2307
/// file: svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare-workers';

export default {
	kit: {
		adapter: adapter()
	}
};
```

## Basic Configuration

This adapter expects to find a [wrangler.toml](https://developers.cloudflare.com/workers/platform/sites/configuration) file in the project root. It should look something like this:

```toml
/// file: wrangler.toml
name = "<your-service-name>"
account_id = "<your-account-id>"

main = "./.cloudflare/worker.js"
site.bucket = "./.cloudflare/public"

build.command = "npm run build"

compatibility_date = "2021-11-12"
workers_dev = true
```

`<your-service-name>` can be anything. `<your-account-id>` can be found by logging into your [Cloudflare dashboard](https://dash.cloudflare.com) and grabbing it from the end of the URL:

```
https://dash.cloudflare.com/<your-account-id>
```

> You should add the `.cloudflare` directory (or whichever directories you specified for `main` and `site.bucket`) to your `.gitignore`.

You will need to install [wrangler](https://developers.cloudflare.com/workers/wrangler/get-started/) and log in, if you haven't already:

```
npm i -g wrangler
wrangler login
```

Then, you can build your app and deploy it:

```sh
wrangler deploy
```

## Custom config

If you would like to use a config file other than `wrangler.toml`, you can do like so:

```js
// @errors: 2307
/// file: svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare-workers';

export default {
	kit: {
		adapter: adapter({ config: '<your-wrangler-name>.toml' })
	}
};
```

## Bindings

The [`env`](https://developers.cloudflare.com/workers/runtime-apis/fetch-event#parameters) object contains your project's [bindings](https://developers.cloudflare.com/workers/platform/environment-variables/), which consist of KV/DO namespaces, etc. It is passed to SvelteKit via the `platform` property, along with `context` and `caches`, meaning that you can access it in hooks and endpoints:

```js
// @errors: 7031
export async function POST({ request, platform }) {
	const x = platform.env.YOUR_DURABLE_OBJECT_NAMESPACE.idFromName('x');
}
```

> SvelteKit's built-in `$env` module should be preferred for environment variables.

To make these types available to your app, reference them in your `src/app.d.ts`:

```diff
/// file: src/app.d.ts
declare global {
	namespace App {
		interface Platform {
+			env?: {
+				YOUR_KV_NAMESPACE: KVNamespace;
+				YOUR_DURABLE_OBJECT_NAMESPACE: DurableObjectNamespace;
+			};
		}
	}
}

export {};
```

### Testing Locally

`platform.env` is only available in the final build and not in dev mode. For testing the build, you can use [wrangler](https://developers.cloudflare.com/workers/cli-wrangler). Once you have built your site, run `wrangler dev`. Ensure you have your [bindings](https://developers.cloudflare.com/workers/wrangler/configuration/#bindings) in your `wrangler.toml`. Wrangler version 3 is recommended.

## Troubleshooting

### Worker size limits

When deploying to workers, the server generated by SvelteKit is bundled into a single file. Wrangler will fail to publish your worker if it exceeds [the size limits](https://developers.cloudflare.com/workers/platform/limits/#worker-size) after minification. You're unlikely to hit this limit usually, but some large libraries can cause this to happen. In that case, you can try to reduce the size of your worker by only importing such libraries on the client side. See [the FAQ](./faq#how-do-i-use-x-with-sveltekit-how-do-i-use-a-client-side-only-library-that-depends-on-document-or-window) for more information.

### Accessing the file system

You can't access the file system through methods like `fs.readFileSync` in Serverless/Edge environments. If you need to access files that way, do that during building the app through [prerendering](https://kit.svelte.dev/docs/page-options#prerender). If you have a blog for example and don't want to manage your content through a CMS, then you need to prerender the content (or prerender the endpoint from which you get it) and redeploy your blog everytime you add new content.