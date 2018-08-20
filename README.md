# String concatenation considered harmful

## (for security and family BBQ's)
You do know what is wrong with the following (Pythonish) code:

```python
cursor.execute("SELECT * FROM USER WHERE USER_NAME='" + user_name + "')
```

If not, refrain from writing production code while reading the search results of googling 'sql injection' (or go to a family BBQ).
Of course, a careful programmer would use a properly parametrized query:

```python
cursor.execute("SELECT * FROM USER WHERE USER_NAME=?", (user_name,))
```

We will not further discuss the specifics of SQL injections here, but the problem with the code above is more general: the programmer was making use of string concatenation to construct an expression in a secondary language SQL involved in the Python project at hand.
The correct parametrized query was constructed using some libary such as ```sqlite``` without using direct string concatenation.

In a modern application, those 'secondary' languages are abundant. Besides SQL, a project could use some XML, HTML or JavaScript. 
Even a 'mini' language like the set of valid URI's can be considered such a language when the application should, for example, access some external Rest API.

>If your program generates expressions in any formal language using 
>string concatenation, you are probably reinventing the wheel, 
>introducting bugs and vunerabilies, or all of the above.

An academic problem? Don't think so.
Every programmer knows (should know) how to prevent SQL injections, but I have seen code like this in the management console of an actual production system:

```python
...
html += "<p>name: " + name + "</p>"
...
```

What if ```name``` happens to be ```"Bert & Ernie"```?
The ```&``` in the generated HTML is not properbly encoded, which might lead to some customer reporting a cosmetical (P5) bug in Jira.
This can be handled next Monday. 
Things get worse if somebody is able to get the following value for ```name``` in the database, the company might be broke next Monday:

```python
"Bert is evil<script>alert('pwned!')</script>"
```

This could have been easily prevented by using some well-established XML/HTML libary:

```javascript
import "WellKnownXmlHtmlLibrary"
...
let element = html.appendElement("name")
element.setTextContent(name)
// element content is OK now: "Bert is evil&#x3C;script&#x3E;alert(&#x27;pwned!&#x27;)&#x3C;/script&#x3E"
```

Back to the 'mini' language of URI's.
Here is some innocent looking (Java) code to generate a string containing an URI of the base of some API:

```java
// format a url to our API at host and port
// hostname shoud be a valid hostname or IP address
public String baseUrl(String host, int port) throws URISyntaxException {
  if (port <= 0) throw new URISyntaxException(Integer.toString(port), "port should be positive");
  return "http://" + host + ":" + port + "/api";
}
```

We should be able to fill in a host name or an IP address (this value might come from some configuration file, who knows).
It looks like it works:

```java
System.out.println(baseUrl("fijma.net", 80));
// http://fijma.net:80/api
System.out.println(baseUrl("192.168.1.1", 8080));
// http://192.168.1.1:8080/api
```
    
The programmer even gave robustness some thought, checking for positive port numbers:

```
System.out.println(baseUrl("fijma.net", 0));
// throws: java.net.URISyntaxException: port should be positive: 0
```

One could argue that the code contains some 'magic' values like ```/api```, but that's not the point here.

Actually, a P1 bug is luring in the code.
The comment (and the original requirements), state that ```host``` can be a valid IP address.
That includes IPv6.
At least, that is what OPS thought (having read the original requirements).

While reinventing the wheel using naive string concatenation, the programmer forgot the case where an IPv6 address in a URI should be enclosed in square brackets:

```java
System.out.println(baseUrl("2001:db8:85a3::8a2e:370:7334", 8080));
// Oops: expected: http://[2001:db8:85a3::8a2e:370:7334]:8080/api
// actual: http://2001:db8:85a3::8a2e:370:7334:8080/api
```

A minor bug, easily fixed using some extra mess:   

```java
// format a url to our API at host and port
// hostname shoud be a valid hostname or ip address (IPv4 or IPv6)
public String baseUrl(String host, int port) throws URISyntaxException {
  if (port <= 0) throw new URISyntaxException(Integer.toString(port), "port should be positive");
    if (host.contains(":")) {
      // assume IPv6 literal address
      host = "[" + host + "]";
  }
  return "http://" + host + ":" + port + "/api";
}
```

The problem with this bug is that it showed up on a Sunday.
The OPS guys were moving the application to and IPv6-only production enviroment.
The programmer had to leave the familiy BBQ to commit a 'trivial' fix to this P1 bug over a broken WiFi network (what was the VPN password, again?).

  
The point is, the platform (Java, in this case), already provides means to construct a valid URI from its components. The function ``baseUrl`` can be rewritten in terms of the standard Java library class ```java.net.URI```:

```java 
import java.net.URI;

public String baseUrl(String host, int port) throws URISyntaxException {
  // 'null' for userInfo, query and fragement, we don't need those
  return new URI("http", null, host, port, "/api", null, null).toString();
}
```

This works OK for IPv6 out-of-the-box:

```java
System.out.println(baseUrl("::1", 8080));
// http://[::1]:8080/api
```

It will even work-out-of-box once they invent IPv8. At least, after upgrading to a future version of Java that implements this imaginary version of the IP protocol.

So, don't reinvent the wheel.
Use a library.
To generate SQL queries without SQL injections.
To generate XML with properly escaped variable content.
To generate HTML without cross-site scripting vunerabilities.
And even to generate a valid URI from a host name and a port value.

You should only use string concatenation a meant here if you are actually the implementor of the libary. But then you probably are working for Oracle, Microsoft, Google or suffer from the "Not invented here syndrome" ([NIHS](http://localhost/#)).





