## How to hide your front-end API keys

Most of web API services require the use of an API key or an access token to get authorization. The problem with this setup is that when you include your credentials with your front-end code, any user who understands the basics of HTTP and knows how to open developer tools can access them and use it for their own benefit.

![1_4_2814JPbHwv-7NYvjaZeA.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1581525839394/jO07dhavX.png)

## API Authorization

When you make the request to the API service you need to include your credentials to let the service know that you are an authorized user. To do this you will be asked to include them on the query string or as a header when you make the request.

If you fail to include them in the request you will not be associated with an authorized user and the API service will return an error indicating that your app isn't authorized to request the data.

![1_FFgdZf0mPXJ_Ug2jqVKe9g.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1581526219917/nhAP-6z9y.png)

If you successfully included the credentials and if you are still below any quota or limit on your account, the service sends an OK response with the requested data.

## The Problem

Leaking your credentials becomes a major issue because anyone with your API key or authorization token will be able to make requests to the API service on your behalf, and those requests will count as yours. This means that if you are on a paid plan for the service, you will end up paying requests someone else is making on your behalf.

Additionally, if someone else uses your credentials outside the terms of service of the API provider or includes them in an app that is used for non-legal activities, you will be held responsible because the API provider will think you are the one making those requests. In the best-case scenario, your credentials will be revoked.

## The Solution

Since you need to include your authorization token on every request made to the API service and there is no way around this, we need to find a way to make the request to the API service including your credentials but at the same time removing them from your web app so they are not exposed on the front-end.

This is where a proxy comes into place.

Think of a proxy as a middleman. You will send the request to that middleman who will make some changes to the request and forward it to the API service. When the API service responds, the middleman will forward that response back to you.

That way you can remove your credentials from the front-end code by sending the request to the proxy, which will add your authorization method and forward it to the API service and back to you once it gets a response.

You will also tell the middleman to only respond to you. The middleman will be able to filter the requests by the domain they come from, using the origin header which is set by the browser once the request is made.

So basically, you are going to tell the middleman to only allow some domains to interact with it. If the request comes from your domain, it's going to be allowed by the middleman and proxied to the API service with the appropriate credentials, and then forward the response back to you. If the domain making the request is not allowed, then the middleman is just going to block that request.

During this process, your web app is completely unaware of your credentials, hence nobody is going to be able to get them.

Note that some API services require you to associate your credentials with one or more domains. The service will check your API key and the domain the request comes from and will authorize the request only when the origin was allowed by the user holding the credentials.

In those cases, there is no need for a proxy server because even if the credentials are visible, nobody is going to be able to make a request with those credentials from a domain that was not authorized by the user.

In some other cases, other API services enforce restrictions that don't allow them to be called directly from the client, so a proxy server is required to be able to handle the requests on behalf of the client.

## Setting your proxy server

Since the proxy is nothing more than a server that is forwarding requests, you can very easily deploy your own proxy server using any cloud provider.

