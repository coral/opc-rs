```
  ___  _ __   ___      _ __ ___
 / _ \| '_ \ / __|____| '__/ __|
| (_) | |_) | (_|_____| |  \__ \
 \___/| .__/ \___|    |_|  |___/
      |_|
```
## OPC-RS
A rust implementation of the open pixel control protocol

## Open Pixel control
Open Pixel Control is a protocol that is used to control arrays of RGB lights like Total Control Lighting (http://www.coolneon.com/) and Fadecandy devices (https://github.com/scanlime/fadecandy).

## Documentation

https://docs.rs/opc

## Examples
You can run the random color example with:

```bash
cargo run --example random
```

If you need to specify another server for the example:

```bash
OPC_ENDPOINT=192.168.0.42:7890 cargo run --example random
```

## Usage:

### Client:

```rust
extern crate opc;
extern crate tokio_core;
extern crate tokio_io;
extern crate futures;
extern crate rand;

use opc::{OpcCodec, Message, Command};
use futures::{stream, Future, Sink, future};
use rand::Rng;

use tokio_io::AsyncRead;
use tokio_core::net::TcpStream;
use tokio_core::reactor::Core;

use std::env;
use std::io;
use std::time::Duration;


fn main() {

    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let endpoint = env::var("OPC_ENDPOINT")
        .unwrap_or(String::from("127.0.0.1:7890"));
    let remote_addr = endpoint.parse().unwrap();
    let mut rng = rand::thread_rng();

    let work = TcpStream::connect(&remote_addr, &handle)
        .and_then(|socket| {

            let transport = socket.framed(OpcCodec);

            let messages = stream::unfold(vec![[0,0,0]; 512], |mut pixels| {

                for pixel in pixels.iter_mut() {
                    for c in 0..3 {
                        pixel[c] = rng.gen();
                    }
                };

                let pixel_msg = Message::from_pixels(0, &pixels);

                std::thread::sleep(Duration::from_millis(100));

                Some(future::ok::<_,io::Error>((pixel_msg, pixels)))
            });

            transport.send_all(messages)

        });

    core.run(work).unwrap();
}
```

### Server:

```rust
extern crate opc;
extern crate futures;
extern crate tokio_core;
extern crate tokio_io;

use opc::OpcCodec;
use futures::{Future, Stream};

use tokio_io::AsyncRead;
use tokio_core::net::TcpListener;
use tokio_core::reactor::Core;

fn main() {
    let mut core = Core::new().unwrap();
    let handle = core.handle();
    let remote_addr = "127.0.0.1:7890".parse().unwrap();

    let listener = TcpListener::bind(&remote_addr, &handle).unwrap();

    // Accept all incoming sockets
    let server = listener.incoming().for_each(move |(socket, _)| {
        // `OpcCodec` handles encoding / decoding frames.
        let transport = socket.framed(OpcCodec);

        let process_connection = transport.for_each(|message| {
            println!("GOT: {:?}", message);
            Ok(())
        });

        // Spawn a new task dedicated to processing the connection
        handle.spawn(process_connection.map_err(|_| ()));

        Ok(())
    });

    // Open listener
    core.run(server).unwrap();
}
```
