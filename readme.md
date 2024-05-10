# Adding an API and a Database with Go

This tutorial expects that you have at least a passing understanding of [Go](https://github.com/bjssacademy/goinaday/blob/main/readme.md), and a rudimentary knowledge of APIs, [JSON](https://github.com/bjssacademy/fundamentals-json) and [HTTP](https://github.com/bjssacademy/fundamentals-http) methods such as GET, POST, PUT etc, as well as an understanding of [SQL basics](https://github.com/bjssacademy/fundamentals-sql/tree/main?tab=readme-ov-file#sql-basics).

This is a simple guide to creating a basic HTTP server API, using almost only the internal Go packages.

1. [Part 1 - Creating Our Web Server](/Part1/creating_our_webserver.md)
2. [Part 2 - Creating an Endpoint](/Part2/users_endpoint.md)
3. [Part 3 - Mocking Our data](/Part3/mocking_our_data.md)
4. [Part 4 - Creating Users with a Post Request](/Part4/posting_and_creating.md)
5. [Part 5 - Routing requests](/Part5/multiplexing.md)
6. [Part 6 - Middleware & CORS](/Part6/middleware_and_cross_origin_requests.md)
7. [Part 7 - Refactoring](/Part7/refactoring_our_code.md)
8. [Part 8 - Connecting to a Database](/Part8/connecting_to_a_database.md)
    1. [Part 8.1 - Querying Our Data](/Part8/querying_our_db.md)
    2. [Part 8.2 - Inserting Into Our Database](/Part8/create_and_return_id.md)
9. [Part 9 - Database Migrations](/Part9/database_migrations.md)
10. [Part X - The Repository Pattern](/PartX/repository-pattern.md)

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
