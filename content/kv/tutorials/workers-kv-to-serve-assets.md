---
updated: 2024-06-17
difficulty: Intermediate
content_type: 📝 Tutorial
pcx_content_type: tutorial
title: Store and retrieve static assets with Workers KV
---

# Store and retrieve static assets with Workers KV

This tutorial will teach you how to store and retrieve static assets using KV, and use them to generate your Workers response.

By storing static assets in Workers KV, you can retrieve these assets globally with  low-latency and high throughput. You can then serve these assets directly, or use them to dynamically generate responses. This can be useful when serving files and images, or when generating dynamic HTML responses from static assets such as translations.

{{<Aside type="note">}}

This tutorial demonstrates how to access data and assets from KV and use them to generate static and dynamic responses. If you want to serve static assets of a frontend web application, consider using [Cloudflare Pages](/pages/) which provides a purpose-built experience for web applications with built-in environments, CI/CD, Pages Functions for dynamic responses, and more.

{{</Aside>}}

{{<render file="_tutorials-before-you-start.md">}}

## 1. Create a new Worker application 

{{<render file="_tutorials-create-worker-application.md" withParameters="tutorial-kv-assets">}}

We'll also install the dependencies we will need for this project.

```sh
$ npm install mimes accept-language-parser 
$ npm install --save-dev @types/accept-language-parser
```

## 2. Create a new KV namespace

<!-- {{<render file="_tutorials-create-kv-namespace.md" withParameters="assets;;assets;;tutorial-kv-assets" >}} -->

Next, we will create a KV store. This can be done through the Cloudflare Portal or the Wrangler CLI. For this tutorial, we will use the Wrangler CLI.

To create a KV store via Wrangler:

1. Open your terminal and run the following command:

```sh
$ wrangler kv namespace create assets
```

The `wrangler kv namespace create assets` subcommand creates a KV namespace by concatenating your Worker's name and the value provided for `assets`. An `id` will be randomly generated for the KV namespace.
 
```sh
$ wrangler kv namespace create assets

🌀 Creating namespace with title "tutorial-kv-assets-assets"
✨ Success!
Add the following to your configuration file in your kv_namespaces array:
[[kv_namespaces]]
binding = "assets"
id = "<GENERATED_NAMESPACE_ID>"
```

2. In your `wrangler.toml` file, add the following with the values generated in the terminal:

```bash
---
filename: wrangler.toml
---
[[kv_namespaces]]
binding = "assets"
id = "<GENERATED_NAMESPACE_ID>"
```

The [KV binding](/kv/reference/kv-bindings) `assets` is how your Worker will interact with the [KV namespace](/kv/reference/kv-namespaces/). This binding will be provided as a runtime variable within your Workers code by the Workers runtime.

We'll also create a preview KV namespace. It is recommended to create a separate KV namespace when developing locally to avoid making changes to the production namespace. When developing locally against remote resources, the Wrangler CLI will only use the namespace specified by `preview_id` in the KV namespace configuration of the `wrangler.toml` file.

3. In your terminal, run the following command:

```sh
$ wrangler kv namespace create assets --preview
```

This command will create a special KV namespace that will be used only when developing with Wrangler against remote resources using `wrangler dev --remote`.

```sh
$ wrangler kv namespace create assets --preview

🌀 Creating namespace with title "tutorial-kv-assets-assets_preview"
✨ Success!
Add the following to your configuration file in your kv_namespaces array:
[[kv_namespaces]]
binding = "assets"
preview_id = "<GENERATED_PREVIEW_NAMESPACE_ID>"
```

4. In your `wrangler.toml` file, add the additional preview_id below kv_namespaces with the values generated in the terminal:

```bash
---
highlight: 4-4
filename: wrangler.toml
---
[[kv_namespaces]]
binding = "assets"
id = "<GENERATED_NAMESPACE_ID>"
preview_id = "<GENERATED_PREVIEW_NAMESPACE_ID>"
```

We now have one KV binding that will use the production KV namespace when deployed and the preview KV namespace when developing locally against remote resources with `wrangler dev --remote`.


