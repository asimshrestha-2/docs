---
title: jsConnect Quickstart
tags:
- Features
- Single Sign-On
- jsConnect
- quickstart
category: help
menu:
  help:
    parent: sso
    weight: 4
aliases:
- /features/sso/jsconnect/seamless
- /help/sso/jsconnect/seamless
---

jsConnect uses javascript to allow cross-domain single-sign-on with another site. We provide several [client libraries](/help/sso/jsconnect/#your-endpoint-client-libraries) to help you implement jsConnect on your site. If your site has been programmed in a language that doesn't have a client library than this is documentation is for you.

## Functions You'll Need

jsConnect makes use of some functions that are available in most modern programming languages. You might want to do a bit of research to see if your language supports these functions because they are quite difficult to implement yourself.

We've tried to make the functions a generic as possible, but they'll most likely be called something different in our language. A little googling should help you find what you need.

- **timestamp**: Return the current unix timestamp. Note that your server has to have it's time reasonably in synch with Vanilla's server.
- **JSON encode**: Encode an object/associative array in the JSON (ex. `{"name": "Joe", "email": "joe@noreply.com"}`)
- **query string encode**: Encode an object/associative array like a query string in RFC1738 (ex. `name=Joe&amp;email=joe%40noreply.com`)
- **MD5, SHA1, SHA256**: Create a hash of a string. We strongly recommend SHA256.

We also have an SSO module for automatically signing in to an embedded forum. If you are for sure going to just embed Vanilla then you might want to try this first.

- **base64 encode**: Encode a string in base64 notation.
- **hmac**. Call the HMAC-SHA256 hashing algorithm on a string and a secret key.

## Concepts

### JSONP
JSONP is a technology that browsers can use to get around the same-origin limitation of browsers and make cross-domain requests. Vanilla uses JSONP to transmit some of its SSO information. When you program a jsonp page it must follow a specific format. To best illustrate this lets use an example.

Let's say I want to request a jsonp response from the `/getname.json` page. What I'd do is make a request to that page with a querystring parameter called **callback** like this: `/getname.json?callback=displayname.`

Now if I'm programming /getname.json it's my job to display my response in jsonp format which would look something like this:

```js
displayname({"name": "Todd"});
```

You'll notice a couple of things here:

1. I passed a **callback** parameter to the page. The value of the parameter is used as the name of a function I'm calling in the response.
1. The function is passed an argument encoded in json. Whatever I want to pass to the other site must be json encoded like this.


**Important: When writing a page that returns jsonp you must ALWAYS return valid javascript. If you return HTML or something else then the page won't work. This is something that a lot of people get wrong.**

### Client ID and Secret
In order to secure your single-sign-on connection you must share a **client ID** and **secret** between your site and Vanilla. Your client ID is used to identify your site and is public. Your secret is like a site-password and must be kept secret. **Do not share your secret with anyone.**

When setting up jsConnect in Vanilla's dashboard there is a button to generate a client ID and secret. You can use this button or create your own. You can use a more freindly name for your client ID, but stick to numbers and letter. We recommend using a secret that is generated by Vanilla.


#### Signing a Request

If you've done any single sign-on or authentication work you've probably run into the concept of **signing a request**. Basically, you sign a request with your shared secret so that both sites know that the request came from a legitimate source. When you transmit user information you want to also sign that information so that I know it came from your site and not someone else trying to hack into Vanilla.

jsConnect uses signatures to secure its requests. To sign a request you use a **hash function** such as MD5, SHA1 or SHA256. You can select a variety of hash functions, but we recommend using **SHA256** because it is more secure and widely available.

We recommend getting jsConnect working without signatures first and then securing them. In Vanilla you can put your jsConnect connection in **test mode** and it won't check for a signature. **Just make sure you don't go live without signing your requests**.

## Site-Wide SSO
For site-wide SSO you'll need to make a page on **your site** that provides authentication information. We call this your **authentication page**. You can call it whatever you want, but for the purposes of this documentation we'll call it **/authenticate.json**

### Request

```bash
GET /authenticate.json?parameters
```

### Parameters

- **client_id: REQUIRED. ** Your shared client ID that you've configured in Vanilla's dashboard.
- **callback: REQUIRED. **The name of the callback function.
- **timestamp: OPTIONAL. ** For secure requests, Vanilla will send you the timestamp of its request. If you are checking security then you want make sure that the timestamp isn't too old. We recommend a timeframe of  5-30 minutes.
- **signature: OPTIONAL. ** For secure requests, Vanilla will sign the timestamp parameter

### Response

There are four valid responses you can return depending on the request and whether or not the user is signed in.

#### No User Response: The user is not signed in to your site .

```js
callback({"name": "", "photourl": ""});
```

**Notes**: Even if there is no user signed in you still must return a valid jsonp response. So in this case we just return an empty user.


#### User Stub Response: The user is signed in to your site, but the request hasn't been signed.

```js
callback({
    "name": "John Doe",
    "photourl": "http://nosite.com/johndoe.png"
});
```

**Notes**: Vanilla sends an un-signed request to your page just to provide a picture and photo to the user as a notice that they are signed on to your site.

#### User Response: The user is signed in to your site, and the request is signed (or you are testing).

```js
callback({
    "uniqueid": "1234", // REQUIRED. The ID that uniquely identifies the user on your site.
    "name": "John Doe", // REQUIRED. The name of the user on your site.
    "email": "johndoe@noreply.com", // REQUIRED. The user's email address
    "photourl": "http://nosite.com/johndoe.png", // OPTIONAL. A photo for the user.
    "roles": "member,administrator", // OPTIONAL. You can configure jsConnect to synchronize roles
    "client_id": "123456789", // REQUIRED. Your client ID.
    "signature": "cdb398fdab244999d8ba301eb6334298" // REQUIRED. The signature of this response.
});
```

**Notes**: This response is the heart of jsConnect. It tells Vanilla about the user signed in to your system so that Vanilla can sign the user into Vanilla too. There are a few important points about this response.

- **uniqueid, name, client_id, **and **signature** are required. Make sure you provide us with these values in your response and make sure they have values.
- The **uniqueid** field must never change for a given user. Your site usually has some sort of integer or guid that uniquely identifies a user. Use this.
- The signature is used to secure the response and is very important, but can be left out if you are just testing. How to sign the response is explained below.

#### Error Response: Some error has occurred (error response).

```js
callback({
    "error": "invalid_client", // REQUIRED. A string identifier for the error.
    "message": "Client ID does not match.", // REQUIRED. A user-readable error message.
});
```

**Notes**: You usually return an error response if a request has invalid security information. However, you can also return an error response if your application encounters an unknown error on its own.



### Other Notes on your Response

- Make sure you return an HTTP 200 response (i.e. no error codes).
- You can set the Content-Type to application/javascript if you wish, but it's not necessary.

## Securing Site-Wide SSO
You can get your jsConnect page working without worrying about security, but in order for your SSO to be secure you have to a) check the request to make sure it is secure, and b) sign your response so that Vanilla can make sure it's secure.
### Checking the Request for Security
In order to make sure the request from Vanilla is legitimate you must first check the request. You do this on your **authentication page** before displaying your response. If the request doesn't check out you need to display an error response with some information on what might have gone wrong. Here are the things you should check.

