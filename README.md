# go-dbase

Golang package for reading FoxPro dBase database files.
This package provides a reader for reading FoxPro database files.

Since these files are almost always used on Windows platforms the default encoding is from Windows-1250 to UTF8 but a universal encoder will be provided for other code pages.
# Features 

This project is an extended clone of the [go-foxpro-dbf](https://github.com/SebastiaanKlippert/go-foxpro-dbf) package from [Sebastiaan Klippert](https://github.com/SebastiaanKlippert).

There are several similar packages like the go-foxpro-dbf package but they are not suited for our use case, this package implemented:

* Support for FPT (memo) files
* Full support for Windows-1250 encoding to UTF8
* File readers for scanning files (instead of reading the entire file to memory)
* Conversion to map, json and struct
* Non blocking IO operation with syscall

We also aim to support the following features:

* Writing to dBase database files

The focus is on performance while also trying to keep the code readable and easy to use.

# Supported field types

At this moment not all FoxPro field types are supported.
When reading field values, the value returned by this package is always `interface{}`. 
If you need to cast this to the correct value helper functions are provided.

The supported field types with their return Go types are: 

| Field Type | Field Type Name | Golang type |
|------------|-----------------|-------------|
| B | Double | float64 |
| C | Character | string |
| D | Date | time.Time |
| F | Float | float64 |
| I | Integer | int32 |
| L | Logical | bool |
| M | Memo  | string |
| M | Memo (Binary) | []byte |
| N | Numeric (0 decimals) | int64 |
| N | Numeric (with decimals) | float64 |
| T | DateTime | time.Time |
| Y | Currency | float64 |

# Installation
``` 
go get github.com/Valentin-Kaiser/go-dbase@latest
```

# Example

```go
package main

import (
	"fmt"
	"time"

	"github.com/Valentin-Kaiser/go-dbase"
)

type Test struct {
	ID          int32     `json:"ID"`
	Niveau      int32     `json:"NIVEAU"`
	Date        time.Time `json:"DATUM"`
	TIJD        string    `json:"TIJD"`
	SOORT       float64   `json:"SOORT"`
	ID_NR       int32     `json:"ID_NR"`
	UserNR      int32     `json:"USERNR"`
	CompanyName string    `json:"COMP_NAME"`
	CompanyOS   string    `json:"COMP_OS"`
	Melding     string    `json:"MELDING"`
	Number      float64   `json:"NUMBER"`
	Float       int64     `json:"FLOAT"`
	Bool        bool      `json:"BOOL"`
}

func main() {
	// Open file
	dbf, err := dbase.OpenFile("./test_data/TEST.DBF", new(dbase.Win1250Decoder))
	if err != nil {
		panic(err)
	}
	defer dbf.Close()

	// Print all the fieldnames
	for _, name := range dbf.FieldNames() {
		fmt.Println(name)
	}

	fmt.Println("--- database file fields --- \n")

	// Get fieldinfo for all fields
	for _, field := range dbf.Fields() {
		fmt.Println(field.FieldName(), field.FieldType(), field.Decimals)
	}

	err = dbf.GoTo(1)
	if err != nil {
		panic(err)
	}

	// Read the complete second record
	record, err := dbf.GetRecord()
	if err != nil {
		panic(err)
	}

	fmt.Println("--- database row as slice --- \n")

	// Print all the fields in their Go values
	fmt.Println(record.FieldSlice())

	// Go back to start
	err = dbf.GoTo(0)
	if err != nil {
		panic(err)
	}

	// Loop through all records using recordPointer in DBF struct
	// Reads the complete record
	for !dbf.EOF() {
		// This reads the complete record
		record, err := dbf.GetRecord()
		if err != nil {
			panic(err)
		}

		dbf.Skip(1)
		// skip deleted records
		if record.Deleted {
			continue
		}

		// get field by position
		_, err = record.Field(0)
		if err != nil {
			panic(err)
		}

		// get field by name
		_, err = record.Field(dbf.FieldPos("COMP_NAME"))
		if err != nil {
			panic(err)
		}

		fmt.Println("\n --- converted to struct --- \n")

		// convert record into struct
		t := &Test{}
		err = record.ToStruct(t)
		if err != nil {
			panic(err)
		}
		fmt.Printf("TESTDATA Company: %+v \n", t.CompanyName)
	}

	// Read only the third field of records 1, 2 and 3
	recnumbers := []uint32{1, 2, 3}
	for _, rec := range recnumbers {
		err := dbf.GoTo(rec)
		if err != nil {
			panic(err)
		}

		deleted, err := dbf.Deleted()
		if err != nil {
			panic(err)
		}

		if !deleted {
			field3, err := dbf.Field(3)
			if err != nil {
				panic(err)
			}
			fmt.Println(field3)
		}
	}
}

```

# Thanks

* To [Sebastiaan Klippert](https://github.com/SebastiaanKlippert) for the inspiration and the source code