# Go ↔ Python: Part I - gRPC

### Introduction

Like tools, programming languages tend to solve problems they are designed to. You can use a knife to tighten a screw, but it's better to use a screwdriver, and there are less chances of you getting hurt in the process.

The Go programming language shines when writing high throughput services, and Python shines when doing data science. In this series of blog posts, we're going to explore how you can use each language to do the part it's better at. We'll see how you can communicate between and Python efficiently and explore the tradeoffs for each approach.

In this post, we’ll use [gRPC](https://grpc.io/) to pass messages from Go to Python and back.

This post assumes you know how to program both in Go and in Python. You don’t need to be an expert, but know the basics of each language.

### gRPC Overview

gRPC is an RPC (remote procedure call) framework from Google. It uses [Protocol Buffers](https://developers.google.com/protocol-buffers) as serialization format and uses [HTTP2](https://en.wikipedia.org/wiki/HTTP/2) as the transport medium.

By using these two well established technologies, you gain access to a lot of knowledge and tooling that's already available. Many companies I consult with use gRPC to connect internal services.

One more advantage of using protocol buffers is that you write the message definition once, and then generate bindings to every language from the same source. This means that various micro services can be written in different programming languages and agree on the format of message passed around.

Protocol buffers is also an efficient binary format, you gain both faster serialization times and less bytes over the wire, and this alone will save you money. Benchmarking on my machine, serialization time compared to JSON is about 7.5 faster and the data generated is about 4 times smaller.

### Example: Outlier Detection

[Outlier detection](https://en.wikipedia.org/wiki/Anomaly_detection), also called anomaly detection, is a way of finding suspicious values in data. Modern systems collect a lot of metrics on their services, and it's hard to come up with simple thresholds for when to wake someone at 2am.

We're going to have a Go service that collects metrics and then send them to a Python process using gRPC to do outlier detection. 

### Project Structure

There are many ways to structure a multi-service project. We're going with the simple approach of having Go as the main project and the Python project in a sub directory.

Listing show shows our directory structure

**Listing 1**  
```
.
├── client.go
├── gen.go
├── go.mod
├── go.sum
├── outliers.proto
├── pb
│   └── outliers.pb.go
└── py
    ├── Makefile
    ├── outliers_pb2_grpc.py
    ├── outliers_pb2.py
    ├── requirements.txt
    └── server.py
```

YOU NEED A SENTENCE HERE ABOUT LISTING 1
In listing 1 you see how I organized the project in N layers….  Something like that.

Our code is [go module](https://blog.golang.org/using-go-modules), the module name is defined in `go.mod` (see listing 2). I mention this since we’re going to reference the module name ( `github.com/ardanlabs/metrics`) in several places.

**Listing 2**
```
 1	module github.com/ardanlabs/metrics
 2	
 3	go 1.14
 4	
 5	require (
 6		github.com/golang/protobuf v1.4.2
 7		google.golang.org/grpc v1.29.1
 8		google.golang.org/protobuf v1.23.0
 9	)
```


### Defining Messages & Services 

In gRPC, you start by writing a `.proto` file which defines the messages being sent and the RPC methods.

Here's how `outliers.proto` looks:

**Listing 3**  
```
     1	syntax = "proto3";
     2	import "google/protobuf/timestamp.proto";
     3	package pb;
     4	
     5	option go_package = "github.com/ardanlabs/metrics/pb";
     6	
     7	message Metric {
     8	    google.protobuf.Timestamp time = 1;
     9	    string name = 2;
    10	    double value = 3;
    11	}
    12	
    13	message OutliersRequest {
    14	    repeated Metric metrics = 1;
    15	}
    16	
    17	message OutliersResponse {
    18	    repeated int32 indices = 1;
    19	}
    20	
    21	service Outliers {
    22	    rpc Detect(OutliersRequest) returns (OutliersResponse) {}
    23	}
```

In line 2 we import the protocol buffers definition of timestamp, and in line 5 we define the Go full package path.

A `Metric` is a piece of information with a timestamp, name (e.g. "CPU") and float value.

Every RPC method has an input type (or types) and an output type. Our input is `OutliersRequest` which is a list/slice of Metric. The `OutliersResponse` type is a list/slice of indices where the outlier values are.

### Python Server

In this section we’ll go over the Python server code. The Python server code is in the `py` directory. Here’s a refresher on how the directory structure looks like:

**Listing 3**  
```
.
├── client.go
├── gen.go
├── go.mod
├── go.sum
├── outliers.proto
├── pb
│   └── outliers.pb.go
└── py
    ├── Makefile
    ├── outliers_pb2_grpc.py
    ├── outliers_pb2.py
    ├── requirements.txt
    └── server.py
```

To generate the Python bindings you'll need the `protoc` compiler, which you can download from [here](https://developers.google.com/protocol-buffers/docs/downloads) or install using your operating system package manager (e.g. `apt-get`, `brew` ...)

Next you'll need to install the Python `grpcio-tools` package. 

_Note: I highly recommend using a virtual environment for all your Python projects. This topic is too much of a distraction. See [here](https://docs.python.org/3/tutorial/venv.html) to learn more.

**Listing 4**  
```
$ cat requirements.txt
grpcio-tools==1.29.0
numpy==1.18.4
$ python -m pip install -r requirements.txt
```

`requirements.txt` specifies the external requirements for our project, very much like how `go.mod` specifies requirements for Go projects. As you can see from the output of the `cat` command in listing 4 - we need two external modules `grpcio-tools` and `numpy`. A good practice is to have this file in source control and always version your dependencies (e.g. `numpy==1.18.4`), you probably do the same with your `go.mod` in your go projects.

Once you have the tools installed, you can generate the Python bindings:

**Listing 5**  
```
$ python -m grpc_tools.protoc \
	    -I.. --python_out=. --grpc_python_out=. \
	    ../outliers.proto
```

Let's break this long command in listing 5 down:

- `python -m grpc_tools.protoc` runs the `grpc_tools.protoc` module as a script
- `-I..` tells the tool where `.proto` files can be found
- `--python_out=.` tells the tool to generate the protocol buffers serialization code in the current directory
- `--grpc_python_out=.` tells the tool to generate the gRPC code in the current directory
- `../outliers.proto` is the name of the protocol buffers + gRPC definitions file

This command will run without any output, and at the end you'll see two new files: `outliers_pb2.py` which is the protocol buffers code and `outliers_pb2_grpc.py` which is the gRPC client and server code.

_Note: I usually use a `Makefile` to automate tasks in Python projects and create a `make` rule to run this command. I add the generated files to source control so that the deployment machine won't have to install the `protoc` compiler.

To write the Python server, you need to inherit from the `OutliersServicer` defined in `outliers_pb2_grpc.py` and override the `Detect` method. We're going to use the [numpy](https://numpy.org/) package and use a simple method of picking all the values that are more than two [standard deviations](https://en.wikipedia.org/wiki/Standard_deviation) from the [mean](https://en.wikipedia.org/wiki/Mean).

**Listing 6**  - server.py
```
     1	import logging
     2	from concurrent.futures import ThreadPoolExecutor
     3	
     4	import grpc
     5	import numpy as np
     6	
     7	from outliers_pb2 import OutliersResponse
     8	from outliers_pb2_grpc import OutliersServicer, add_OutliersServicer_to_server
     9	
    10	
    11	def find_outliers(data: np.ndarray):
    12	    """Return indices where values more than 2 standard deviations from mean"""
    13	    out = np.where(np.abs(data - data.mean()) > 2 * data.std())
    14	    # np.where returns a tuple for each dimension, we want the 1st element
    15	    return out[0]
    16	
    17	
    18	class OutliersServer(OutliersServicer):
    19	    def Detect(self, request, context):
    20	        logging.info('detect request size: %d', len(request.metrics))
    21	        # Convert metrics to numpy array of values only
    22	        data = np.fromiter((m.value for m in request.metrics), dtype='float64')
    23	        indices = find_outliers(data)
    24	        logging.info('found %d outliers', len(indices))
    25	        resp = OutliersResponse(indices=indices)
    26	        return resp
    27	
    28	
    29	if __name__ == '__main__':
    30	    logging.basicConfig(
    31	        level=logging.INFO,
    32	        format='%(asctime)s - %(levelname)s - %(message)s',
    33	    )
    34	    server = grpc.server(ThreadPoolExecutor())
    35	    add_OutliersServicer_to_server(OutliersServer(), server)
    36	    port = 9999
    37	    server.add_insecure_port(f'[::]:{port}')
    38	    server.start()
    39	    logging.info('server ready on port %r', port)
    40	    server.wait_for_termination()

```

Now you can run the server:

**Listing 7**  
```
$ python server.py
OUTPUT:
2020-05-23 13:45:12,578 - INFO - server ready on port 9999
```


### Go Client

Now that we have our Python server running, we can write the Go code to communicate with it. 

We'll start by generating Go bindings for gRPC. To automate this process, I usually have a file called `gen.go` with a `go:generate`  commands to generate the bindings. Other than the `protoc` compiler, you will need to get `github.com/golang/protobuf/protoc-gen-go` which is the gRPC plugin for Go.

**Listing 8**  
```
     1	package main
     2	
     3	//go:generate mkdir -p pb
     4	//go:generate protoc --go_out=plugins=grpc:pb --go_opt=paths=source_relative outliers.proto

```

Let's break down the command in line 4 which generates the bindings:
- `protoc` is the protocol buffer compiler
- `--go-out=plugins=grpc:pb` tells `protoc` to use the grpc plugin and place the files in the `pb` directory
- `--go_opt=source_relative` tells `protoc` to generate the code in the `pb` directory relative to the current directory
- `outliers.proto` is the name of the protocol buffers + gRPC definitions file

When you run `go generate` in the shell, you should see no output, but there will be a new file called `outliers.pb.go` in the `pb` directory. I add the `pb` directory to source control so `go get` of this package will work without requiring `protoc` to be installed on the client machine.

Now we can run the Go client. We'll fill an `OutliersRequest` struct with some dummy data and when calling the Python server we'll get back an `OutliersResponse` struct.

To help demonstrate the code, we have a `dummyData` function on line 35 that generates some data with outliers at indices 7, 113 and 835.

**Listing 9**  
```
     1	package main
     2	
     3	import (
     4		"context"
     5		"log"
     6		"math/rand"
     7		"time"
     8	
     9		"google.golang.org/grpc"
    10		pbtime "google.golang.org/protobuf/types/known/timestamppb"
    11	
    12		"github.com/ardanlabs/metrics/pb"
    13	)
    14	
    15	func main() {
    16		addr := "localhost:9999"
    17		conn, err := grpc.Dial(addr, grpc.WithInsecure(), grpc.WithBlock())
    18		if err != nil {
    19			log.Fatal(err)
    20		}
    21		defer conn.Close()
    22	
    23		client := pb.NewOutliersClient(conn)
    24		req := &pb.OutliersRequest{
    25			Metrics: dummyData(),
    26		}
    27	
    28		resp, err := client.Detect(context.Background(), req)
    29		if err != nil {
    30			log.Fatal(err)
    31		}
    32		log.Printf("outliers at: %v", resp.Indices)
    33	}
    34	
    35	func dummyData() []*pb.Metric {
    36		const size = 1000
    37		out := make([]*pb.Metric, size)
    38		t := time.Date(2020, 5, 22, 14, 13, 11, 0, time.UTC)
    39		for i := 0; i < size; i++ {
    40			m := &pb.Metric{
    41				Time: Timestamp(t),
    42				Name: "CPU",
    43				// normally we're below 40% CPU utilization
    44				Value: rand.Float64() * 40,
    45			}
    46			out[i] = m
    47			t.Add(time.Second)
    48		}
    49		// Create some outliers
    50		out[7].Value = 97.3
    51		out[113].Value = 92.1
    52		out[835].Value = 93.2
    53		return out
    54	}
    55	
    56	// Timestamp converts time.Time to protobuf *Timestamp
    57	func Timestamp(t time.Time) *pbtime.Timestamp {
    58		var pbt pbtime.Timestamp
    59		pbt.Seconds = t.Unix()
    60		pbt.Nanos = int32(t.Nanosecond())
    61		return &pbt
    62	}


```

Some highlights from the code in listing 9
In line 17 we connect to the Python server, using the `WithInsecure` option since the Python server we wrote do not support HTTPS
In line 23 we create a new OutliersClient with the connection we’ve made on line 17
In line 24 we create the gPRC request
In line 28 we perform the actual gRPC call. Every gRPC call has a `context.Context` as first parameter, allowing you to control timeouts and cancellations
gRPC has it’s own implementation of a `Timestamp` struct. In line 57 we have a utility function to convert from Go’s `time.Time` to gRPC `Timestamp`

Assuming the Python server is running on the same machine, we can run the client:

**Listing 10**  
```
$ go run client.go

OUTPUT:
2020/05/23 14:07:18 outliers at: [7 113 835]
```

### Conclusion

gRPC makes it easy and safe to call from one service to another. You get one place where all data types and methods are defined and great tooling and best practices around the framework.

The whole code: `weather.proto`, `py/server.py` and `client.go` is less than a 100 lines. You can view the project code [here](https://github.com/353solutions/blog-code/blob/master/grpc/)

There is much more to gRPC - timeout, load balancing, TLS, streaming and more. I highly recommend going over the [official site](https://grpc.io/) read the documentation and play with the provided examples.


