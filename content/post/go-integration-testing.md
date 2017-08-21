---
title: "Go Integration Testing"
date: 2017-07-21T11:40:41+02:00
draft: true
---

```go

package blacklist

import (
	"context"
	"database/sql"
	"fmt"
	"net"
	"os"
	"testing"

	"google.golang.org/grpc"

	"sisu.sh/go/code/blacklist/store"
	"sisu.sh/go/code/blacklist/store/postgres"
	"sisu.sh/go/code/lib/config"
	"sisu.sh/go/code/lib/pgdb"
	"sisu.sh/go/code/lib/rpcutil"
	"sisu.sh/go/code/lib/sisuio"
	"sisu.sh/go/pb/blacklistpb"
)

var (
	cfg map[string]string
	clt blacklistpb.BlacklistServiceClient
	db  *sql.DB
)

var (
	ctx      = context.Background()
	dbTables = []string{"blacklist"}
)

func TestDeleteBlock(t *testing.T) {
	resetDB()

	var (
		ftype  = blacklistpb.FieldType_EMAIL
		fvalue = "test@simplesurance.de"
	)

	req := blacklistpb.CreateRequest{
		FieldType:  ftype,
		FieldValue: fvalue,
		ReasonText: "test reason",
	}
	_, err := clt.Create(ctx, &req)
	if err != nil {
		t.Fatalf("Can not add block for value '%s'\n", req.FieldValue)
	}
	breq := blacklistpb.BlockRequest{
		FieldType:  ftype,
		FieldValue: fvalue,
	}
	if _, err := clt.Delete(ctx, &breq); err != nil {
		t.Fatalf("Can't remove a block for value '%s'\n", req.FieldValue)
	}
	if _, err := clt.Check(ctx, &breq); err == nil {
		t.Fatalf("Block wasn't removed from blacklist '%s'\n", req.FieldValue)
	}
}

func startBlacklistService(storage store.Storer) {
	listener, err := net.Listen("tcp", cfg["rpc.address"])
	if err != nil {
		fmt.Println("failed to start listening: ", err)
		os.Exit(1)
	}
	server := rpcutil.DefaultGRPCServer()

	blacklistpb.RegisterBlacklistServiceServer(server, NewService(storage))

	rpcutil.RunSever(server, listener)
}

func resetDB() {
	for _, table := range dbTables {
		_, err := db.Exec("DELETE FROM " + table)
		if err != nil {
			fmt.Printf("deleting DB data from table %v failed, "+
				"error: %v\n", table, err)
			os.Exit(1)
		}
	}
}

func TestMain(m *testing.M) {
	var err error

	if err := config.RegisterDefaultFlags(); err != nil {
		fmt.Println("config.RegisterDefaultFlags() failed: ", err)
		os.Exit(1)
	}

	cfg, err = config.ParseAndProcess()
	if err != nil {
		fmt.Println("config.ParseAndProcess() failed: ", err)
		os.Exit(1)
	}

	db, err = pgdb.NewPGDB(pgdb.PGConfig{
		User:     cfg["database.user"],
		Password: cfg["database.password"],
		Host:     cfg["database.host"],
		Database: cfg["database.database"],
	})

	if err != nil {
		fmt.Println("Connecting to DB failed: ", err)
		os.Exit(1)
	}
	defer sisuio.Close(ctx, db)

	go startBlacklistService(postgres.NewStorage(db))

	srvAddr := cfg["rpc.address"]
	if string(cfg["rpc.address"][0]) == ":" {
		srvAddr = "localhost" + cfg["rpc.address"]
	}

	conn, err := grpc.Dial(srvAddr, grpc.WithInsecure())
	if err != nil {
		fmt.Printf("couldn't connect to RPC service %v: %v\n", srvAddr, err)
	}
	defer sisuio.Close(ctx, conn)

	clt = blacklistpb.NewBlacklistServiceClient(conn)
	// Call is done with FailFast=false to ensure the  the GRPCserver
	// started, otherwise tests might run because startGRPCService()
	// finished
	// unnecessary "_, _ =" assignment is to mute the static code checker
	// about unhandled error check here
	_, _ = clt.CheckFields(ctx, &blacklistpb.BlockListRequest{},
		grpc.FailFast(false))

	os.Exit(m.Run())
}
```