## 3. Store static assets in KV using Wrangler

To store static assets in KV, you can use the Wrangler CLI, the KV binding from a Worker application, or the KV REST API. We'll demonstrate how to use the Wrangler CLI. 

For this scenario, we'll be storing a sample HTML file within our KV store. Create a new file `index.html` in the root of project with the following content:

```html
---
filename: index.html
---
Hello World!
```

We can then use the following Wrangler commands to create a KV pair for this file within our production and preview namespaces: 

```sh
$ wrangler kv key put index.html --path index.html --binding assets
$ wrangler kv key put index.html --path index.html --binding assets --preview
```

This will create a KV pair with the filename as key and the file content as value, within the our production and preview namespaces specified by your binding in your `wrangler.toml` file. 


## 4. Serve static assets from KV from your Worker application

Within the `index.ts` file of our Worker project, replace the contents with the following:

```js
---
filename: index.ts
---

import mime from 'mime';

interface Env {
	assets: KVNamespace;
}

export default {
	async fetch(request, env, ctx): Promise<Response> {
		//return error if not a get request
		if(request.method !== 'GET'){
			return new Response('Method Not Allowed', {
				status: 405,
			}) 
		}

		//get the key from the url & return error if key missing
		const parsedUrl = new URL(request.url)
		const key = parsedUrl.pathname.replace(/^\/+/, '') // strip any preceding /'s
		if(!key){
			return new Response('Missing path in URL', {
				status: 400
			})
		}

		//get the mimetype from the key path
		const extension = key.split('.').pop();
		let mimeType = mime.getType(key) || "text/plain";
		if (mimeType.startsWith("text") || mimeType === "application/javascript") {
			mimeType += "; charset=utf-8";
		}

		//get the value from the KV store and return it if found
		const value = await env.assets.get(key, 'arrayBuffer')
		if(!value){
			return new Response("Not found", {
				status: 404
			})
		}
		return new Response(value, {
			status: 200,
			headers: new Headers({
				"Content-Type": mimeType
			})
		});
	},
} satisfies ExportedHandler<Env>;
```

This code will use the path within the URL and find the file associated to the path within the KV store. It also sets the proper MIME type in the response to indicate to the browser how to handle the response. To retrieve the value from the KV store, this code uses `arrayBuffer` to properly handle binary data such as images, document files, video and audio files, etc. 

To start the Worker, run the following within a terminal:

```sh
$ wrangler dev --remote
```

This will run you Worker code against your remote resources, specifically using the preview KV namespace as configured.

```sh
$ wrangler dev --remote

Your worker has access to the following bindings:
- KV Namespaces:
  - assets: <GENERATED_PREVIEW_NAMESPACE_ID>
[wrangler:inf] Ready on http://localhost:<PORT>
```

Access the URL provided by the Wrangler command as such `http://localhost:<PORT>/index.html`. You will be able to see the returned HTML file containing the file contents of our `index.html` file that was added to our KV store. Try it out with an image or a document and you will see that this Worker is also properly serving those assets from KV.

## 5. Create an endpoint to generate dynamic responses from your key-value pairs

We'll add a `hello-world` endpoint to our Workers application, which will return a "Hello World!" message based on the language requested to demonstrate how to generate a dynamic response from our KV-stored assets.

Start by creating this file in the root of your project:

```json
---
filename: hello-world.json
---

[
    {
        "language_code": "en",
        "message": "Hello World!"
    },
    {
        "language_code": "es",
        "message": "¡Hola Mundo!"
    },
    {
        "language_code": "fr",
        "message": "Bonjour le monde!"
    },
    {
        "language_code": "de",
        "message": "Hallo Welt!"
    },
    {
        "language_code": "zh",
        "message": "你好，世界！"
    },
    {
        "language_code": "ja",
        "message": "こんにちは、世界！"
    },
    {
        "language_code": "hi",
        "message": "नमस्ते दुनिया!"
    },
    {
        "language_code": "ar",
        "message": "مرحبا بالعالم!"
    }
]
```

Open a terminal and enter the following KV command to create a KV entry for the translations file:

