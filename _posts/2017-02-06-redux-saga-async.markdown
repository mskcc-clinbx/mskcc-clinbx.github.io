---
layout: post
title:  "Graceful Asynchronicity with Redux-Saga and Friends"
date:   2017-02-06 12:00:00 -0500
author: Aaron Dack
categories: react front-end redux
---
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-91132149-1', 'auto');
  ga('send', 'pageview');

</script>

Here at MSK we have a large array of applications that are constantly communicating with remote API's. Naturally, we have the desire to not handle these actions synchronously. We want the ability to hit off a request, 
and then return to it some period later and handle it. We also want the ability to manage our application state in a centralized location, not just isolated to separate components that are unaware of each other. This blog post will
focus on asynchronous libraries specifically centered around redux. Redux is a predictable state container for JavaScript applications, allowing you to write consistent, predictable applications. Enough on that though, if you are interested
in the workings of redux, I strongly suggest to visit their docs (here)[http://redux.js.org/].

There are two main libraries right now that handles asynchronous processes for redux. The first is redux-thunk, and the second is redux-saga. The main difference between thunk and saga is that thunk utilizes promises while saga utilizes generator functions. Here at MSK we are big advocates for 
redux-saga. One of the main reasons we love sagas is that it allows us the ability to write code that looks synchronous but behaves asynchronously. This is especially powerful for keeping a grasp of where you are in your app and the logic that is driving your data fetching and processing. Besides that, the differences are quite small between the two libraries, depending on whether you like to work with generators or promises, and of course different API vernacular. 

###Working with API Requests - A Map Approach

There are a multitude of different ways to structure how you actual call an API, this is just one technique that we are implementing, and in no means exhaustive. We are going to construct our API map. 
First we are going to compose our header settings into an object that we are calling ```API_REQUEST_SETTING```.

```javascript
export const API_REQUEST_SETTINGS = {
  headers: {
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
};
```

Next, we construct the apiRequestMap using some dummy endpoints. 

```javascript
const API = 'http://foobar.com:8080';

const apiRequestMap = {
  postData({ postObject }) {
    return {
      url: `${API}/someEndpoint`,
      settings: {
        method: 'POST',
        body: JSON.stringify(postObject),
      },
    };
  },
  getData() {
    return {
      url: `${API}/someEndpoint`,
      settings: {
        method: 'GET',
      },
    };
  },
};

```

As you can see the apiRequestMap is contained within an object. We place each different endpoint and its associated HTTP actions as key-value pairs within the object. Next, we proceed to building what we like to call the service, which in essence
a plain function that takes the key-pair selected and fetches it. 

For example...

```javascript
export function dummyWebService(type, data) {
  const apiRequestConfiguration = apiRequestMap[type](data);
  return fetch(apiRequestConfiguration.url, {
    ...apiRequestConfiguration.settings,
    ...API_REQUEST_SETTINGS,
  }).then(res => {
    if (res.status >= 400) {
      return res.json().then(err => {
        throw err;
      });
    }
    return res.json();
  });
}
```

We select on the apiRequestMap with its given type, and pass in associated data in this lince. 

```javascript
const apiRequestConfiguration = apiRequestMap[type](data);
```

From there, we now have the given key and passed in its accompanying data, so we just return a fetch for that endpoint, combining the settings from the value in the apiRequestMap with the given API_REQUEST_SETTINGS that we declared above. 

```javascript
return fetch(apiRequestConfiguration.url, {
    ...apiRequestConfiguration.settings,
    ...API_REQUEST_SETTINGS,
  })
```

Which returns a promise that we handle gracefully here...

```javascript
.then(res => {
    if (res.status >= 400) {
      return res.json().then(err => {
        throw err;
      });
    }
    return res.json();
  });
}
```

And that's it. From here on out we can just plug whatever endpoint we want into our apiRequestMap and then just key on it to return our settings. But... where does redux-saga come in? How would we call this dummyWebService? 

###Using Redux-Saga

This section requires some nascent understanding of how generators work, but altogether will not be difficult to understand if you understand the methodologies underlying them. 

Here's what we like to call a watcher in redux-saga. 

```javascript
export function* watchFetchSomeFooData() {
  while (true) {
    const { someFooThing } = yield take(actions.REQUEST_SOME_ACTION);
    yield call(fetchTheFooData, someFooThing);
  }
}
```

For each type of action that you are listening for, there should be a separate watcher function that listens for it. This is an infinite loop that is always running in the background waiting for an action to be dispatched. When an action is dispatched the generator yields on the action that gets dispatched to the store (That should be familiar because its redux specific). Here we are destructuring our returned object on someFooThing which could potentially have some useful information (userIds, page number to paginate API with, etc). Next, we call our next generator function fetchTheFooData with the accompanying object.

If you are curious as to what the actual yield return on an action take, you could....

```javascript
const fooThing = yield take(actions.REQUEST_SOME_ACTION);
console.log(fooThing)
//It ends up containing a lot of content related to the actual redux request, so usually its best to destructure on the key you want from the object.

export function* fetchTheFooData(someFooThing) {
  const data = yield call(dummyWebService, 'postData', { someFooThing });
  yield put(actions.RECEIVE_SOME_FOO_DATA(data));
}
```

Here is where we actually fetch the data. We call the dummyWebService that we created above with two parameters, the key for the API endpoint that we are trying to access, and optionally some data that we will pass in. In this case
we are calling the dummyWebService with the postData string and an object containing someFooThing data. This will enter into the dummyWebService function and immediately get used in the first line here...

```javascript
const apiRequestConfiguration = apiRequestMap[type](data)

AKA...

const apiRequestConfiguration = apiRequestMap['postData']({ someFooThing });

```

This one-liner,

```javascript
const data = yield call(dummyWebService, 'postData', { someFooThing });
```

is all we have to wait on for our entire API response. We yield on it and smartly name our variable data, which is the return of the API response. From their we can let our redux store know that we have the data
by dispatching some action named ```RECEIEVE_SOME_FOO_DATA``` with the accompanying data. The 'put' method that redux-saga exposes to us from their API is essentially just a way for us to dispatch an action to our
redux store. 

Last, all we have to do is hook this saga up to our rootSaga and it will permanently listen for any actions with a given type that is kicked off. It will then proceed to fetch the data, and return it to your redux store.

```javascript
export default function* rootSaga() {
  yield [
    fork(watchFetchSomeFooData),
  ];
}
```

The reason we are using fork here is ... according to their docs ...

> fork, like call, can be used to invoke both normal and Generator functions. But, the calls are non-blocking, the middleware doesn't suspend the Generator while waiting for the result of fn. Instead as soon as fn is invoked, the Generator resumes immediately.

###Conclusion

In the end, handling asynchronicity and concurrent computations is oftentimes painful and difficult in applications. As the larger an application grows the need for a base to handle all API requests as well as a way to 
gracefully consume the responses becomes apparent. Luckily, redux-saga exposes a way for our applications to handle these challenges and does so in a way allowing us to write synchronous looking code. 

Stay on the look out for more updates from our team as we continue to expand our technology base while tackling complex challenges.

<br>
<br>

{% assign author = site.data.members | where:"name", page.author %}
{% for member in author %}
  <div class="teamImage">
    <img style="border-radius: 50%" src="{{site.url}}/images/team/{{member.image}}">
    <div class="profile">
  <span>{{member.name}}</span><br>
  <span style="font-style: italic; color: #82827A">
  {{member.title}}
  {% if member.linked-in != "" %}
  <a href="{{member.linked-in}}"><img style="width: 15px; height: 15px" src="{{site.url}}/images/linked-in-logo.png"></a>
  {% endif %}
  </span>
    </div>
  </div>
{% endfor %}
