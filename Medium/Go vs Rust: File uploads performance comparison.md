


Go vs Rust: File uploads performance comparison
===============================================


This article is a part of the real-world use case series. The index of all the real-world comparisons is present in the article [here](/deno-the-complete-reference/list-of-all-of-my-real-world-performance-comparisons-a8e9182ac50).

I’m this article, I’m going to test & compare the multipart file upload performance between Go— Gin & Rust— Actix.

Setup
=====

All tests are executed on MacBook Pro M1 with 16G RAM.

The software versions are:

*   Go v1.25.0
*   Rust v1.70.0

The test tool is a custom tool built over libcurl with std threads that is capable of sending multipart requests.

There are 100K files in the asset directory. Each file size is exactly 100K. The number of files gets distributed between the tester workers. Same file is not uploaded again and again in all the requests. The workers loop through the allotted files. Once it exhausts all the allotted files, it goes back to the first file.

Each request carries two files in the multipart body. The request header & body is something like this:

```
// \-- Headers  
  
{  
  "content-length": "205150",  
  "content-type": "multipart/form-data; boundary=------------------------3f6a15690b315b91",  
}  
  
// \-- Body  
  
\--------------------------3f6a15690b315b91  
Content-Disposition: form-data; name="files"; filename="45469"  
Content-Type: application/octet-stream  
  
<<File suppressed>>  
\--------------------------3f6a15690b315b91  
Content-Disposition: form-data; name="files"; filename="42102"  
Content-Type: application/octet-stream  
  
<<file suppressed>>  
\--------------------------3f6a15690b315b91--
```

Code
====

Go
----

```go
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/jaevor/go-nanoid"
)

func main() {
	dst := "/Users/mayankc/Work/source/perfComparisons/uploads/"
	canonicID, err := nanoid.Standard(21)
	if err != nil {
		panic(err)
	}

	router := gin.New()
	router.POST("/upload", func(c *gin.Context) {
		form, _ := c.MultipartForm()
		files := form.File["files"]

		for _, file := range files {
			c.SaveUploadedFile(file, dst+canonicID())
		}
		c.Writer.WriteHeader(201)
	})
	router.Run(":3000")
}
```

Rust
----
```cargo.toml
[package]
name = "actix_multipart"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4" 
actix-multipart = "0.6"
nanoid = "0.4.0"
```

```rust
use actix_multipart::{
    form::{
        tempfile::{TempFile, TempFileConfig},
        MultipartForm,
    }
};
use actix_web::{middleware, web, App, Error, HttpResponse, HttpServer, Responder};
use nanoid::nanoid;

const BASE_DIR: &str = "/Users/mayankc/Work/source/perfComparisons/uploads/";

#[derive(Debug, MultipartForm)]
struct UploadForm {
    #[multipart(rename = "files")]
    files: Vec<TempFile>,
}

async fn save_files(
    MultipartForm(form): MultipartForm<UploadForm>,
) -> Result<impl Responder, Error> {
    for f in form.files {
        let path = format!("{}{}", BASE_DIR, nanoid!());
        f.file.persist(path).unwrap();
    }

    Ok(HttpResponse::Ok())
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Logger::default())
            .app_data(TempFileConfig::default().directory(BASE_DIR))
            .service(
                web::resource("/upload")
                    .route(web::post().to(save_files)),
            )
    })
    .bind(("127.0.0.1", 3000))?
    .run()
    .await
}
```

Rust code has been compiled in release mode.

Results
=======

Tests are executed for 10, 50, and 100 concurrent connections. A total of 100K requests are executed for each test.

Here are the results:

![](https://miro.medium.com/v2/resize:fit:700/1*tu598Rdvd3geM3l9eP5D-g.png)

![](https://miro.medium.com/v2/resize:fit:700/1*11jnJakY6fKUIyJ2nUVBaw.png)

![](https://miro.medium.com/v2/resize:fit:700/1*mnzQIE_13jdAmBUmqxxNBQ.png)

![](https://miro.medium.com/v2/resize:fit:700/1*_fJi0BV7d5gBIwdAepMJKw.png)

![](https://miro.medium.com/v2/resize:fit:700/1*9HCKBxkOsWF9kW-F-FBRPw.png)

![](https://miro.medium.com/v2/resize:fit:700/1*rjeix9wQdydjo7pKZT4MuQ.png)

![](https://miro.medium.com/v2/resize:fit:700/1*VlnQ0OAOoG7S0aGu5ooB2g.png)

![](https://miro.medium.com/v2/resize:fit:700/1*6DTz8zrMQbgi-C7jUhZTpA.png)

![](https://miro.medium.com/v2/resize:fit:700/1*UuR8V2sc2DXBS9fHVDOv7A.png)

Conclusion
==========

A scorecard is generated from the results using the following formula. For each measurement, get the winning margin. If the winning margin is:

*   < 5%, no points are given
*   between 5% and 20%, 1 point is given to the winner
*   between 20% and 50%, 2 points are given to the winner
*   \> 50%, 3 points are given to the winner

![](https://miro.medium.com/v2/resize:fit:600/1*tcbwnZOVk3nTjJKrMem_sQ.png)

![](https://miro.medium.com/v2/resize:fit:640/1*YFzEMORmcafYwDa8cF-OrA.png)