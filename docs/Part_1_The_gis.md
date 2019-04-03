# Hacking into Instagrams unofficial API Part 1: The gis
11/18/2018



Instagram has a hidden API that not many people know about. This "hidden" API is not part of the [official API] that instagram offers. Instead, it's the API being used underneath all functionality of the social network. To use the public API is not easy, developers have to rely on reverse engineering HTTP requests and its data being sent. They must understand how each request connects with one another.

So why use the unofficial API instead of the public API? The answer is simple: Less restrictions . The official API requires every user who requests an API go through a lengthty form process. You will need to have a business, website, or app and explain to Instagram how the API will be used. This might not seem like a big deal, but for most cases one just wants to be able to access an endpoint without having any of those.

Another reason is the official API restricts their API to only a small amount of endpoints. Lets visit the [official endpoint] page to see what endpoints are currently available. As you can see, there's only endpoints for users and comments. There are many endpoints which are not currently in the official API. Would like to access instagram stories, maybe upload a photo? We're out of luck. The point is: If you find yourself in a position where you want to replicate some instagram functionality you will be out of luck and you will have to wait until such feature is available for you to use.

The good news is, you don't have to wait. There are many third-party APIs that make use of the unofficial API. If you're using Python, I highly recommend insta_api. [insta_api] is a python library that makes use of the unofficial API. I wrote it because most current API libraries nowadays are very unpythonic and use outdated technology, outdated endpoints which are currently deprecated and may dissapear in the future. insta_api has been tested throughly and also has tests, which many other libraries don't. This way you can be sure that the API will be working when you start developing on it.

But this section is not about insta_api. We will cover that in the last section. This section will be about reverse engineering into the hidden Instagram endpoints. So let's begin.

Instagram endpoints require a special hash called the `rhx-gis`. The rhx-gis cannot be easily obtained by their API and so the process of getting this hash value involves a little work.

First, we need to scrape the data from their homepage or somewhere else. The homepage will be just fine for this purpose. The `rhx-gis` value is obfuscated in some compiled javascript code. In order to get this value we will need to some regular expression manipulation
```python
ptrn = re.compile('("rhx_gis"):"([a-zA-Z0-9]*)')
match = ptrn.search(resp.text)
rhx_gis = match.group(2)
```
This regular expression is really simple. It searches the key-value pair for values matching the key `rhx_gis` and values with alphanumeric letters. The `rhx_gis` can be obtained from the second matching group

The rhx_gis will allow us to scrape Instagram data from public endpoints. This is important step because if we don't do this Instagram will throw us a forbidden error.

Here is a short list of what endpoints we can access with the rhx-gis:
```
graphql_endpoint = '/graphql/query/'
user_info_endpoint = '/{username}/?__a=1'
```
The user info endpoint will allow us to get detailed user information. It includes data for an user's username, full name, address (business), latests posts, profile picture, etc. We will be using it here shortly.

Now, For every request that we make we have to combine the rhx_gis that we obtained earlier with the parameters of the request in the following form:
```
rhx_gis: '{"par1": "val1", "par2": "val2", ...}'
```
Note: Don't forget that the colon is actually part of the encoding process and don't make the mistake (like I did) of omitting it.

We certainly don't want to have to hardcode the parameters so deserialize the dictionary of paramets with the `json.dumps` function to make this easier.

Here's how insta_api does it.
```python
def get_instagram_gis(self, params):
        """ Returns a generated gis to be used in request headers"""

        if not self.rhx_gis:
            self.get_rhx_gis()

        # Stringify
        stringified = None
        if isinstance(params, dict):
            stringified = json.dumps(params)
            log.info("STRINGIFIED: {}".format(stringified))
        else:
            stringified = params
            
        unhashed_gis = "{}:{}".format(self.rhx_gis, stringified)
        unhashed_gis = unhashed_gis.encode('utf-8')
        log.info("Unhashed gis: {}".format(unhashed_gis))
        encoded_gis = hashlib.md5(unhashed_gis).hexdigest()
        return encoded_gis
```
Because this is python 3 I encode to UTF-8 first and then using the md5 `hexdigest` function the [hashlib] library. You can use any library you want but hashlib is already built into python so that's what I recommend.

You might be wondering why I am doing type checking here. This is because some endpoints simply don't have any parameters on them so there's no need to stringify them in the first place.

Ok, now it's time for the fun. Let's test that this works. At the time of this writing the rhx_gis encoded syntax I showed earlier is what's being used internally for Instagram. However, in the past we needed to include user agent and other properties when combining. Obviously things have changed, so there's not guarantee that this method will work again in the future. But for now this works, so let's get started.

Initiate a request to the user info endpoint (i.e `/{username}/?__a=1`) where username is the username you want to fetch information from.

Let's use requests for this purpose
```python
import requests
headers = {'x-instagram-gis': instagram_gis}

resp = requests.get(`https://instagram.com/{username}?__a=1`, headers=headers)
```
The encoded gis will be sent via a header key called `x-instagram-gis`. Again you must do this for every instagram endpoint

If you did this right you should have received a `200` response json document. With that we can manipulate it in any way we wanted to.

The next section will cover authentication. You can navigate by clicking the link below
Hacking into instagrams API Part 2

[official API]: <https://www.instagram.com/developer/>
[official endpoint]: <https://www.instagram.com/developer/endpoints>
[insta_api]: <https://github.com/orlandodiaz/insta_api>
[hashlib]: <https://docs.python.org/3/library/hashlib.html>