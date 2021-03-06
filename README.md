## Lease

Lease is a general DynamoDB-based lease implementation, ideal for long-lived work items, with coarse-grained leases.

#### Usage

Firstly, we need to initialize a lock store that wraps DynamoDB:

```go
// get the aws Config and DynamoDB instance however you like...
config := &aws.Config{
	Region: aws.String("us-east-1")}
ddb := dynamodb.New(config)

// create a new store, specifying the table and hash key
locker := lease.NewLockStore(ddb, "lock_table_name", "LockHashKeyName")
```

Then, we need to implement the `LeaseRequest` interface. We do this so we can  give the locker an identifier so it knows who's leasing what.

```go

type exampleLeaseRequest struct {
	processID string
}

// we only want one lease per process, so using the hostname:process_id,
func (er *exampleLeaseRequest) LesseeID() string {
	return er.processID
}

// take out a lease for 30 seconds
func (er *exampleLeaseRequest) LeaseDuration() time.Duration {
	return time.Second * 30
}
```

We can use this lease request to scan over all available items to lease until we get one:

```go
// returns an error if we fail to get a lease
lease, err := locker.ObtainLease(&exampleLeaseRequest{processID: "132123"})

// or we can block until we get a lease
lease := locker.WaitUntilLeaseObtained(&exampleLeaseRequest{processID: "132123"}, time.Second * 15)
```

Finally we can tell the locker to heartbeat to keep renewing a lease:

```go
go locker.Heartbeat(lease, time.Second * 15)
```

#### Testing

`go test ./...`

or with integration tests that require a real DynamoDB to connect to:

`LEASETESTTABLENAME=ddb_table_name LEASETESTHASHKEY=hash_key go test ./... -tags=integration`

#### Logging

Implement and set a logger conforming to the small LeaseLogger interface. Do this prior to use if you want logging to go anywhere. By default the library will work but you won't get any log messages.

```go
lease.SetLogger(myLogger)
```