1. Make sure the request provides a **client_id**. **error: invalid_request, message: The client_id parameter is missing.**
1. Make sure **client_id** matches the client ID you've set up on your site. **error: invalid_client, message: Unknown client.**
1. See if a **timestamp** was provided. If no timestamp was provided return the **no user response** or the **user stub response.**


If a timestamp was provided you need to make sure it checks out.




1. Make sure the timestamp isn't at most 5-30 minutes in the past or future. **error: invalid_request, message: The timestamp is invalid.**
1. Make sure the request has a signature. **error: invalid_request, message: Missing  signature parameter.**
1. Make sure the signature is valid. To check the signature you must do the following:

```js
if (hash(concat(timestamp, secret)) == signature)
// valid
else
// invalid
```

Basically, you concatenate your secret to the end of the timestamp and call your hash function. If your calculated signature matches the signature passed to the request then the request is valid. **error: access_denied, message: Signature invalid.**

If you've checked all of the above conditions and haven't encountered any errors then the request is valid and you can display the appropriate response. First you need to sign it though.

### Signing your Response for Security

Once you've checked the request for security you need to sign your response so that Vanilla can check security on its end. To show you how to do this consider the following user's sso information and the following client ID and secret.

```js
user = {
    uniqueid: "1234",
    name: "John Doe",
    email: "johndoe@noreply.com",
    photourl: "http://nosite.com/johndoe.png"
};
```

```js
client_id = "123456789";
secret = "985d2f9eb57a8b55db3c04c20272bce9308764b0"; // don't give this out!!!
```

Here is how you sign the user with the secret.

1. Sort the user by keys:

```
user_sorted = {
    email:"johndoe@noreply.com",
    name:"John Doe",
    photourl:"http://nosite.com/johndoe.png",
    uniqueid:"1234"
};
```

2. Url encode the sorted user with RFC1738 (a fancy way of saying make it a query string). This gives you your **signature string**.

```js
signature_string =
"email=johndoe%40noreply.com&amp;name=John+Doe&amp;photourl=http%3A%2F%2Fnosite.com%2Fjohndoe.png&amp;uniqueid=1234";
```

3. Concatenate the signature with your secret and call your hash function around everything:

```js
signature = sha256(concat(signature_string, secret));
```

4. Now you want to add both your client_id and your signature back into the user.

```js
user = {
    uniqueid: "1234",
    name: "John Doe",
    email: "johndoe@noreply.com",
    photourl: "<a href="http://nosite.com/johndoe.png">http://nosite.com/johndoe.png</a>",
    client_id: "123456789",
    signature: "3c982c0b50bc06deb0b9df2a9a0770b6f88b3749"
};
```

5. Now that you have your signed response you can display it as jsonp.
6. You're done!

### Troubleshooting

- Make sure you don't add your client_id or signature to your sso user until the **last** step.
- If your usernames are going to use non-ascii characters such as accents then make sure your page is encoded using utf-8 and that you have an appropriate Content-Type header.
- Try and make sure your system doesn't have any spaces at the beginning or end of usernames or email addresses. You won't be able to see the spaces, but they usually aren't allowed in usernames and will mean the user isn't allowed to register with Vanilla.

### What's Next
At this point you should have a valid jsConnect authentication page and Vanilla will be able to recognize your users. You just need to set up the connection in Vanilla and you're good to go.

If you want to implement embedded SSO then visit [our embedded SSO documentation](/help/sso/jsconnect/#embedding-with-seamless-jsconnect).