You can set up a simple Node.js server using Express.js and the [http-proxy-middleware](https://github.com/chimurai/http-proxy-middleware) package, and deploy it to Heroku or any other cloud provider.

If you decide to use Heroku you can store your credentials as config vars. That way, they will be safely hidden and the proxy will add them on each request it forwards to the API service. Your front-end app won't know anything about your credentials, that's the job of the middleman.

Once your proxy server is up and running, you can then redirect the requests on your front-end code to target it. It will safely add your credentials and forward the request to the API service, and once it gets the response from the API service it will forward it back to your app.

## OK, but show me the code

To get your proxy server up and running on Heroku you can use the [api-key-proxy-server](https://github.com/MauricioRobayo/api-key-proxy-server) repository that has everything setup:

%[https://github.com/MauricioRobayo/api-key-proxy-server]

###### 1. Clone the repository

```sh
git clone https://github.com/MauricioRobayo/api-key-proxy-heroku.git
```

###### 2. Move into the repository and create a new Heroku app

```sh
cd api-key-proxy-heroku
heroku create
```

###### 3. Include your API keys

Include your API keys on the Heroku app. On the dashboard go to Settings and lookup for the Config Vars section. Copy and paste your API keys there using the same variable name you are using to retrieve it as an environment variable on each proxy service on the [config file](https://github.com/MauricioRobayo/api-key-proxy-server/blob/master/src/config.js). For example, in the case of the [Open Weather API proxy configuration](https://github.com/MauricioRobayo/api-key-proxy-server/blob/502806c8b5af5c95a135d88e9a5193ebdd6532b8/src/config.js#L12) that is included with the code, the variable name is `WEATHER_API_KEY`.

###### 4. Set up your API proxies

You can include all the API services you want using the [config file](https://github.com/MauricioRobayo/api-key-proxy-server/blob/master/src/config.js) which exports an object with two options: __allowedDomains__ and __proxies__.

```js
module.exports = {
  allowedDomains: ['https://www.example.com'],
  proxies: [ // We will handle this in a second ],
}
```

__allowedDomains__: An array of domains you want to allow to make calls to the proxy server. All the domains not listed here will be rejected with a CORS error.

Do not include pathnames:

* Wrong: https://example.com/some-path ❌
* Right: https://example.com ✔

Do not include trailing slash:

* Wrong: https://example.com/ ❌
* Right: https://example.com ✔

__proxies__: An array with the configuration options for each API service. The [config file](https://github.com/MauricioRobayo/api-key-proxy-server/blob/master/src/config.js) included provides configurations for the [Open Weather API](https://openweathermap.org/api), the [ipinfo API](https://ipinfo.io/), and the [GitHub API](https://developer.github.com/v3/). You can remove or add as many as you need:

```js
module.exports = {
  allowedDomains: ['https://www.example.com'],
  proxies: [
    {
      route: '/weather',
      allowedMethods: ['GET'],
      target: 'https://api.openweathermap.org/data/2.5/weather',
      queryparams: {
        appid: process.env.WEATHER_API_KEY,
      },
    },
    {
      route: '/ipinfo',
      allowedMethods: ['GET'],
      target: 'https://ipinfo.io/',
      queryparams: {
        token: process.env.IPINFO_TOKEN,
      },
    },
    {
      route: '/github',
      allowedMethods: ['GET'],
      target: 'https://api.github.com',
      headers: {
        Accept: 'application/vnd.github.v3+json',
        Authorization: `Token ${process.env.GITHUB_TOKEN}`,
      },
    },
  ]
}
```

Let's take a look at a proxy configuration from the above, it's an object that looks like this:

```js
{
  route: '/weather',
  allowedMethods: ['GET'],
  target: 'https://api.openweathermap.org/data/2.5/weather',
  queryparams: {
    appid: process.env.WEATHER_API_KEY,
  },
}
```

The __route__ is the path where your API proxy is going to listen for your requests. If you are using Heroku, you could reach your weather API proxy using the `weather` path on your Heroku app:

```http
https://your-proxy-server.heroku.app/weather
```

The __allowedMethods__ is an array of HTTP methods the server will proxy to the API service. It defaults to `GET` if omitted. For this example, we are explicitly setting it to `GET` as that's the only type of request we are going to do to the API.

The __target__ is the API endpoint that the proxy server is going to forward requests to. All requests made to the `route` on the proxy server will be proxied to the `target` endpoint.

The __queryparams__ is an object with additional query params to be added to the request that the proxy makes to the `target` endpoint. Since the [authorization by the Open Weather API](https://openweathermap.org/appid#use) is done by means of a query parameter, we include it here. Notice that the API key is going to be read from the environment variables that you set on the `Config Vars` section on Heroku and it's called `WEATHER_API_KEY`:

![1_EuPncS991yUhD9WldHfA_g.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1581527161159/3-mFbryj0.png)

For the Open Weather API, that's all you need. But there are some other options supported if you need more:

* __allowedDomains__: You can include specific allowed domains for a single proxy.
* __headers__: An object with the headers that will be added to the request made to the proxy server.
* __auth__: Basic authentication, for example: `user:password` to compute an Authorization header.

###### 5. Commit your changes and deploy to Heroku

```sh
git commit -am"Update proxy settings"
git push heroku
```

Finally, you can use your Proxy server to redirect the requests from your front-end code. Notice that you do not need to include any authorization key or token on the request to the proxy.

Request including authorization key 😢:

```js
const apiService = 'https://api.openweathermap.org/data/2.5/weather'

fetch(`${apiService}?q=${city}&units=${units}&appid=${apiKey}`)
  .then(response => response.json())
  .then(json => handleData(json))
```

Request to the proxy without authorization key 😊:

```js
const apiProxy = 'https://your-proxy-server.herokuapp.com/weather'

fetch(`${apiProxy}?q=${city}&units=${units}`)
  .then(response => response.json())
  .then(json => handleData(json)
```

If you want to test drive the proxy server without deploying to Heroku or any other cloud provider, you can check the section [test it on development](https://github.com/MauricioRobayo/api-key-proxy-server#test-it-on-development) of the documentation.

You can also check this weather app and its code which is using the proxy server.

## Conclusion

Your credentials should be private and it is a bad practice to let them out there on the wild where anybody can pick them up and use them for their own purposes.

When deploying a front-end app using a third-party API service that is not verifying the origin of the requests, make sure that you are using a proxy server to protect your identity from being stolen.

If you have any app that you deployed including your private credentials, go ahead and deploy your proxy server using the one suggested here or by configuring your own. Once you do that, remove the credentials from your front-end code and don't forget to revoke them as they were already exposed.

## Acknowledgments

Huge thanks to [@matheus-fls](https://github.com/matheus-fls) who provided invaluable feedback to make the repository code better, and to [@majovanilla](https://github.com/majovanilla) who took a lot of time to make this article more readable.

