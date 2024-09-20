---
layout: project.liquid
name: hmv-digital
title: hmv-digital
---

Since moving to New York, I’ve met a lot of cool people in music tech that are always working on cool projects and building new digital products. One sharp music product guy out of Brooklyn, [Jason Herskowitz](http://twitter.com/jherskowitz), has been in the music streaming space for quite some time and has been working on a product of his own:

### [Tomahawk](https://www.tomahawk-player.org/)
> “A new kind of music player that invites all your streams, downloads, cloud music storage, playlists, radio stations and friends to the same party. It’s about time they all mingle.” 

Jason reached out with an interesting project, in collaboration with [hmv Digital](hmvdigital.com). hmv Digital was looking to expand the functionality of their native Mac OSX hmv Player and integrate a E-Commerce checkout popup utilizing [7Digital’s](http://7digital.com) APIs. I loved the idea of embedding a widget iFrame into a native app and utilizing web functionality in a native webview.

## Discovery

Considering hmv Digital had a brand style guide, most of the discovery was not necessarily in the design, but in utilization of 7Digital’s palette of APIs. I spent a good amount of time studying the [API Docs](http://docs.7digital.com/) and was excited to utilize the [7Digital Node API Client](https://github.com/raoulmillais/node-7digital-api). The main services we decided would be useful were:

- Catalogue (searching the desired tracks and releases)
- User Management (to register/login users)
- Locker (checking a user’s existing locker for items already purchased)
- Card Management (adding/removing new credit cards)
- Purchasing (charging through 7Digital accounts)

<br>

Along with utilizing the API's functionality, hmv needed localization support for purchasing being done in the UK, Ireland, and Canada. We learned that this could be handled through a specific `shopid` that was registered directly with 7Digital for store accounts.

## UI/UX

Within the player, users needed the ability to click a “buy” button on a track or release, and open a popup to navigate through entire checkout flow, make a purchase, and close the popup. Considering there was a similar checkout flow for 7Digital’s own store, we decided to start with that and iterate on the experience as needed. Upon selecting a track or release the popup would pre-populate the item information. The checkout experience from there was fairly straightforward:

[Register/Login] -> [Add/Choose Credit Card] -> [Confirmation/Ad/Download]

![](/assets/images/hmv-digital/screen_connect.png)
![](/assets/images/hmv-digital/screen_pay.png)
![](/assets/images/hmv-digital/screen_download.png)

## The Challenge

Along with utilizing the 7Digital Node.js client, I needed to implement client-side routing with Angular on top of an express app. With this set up, I could simply serve the client app and build a small API to proxy 7Digital requests and handle session management.

When selecting a release or track in the player, we sent in the authenticated user OAuth token and `release_id` or `track_id` depending on which was selected. Using the Catalogue API we needed to search for the release/track and cache the item in a session.

With the User Management API we would then register a new user or login an existing user and create a new session for them. They would then need an existing card to make a purchase, so the Card Management API allowed us to either register a new localized card in compliance with the shopid locations (UK, Ireland, Canada) or proceed with their existing card.

Finally with the Purchasing API we could charge their card for the item and it would POST an update to their locker.


## Results

For client-side routing I used [UI Router](https://github.com/angular-ui/ui-router) and handled API requests with promises inside a resolve on each route. Using `$stateChangeSuccess` I could cache a user’s state and check if they were still logged in or not.

Within Angular, I utilized the `$http`, `$q`, and `$cookieStore` services to handle API requests. I maintained state within a global App service, similar to an App Store in Flux. When a user logged in, I could update the App model and set a cookie.

### App

```js
  login: function (email) {
    // set user logged in
    this.user.loggedIn = true;
    $cookieStore.put(‘userLoggedIn’, true);
    $cookieStore.put(‘userEmail’, email);
    return email;
  }
```

### Login

```js
  loginUser: function (user) {
  	var dfd = $q.defer();

    $http.post(‘/sdigital/login’, user).then(function (res) {
       App.user.login(res.data.email);
       dfd.resolve(res.data);
    }, function (err) {
       App.errors.push(err.data.response);
       dfd.reject(err.data);
    });

    return dfd.promise;
  }
```

On the backend it got a little crazier:

```js
router.post(‘/sdigital/login’, function (req, res, next) {
  
  step(
    // check if the user exists
    function checkExistingUser() {
      userInternal.find({
        emailAddress: req.body.userName
      }, this);
    },
    // handle user signup
    function handleUserSignup(err, response) {
      // create new credentials
      creds = {};
      
      if (err) {
        res.json(err);
        return;
      }
      
      // sign up if no users were found
      if (!response.users) {
        userPublic.signup({
          emailAddress: req.body.userName,
          password: req.body.password
        }, this);
      } else {
        creds.userId = response.users.user.id;
        return creds;
      }

    },
    function getRequestToken(err, response) {

      if (err) {
        res.json(err);
        return;
      }

      if (response.users) {
        creds.userId = response.users.user.id;
      }

      // Get a request token using the oath helper
      var host = req.hostname;
      oauth.getRequestToken(host, this);
    },
    function authorise(err, requestToken, requestSecret, authoriseUrl) {

      if (err) {
        // Something went wrong, show the user and exit
        console.error(‘Error getting the request token’);
        console.error(util.inspect(err, { depth: null, colors: true }));
        res.status(400).send(err);
        return;
      }

      // Remember the token and secret so we can access it after the
      // user presses enter
      creds.requestToken = requestToken;
      creds.requestSecret = requestSecret;

      oauth.authoriseRequestToken({
        username: req.body.userName,
        password: req.body.password,
        token: creds.requestToken
      }, this);

    },
    function continueAfterAuthorisation(err) {
      if (err) {
        req.session.errors = err;
        res.status(401).json(err);
        return;
      }

      // Get an access token using the oath helper using the authorized
      // request token and secret
      oauth.getAccessToken({
        requesttoken: creds.requestToken,
        requestsecret: creds.requestSecret
      }, this);

    },
    function logTheAccessToken(err, accessToken, accessSecret) {
      if (err) {
        // Something went wrong, show the user and exit
        console.error(‘Error getting the access token’);
        console.error(util.inspect(err, { depth: null, colors: true }));
        req.session.errors = err;
        res.status(400).send(err);
        return;
      }

      // store user
      req.session.user = {
        email: req.body.userName,
        accessToken: accessToken,
        accessSecret: accessSecret
      };

      userAPI = api.reconfigure({
        defaultParams: {
          accesstoken: req.session.user.accessToken,
          accesssecret: req.session.user.accessSecret,
          country: req.session.country.code
        }
      });
      
      newUser = new userAPI.User();

      res.json({
        email: req.session.user.email,
        accessToken: req.session.user.accessToken
      });
      return;
    }
  );

});
```

With that out of the way, we could start accessing the user’s locker to check for the existing track/release. If they had the track, we would just use `$state.go(‘download’)` to send them to the confirmation page. Otherwise, we could take them to the pay route, where they could add/choose a credit card.

Without going into the details on how exactly they added their card, it allowed for the experience to proceed, and furthermore charge the retail price or `rrp` of the selected item.

Finally, they were taken to a confimration screen, where they could finish the flow and view instructions and an Ad to finish their experience.

### Conclusion

Overall, hmv Digital was pleased with the final experience. We tested it during the QA phase and adjusted a few styles here and there. The 7Digital API was mostly a joy to work with and it was an exciting project to take the lead on. Go download the new [hmvPlayer](https://hmvdigital.com/pages/836) for FREE to check it out!

As an aside, I must have spent more money on music during the course of the project, but in the end it was worth it for the music and technology.

<div class="text-center" style="margin-bottom: 1em;">
  <a href="http://gigwax.com">Visit Live Project</a> | <a href="/projects" class="text-muted small">View More Projects</a>
</div>
