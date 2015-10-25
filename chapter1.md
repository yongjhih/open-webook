# 案例1 - signInWithTwitch 以 Twitch 登入，以 Parse + Oauth.io 為例

我們要有使用者的 Twitch 授權，並幫使用者建立帳號。

所以委託 oauth.io 取得 Twitch 授權，請 Parse 與 Twitch 驗證授權是否合法後，建立 Parse 帳號換證。

```html
<script src="//code.jquery.com/jquery-1.11.3.min.js"></script>
<script src="//code.jquery.com/jquery-migrate-1.2.1.min.js"></script>
<script src="oauth.min.js"></script>
<!-- https://parse.com/docs/downloads -->
<script src="//parse.com/downloads/javascript/parse/latest/min.js"></script>

<script>
    Parse.initialize("{parse_app_id}", "{parse_js_key}");

    $(document).ready(function() {
        $("#login").click(function(event) {
            OAuth.initialize('{oauth_app_id}')
            OAuth.redirect('twitch', "https://yongjhih.github.io/");
        });

        // ...

        OAuth.callback('twitch').done(function(result) {
            console.log(result);
            Parse.Cloud.run("signInWithTwitch", access_token).then(function(sessionToken) {
                console.log(sessionToken);
                return Parse.User.become(sessionToken);
            }, function(e) {
                console.log(e);
            });
        }).fail(function(e) {
            console.log(e);
        });
    });
</script>
```

cloud/twitch.js:

```js
var Parse = require('parse-cloud-express').Parse;

function s4() {
      return Math.floor((1 + Math.random()) * 0x10000).toString(16).substring(1);
}

function guid() {
      return s4() + s4() + '-' + s4() + '-' + s4() + '-' +
                   s4() + '-' + s4() + s4() + s4();
}

function guid20() {
      return s4() + s4() + s4() + s4() + s4();
}

/**
 * Returns twitchUser
 *
 * @param {String} token
 * @returns {Promise<TwitchUser>} twitchUser
 * @see https://github.com/justintv/twitch-api
 * @see https://github.com/justintv/Twitch-API/blob/master/authentication.md#scope
 */
function getTwitchUser(accessToken) {
    return Parse.Cloud.httpRequest({
        url: "https://api.twitch.tv/kraken/user",
           params: {
               oauth_token: accessToken
           }
    }).then(function(httpResponse) {
        return JSON.parse(httpResponse.text);
    });
}

/**
 * Returns email
 *
 * @param {String} token
 * @returns {Promise<String>} email
 */
function getEmail(accessToken) {
    return getTwitchUser(accessToken).then(function(twitchUser) {
        return twitchUser.email; // { email: "abc@example.com" }
    });
}

/**
 * Returns the session token of available parse user via twitch access token within `request.params.accessToken`.
 *
 * @param {Object} request Require request.params.accessToken
 * @param {Object} response
 * @returns {String} sessionToken
 */
function signInWithTwitch(request, response) {
    promiseResponse(signInWithTwitchPromise(request.user, request.params.accessToken, request.params.expiresTime), response);
}

/**
 * Returns the session token of available parse user via twitch access token.
 *
 * @param {Parse.User} user
 * @param {String} accessToken
 * @param {Number} expiresTime
 * @returns {Promise<String>} sessionToken
 */
function signInWithTwitchPromise(user, accessToken, expiresTime) {
    Parse.Cloud.useMasterKey();

    var userPromise;

    if (user) { // login
        userPromise = Parse.Promise.as(user);
    } else { // not login
        if (!accessToken) {
            return Parse.Promise.error("Require accessToken parameter");
        }

        userPromise = getTwitchUser(accessToken).then(function(twitchUser) {
            if (!twitchUser) return Parse.Promise.error("Invalid token");

            var userQuery = new Parse.Query(Parse.User);
            userQuery.equalTo("email", twitchUser.email);
            userQuery.equalTo("username", twitchUser.name);
            return userQuery.first().then(function(user) {
                if (user) return Parse.Promise.as(user);

                var newUser = new Parse.User();
                var username = twitchUser.name;
                var password = "twitch" + guid20();

                newUser.setUsername(username); // assert username not duplicated
                newUser.setPassword(password);
                newUser.setEmail(twitchUser.email);

                var attrs = {
                    // exception: {"code":251,"message":"custom credential verification for auth service twitch failed"}
                    //"authData": {
                    //    "twitch": {
                    //        "id": twitchUser._id,
                    //        "access_token": accessToken,
                    //        "expires_time": expiresTime
                    //    }
                    //},
                    uid: twitchUser._id.toString()
                };
                return newUser.signUp(attrs);
            });
        });
    }

    return userPromise.then(function(user) {
        return user.fetch();
    }).then(function(user) {
        if (user) {
            return Parse.Promise.as(user._sessionToken); // getSessionToken()?
        } else {
            return Parse.Promise.error("Twitch user not found");
        }
    });
};

/**
 * @param {Function(request, response)} func
 */
function defineParseCloud(func) {
    Parse.Cloud.define(func.name, func);
}

function promiseResponse(promise, response) {
    promise.then(function(o) {
        response.success(o);
    }, function(error) {
        response.error(error);
    })
}

defineParseCloud(signInWithTwitch);
```

https://api.twitch.tv/kraken/user?oauth_token={token}

```json
{
  "display_name": "yongjhih",
  "_id": 35947509,
  "name": "yongjhih",
  "type": "user",
  "bio": null,
  "created_at": "2012-09-04T01:50:36Z",
  "updated_at": "2015-10-19T08:48:00Z",
  "logo": "http://static-cdn.jtvnw.net/jtv_user_pictures/yongjhih-profile_image-d800fd20bcb0b04f-300x300.jpeg",
  "_links": {
    "self": "https://api.twitch.tv/kraken/users/yongjhih"
  },
  "email": "abc@example.com",
  "partnered": false,
  "notifications": {
    "push": false,
    "email": true
  }
}
```
