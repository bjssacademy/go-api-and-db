# Adding an API and a Database with Go

This tutorial expects that you have at least a passing understanding of Go, and a rudimentary knowledge of APIs and HTTP methods such as GET, POST, PUT etc.

This is a simple guide to creating a basic HTTP server API, using almost only the internal Go packages.

1. [Part 1 - Creating Our Web Server](/Part1/creating_our_webserver.md)
2. [Part 2 - Creating an Endpoint](/Part2/users_endpoint.md)
3. [Part 3 - Connecting to our Database](/Part3/connecting_to_a_database.md)
4. [Part 4 - Querying our Data](/Part4/querying_our_db.md)
5. [Part 5 - Creating Users with a Post Request](/Part5/posting_and_creating.md)
6. [Part 6 - Routing Requests](/Part6/multiplexing.md)
7. [Part 7 - Middleware & CORS](/Part7/middleware_and_cross_origin_requests.md)

---

> You may have problems accessing localhost on Chrome. If so follow these steps:
>
> As of 2020, Chrome automatically redirects all HTTP URLs to HTTPS. You can stop Chrome from automatically redirecting HTTP URLs to HTTPS for localhost by doing the following:
>
> Go to [chrome://net-internals/#hsts](chrome://net-internals/#hsts)

> Enter "localhost" (without quotes) in the box underneath "delete domain security policies"

> Click Delete

> Try accessing localhost again

> This only changes the setting for the localhost domain. It does not compromise security for other domains.