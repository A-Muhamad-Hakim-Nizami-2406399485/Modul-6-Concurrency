# Advanced Programming Module 6: Concurrency (Web Server)

**Name**: Muhamad Hakim Nizami  
**NPM**: 2406399485  
**Course**: Advanced Programming (Pemrograman Lanjut)

This repository contains the tutorial implementation of a multithreaded web server in Rust, based on Chapter 20 of _The Rust Programming Language_ book.

---

## Reflection Notes

### Commit 1

In this first milestone, I built the foundation of the web server by setting up a TCP listener that binds to the local address `127.0.0.1:7878`. The `TcpListener::bind` function attempts to claim that port on the local machine, and `.unwrap()` is used to panic immediately if binding fails, for example, if the port is already in use. Once the listener is active, the f`or stream in listener.incoming()` loop continuously accepts incoming TCP connections, where each `stream` represents an individual client connection (the browser, in this case).

In the second version, I introduced the `handle_connection` function which takes a mutable `TcpStream` and reads data from it using a `BufReader`. The `BufReader` wraps the stream to provide efficient buffered reading, which is important because reading directly from a raw TCP stream byte-by-byte would be inefficient. I then used an iterator chain with `.lines()`, `.map()`, `.take_while()`, and `.collect()` to read the HTTP request line by line and stop at the first empty line. This works because HTTP requests end their headers section with an empty line (CRLF), so `take_while(|line| !line.is_empty())` is the exact signal that tells us we've read all the headers.

When I tested it in the browser, I could see the full HTTP request printed in the console, including the method, path, host, user-agent, and other headers. I also noticed that the browser sometimes sends multiple connection attempts because it expected an actual HTTP response back. This helped me understand that at this stage, the server is only reading requests, it's not yet responding to them, which is why the browser just hangs.

### Commit 2

In this milestone, I extended `handle_connection` to actually write a response back to the browser, so that it can render an HTML page instead of just hanging. The key insight here is the structure of an HTTP response: it needs a status line (`HTTP/1.1 200 OK`), followed by headers (like `Content-Length`), followed by a blank line, and then the response body. Each part is separated by `\r\n` (carriage return + line feed), which is the standard line terminator defined by the HTTP specification, browsers strictly expect this format, and using just \n would not work.

The `Content-Length` header is important because it tells the browser exactly how many bytes of body content to expect. Without it, the browser might wait indefinitely or misinterpret where the response ends. I calculated this using `contents.len()` after reading the HTML file with `fs::read_to_string("hello.html")`. One thing I had to be careful about is that `cargo run` must be executed from the project root directory, because the path `"hello.html"` is resolved relative to the current working directory if I ran it from a subdirectory, the file lookup would fail with a panic.

Finally, I used `stream.write_all(response.as_bytes())` to send the formatted response. The `write_all` method ensures the entire byte buffer is written (as opposed to `write`, which might only write part of it), and `as_bytes()` is needed because TCP streams work with raw bytes, not Rust strings. After running the server and visiting `http://127.0.0.1:7878`, my browser successfully rendered my personalized "Hello!" page, which was very satisfying to see. I also captured a screenshot and embedded it in the README using the standard markdown image syntax.

### Commit 3

In this milestone, I added basic routing to the server so it can distinguish between valid and invalid requests. Instead of reading the entire request into a `Vec`, I only read the first line (the request line) because that's the part that contains the HTTP method and the requested path (e.g., `GET / HTTP/1.1`). This is more efficient and matches what we actually need for routing decisions. If the request line matches `GET / HTTP/1.1`, the server serves `hello.html` with a `200 OK` status; otherwise, it serves `404.html` with a `404 NOT FOUND` status.

The refactoring step was an important lesson in applying the DRY (Don't Repeat Yourself) principle. In the unrefactored version, there were two nearly identical blocks of code, one for the success case and one for the 404 case, each building and writing the response separately. This kind of duplication is a code smell: if I later wanted to add a new header or change the response format, I'd have to remember to update it in both places, and it's easy to get it wrong. The refactored version uses a single `if/else` (or `match`) expression that returns a tuple `(status_line, filename)`, and the actual response-building logic is written only once below it. This makes the code shorter, easier to extend (adding a new route is just adding a new branch), and much less error-prone.

This pattern also mirrors how real web servers work internally, they have a routing layer that determines which handler and status to use, and a separate response layer that actually serializes and sends the HTTP response. Seeing this separation emerge naturally from refactoring was a nice way to appreciate why larger frameworks like Rocket or Actix-web are structured the way they are. I tested both `http://127.0.0.1:7878/` (which shows the Hello page) and `http://127.0.0.1:7878/awoakwoa` (which shows the Oops 404 page), and captured the 404 screen as evidence.

### Commit 4

In this milestone, I simulated a slow endpoint by adding a `/sleep` route that calls `thread::sleep(Duration::from_secs(10))` before responding. This artificially delays the response by 10 seconds, mimicking what might happen in a real application when a request involves a slow database query, a heavy computation, or an external API call that takes a long time to return. The purpose was to expose the fundamental limitation of a single-threaded server.

When I opened two browser tabs and hit `/sleep` in the first tab and then immediately hit `/` in the second tab, I observed that the second tab had to wait for the first one to finish before it could load, even though `/` by itself is essentially instant. This happens because our server processes requests sequentially inside a single thread: the `for stream in listener.incoming()` loop can only handle one connection at a time, and `handle_connection` is a blocking call. Until it returns, no other connection can be accepted or processed, so every subsequent request gets queued up behind the slow one.

This is a serious problem in any real-world scenario. If even one user hits a slow endpoint, every other user visiting any page on the site would be blocked until that slow request completes. In production, with hundreds or thousands of concurrent users, this single-threaded design would effectively make the server unusable. The experiment gave me a very concrete, visible demonstration of why concurrency matters and why every serious web server uses multiple threads, async I/O, or both. It also motivated the next milestone — instead of handling one request at a time, we need a way to handle multiple requests simultaneously, which leads to the ThreadPool pattern.
