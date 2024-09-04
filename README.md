# test_pyframe


# Connections.rs


```rust

use std::collections::HashMap;
use std::net::SocketAddr;
use std::sync::Arc;
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio::sync::Mutex;
use serde_json::Value;



#[allow(dead_code)]
pub(crate) struct Connection {
    write: Arc<Mutex<tokio::io::WriteHalf<TcpStream>>>,
    read: Arc<Mutex<tokio::io::ReadHalf<TcpStream>>>,
    callback_store: Arc<Mutex<HashMap<String, Box<dyn Fn(String) + Send + Sync>>>>,
    acivate:Option<bool>
}

impl Connection {
    #[allow(dead_code)]
    pub(crate) async fn emit(&self, event: &str, data: &str) -> io::Result<()> {
        // Wir setzen das Event in den JSON-String
        if let Some(true) = self.acivate {
            let mut json_data: Value = serde_json::from_str(data)?;
            if let Value::Object(ref mut map) = json_data {
                map.insert("event".to_string(), Value::String(event.to_string()));
            }
            let json = serde_json::to_string(&json_data)?; 
            let bytes = json.as_bytes(); 

            let mut write_handle = self.write.lock().await;
            write_handle.write_all(bytes).await?;
        }
        Ok(())
    }

    #[allow(dead_code)]
    pub(crate) async fn on<F>(&self, event: &str, callback: F) -> io::Result<()>
    where
        F: Fn(String) + Send + Sync + 'static,
    {
        if let Some(true) = self.acivate {
            let mut store = self.callback_store.lock().await;
            store.insert(event.to_string(), Box::new(callback));
        
        }
        Ok(())
    }

    #[allow(dead_code)]
    pub(crate) async fn start_tcp(socket_addr: SocketAddr, acivate:bool) -> io::Result<Self> {
        let stream = TcpStream::connect(socket_addr).await?;
        let (read, write) = tokio::io::split(stream);
        Ok(Connection {
            read: Arc::new(Mutex::new(read)),
            write: Arc::new(Mutex::new(write)),
            callback_store: Arc::new(Mutex::new(HashMap::new())),
            acivate:Some(acivate)
        })
    }

    #[allow(dead_code)]
    pub(crate) async fn listen_for_events(self: Arc<Self>) {
        if let Some(true) = self.acivate {
            let mut buf = [0; 1024];
            loop {
                let mut read_handle = self.read.lock().await;
                let n = read_handle.read(&mut buf).await.expect("Failed to read from socket");
                if n == 0 {
                    break;
                }
                let message = String::from_utf8_lossy(&buf[..n]);
                if let Ok(parsed_message) = serde_json::from_str::<Value>(&message) {
                    if let Some(event) = parsed_message.get("event").and_then(|v| v.as_str()) {
                        let callback_store = self.callback_store.lock().await;
                        if let Some(callback) = callback_store.get(event) {
                            callback(message.to_string()); 
                        }
                    }
                } else {
                    eprintln!("Failed to parse message: {}", message);
                }
            }   
        }
    }
}




```
# main App
```rust
use std::sync::Arc;

use tao::event_loop::EventLoopProxy;


use super::{connections::Connection, events::{GlobalAPI, PythonEvent}};



#[allow(dead_code)]
pub(crate) async fn python_event_engine(
    from_py_to_proxy: EventLoopProxy<GlobalAPI>, 
    connection: Arc<Connection>

) {

    let handler = connection.on("evaluate_script", move |data|{
        let proxy = from_py_to_proxy.clone();
        let proxy_procres = proxy.send_event(GlobalAPI::Python(PythonEvent::EvaluateScript(data)));
        if let Err(e) = proxy_procres {
            eprintln!("Failed to send a proxy message from evaluate_script TCP Endpoint to main loop to evaluate_script from python: {:?}", e);
        }
    });
    if let Err(e) = handler.await {
        eprintln!("Failed to send a proxy message from evaluate_script TCP Endpoint to main loop to evaluate_script from python: {:?}", e);
    }
    
}


```

# main.rs
```rust
// remove console window in windows system
#![cfg_attr(
    all(not(debug_assertions), target_os = "windows"),
    windows_subsystem = "windows"
)]

use app::pyframe;

mod app;





#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    pyframe().await?;
    Ok(())
}



```
