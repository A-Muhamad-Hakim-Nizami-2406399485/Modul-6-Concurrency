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
