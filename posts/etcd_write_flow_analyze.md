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

### Reference

- https://github.com/etcd-io/etcd
- https://etcd.io/docs/v3.6/