```sh
$ wrangler kv key put hello-world.json --path hello-world.json --binding assets
$ wrangler kv key put hello-world.json --path hello-world.json --binding assets --preview
```

Update your Workers code to add logic to serve a translated HTML file based on the language of the Accept-Language header of the request:

```js
---
filename: index.ts
highlight: 2-2, 26-63
---
import mime from 'mime';
import parser from 'accept-language-parser'

interface Env {
	assets: KVNamespace;
}

export default {
	async fetch(request, env, ctx): Promise<Response> {
		//return error if not a get request
		if(request.method !== 'GET'){
			return new Response('Method Not Allowed', {
				status: 405,
			}) 
		}

		//get the key from the url & return error if key missing
		const parsedUrl = new URL(request.url)
		const key = parsedUrl.pathname.replace(/^\/+/, '') // strip any preceding /'s
		if(!key){
			return new Response('Missing path in URL', {
				status: 400
			})
		}

		//add handler for translation path
		if(key === 'hello-world'){
			//retrieve the language header from the request and the translations from KV
			const languageHeader = request.headers.get('Accept-Language') || 'en'//default to english
			const translations : {
				"language_code": string,
				"message": string
			}[] = await env.assets.get('hello-world.json', 'json') || [];

			//extract the requested language
			const supportedLanguageCodes = translations.map(item => item.language_code)
			const languageCode = parser.pick(supportedLanguageCodes, languageHeader, {
				loose: true
			})

			//get the message for the selected language
			let selectedTranslation = translations.find(item => item.language_code === languageCode)
			if(!selectedTranslation) selectedTranslation = translations.find(item => item.language_code === "en")
			const helloWorldTranslated = selectedTranslation!['message'];
			
			//generate and return the translated html
			const html = `<!DOCTYPE html>
			<html>
				<head>
					<title>Hello World translation</title>
				</head>
				<body>
					<h1>${helloWorldTranslated}</h1>
				</body>
			</html>
			`	
			return new Response(html, {  
				status: 200,
				headers: {
					'Content-Type': 'text/html; charset=utf-8'
				}
			})
		}

		//get the mimetype from the key path
		const extension = key.split('.').pop();
		let mimeType = mime.getType(key) || "text/plain";
		if (mimeType.startsWith("text") || mimeType === "application/javascript") {
			mimeType += "; charset=utf-8";
		}

		//get the value from the KV store and return it if found
		const value = await env.assets.get(key, 'arrayBuffer')
		if(!value){
			return new Response("Not found", {
				status: 404
			})
		}
		return new Response(value, {
			status: 200,
			headers: new Headers({
				"Content-Type": mimeType
			})
		});
	},
} satisfies ExportedHandler<Env>;
```

This new code provides a specific endpoint, `/hello-world`, which will provide translated responses. When this URL is accessed, our Worker code will first retrieve the language that is requested by the client in the `Accept-Language` request header and the translations from our KV store for the `hello-world.json` key. It then gets the translated message and returns the generated HTML. 

With the Worker code running, we can notice that our application is now returning the properly translated "Hello World" message. From your browser's developer console, change the locale language (on Chromium browsers, Run `Show Sensors` to get a dropdown selection for locales).

## 6. Deploy your project

Run `wrangler deploy` to deploy your Workers project to Cloudflare with the binding to the KV namespace. Wrangler will automatically set your KV binding to use the production KV namespace set in our `wrangler.toml` file with the KV namespace id. Throughout this tutorial, we uploaded our assets to both the preview and the production KV namespaces.

We can now verify that our project is properly working by accessing our Workers default hostname and accessing `<WORKER-SUBDOMAIN>.<DEFAULT-ACCOUNT-HOSTNAME>.dev/index.html` or `<WORKER-SUBDOMAIN>.<DEFAULT-ACCOUNT-HOSTNAME>.dev/hello-world` to see our deployed Worker in action, generating responses from the values in our KV store. 

## Related resources

- [Rust support in Workers](/workers/languages/rust/).
- [Using KV in Workers](/kv/get-started/).