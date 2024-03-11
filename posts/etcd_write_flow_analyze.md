### Overview map

The best way to learn how a distributed storage system work is to analyze the execution flow of one of its basic operations.

Etcd opensource code provide a picture describe a write workflow on leader node. Let's start with it:

![Etcd write operation flow](https://github.com/etcd-io/etcd/blob/main/Documentation/etcd-internals/diagrams/write_workflow_leader.png?raw=true)

When we execute a put command with etcdctl.

```
etcdctl put mykey "this is awesome"
```

We well create a grpc client and call `Put` Rpc to comminicate with etcd server

```
code file : https://github.com/etcd-io/etcd/blob/main/etcdctl/ctlv3/command/put_command.go

core logic:

// putCommandFunc executes the "put" command.
func putCommandFunc(cmd *cobra.Command, args []string) {
	key, value, opts := getPutOp(args)

	ctx, cancel := commandCtx(cmd)
	resp, err := mustClientFromCmd(cmd).Put(ctx, key, value, opts...)
	cancel()
	if err != nil {
		cobrautl.ExitWithError(cobrautl.ExitError, err)
	}
	display.Put(*resp)
}
```

mustClientFromCmd create and return an `*clientv3.Client`, Within the Client implementation, a Put RPC request is sent to the etcd server.

We can find the RPC definition in file 

`https://github.com/etcd-io/etcd/blob/main/api/etcdserverpb/rpc.proto`

Now, let's look at the architecture diagram below.

![grpc](https://grpc-ecosystem.github.io/grpc-gateway/assets/images/architecture_introduction_diagram.svg)

Etcd use gRPC-Gateway for provide etcd's APIs in both gRPC and HTTP/JSON format at the same time, gRPC-Gateway converts HTTP requests into gRPC messages.

So we only need to focus on the implementation of the gRPC service.

We can find the ectdserver `Put` implementation code in https://github.com/etcd-io/etcd/blob/main/server/etcdserver/v3_server.go#L142

```
func (s *EtcdServer) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
	ctx = context.WithValue(ctx, traceutil.StartTimeKey{}, time.Now())
	resp, err := s.raftRequest(ctx, pb.InternalRaftRequest{Put: r})
	if err != nil {
		return nil, err
	}
	return resp.(*pb.PutResponse), nil
}
```
It called raftRequest, eventually `processInternalRaftRequestOnce` will be called

Here is the core logic of Etcd submitting requests to the state machine for processing.  I added detailed comments to this core code, Hope this helps you better understand the `Put` writing process.

```
func (s *EtcdServer) processInternalRaftRequestOnce(ctx context.Context, r pb.InternalRaftRequest) (*apply2.Result, error) {
	ai := s.getAppliedIndex()
	ci := s.getCommittedIndex()
	// If the server blocks too many requests, there is no apply, It will deny new requests coming in.
	if ci > ai+maxGapBetweenApplyAndCommitIndex {
		return nil, errors.ErrTooManyRequests
	}

	// Generate a unique ID for the request
	r.Header = &pb.RequestHeader{
		ID: s.reqIDGen.Next(),
	}

	// check authinfo if it is not InternalAuthenticateRequest
	if r.Authenticate == nil {
		authInfo, err := s.AuthInfoFromCtx(ctx)
		if err != nil {
			return nil, err
		}
		if authInfo != nil {
			r.Header.Username = authInfo.Username
			r.Header.AuthRevision = authInfo.Revision
		}
	}

	// Convert pb.InternalRaftRequest request to byte sequence.
	data, err := r.Marshal()
	if err != nil {
		return nil, err
	}

	if len(data) > int(s.Cfg.MaxRequestBytes) {
		return nil, errors.ErrRequestTooLarge
	}

	id := r.ID
	if id == 0 {
		id = r.Header.ID
	}
	ch := s.w.Register(id)

	cctx, cancel := context.WithTimeout(ctx, s.Cfg.ReqTimeout())
	defer cancel()

	start := time.Now()

	// Call raft core library propose to Submit a request to raft.
	err = s.r.Propose(cctx, data)
	if err != nil {
		proposalsFailed.Inc()
		s.w.Trigger(id, nil) // GC wait
		return nil, err
	}
	proposalsPending.Inc()
	defer proposalsPending.Dec()

	select {
	case x := <-ch:
	    // Wait for request to succeed.
		return x.(*apply2.Result), nil
	case <-cctx.Done():
		proposalsFailed.Inc()
		s.w.Trigger(id, nil) // GC wait
		return nil, s.parseProposeCtxErr(cctx.Err(), start)
	case <-s.done:
		return nil, errors.ErrStopped
	}
}
```



### Reference

- https://github.com/etcd-io/etcd
- https://etcd.io/docs/v3.6/