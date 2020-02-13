## 10x your code with the facade pattern, currying, and closures

Doing fetch requests is one of the most frequent tasks as a front-end JavaScript developer. If you are doing different fetch calls to different services on the same project, you might benefit from the [facade pattern](http://jargon.js.org/_glossary/FACADE_PATTERN.md), [currying](https://javascript.info/currying-partials), and [closures](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Closures).

### The Setup

We won't worry about error handling or edge cases as this is going to be just an example to illustrate the main points of this article.

Let's assume we are going to build a Weather App and we want to fetch the data from the [Open Weather API](https://openweathermap.org/api). We might use something like this in our code:

```js
const fetchWeather = async city => {
  const endpoint = 'https://api.openweathermap.org/data/2.5/weather'
  const querystring = `q=${city}&appid=${WEATHER_API_KEY}`
  const response = await fetch(`${endpoint}?${querystring}`)
  if (!response.ok) {
    throw new Error(`${response.statusText} (${response.status})`)
  }
  return response.json()
}

const init = async () => {
  const weatherData = await fetchWeather('barranquilla')
  console.log(weatherData)
}

init()
```

This looks fine so far. But now let's say that we want to get the city based on the IP address of the client using the [ipinfo API](https://ipinfo.io/), so we might be tempted to create another fetch function for that:

```js
const fetchIpinfo = async () => {
  const endpoint = 'https://ipinfo.io'
  const querystring = `token=${IPINFO_TOKEN}`
  const response = await fetch(`${endpoint}?${querystring}`)
  if (!response.ok) {
    throw new Error(`${response.statusText} (${response.status})`)
  }
  return response.json()
}

const fetchWeather = async city => {
  const endpoint = 'https://api.openweathermap.org/data/2.5/weather'
  const querystring = `q=${city}&appid=${WEATHER_API_KEY}`
  const response = await fetch(`${endpoint}?${querystring}`)
  if (!response.ok) {
    throw new Error(`${response.statusText} (${response.status})`)
  }
  const json = response.json()
  return json
}

const init = async () => {
  const { city, country } = await fetchIpinfo()
  const weatherData = await fetchWeather(`${city}, ${country}`)
  console.log(weatherData)
}

init()
```

By now you should have noticed that there is a big issue here: we are repeating ourselves while using the `fetch` web API. It is almost the same code on both the `fetchWeather` and the `fetchIpinfo` functions.

We know how to solve this. Let's refactor by moving the duplicate code to its own function which we will call `fetchData` so we can reuse it for both cases or any other that could come up:

```js
const fetchData = async (endpoint, querystring) => {
  const response = await fetch(`${endpoint}?${querystring}`)
  if (!response.ok) {
    throw new Error(`${response.statusText} (${response.status})`)
  }
  const json = await response.json()
  return json
}

const fetchWeather = async city => {
  const endpoint = 'https://api.openweathermap.org/data/2.5/weather'
  const querystring = `q=${city}&appid=${WEATHER_API_KEY}`
  return fetchData(endpoint, querystring)
}

const fetchIpinfo = async () => {
  const endpoint = 'https://ipinfo.io'
  const querystring = `token=${IPINFO_TOKEN}`
  return fetchData(endpoint, querystring)
}

const init = async () => {
  const { city, country } = await fetchIpinfo()
  const weatherData = await fetchWeather(`${city}, ${country}`)
  console.log(weatherData)
}

init()
```

There we go. Basically, we are providing a simplified interface to the `fetch` web API, and we have successfully implemented the __facade pattern__ to improve our code. That's all there is to it,  we are mastering OOP design patterns 😄.

But still, something doesn't seem right with the code. The `fetchWeather` and the `fetchIpinfo` functions are very similar. It's time for some __currying__. We will remove the first argument of our `fetchData` function and make it return a new function based on that, so we can create as many `fetch*` functions as we need based on the endpoint provided:

```js
const fetchData = endpoint => async querystring => {
  const response = await fetch(`${endpoint}?${querystring}`)
  if (!response.ok) {
    throw new Error(`${response.statusText} (${response.status})`)
  }
  return response.json()
}

const init = async () => {
  const fetchIpinfo = fetchData('https://ipinfo.io')
  const fetchWeather = fetchData('https://api.openweathermap.org/data/2.5/weather')

  const { city, country } = await fetchIpinfo(`token=${IPINFO_TOKEN}`)
  const weatherData = await fetchWeather(`q=${city}, ${country}&appid=${WEATHER_API_KEY}`)
  console.log(weatherData)
}

init()
```

Dam! That code is looking sexy! We have successfully removed a lot of the duplicate code using the __facade pattern__, and we are reducing the arity of our `fetchData` function by __currying__ it and making it return a new function depending on the `endpoint` that we are going to use.

Code is looking good, but now let's say we want to implement a cache to avoid doing repeated calls to the API services as the information is not likely to change between a few minutes.

What's more, the Open Weather API suggests making calls to the service [no more than once every 10 minutes](https://openweathermap.org/appid#use) because the information is not updated more than that.

> We recommend making calls to the API no more than one time every 10 minutes for one location (city / coordinates / zip-code). This is due to the fact that weather data in our system is updated no more than one time every 10 minutes.

Also, we won't need to be calling the `ipinfo` API very often because is not probable that we are going to change the city we are located between a few minutes.

Maybe we can store the data we have previously fetched using `localStorage` and set an expire time to get the information from there.

We can use 10 minutes expire time for the Open Weather API `localStorage` cache, and a longer cache time for the ipinfo API, for example, 30 minutes. That way we are lowering the requests made to the services decreasing the use of our requests quota, without affecting the accuracy of our app.

To do that, we can take advantage of the closure we have created when currying the `fetchData` function, passing an object literal with the endpoint and the expiration time in minutes for the cache:

```
const fromCache = (key, cacheInMinutes) => {
  const cacheInMilliseconds = cacheInMinutes * 60 * 1000
  if (localStorage[key] !== undefined) {
    const cache = JSON.parse(localStorage[key])
    if (Date.now() - cache.datetime < cacheInMilliseconds) {
      return { ...cache.data, cache: cache.datetime + cacheInMilliseconds }
    }
    localStorage.removeItem(key)
  }
  return false
}

const fetchData = ({ endpoint, cacheInMinutes }) => async querystring => {
  const url = `${endpoint}?${querystring}`
  const cache = fromCache(url, cacheInMinutes)
  if (cache) {
    return cache
  }
  const response = await fetch(url)
  if (!response.ok) {
    throw new Error(`${response.statusText} (${response.status})`)
  }
  const data = await response.json()
  localStorage[url] = JSON.stringify({ datetime: Date.now(), data })
  return { ...data, cache: false }
}

const init = async () => {
  const fetchIpinfo = fetchData({
    endpoint: 'https://ipinfo.io',
    cacheInMinutes: 30,
  })
  const fetchWeather = fetchData({
    endpoint: 'https://api.openweathermap.org/data/2.5/weather',
    cacheInMinutes: 10,
  })

  const { city, country } = await fetchIpinfo(`token=${IPINFO_TOKEN}`)
  const weatherData = await fetchWeather(`q=${city}, ${country}&appid=${WEATHER_API_KEY}`)
  console.log(weatherData)
}

init()
```

Take a look at that `init` function, isn't it expressive? We have written some clean and maintainable code using the __facade pattern__, __currying__, and __closures__. Now we can create as many `fetch*` functions as we want and we can give an expiration time in minutes for the `localStorage` cache for each of those functions.





